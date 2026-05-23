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

---

## 八、关键遗漏深度分析

### 8.1 Checkout 分支下预留记录的释放时机

#### 8.1.1 级联删除机制

**模型定义** (`saleor/warehouse/models.py:665-670`):
```python
class Reservation(models.Model):
    checkout_line = models.ForeignKey(
        CheckoutLine,
        on_delete=models.CASCADE,  # 级联删除
        related_name="reservations",
    )
```

当 `CheckoutLine` 被删除时，关联的 `Reservation` 会自动级联删除。

#### 8.1.2 条件化释放路径

**存在两条调用路径，释放行为取决于入口函数**：

| 入口函数 | delete_checkout 参数 | Reservation 释放行为 |
|---------|---------------------|---------------------|
| `complete_checkout_post_payment_part()` | 硬编码无条件删除 | ✅ 订单创建成功后立即释放 |
| `create_order_from_checkout()` | 可选参数（默认 True） | ✅/❌ 取决于参数 |
| `complete_checkout_with_transaction()` | 硬编码 `delete_checkout=True` | ✅ 始终释放 |

**路径 1: complete_checkout_post_payment_part（无条件释放）**

**调用链** (`saleor/checkout/complete_checkout.py:1138-1187`):
```
complete_checkout_post_payment_part()
    │
    ├─ _create_order()  # 在事务内创建订单
    │   │
    │   └─ _handle_allocations_of_order_lines()  [line 1265-1295]
    │       └─ allocate_stocks(
    │           check_reservations=True,
    │           checkout_lines=[line.line for line in checkout_lines]
    │       )
    │           │
    │           ├─ _prepare_stock_to_reserved_quantity_map()
    │           │   └─ Reservation.objects.filter(...)
    │           │       .not_expired()
    │           │       .exclude_checkout_lines(checkout_lines)  # 逻辑排除自身
    │           │
    │           └─ _create_allocations()  # 创建 Allocation，此时 Reservation 仍存在
    │
    └─ ✅ 订单创建成功后（无条件）
        └─ delete_checkouts([checkout_info.checkout.pk])  [line 1186]
            │
            └─ checkout/utils.py:148-162 delete_checkouts()
                ├─ 先删除 CheckoutLine (CASCADE → Reservation 被删除)
                └─ 再删除 Checkout
```

**路径 2: create_order_from_checkout（条件化释放）**

**调用链** (`saleor/checkout/complete_checkout.py:1626-1667`):
```
create_order_from_checkout(..., delete_checkout=True)
    │
    ├─ _create_order_from_checkout()  # 创建订单和分配
    │
    └─ 🔀 条件分支 [line 1653]
        │
        ├─ if delete_checkout:
        │   └─ delete_checkouts([checkout_pk])  # ✅ 释放 Reservation
        │
        └─ else:
            ├─ 不删除 Checkout
            └─ ❌ Reservation 仍保留（直到过期或 Checkout 被删除）
```

**调用方参数传递**:
- `complete_checkout_with_transaction()` [line 1799]: 硬编码 `delete_checkout=True`
- 其他调用方需显式传递 `delete_checkout=False` 才会保留

#### 8.1.3 关键设计细节

| 时间点 | 状态 | 说明 |
|--------|------|------|
| `allocate_stocks` 调用时 | Reservation 存在 | 用于计算其他 checkout 的预留占用 |
| `exclude_checkout_lines()` | 逻辑排除 | 自身的预留不计入占用（即将被删除） |
| `delete_checkouts()` 调用 | 物理删除 | CheckoutLine 删除 → Reservation 级联删除 |

**设计意图**: 在分配计算时排除自身预留，确保"预留→分配"的平滑过渡，不会因为自身预留导致"自己占了自己的库存"。

#### 8.1.4 边界情况：`delete_checkout=False`

当 `delete_checkout=False` 时：
- Checkout 保留，Reservation 也保留
- 可能导致"预留"和"分配"同时存在（双重占用）
- 依赖 `exclude_checkout_lines()` 逻辑避免自身冲突
- 需确保后续有其他机制释放 Reservation（如定时任务 `delete_expired_reservations_task`）

---

### 8.2 check_reservations 与库存范围开关对多仓分配的影响

#### 8.2.1 check_reservations 参数

