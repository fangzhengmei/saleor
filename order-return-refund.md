# Saleor Order Return 与退款数据链路处理

## 一、整体架构概览

Saleor 的退货退款系统围绕 **Fulfillment（履约单）** 模型设计，退货和退款在 Saleor 中是两个独立的操作维度，通过 `FulfillmentStatus` 状态字段区分。系统提供两个 GraphQL Mutation：

| Mutation | 核心函数 | 业务语义 |
|---------|---------|---------|
| `FulfillmentReturnProducts` | `create_fulfillments_for_returned_products` | 退货（可选叠加退款/换货） |
| `FulfillmentRefundProducts` | `create_refund_fulfillment` | 仅退款（不退货） |

核心文件位置：
- 业务逻辑：`saleor/order/actions.py`
- GraphQL 入口 + 校验基类：`saleor/graphql/order/mutations/`
- 库存管理：`saleor/warehouse/management.py`
- 模型定义：`saleor/order/models.py`
- 工具函数：`saleor/order/utils.py`
- 数据结构定义：`saleor/order/fetch.py`、`saleor/order/__init__.py`、`saleor/payment/interface.py`

---

## 二、数据结构精确定义与字段语义

### 2.1 OrderLineInfo（`saleor/order/fetch.py:44-50`）

```python
@dataclass
class OrderLineInfo:
    line: "OrderLine"              # 关联的订单行 ORM 对象
    quantity: int                  # 本操作涉及的**变更数量**（非订单行总量）
    variant: Optional["ProductVariant"] = None  # 商品变体（可能为 None，已删除商品）
    replace: bool = False         # 仅 Return 流程使用：是否换货
    warehouse_pk: UUID | None = None  # 发货仓库 PK（退货流程中始终为 None）
    line_discounts: Iterable["OrderLineDiscount"] | None = None  # 行级折扣
```

**字段语义要点**：
- `quantity` 的含义取决于上下文：在退货/退款中代表**要退/退款的件数**，在 `fetch_order_lines` 中等于 `line.quantity`（订单行总量）
- `replace` 仅在 `FulfillmentReturnProducts` 的 `clean_lines` 中被设为 `True`，`FulfillmentRefundProducts` 不使用此字段
- `warehouse_pk` 在退货流程中始终为 `None`，因为退货操作不涉及指定仓库发货；仅在正向 Fulfillment 创建时使用
- `variant` 可能为 `None`：当关联商品已被删除时。此时如果 `replace=True` 会在校验阶段报错

### 2.2 FulfillmentLineData（`saleor/order/__init__.py:306-310`）

```python
@dataclass
class FulfillmentLineData:
    line: "FulfillmentLine"   # 关联的履约行 ORM 对象
    quantity: int             # 要退/退款的件数
    replace: bool = False     # 仅 Return 流程使用：是否换货
```

**与 OrderLineInfo 的关键区别**：
- `FulfillmentLineData.line` 是 `FulfillmentLine`（已发货记录），不是 `OrderLine`（订单行）
- `FulfillmentLineData` 没有 `variant` 和 `warehouse_pk` 字段，因为 variant 可通过 `line.order_line.variant` 获取，仓库通过 `line.stock.warehouse` 获取
- `FulfillmentLineData.line.fulfillment.status` 直接决定了该行能否参与退款计算（`REFUNDED` 状态的行被跳过）

### 2.3 RefundData（`saleor/payment/interface.py:368-373`）

```python
@dataclass
class RefundData:
    order_lines_to_refund: list[OrderLineInfo] = field(default_factory=list)
    fulfillment_lines_to_refund: list[FulfillmentLineData] = field(default_factory=list)
    refund_shipping_costs: bool = False
    refund_amount_is_automatically_calculated: bool = True
```

**用途**：`RefundData` 是传递给支付网关的**元信息**，告知网关本次退款涉及的行和计算方式。它不参与 Saleor 内部的退款金额计算，而是通过 `PaymentInformation` 传递给第三方支付插件，使插件可以据此做额外逻辑（如部分退款的精确行映射）。

**`refund_amount_is_automatically_calculated`**：当调用方传入 `amount_to_refund` 时为 `False`，否则为 `True`。此标志帮助支付插件区分"Saleor 自动算出的金额"和"商户手动指定的金额"。

### 2.4 OrderFulfillmentLineInfo（`saleor/order/actions.py:89-91`）

```python
class OrderFulfillmentLineInfo(TypedDict):
    order_line: OrderLine
    quantity: int
```

**用途**：仅在正向 `create_fulfillments` 流程中使用，与退货退款流程无关。不要与 `OrderLineInfo` 混淆。

### 2.5 Fulfillment 模型关键字段（`saleor/order/models.py:750-814`）

