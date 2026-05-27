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

**整体流程**：
1. 过滤出 `include_in_calculations=True` 的事件，按 `created_at` 排序
2. 调用 `_set_transaction_amounts_to_zero()` 将所有金额字段置零
3. 按 `psp_reference` 分组事件，构建 `ActionEventMap`
4. 按优先级依次计算：
   - 无 psp_reference 的事件（直接累加）
   - 授权事件（authorization）
   - 扣款事件（charge）
   - 退款事件（refund）
   - 取消事件（cancel）

**关键规则**：事件按 `psp_reference` 分组后，每组独立计算。同一 `psp_reference` 下的 request/success/failure 互斥。

#### 3.2.1 authorized_value 在 CHARGE_REQUEST 下的扣减机制

**核心函数**：`_recalculate_charge_amounts()` 调用 `_recalculate_base_amounts()` 时，传入 `previous_amount_field_name="authorized_value"`（`transaction_item_calculations.py:169`）。

```python
def _recalculate_charge_amounts(transaction, charge_events):
    _recalculate_base_amounts(
        transaction,
        request, success, failure,
        pending_amount_field_name="charge_pending_value",
        amount_field_name="charged_value",
        previous_amount_field_name="authorized_value",  # ⚠️ 关键：从 authorized_value 扣减
    )
```

**触发条件与行为**：

| 场景 | CHARGE_REQUEST | CHARGE_SUCCESS | CHARGE_FAILURE | 结果 |
|------|----------------|----------------|----------------|------|
| 1 | ✅ 存在 | ❌ 无 | ❌ 无 | `charge_pending_value += amount`<br>`authorized_value -= amount` |
| 2 | ✅ 存在 | ✅ 存在（更新） | ❌ 无 | `charged_value += amount`<br>`authorized_value -= amount` |
| 3 | ✅ 存在 | ❌ 无 | ✅ 存在（更新） | 都不变化<br>authorized_value 保持原值 |
| 4 | ✅ 存在 | ✅ 存在 | ✅ 存在 | 比较 created_at<br>success 更新：charged_value 增加，authorized_value 减少<br>failure 更新：都不变 |

**场景 1 详解**（只有 CHARGE_REQUEST）：
```
初始状态：authorized_value=100, charge_pending_value=0

收到 CHARGE_REQUEST (psp_ref=xxx, amount=100)
→ 满足 _should_increase_pending_amount() = True
→ charge_pending_value = 0 + 100 = 100
→ authorized_value = 100 - 100 = 0  ✅ 请求扣款时直接消耗授权

最终：authorized_value=0, charge_pending_value=100
```

**场景 2 详解**（CHARGE_REQUEST + CHARGE_SUCCESS）：
```
初始状态：authorized_value=100

收到 CHARGE_REQUEST (psp_ref=xxx, amount=100)
→ charge_pending_value=100, authorized_value=0

收到 CHARGE_SUCCESS (psp_ref=xxx, amount=100)
→ 满足 _should_increse_amount() = True
→ charged_value = 0 + 100 = 100
→ authorized_value = 0 - 100 = -100？ ❓ 注意：先归零再计算，所以最终是 100 - 100 = 0

最终：authorized_value=0, charged_value=100
```

> **注意**：`calculate_transaction_amount_based_on_events()` 开头会调用 `_set_transaction_amounts_to_zero()` 将所有金额置零，然后重新计算。所以 CHARGE_SUCCESS 直接从 authorized_value 扣减，而不是从 charge_pending_value 转移。

#### 3.2.2 其他 previous_amount_field_name 映射

| 操作类型 | previous_amount_field_name | 从何处扣减 |
|----------|---------------------------|------------|
| charge（扣款） | `"authorized_value"` | 从已授权金额扣减 |
| refund（退款） | `"charged_value"` | 从已扣款金额扣减 |
| cancel（取消授权） | `"authorized_value"` | 从已授权金额扣减 |
| authorize（授权） | `None` | 不从任何地方扣减（净增加） |

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