**参数传递** (`saleor/checkout/complete_checkout.py:1278-1288`):
```python
allocate_stocks(
    ...
    check_reservations=True,  # 🔑 始终为 True
    checkout_lines=[line.line for line in checkout_lines],
)
```

**在 `_prepare_stock_to_reserved_quantity_map` 中的作用** (`saleor/warehouse/management.py:243-265`):

```python
if check_reservations:
    # 计算其他 checkout 的预留占用
    quantity_reservation = (
        Reservation.objects.filter(stock_id__in=stocks_id)
        .not_expired()
        .exclude_checkout_lines(checkout_lines or [])  # 排除自身
        .values("stock")
        .annotate(quantity_reserved=Sum("quantity_reserved"))
    )
    # 结果存入 quantity_reservation_for_stocks
```

**对可用量计算的影响** (`_create_allocations`):
```python
available = stock_data.quantity 
available -= quantity_allocation_for_stocks.get(stock_data.pk, 0)  # 已分配
if check_reservations:
    available -= quantity_reservation_for_stocks.get(stock_data.pk, 0)  # 🔑 其他预留
available = max(available, 0)
```

| check_reservations | 可用量公式 | 适用场景 |
|-------------------|-----------|---------|
| `True` | `quantity - allocation - other_reservations` | 订单创建时（严格防止超卖） |
| `False` | `quantity - allocation` | 后台手动操作、履约扣减 |

#### 8.2.2 库存范围开关 `calculate_stocks_with_shipping_zones`

**参数传递** (`saleor/warehouse/management.py:138-143`):
```python
stocks = Stock.objects.for_channel_or_country(
    channel_slug,
    country_code,
    include_shipping_zones=calculate_stocks_with_shipping_zones,  # 🔑
)
```

**控制库存查询范围**:
- `True`: 考虑配送区域关联的仓库 → 更多候选仓库
- `False`: 只考虑渠道直接关联的仓库 → 更少候选仓库

#### 8.2.3 协同影响多仓分配结果

```
┌─────────────────────────────────────────────────────────────┐
│  多仓分配结果 = f(仓库范围 × 各仓库可用量)                   │
│                                                             │
│  仓库范围 = for_channel_or_country(include_shipping_zones)   │
│  各仓库可用量 = quantity - allocation - (reservation if check_reservations) │
└─────────────────────────────────────────────────────────────┘
```

**场景对比**:

| 场景 | calculate_stocks_with_shipping_zones | check_reservations | 分配结果 | 额外风险 |
|------|-------------------------------------|--------------------|---------|---------|
| 订单创建 | True | True | 最严格：仓库最多 + 可用量最少 | 🔴 触发生成器 bug，可用量被高估 |
| 后台补货 | False | False | 最宽松：仓库最少 + 可用量最多 | ✅ bug 被隐藏，恰好正常工作 |
| 自提订单 | N/A（走 click_and_collect 分支） | True | 只考虑自提点仓库 | 🔴 触发生成器 bug |

**关键交互**: `check_reservations=True` 不仅控制可用量计算公式，还决定了生成器是否被提前消费，从而触发 bug。

---

### 8.3 allocate_stocks 中 stocks_id 迭代器重复消费问题

#### 8.3.1 问题代码

**位置**: `saleor/warehouse/management.py:150-163`

```python
# 🔴 BUG: 使用生成器表达式 (generator expression)
stocks_id = (stock.pop("id") for stock in stocks)  # line 150

# 第一次消费（条件触发）：查询 Reservation
quantity_reservation_for_stocks: dict = _prepare_stock_to_reserved_quantity_map(
    checkout_lines, check_reservations, stocks_id  # line 153
)

# 第二次消费：查询 Allocation
quantity_allocation_list = list(
    Allocation.objects.filter(
        stock_id__in=stocks_id,  # line 158 - 🔴 生成器可能已耗尽！
        quantity_allocated__gt=0,
    )
    .values("stock")
    .annotate(quantity_allocated_sum=Sum("quantity_allocated"))
)
```

#### 8.3.2 根本原因

Python 中**生成器表达式 `(...)` 只能被迭代一次**，且存在副作用 `pop("id")` 会修改原始 `stocks` 列表。

**Django ORM `__in=[]` 的真实语义**:
- ⚠️ **之前的结论错误**：`stock_id__in=[]` **不是**全表扫描
- ✅ **正确行为**：Django 将 `__in=[]` 翻译为 SQL `WHERE stock_id IN ()`，返回**空结果集**（0 条记录）
- 验证：`Allocation.objects.filter(stock_id__in=[]).count()` → 0