```python
class Fulfillment(ModelWithMetadata):
    fulfillment_order = PositiveIntegerField(editable=False)  # 自增序号
    order = ForeignKey(Order, related_name="fulfillments")
    status = CharField(max_length=32, default=FulfillmentStatus.FULFILLED)
    tracking_number = CharField(max_length=255)
    shipping_refund_amount = DecimalField(null=True, blank=True)  # 运费退款额
    total_refund_amount = DecimalField(null=True, blank=True)     # 商品退款额
```

**`total_refund_amount` 和 `shipping_refund_amount` 的语义**：
- 两者都是**记录性字段**，仅在创建退款/退货 Fulfillment 时写入，不参与后续金额计算
- `total_refund_amount` = `_process_refund` 返回的商品退款额
- `shipping_refund_amount` = 由 `__get_shipping_refund_amount` 计算的运费退款额
- 当 `refund=False`（仅退货不退款）时，两者都为 `None`

---

## 三、输入校验约束详解

校验分为三层：GraphQL 层、基类 `FulfillmentRefundAndReturnProductBase`、业务函数内部。

### 3.1 Return 与 Refund 的校验差异

| 校验项 | FulfillmentReturnProducts | FulfillmentRefundProducts |
|-------|--------------------------|--------------------------|
| 支付校验 | 仅 `refund=True` 时校验 | **始终校验** |
| FulfillmentLine 允许状态 | `FULFILLED`、`REFUNDED`、`WAITING_FOR_APPROVAL` | `FULFILLED`、`RETURNED`、`WAITING_FOR_APPROVAL` |
| `replace` 字段 | 有 | 无 |
| `refund` 字段 | 有 | 无（隐式始终退款） |
| `amount_to_refund` 与礼品卡 | `refund=True` 时限制 | 始终限制 |

**关键差异**：Return 允许对 `REFUNDED` 状态的 FulfillmentLine 操作（已退款再退货），Refund 允许对 `RETURNED` 状态的 FulfillmentLine 操作（已退货再退款）。这确保了双向操作的完整性。

### 3.2 `clean_order_payment`（`fulfillment_refund_and_return_product_base.py:18-28`）

```python
@classmethod
def clean_order_payment(cls, payment, cleaned_input):
    if not payment or not payment.can_refund():
        raise ValidationError(...)
    cleaned_input["payment"] = payment
```

**`Payment.can_refund()` 条件**（`saleor/payment/models.py:446-452`）：
```python
def can_refund(self):
    can_refund_charge_status = (
        ChargeStatus.PARTIALLY_CHARGED,
        ChargeStatus.FULLY_CHARGED,
        ChargeStatus.PARTIALLY_REFUNDED,
    )
    return self.charge_status in can_refund_charge_status
```

即 `ChargeStatus.NOT_CHARGED` 和 `ChargeStatus.FULLY_REFUNDED` 时不能退款。

### 3.3 `clean_amount_to_refund`（`fulfillment_refund_and_return_product_base.py:31-60`）

三条规则：
1. **订单含礼品卡行时禁止手动指定金额**：`order_has_gift_card_lines(order)` 为 True 时，不允许传入 `amount_to_refund`，因为礼品卡退款需要精确到行
2. **手动退款金额不能超过已收款**：`amount_to_refund > charged_value` 时报错
3. **`amount_to_refund` 可为 None**：此时由 Saleor 自动计算

### 3.4 `clean_lines`（`fulfillment_refund_and_return_product_base.py:147-198`）

对**未发货订单行**（`order_lines`）的校验：

| 校验规则 | 错误码 | 说明 |
|---------|--------|------|
| `line.is_gift_card` | `GIFT_CARD_LINE` | 礼品卡行不允许退货/退款 |
| `line.quantity < quantity` | `INVALID_QUANTITY` | 请求退款件数超过订单行总量 |
| `line.quantity_unfulfilled < quantity` | `INVALID_QUANTITY` | 请求退件数超过未发货量 |
| `replace=True` 且 `line.variant_id is None` | `NOT_FOUND` | 已删除商品无法换货 |

**`quantity_unfulfilled` 计算方式**（`saleor/order/models.py:746-747`）：
```python
@property
def quantity_unfulfilled(self):
    return self.quantity - self.quantity_fulfilled
```

### 3.5 `clean_fulfillment_lines`（`fulfillment_refund_and_return_product_base.py:87-144`）

对**已发货履约行**（`fulfillment_lines`）的校验：

