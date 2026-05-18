# Saleor 客户结账全链路分析报告（可复核版）

**文档状态**：可评审  
**分析范围**：客户结账从购物车快照到订单生成与库存扣减的完整链路  
**重点关注**：事务边界、数据快照固化、异常回滚机制  
**事实核对**：已对照源码逐行验证

---

## 1. 核心模块概览

| 模块 | 主要职责 | 关键文件 |
|------|---------|---------|
| Checkout 核心 | 结账流程编排、订单创建 | `saleor/checkout/complete_checkout.py` |
| Checkout 计算 | 价格、税费、运费、折扣计算 | `saleor/checkout/calculations.py` |
| Checkout 模型 | 购物车数据结构 | `saleor/checkout/models.py` |
| Order 核心 | 订单数据模型 | `saleor/order/models.py` |
| Warehouse | 库存分配管理 | `saleor/warehouse/management.py` |
| Payment | 支付网关处理 | `saleor/payment/gateway.py` |
| GraphQL API | 对外接口层 | `saleor/graphql/checkout/mutations/checkout_complete.py` |

---

## 2. 全链路流程详解

### 2.1 第一阶段：请求入口与初步验证

**入口点**：`CheckoutComplete` GraphQL Mutation (`checkout_complete.py:49`)

**执行步骤**：
1. **Checkout 定位与幂等性检查**
   - 验证 checkout ID/token
   - 若 checkout 已不存在，通过 `Order.objects.get_by_checkout_token()` 查找已创建的订单
   - 若找到已创建订单，直接返回成功（幂等性保障）

2. **基础数据验证**
   - 验证 checkout 邮箱格式有效性
   - 拉取 checkout 商品行，检查商品变体在当前渠道是否可用
   - 验证配送地址与账单地址的格式完整性

3. **调用核心处理函数**
   - 调用 `complete_checkout()` 进入核心流程

---

### 2.2 第二阶段：流程分支决策

**核心函数**：`complete_checkout()` (`complete_checkout.py:1699`)

根据支付状态选择两条不同的执行路径：

| 分支条件 | 执行流程 | 适用场景 |
|---------|---------|---------|
| `authorize_status == FULL` OR 存在交易记录 OR 渠道允许未支付订单 OR 订单金额为 0 | `complete_checkout_with_transaction()` | 已完成支付/授权、零金额订单 |
| 其他情况（需处理支付） | `complete_checkout_with_payment()` | 需实时调用支付网关 |

---

### 2.3 第三阶段 A：交易流程（无需实时支付）

**核心函数**：`complete_checkout_with_transaction()` (`complete_checkout.py:1775`)

```
1. 验证 checkout 准备状态（地址、支付状态等）
2. 调用 create_order_from_checkout() 直接创建订单
3. 返回创建的订单
```

---

### 2.4 第三阶段 B：支付流程（需实时支付）

**核心函数**：`complete_checkout_with_payment()` (`complete_checkout.py:1823`)

此流程采用**三事务分段**设计，通过临时库存预留（Reservation）机制，在保证数据一致性的同时避免长时间持有行锁。

> **设计说明**：代码注释第1874-1875行写着 "Process payments out of transaction"，但**实际实现中支付处理完全包裹在 `transaction_with_commit_on_errors()` 事务内**（第1881行）。库存通过临时 Reservation 机制保证不被超卖，而非依赖长事务行锁。

#### 事务 1：锁定 Checkout、准备数据、创建临时库存预留
```
[transaction_with_commit_on_errors]  # L1842
    ↓
select_for_update 锁定 checkout 行
    ↓
设置 completing_started_at 时间戳（标记处理中，防止并发）
    ↓
重新拉取最新 checkout 与商品行数据
    ↓
调用 complete_checkout_pre_payment_part()
    ├─ 验证支付状态有效性
    ├─ 验证配送方式与地址
    ├─ fetch_checkout_data() 计算最新价格
    ├─ check_stock_and_preorder_quantity_bulk() 库存可用性检查
    └─ _get_order_data() 构建订单快照数据
    ↓
_reserve_stocks_without_availability_check() 创建临时库存预留
    ├─ 查询相关 Stock 记录
    ├─ 创建 Reservation 记录，reserved_until = now + RESERVE_DURATION
    └─ 关联 checkout_line_id，后续订单创建时可识别
    ↓
提交事务（checkout 行锁释放）
```

