# Saleor 支付流程分析

## 1. 核心数据模型

### 1.1 三层数据结构

Saleor 支付系统采用三层数据结构来跟踪支付状态：

| 模型 | 定义 | 核心字段 | 文件 |
|------|------|----------|------|
| **Payment** | 支付单，代表一次支付意图 | gateway, total, captured_amount, charge_status | `saleor/payment/models.py:273` |
| **TransactionItem** | 交易项，新交易模型 | authorized_value, charged_value, refunded_value, psp_reference | `saleor/payment/models.py:31` |
| **TransactionEvent** | 交易事件，记录每次状态变更 | type, amount_value, psp_reference, include_in_calculations | `saleor/payment/models.py:206` |

> **注意**：`Transaction` 模型（`saleor/payment/models.py:461`）是旧版实现，已逐步被 `TransactionItem` + `TransactionEvent` 的事件溯源模式取代。

### 1.2 TransactionItem 金额字段

`TransactionItem` 维护 8 个金额字段，所有字段都通过事件重新计算得出：

| 字段 | 含义 |
|------|------|
| `authorized_value` | 已授权金额 |
| `charged_value` | 已扣款金额 |
| `refunded_value` | 已退款金额 |
| `canceled_value` | 已取消金额 |
| `authorize_pending_value` | 授权待处理 |
| `charge_pending_value` | 扣款待处理 |
| `refund_pending_value` | 退款待处理 |
| `cancel_pending_value` | 取消待处理 |

### 1.3 TransactionEvent 类型

事件类型定义在 `saleor/payment/__init__.py:170`，共 18 种：

```python
# 授权类
AUTHORIZATION_REQUEST, AUTHORIZATION_SUCCESS, AUTHORIZATION_FAILURE, AUTHORIZATION_ADJUSTMENT, AUTHORIZATION_ACTION_REQUIRED

# 扣款类
CHARGE_REQUEST, CHARGE_SUCCESS, CHARGE_FAILURE, CHARGE_BACK, CHARGE_ACTION_REQUIRED

# 退款类
REFUND_REQUEST, REFUND_SUCCESS, REFUND_FAILURE, REFUND_REVERSE

# 取消类
CANCEL_REQUEST, CANCEL_SUCCESS, CANCEL_FAILURE

# 其他
INFO
```

---

## 2. 支付网关适配层

### 2.1 总体架构

```
Saleor Core ←→ Gateway Interface ←→ Plugin Manager ←→ 具体网关插件
                                                      ├─ StripePlugin
                                                      ├─ BraintreePlugin
                                                      ├─ RazorpayPlugin
                                                      └─ ...
```

### 2.2 统一接口定义

所有支付网关插件必须实现统一的接口，定义在 `saleor/payment/interface.py`：

**核心数据结构**：
- `GatewayResponse` - 网关响应统一格式 (`interface.py:303`)
- `PaymentData` - 支付请求统一格式 (`interface.py:377`)
- `GatewayConfig` - 网关配置 (`interface.py:428`)

**网关必须实现的方法**：
```python
process_payment()       # 处理支付
authorize_payment()     # 授权
capture_payment()       # 扣款
refund_payment()        # 退款
void_payment()          # 取消授权
confirm_payment()       # 确认支付
```

### 2.3 Stripe 网关实现示例

以 Stripe 插件为例（`saleor/payment/gateways/stripe/plugin.py:56`）：

**核心流程 - process_payment**：
1. 读取配置（API Key、自动捕获等）
2. 创建/获取 Stripe Customer
3. 调用 `create_payment_intent()` 创建支付意图
4. 处理 Stripe 响应状态：
   - `requires_action` → `ACTION_TO_CONFIRM`
   - `processing` → `PENDING`
   - `succeeded` → `CAPTURE`
   - `requires_capture` → `AUTH`
5. 返回统一格式的 `GatewayResponse`

**状态映射**（`plugin.py:160`）：
```python
Stripe 状态       → Saleor TransactionKind
requires_action  → ACTION_TO_CONFIRM
processing       → PENDING
succeeded        → CAPTURE
requires_capture → AUTH
```

### 2.4 网关调用流程

网关调用入口在 `saleor/payment/gateway.py`，使用装饰器模式：

