# Saleor Order Return 与退款数据链路处理

## 一、整体架构概览

Saleor 的退货与退款系统围绕 **Fulfillment（履约单）** 模型设计，通过不同的 `FulfillmentStatus` 区分业务场景，核心入口为两个 GraphQL Mutation：

| Mutation | 核心函数 | 适用场景 |
|---------|---------|---------|
| `FulfillmentReturnProducts` | `create_fulfillments_for_returned_products` | 退货（支持换货/退款） |
| `FulfillmentRefundProducts` | `create_refund_fulfillment` | 仅退款（不退货） |

核心文件位置：
- 业务逻辑：`saleor/order/actions.py`
- GraphQL 入口：`saleor/graphql/order/mutations/`
- 库存管理：`saleor/warehouse/management.py`
- 模型定义：`saleor/order/models.py`
- 工具函数：`saleor/order/utils.py`

---

## 二、FulfillmentStatus 状态机

`Fulfillment` 模型在 `saleor/order/__init__.py:56-77` 定义了 7 种状态，退货退款相关的核心状态：

```python
class FulfillmentStatus:
    FULFILLED = "fulfilled"           # 已发货
    REFUNDED = "refunded"             # 已退款（不退货）
    RETURNED = "returned"             # 已退货（不退款）
    REFUNDED_AND_RETURNED = "refunded_and_returned"  # 已退货且退款
    REPLACED = "replaced"             # 已换货
    CANCELED = "canceled"             # 已取消
    WAITING_FOR_APPROVAL = "waiting_for_approval"     # 待审批
```

`Fulfillment` 模型关键字段 (`saleor/order/models.py:750-814`)：
```python
class Fulfillment(ModelWithMetadata):
    order = models.ForeignKey(Order, related_name="fulfillments", ...)
    status = models.CharField(max_length=32, default=FulfillmentStatus.FULFILLED, ...)
    tracking_number = models.CharField(max_length=255, ...)
    shipping_refund_amount = models.DecimalField(..., null=True)  # 运费退款
    total_refund_amount = models.DecimalField(..., null=True)    # 商品退款总额
```

---

## 三、退货（Return）流程：反向 Fulfillment 创建

### 3.1 GraphQL 入口

`FulfillmentReturnProducts` Mutation (`saleor/graphql/order/mutations/fulfillment_return_products.py`) 支持三种操作模式：

1. **仅退货**：`refund=False`，创建 `RETURNED` 状态 Fulfillment
2. **退货+退款**：`refund=True`，创建 `REFUNDED_AND_RETURNED` 状态 Fulfillment
3. **换货**：`replace=True`，创建 `REPLACED` 状态 Fulfillment + 新 Draft Order

### 3.2 核心处理逻辑

主函数：`create_fulfillments_for_returned_products` (`saleor/order/actions.py:1864-1957`)

#### 处理流程：

```
输入验证（clean_input）
    ↓
按 replace 标志分组
    ├─ return_order_lines / return_fulfillment_lines
    └─ replace_order_lines / replace_fulfillment_lines
    ↓
[可选] 处理退款（refund=True 时）
    → _process_refund()
    → 调用 payment gateway refund()
    ↓
[可选] 处理换货（replace 非空时）
    → process_replace()
    → 创建 REPLACED 状态 Fulfillment
    → 创建新的 Draft Order（_populate_replace_order_fields）
    ↓
创建退货 Fulfillment
    → create_return_fulfillment()
    → status = REFUNDED_AND_RETURNED / RETURNED
    ↓
移动行到目标 Fulfillment
    → _move_lines_to_return_fulfillment()
    → _move_order_lines_to_target_fulfillment()  # 处理未发货行
    → _move_fulfillment_lines_to_target_fulfillment()  # 处理已发货行
    ↓
清理空 Fulfillment → 触发事件
```

### 3.3 行移动机制

#### 未发货订单行（OrderLine）移动

`_move_order_lines_to_target_fulfillment` (`saleor/order/actions.py:1384-1438`)：

```python
def _move_order_lines_to_target_fulfillment(
    order_lines_to_move: list[OrderLineInfo],
    target_fulfillment: Fulfillment,
    ...
):
    for line_data in order_lines_to_move:
        line_to_move = line_data.line
        quantity_to_move = line_data.quantity
        
        # 计算可移动的未发货数量
        unfulfilled_to_move = min(
            line_to_move.quantity_unfulfilled, quantity_to_move
        )
        line_to_move.quantity_fulfilled += unfulfilled_to_move
        
        # 创建 FulfillmentLine，stock_id=None（未实际出库）
        fulfillment_line = FulfillmentLine(
            fulfillment=target_fulfillment,
            order_line_id=line_to_move.id,
            stock_id=None,
            quantity=unfulfilled_to_move,
        )
        
        # 释放库存分配
        if line_allocations_exists:
            deallocate_stock(lines_to_dellocate, ...)
```