**关键机制**：
- 临时 Reservation 保证了在支付处理期间库存不被其他用户超卖
- 事务提交后 checkout 行锁立即释放，不阻塞其他操作
- Reservation 会在过期时间后自动失效（默认为 15 分钟）

#### 事务 2：支付处理与有效性验证
```
[transaction_with_commit_on_errors]  # L1881
    ↓
select_for_update 重新锁定 checkout
    ↓
select_for_update 锁定 payment 记录
    ↓
调用 _process_payment()
    ├─ gateway.process_payment() 或 gateway.confirm()
    ├─ 支付失败 → _complete_checkout_fail_handler() → 抛出异常
    └─ 返回交易结果 txn
    ↓
payment.refresh_from_db() 刷新支付状态
    ↓
验证支付是否仍处于活跃状态
    ↓
支付已失效 → _complete_checkout_fail_handler() 退款 → 抛出异常
    ↓
提交事务
```

**重要事实**：支付处理**完全在数据库事务内执行**。如果支付网关调用耗时较长，该事务会持有 checkout 和 payment 的行锁。

#### 事务 3：创建订单与清理
```
[transaction_with_commit_on_errors]  # L1920
    ↓
select_for_update 重新锁定 checkout
    ↓
重新拉取 checkout 与商品行数据
    ↓
调用 complete_checkout_post_payment_part()
    ├─ 检查是否需要额外验证（如 3D Secure action_required）
    ├─ 无需额外验证 → 调用 _create_order()
    │   ├─ @traced_atomic_transaction() 嵌套事务
    │   ├─ 检查 Order.objects.filter(checkout_token=...) 幂等性
    │   ├─ 创建 Order 记录
    │   ├─ OrderLine.objects.bulk_create() 批量创建订单行
    │   ├─ OrderLineDiscount.objects.bulk_create() 批量创建折扣
    │   ├─ allocate_stocks() 正式分配库存（独立事务）
    │   │   └─ 失败 → 抛出 InsufficientStock → 外层捕获并回滚
    │   ├─ allocate_preorders() 分配预售库存（独立事务）
    │   │   └─ 失败 → 抛出异常 → 外层捕获并回滚
    │   ├─ add_gift_cards_to_order() 处理礼品卡
    │   ├─ 关联支付记录：checkout.payments.update(order=order)
    │   ├─ 复制 metadata 到 Order
    │   ├─ 更新搜索向量
    │   ├─ order.save()
    │   └─ transaction.on_commit() 注册订单创建通知
    ├─ delete_checkouts([checkout.pk]) 删除购物车
    └─ 清空 checkout.completing_started_at 标记
    ↓
提交事务
```

---

### 2.5 第四阶段：购物车信息快照固化

购物车信息在订单创建时被**完整快照固化**，确保订单数据不随后续商品/价格变动而改变。

#### 2.5.1 地址快照固化

**机制**：`Address.get_copy()` (`account/models.py:122`)
```python
def get_copy(self):
    """Return a new instance of the same address."""
    return Address.objects.create(**self.as_data())
```

**触发时机**：
1. **配送地址** (`_process_shipping_data_for_order`:209, 212)
   - 用户选择保存地址且地址已存在于用户地址簿时
   - 自提订单（有 collection_point）时
   - 创建全新的 Address 记录，与原地址脱离关联

2. **账单地址** (`_process_user_data_for_order`:282)
   - 用户选择保存账单地址且地址已存在时
   - 创建全新的 Address 记录

**数据流向**：
```
Checkout.shipping_address → get_copy() → Order.shipping_address
Checkout.billing_address  → get_copy() → Order.billing_address
```

