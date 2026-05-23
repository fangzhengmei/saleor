# 仓库分配策略与库存预留模型分析

## 一、核心数据模型

### 1.1 库存与分配模型关系

```
Stock (仓库库存)
  ├── quantity: int                    # 总库存量
  ├── quantity_allocated: int          # 已分配量
  └── relations:
      ├── allocations → Allocation     # 订单分配
      └── reservations → Reservation   # 购物车预留

Allocation (订单分配)
  ├── order_line → OrderLine           # 关联订单行
  ├── stock → Stock                    # 关联库存
  └── quantity_allocated: int          # 分配数量

Reservation (购物车预留)
  ├── checkout_line → CheckoutLine     # 关联购物车行
  ├── stock → Stock                    # 关联库存
  ├── quantity_reserved: int           # 预留数量
  └── reserved_until: DateTime         # 过期时间
```

**关键文件**: `saleor/warehouse/models.py:527-689`

### 1.2 可用库存计算公式

```python
# StockQuerySet.annotate_available_quantity() [models.py:330-346]
available_quantity = Stock.quantity - SUM(Allocation.quantity_allocated)

# 计算预留时需额外减去
available_quantity -= SUM(Reservation.quantity_reserved WHERE reserved_until > now)
```

---

## 二、仓库分配策略 (AllocationStrategy)

### 2.1 策略定义

**文件**: `saleor/channel/__init__.py:1-17`

| 策略 | 说明 | 排序方式 |
|------|------|----------|
| `PRIORITIZE_HIGH_STOCK` | 优先从库存最多的仓库分配 | 按 `(quantity - allocated)` 降序 |
| `PRIORITIZE_SORTING_ORDER` | 按渠道内仓库排序顺序分配 | 按 `ChannelWarehouse.sort_order` 升序 |

### 2.2 排序实现

**文件**: `saleor/warehouse/management.py:268-313`

```python
def sort_stocks(allocation_strategy, stocks, channel, ...):
    # 自提订单优先从自提点仓库分配
    if warehouse_id == collection_point_pk:
        return -math.inf  # 最高优先级
    
    strategy_map = {
        PRIORITIZE_HIGH_STOCK: (sort_by_available_qty, True),  # 降序
        PRIORITIZE_SORTING_ORDER: (sort_by_channel_order, False),  # 升序
    }
```

---

## 三、订单生命周期中的写入与释放节奏

### 3.1 完整生命周期时序

```
购物车阶段 (Checkout)
    │
    ├─ 加入商品 → reserve_stocks_and_preorders() [reservations.py:28-89]
    │   └─ 写入: Reservation (带 reserved_until)
    │
    ├─ 库存检查 → check_stock_and_preorder_quantity_bulk() [availability.py:149-197]
    │   └─ 读取: Stock.quantity, Allocation, Reservation
    │
订单创建阶段 (Order Creation)
    │
    ├─ _create_order() [complete_checkout.py:764-909]
    │   ├─ allocate_stocks() [management.py:91-241]
    │   │   ├─ 读取: Stock (select_for_update 行锁)
    │   │   ├─ 写入: Allocation
    │   │   └─ 更新: Stock.quantity_allocated += allocated
    │   ├─ allocate_preorders() [management.py:811-909]
    │   │   └─ 写入: PreorderAllocation
    │   └─ 隐含: Reservation 自动失效 (订单创建后 checkout 被删除)
    │
履约阶段 (Fulfillment)
    │
    ├─ approve_fulfillment() [order/actions.py:863-941]
    │   └─ decrease_stock() [management.py:586-702]
    │       ├─ decrease_allocations()
    │       │   └─ 更新: Allocation.quantity_allocated -= fulfilled
    │       └─ 更新: Stock.quantity -= fulfilled
    │
订单取消/过期阶段 (Cancellation/Expiry)
    │
    ├─ cancel_order() [order/actions.py:421-453]
    │   └─ deallocate_stock_for_orders() [management.py:763-808]
    │       ├─ 更新: Allocation.quantity_allocated = 0
    │       └─ 更新: Stock.quantity_allocated -= deallocated
    │
    └─ 定期清理任务
        ├─ delete_expired_reservations_task() [tasks.py:25-40]
        │   └─ 删除: Reservation WHERE reserved_until < now
        ├─ delete_empty_allocations_task() [tasks.py:14-22]
        │   └─ 删除: Allocation WHERE quantity_allocated = 0
        └─ expire_orders_task() [order/tasks.py:174-179]
            └─ deallocate_stock_for_orders()
```