**关键点**：
- 只移动 `quantity_unfulfilled` 范围内的数量
- 增加 `order_line.quantity_fulfilled`（反向逻辑：标记为"已处理"）
- 调用 `deallocate_stock` 释放库存分配

#### 已发货履约行（FulfillmentLine）移动

`_move_fulfillment_lines_to_target_fulfillment` (`saleor/order/actions.py:1441-1493`)：

```python
def _move_fulfillment_lines_to_target_fulfillment(
    fulfillment_lines_to_move: list[FulfillmentLineData],
    ...
):
    for fulfillment_line_data in fulfillment_lines_to_move:
        fulfillment_line = fulfillment_line_data.line
        quantity_to_move = fulfillment_line_data.quantity
        
        # 从原 Fulfillment 扣除数量
        fulfilled_to_move = min(fulfillment_line.quantity, quantity_to_move)
        moved_line.quantity += fulfilled_to_move
        fulfillment_line.quantity -= fulfilled_to_move
        
        # 原行数量为 0 时删除
        if fulfillment_line.quantity == 0:
            empty_fulfillment_lines_to_delete.append(fulfillment_line)
```

**关键点**：
- 保留 `stock_id` 关联（实际出库的仓库）
- 数量在原 Fulfillment 和目标 Fulfillment 间转移
- 处理已退款行的特殊逻辑：已退款行退货时创建 `REFUNDED_AND_RETURNED` 状态

### 3.4 换货（Replace）流程

`process_replace` (`saleor/order/actions.py:1818-1861`) 处理换货逻辑：

1. 创建 `REPLACED` 状态的 Fulfillment，将换货行移入
2. `create_replace_order` 创建新的 Draft Order：
   - 复制原订单的用户、地址、渠道等信息
   - `origin = OrderOrigin.REISSUE`，`original = original_order`
   - 创建新的 OrderLine，数量为换货数量，`quantity_fulfilled=0`
3. 新 Draft Order 可由后续流程处理（确认、付款、发货）

---

## 四、退款（Refund）流程：金额计算

### 4.1 退款入口

退款有两个入口：

1. **退货时退款**：`create_fulfillments_for_returned_products` 中 `refund=True`
2. **仅退款**：`create_refund_fulfillment` (`saleor/order/actions.py:1508-1574`)

### 4.2 退款金额计算

核心函数：`_calculate_refund_amount` (`saleor/order/actions.py:1960-1990`)

```python
def _calculate_refund_amount(
    return_order_lines: list[OrderLineInfo],
    return_fulfillment_lines: list[FulfillmentLineData],
    lines_to_refund: dict[OrderLineIDType, tuple[QuantityType, OrderLine]],
) -> Decimal:
    refund_amount = Decimal(0)
    
    # 1. 计算未发货行的退款
    for line_data in return_order_lines:
        refund_amount += line_data.quantity * line_data.line.unit_price_gross_amount
        lines_to_refund[line_data.line.id] = (line_data.quantity, line_data.line)
    
    # 2. 计算已发货行的退款（跳过已退款的行）
    for line_data in return_fulfillment_lines:
        if line_data.line.fulfillment.status == FulfillmentStatus.REFUNDED:
            continue  # 跳过已退款行
        order_line = order_lines_with_fulfillment[line_data.line.order_line_id]
        refund_amount += line_data.quantity * order_line.unit_price_gross_amount
```

**计算规则**：
- 使用 `unit_price_gross_amount`（含税单价）× 数量
- 已处于 `REFUNDED` 状态的行不重复计算
- 合并同一 OrderLine 的多次退款数量

### 4.3 退款处理流程

`_process_refund` (`saleor/order/actions.py:1993-2056`)：

```python
@transaction_with_commit_on_errors()
def _process_refund(...):
    # 1. 自动计算或使用手动指定金额
    if amount is None:
        amount = _calculate_refund_amount(...)
        if refund_shipping_costs:
            amount += order.shipping_price_gross_amount
    
    # 2. 金额封顶：不超过已收款金额
    if amount and payment:
        amount = min(payment.captured_amount, amount)
    
    # 3. 调用支付网关退款
    if amount and payment:
        from ..payment.gateway import refund
        refund(
            payment,
            manager,
            amount=amount,
            channel_slug=order.channel.slug,
            refund_data=refund_data,
        )
        
        # 4. 更新订单状态，触发事件
        order_refunded(
            order=order,
            amount=amount,
            payment=payment,
            manager=manager,
            trigger_order_updated=False,
        )
    
    return amount
```