> 关键点：订单关联的是地址**副本**，后续用户修改地址簿不影响历史订单。

#### 2.5.2 商品行快照固化

**核心函数**：`_create_line_for_order()` (`complete_checkout.py:292`)

每个 `CheckoutLine` 转化为 `OrderLine` 时，以下信息被完整快照：

| 快照字段 | 来源 | 说明 |
|---------|------|-----|
| `product_name` | `str(product)` | 商品名称（快照） |
| `variant_name` | `str(variant)` | 变体名称（快照） |
| `translated_product_name` | ProductTranslation | 翻译后的商品名 |
| `translated_variant_name` | ProductVariantTranslation | 翻译后的变体名 |
| `product_sku` | `variant.sku` | SKU 编码（快照） |
| `product_variant_id` | `variant.get_global_id()` | 变体 GraphQL ID |
| `product_type_id` | `product.product_type_id` | 商品类型 ID（反范式） |
| `is_shipping_required` | `variant.is_shipping_required()` | 是否需要配送 |
| `is_gift_card` | `variant.is_gift_card()` | 是否为礼品卡 |
| `quantity` | `checkout_line.quantity` | 购买数量 |
| `is_gift` | `checkout_line.is_gift` | 是否为赠品 |
| `metadata` | `checkout_line.metadata` | 自定义元数据 |
| `private_metadata` | `checkout_line.private_metadata` | 私有元数据 |

#### 2.5.3 价格快照固化

价格计算在 `_get_order_data()` 阶段完成，以下价格维度被固化：

| 价格字段 | 计算逻辑 | 说明 |
|---------|---------|-----|
| `base_unit_price` | `calculate_base_line_unit_price()` | 基础单价（含促销折扣） |
| `undiscounted_base_unit_price` | `calculate_undiscounted_base_line_unit_price()` | 未折扣基础单价 |
| `undiscounted_base_total_price` | `calculate_undiscounted_base_line_total_price()` | 未折扣总价 |
| `unit_price` | `calculations.checkout_line_unit_price()` | 最终单价（含所有折扣、税费） |
| `total_price` | `calculations.checkout_line_total()` | 最终总价 |
| `undiscounted_unit_price` | `get_taxed_undiscounted_price()` | 未折扣含税单价 |
| `undiscounted_total_price` | `get_taxed_undiscounted_price()` | 未折扣含税总价 |
| `tax_rate` | `calculations.checkout_line_tax_rate()` | 税率 |
| `unit_discount` | `_get_unit_discount()` | 单位折扣金额 |
| `unit_discount_reason` | `_get_unit_discount_reason()` | 折扣原因描述 |

#### 2.5.4 折扣快照固化

**订单级折扣**：`_create_order_discount()` (`complete_checkout.py:1298`)
- 促销折扣：从 `CheckoutDiscount` 复制到 `OrderDiscount`
- 订单级凭证折扣：直接创建 `OrderDiscount` 记录

**行级折扣**：`_create_order_line_discounts()` (`complete_checkout.py:535`)
- 促销折扣：从 `CheckoutLineDiscount` 复制
- 行级凭证折扣：创建 `OrderLineDiscount` 记录

#### 2.5.5 其他信息快照

| 信息类型 | 固化方式 |
|---------|---------|
| 配送方式 | 复制 shipping_method_name、metadata 到 Order |
| 税费配置 | 复制 tax_class 相关字段到 OrderLine |
| 支付记录 | 更新 Payment.checkout_id = None, Payment.order_id = order.id |
| 礼品卡 | 调用 `add_gift_cards_to_order()` 关联礼品卡使用记录 |
| Metadata | 复制 checkout.metadata 和 checkout.private_metadata 到 Order |

---

### 2.6 第五阶段：库存分配

#### 2.6.1 普通库存分配

**核心函数**：`allocate_stocks()` (`warehouse/management.py:91`)