| 校验规则 | 错误码 | 说明 |
|---------|--------|------|
| `line.order_line.is_gift_card` | `GIFT_CARD_LINE` | 礼品卡行不允许退货/退款 |
| `line.quantity < quantity` | `INVALID_QUANTITY` | 请求件数超过该履约行数量 |
| `line.fulfillment.status not in whitelisted_statuses` | `INVALID` | Fulfillment 状态不在允许列表 |
| `replace=True` 且 `line.order_line.variant_id is None` | `NOT_FOUND` | 已删除商品无法换货 |

### 3.6 校验中的边界场景

**场景 1：同一 OrderLine 同时出现在 `order_lines` 和 `fulfillment_lines` 中**
- 这是合法的。例如一个订单行有 5 件，2 件已发货、3 件未发货，可以同时指定 `order_lines: [{orderLineId, quantity: 3}]` 和 `fulfillment_lines: [{fulfillmentLineId, quantity: 2}]`

**场景 2：同一 FulfillmentLine 部分数量退款**
- 合法。`quantity` 可以小于 `fulfillment_line.quantity`，剩余数量留在原 Fulfillment 中

**场景 3：`order_lines` 和 `fulfillment_lines` 都为空**
- 校验通过，但后续 `create_return_fulfillment` 会创建一个空的 Fulfillment（`status=RETURNED`），这在业务上无意义

---

## 四、退款触发条件与金额计算分支

### 4.1 退款触发的三层前提

```
第一层：GraphQL 校验
  refund=True（Return）或始终触发（Refund）
  → clean_order_payment: payment.can_refund() 必须为 True
  → clean_amount_to_refund: 手动金额校验
    ↓
第二层：业务函数入口
  create_fulfillments_for_returned_products:
    if refund and payment:  ← payment 为 None 时不退款（静默跳过）
  create_refund_fulfillment:
    payment 在校验阶段已保证非 None
    ↓
第三层：_process_refund 内部
  if amount is None → 自动计算
  if amount and payment → 实际执行退款
  amount=0 时静默跳过（不调用支付网关，不触发事件）
```

### 4.2 金额计算的两个分支

**分支 A：手动指定金额**（`amount_to_refund is not None`）

```
amount = amount_to_refund（用户指定）
运费不叠加（include_shipping_costs 被忽略）
shipping_refund_amount = None（Fulfillment 记录中为空）
```

**分支 B：自动计算**（`amount_to_refund is None`）

```
amount = _calculate_refund_amount(return_order_lines, return_fulfillment_lines)
if refund_shipping_costs:
    amount += order.shipping_price_gross_amount
shipping_refund_amount = order.shipping_price_gross_amount（如果 refund_shipping_costs=True）
```

### 4.3 `_calculate_refund_amount` 精确逻辑（`actions.py:1960-1990`）

```python
def _calculate_refund_amount(
    return_order_lines,          # 未发货的退货行
    return_fulfillment_lines,    # 已发货的退货行
    lines_to_refund,             # 输出参数：累计每行退款数量
) -> Decimal:
    refund_amount = Decimal(0)

    # 步骤 1：未发货行 —— 直接按 unit_price_gross_amount × quantity 累加
    for line_data in return_order_lines:
        refund_amount += line_data.quantity * line_data.line.unit_price_gross_amount
        lines_to_refund[line_data.line.id] = (line_data.quantity, line_data.line)

    # 步骤 2：已发货行 —— 跳过 fulfillment.status == REFUNDED 的行
    order_lines_with_fulfillment = OrderLine.objects.in_bulk(...)
    for line_data in return_fulfillment_lines:
        if line_data.line.fulfillment.status == FulfillmentStatus.REFUNDED:
            continue  # ← 已退款的行不再重复计入
        order_line = order_lines_with_fulfillment[line_data.line.order_line_id]
        refund_amount += line_data.quantity * order_line.unit_price_gross_amount
        # 累加到 lines_to_refund（同 OrderLine 的数量会合并）
```

**关键边界**：
- `REFUNDED_AND_RETURNED` 状态的行**不被跳过**，只有 `REFUNDED` 状态被跳过。这意味着"已退货且退款"的行如果再次出现在退货请求中，金额会被重复计算——但这种情况在正常业务中不应发生，因为 `clean_fulfillment_lines` 的白名单在 Return 中不含 `REFUNDED_AND_RETURNED`
- **折扣不影响退款金额**：退款使用 `unit_price_gross_amount`（含税原价），不考虑 `unit_discount_amount`。这是一个容易混淆的点：虽然订单行有折扣字段，但退款计算并未引用它。如果订单使用了折扣，退款金额可能**大于**客户实际支付的金额

### 4.4 金额封顶与静默行为

```python
# actions.py:2020-2021
if amount and payment:
    amount = min(payment.captured_amount, amount)
```