#### 8.3.3 分场景实际后果

**场景 A: `check_reservations=True`（订单创建时的默认行为）**

```
第一次消费发生在 _prepare_stock_to_reserved_quantity_map:
  → stock.pop("id") 执行，stocks 变成 [{'pk':1,...}, ...]（无 id 字段）
  → 生成器耗尽

第二次消费在 Allocation 查询:
  → stocks_id 生成器返回 []
  → Allocation.objects.filter(stock_id__in=[])
  → 返回空结果集
  → quantity_allocation_for_stocks = {} （空字典）
```

| 阶段 | 预期行为 | 实际行为 | 影响 |
|------|---------|---------|------|
| Reservation 查询 | `stock_id__in=[1,2,3]` | ✅ 正确 | 第一次消费正常 |
| Allocation 查询 | `stock_id__in=[1,2,3]` | `stock_id__in=[]` → 返回 0 条 | 🔴 **严重** |

**具体危害（check_reservations=True 时）**:
1. **可用量被严重高估**：`quantity_allocation_for_stocks` 为空字典，所有已分配量都被忽略
2. **超卖风险**：`available = quantity - 0 = quantity`，可能分配超过实际可用库存
3. **静默的一致性破坏**：不抛异常但数据错误，事后很难排查

**场景 B: `check_reservations=False`（后台操作场景）**

```
_prepare_stock_to_reserved_quantity_map 直接返回 {}
  → 生成器未被消费！

第一次消费发生在 Allocation 查询:
  → stocks_id 生成器返回 [1,2,3]
  → Allocation.objects.filter(stock_id__in=[1,2,3])
  → ✅ 正确查询
```

| 阶段 | 预期行为 | 实际行为 | 影响 |
|------|---------|---------|------|
| Reservation 查询 | 不执行 | ✅ 正确 | check_reservations=False |
| Allocation 查询 | `stock_id__in=[1,2,3]` | ✅ 正确 | 生成器未被提前消费 |

**结论**：当 `check_reservations=False` 时，bug 被**隐藏**，代码恰好工作正常。

#### 8.3.4 Django `__in` 行为验证

```python
# 测试 Django ORM 对 __in=[] 的处理
Allocation.objects.filter(stock_id__in=[1, 2, 3]).count()  # N 条
Allocation.objects.filter(stock_id__in=[]).count()          # 0 条（不是全表！）

# 生成器副作用验证
stocks = [{'id': 1, 'name': 'A'}, {'id': 2, 'name': 'B'}]
gen = (s.pop('id') for s in stocks)

list(gen)  # 第一次: [1, 2]
stocks     # 变成: [{'name': 'A'}, {'name': 'B'}]
list(gen)  # 第二次: [] （已耗尽）
```

#### 8.3.5 修复方案

将生成器改为**列表推导式**:

```python
# 🟢 FIX: 使用列表推导式 (list comprehension)
stocks_id = [stock.pop("id") for stock in stocks]  # 用 [] 代替 ()
```

#### 8.3.6 同类问题排查

**`reservations.py` 校验结果**：

**位置**: `saleor/warehouse/reservations.py:123`
```python
stocks_id = [stock.pop("id") for stock in stocks]  # ✅ 列表推导式，没有问题！
```

✅ **结论**：`reservations.py` 中 `reserve_stocks()` 函数**不存在**同类问题，使用的是列表推导式。

#### 8.3.7 影响范围总结

| 函数 | 文件 | 是否有 bug | 影响场景 |
|------|------|-----------|---------|
| `allocate_stocks()` | `management.py:150` | 🔴 是 | `check_reservations=True` 时超卖 |
| `reserve_stocks()` | `reservations.py:123` | ✅ 否 | 无 |

**高危调用链**：
```
_handle_allocations_of_order_lines()  [complete_checkout.py:1278]
    └─ allocate_stocks(check_reservations=True)
        └─ quantity_allocation_for_stocks = {} （空！）
            └─ available = stock.quantity （忽略已分配量！）
```

---

## 九、支付后分支与草稿单转正链路深度分析

### 9.1 action_required 与退款门槛对订单创建及预留释放的影响

#### 9.1.1 支付后分支控制逻辑