### 3.2 关键写入点详解

#### 3.2.1 购物车预留写入

**文件**: `saleor/warehouse/reservations.py:91-188`

```python
@traced_atomic_transaction()
def reserve_stocks(checkout_lines, ..., reserved_until, ...):
    # 1. 行锁锁定库存
    stocks = stock_qs_select_for_update().get_variants_stocks(...)
    
    # 2. 计算已有分配和预留
    quantity_allocation_for_stocks = SUM(Allocation)
    quantity_reservation_for_stocks = SUM(Reservation.not_expired())
    
    # 3. 按策略排序仓库
    stocks = sort_stocks(channel.allocation_strategy, ...)
    
    # 4. 多仓分配逻辑
    for line in checkout_lines:
        insufficient_stocks, reserved_items = _create_stock_reservations(
            line, stocks, allocations, reservations, reserved_until
        )
    
    # 5. 原子性写入
    if replace:
        Reservation.objects.filter(checkout_line__in=checkout_lines).delete()
    Reservation.objects.bulk_create(reservations)
```

**多仓遍历逻辑** (`_create_stock_reservations`):
```python
for stock_data in stocks:  # 按策略排序后的仓库列表
    available = stock.quantity - allocated - reserved
    to_reserve = min(remaining, available)
    if to_reserve > 0:
        reservations.append(Reservation(stock_id=stock.pk, quantity_reserved=to_reserve))
        remaining -= to_reserve
        if remaining == 0:
            break  # 已满足，跳出循环

if remaining != 0:
    insufficient_stocks.append(...)  # 库存不足
```

#### 3.2.2 订单分配写入

**文件**: `saleor/warehouse/management.py:91-241`

```python
@traced_atomic_transaction()
def allocate_stocks(order_lines_info, ...):
    # 1. 行锁锁定库存和分配
    stocks = stock_select_for_update_for_existing_qs(...).filter(...)
    
    # 2. 按策略排序仓库（同预留逻辑）
    stocks = sort_stocks(channel.allocation_strategy, ...)
    
    # 3. 多仓分配（同预留的 _create_stock_reservations 模式）
    for line_info in order_lines_info:
        insufficient_stock, allocation_items = _create_allocations(...)
    
    # 4. 写入分配 + 更新库存已分配字段
    if allocations:
        Allocation.objects.bulk_create(allocations)
        Stock.objects.bulk_update(stocks_to_update, ["quantity_allocated"])
```

#### 3.2.3 履约扣减

**文件**: `saleor/warehouse/management.py:586-702`

```python
def decrease_stock(order_lines_info, ..., allow_stock_to_be_exceeded=False):
    # 1. 先减少分配
    decrease_allocations(order_lines_info, ...)
    
    # 2. 再扣减实际库存
    for line_info in order_lines_info:
        stock = variant_and_warehouse_to_stock[variant.pk][warehouse_pk]
        if stock is None and not allow_stock_to_be_exceeded:
            insufficient_stocks.append(...)  # 兜底：库存不存在
            continue
        
        is_exceeded = stock.quantity - allocated < line_info.quantity
        if is_exceeded and not allow_stock_to_be_exceeded:
            insufficient_stocks.append(...)  # 兜底：库存不足
            continue
        
        stock.quantity -= line_info.quantity  # 实际扣减
```

### 3.3 关键释放点

