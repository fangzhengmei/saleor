# 结账完成触发订单生成事件链分析

## 一、整体事件链概述

### 1.1 入口点

结账完成流程有两个主要入口：

1. **`CheckoutComplete` mutation**（面向普通用户）
   - 文件：`saleor/graphql/checkout/mutations/checkout_complete.py`
   - 支持支付确认、3D Secure 等额外验证步骤

2. **`OrderCreateFromCheckout` mutation**（面向应用/管理员）
   - 文件：`saleor/graphql/checkout/mutations/order_create_from_checkout.py`
   - 需要 `HANDLE_CHECKOUTS` 权限
   - 直接从结账创建订单，无需支付处理

### 1.2 核心流程

核心逻辑在 `saleor/checkout/complete_checkout.py` 中：

```
CheckoutComplete.perform_mutation()
    ↓
complete_checkout()
    ├─→ 检查支付状态（已授权/零金额/允许未付款）
    │   ├─→ 是：complete_checkout_with_transaction()
    │   └─→ 否：complete_checkout_with_payment()
    │
    └─→ create_order_from_checkout()
            ↓
        _create_order_from_checkout()
            ├─→ 准备订单数据
            ├─→ 创建订单
            ├─→ 创建订单行
            ├─→ 库存分配
            ├─→ 关联支付
            └─→ 触发订单创建事件
```

---

## 二、库存预占写入时点

库存预占分为两个阶段：**临时预占**（支付期间）和 **正式分配**（订单创建后）。

### 2.1 临时预占（Temporary Reservation）

**时点**：支付处理前，在 `complete_checkout_with_payment()` 的第一个事务块内

**代码位置**：`saleor/checkout/complete_checkout.py:1868-1993`

**函数**：`_reserve_stocks_without_availability_check()`

**触发时机**：
```python
# complete_checkout_with_payment() 函数内
with transaction_with_commit_on_errors():
    # ... 锁定 checkout ...
    payment, customer_id, order_data = complete_checkout_pre_payment_part(...)
    
    # 【时点1】创建临时库存预留
    _reserve_stocks_without_availability_check(
        checkout_info,
        lines,
        site_settings.use_legacy_shipping_zone_stock_availability,
    )
```

**写入内容**：
- 创建 `Reservation` 对象
- 字段：
  - `quantity_reserved`：预留数量
  - `reserved_until`：预留过期时间（当前时间 + `RESERVE_DURATION` 秒）
  - `stock`：关联的库存记录
  - `checkout_line`：关联的结账行

**设计目的**：在外部支付网关调用期间（事务外执行），防止其他用户抢占相同库存。

---

### 2.2 正式分配（Stock Allocation）

**时点**：订单创建后，在 `_create_order()` 或 `_create_order_from_checkout()` 内

**代码位置**：`saleor/checkout/complete_checkout.py:835-852` 和 `saleor/checkout/complete_checkout.py:1514-1522`

**函数**：`allocate_stocks()` 和 `allocate_preorders()`

**触发时机**：
```python
# _create_order() 函数内
OrderLine.objects.bulk_create(order_lines)
OrderLineDiscount.objects.bulk_create(order_line_discounts)

# 【时点2】正式分配库存
allocate_stocks(
    order_lines_info,
    country_code,
    checkout_info.channel,
    site_settings,
    app or user,
    ...
)
allocate_preorders(
    order_lines_info,
    checkout_info.channel.slug,
    ...
)
```

**写入内容**：
- 创建 `Allocation` 对象（普通商品）
  - `order_line`：关联订单行
  - `stock`：关联库存
  - `quantity_allocated`：分配数量
- 创建 `PreorderAllocation` 对象（预售商品）
  - `order_line`：关联订单行
  - `product_variant_channel_listing`：关联渠道 listing
  - `quantity`：分配数量
- 更新 `Stock.quantity_allocated` 字段（使用 F() 表达式原子更新）

**代码位置**：`saleor/warehouse/management.py:91-241`

---

## 三、支付意向写入时点

支付处理根据流程不同分为两个分支。

### 3.1 交易流程（Transaction Flow）

**适用场景**：
- 结账已全额授权（`authorize_status == FULL`）
- 存在交易记录但无活跃支付
- 允许未付款订单
- 零金额订单 + 交易流程策略