**核心控制条件** (`saleor/checkout/complete_checkout.py:1171`):
```python
if not action_required and not _is_refund_ongoing(payment):
    order = _create_order(...)
    delete_checkouts([checkout_info.checkout.pk])
```

两个门槛条件**任一满足**都会阻止订单创建：

| 条件 | 判定逻辑 | 来源 |
|------|---------|------|
| `action_required` | `txn.action_required` | 支付网关返回（如 3DS 验证） |
| `_is_refund_ongoing(payment)` | 存在 `REFUND_ONGOING` 且成功的交易 | 支付退款中 |

#### 9.1.2 完整分支流程图

```
complete_checkout_post_payment_part()
    │
    ├─ 检查 action_required
    │   ├─ True: 释放 voucher 占用（lines 1162-1168）
    │   │   └─ _release_checkout_voucher_usage()
    │   └─ False: 继续
    │
    ├─ 检查 _is_refund_ongoing(payment)
    │   ├─ True: 跳过订单创建
    │   └─ False: 继续
    │
    ├─ 🔀 分支判断
    │   │
    │   ├─ ✅ 两个条件都不满足：创建订单路径
    │   │   ├─ _create_order() → allocate_stocks()
    │   │   ├─ delete_checkouts() → CASCADE 删除 Reservation
    │   │   └─ 返回 (order, action_required, action_data)
    │   │
    │   └─ ❌ 任一条件满足：跳过订单创建
    │       ├─ order = None
    │       ├─ Checkout 保留 → Reservation 保留
    │       └─ 返回 (None, action_required, action_data)
```

#### 9.1.3 Reservation 释放的条件化结论

| 场景 | action_required | refund_ongoing | 订单创建 | Checkout 删除 | Reservation 释放 |
|------|----------------|----------------|---------|--------------|-----------------|
| 正常支付成功 | False | False | ✅ 是 | ✅ 是 | ✅ CASCADE 删除 |
| 需 3DS 验证 | True | False | ❌ 否 | ❌ 否 | ❌ **保留** |
| 退款中 | False | True | ❌ 否 | ❌ 否 | ❌ **保留** |
| 两者都有 | True | True | ❌ 否 | ❌ 否 | ❌ **保留** |

**关键结论**：
- ✅ **无条件释放仅存在于正常支付成功场景**（`complete_checkout_post_payment_part` 硬编码删除）
- ❌ **action_required 和退款中场景，Reservation 不会被释放**，依赖定时任务 `delete_expired_reservations_task` 过期清理
- ⚠️ **voucher 在 action_required 时会被释放**（line 1162-1168），但 Reservation 不释放，存在不一致性

#### 9.1.4 失败处理分支的预留行为

**`_complete_checkout_fail_handler`** (`saleor/checkout/complete_checkout.py:1996-2033`):
```python
def _complete_checkout_fail_handler(checkout_info, manager, *, voucher_code=None, voucher=None, payment=None):
    # 1. 释放 checkout processing indicator
    # 2. 释放 voucher usage
    # 3. 退款或取消支付
    # ⚠️ 注意：没有释放 Reservation！
```

**失败场景下的 Reservation 状态**：
- `InsufficientStock` 异常：Checkout 保留 → Reservation 保留
- `GiftCardNotApplicable` 异常：Checkout 保留 → Reservation 保留
- 其他异常：Checkout 保留 → Reservation 保留

**设计意图**：失败时保留购物车，用户可调整后重试，无需重新预留。

---

### 9.2 draft_order_complete 调用差异分析

#### 9.2.1 两条转正链路对比

| 维度 | 正常结账链路 `_handle_allocations_of_order_lines` | 草稿单转正链路 `DraftOrderComplete.perform_mutation` |
|------|------------------------------------------------|---------------------------------------------------|
| **位置** | `complete_checkout.py:1265-1295` | `draft_order_complete.py:201-234` |
| **调用方式** | 批量调用（所有订单行一次调用） | **逐行调用**（每行单独事务） |
| `check_reservations` | 硬编码 `True` | `is_reservation_enabled(site_settings)`（配置驱动） |
| `checkout_lines` | 传入，用于 `exclude_checkout_lines` | ❌ **不传入**（草稿单无 Checkout） |
| `calculate_stocks_with_shipping_zones` | 传入 | ✅ 同样传入，来自 `site_settings` |
| `additional_filter_lookup` | 传入（配送方式过滤） | ❌ 不传入 |
| `collection_point_pk` | 传入（自提点过滤） | ❌ 不传入 |