| 触发时机 | 操作 | 调用点 |
|---------|------|--------|
| 订单取消 | 释放 Allocation | `cancel_order()` → `deallocate_stock_for_orders()` |
| 订单过期 | 释放 Allocation | `expire_orders_task()` → `deallocate_stock_for_orders()` |
| 部分履约 | 减少 Allocation | `decrease_stock()` → `decrease_allocations()` |
| 购物车过期 | 删除 Reservation | `delete_expired_reservations_task()` |
| 订单创建 | 隐式删除 Reservation | Checkout 删除 → 级联删除 Reservation |
| 分配为0 | 清理空分配 | `delete_empty_allocations_task()` |

---

## 四、多仓分配与缺货分裂兜底分支

### 4.1 多仓分配核心逻辑

**分配算法** (`_create_allocations`, `_create_stock_reservations`):

```python
# 输入：按策略排序的仓库库存列表
# 算法：贪心算法，逐个仓库分配直到满足需求
def _create_allocations(line_info, stocks, allocations, reservations, ...):
    quantity = line_info.quantity
    allocated = 0
    result = []
    
    for stock_data in stocks:  # stocks 已按 allocation_strategy 排序
        # 计算该仓库可用量
        available = stock_data.quantity 
        available -= allocations.get(stock_data.pk, 0)  # 已分配
        available -= reservations.get(stock_data.pk, 0)  # 已预留
        available = max(available, 0)
        
        # 分配可满足的部分
        to_allocate = min(quantity - allocated, available)
        if to_allocate > 0:
            result.append(Allocation(
                order_line=line_info.line,
                stock_id=stock_data.pk,
                quantity_allocated=to_allocate
            ))
            allocated += to_allocate
            
            if allocated == quantity:
                return insufficient_stock, result  # 已满足，提前返回
    
    # 遍历完所有仓库仍不满足
    if allocated != quantity:
        insufficient_stock.append(InsufficientStockData(
            variant=line_info.variant,
            available_quantity=0  # 注意：这里不返回实际可用量
        ))
        return insufficient_stock, []  # 兜底：整个订单行失败
    
    return [], result
```

**关键特性**:
1. **跨仓分裂**: 同一订单行可分配到多个仓库（每个仓库一条 Allocation 记录）
2. **按策略排序**: 严格按照 `AllocationStrategy` 决定仓库优先级
3. **全部失败兜底**: 只要有一个商品无法完全满足，整个分配失败并抛出异常

### 4.2 缺货分裂兜底分支

#### 4.2.1 库存检查兜底

**文件**: `saleor/warehouse/availability.py:103-146`

```python
def check_stock_quantity(variant, country_code, channel_slug, quantity, ...):
    if variant.track_inventory:
        stocks = Stock.objects.get_variant_stocks(...)
        if not stocks:
            # 兜底分支1：该商品在任何可用仓库都没有库存
            raise InsufficientStock([
                InsufficientStockData(variant=variant, available_quantity=0)
            ])
        
        available_quantity = _get_available_quantity(stocks, ...)
        if quantity > available_quantity:
            # 兜底分支2：总可用库存不足
            raise InsufficientStock([
                InsufficientStockData(variant=variant, available_quantity=0)
            ])
```

#### 4.2.2 超卖允许机制 (`allow_stock_to_be_exceeded`)

**文件**: `saleor/warehouse/management.py:651-702`

```python
def _decrease_stocks_quantity(..., allow_stock_to_be_exceeded=False):
    for line_info in order_lines_info:
        stock = variant_and_warehouse_to_stock.get(variant.pk, {}).get(warehouse_pk)
        
        if stock is None:
            # 兜底分支A：指定仓库无此商品库存
            if not allow_stock_to_be_exceeded:
                insufficient_stocks.append(...)  # 正常：抛错
                continue
            # 开启超卖：静默跳过，不扣减库存（但订单仍标记为已履约）
        
        is_exceeded = stock.quantity - quantity_allocated < line_info.quantity
        if is_exceeded:
            # 兜底分支B：库存不足
            if not allow_stock_to_be_exceeded:
                insufficient_stocks.append(...)  # 正常：抛错
                continue
            # 开启超卖：允许扣减为负数
        
        stock.quantity -= line_info.quantity  # 可能变为负数
```

