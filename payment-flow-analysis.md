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

---

## 7. 多网关排障：先授权后扣款与异步回调的分歧处理

### 7.1 两种支付网关对比

#### 7.1.1 Stripe 网关（新交易模型）

**核心特性**：
- 使用 `TransactionItem` + `TransactionEvent` 的事件溯源模式
- 通过 webhook 异步通知状态变更
- 支持 3D Secure 等强认证流程
- 授权和扣款可使用同一 `psp_reference`（PaymentIntent ID）

**先授权后扣款流程**（`saleor/payment/gateways/stripe/plugin.py:56`）：

```
1. transactionInitialize → 创建 AUTHORIZATION_REQUEST
2. transactionProcess → 调用 Stripe API 创建 PaymentIntent
3. Stripe 返回 requires_capture → 创建 AUTHORIZATION_SUCCESS
4. 业务侧调用 capture 接口 → 创建 CHARGE_REQUEST
5. Stripe 处理扣款 → webhook 通知 payment_intent.succeeded
6. webhook handler 创建 CHARGE_SUCCESS
```

**异步回调处理**（`saleor/payment/gateways/stripe/webhooks.py:50`）：

```python
webhook_handlers = {
    "payment_intent.succeeded": handle_successful_payment_intent,
    "payment_intent.amount_capturable_updated": handle_authorized_payment_intent,
    "payment_intent.processing": handle_processing_payment_intent,
    "payment_intent.payment_failed": handle_failed_payment_intent,
    "payment_intent.canceled": handle_failed_payment_intent,
    "charge.refunded": handle_refund,
}
```

**关键处理逻辑**（`webhooks.py:301`）：
```python
def handle_authorized_payment_intent(payment_intent, gateway_config, channel_slug):
    payment = _get_payment(payment_intent.id)  # 按 psp_reference 查询
    checkout = _get_checkout(payment.id)       # 先锁 checkout 再锁 payment
    # 幂等检查：是否已存在同类型 transaction
    transaction = _get_or_create_transaction(payment, payment_intent, AUTH, ...)
    # 如果支付已激活但关联了 checkout，触发 complete_checkout
```

#### 7.1.2 Braintree 网关（旧交易模型）

**核心特性**：
- 使用 `Payment` + `Transaction` 的旧模型
- 同步调用为主，部分场景支持 webhook
- `authorize()` 和 `capture()` 是两个独立的 API 调用
- 授权和扣款使用不同的 `transaction_id`

**先授权后扣款流程**（`saleor/payment/gateways/braintree/__init__.py:134`）：

```python
def authorize(payment_information, config):
    # 1. 创建 Braintree transaction，submit_for_settlement=False
    result = gateway.transaction.sale({
        "amount": str(payment_information.amount),
        "payment_method_nonce": payment_information.token,
        "options": {
            "submit_for_settlement": False,  # 仅授权，不扣款
        },
    })
    # 2. 返回 GatewayResponse，kind=AUTH
    return GatewayResponse(
        is_success=result.is_success,
        kind=TransactionKind.AUTH,
        transaction_id=result.transaction.id,  # 授权交易ID
        ...
    )

def capture(payment_information, config):
    # 1. 调用 Braintree submit_for_settlement
    result = gateway.transaction.submit_for_settlement(
        transaction_id=payment_information.token,  # 使用授权返回的ID
        amount=str(payment_information.amount),
    )
    # 2. 返回 GatewayResponse，kind=CAPTURE
    return GatewayResponse(
        is_success=result.is_success,
        kind=TransactionKind.CAPTURE,
        transaction_id=result.transaction.id,  # 扣款交易ID（与授权不同！）
        ...
    )
```

**关键差异**：Braintree 的授权和扣款产生两个不同的 `transaction_id`，而 Stripe 使用同一个 `PaymentIntent ID`。

#### 7.1.3 网关差异对比表