#### 9.2.2 逐行调用 vs 批量调用的关键差异

**正常结账（批量）**:
```python
# complete_checkout.py:1278-1289
allocate_stocks(
    order_lines_info,  # 所有行一次性传入
    ...
    check_reservations=True,
    checkout_lines=[line.line for line in checkout_lines],
)
```

**草稿单转正（逐行）**:
```python
# draft_order_complete.py:212-234
for line in lines:
    if line.variant.track_inventory or line.variant.is_preorder_active():
        line_data = OrderLineInfo(...)
        order_lines_info.append(line_data)
        try:
            with traced_atomic_transaction():  # 每行单独事务
                allocate_stocks(
                    [line_data],  # 🔴 只传一行
                    ...
                    check_reservations=is_reservation_enabled(site_settings),
                    # ❌ 没有 checkout_lines 参数
                )
                allocate_preorders(...)
        except InsufficientStock as e:
            # 逐行失败逐行抛错
```

**逐行调用的后果**（⚠️ 含修正结论）:

1. **✅ 跨调用复用问题不存在，但单次调用内部 bug 仍然存在**：
   - 每次循环都会重新查询 stocks，创建新的 `stocks_id` 生成器，所以**跨调用复用问题不存在**
   - 但是！**单次 `allocate_stocks` 调用内部的生成器两次消费问题仍然存在**
   - 如果 `check_reservations=True`（预留功能开启），**每一行的分配都会独立触发 bug**，导致 `quantity_allocation_for_stocks = {}`，可用量被高估
   - 如果 `check_reservations=False`（预留功能关闭），bug 被隐藏（生成器只在 Allocation 查询时消费一次）

2. **性能较差**：N 行商品 → N 次事务 + N 次库存查询 + N 次分配写入

3. **❌ 修正：部分成功风险不存在，任何失败全量回滚**：
   - 之前结论错误："前几行分配成功，某一行失败时已成功的分配不会回滚"
   - ✅ **正确结论**：任何一行分配失败，**整个外层事务都会回滚**，所有已分配的都会被撤销
   - **原因分析**：
     ```python
     try:
         with traced_atomic_transaction():  # 内层事务
             allocate_stocks(...)
     except InsufficientStock as e:
         raise ValidationError(...) from e  # 🔴 重新抛出异常！
     ```
     a. 内层 `InsufficientStock` 异常被捕获后，**重新抛出** `ValidationError`
     b. 新异常没有被外层捕获，传播到最外层的 `traced_atomic_transaction()`（line 153）
     c. 外层事务捕获异常后执行回滚，所有已分配的 Allocation 和 Stock 更新都会被撤销
     d. 另外，Django 嵌套事务特性：内层 atomic 抛出异常会将整个事务标记为 rollback-only，即使捕获也无法提交

4. **可用量计算偏差**：前一行分配成功会扣减 `Stock.quantity_allocated`，后一行查询时在同一事务内能看到扣减后的值，与批量计算（基于查询开始时的快照）结果可能不一致

---

#### 9.2.2.1 嵌套事务回滚边界深度分析

**事务结构** (`draft_order_complete.py:153-234`):
```
外层事务（line 153）: traced_atomic_transaction()
    ├─ order.save()  # 更新订单状态
    ├─ handle_order_voucher()  # 处理优惠券
    └─ for line in lines:
           try:
               内层事务（line 213）: traced_atomic_transaction()
                   ├─ allocate_stocks()  # 写入 Allocation + 更新 Stock
                   └─ allocate_preorders()
           except InsufficientStock:
               raise ValidationError(...)  # 🔴 异常向外传播
```

**回滚边界矩阵**:

| 异常位置 | 异常是否被捕获 | 内层事务 | 外层事务 | 已成功的分配 |
|---------|---------------|---------|---------|-------------|
| 内层 allocate_stocks | 是，但重新抛出 | 回滚 | 🔴 **回滚** | ❌ 全部撤销 |
| 内层 allocate_stocks | 是，不重新抛出 | 回滚 | ⚠️ 标记为 rollback-only | ❌ 最终提交失败 |
| 外层 order.save() | 否 | - | 🔴 回滚 | ❌ 全部撤销 |