```
@traced_atomic_transaction()  # 完全独立的事务
    ↓
过滤需要追踪库存的商品行（排除不追踪库存、预售商品）
    ↓
stock_select_for_update_for_existing_qs() 按 stock_id 排序锁定库存行
    ↓
查询现有分配量（quantity_allocated）
    ↓
查询现有预留量（Reservation，排除当前 checkout 的预留）
    ↓
按分配策略排序仓库
    ├─ PRIORITIZE_HIGH_STOCK: 优先从库存最多的仓库分配
    └─ PRIORITIZE_SORTING_ORDER: 按渠道配置的仓库顺序分配
    ↓
_create_allocations() 创建 Allocation 记录
    ├─ 遍历每个商品行，从可用库存中分配
    └─ 库存不足 → 收集到 insufficient_stock 列表
    ↓
insufficient_stock 非空 → 抛出 InsufficientStock 异常
    ↓
Allocation.objects.bulk_create(allocations)
    ↓
Stock.objects.bulk_update() 使用 F() 原子更新 quantity_allocated
    ↓
检查库存是否售罄，transaction.on_commit 触发 out_of_stock 事件
```

**事务边界**：`allocate_stocks()` 拥有完全独立的 `@traced_atomic_transaction()`。
- 成功：事务提交，Alloction 记录持久化，Stock.quantity_allocated 更新
- 失败：事务回滚，抛出 `InsufficientStock` 异常，由上层 `complete_checkout_post_payment_part()` 捕获并调用 `_complete_checkout_fail_handler()`

#### 2.6.2 预售库存分配

**核心函数**：`allocate_preorders()` (`warehouse/management.py:811`)

```
@traced_atomic_transaction()  # 完全独立的事务
    ↓
过滤预售商品行
    ↓
select_for_update(of=("self",)) 锁定 ProductVariantChannelListing 行
    ↓
检查渠道级预售阈值（preorder_quantity_threshold）
    ↓
检查全局预售阈值（preorder_global_threshold）
    ↓
创建 PreorderAllocation 记录
```

**事务边界**：同样拥有独立的 `@traced_atomic_transaction()`，失败时向外抛出异常。

---

## 3. 事务边界精确分析

### 3.1 事务装饰器说明

| 装饰器 | 行为 | 用途 |
|--------|------|-----|
| `@traced_atomic_transaction()` | 标准 Django 事务 + 链路追踪，异常自动回滚 | 库存操作、订单创建核心逻辑，**创建独立事务边界** |
| `transaction_with_commit_on_errors()` | 即使内部抛出异常也先提交事务，再向外抛出 | 支付流程的分段提交，确保各阶段数据持久化 |
| `transaction.atomic()` | 标准 Django 原子事务 | 凭证使用量更新等简单操作 |

### 3.2 支付流程事务边界全景图

```
complete_checkout_with_payment() [L1823]
│
├─ [transaction_with_commit_on_errors] # 事务 1: 锁定、准备、预留 [L1842]
│   ├─ select_for_update(checkout)
│   ├─ 设置 completing_started_at
│   ├─ complete_checkout_pre_payment_part()
│   └─ _reserve_stocks_without_availability_check()
│   └─ 事务提交 → Reservation 记录持久化，行锁释放
│
├─ [transaction_with_commit_on_errors] # 事务 2: 支付处理 [L1881]
│   ├─ select_for_update(checkout)
│   ├─ select_for_update(payment)
│   ├─ _process_payment() → 调用外部支付网关
│   └─ 验证支付活跃性
│   └─ 事务提交 → 支付结果持久化
│
└─ [transaction_with_commit_on_errors] # 事务 3: 订单创建 [L1920]
    ├─ select_for_update(checkout)
    └─ complete_checkout_post_payment_part()
        └─ _create_order()
            └─ @traced_atomic_transaction()  # 嵌套事务（保存点） [L764]
                ├─ 幂等性检查：Order.objects.filter(checkout_token=...)
                ├─ Order.objects.create()
                ├─ OrderLine.objects.bulk_create()
                ├─ OrderLineDiscount.objects.bulk_create()
                ├─ allocate_stocks()
                │   └─ @traced_atomic_transaction()  # 独立事务 [L91]
                │       ├─ 锁定 Stock 行
                │       ├─ 检查库存
                │       ├─ 创建 Allocation
                │       ├─ 更新 Stock.quantity_allocated (F())
                │       └─ 事务提交 或 抛出 InsufficientStock
                ├─ allocate_preorders()
                │   └─ @traced_atomic_transaction()  # 独立事务 [L811]
                │       ├─ 锁定 ProductVariantChannelListing
                │       ├─ 检查预售阈值
                │       ├─ 创建 PreorderAllocation
                │       └─ 事务提交 或 抛出异常
                ├─ add_gift_cards_to_order()
                ├─ checkout.payments.update(order=order)
                ├─ 复制 metadata
                ├─ order.save()
                └─ transaction.on_commit() 注册事件
            └─ 嵌套事务提交
        ├─ delete_checkouts([checkout.pk])
        └─ 清空 completing_started_at
    └─ 事务提交
```