- 退款金额被 `payment.captured_amount` 封顶，不会超过已收款
- **`amount=0` 时的行为**：`if amount and payment` 不成立，跳过退款调用。但 `_process_refund` 仍然返回 `amount=0`，后续 `create_refund_fulfillment` 会创建 `status=REFUNDED` 的 Fulfillment 并设置 `total_refund_amount=0`
- **`payment=None` 时的行为**（仅 Return 流程可能）：`if refund and payment` 不成立，跳过整个 `_process_refund`，`total_refund_amount=None`

### 4.5 `_validate_refund_amount` 二次校验（`payment/gateway.py:539-543`）

```python
def _validate_refund_amount(payment: Payment, amount: Decimal):
    if amount <= 0:
        raise PaymentError("Amount should be a positive number.")
    if amount > payment.captured_amount:
        raise PaymentError("Cannot refund more than captured.")
```

此校验在 `gateway.refund()` 内部，是对 `min(amount, captured_amount)` 之后的**双重保障**。正常流程中 `amount <= 0` 不应到达此处（已被 `if amount and payment` 过滤），但如果支付网关被其他路径调用，此校验仍然生效。

---

## 五、库存分配释放失败分支

### 5.1 `_move_order_lines_to_target_fulfillment` 中的 deallocate 失败处理

```python
# actions.py:1419-1432
line_allocations_exists = line_to_move.allocations.exists()
if line_allocations_exists:
    lines_to_dellocate.append(
        OrderLineInfo(line=line_to_move, quantity=unfulfilled_to_move)
    )

if lines_to_dellocate:
    try:
        deallocate_stock(lines_to_dellocate, site_settings, requestor)
    except AllocationError as e:
        lines = [str(line.pk) for line in e.order_lines]
        logger.warning(
            "Unable to deallocate stock for lines.", extra={"lines": lines}
        )
```

**关键行为**：`AllocationError` 被 **catch 并仅记录 warning**，不抛出异常。这意味着：
- 即使库存分配释放失败，退货/退款操作**仍然继续**
- 原因：`AllocationError` 表示"该行没有足够的 allocation 来释放"——这通常发生在 `track_inventory` 被关闭后又开启、或 Allocation 已经被其他操作释放的场景
- **后果**：`quantity_allocated` 可能不一致，需要手动修复

### 5.2 `deallocate_stock` 内部的 AllocationError 逻辑

```python
# warehouse/management.py:389-452
for line_info in order_lines_data:
    quantity_dealocated = 0
    for allocation in allocations:
        quantity_to_deallocate = min(
            (quantity - quantity_dealocated), allocation.quantity_allocated
        )
        ...
        if quantity_dealocated == quantity:
            break
    if not quantity_dealocated == quantity:
        not_dellocated_lines.append(order_line)  # ← 释放不足

if not_dellocated_lines:
    raise AllocationError(not_dellocated_lines)  # ← 抛出异常
```

**触发条件**：当 OrderLine 的 allocation 总量不足以释放请求的数量时抛出。典型场景：
1. `track_inventory` 曾被关闭（此时不会创建 Allocation），后来又被打开
2. Allocation 已被其他并发操作释放
3. 部分数量已发货（`decrease_stock` 已扣减 allocation），但请求释放的数量包含了已发货部分

### 5.3 `decrease_allocations` 中的容错处理

```python
# warehouse/management.py:568-582
def decrease_allocations(lines_info, site_settings, requestor):
    lines_to_deallocate = get_order_lines_to_deallocate(lines_info)
    if not lines_to_deallocate:
        return  # ← 无 allocation 时静默返回
    try:
        deallocate_stock(lines_info, site_settings, requestor)
    except AllocationError as exc:
        Allocation.objects.order_by("stock_id").filter(
            order_line__in=exc.order_lines
        ).update(quantity_allocated=0)  # ← 强制清零
```

**容错策略**：当释放失败时，将相关行的 `quantity_allocated` 强制置为 0。这是一种"宁可丢失分配记录也不阻塞业务"的策略。

### 5.4 退货流程中库存操作的完整链路

```
退货未发货行:
  _move_order_lines_to_target_fulfillment
    → deallocate_stock  [释放 allocation，减少 quantity_allocated]
    → 如果 AllocationError → warning 日志，继续执行
    → 注意：不增加 stock.quantity（因为商品从未实际出库）

退货已发货行:
  _move_fulfillment_lines_to_target_fulfillment
    → 纯数据移动（FulfillmentLine 间转移数量）
    → 不触发任何库存操作
    → 注意：stock.quantity 不回滚，需要通过 cancel_fulfillment 手动处理

取消发货:
  cancel_fulfillment
    → [warehouse 指定时] restock_fulfillment_lines
      → increase_stock(order_line, warehouse, quantity, allocate=True)
         ↑ stock.quantity += quantity 且 stock.quantity_allocated += quantity
    → [warehouse 未指定时] decrease_fulfilled_quantity
      → 仅减少 order_line.quantity_fulfilled，不操作库存
```