**关键 Django 事务特性**:
> 当嵌套的 `atomic()` 块中抛出异常时，Django 会将整个事务标记为需要回滚。即使外层捕获了异常，当尝试提交外层事务时，Django 会抛出 `TransactionManagementError`。唯一安全的做法是让异常向外传播，由最外层 `atomic()` 处理回滚。

---

#### 9.2.2.2 逐行调用下的 stocks_id 行为分析

**每次循环的执行流程**:
```python
for line in lines:
    line_data = OrderLineInfo(...)
    allocate_stocks([line_data], check_reservations=...)
        ├─ stocks = Stock.objects.for_channel_or_country(...)  # 全新查询
        ├─ stocks_id = (stock.pop("id") for stock in stocks)   # 全新生成器
        ├─ _prepare_stock_to_reserved_quantity_map(..., stocks_id)  # 第1次消费
        └─ Allocation.objects.filter(stock_id__in=stocks_id, ...)   # 第2次消费
```

**行为对比**:

| 场景 | check_reservations | 第1次消费（Reservation） | 第2次消费（Allocation） | 结果 |
|------|-------------------|-------------------------|------------------------|------|
| 批量调用（正常结账） | True | ✅ 消费 `[1,2,3]` | ❌ 空 `[]` | 🔴 触发 bug，可用量高估 |
| 逐行调用（草稿单） | True | ✅ 消费 `[1]` | ❌ 空 `[]` | 🔴 **每行独立触发 bug** |
| 批量调用（正常结账） | False | 不执行 | ✅ 消费 `[1,2,3]` | ✅ 正常 |
| 逐行调用（草稿单） | False | 不执行 | ✅ 消费 `[1]` | ✅ 正常 |

**⚠️ 关键修正结论**：
- ❌ 之前错误："逐行调用隐藏生成器 bug"
- ✅ **正确结论**：逐行调用不会隐藏 bug，只是将 bug 的影响范围从"整个订单"缩小到"单行"。如果 `check_reservations=True`，**每一行的分配都会独立触发 bug**，导致每一行的可用量计算都忽略已分配量。

#### 9.2.3 check_reservations 参数差异的业务影响

**正常结账** (`check_reservations=True` 硬编码):
```python
available = quantity - allocation - other_reservations
```

**草稿单转正** (`check_reservations=is_reservation_enabled(site_settings)`):
```python
if is_reservation_enabled(site_settings):
    available = quantity - allocation - other_reservations
else:
    available = quantity - allocation  # 不考虑其他 checkout 的预留
```

**`is_reservation_enabled` 判定** (`saleor/warehouse/reservations.py:401-405`):
```python
def is_reservation_enabled(settings) -> bool:
    return bool(
        settings.reserve_stock_duration_authenticated_user
        or settings.reserve_stock_duration_anonymous_user
    )
```

**场景对比**:
- 预留功能开启：两条链路可用量计算一致
- 预留功能关闭：草稿单转正链路可用量更多（不扣除预留），可能与购物车链路的库存检查不一致

#### 9.2.4 库存范围开关的一致性

两边都使用 `site_settings.use_legacy_shipping_zone_stock_availability` 控制 `calculate_stocks_with_shipping_zones`，行为一致：
- `True`: 考虑配送区域关联的仓库（legacy 行为）
- `False`: 只考虑渠道直接关联的仓库

**测试验证**（`test_draft_order_complete.py:709-741`）:
```python
def test_draft_order_complete_channel_with_shipping_zones_excluded_from_stock_calculation():
    # use_legacy_shipping_zone_stock_availability = False
    # 即使 channel.shipping_zones.clear()，也不会报 INSUFFICIENT_STOCK
    # 因为库存范围不考虑 shipping zones
```

---

### 9.3 现有测试梳理与遗漏覆盖点

#### 9.3.1 已覆盖的测试场景

| 测试用例 | 覆盖内容 | 文件位置 |
|---------|---------|---------|
| `test_draft_order_complete` | 基本分配、事件触发 | line 108 |
| `test_draft_order_complete_no_automatically_confirm_all_new_orders` | 订单状态 | line 175 |
| `test_draft_order_complete_by_user_no_channel_access` | 权限控制 | line 219 |
| `test_draft_order_complete_by_app` | App 调用 | line 242 |
| `test_draft_order_complete_with_voucher` | 优惠券处理 | line 272 |
| `test_draft_order_complete_0_total` | 零元订单 | line 413 |
| `test_draft_order_complete_channel_without_shipping_zones_assigned` | 无配送区域时的库存不足 | line 674 |
| `test_draft_order_complete_channel_with_shipping_zones_excluded_from_stock_calculation` | 库存范围开关 | line 709 |
| `test_draft_order_complete_product_without_inventory_tracking` | 无库存跟踪商品 | line 743 |
| `test_draft_order_complete_with_preorder_lines` | 预购商品 | line 820 |