### 3.3 关键事务边界结论

| 操作 | 事务归属 | 失败影响 |
|-----|---------|---------|
| 库存预留（Reservation） | 事务1 | 失败回滚事务1，无持久化副作用 |
| 支付处理 | 事务2 | 失败回滚事务2，调用 _complete_checkout_fail_handler() |
| 订单创建 | 事务3 + 嵌套事务 | 失败由外层捕获，调用 _complete_checkout_fail_handler() |
| 库存分配（Allocation） | 独立事务 | 失败抛出 InsufficientStock，外层回滚并补偿 |
| 预售库存分配 | 独立事务 | 失败抛出异常，外层回滚并补偿 |

### 3.4 行级锁定策略

**锁定对象与时机**：

| 表 | 锁定时机 | 锁定方式 |
|----|---------|---------|
| `Checkout` | 进入每个分段事务时 | `select_for_update()` 按 pk 锁定单行 |
| `Payment` | 支付处理前（事务2） | `select_for_update()` 按 pk 锁定 |
| `Stock` | 库存分配时 | `stock_select_for_update_for_existing_qs()` 按 stock_id 排序锁定 |
| `ProductVariantChannelListing` | 预售分配时 | `select_for_update(of=("self",))` |
| `Voucher` / `VoucherCode` | 凭证使用量更新时 | `get_voucher_for_checkout_info(with_lock=True)` |

**死锁预防**：
- 所有批量锁定操作按主键排序
- Stock 表按 `stock_id` 排序锁定
- Allocation 表按 `stock_id` 排序锁定

---

## 4. 异常处理与回滚机制

### 4.1 核心失败处理器

**函数**：`_complete_checkout_fail_handler()` (`complete_checkout.py:1996`)

**执行的补偿操作**：

| 操作 | 条件 | 说明 |
|-----|------|-----|
| 清空 `completing_started_at` | 该字段非空时 | 释放 checkout 处理标记，允许重试 |
| 释放凭证使用量 | 传入 voucher 参数时 | 调用 `_release_checkout_voucher_usage()` |
| 退款/作废支付 | 传入 payment 参数时 | 调用 `gateway.payment_refund_or_void()` |

> **重要**：`_complete_checkout_fail_handler()` 不会回滚已提交的事务。它只执行**补偿操作**，已持久化的数据（如 Reservation、支付交易记录）不会被回滚。

### 4.2 异常场景与回滚行为