### 4.4 支付网关退款

`refund` 函数 (`saleor/payment/gateway.py:391-432`)：

```python
def refund(payment: Payment, manager: "PluginsManager", channel_slug: str,
           amount: Decimal | None = None, refund_data: Optional["RefundData"] = None):
    if amount is None:
        amount = payment.captured_amount
    _validate_refund_amount(payment, amount)
    
    if payment.is_manual():
        # 手动支付：直接创建 REFUND 交易记录
        return create_transaction(
            payment,
            kind=TransactionKind.REFUND,
            payment_information=payment_data,
            is_success=True,
        )
    
    # 调用实际支付网关
    response, error = _fetch_gateway_response(
        manager.refund_payment, payment.gateway, payment_data, ...
    )
```

### 4.5 运费退款

`__get_shipping_refund_amount` (`saleor/order/actions.py:1496-1505`)：

```python
def __get_shipping_refund_amount(
    refund_shipping_costs: bool,
    refund_amount: Decimal | None,
    shipping_price: Decimal,
) -> Decimal | None:
    shipping_refund_amount = None
    if refund_shipping_costs and refund_amount is None:
        shipping_refund_amount = shipping_price
    return shipping_refund_amount
```

**关键点**：
- 只有自动计算退款金额时（`refund_amount is None`）才包含运费
- 手动指定退款金额时，`include_shipping_costs` 参数被忽略

---

## 五、库存回滚机制

### 5.1 库存模型核心概念

`Stock` 模型 (`saleor/warehouse/models.py`) 维护两个关键字段：
- `quantity`：实际库存数量
- `quantity_allocated`：已分配（锁定）但未出库的数量

可用库存 = `quantity` - `quantity_allocated`

### 5.2 未发货订单：释放分配（Deallocate）

`deallocate_stock` (`saleor/warehouse/management.py:359-452`)：

```python
def deallocate_stock(
    order_lines_data: list["OrderLineInfo"],
    site_settings: "SiteSettings",
    requestor: T_REQUESTOR,
):
    # 1. 锁定相关 Allocation 和 Stock 记录
    lines_allocations = allocation_with_stock_qs_select_for_update().filter(
        order_line__in=lines
    )
    
    # 2. 按 OrderLine 分组处理
    for line_info in order_lines_data:
        order_line = line_info.line
        quantity = line_info.quantity
        allocations = line_to_allocations[order_line.pk]
        
        quantity_dealocated = 0
        for allocation in allocations:
            quantity_to_deallocate = min(
                (quantity - quantity_dealocated), 
                allocation.quantity_allocated
            )
            if quantity_to_deallocate > 0:
                # 原子操作减少 allocation 和 stock 的 allocated 数量
                allocation.quantity_allocated = (
                    allocation.quantity_allocated - quantity_to_deallocate
                )
                stock = allocation.stock
                stock.quantity_allocated = (
                    F("quantity_allocated") - quantity_to_deallocate  # 使用 F() 原子更新
                )
                stocks_to_update.append(stock)
                quantity_dealocated += quantity_to_deallocate
    
    # 3. 批量更新
    Allocation.objects.bulk_update(allocations_to_update, ["quantity_allocated"])
    Stock.objects.bulk_update(stocks_to_update, ["quantity_allocated"])
    
    # 4. 触发 back_in_stock 事件
    if available_stock_now > 0 and allocation_before_update.stock_available_quantity <= 0:
        transaction.on_commit(
            lambda: trigger_product_variant_back_in_stock(...)
        )
```

**调用时机**：
- 退货时处理未发货行：`_move_order_lines_to_target_fulfillment` → `deallocate_stock`
- 取消订单：`cancel_order` → `deallocate_stock_for_orders`
- 取消发货：`cancel_fulfillment` → `restock_fulfillment_lines`

### 5.3 已发货订单：库存回滚（Restock）

`restock_fulfillment_lines` (`saleor/order/utils.py:662-674`)：