#### 9.3.2 严重遗漏的测试覆盖点（⚠️ 含修正结论）

| 遗漏场景 | 风险等级 | 说明 |
|---------|---------|------|
| 🔴 **`action_required=True` 路径** | P0 | Reservation 保留，voucher 释放的不一致性 |
| 🔴 **`_is_refund_ongoing=True` 路径** | P0 | Reservation 保留，订单不创建 |
| 🔴 **`check_reservations=True` 时逐行 bug** | P0 | 草稿单转正时每行独立触发生成器 bug |
| 🔴 **`check_reservations=True/False` 差异** | P1 | 草稿单转正与正常结账的可用量计算差异 |
| 🟠 **逐行分配失败全量回滚** | P1 | 第 N 行失败，前 N-1 行已分配的全部回滚 |
| 🟠 **多仓分配在草稿单中的行为** | P1 | 逐行调用 vs 批量调用的多仓分配结果差异 |
| 🟠 **草稿单转正时现有 Reservation 冲突** | P1 | 其他 checkout 已预留该商品库存时的处理 |
| 🟡 **`delete_checkout=False` 路径** | P2 | Reservation 与 Allocation 双重占用 |
| 🟡 **失败处理分支的 Reservation 状态** | P2 | `_complete_checkout_fail_handler` 不释放 Reservation |
| 🟡 **嵌套事务回滚边界验证** | P2 | 内层异常重新抛出导致外层全量回滚的行为 |

#### 9.3.3 建议补充的测试用例（⚠️ 含修正）

**P0 级测试**:
```python
def test_complete_checkout_action_required_keeps_reservation():
    # 验证 action_required=True 时 Reservation 保留

def test_complete_checkout_refund_ongoing_keeps_reservation():
    # 验证退款中时 Reservation 保留

def test_draft_order_complete_check_reservations_enabled():
    # 验证 is_reservation_enabled=True 时会扣除其他 checkout 的预留

def test_draft_order_complete_check_reservations_disabled():
    # 验证 is_reservation_enabled=False 时不扣除其他 checkout 的预留

def test_draft_order_complete_generator_bug_per_line():
    # 验证草稿单转正时每行独立触发生成器 bug
    # 准备：已有 allocation 占用部分库存
    # 预期：如果 check_reservations=True，由于 bug 可用量被高估，可能超卖
```

**P1 级测试**:
```python
def test_draft_order_complete_any_failure_full_rollback():
    # 修正：之前的"部分成功不回滚"结论错误
    # 验证：多行分配时，第 N 行失败，前 N-1 行已分配的全部回滚
    # 断言：Allocation.objects.count() == 0

def test_draft_order_complete_multi_warehouse_allocation():
    # 验证草稿单转正的多仓分配结果与正常结账一致

def test_draft_order_complete_with_existing_reservations():
    # 验证其他 checkout 已预留时，草稿单转正的可用量计算正确
```

**P2 级测试**:
```python
def test_create_order_from_checkout_keep_reservation():
    # 验证 delete_checkout=False 时 Reservation 保留

def test_complete_checkout_fail_keeps_reservation():
    # 验证 InsufficientStock 异常时 Reservation 保留

def test_allocate_stocks_generator_bug_triggered_per_line_in_draft_order():
    # 修正：之前的"逐行调用隐藏 bug"结论错误
    # 验证：草稿单转正逐行调用时，每一行都会独立触发生成器 bug
    # 断言：每次 allocate_stocks 调用内部，stocks_id 生成器被消费两次
```

---

### 9.4 修正结论汇总表

| 之前结论 | 修正后结论 | 影响范围 |
|---------|-----------|---------|
| 逐行调用隐藏生成器 bug | 逐行调用每行独立触发 bug | 草稿单转正链路超卖风险 |
| 逐行分配部分成功不回滚 | 任何失败全量回滚 | 事务一致性（实际更安全） |
| stocks_id 是复用问题 | 是单次调用内部两次消费问题 | 所有 `check_reservations=True` 场景 |