**纠正之前文档的错误**：之前的文档说"常规退货不自动回滚库存"——这是正确的，但缺少对 `_move_order_lines_to_target_fulfillment` 中 `deallocate_stock` 的说明。实际上退货未发货行时**确实会释放 allocation**（减少 `quantity_allocated`），只是不增加 `stock.quantity`（因为未发货商品从未从 `quantity` 中扣除）。只有已发货商品的退货才涉及 `stock.quantity` 的回滚。

---

## 六、事件语义差异详解

Saleor 的事件系统分为两层：**OrderEvent**（内部审计记录）和 **WebhookEvent**（外部通知）。退货退款流程涉及多个事件，语义容易混淆。

### 6.1 OrderEvent 层面

| 事件类型 | 触发函数 | 触发时机 | 参数 | 关联对象 |
|---------|---------|---------|------|---------|
| `FULFILLMENT_RETURNED` | `order_returned_event` | `create_return_fulfillment` 事务提交后（`transaction.on_commit`） | `{lines: [{quantity, line_pk, item}]}` | 写入**原订单** |
| `FULFILLMENT_REFUNDED` | `fulfillment_refunded_event` | `_process_refund` 事务提交后（`transaction.on_commit`） | `{lines: [{quantity, line_pk, item}], amount, shipping_costs_included}` | 写入**原订单** |
| `PAYMENT_REFUNDED` | `payment_refunded_event` | `order_refunded` 中（同步） | `{parameters: {amount, payment_id, payment_gateway}}` | 写入**原订单** |
| `FULFILLMENT_REPLACED` | `fulfillment_replaced_event` | `process_replace` 中（同步） | `{lines: [{quantity, line_pk, item}]}` | 写入**原订单** |
| `DRAFT_CREATED_FROM_REPLACE` | `draft_order_created_from_replace_event` | `process_replace` 中（同步） | `{related_order_pk, lines}` | 写入**新 Draft Order** |
| `ORDER_REPLACEMENT_CREATED` | `order_replacement_created` | `process_replace` 中（同步） | `{related_order_pk}` | 写入**原订单** |
| `FULFILLMENT_CANCELED` | `fulfillment_canceled_event` | `cancel_fulfillment` 中（同步） | `{composed_id}` | 写入**原订单** |
| `FULFILLMENT_RESTOCKED_ITEMS` | `fulfillment_restocked_items_event` | `cancel_fulfillment` 中（同步） | `{quantity, warehouse}` | 写入**原订单** |

### 6.2 事件触发顺序对比

**退货+退款场景** (`create_fulfillments_for_returned_products`, `refund=True`)：

```
[事务内]
1. _process_refund():
   1.1 gateway.refund() → 支付网关执行退款
   1.2 order_refunded():
       1.2.1 payment_refunded_event    ← 同步，OrderEvent
       1.2.2 send_order_refunded_confirmation  ← 同步，发邮件
       1.2.3 call_order_events(ORDER_REFUNDED / ORDER_FULLY_REFUNDED)  ← 同步，Webhook
   → transaction.on_commit:
       1.3 fulfillment_refunded_event  ← 延迟，OrderEvent

2. process_replace()（如果有换货行）:
   2.1 fulfillment_replaced_event      ← 同步
   2.2 draft_order_created_from_replace_event  ← 同步（写入新 Order）
   2.3 order_replacement_created       ← 同步

3. create_return_fulfillment():
   3.1 _move_lines_to_return_fulfillment()  ← 数据移动
   → transaction.on_commit:
       3.2 order_returned() → order_returned_event  ← 延迟
           → update_order_status()                 ← 延迟

4. call_order_event(ORDER_UPDATED)      ← 同步，Webhook
```

**仅退款场景** (`create_refund_fulfillment`)：

```
[事务内]
1. _process_refund():  ← 同上
2. 创建 status=REFUNDED 的 Fulfillment
3. _move_order_lines_to_target_fulfillment()
4. _move_fulfillment_lines_to_target_fulfillment()
5. 删除空 Fulfillment
6. call_order_event(ORDER_UPDATED)
```

### 6.3 事件语义的关键差异

**`FULFILLMENT_RETURNED` vs `FULFILLMENT_REFUNDED`**：

| 维度 | FULFILLMENT_RETURNED | FULFILLMENT_REFUNDED |
|-----|---------------------|---------------------|
| 语义 | 商品被标记为退货 | 退款被实际执行 |
| 触发条件 | `create_return_fulfillment` 完成后 | `_process_refund` 完成后 |
| 参数 | 仅 `{lines}` | `{lines, amount, shipping_costs_included}` |
| 金额信息 | 无 | 有 |
| 触发方式 | `transaction.on_commit`（延迟） | `transaction.on_commit`（延迟） |