| 对比项 | Stripe | Braintree |
|--------|--------|-----------|
| **交易模型** | TransactionItem + TransactionEvent | Payment + Transaction |
| **psp_reference** | 授权和扣款共用 PaymentIntent ID | 授权和扣款各有独立 transaction_id |
| **状态通知** | 主要通过 webhook 异步通知 | 主要同步返回，可选 webhook |
| **3D Secure** | 原生支持，requires_action 状态 | 通过 three_d_secure 参数配置 |
| **自动捕获** | capture_method=automatic | submit_for_settlement=True |
| **部分扣款** | 支持，capture_amount < authorized_amount | 支持，submit_for_settlement 可指定金额 |
| **幂等键** | 客户端生成 idempotency_key | 由 Braintree SDK 内部处理 |
| **webhook 签名** | Stripe-Signature 头部 | Braintree webhook 签名验证 |

### 7.2 异步回调补事件的分歧处理

#### 7.2.1 Stripe webhook 处理流程

**处理顺序**（`webhooks.py:134-138`）：
```python
# 关键：先锁 checkout，再锁 payment，避免死锁
checkout = _get_checkout(payment.id)       # select_for_update on checkout
payment = _get_payment(payment_intent.id)  # select_for_update on payment
```

**幂等处理**（`webhooks.py:226-239`）：
```python
def _get_or_create_transaction(payment, stripe_object, kind, amount, currency):
    # 按 token=psp_reference + kind + is_success 查询
    transaction = payment.transactions.filter(
        token=stripe_object.id,
        action_required=False,
        is_success=True,
        kind=kind,
    ).last()
    if not transaction:
        # 创建新 transaction
        transaction = _update_payment_with_new_transaction(...)
    return transaction
```

**业务补单逻辑**（`webhooks.py:143-224`）：
```python
def _finalize_checkout(checkout, payment, payment_intent, kind, amount, currency):
    # 1. 检查是否已存在相同 transaction（防止重复处理）
    if not transaction:
        # 2. 创建 Transaction
        transaction = create_transaction(...)
        # 3. 更新 Payment charge_status
        update_payment_charge_status(payment, transaction)
        # 4. 调用 complete_checkout 创建订单
        order, _, _ = complete_checkout(...)
```

#### 7.2.2 Braintree webhook 处理（简化版）

Braintree 网关的 webhook 处理相对简单，主要依赖同步调用结果：

```python
# Braintree 的 webhook 主要用于：
# 1. 结算状态变更（transaction_settled）
# 2. 退款状态变更（transaction_refunded）
# 3. 争议处理（dispute_opened）
```

#### 7.2.3 常见分歧场景及处理

**场景 1：重复 webhook 通知**

| 网关 | 处理方式 | 代码位置 |
|------|----------|----------|
| Stripe | 按 `psp_reference + kind + is_success` 查询，已存在则跳过 | `webhooks.py:226` |
| Braintree | 同步调用时已处理，webhook 仅更新状态 | N/A |

**场景 2：webhook 先于同步响应到达**

Stripe 的处理逻辑天然支持：
```python
# 同步调用返回 requires_action
# 用户完成 3D Secure
# Stripe 发送 webhook（可能先于前端回调）
# webhook handler 检查 payment 是否已存在
# 如果已存在且关联 checkout，触发 complete_checkout
# 前端回调时发现 order 已创建，直接返回
```

**场景 3：授权成功但扣款失败**

Stripe 事件流：
```
1. AUTHORIZATION_SUCCESS (psp_ref=pi_xxx, amount=100)
2. CHARGE_REQUEST (psp_ref=pi_xxx, amount=100)
3. CHARGE_FAILURE (psp_ref=pi_xxx, amount=100)

计算结果：
authorized_value = 0  # 因为有 CHARGE_REQUEST，授权被消耗
charged_value = 0     # CHARGE_FAILURE 不增加
charge_pending_value = 0
```

Braintree 事件流：
```
1. Transaction (AUTH, txn_id=auth_123, amount=100, success=True)
2. Transaction (CAPTURE, txn_id=cap_456, amount=100, success=False)

计算结果：
Payment.captured_amount = 0
Payment.charge_status = NOT_CHARGED
```