| 异常类型 | 触发点 | 捕获位置 | 补偿操作 | 已持久化数据 |
|---------|--------|---------|---------|-------------|
| `InsufficientStock` | 事务1 库存检查 | `complete_checkout_pre_payment_part` [L1122] | 释放凭证、退款（若有支付） | Reservation 回滚（事务未提交） |
| `InsufficientStock` | `allocate_stocks()` | `complete_checkout_post_payment_part` [L1188] | 释放凭证、退款支付 | Reservation 已持久化（会过期），支付已完成，Allocation 回滚 |
| `PaymentError` | `_process_payment()` | `_process_payment` [L1083] | 释放凭证 | Reservation 已持久化，支付回滚（事务未提交） |
| `GiftCardNotApplicable` | `_create_order()` | `complete_checkout_post_payment_part` [L1198] | 释放凭证、退款支付 | Reservation 已持久化，支付已完成 |
| `NotApplicable` | 凭证验证 | 各异常捕获点 | 释放凭证、退款（若有支付） | 视阶段而定 |
| `TaxError` / `TaxDataError` | 税费计算 | `create_order_from_checkout` | 释放凭证 | Reservation 回滚 |
| `ValidationError` | 各类数据验证 | 各异常捕获点 | 释放凭证、退款（若有支付） | 视阶段而定 |

### 4.3 部分失败场景详细分析

#### 场景 1：事务1 失败（库存预留前）
- **原因**：库存检查失败、数据验证失败等
- **回滚**：事务1 回滚，所有变更撤销
- **状态**：checkout.completing_started_at 被 _complete_checkout_fail_handler 清空，可重试
- **残留**：无

#### 场景 2：事务1 成功，事务2 失败（支付失败）
- **原因**：支付网关拒绝、支付超时等
- **回滚**：事务2 回滚，支付不生效
- **状态**：checkout.completing_started_at 被清空，可重试
- **残留**：事务1 提交的 Reservation 记录会在过期时间后自动失效

#### 场景 3：事务1、2 成功，事务3 中 allocate_stocks 失败
- **原因**：支付成功但正式库存分配时发现库存不足（极小概率，因事务1已预留）
- **回滚**：
  - allocate_stocks 独立事务回滚，Allocation 不持久化
  - 异常向外抛出，_complete_checkout_fail_handler 被调用
  - 清空 completing_started_at
  - 释放凭证使用量
  - **调用 gateway.payment_refund_or_void() 退款**
- **状态**：订单未创建，支付已退款，用户可重新下单
- **残留**：Reservation 记录（过期自动失效）、支付交易退款记录

#### 场景 4：订单创建成功但 webhook 通知失败
- **原因**：通知服务不可用
- **处理**：所有通知通过 `transaction.on_commit()` 注册，事务提交后才触发
- **影响**：通知失败不影响订单状态，依赖异步重试机制
- **残留**：订单已成功创建

### 4.4 补偿机制的局限性

**Reservation 不会被主动清理**：
- 失败处理器不会删除已创建的 Reservation 记录
- 依赖 `reserved_until` 字段的自动过期机制
- 过期的 Reservation 不会影响库存可用性

**支付退款不是原子操作**：
- 退款是调用外部支付网关的 API，可能失败
- 若退款失败，需要人工介入处理
- 系统不保证退款一定成功

---

## 5. 并发控制机制

### 5.1 乐观锁：Checkout 处理标记

```python
checkout.completing_started_at = timezone.now()
checkout.save(update_fields=["completing_started_at"])
```

其他请求可通过检查此字段判断 checkout 是否正在处理中。

### 5.2 悲观锁：数据库行锁

```python
Checkout.objects.select_for_update().filter(pk=checkout_pk).first()
```

### 5.3 原子更新：F() 表达式

```python
Stock.objects.filter(pk=stock_pk).update(
    quantity_allocated=F("quantity_allocated") + quantity
)
```

避免读取-修改-写入的竞态条件。

### 5.4 幂等性保障

1. **Checkout 已删除处理**：不存在时尝试通过 token 查找订单
2. **订单重复创建检查**：`_create_order()` 首先检查 `Order.objects.filter(checkout_token=checkout.token)`
3. **并发处理防护**：`completing_started_at` 标记 + `select_for_update` 双重保障