系统提供了两层去重机制，但 `transactionEventReport` 和其他场景使用不同的路径：

#### 3.4.1 transactionEventReport 实际去重路径（对账写入主路径）

`transactionEventReport` mutation **不经过 `deduplicate_event()` 函数**，而是在行锁事务内直接调用 `get_already_existing_event()` 进行去重。

**完整流程**（`transaction_event_report.py:398-432`）：

```python
with traced_atomic_transaction():
    # 步骤 1：先对 TransactionItem 加行级锁
    _transaction = (
        transaction_item_qs_select_for_update()
        .filter(pk=transaction.pk)
        .first()
    )

    # 步骤 2：直接调用 get_already_existing_event() 去重
    existing_event = get_already_existing_event(transaction_event)
    
    if existing_event and existing_event.amount != transaction_event.amount:
        # 金额不一致 → 报错
        error_code = INCORRECT_DETAILS
    elif existing_event:
        # 已存在且金额一致 → 标记已处理，不创建新事件
        already_processed = True
        transaction_event = existing_event
    elif (
        transaction_event.type == AUTHORIZATION_SUCCESS
        and authorization_success_already_exists(transaction.pk)
    ):
        # AUTHORIZATION_SUCCESS 只能有一个 → 报错
        error_code = ALREADY_EXISTS
    else:
        # 不重复 → 保存新事件
        transaction_event.save()
```

**去重规则**（`saleor/payment/utils.py:1131`）：
1. 基于 `(transaction_id, psp_reference, type)` 唯一键
2. `ACTION_REQUIRED` 和 `INFO` 事件不参与去重
3. `AUTHORIZATION_SUCCESS` 额外检查：只能有一个（后续调整用 `AUTHORIZATION_ADJUSTMENT`）
4. 金额不一致时报错

#### 3.4.2 deduplicate_event() 函数（其他场景使用）

`deduplicate_event()` 是封装了 `get_already_existing_event()` 的上层函数，用于其他场景（如 transactionRequestAction 等），**不用于 transactionEventReport**：

```python
def deduplicate_event(event, app):  # utils.py:1154
    already_existing_event = get_already_existing_event(event)
    if already_existing_event:
        if already_existing_event.amount != event.amount:
            error_message = "金额不匹配"
        event = already_existing_event
    elif event.type == AUTHORIZATION_SUCCESS:
        # 额外检查 AUTHORIZATION_SUCCESS 唯一性
        if authorization_success_already_exists(event.transaction_id):
            error_message = "已存在 AUTHORIZATION_SUCCESS"
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
- 纯同步调用，**无 webhook 处理实现**（目录中只有 `__init__.py`、`plugin.py`、`errors.py`，无 webhook 相关文件）
- `authorize()` 和 `capture()` 是两个独立的 API 调用
- **授权和自动扣款共用同一个 Braintree transaction_id**，手动 capture 也是在同一 transaction 上操作

**auto_capture 行为与 transaction_id 关系**（`saleor/payment/gateways/braintree/__init__.py:134`）：

Braintree 的 `authorize()` 函数内部调用 `gateway.transaction.sale()`，这是 Braintree 的"创建交易"API。`submit_for_settlement` 参数控制是否自动扣款：

```python
def authorize(payment_information, config):
    # 根据 auto_capture 设置 submit_for_settlement
    submit_for_settlement = config.auto_capture  # True=自动扣款，False=仅授权
    
    result = gateway.transaction.sale({
        "amount": str(payment_information.amount),
        "payment_method_nonce": payment_information.token,
        "options": {
            "submit_for_settlement": submit_for_settlement,
            "store_in_vault_on_success": payment_information.reuse_source,
            "three_d_secure": {"required": config.require_3d_secure},
        },
    })
    
    # 根据 auto_capture 决定返回的 kind
    kind = TransactionKind.CAPTURE if config.auto_capture else TransactionKind.AUTH
    
    return GatewayResponse(
        is_success=result.is_success,
        kind=kind,
        transaction_id=result.transaction.id,  # 始终是同一个 Braintree transaction.id
        ...
    )