**代码位置**：`saleor/checkout/complete_checkout.py:1775-1821`

**写入时点**：
```python
# 1. 准备阶段检查支付状态
_prepare_checkout_with_transactions(
    manager=manager,
    checkout_info=checkout_info,
    lines=lines,
    redirect_url=redirect_url,
)
```

在 `_prepare_checkout_with_transactions()` 中：
```python
# 检查授权状态
if (checkout_info.checkout.authorize_status != CheckoutAuthorizeStatus.FULL
    and not checkout_info.channel.allow_unpaid_orders):
    raise ValidationError(...)
```

**支付状态更新**：
- 支付状态由 `update_checkout_payment_statuses()` 维护
- 文件：`saleor/checkout/payment_utils.py:82-119`
- 更新字段：
  - `checkout.authorize_status`：NONE / PARTIAL / FULL
  - `checkout.charge_status`：NONE / PARTIAL / FULL / OVERCHARGED

---

### 3.2 支付流程（Payment Flow）

**适用场景**：需要处理活跃支付的情况

**代码位置**：`saleor/checkout/complete_checkout.py:1823-1958`

**写入时点**：

```python
# 【时点1】支付前准备
payment, customer_id, order_data = complete_checkout_pre_payment_part(...)
    ↓ 内部调用
    _prepare_checkout_with_payment(...)
        ↓ 内部调用
        clean_checkout_payment(...)  # 检查支付是否足额
```

```python
# 【时点2】处理支付（在独立事务中）
if payment:
    with transaction_with_commit_on_errors():
        # 锁定 checkout 和 payment
        txn = _process_payment(
            checkout_info=checkout_info,
            payment=payment,
            customer_id=customer_id,
            store_source=store_source,
            payment_data=payment_data,
            manager=manager,
            channel_slug=channel_slug,
            ...
        )
```

**`_process_payment()` 函数**（`saleor/checkout/complete_checkout.py:1048-1085`）：
- 调用 `gateway.confirm()` 或 `gateway.process_payment()`
- 创建 `Transaction` 记录
- 检查交易是否成功

**写入内容**：
- `Transaction` 对象：
  - `kind`：TransactionKind.AUTH / CAPTURE / CONFIRM 等
  - `is_success`：是否成功
  - `action_required`：是否需要额外验证（如 3D Secure）
  - `action_required_data`：验证所需数据
  - `customer_id`：支付网关客户 ID

```python
# 【时点3】支付后关联到订单
# 在 _create_order() 中
checkout.payments.update(order=order)
checkout.payment_transactions.update(order=order, checkout_id=None)
```

---

## 四、税费冻结写入时点

税费冻结是指在订单创建时将计算好的税费快照保存到订单中，确保后续价格变动不影响已创建订单。

### 4.1 税费重新计算

**时点**：订单数据准备阶段

**代码位置**：`saleor/checkout/complete_checkout.py:656-761`

**触发时机**：
```python
# complete_checkout_pre_payment_part() 内
fetch_checkout_data(checkout_info, manager, lines, requestor=app or user).get()

# _prepare_order_data() 内
try:
    manager.preprocess_order_creation(checkout_info, lines)
except TaxError:
    # 出错时回滚优惠券使用
    _release_checkout_voucher_usage(...)
    raise
```

**税费计算方式**：
1. **Flat Rate**：`update_checkout_prices_with_flat_rates()`
   - 文件：`saleor/tax/calculations/checkout.py:24-83`
   - 基于 `TaxClassCountryRate` 计算

2. **Tax App Webhook**：通过 `manager.preprocess_order_creation()` 触发
   - 同步 Webhook：`CHECKOUT_CALCULATE_TAXES`
   - 由外部税务应用返回税费数据

### 4.2 税费写入订单（冻结）

**时点**：订单创建时

**代码位置**：`saleor/checkout/complete_checkout.py:807-815`（`_create_order()`）和 `:1459-1478`（`_create_order_from_checkout()`）

**写入内容**：

**订单级税费字段**：
```python
order = Order.objects.create(
    ...
    tax_exemption=checkout_info.checkout.tax_exemption,
    tax_error=checkout_info.checkout.tax_error,
    shipping_tax_rate=shipping_tax_rate,  # 运费税率
    ...
)
```