> **事实修正**：`Order.checkout_token` 字段仅建有普通 BTree 索引（`checkout_token_btree_idx`，`saleor/order/models.py:403`），**并非唯一约束**。幂等性完全通过应用层检查保证，而非数据库级约束。

---

## 6. 关键设计决策分析

### 6.1 临时库存预留（Reservation）机制

**设计意图**：
- 代码注释表明设计者希望 "Process payments out of transaction"（事务外处理支付）
- 但实际实现中支付仍在事务内执行
- 通过 Reservation 机制在事务1和事务3之间锁定库存，避免长事务

**收益**：
- 事务1提交后立即释放 checkout 行锁
- 其他用户可继续浏览和操作
- 通过 Reservation 保证库存不被超卖

**代价**：
- 增加了系统复杂度
- 失败时 Reservation 不会被主动清理，依赖过期机制
- 预留时间窗内库存对其他用户不可用

### 6.2 三事务分段设计

**模式**：`锁定并预留 → 支付处理 → 最终确认`

**适用场景**：包含外部调用的长流程，在数据一致性和并发性能间取得平衡。

**权衡**：
- ✅ 避免单个长事务持有锁
- ✅ 各阶段失败边界清晰
- ❌ 增加了部分失败场景的处理复杂度
- ❌ 需要补偿机制处理跨事务失败

### 6.3 transaction.on_commit 的使用

所有异步事件（订单通知、库存预警等）通过 `transaction.on_commit()` 注册，确保：
- 只有事务真正提交后才触发事件
- 事务回滚时不会发送错误通知

---

## 7. 总结

Saleor 结账流程采用**三事务分段 + 悲观锁 + 临时库存预留 + 补偿机制**的设计模式：

| 设计目标 | 实现方式 |
|---------|---------|
| **数据一致性** | 数据库事务 + 行级锁 + 完整快照固化 + Reservation 机制 |
| **并发性能** | 分段事务减少锁持有时间 + 按序锁定防死锁 |
| **故障恢复** | `_complete_checkout_fail_handler()` 统一补偿 + 幂等性检查 |
| **历史完整性** | 地址、商品、价格、折扣全量快照，订单数据永不失效 |

### 关键事实结论

1. ✅ **支付处理在事务内执行**：与代码注释的设计意图不符，实际 `_process_payment()` 完全包裹在 `transaction_with_commit_on_errors()` 内

2. ✅ **库存分配是独立事务**：`allocate_stocks()` 和 `allocate_preorders()` 各自拥有 `@traced_atomic_transaction()`，失败时向外抛出异常由上层处理

3. ✅ **库存分配失败的补偿**：支付已完成的情况下，`_complete_checkout_fail_handler()` 会主动调用支付网关退款

4. ✅ **Reservation 不主动回滚**：失败处理器不会删除已创建的 Reservation，依赖过期机制自动清理

5. ✅ **checkout_token 无唯一约束**：仅建有普通 BTree 索引，幂等性通过应用层检查保证

该设计在保证数据正确性的同时，最大限度提升了高并发场景下的系统吞吐量，但也增加了部分失败场景的处理复杂度。

---

## 8. 关键代码引用

| 功能 | 文件位置 |
|------|---------|
| 结账主流程 | `saleor/checkout/complete_checkout.py:1699-1772` |
| 支付分段流程（三事务） | `saleor/checkout/complete_checkout.py:1823-1958` |
| 库存预留机制 | `saleor/checkout/complete_checkout.py:1961-1993` |
| 订单创建核心 | `saleor/checkout/complete_checkout.py:764-909` |
| 库存分配（独立事务） | `saleor/warehouse/management.py:91-241` |
| 失败处理器 | `saleor/checkout/complete_checkout.py:1996-2033` |
| 支付处理 | `saleor/checkout/complete_checkout.py:1048-1085` |
| 订单创建后异常捕获 | `saleor/checkout/complete_checkout.py:1188-1206` |
| 地址复制 | `saleor/account/models.py:122-124` |
| Order 模型索引定义 | `saleor/order/models.py:373-414` |