**`PAYMENT_REFUNDED` vs `FULFILLMENT_REFUNDED`**：

| 维度 | PAYMENT_REFUNDED | FULFILLMENT_REFUNDED |
|-----|-----------------|---------------------|
| 语义 | 支付通道确认退款 | 业务层面的行级退款记录 |
| 参数 | `{amount, payment_id, payment_gateway}` | `{lines, amount, shipping_costs_included}` |
| 关联对象 | Payment 对象 | OrderLine 列表 |
| 触发方式 | 同步 | `transaction.on_commit`（延迟） |

**重要**：在退货+退款场景中，`FULFILLMENT_REFUNDED` 和 `FULFILLMENT_RETURNED` 都会被触发。前者在退款事务提交后，后者在退货 Fulfillment 创建事务提交后。两者记录的行可能不同——`FULFILLMENT_REFUNDED` 的 `lines_to_refund` 排除了 `replace=True` 的换货行，而 `FULFILLMENT_RETURNED` 的 `returned_lines` 包含所有非换货行。

### 6.4 Webhook 事件层

| Webhook 事件 | 触发时机 | 与 OrderEvent 的关系 |
|-------------|---------|---------------------|
| `ORDER_REFUNDED` | `order_refunded` 中 | 对应 `PAYMENT_REFUNDED` OrderEvent |
| `ORDER_FULLY_REFUNDED` | 退款总额 ≥ 订单总额时 | 无对应 OrderEvent |
| `ORDER_UPDATED` | 每个操作末尾 | 无对应 OrderEvent |
| `DRAFT_ORDER_CREATED` | 换货创建新订单时 | 对应 `DRAFT_CREATED_FROM_REPLACE` |

**`ORDER_FULLY_REFUNDED` 的判定逻辑**（`actions.py:496-516`）：
```python
total_refunded = Decimal(0)
# 累加旧 Payment 模型的退款
last_payment = payment if payment else order.get_last_payment()
if last_payment and last_payment.charge_status in [
    ChargeStatus.PARTIALLY_REFUNDED,
    ChargeStatus.FULLY_REFUNDED,
]:
    total_refunded += sum(
        last_payment.transactions.filter(
            kind=TransactionKind.REFUND, is_success=True
        ).values_list("amount", flat=True),
        Decimal(0),
    )
# 累加新 TransactionItem 模型的退款
total_refunded += sum(
    order.payment_transactions.all().values_list("refunded_value", flat=True),
    Decimal(0),
)
if total_refunded >= order.total.gross.amount:
    webhook_events.append(WebhookEventAsyncType.ORDER_FULLY_REFUNDED)
```

注意此判定**同时查询**了旧 Payment 和新 TransactionItem 两种支付模型，确保兼容性。

---

## 七、FulfillmentStatus 状态与校验白名单的对应关系

### 7.1 状态流转图

```
                    create_fulfillments (auto_approved=True)
  [新创建] ──────────────────────────────────────────────────→ FULFILLED
                    create_fulfillments (auto_approved=False)
  [新创建] ──────────────────────────────────────────────────→ WAITING_FOR_APPROVAL

  FULFILLED ── cancel_fulfillment(with warehouse) ──→ CANCELED + 库存回滚
  FULFILLED ── cancel_fulfillment(no warehouse) ────→ CANCELED + 仅减 quantity_fulfilled
  WAITING_FOR_APPROVAL ── approve_fulfillment ──────→ FULFILLED

  FULFILLED ── FulfillmentReturnProducts(refund=False) ──→ RETURNED（新建）
  FULFILLED ── FulfillmentReturnProducts(refund=True)  ──→ REFUNDED_AND_RETURNED（新建）
  FULFILLED ── FulfillmentRefundProducts               ──→ REFUNDED（新建）
  FULFILLED ── FulfillmentReturnProducts(replace=True) ──→ REPLACED（新建）
```

**注意**：退货/退款/换货**不是**修改原 Fulfillment 的状态，而是创建新 Fulfillment 并移动行。原 Fulfillment 状态不变，但其行可能被移空。

### 7.2 `clean_fulfillment_lines` 白名单差异详解

| Mutation | 允许的 FulfillmentLine 状态 | 业务含义 |
|---------|---------------------------|---------|
| `FulfillmentReturnProducts` | `FULFILLED`、`REFUNDED`、`WAITING_FOR_APPROVAL` | 退货可操作已退款行（退款后再退货） |
| `FulfillmentRefundProducts` | `FULFILLED`、`RETURNED`、`WAITING_FOR_APPROVAL` | 退款可操作已退货行（退货后再退款） |