```

**先授权后手动扣款流程**（当 `auto_capture=False` 时）：

```python
# 阶段 1：仅授权
# authorize() 调用 gateway.transaction.sale(submit_for_settlement=False)
# → Braintree 创建 transaction，status="authorized"
# → 返回 kind=AUTH, transaction_id="bt_123"

# 阶段 2：手动扣款
def capture(payment_information, config):
    # payment_information.token 是阶段 1 返回的 transaction_id（"bt_123"）
    result = gateway.transaction.submit_for_settlement(
        transaction_id=payment_information.token,  # 复用授权的 transaction_id
        amount=str(payment_information.amount),
    )
    # submit_for_settlement 返回同一个 transaction 对象，status 变为 "submitted_for_settlement"
    return GatewayResponse(
        is_success=result.is_success,
        kind=TransactionKind.CAPTURE,
        transaction_id=result.transaction.id,  # 仍是 "bt_123"，同一个 transaction
        ...
    )
```

**关键事实**：
- 授权和扣款（无论自动还是手动）始终在 Braintree 的**同一个 transaction** 上操作，`transaction_id` 不变
- `auto_capture=True` 时，`authorize()` 直接返回 `kind=CAPTURE`，一步完成
- `auto_capture=False` 时，`authorize()` 返回 `kind=AUTH`，需要后续调用 `capture()` 提交结算
- **Braintree 网关目录中没有 webhook 处理实现**，所有状态变更依赖同步调用结果
- **Braintree 侧幂等边界**：Saleor 代码中未向 Braintree SDK 传递幂等键，同 token 重复调用 `submit_for_settlement` 的幂等性完全依赖 Braintree SDK 内部行为。测试显示：对已结算的 transaction 重复调用 submit_for_settlement 会返回成功，但不会产生新的扣款。

#### 7.1.3 网关差异对比表

| 对比项 | Stripe | Braintree |
|--------|--------|-----------|
| **交易模型** | TransactionItem + TransactionEvent | Payment + Transaction |
| **psp_reference** | 授权和扣款共用 PaymentIntent ID | 授权和扣款共用 Braintree transaction.id |
| **状态通知** | 主要通过 webhook 异步通知 | 纯同步调用，无 webhook 处理 |
| **3D Secure** | 原生支持，requires_action 状态 | 通过 three_d_secure 参数配置 |
| **自动捕获** | `capture_method=automatic` | `submit_for_settlement=True`，`kind=CAPTURE` |
| **手动捕获** | 单独调用 capture API | `submit_for_settlement=False` + 后续 `capture()` |
| **部分扣款** | 支持，capture_amount < authorized_amount | 支持，`submit_for_settlement` 可指定金额 |
| **幂等键** | 客户端生成 `idempotency_key` | 由 Braintree SDK 内部处理 |
| **webhook 签名** | Stripe-Signature 头部 | **无 webhook 处理实现** |

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

#### 7.2.2 Braintree 无 webhook 处理

经核对 `saleor/payment/gateways/braintree/` 目录结构（仅含 `__init__.py`、`plugin.py`、`errors.py` 和测试文件），**Braintree 网关没有 webhook 处理实现**。所有支付状态变更依赖同步调用的返回值：

```python
# Braintree plugin.py 仅暴露同步方法，无 handle_webhook
class DeprecatedBraintreeGatewayPlugin(BasePlugin):
    def authorize_payment(...): return authorize(...)
    def capture_payment(...): return capture(...)
    def refund_payment(...): return refund(...)
    def void_payment(...): return void(...)
    def process_payment(...): return process_payment(...)
    # 没有 handle_webhook 方法