**订单行级税费字段**：
```python
# _create_line_for_order() 中
line = OrderLine(
    ...
    unit_price=unit_price,              # 含税单价
    total_price=total_line_price,       # 含税总价
    tax_rate=tax_rate,                  # 税率
    undiscounted_unit_price=undiscounted_unit_price,
    undiscounted_total_price=undiscounted_total_price,
    ...
    **get_tax_class_kwargs_for_order_line(tax_class),  # 税类信息
)
```

**运费税费**：
```python
# _prepare_order_data() 中
order_data.update(_process_shipping_data_for_order(
    ...
    shipping_price=shipping_total,      # 含税运费
    ...
))
```

**税配置快照**：
```python
# _create_order() 中
update_order_display_gross_prices(order)
```
- 保存当前的 `display_gross_prices` 配置到订单
- 确保后续税率显示配置变更不影响历史订单

---

## 五、运费快照写入时点

运费信息在订单创建时被完整快照，包括基础价格、折扣、税费等。

### 5.1 运费数据准备

**时点**：订单数据准备阶段

**代码位置**：`saleor/checkout/complete_checkout.py:676-699`

**函数**：`_process_shipping_data_for_order()`

**触发时机**：
```python
# _prepare_order_data() 中
undiscounted_base_shipping_price = base_checkout_undiscounted_delivery_price(
    checkout_info, lines
)
base_shipping_price = base_checkout_delivery_price(checkout_info, lines)
shipping_total = calculations.checkout_shipping_price(
    manager=manager,
    checkout_info=checkout_info,
    lines=lines,
)
shipping_tax_rate = calculations.checkout_shipping_tax_rate(
    manager=manager,
    checkout_info=checkout_info,
    lines=lines,
)

order_data.update(
    _process_shipping_data_for_order(
        checkout_info,
        undiscounted_base_shipping_price,
        base_shipping_price,
        shipping_total,
        manager,
        lines,
    )
)
```

### 5.2 运费写入订单

**代码位置**：`saleor/checkout/complete_checkout.py:1449-1456` 和 `:1459-1478`

**写入内容**：

```python
# _process_shipping_data_for_order() 返回的数据结构
result: dict[str, Any] = {
    "undiscounted_base_shipping_price": undiscounted_base_shipping_price,  # 无折扣基础运费
    "shipping_address": shipping_address,                                   # 收货地址快照
    "base_shipping_price": base_shipping_price,                             # 基础运费（折扣后）
    "shipping_price": shipping_price,                                       # 含税运费
    "weight": calculate_checkout_weight(lines),                             # 包裹重量
    **delivery_method_info.get_details_for_conversion_to_order(),           # 配送方式详情
}

# 外部配送方式元数据
if isinstance(shipping_method, ShippingMethodData):
    result.update({
        "shipping_method_metadata": shipping_method.metadata,
        "shipping_method_private_metadata": shipping_method.private_metadata,
    })
```

**订单创建时写入**：
```python
order = Order.objects.create(
    ...
    shipping_tax_rate=shipping_tax_rate,
    **shipping_details,  # 包含上述所有运费相关字段
    ...
)
```

**配送方式详情**（来自 `get_details_for_conversion_to_order()`）：
- `shipping_method`：配送方式对象
- `shipping_method_name`：配送方式名称
- `collection_point`：自提点（如果是自提）
- `warehouse_pk`：仓库 ID

---

## 六、完整时序图