---

## 8. 完整追踪路径：用一组交易标识串起全链路

### 8.1 统一追踪标识

以 Stripe 网关的一笔完整支付为例，使用以下标识串起全链路：

| 标识类型 | 示例值 | 说明 |
|----------|--------|------|
| **idempotency_key** | `uuid-1234-5678-abcd` | 客户端生成，用于防止重复初始化 |
| **TransactionItem.token** | `uuid-abcd-1234-5678` | Saleor 内部交易 ID，全局唯一 |
| **psp_reference** | `pi_3Nq...L4X` | Stripe PaymentIntent ID，网关侧唯一 |
| **order_id** | `order-001` | 关联的订单 ID |

### 8.2 完整调用链

#### 阶段 1：初始化交易

```
GraphQL: transactionInitialize
  → saleor/graphql/payment/mutations/transaction/transaction_initialize.py:155
    → handle_transaction_initialize_session() [saleor/payment/utils.py:1858]
      → TransactionItem.objects.get_or_create(idempotency_key=uuid-1234...)
      → TransactionEvent.objects.get_or_create(type=AUTHORIZATION_REQUEST)
      → manager.transaction_initialize_session()  [调用支付应用]
      → create_transaction_event_for_transaction_session() [utils.py:1430]
        → 解析支付应用响应
        → 更新 request_event 或创建新事件
        → recalculate_transaction_amounts()  [重算金额]
        → process_order_or_checkout_with_transaction()  [更新关联对象]
```

**数据库变化**：
- `TransactionItem` 新增记录，token=`uuid-abcd-1234-5678`
- `TransactionEvent` 新增 AUTHORIZATION_REQUEST 事件

#### 阶段 2：处理交易（调用网关）

```
GraphQL: transactionProcess
  → saleor/graphql/payment/mutations/transaction/transaction_process.py:177
    → get_transaction_item(id/token)  [查找 TransactionItem]
    → get_request_event(events)  [查找初始化时创建的 request 事件]
    → handle_transaction_process_session() [saleor/payment/utils.py:1952]
      → manager.transaction_process_session()  [调用 Stripe API 创建 PaymentIntent]
      → create_transaction_event_for_transaction_session() [utils.py:1430]
        → 根据网关响应创建 AUTHORIZATION_SUCCESS
        → psp_reference = pi_3Nq...L4X
        → recalculate_transaction_amounts()
          → authorized_value = 100
          → authorize_pending_value = 0
```

**数据库变化**：
- `TransactionEvent` 新增 AUTHORIZATION_SUCCESS 事件，psp_reference=`pi_3Nq...L4X`
- `TransactionItem.authorized_value` 更新为 100

#### 阶段 3：异步回调（webhook）

```
HTTP: POST /payments/stripe/webhook/
  → saleor/payment/gateways/stripe/webhooks.py:50 (handle_webhook)
    → construct_stripe_event()  [验证签名]
    → handle_successful_payment_intent()  [webhooks.py:431]
      → _get_payment(pi_3Nq...L4X)  [按 psp_reference 查询]
      → _get_checkout(payment.id)  [锁 checkout]
      → _get_payment(pi_3Nq...L4X, with_lock=True)  [锁 payment]
      → _get_or_create_transaction()  [幂等检查]
      → create_transaction()  [创建 CAPTURE transaction]
      → gateway_postprocess()  [更新 Payment 状态]
      → complete_checkout()  [创建订单]
        → Order.objects.create(...)  order_id=`order-001`
        → order_charged()  [触发 webhook]
```

**数据库变化**（旧模型）：
- `Transaction` 新增 CAPTURE 记录，token=`pi_3Nq...L4X`
- `Payment.captured_amount` 更新为 100
- `Payment.charge_status` 更新为 FULLY_CHARGED
- `Order` 新增记录，id=`order-001`

#### 阶段 4：新模型事件上报（App 调用）