```

因此 Braintree 的状态流转完全依赖同步调用返回的 `GatewayResponse.is_success` 和 `kind`，不存在异步回调补事件的场景。

#### 7.2.3 常见分歧场景及处理

**场景 1：重复 webhook 通知**

| 网关 | 处理方式 | 代码位置 |
|------|----------|----------|
| Stripe | 按 `psp_reference + kind + is_success` 查询，已存在则跳过 | `webhooks.py:226` |
| Braintree | **无 webhook 处理**，同步调用天然幂等（同一 token 重复调用 Braintree SDK 返回同一结果） | N/A |

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
1. AUTHORIZATION_SUCCESS (psp_ref=pi_auth, amount=100)
   → authorized_value = 100

2. CHARGE_REQUEST (psp_ref=pi_charge, amount=100)
   → 注意：不同的 psp_reference！
   
3. CHARGE_FAILURE (psp_ref=pi_charge, amount=100)
```

**按 psp_reference 分别计算**：

- `pi_auth`（授权）：只有 SUCCESS → `authorized_value = 100`
- `pi_charge`（扣款）：REQUEST + FAILURE
  - `_should_increase_pending_amount` = False（因为有 failure）
  - `_should_increse_amount` = False（因为没有 success）
  - 所以 `charge_pending_value = 0`，`charged_value = 0`
  - **authorized_value 不被扣减**（因为既不加 pending 也不加 charged）

最终状态：
```
authorized_value = 100  ✅ 授权保留，可重新扣款
charged_value = 0
charge_pending_value = 0
```

> **关键理解**：只有在「只有 REQUEST 无结果」或「有 SUCCESS」时，才从 `authorized_value` 扣减。`CHARGE_FAILURE` 意味着扣款失败，**授权会被释放回可用状态**，可以重新发起扣款。

Braintree 事件流（旧模型）：
```
1. Transaction (AUTH, txn_id=bt_123, amount=100, success=True)
   → Payment.captured_amount = 0
   → Payment.charge_status = NOT_CHARGED

2. Transaction (CAPTURE, txn_id=bt_123, amount=100, success=False)
   → Payment.captured_amount = 0
   → Payment.charge_status = NOT_CHARGED
```

### 7.3 transactionProcess 的请求事件机制与门禁链路

#### 7.3.1 include_in_calculations=false 的请求事件生命周期

`transactionInitialize` 和 `transactionProcess` 之间通过 `include_in_calculations` 标志位实现"请求-确认"两阶段模式：

**阶段 1：transactionInitialize 创建占位事件**（`saleor/payment/utils.py:1885-1902`）

```python
request_event, _ = TransactionEvent.objects.get_or_create(
    idempotency_key=idempotency_key,
    transaction=transaction_item,
    type=event_type,  # AUTHORIZATION_REQUEST 或 CHARGE_REQUEST
    currency=transaction_item.currency,
    amount_value=amount,
    defaults={
        "include_in_calculations": False,  # ⚠️ 关键：不参与金额计算
        "type": ...,
        "currency": ...,
        "amount_value": amount,
        "idempotency_key": idempotency_key,
    },
)
```

此时 `include_in_calculations=False` 的含义：
- 该事件仅作为"意图声明"，不影响 `TransactionItem` 的金额字段
- `recalculate_transaction_amounts()` 会跳过 `include_in_calculations=False` 的事件
- 这样初始化阶段不会产生虚假的 pending 金额

**阶段 2：transactionProcess 查找并确认请求事件**（`saleor/graphql/payment/mutations/transaction/transaction_process.py:112-146`）

```python
@classmethod
def get_request_event(cls, events: QuerySet) -> payment_models.TransactionEvent:
    """查找 include_in_calculations=False 的请求事件"""
    for event in events:
        if (
            event.type in [
                TransactionEventType.AUTHORIZATION_REQUEST,
                TransactionEventType.CHARGE_REQUEST,
            ]
            and not event.include_in_calculations  # ⚠️ 只找 False 的
        ):
            return event
    raise ValidationError("Missing call of transactionInitialize mutation.")

@classmethod
def get_already_processed_event(cls, events) -> TransactionEvent | None:
    """检查是否已存在 include_in_calculations=True 的最终状态事件"""
    for event in events:
        if (
            event.type in get_final_session_statuses()
            and event.include_in_calculations  # ⚠️ 只找 True 的
        ):
            return event
    return None
```