#### 4.2.3 分配失败兜底

**文件**: `saleor/warehouse/management.py:568-583`

```python
def decrease_allocations(lines_info, site_settings, requestor):
    try:
        deallocate_stock(lines_info, site_settings, requestor)
    except AllocationError as exc:
        # 兜底：deallocate_stock 失败时强制清零
        Allocation.objects.order_by("stock_id").filter(
            order_line__in=exc.order_lines
        ).update(quantity_allocated=0)
```

#### 4.2.4 预购分配兜底

**文件**: `saleor/warehouse/management.py:1040-1077`

```python
def _get_stock_for_preorder_allocation(...):
    # 预购商品激活时需要转换为普通库存分配
    warehouse = Warehouse.objects.filter(
        shipping_zones__id=order.shipping_method.shipping_zone_id
    ).first()
    
    if not warehouse:
        # 兜底分支：没有匹配的仓库时抛出错误
        raise PreorderAllocationError(preorder_allocation.order_line)
    
    # 返回现有库存或创建新的 Stock 实例（quantity=0）
    return existing_stock or Stock(warehouse=warehouse, product_variant=variant, quantity=0)
```

### 4.3 并发安全机制

**文件**: `saleor/warehouse/lock_objects.py`

```python
# 所有库存操作前都使用 select_for_update 行锁
def stock_qs_select_for_update() -> QuerySet[Stock]:
    return Stock.objects.order_by("pk").select_for_update(of=["self"])

def allocation_with_stock_qs_select_for_update() -> QuerySet[Allocation]:
    return Allocation.objects.order_by("stock_id").select_for_update(of=["self"])
```

**操作模式**:
1. 所有写入操作使用 `@traced_atomic_transaction()` 装饰器
2. 先按主键排序获取行锁（避免死锁）
3. 执行计算和写入
4. 事务提交时释放锁

---

## 五、关键调用链汇总

### 5.1 购物车 → 订单 转换链

```
checkout/complete_checkout.py:_create_order()
    ├─ check_stock_and_preorder_quantity_bulk()  [availability.py]
    ├─ allocate_stocks()                          [management.py]
    │   └─ _create_allocations()                  [management.py]
    └─ allocate_preorders()                       [management.py]
        └─ _create_preorder_allocation()          [management.py]
```

### 5.2 订单履约链

```
order/actions.py:approve_fulfillment()
    └─ warehouse/management.py:decrease_stock()
        ├─ decrease_allocations()
        │   └─ deallocate_stock()
        └─ _decrease_stocks_quantity()
```

### 5.3 订单取消链

```
order/actions.py:cancel_order()
    └─ warehouse/management.py:deallocate_stock_for_orders()
        └─ _reduce_quantity_allocated_for_stocks()
```

---

## 六、异常处理与错误码

| 异常 | 触发场景 | 处理方式 |
|------|---------|----------|
| `InsufficientStock` | 库存不足/无库存 | 前端展示缺货，阻止订单创建 |
| `AllocationError` | 解除分配失败 | 强制清零分配量 |
| `PreorderAllocationError` | 预购转普通库存无可用仓库 | 阻止预购激活 |

---

## 七、定时任务维护

| 任务 | 频率 | 作用 |
|------|------|------|
| `delete_expired_reservations_task` | 定期 | 清理过期购物车预留 |
| `delete_empty_allocations_task` | 定期 | 清理零分配量记录 |
| `update_stocks_quantity_allocated_task` | 定期 | 修正 Stock.quantity_allocated 与实际分配的不一致 |
| `expire_orders_task` | 定期 | 过期未支付订单并释放库存 |