```python
def restock_fulfillment_lines(fulfillment, warehouse):
    """Return fulfilled products to corresponding stocks."""
    order_lines = []
    for line in fulfillment:
        if line.order_line.variant and line.order_line.variant.track_inventory:
            # 增加实际库存并重新分配
            increase_stock(line.order_line, warehouse, line.quantity, allocate=True)
        order_line = line.order_line
        order_line.quantity_fulfilled -= line.quantity
        order_lines.append(order_line)
    OrderLine.objects.bulk_update(order_lines, ["quantity_fulfilled"])
```

`increase_stock` (`saleor/warehouse/management.py:456-494`)：

```python
@traced_atomic_transaction()
def increase_stock(
    order_line: OrderLine,
    warehouse: Warehouse,
    quantity: int,
    allocate: bool = False,
):
    # 1. 锁定 Stock 记录
    stock = stock_qs_select_for_update().filter(
        warehouse=warehouse, product_variant=order_line.variant
    ).first()
    
    # 2. 增加库存（F() 原子操作）
    if stock:
        stock.increase_stock(quantity, commit=True)  # stock.quantity = F("quantity") + quantity
    else:
        stock = Stock.objects.create(
            warehouse=warehouse, product_variant=order_line.variant, quantity=quantity
        )
    
    # 3. 可选：重新创建分配记录
    if allocate:
        allocation = order_line.allocations.filter(stock=stock).first()
        if allocation:
            allocation.quantity_allocated = F("quantity_allocated") + quantity
            allocation.save(update_fields=["quantity_allocated"])
        else:
            Allocation.objects.create(
                order_line=order_line, stock=stock, quantity_allocated=quantity
            )
        stock.quantity_allocated = F("quantity_allocated") + quantity
        stock.save(update_fields=["quantity_allocated"])
```

**`Stock.increase_stock` 方法** (`saleor/warehouse/models.py:541-545`)：
```python
def increase_stock(self, quantity: int, commit: bool = True):
    """Return given quantity of product to a stock."""
    self.quantity = F("quantity") + quantity  # 原子更新，避免竞态
    if commit:
        self.save(update_fields=["quantity"])
```

**调用时机**：
- 取消发货时：`cancel_fulfillment` → `restock_fulfillment_lines`
- 注意：常规退货（`create_fulfillments_for_returned_products`）**不自动调用** `restock_fulfillment_lines`，库存回滚需要业务方主动处理（如退货商品检验合格后）

### 5.4 并发安全机制

Saleor 通过以下机制保证库存操作的线程安全：

1. **`select_for_update` 行锁**：
   ```python
   def stock_qs_select_for_update() -> QuerySet[Stock]:
       return Stock.objects.order_by("pk").select_for_update(of=["self"])
   ```

2. **`F()` 表达式原子更新**：
   ```python
   stock.quantity = F("quantity") + quantity  # 在数据库层面执行 + 操作
   stock.quantity_allocated = F("quantity_allocated") - quantity_to_deallocate
   ```

3. **`traced_atomic_transaction` 事务包裹**：
   所有库存修改操作都在数据库事务中，确保一致性

---

## 六、完整数据链路图

### 6.1 退货+退款完整链路

```
GraphQL: FulfillmentReturnProducts
    ↓
clean_input()  [fulfillment_return_products.py:121-162]
    ├─ 验证支付状态 & 退款金额
    ├─ 验证订单行/履约行数量
    └─ 包装为 OrderLineInfo / FulfillmentLineData
    ↓
create_fulfillments_for_returned_products()  [actions.py:1864]
    ├─ 按 replace 分组
    │   ├─ return_order_lines
    │   └─ replace_order_lines
    ├─ [refund=True] _process_refund()
    │   ├─ _calculate_refund_amount()
    │   │   └─ Σ(quantity × unit_price_gross_amount)
    │   ├─ [+ 运费] if refund_shipping_costs
    │   ├─ min(amount, payment.captured_amount)
    │   ├─ refund(payment, amount)  [payment/gateway.py]
    │   │   └─ manager.refund_payment()  # 调用支付网关
    │   └─ order_refunded()  # 触发事件 & webhook
    ├─ [replace 非空] process_replace()
    │   ├─ _move_lines_to_replace_fulfillment()  # status=REPLACED
    │   └─ create_replace_order()  # 创建 Draft Order
    ├─ create_return_fulfillment()
    │   └─ _move_lines_to_return_fulfillment()
    │       ├─ _move_order_lines_to_target_fulfillment()
    │       │   ├─ order_line.quantity_fulfilled += quantity
    │       │   └─ deallocate_stock()  # 释放库存分配
    │       └─ _move_fulfillment_lines_to_target_fulfillment()
    │           └─ 数量在 FulfillmentLine 间转移
    ├─ 删除空 Fulfillment
    └─ 触发 ORDER_UPDATED 事件
```