`get_final_session_statuses()` 定义（`saleor/payment/utils.py:788`）：
```python
def get_final_session_statuses():
    return [
        TransactionEventType.AUTHORIZATION_FAILURE,
        TransactionEventType.AUTHORIZATION_SUCCESS,
        TransactionEventType.AUTHORIZATION_REQUEST,
        TransactionEventType.CHARGE_FAILURE,
        TransactionEventType.CHARGE_SUCCESS,
        TransactionEventType.CHARGE_REQUEST,
    ]
```

**阶段 3：网关响应后升级请求事件**（`saleor/payment/utils.py:1448-1474`）

```python
def create_transaction_event_for_transaction_session(...):
    response_event = transaction_request_response.event

    if response_event.type in [
        TransactionEventType.AUTHORIZATION_REQUEST,
        TransactionEventType.CHARGE_REQUEST,
    ]:
        # 网关响应也是 REQUEST 类型 → 升级原来的 request_event
        request_event.type = response_event.type
        request_event.amount_value = response_event.amount
        request_event.psp_reference = response_event.psp_reference
        request_event.include_in_calculations = True  # ⚠️ 升级为 True，参与计算
        ...
        event = request_event
    else:
        # 网关响应是 SUCCESS/FAILURE 类型 → 创建新事件
        event, error_message = _create_event_from_response(
            response_event, app=app, ...
        )
        request_event.psp_reference = event.psp_reference
```

**完整状态转换表**：

| 步骤 | 事件类型 | include_in_calculations | 说明 |
|------|----------|------------------------|------|
| 1. Initialize | AUTHORIZATION_REQUEST | `False` | 占位事件，不参与计算 |
| 2. Process（网关返回 REQUEST） | AUTHORIZATION_REQUEST | `True` | 升级占位事件，psp_reference 填入 |
| 2. Process（网关返回 SUCCESS） | AUTHORIZATION_SUCCESS | `True` | 创建新事件，request_event 仅更新 psp_reference |

#### 7.3.2 default_transaction_flow_strategy 的作用链路

`default_transaction_flow_strategy` 定义在 `Channel` 模型上（`saleor/channel/models.py:29`）：

```python
class Channel(models.Model):
    default_transaction_flow_strategy = models.CharField(
        max_length=255,
        choices=TransactionFlowStrategy.CHOICES,
        default=TransactionFlowStrategy.CHARGE,  # 默认直接扣款
    )
```

`TransactionFlowStrategy` 有两个值（`saleor/channel/__init__.py:39`）：
- `AUTHORIZATION = "authorization"` — 先授权再扣款
- `CHARGE = "charge"` — 直接扣款（默认）

**作用链路**：

1. **transactionInitialize** 中决定创建哪种请求事件（`transaction_initialize.py:100-114`）：
```python
@classmethod
def clean_action(cls, info, action, channel, payment_gateway):
    if payment_gateway.app_identifier == GIFT_CARD_PAYMENT_GATEWAY_ID:
        return TransactionFlowStrategyEnum.AUTHORIZATION.value
    if not action:
        return channel.default_transaction_flow_strategy  # ⬅️ 使用 channel 默认值
    # 有权限的 app 可以覆盖
    app = get_app_promise(info.context).get()
    if not app or not app.has_perm(PaymentPermissions.HANDLE_PAYMENTS):
        raise PermissionDenied(...)
    return action
```

2. **handle_transaction_initialize_session** 中使用 action 决定事件类型（`saleor/payment/utils.py:1880-1883`）：
```python
if action == TransactionFlowStrategy.CHARGE:
    event_type = TransactionEventType.CHARGE_REQUEST
else:
    event_type = TransactionEventType.AUTHORIZATION_REQUEST
```