```
GraphQL: transactionEventReport
  → saleor/graphql/payment/mutations/transaction/transaction_event_report.py:65
    → 权限检查
    → 对 TransactionItem 加行级锁
    → deduplicate_event()  [去重检查]
    → TransactionEvent.objects.create(type=CHARGE_SUCCESS, psp_ref=pi_3Nq...L4X)
    → recalculate_transaction_amounts()
      → 按 psp_reference 分组事件
      → 找到 CHARGE_REQUEST 和 CHARGE_SUCCESS
      → charged_value = 100
      → authorized_value = 0  [因为 CHARGE_REQUEST 消耗了授权]
    → process_order_with_transaction()
      → updates_amounts_for_order()  [更新订单金额]
        → Order.total_charged_amount = 100
        → Order.charge_status = FULLY_CHARGED
```

**数据库变化**（新模型）：
- `TransactionEvent` 新增 CHARGE_SUCCESS 事件
- `TransactionItem.charged_value` 更新为 100
- `TransactionItem.authorized_value` 更新为 0
- `Order.total_charged_amount` 更新为 100

### 8.3 端到端排查 SQL

当需要追踪某笔交易时，使用以下 SQL 查询：

```sql
-- 1. 按 psp_reference 查询 TransactionItem
SELECT * FROM payment_transactionitem 
WHERE psp_reference = 'pi_3Nq...L4X';

-- 2. 查询所有相关事件
SELECT * FROM payment_transactionevent 
WHERE transaction_id = (SELECT id FROM payment_transactionitem WHERE psp_reference = 'pi_3Nq...L4X')
ORDER BY created_at;

-- 3. 查询关联订单
SELECT * FROM order_order 
WHERE id = (SELECT order_id FROM payment_transactionitem WHERE psp_reference = 'pi_3Nq...L4X');

-- 4. 新旧模型关联查询（迁移期间）
SELECT * FROM payment_payment 
WHERE psp_reference = 'pi_3Nq...L4X';

SELECT * FROM payment_transaction 
WHERE payment_id = (SELECT id FROM payment_payment WHERE psp_reference = 'pi_3Nq...L4X')
ORDER BY created_at;
```

### 8.4 排障 Checklist

| 检查项 | 查询/操作 | 预期结果 |
|--------|-----------|----------|
| TransactionItem 是否存在 | 按 token 或 psp_reference 查询 | 存在且关联正确的 order/checkout |
| 事件序列是否完整 | 查询所有 events 按时间排序 | AUTHORIZATION_* → CHARGE_* 顺序正确 |
| 金额计算是否正确 | 调用 recalculate_transaction_amounts() | 各金额字段与事件总和一致 |
| 订单金额是否同步 | 对比 Order.total_charged_amount 与 TransactionItem.charged_value | 一致 |
| webhook 是否重复处理 | 查询 events 中是否有重复 psp_reference | 同一 psp_reference + type 只有一个 |
| 幂等键是否正确 | 检查 TransactionItem.idempotency_key | 与客户端传入一致 |

---

## 9. 关键代码索引（补充）

| 功能 | 文件 | 行号 |
|------|------|------|
| 交易初始化 mutation | `saleor/graphql/payment/mutations/transaction/transaction_initialize.py` | 155 |
| 交易处理 mutation | `saleor/graphql/payment/mutations/transaction/transaction_process.py` | 177 |
| 初始化会话处理 | `saleor/payment/utils.py` | 1858 |
| 处理会话处理 | `saleor/payment/utils.py` | 1952 |
| 创建交易会话事件 | `saleor/payment/utils.py` | 1430 |
| Stripe webhook 入口 | `saleor/payment/gateways/stripe/webhooks.py` | 50 |
| Stripe 成功支付处理 | `saleor/payment/gateways/stripe/webhooks.py` | 431 |
| Stripe 授权成功处理 | `saleor/payment/gateways/stripe/webhooks.py` | 301 |
| Braintree 授权 | `saleor/payment/gateways/braintree/__init__.py` | 134 |
| Braintree 扣款 | `saleor/payment/gateways/braintree/__init__.py` | 227 |