```
用户调用 checkoutComplete
    │
    ▼
1. 验证结账信息（地址、商品、邮箱）
    │
    ▼
2. 根据支付状态选择分支
    ├─ 已授权/零金额/允许未付款 → Transaction Flow
    │     │
    │     ▼
    │  2a. 验证支付状态
    │     │
    │     ▼
    │  2b. 直接调用 create_order_from_checkout()
    │
    └─ 需要处理支付 → Payment Flow
          │
          ▼
       2a. 锁定 checkout
          │
          ▼
       2b. 准备订单数据（含税费重算）
          │
          ▼
       2c. 创建临时库存预留 ──── 【库存预占时点1】
          │
          ▼
       2d. 处理支付（事务外）
          │  ├─ 调用支付网关
          │  └─ 创建 Transaction 记录 ─ 【支付意向时点】
          │
          ▼
       2e. 重新锁定 checkout
          │
          ▼
       2f. 调用 create_order_from_checkout()
          │
          ▼
3. create_order_from_checkout() 执行
    │
    ├─ 3a. 锁定 checkout（防并发）
    │
    ├─ 3b. 重新获取最新 checkout 数据
    │
    ├─ 3c. 增加优惠券使用量
    │
    ├─ 3d. 调用 _create_order_from_checkout()
    │     │
    │     ├─ 3d1. 计算总价、运费、税费
    │     │
    │     ├─ 3d2. 创建 Order 对象
    │     │     │
    │     │     ├─ 写入运费快照 ────────── 【运费快照时点】
    │     │     ├─ 写入税费信息 ────────── 【税费冻结时点】
    │     │     └─ 写入税配置快照
    │     │
    │     ├─ 3d3. 创建订单行和折扣
    │     │     └─ 写入每行的税率、含税价格 ─ 【税费冻结时点】
    │     │
    │     ├─ 3d4. 正式分配库存
    │     │     ├─ allocate_stocks() ───── 【库存预占时点2】
    │     │     └─ allocate_preorders()
    │     │
    │     ├─ 3d5. 关联支付到订单
    │     │
    │     ├─ 3d6. 添加礼品卡
    │     │
    │     └─ 3d7. 保存订单并提交事务
    │
    ├─ 3e. transaction.on_commit 触发事件
    │     ├─ order_created() 事件
    │     └─ 发送订单确认邮件
    │
    └─ 3f. 删除 checkout（可选）
          │
          ▼
4. 返回订单给用户
```

---

## 七、关键代码引用表

| 功能模块 | 文件位置 | 关键函数 |
|---------|---------|---------|
| 结账完成入口 | `saleor/graphql/checkout/mutations/checkout_complete.py` | `CheckoutComplete.perform_mutation()` |
| 订单创建入口 | `saleor/graphql/checkout/mutations/order_create_from_checkout.py` | `OrderCreateFromCheckout.perform_mutation()` |
| 核心流程控制 | `saleor/checkout/complete_checkout.py` | `complete_checkout()` |
| 交易流程 | `saleor/checkout/complete_checkout.py` | `complete_checkout_with_transaction()` |
| 支付流程 | `saleor/checkout/complete_checkout.py` | `complete_checkout_with_payment()` |
| 临时库存预留 | `saleor/checkout/complete_checkout.py:1868` | `_reserve_stocks_without_availability_check()` |
| 支付处理 | `saleor/checkout/complete_checkout.py:1048` | `_process_payment()` |
| 创建订单 | `saleor/checkout/complete_checkout.py:764` | `_create_order()` |
| 从结账创建订单 | `saleor/checkout/complete_checkout.py:1377` | `_create_order_from_checkout()` |
| 订单数据准备 | `saleor/checkout/complete_checkout.py:656` | `_prepare_order_data()` |
| 运费数据处理 | `saleor/checkout/complete_checkout.py:180` | `_process_shipping_data_for_order()` |
| 库存分配 | `saleor/warehouse/management.py:91` | `allocate_stocks()` |
| 预售分配 | `saleor/warehouse/management.py:811` | `allocate_preorders()` |
| 支付状态更新 | `saleor/checkout/payment_utils.py:82` | `update_checkout_payment_statuses()` |
| 税费计算（Flat Rate） | `saleor/tax/calculations/checkout.py:24` | `update_checkout_prices_with_flat_rates()` |
| 结账清理 | `saleor/checkout/checkout_cleaner.py` | `clean_checkout_payment()`, `clean_checkout_shipping()` |

---

## 八、事务与并发控制要点

1. **`select_for_update` 行锁**：在关键操作前锁定 checkout、payment 等记录，防止并发修改

2. **`transaction_with_commit_on_errors`**：即使发生错误也提交事务，确保错误状态被持久化

3. **`traced_atomic_transaction`**：带追踪的原子事务装饰器

4. **支付在事务外执行**：支付处理在独立事务中进行，避免长时间锁定库存行

5. **临时库存预留**：支付期间创建临时预留，防止超卖

6. **优惠券使用量原子增加**：使用 `select_for_update` 锁定优惠券后增加使用量