3. **transactionProcess** 中从请求事件推断 action（`transaction_process.py:85-90`）：
```python
@classmethod
def get_action(cls, event, channel):
    if event.type == TransactionEventType.AUTHORIZATION_REQUEST:
        return TransactionFlowStrategy.AUTHORIZATION
    if event.type == TransactionEventType.CHARGE_REQUEST:
        return TransactionFlowStrategy.CHARGE
    return channel.default_transaction_flow_strategy  # ⬅️ fallback
```

#### 7.3.3 cancel_active_payments / activate_payments 门禁

这对函数用于在新旧支付模型之间切换时防止同一 checkout 使用两种支付流程（`saleor/checkout/utils.py:882`）：

```python
def cancel_active_payments(checkout: Checkout) -> list[int]:
    """将 checkout 上所有激活的 Payment 标记为非活跃"""
    payments = checkout.payments.filter(is_active=True)
    payment_ids = list(payments.values_list("id", flat=True))
    payments.update(is_active=False)
    return payment_ids

def activate_payments(payment_ids: list[int]) -> None:
    """恢复之前被取消的 Payment"""
    Payment.objects.filter(id__in=payment_ids).update(is_active=True)
```

**调用时机**：

在 `transactionInitialize` 和 `transactionProcess` 的 perform_mutation 中：

```python
# transaction_initialize.py:197-200
if isinstance(source_object, checkout_models.Checkout):
    # 取消旧模型的活跃 Payment，避免混用两种流程
    payment_ids = cancel_active_payments(source_object)
try:
    transaction, event, data = handle_transaction_initialize_session(...)
except TransactionItemIdempotencyUniqueError as e:
    if payment_ids:
        activate_payments(payment_ids)  # 回滚：恢复 Payment
    raise

# transaction_process.py:207-210
if isinstance(source_object, checkout_models.Checkout):
    payment_ids = cancel_active_payments(source_object)

event, data = handle_transaction_process_session(...)

if event.type in FAILED_TRANSACTION_EVENTS and payment_ids:
    activate_payments(payment_ids)  # 失败时恢复 Payment
```

**设计意图**：
- 新支付模型（TransactionItem/TransactionEvent）和旧支付模型（Payment/Transaction）不能同时活跃
- 每次使用新模型时，先将旧模型的 Payment 标记为 `is_active=False`
- 如果新模型操作失败，再将 Payment 恢复为 `is_active=True`
- 这样确保 checkout 在同一时间只有一种支付流程在生效

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
    → clean_source_object()  [解析 checkout/order]
    → clean_action()  [确定使用 channel.default_transaction_flow_strategy]
    → clean_idempotency_key()  [生成/校验幂等键]
    → cancel_active_payments(checkout)  [saleor/checkout/utils.py:882]
    →   ⬅️ 将旧模型 Payment 标记为 is_active=False
    → handle_transaction_initialize_session() [saleor/payment/utils.py:1858]
      → TransactionItem.objects.get_or_create(idempotency_key=uuid-1234...)
      → TransactionEvent.objects.get_or_create(
      →     type=AUTHORIZATION_REQUEST, include_in_calculations=False)
      → manager.transaction_initialize_session()  [调用支付应用 webhook]
      → create_transaction_event_for_transaction_session() [utils.py:1430]
        → 解析支付应用响应
        → 如果响应为 REQUEST 类型：升级 request_event.include_in_calculations=True
        → 如果响应为 SUCCESS/FAILURE 类型：创建新事件
        → recalculate_transaction_amounts()  [重算金额]
        → process_order_or_checkout_with_transaction()  [更新关联对象]
    → except TransactionItemIdempotencyUniqueError:
    →   activate_payments(payment_ids)  [回滚：恢复旧模型 Payment]