### 6.2 仅退款链路

```
GraphQL: FulfillmentRefundProducts
    ↓
clean_input()  [fulfillment_refund_products.py:99-139]
    ↓
create_refund_fulfillment()  [actions.py:1508]
    ├─ _process_refund()  # 同上
    ├─ 创建 status=REFUNDED 的 Fulfillment
    ├─ _move_order_lines_to_target_fulfillment()
    ├─ _move_fulfillment_lines_to_target_fulfillment()
    └─ 触发 ORDER_UPDATED 事件
```

### 6.3 取消发货链路（含库存回滚）

```
cancel_fulfillment()  [actions.py:807-850]
    ├─ Fulfillment.objects.select_for_update().get(pk=...)
    ├─ [warehouse 非空] restock_fulfillment_lines()
    │   ├─ increase_stock()  # 增加实际库存
    │   │   ├─ stock.quantity = F("quantity") + quantity
    │   │   └─ [allocate=True] 重新创建 Allocation
    │   └─ order_line.quantity_fulfilled -= quantity
    ├─ [warehouse 为空] decrease_fulfilled_quantity()
    ├─ fulfillment.status = CANCELED
    └─ 触发 webhook
```

---

## 七、关键数据结构

### 7.1 OrderLineInfo

```python
class OrderLineInfo(TypedDict):
    line: OrderLine
    quantity: int
    variant: ProductVariant | None
    warehouse_pk: UUID | None
    replace: bool  # 仅退货流程使用
```

### 7.2 FulfillmentLineData

```python
class FulfillmentLineData(TypedDict):
    line: FulfillmentLine
    quantity: int
    replace: bool  # 仅退货流程使用
```

### 7.3 RefundData

```python
class RefundData(TypedDict):
    order_lines_to_refund: list[OrderLineInfo]
    fulfillment_lines_to_refund: list[FulfillmentLineData]
    refund_shipping_costs: bool
    refund_amount_is_automatically_calculated: bool
```

---

## 八、重要设计决策

### 8.1 为什么使用 Fulfillment 作为退货载体？

- 保持数据完整性：所有数量变更都有 Fulfillment 记录可追溯
- 状态机清晰：通过 `status` 字段区分不同业务场景
- 支持部分退货/退款：每个 Fulfillment 可以只包含部分行

### 8.2 为什么退货不自动回滚库存？

- 业务流程差异：退货商品可能需要质检、重新入库等流程
- 灵活性：业务方可以选择何时、是否回滚库存
- 手动控制：通过 `cancel_fulfillment(warehouse=...)` 主动触发回滚

### 8.3 金额计算为什么使用 `unit_price_gross_amount`？

- 退款通常按用户实际支付金额（含税）计算
- 与订单总额 `total_gross_amount` 保持一致
- 税费处理由支付网关负责

---

## 九、核心代码索引

| 功能 | 文件位置 | 函数/类 |
|-----|---------|---------|
| 退货主逻辑 | `saleor/order/actions.py:1864` | `create_fulfillments_for_returned_products` |
| 仅退款主逻辑 | `saleor/order/actions.py:1508` | `create_refund_fulfillment` |
| 退款金额计算 | `saleor/order/actions.py:1960` | `_calculate_refund_amount` |
| 退款处理 | `saleor/order/actions.py:1993` | `_process_refund` |
| 移动未发货行 | `saleor/order/actions.py:1384` | `_move_order_lines_to_target_fulfillment` |
| 移动已发货行 | `saleor/order/actions.py:1441` | `_move_fulfillment_lines_to_target_fulfillment` |
| 释放库存分配 | `saleor/warehouse/management.py:359` | `deallocate_stock` |
| 增加库存 | `saleor/warehouse/management.py:456` | `increase_stock` |
| 回滚库存 | `saleor/order/utils.py:662` | `restock_fulfillment_lines` |
| 取消发货 | `saleor/order/actions.py:807` | `cancel_fulfillment` |
| 退货 GraphQL | `saleor/graphql/order/mutations/fulfillment_return_products.py` | `FulfillmentReturnProducts` |
| 退款 GraphQL | `saleor/graphql/order/mutations/fulfillment_refund_products.py` | `FulfillmentRefundProducts` |
| Fulfillment 模型 | `saleor/order/models.py:750` | `Fulfillment` |
| FulfillmentStatus | `saleor/order/__init__.py:56` | `FulfillmentStatus` |