```python
@raise_payment_error       # 错误处理
@require_active_payment    # 检查支付是否激活
@with_locked_payment       # 行级锁防止并发
@payment_postprocess       # 交易后处理
def process_payment(payment, token, manager, ...):
    payment_data = create_payment_information(...)
    response, error = _fetch_gateway_response(
        manager.process_payment, payment.gateway, payment_data, ...
    )
    return get_already_processed_transaction_or_create_new_transaction(...)
```

**关键装饰器说明**：
- `@with_locked_payment` (`gateway.py:83`)：使用 `select_for_update` 对 Payment 加行级锁，防止并发修改
- `@payment_postprocess` (`gateway.py:65`)：交易成功后更新 Payment 状态
- `@raise_payment_error` (`gateway.py:55`)：统一错误处理

---

## 3. 交易状态机（事件溯源模式）

### 3.1 设计思想

Saleor 采用 **事件溯源（Event Sourcing）** 模式来管理交易状态：
- 不直接更新状态字段
- 所有状态变更都通过 `TransactionEvent` 记录
- 当前状态通过聚合所有事件计算得出

### 3.2 金额重计算逻辑

核心算法在 `saleor/payment/transaction_item_calculations.py:329` 的 `calculate_transaction_amount_based_on_events()`：

**步骤**：
1. 过滤出 `include_in_calculations=True` 的事件
2. 按 `psp_reference` 分组事件
3. 对每组事件进行计算：
   - 只有 `request` 事件但无 `success/failure` → 增加 pending 金额
   - 有 `success` 事件 → 增加对应金额（如 charged_value）
   - `success` 和 `failure` 同时存在 → 比较 `created_at`，取较新的
   - `adjustment` 事件 → 直接覆盖 authorized_value
   - `charge_back` → 减少 charged_value
   - `refund_reverse` → 增加 charged_value，减少 refunded_value

**关键规则**（`transaction_item_calculations.py:64`）：
```python
def _should_increase_pending_amount(request, success, failure):
    # 只有request且无success/failure时，才增加pending金额
    if request and not failure and not success:
        return True
    return False

def _should_increse_amount(success, failure):
    if success and failure:
        # 同时存在时，取时间较新的
        return success.created_at > failure.created_at
    elif success:
        return True
    return False
```

### 3.3 状态流转示例

一次完整的支付流程可能产生如下事件序列：

```
1. AUTHORIZATION_REQUEST (amount=100)
   → authorize_pending_value = 100

2. AUTHORIZATION_SUCCESS (psp_ref=xxx, amount=100)
   → authorized_value = 100, authorize_pending_value = 0

3. CHARGE_REQUEST (psp_ref=yyy, amount=100)
   → charge_pending_value = 100, authorized_value = 0

4. CHARGE_SUCCESS (psp_ref=yyy, amount=100)
   → charged_value = 100, charge_pending_value = 0
```

### 3.4 事件去重机制

为防止网关重复上报，系统实现了完善的去重逻辑（`saleor/payment/utils.py:1154`）：

**去重规则**：
1. 基于 `(transaction_id, psp_reference, type)` 唯一键
2. `AUTHORIZATION_SUCCESS` 只能有一个（后续调整用 `AUTHORIZATION_ADJUSTMENT`）
3. 金额不一致时创建失败事件并告警

**代码**（`utils.py:1154`）：
```python
def deduplicate_event(event, app):
    already_existing_event = get_already_existing_event(event)
    if already_existing_event:
        if already_existing_event.amount != event.amount:
            error_message = "金额不匹配"
        event = already_existing_event
    return event, error_message
```

---

## 4. 对账写入流程

### 4.1 事件上报入口

网关通过 `transactionEventReport` mutation 上报交易事件（`saleor/graphql/payment/mutations/transaction/transaction_event_report.py:65`）。

**核心参数**：
```graphql
mutation {
  transactionEventReport(
    id: "VHJhbnNhY3Rpb25JdGVtOjE="  # 或 token
    pspReference: "pi_xxx"           # 网关侧交易ID
    type: CHARGE_SUCCESS             # 事件类型
    amount: 100.00                   # 事件金额
    time: "2024-01-01T00:00:00Z"     # 事件发生时间
  ) {
    alreadyProcessed
    transaction { id }
  }
}
```

### 4.2 处理流程

完整的事件处理流程（`transaction_event_report.py:299`）：