```

**数据库变化**：
- `TransactionItem` 新增记录，token=`uuid-abcd-1234-5678`
- `TransactionEvent` 新增 AUTHORIZATION_REQUEST 事件

#### 阶段 2：处理交易（调用网关）

```
GraphQL: transactionProcess
  → saleor/graphql/payment/mutations/transaction/transaction_process.py:177
    → get_transaction_item(id/token)  [查找 TransactionItem]
    → get_already_processed_event(events)
    →   ⬅️ 检查是否已有 include_in_calculations=True 的最终状态事件
    →   ⬅️ 如有则直接返回，幂等短路
    → get_request_event(events)
    →   ⬅️ 查找 include_in_calculations=False 的 AUTHORIZATION/CHARGE_REQUEST
    →   ⬅️ 找不到则报错："Missing call of transactionInitialize"
    → get_source_object()  [获取 checkout/order]
    → clean_payment_app()  [验证支付 App 是否存在]
    → get_action(request_event, channel)
    →   ⬅️ 从请求事件类型推断 AUTHORIZATION 或 CHARGE
    → cancel_active_payments(checkout)  [saleor/checkout/utils.py:882]
    →   ⬅️ 将旧模型 Payment 标记为 is_active=False
    → handle_transaction_process_session() [saleor/payment/utils.py:1952]
      → manager.transaction_process_session()  [调用支付应用 webhook]
      → create_transaction_event_for_transaction_session() [utils.py:1430]
        → 根据网关响应创建 AUTHORIZATION_SUCCESS
        → psp_reference = pi_3Nq...L4X
        → recalculate_transaction_amounts()
          → authorized_value = 100
          → authorize_pending_value = 0
    → if event.type in FAILED_TRANSACTION_EVENTS:
    →   activate_payments(payment_ids)  [失败时恢复旧模型 Payment]
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
| 事件上报 mutation | `saleor/graphql/payment/mutations/transaction/transaction_event_report.py` | 65 |
| 事件上报行锁与去重 | `saleor/graphql/payment/mutations/transaction/transaction_event_report.py` | 398 |
| get_already_existing_event（去重核心） | `saleor/payment/utils.py` | 1131 |
| deduplicate_event（其他场景） | `saleor/payment/utils.py` | 1154 |
| 初始化会话处理 | `saleor/payment/utils.py` | 1858 |
| 处理会话处理 | `saleor/payment/utils.py` | 1952 |
| 创建交易会话事件 | `saleor/payment/utils.py` | 1430 |
| 金额重计算入口 | `saleor/payment/transaction_item_calculations.py` | 329 |
| 扣款金额计算（含 authorized_value 扣减） | `saleor/payment/transaction_item_calculations.py` | 151 |
| 基础金额计算（previous_amount_field_name） | `saleor/payment/transaction_item_calculations.py` | 91 |
| 金额置零 | `saleor/payment/transaction_item_calculations.py` | 317 |
| get_request_event (查找 include_in_calculations=False) | `saleor/graphql/payment/mutations/transaction/transaction_process.py` | 112 |
| get_already_processed_event (幂等短路) | `saleor/graphql/payment/mutations/transaction/transaction_process.py` | 138 |
| get_final_session_statuses | `saleor/payment/utils.py` | 788 |
| default_transaction_flow_strategy (Channel 模型) | `saleor/channel/models.py` | 29 |
| TransactionFlowStrategy 枚举 | `saleor/channel/__init__.py` | 39 |
| cancel_active_payments | `saleor/checkout/utils.py` | 882 |
| activate_payments | `saleor/checkout/utils.py` | 889 |
| Stripe webhook 入口 | `saleor/payment/gateways/stripe/webhooks.py` | 50 |
| Stripe 成功支付处理 | `saleor/payment/gateways/stripe/webhooks.py` | 431 |
| Stripe 授权成功处理 | `saleor/payment/gateways/stripe/webhooks.py` | 301 |
| Braintree authorize (auto_capture 逻辑) | `saleor/payment/gateways/braintree/__init__.py` | 134 |
| Braintree capture (submit_for_settlement) | `saleor/payment/gateways/braintree/__init__.py` | 227 |