**不允许的操作**：
- 对 `CANCELED` 状态的行操作：取消的发货已经处理完毕
- 对 `REPLACED` 状态的行操作：换货行已经关联到新订单
- Return 中对 `RETURNED` 状态行操作：已经退回的不能再退回
- Refund 中对 `REFUNDED` 状态行操作：已经退款的不能再退款

### 7.3 已退款行退货时的特殊处理

```python
# actions.py:1700-1734
fulfillment_lines_already_refunded = FulfillmentLine.objects.filter(
    fulfillment__order=order, fulfillment__status=FulfillmentStatus.REFUNDED
).values_list("id", flat=True)

for line_data in fulfillment_lines:
    if line_data.line.id in fulfillment_lines_already_refunded:
        refunded_fulfillment_lines_to_return.append(line_data)
    else:
        fulfillment_lines_to_return.append(line_data)
```

当退货请求包含已退款的 FulfillmentLine 时：
- 已退款的行被移动到一个**单独的** `REFUNDED_AND_RETURNED` 状态 Fulfillment
- 非退款的行移动到主退货 Fulfillment（`RETURNED` 或 `REFUNDED_AND_RETURNED`）
- 这保证了"退款后再退货"的数据清晰性

---

## 八、事务与 on_commit 语义

### 8.1 `transaction_with_commit_on_errors` vs `traced_atomic_transaction`

| 装饰器 | 行为 | 使用位置 |
|-------|------|---------|
| `transaction_with_commit_on_errors` | 事务块，错误时仍 commit（Django TEST 模式兼容） | `create_refund_fulfillment`、`_process_refund` |
| `traced_atomic_transaction` | 标准 atomic + tracing | `create_fulfillments_for_returned_products`、`create_return_fulfillment` |

**`_process_refund` 使用 `transaction_with_commit_on_errors`** 的原因：退款操作即使部分失败（如事件记录失败），退款本身已经提交给支付网关，不应回滚。

### 8.2 `transaction.on_commit` 的使用场景

在退货退款流程中，以下操作被延迟到事务提交后执行：

1. **`fulfillment_refunded_event`**：确保退款数据已持久化后再记录事件
2. **`order_returned` → `order_returned_event` + `update_order_status`**：确保退货 Fulfillment 已创建后再更新订单状态

**不使用 `on_commit` 的操作**：
- `payment_refunded_event`：同步执行，因为 `order_refunded` 在退款事务内被调用
- `fulfillment_replaced_event`：同步执行
- Webhook 触发（`call_order_event`）：同步执行

---

## 九、完整数据链路图（修正版）

### 9.1 退货+退款完整链路

```
GraphQL: FulfillmentReturnProducts
    ↓
clean_input()
    ├─ [refund=True] clean_order_payment → can_refund() 校验
    ├─ [refund=True] clean_amount_to_refund → 礼品卡/金额上限校验
    ├─ clean_lines → 未发货行校验（礼品卡/数量/换货variant）
    └─ clean_fulfillment_lines → 已发货行校验（白名单状态）
    ↓
create_fulfillments_for_returned_products()
    ├─ 按 replace 分组
    │   ├─ return_order_lines / return_fulfillment_lines (replace=False)
    │   └─ replace_order_lines / replace_fulfillment_lines (replace=True)
    │
    ├─ [refund=True AND payment≠None] _process_refund()
    │   ├─ [amount=None] _calculate_refund_amount()
    │   │   ├─ 未发货行: quantity × unit_price_gross_amount
    │   │   └─ 已发货行: 跳过 REFUNDED 状态, quantity × unit_price_gross_amount
    │   ├─ [amount=None AND refund_shipping_costs] += shipping_price_gross_amount
    │   ├─ amount = min(amount, payment.captured_amount)
    │   ├─ [amount>0] gateway.refund() → 支付网关
    │   │   ├─ _validate_refund_amount (二次校验)
    │   │   └─ [manual] create_transaction(REFUND) / [auto] manager.refund_payment()
    │   ├─ [amount>0] order_refunded()
    │   │   ├─ payment_refunded_event (OrderEvent, 同步)
    │   │   ├─ send_order_refunded_confirmation (邮件)
    │   │   └─ call_order_events(ORDER_REFUNDED, [ORDER_FULLY_REFUNDED]) (Webhook)
    │   └─ on_commit: fulfillment_refunded_event (OrderEvent, 延迟)
    │
    ├─ [replace 非空] process_replace()
    │   ├─ _move_lines_to_replace_fulfillment (status=REPLACED)
    │   ├─ create_replace_order (Draft Order, origin=REISSUE)
    │   ├─ fulfillment_replaced_event (同步)
    │   ├─ draft_order_created_from_replace_event (同步, 写入新订单)
    │   └─ order_replacement_created (同步, 写入原订单)
    │
    ├─ create_return_fulfillment()
    │   ├─ _move_lines_to_return_fulfillment()
    │   │   ├─ 创建 Fulfillment (status=RETURNED / REFUNDED_AND_RETURNED)
    │   │   ├─ _move_order_lines_to_target_fulfillment() (未发货行)
    │   │   │   ├─ order_line.quantity_fulfilled += unfulfilled_to_move
    │   │   │   ├─ deallocate_stock() → 释放 allocation (失败则 warning)
    │   │   │   └─ FulfillmentLine(stock_id=None)
    │   │   ├─ 分离已退款的 fulfillment_lines
    │   │   ├─ _move_fulfillment_lines_to_target_fulfillment() (已发货行)
    │   │   │   └─ 数量在 FulfillmentLine 间转移 (保留 stock_id)
    │   │   └─ [有已退款行] 创建 REFUNDED_AND_RETURNED Fulfillment
    │   └─ on_commit: order_returned() → order_returned_event + update_order_status
    │
    ├─ 删除空 Fulfillment (FULFILLED / WAITING_FOR_APPROVAL)
    ├─ call_order_event(ORDER_UPDATED)
    └─ [有新订单] call_order_event(DRAFT_ORDER_CREATED)
```