```
1. 权限检查
   → 检查 app/user 是否有权限操作该 transaction

2. 参数校验
   → amount 必填性检查
   → amount 精度格式化

3. 事务内处理
   → 对 TransactionItem 加行级锁 (select_for_update)
   → 检查事件是否已存在（去重）
   → 保存新 TransactionEvent

4. 更新 TransactionItem
   → 调用 recalculate_transaction_amounts() 重算金额
   → 更新 available_actions
   → 更新 payment_method_details

5. 更新关联对象
   → 订单：process_order_with_transaction()
   → 购物车：transaction_amounts_for_checkout_updated()
```

### 4.3 订单金额更新

当交易事件关联订单时，调用 `updates_amounts_for_order()`（`saleor/order/utils.py:1085`）：

```python
def updates_amounts_for_order(order, save=True):
    # 1. 更新扣款状态和金额
    update_order_charge_data(order, ...)
    # 2. 更新授权状态和金额
    update_order_authorize_data(order, ...)
    # 3. 保存订单
    order.save(update_fields=[
        "total_charged_amount", "charge_status",
        "total_authorized_amount", "authorize_status"
    ])
```

**订单状态流转**：
- `charge_status`：NOT_CHARGED → PENDING → PARTIALLY_CHARGED → FULLY_CHARGED
- `authorize_status`：NONE → PARTIAL → FULL
- 触发 `ORDER_PAID` / `ORDER_FULLY_PAID` 等 webhook

### 4.4 幂等性保证

系统通过多层机制保证幂等：

1. **数据库唯一约束**：`TransactionEvent` 的 `(transaction_id, idempotency_key)` 唯一约束 (`models.py:266`)
2. **应用层去重**：`deduplicate_event()` 基于 `psp_reference` + `type` 去重
3. **幂等性key**：`TransactionItem.idempotency_key` 用于防止重复请求

---

## 5. 跟踪交易流的建议

### 5.1 关键查询字段

跟踪一笔交易时，按以下字段关联查询：

| 层级 | 关联字段 | 说明 |
|------|----------|------|
| 网关侧 | `psp_reference` | 网关返回的交易ID，全局唯一 |
| Saleor侧 | `TransactionItem.token` | UUID，Saleor内部交易ID |
| 事件关联 | `TransactionEvent.psp_reference` | 关联到网关事件 |
| 订单关联 | `TransactionItem.order_id` | 关联到订单 |

### 5.2 典型问题排查路径

```
问题：扣款金额不匹配
1. 按 psp_reference 查询 TransactionEvent
   → 检查所有相关事件的 amount_value
2. 检查事件时间线
   → 是否有重复上报？
   → success 和 failure 的时间顺序？
3. 调用 recalculate_transaction_amounts() 手动重算
   → 确认计算逻辑是否正确
4. 检查订单的 total_charged_amount
   → 对比 TransactionItem.charged_value
```

### 5.3 一次扣款的多次状态变更示例

以 Stripe 自动捕获为例，完整的事件流可能包含：

```
时间线：
T1: transactionInitialize 创建 TransactionItem
T2: AUTHORIZATION_REQUEST (pending)
T3: AUTHORIZATION_SUCCESS (psp_ref=pi_xxx)
T4: CHARGE_REQUEST (psp_ref=pi_xxx)  ← 同一psp_ref
T5: CHARGE_SUCCESS (psp_ref=pi_xxx)    ← 同一psp_ref

最终状态：
authorized_value = 0
charged_value = 100
```

> **关键点**：授权和扣款使用同一个 `psp_reference`，计算时会自动从 authorized_value 转移到 charged_value

---

## 6. 关键代码索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 网关调用入口 | `saleor/payment/gateway.py` | 272 |
| 事件上报mutation | `saleor/graphql/payment/mutations/transaction/transaction_event_report.py` | 65 |
| 金额重计算 | `saleor/payment/transaction_item_calculations.py` | 329 |
| 事件去重 | `saleor/payment/utils.py` | 1154 |
| 订单金额更新 | `saleor/order/utils.py` | 1085 |
| Stripe插件 | `saleor/payment/gateways/stripe/plugin.py` | 56 |
| 数据模型定义 | `saleor/payment/models.py` | 31, 206, 273 |
| 事件类型枚举 | `saleor/payment/__init__.py` | 170 |