---

## 十、核心代码索引

| 功能 | 文件位置 | 函数/类 |
|-----|---------|---------|
| OrderLineInfo 定义 | `saleor/order/fetch.py:44` | `OrderLineInfo` |
| FulfillmentLineData 定义 | `saleor/order/__init__.py:306` | `FulfillmentLineData` |
| RefundData 定义 | `saleor/payment/interface.py:368` | `RefundData` |
| 校验基类 | `saleor/graphql/order/mutations/fulfillment_refund_and_return_product_base.py` | `FulfillmentRefundAndReturnProductBase` |
| 退货 Mutation | `saleor/graphql/order/mutations/fulfillment_return_products.py` | `FulfillmentReturnProducts` |
| 退款 Mutation | `saleor/graphql/order/mutations/fulfillment_refund_products.py` | `FulfillmentRefundProducts` |
| 退货主逻辑 | `saleor/order/actions.py:1864` | `create_fulfillments_for_returned_products` |
| 仅退款主逻辑 | `saleor/order/actions.py:1508` | `create_refund_fulfillment` |
| 退货 Fulfillment 创建 | `saleor/order/actions.py:1763` | `create_return_fulfillment` |
| 行移动+库存分配 | `saleor/order/actions.py:1384` | `_move_order_lines_to_target_fulfillment` |
| 行移动（已发货） | `saleor/order/actions.py:1441` | `_move_fulfillment_lines_to_target_fulfillment` |
| 已退款行退货分离 | `saleor/order/actions.py:1678` | `_move_lines_to_return_fulfillment` |
| 退款金额计算 | `saleor/order/actions.py:1960` | `_calculate_refund_amount` |
| 退款处理 | `saleor/order/actions.py:1993` | `_process_refund` |
| 支付网关退款 | `saleor/payment/gateway.py:391` | `refund` |
| 退款金额二次校验 | `saleor/payment/gateway.py:539` | `_validate_refund_amount` |
| 运费退款计算 | `saleor/order/actions.py:1496` | `__get_shipping_refund_amount` |
| 释放库存分配 | `saleor/warehouse/management.py:359` | `deallocate_stock` |
| 增加库存 | `saleor/warehouse/management.py:456` | `increase_stock` |
| 回滚库存（取消发货） | `saleor/order/utils.py:662` | `restock_fulfillment_lines` |
| AllocationError 定义 | `saleor/core/exceptions.py:52` | `AllocationError` |
| InsufficientStock 定义 | `saleor/core/exceptions.py:44` | `InsufficientStock` |
| can_refund 判定 | `saleor/payment/models.py:446` | `Payment.can_refund` |
| 事件定义 | `saleor/order/__init__.py:80-200` | `OrderEvents` |
| order_returned_event | `saleor/order/events.py:613` | `order_returned_event` |
| fulfillment_refunded_event | `saleor/order/events.py:650` | `fulfillment_refunded_event` |
| payment_refunded_event | `saleor/order/events.py:438` | `payment_refunded_event` |
| Fulfillment 模型 | `saleor/order/models.py:750` | `Fulfillment` |
| FulfillmentStatus | `saleor/order/__init__.py:56` | `FulfillmentStatus` |
| quantity_unfulfilled 属性 | `saleor/order/models.py:746` | `OrderLine.quantity_unfulfilled` |
