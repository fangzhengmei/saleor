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

---

## 九、失败场景分析

### 9.1 事务回滚机制

#### 两种事务包装器的区别

| 包装器 | 行为 | 适用场景 |
|--------|------|---------|
| `transaction_with_commit_on_errors()` | 发生异常时先提交事务，再抛出异常 | 支付前准备、支付处理等需要持久化错误状态的场景 |
| `traced_atomic_transaction()` / `transaction.atomic()` | 发生异常时回滚事务 | 订单创建、库存分配等需要原子性的场景 |

**`transaction_with_commit_on_errors` 实现**（`saleor/core/transactions.py:8-20`）：
```python
@contextmanager
def transaction_with_commit_on_errors():
    error = None
    with traced_atomic_transaction():
        try:
            yield
        except DatabaseError:
            raise
        except Exception as e:
            error = e
    if error:
        raise error
```

> **关键区别**：异常被捕获后，内层事务正常提交，然后在外层重新抛出。这样即使失败，`checkout.completing_started_at`、`checkout.is_voucher_usage_increased` 等状态变更也会被持久化。

---

### 9.2 统一失败处理器 `_complete_checkout_fail_handler`

**代码位置**：`saleor/checkout/complete_checkout.py:1996-2033`

**核心职责**：
1. 释放 `checkout.completing_started_at` 处理标记
2. 回滚优惠券使用量（`_release_checkout_voucher_usage`）
3. 退款或取消支付（`gateway.payment_refund_or_void`）

**处理逻辑**：
```python
def _complete_checkout_fail_handler(
    checkout_info, manager, *, voucher_code=None, voucher=None, payment=None
):
    checkout = checkout_info.checkout
    update_fields = []
    
    # 1. 释放处理标记（总是执行）
    if checkout.completing_started_at is not None:
        checkout.completing_started_at = None
        update_fields.append("completing_started_at")
    
    # 2. 释放优惠券使用量（仅当传入 voucher 时执行）
    if voucher:
        _release_checkout_voucher_usage(
            checkout, voucher_code, voucher, customer_email, update_fields
        )
    
    if update_fields:
        checkout.save(update_fields=update_fields)
    
    # 3. 退款/取消支付（仅当传入 payment 时执行）
    if payment:
        gateway.payment_refund_or_void(
            payment, manager, channel_slug=checkout_info.channel.slug
        )
```

> **⚠️ 关键收敛**：优惠券回滚和支付退款**不是总能执行**，取决于调用时是否传入了 `voucher` 和 `payment` 参数。

#### 各调用点参数对比

| 调用位置 | 传入 `payment` | 传入 `voucher` | 说明 |
|---------|--------------|---------------|------|
| `_process_payment` 支付失败 | ❌ 不传 | ❌ 不传 | 支付失败时**不会**退款，**不会**回滚优惠券 |
| `complete_checkout_pre_payment_part` 准备失败 | ✅ 传 | ❌ 不传 | 会退款，但**不会**回滚优惠券 |
| `complete_checkout_post_payment_part` 订单创建失败 | ✅ 传 | ✅ 传 | 会退款，会回滚优惠券 |
| `complete_checkout_with_payment` 支付变为 inactive | ✅ 传 | ⚠️ 传 `order_data["voucher"]` | 可能回滚优惠券（如果 order_data 中包含） |
| `create_order_from_checkout` 订单创建失败 | ❌ 不传 | ✅ 传 | **不会**退款，但会回滚优惠券 |

**支付退款前置条件**（`saleor/payment/gateway.py:546-574`）：

`payment_refund_or_void` 内部还有两层检查：
1. **`can_refund()`**：`payment.charge_status` 必须是 `PARTIALLY_CHARGED` / `FULLY_CHARGED` / `PARTIALLY_REFUNDED`
2. **`can_void()`**：必须同时满足 `payment.not_charged` **且** `payment.is_authorized`

> **⚠️ 关键收敛**：即使传入了 `payment`，也不一定能退款/取消。如果支付在授权阶段就失败了：
> - `is_authorized` = False（没有成功的 AUTH 交易）
> - `charge_status` = NOT_CHARGED
> - 结果：`can_refund()` 和 `can_void()` 都返回 False，**什么也不做**

**优惠券回滚前置条件**（`saleor/checkout/complete_checkout.py:156-177`）：

`_release_checkout_voucher_usage` 内部还有检查：
```python
if not checkout.is_voucher_usage_increased:
    return  # 优惠券未增加过，直接返回
```

> **⚠️ 关键收敛**：即使传入了 `voucher`，如果 `is_voucher_usage_increased` 为 False，也不会执行回滚。

**防止重复退款**：
根据 `TransactionKind.REFUND_ONGOING` 或 `TransactionKind.VOID` 检查是否已处理，避免重复操作。

---

### 9.3 各失败时点详细分析

#### 时点 1：支付前准备阶段失败

**发生位置**：`complete_checkout_pre_payment_part()`

**可能场景**：
- `_prepare_checkout_with_payment()` 验证失败（支付不足额、支付未激活等）
- `_get_order_data()` 中 `_prepare_order_data()` 失败
  - 税费计算失败（`TaxError`）
  - 运费方法无效
  - 商品已下架/不可购买
  - 优惠券不可用（`NotApplicable`）

**代码位置**：`saleor/checkout/complete_checkout.py:1121-1129`

**处理流程**：
```python
try:
    _prepare_checkout_with_payment(...)
except ValidationError as exc:
    _complete_checkout_fail_handler(checkout_info, manager, payment=payment)
    raise exc

try:
    order_data = _get_order_data(...)
except ValidationError as exc:
    _complete_checkout_fail_handler(checkout_info, manager, payment=payment)
    raise exc
```

> **⚠️ 关键收敛**：此处调用 `_complete_checkout_fail_handler` 时传入了 `payment`，但**没有传入 `voucher`**。
> - 如果 `_prepare_checkout_with_payment` 失败，优惠券还未增加，无需回滚
> - 如果 `_get_order_data` 失败（在 `_prepare_order_data` 中调用了 `_process_voucher_data_for_order` 增加了优惠券），则优惠券**不会被回滚**（因为没传 voucher 参数）

**各要素状态**：

| 要素 | 状态 |
|------|------|
| **订单创建** | ❌ 未创建（`_create_order` 尚未调用） |
| **库存预占** | ❌ 临时预占未创建（在 `_reserve_stocks_without_availability_check` 之前失败） |
| **优惠券** | ⚠️ 视失败时机而定<br>• `_prepare_checkout_with_payment` 失败：未增加，无需回滚<br>• `_get_order_data` 失败：已增加但**不回滚**（调用时未传 voucher 参数） |
| **支付** | ✅ 尝试退款/取消（传入了 payment）<br>⚠️ 但受 `can_refund()`/`can_void()` 前置条件限制，可能实际不执行 |
| **事件/通知** | ❌ 不触发任何订单事件或邮件 |
| **事务** | 提交（使用 `transaction_with_commit_on_errors`） |

---

#### 时点 2：临时库存预占后、支付处理前失败

**发生位置**：第一个事务块结束后，支付处理前

**可能场景**：
- 极少发生，因为第一个事务块只有数据库操作，无外部调用
- 极端情况：进程崩溃、服务器重启等

**各要素状态**：

| 要素 | 状态 |
|------|------|
| **订单创建** | ❌ 未创建 |
| **库存预占** | ✅ 临时预占已创建<br>⚠️ **不会主动释放**，等待过期自动清理 |
| **优惠券** | ⚠️ 如果已增加，`is_voucher_usage_increased=True`<br>下次重试时会跳过重复增加 |
| **支付** | ❌ 尚未处理 |
| **事件/通知** | ❌ 不触发 |
| **事务** | 已提交（第一个事务块正常结束） |

> **注意**：此场景下临时库存预占不会被主动删除，依赖 `reserved_until` 过期时间和定时任务自动清理。

---

#### 时点 3：支付处理阶段失败

**发生位置**：`_process_payment()` 内

**可能场景**：
- 支付网关拒绝交易（余额不足、卡过期等）
- 3D Secure 验证失败
- 网络超时
- 支付网关内部错误

**代码位置**：`saleor/checkout/complete_checkout.py:1082-1084`

**处理流程**：
```python
try:
    txn = gateway.process_payment(...)  # 或 gateway.confirm(...)
    if not txn.is_success:
        raise PaymentError(txn.error)
except PaymentError as e:
    # ⚠️ 关键问题：只传了 checkout_info 和 manager，没有传 payment 和 voucher!
    _complete_checkout_fail_handler(checkout_info, manager)
    raise ValidationError(str(e), code=CheckoutErrorCode.PAYMENT_ERROR.value) from e
```

> **⚠️ 关键收敛**：此处调用 `_complete_checkout_fail_handler` 时 **既没有传 `payment`，也没有传 `voucher`**！
> - 支付不会被退款/取消
> - 优惠券不会被回滚
> - 只有 `checkout.completing_started_at` 会被清理

**各要素状态**：

| 要素 | 状态 |
|------|------|
| **订单创建** | ❌ 未创建 |
| **库存预占** | ✅ 临时预占已创建<br>⚠️ 等待过期自动清理 |
| **优惠券** | ⚠️ 已增加但**不回滚**（调用时未传 voucher 参数）<br>下次重试时 `is_voucher_usage_increased=True` 会跳过重复增加 |
| **支付** | ⚠️ **不退款/不取消**（调用时未传 payment 参数）<br>⚠️ 且 `can_refund()`/`can_void()` 前置条件可能不满足 |
| **Transaction 记录** | ✅ 已创建（`kind=AUTH/CAPTURE`, `is_success=False`），用于审计和追踪 |
| **支付状态** | ⚠️ 不更新（`gateway_postprocess` 对失败交易直接 return） |
| **事件/通知** | ❌ 不触发 |
| **事务** | 提交（`_process_payment` 被 `transaction_with_commit_on_errors` 包裹） |

> **补充说明**：即使传入了 payment，支付状态的更新也只在 `gateway_postprocess` 中对成功交易执行。失败交易的支付状态保持原样，charge_status 仍为 NOT_CHARGED，is_authorized 仍为 False。

---

#### 时点 4：支付成功后、订单创建前失败

**发生位置**：`complete_checkout_post_payment_part()` 内

**子场景 4a：支付需要额外验证（action_required=True）**

**代码位置**：`saleor/checkout/complete_checkout.py:1159-1168`

```python
action_required = txn.action_required
if action_required:
    action_data = txn.action_required_data
    # 释放优惠券，不创建订单
    _release_checkout_voucher_usage(
        checkout_info.checkout,
        checkout_info.voucher_code,
        checkout_info.voucher,
        user_email,
    )
    # 不调用 _create_order，直接返回
    return None, True, action_data
```

**典型场景**：3D Secure 验证、SCA 强客户认证

> **⚠️ 关键收敛**：此处直接调用 `_release_checkout_voucher_usage`，**没有通过 `_complete_checkout_fail_handler`**。
> - 优惠券会被回滚（直接调用释放函数）
> - 支付**不会**被退款（action_required 表示需要用户进一步验证，不是支付失败）
> - 没有 `_complete_checkout_fail_handler` 中的其他清理逻辑

**各要素状态**：

| 要素 | 状态 |
|------|------|
| **订单创建** | ❌ 未创建（等待用户完成验证后重试） |
| **库存预占** | ✅ 临时预占已创建<br>⚠️ 等待过期，如果用户在过期前完成验证，重试时会重新创建 |
| **优惠券** | ✅ 释放（直接调用 `_release_checkout_voucher_usage`，重试时重新增加） |
| **支付** | ⚠️ 不退款，等待验证结果<br>`Transaction.action_required=True` |
| **事件/通知** | ❌ 不触发 |
| **事务** | 提交 |

**子场景 4b：支付变为 inactive**

**代码位置**：`saleor/checkout/complete_checkout.py:1907-1918`

```python
payment.refresh_from_db()
if not payment.is_active:
    _complete_checkout_fail_handler(
        checkout_info, manager,
        voucher=order_data.get("voucher"),  # ⚠️ 传入的是 order_data 中的 voucher
        payment=payment,
    )
    raise ValidationError(...)
```

> **⚠️ 关键收敛**：此处传入的 `voucher` 来自 `order_data.get("voucher")`，只有在 `_prepare_order_data` 中成功调用 `_process_voucher_data_for_order` 并返回 voucher 时才会有值。

**各要素状态**：

| 要素 | 状态 |
|------|------|
| **订单创建** | ❌ 未创建 |
| **库存预占** | ✅ 临时预占已创建，等待过期 |
| **优惠券** | ⚠️ 可能回滚（取决于 `order_data["voucher"]` 是否存在） |
| **支付** | ✅ 尝试退款/取消（传入了 payment）<br>⚠️ 受 `can_refund()`/`can_void()` 前置条件限制 |
| **事件/通知** | ❌ 不触发 |
| **事务** | 提交 |

---

#### 时点 5：订单创建过程中失败

**发生位置**：`_create_order()` 或 `_create_order_from_checkout()` 内

**可能场景**：
- 库存不足（`InsufficientStock`）—— 支付期间库存被其他订单占用
- 礼品卡不可用（`GiftCardNotApplicable`）
- 其他数据库错误

**有两个调用路径，处理逻辑不同**：

**路径 A：支付流程中创建订单失败**（`complete_checkout_post_payment_part`）
**代码位置**：`saleor/checkout/complete_checkout.py:1188-1206`

```python
try:
    order = _create_order(...)
except InsufficientStock as e:
    _complete_checkout_fail_handler(
        checkout_info, manager,
        voucher_code=checkout_info.voucher_code,
        voucher=checkout_info.voucher,
        payment=payment,  # ✅ 传入了 payment
    )
    error = prepare_insufficient_stock_checkout_validation_error(e)
    raise error from e
```

**路径 B：直接从 checkout 创建订单失败**（`create_order_from_checkout`）
**代码位置**：`saleor/checkout/complete_checkout.py:1668-1683`

```python
except InsufficientStock:
    _complete_checkout_fail_handler(
        checkout_info, manager,
        voucher_code=code,
        voucher=voucher,  # ✅ 传入了 voucher
        # ❌ 没有传入 payment！
    )
    raise
```

> **⚠️ 关键收敛**：两个路径的参数传递不同！
> - 路径 A（支付流程）：传入了 `payment` 和 `voucher`，会尝试退款和回滚优惠券
> - 路径 B（直接创建订单）：只传入了 `voucher`，**不会退款**，但会回滚优惠券

**子场景 5a：`Order.objects.create()` 之前失败**

| 要素 | 路径 A（支付流程） | 路径 B（直接创建） |
|------|------------------|------------------|
| **订单创建** | ❌ 未创建 | ❌ 未创建 |
| **库存预占** | ❌ 正式分配未创建<br>✅ 临时预占已创建，等待过期 | ❌ 无临时预占<br>❌ 正式分配未创建 |
| **优惠券** | ✅ 回滚（传入了 voucher） | ✅ 回滚（传入了 voucher） |
| **支付** | ✅ 尝试退款/取消（传入了 payment）<br>⚠️ 受 `can_refund()`/`can_void()` 限制 | ❌ 不退款（未传入 payment） |
| **事件/通知** | ❌ 不触发 | ❌ 不触发 |
| **事务** | 回滚（`_create_order` 使用 `traced_atomic_transaction`） | 回滚 |

**子场景 5b：`Order.objects.create()` 之后、`allocate_stocks()` 之前失败**

| 要素 | 路径 A（支付流程） | 路径 B（直接创建） |
|------|------------------|------------------|
| **订单创建** | ❌ 已创建但事务回滚，最终不存在 | ❌ 已创建但事务回滚，最终不存在 |
| **库存预占** | ❌ 正式分配未创建<br>✅ 临时预占等待过期 | ❌ 正式分配未创建 |
| **优惠券** | ✅ 回滚 | ✅ 回滚 |
| **支付** | ✅ 尝试退款/取消<br>⚠️ 受前置条件限制 | ❌ 不退款 |
| **事件/通知** | ❌ 不触发（`transaction.on_commit` 回调因回滚不执行） | ❌ 不触发 |
| **事务** | 回滚 | 回滚 |

**子场景 5c：`allocate_stocks()` 之后失败**

| 要素 | 路径 A（支付流程） | 路径 B（直接创建） |
|------|------------------|------------------|
| **订单创建** | ❌ 已创建但事务回滚，最终不存在 | ❌ 已创建但事务回滚，最终不存在 |
| **库存预占** | ❌ 正式分配已创建但事务回滚，最终不存在<br>✅ 临时预占等待过期 | ❌ 正式分配已创建但事务回滚，最终不存在 |
| **优惠券** | ✅ 回滚 | ✅ 回滚 |
| **支付** | ✅ 尝试退款/取消<br>⚠️ 受前置条件限制 | ❌ 不退款 |
| **事件/通知** | ❌ 不触发 | ❌ 不触发 |
| **事务** | 回滚 | 回滚 |

> **关键洞察**：所有在 `_create_order` / `_create_order_from_checkout` 内的数据库操作都在事务中，任何失败都会导致全部回滚。`order_created` 和 `send_order_confirmation` 通过 `transaction.on_commit` 注册，只有事务成功提交才会执行。

---

#### 时点 6：订单创建成功后失败

**发生位置**：订单创建成功并提交后，返回给用户前

**可能场景**：
- 进程崩溃、网络中断
- 序列化响应时出错（极罕见）

**各要素状态**：

| 要素 | 状态 |
|------|------|
| **订单创建** | ✅ 已创建（事务已提交） |
| **库存预占** | ✅ 正式分配已创建 |
| **优惠券** | ✅ 已使用 |
| **支付** | ✅ 已关联到订单 |
| **事件/通知** | ✅ `order_created` 和 `send_order_confirmation` 已触发<br>（通过 `transaction.on_commit` 在事务提交后执行） |
| **事务** | 已提交 |

> **注意**：此场景下订单实际已创建成功，用户可能因网络问题看不到结果。用户重试时会通过 `Order.objects.get_by_checkout_token()` 返回已有订单，不会重复创建。

---

### 9.4 库存预占释放机制

#### 临时预占（Reservation）的生命周期

**创建**：`_reserve_stocks_without_availability_check()`
```python
Reservation.objects.bulk_create([
    Reservation(
        quantity_reserved=line.line.quantity,
        reserved_until=timezone.now() + datetime.timedelta(
            seconds=settings.RESERVE_DURATION
        ),
        stock=stock,
        checkout_line=line.line,
    )
])
```

**释放方式**：
1. **自动过期释放**：
   - `reserved_until` 字段标记过期时间
   - 定时任务 `delete_expired_reservations_task` 定期清理
   - 代码位置：`saleor/warehouse/tasks.py:27-37`
   ```python
   Reservation.objects.filter(reserved_until__lt=timezone.now()).delete()
   ```

2. **正式分配时自动排除**：
   - `allocate_stocks()` 中计算可用库存时，会排除当前 checkout 的预占
   - 代码位置：`saleor/warehouse/management.py:249-255`
   ```python
   Reservation.objects.filter(stock_id__in=stocks_id)
       .not_expired()
       .exclude_checkout_lines(checkout_lines or [])
   ```

3. **结账删除时级联删除**：
   - `CheckoutLine` 有外键关联 `Reservation.checkout_line`
   - 结账删除时，`Reservation` 会级联删除

**⚠️ 失败场景不主动删除**：
`_complete_checkout_fail_handler` 中 **不** 包含删除临时预占的逻辑。所有失败场景下都不会主动删除临时预占，完全依赖：
1. **过期自动清理**（主要方式）
2. **结账删除时级联删除**（仅成功场景）
3. **正式分配时自动排除**（不删除，仅计算可用量时排除）

#### 正式分配（Allocation）的回滚

- 正式分配在 `_create_order` / `_create_order_from_checkout` 的事务内创建
- 如果订单创建失败，事务回滚会自动撤销 `Allocation` 记录
- `Stock.quantity_allocated` 的更新也会被回滚

> **⚠️ 关键收敛**：库存预占的释放是**最终一致**而非**强一致**的。失败场景下临时预占会存在 `RESERVE_DURATION` 秒，这段时间内其他用户可能看到"库存不足"，即使原订单已失败。

---

### 9.5 事件与通知触发规则

#### 成功场景触发的事件

通过 `transaction.on_commit` 注册，只有事务成功提交后才执行：

**代码位置**：`saleor/checkout/complete_checkout.py:893-907` 和 `:1358-1374`

```python
transaction.on_commit(
    lambda: order_created(
        order_info=order_info,
        user=user,
        app=app,
        manager=manager,
        site_settings=site_settings,
        automatic=is_automatic_completion,
    )
)

transaction.on_commit(
    lambda: send_order_confirmation(order_info, checkout.redirect_url, manager)
)
```

**`order_created` 触发的事件链**（`saleor/order/actions.py:286-345`）：
1. `ORDER_CREATED` webhook（异步）
2. 如果是预授权订单 → `ORDER_AUTHORIZED` webhook
3. 如果已全额支付 → `ORDER_FULLY_PAID` webhook + 支付确认邮件
4. 如果自动确认 → `ORDER_CONFIRMED` webhook + 订单确认邮件

#### 失败场景事件触发

**所有失败场景下：**
- ❌ `transaction.on_commit` 回调不执行（要么事务回滚，要么 `_create_order` 未被调用）
- ❌ 不触发任何 `ORDER_*` webhook
- ❌ 不发送订单确认邮件
- ❌ `_complete_checkout_fail_handler` 中不触发任何事件或通知

**例外情况 - 支付失败的交易记录：**
- 支付失败时会创建 `Transaction` 记录（`is_success=False`）
- 但这是支付网关层面的记录，不是订单层面的事件

---

### 9.6 失败场景汇总表

> **⚠️ 收敛说明**：以下状态基于代码分析的"最坏情况"或"典型情况"，实际行为受多种前置条件影响，详见各时点详细分析。

| 失败时点 | 订单创建 | 库存临时预占 | 库存正式分配 | 优惠券 | 支付 | 事件/通知 | 事务处理 |
|---------|---------|-------------|-------------|--------|------|----------|---------|
| **支付前准备失败** | ❌ 未创建 | ❌ 未创建 | ❌ 未创建 | ⚠️ 视时机而定<br>• 早期失败：未增加<br>• 后期失败：已增加但**不回滚**（调用未传 voucher） | ⚠️ 尝试退款（传入了 payment）<br>但受 `can_refund()`/`can_void()` 限制，可能不执行 | ❌ 不触发 | 提交 |
| **临时预占后崩溃** | ❌ 未创建 | ✅ 已创建<br>等待过期 | ❌ 未创建 | ⚠️ 可能已增加<br>`is_voucher_usage_increased=True` | ❌ 未处理 | ❌ 不触发 | 已提交 |
| **支付处理失败** | ❌ 未创建 | ✅ 已创建<br>等待过期 | ❌ 未创建 | ⚠️ 已增加但**不回滚**<br>（调用未传 voucher） | ⚠️ **不退款/不取消**<br>（调用未传 payment + 前置条件可能不满足） | ❌ 不触发 | 提交 |
| **action_required** | ❌ 未创建<br>等待重试 | ✅ 已创建<br>等待过期 | ❌ 未创建 | ✅ 释放（直接调用释放函数） | ⚠️ 不退款，等待验证<br>`Transaction.action_required=True` | ❌ 不触发 | 提交 |
| **支付变为inactive** | ❌ 未创建 | ✅ 已创建<br>等待过期 | ❌ 未创建 | ⚠️ 可能回滚（取决于 `order_data["voucher"]` 是否存在） | ⚠️ 尝试退款（传入了 payment）<br>但受前置条件限制 | ❌ 不触发 | 提交 |
| **订单创建前失败（路径A）** | ❌ 未创建 | ✅ 已创建<br>等待过期 | ❌ 未创建 | ✅ 回滚（传入了 voucher） | ⚠️ 尝试退款（传入了 payment）<br>但受前置条件限制 | ❌ 不触发 | 回滚 |
| **订单创建前失败（路径B）** | ❌ 未创建 | ❌ 无临时预占 | ❌ 未创建 | ✅ 回滚（传入了 voucher） | ❌ 不退款（未传 payment） | ❌ 不触发 | 回滚 |
| **订单创建后失败（路径A）** | ❌ 已创建但回滚 | ✅ 已创建<br>等待过期 | ❌ 已创建但回滚 | ✅ 回滚 | ⚠️ 尝试退款<br>但受前置条件限制 | ❌ 不触发 | 回滚 |
| **订单创建后失败（路径B）** | ❌ 已创建但回滚 | ❌ 无临时预占 | ❌ 已创建但回滚 | ✅ 回滚 | ❌ 不退款 | ❌ 不触发 | 回滚 |
| **库存分配后失败（路径A）** | ❌ 已创建但回滚 | ✅ 已创建<br>等待过期 | ❌ 已创建但回滚 | ✅ 回滚 | ⚠️ 尝试退款<br>但受前置条件限制 | ❌ 不触发 | 回滚 |
| **库存分配后失败（路径B）** | ❌ 已创建但回滚 | ❌ 无临时预占 | ❌ 已创建但回滚 | ✅ 回滚 | ❌ 不退款 | ❌ 不触发 | 回滚 |
| **订单成功后崩溃** | ✅ 已创建 | ❌ 已释放<br>（结账删除级联） | ✅ 已创建 | ✅ 已使用 | ✅ 已关联 | ✅ 已触发 | 已提交 |

> **路径 A** = 支付流程中创建订单（`complete_checkout_post_payment_part`）<br>
> **路径 B** = 直接从 checkout 创建订单（`create_order_from_checkout` / `OrderCreateFromCheckout` mutation）

---

### 9.7 关键代码引用（失败处理）

| 功能 | 文件位置 | 关键函数 |
|------|---------|---------|
| 事务包装器 | `saleor/core/transactions.py` | `transaction_with_commit_on_errors()` |
| 失败处理器 | `saleor/checkout/complete_checkout.py:1996` | `_complete_checkout_fail_handler()` |
| 支付退款/取消 | `saleor/payment/gateway.py:546` | `payment_refund_or_void()` |
| 优惠券回滚 | `saleor/checkout/complete_checkout.py:156` | `_release_checkout_voucher_usage()` |
| 支付处理失败（缺参数） | `saleor/checkout/complete_checkout.py:1082` | `_process_payment()` 异常处理 |
| 订单创建失败（支付流程） | `saleor/checkout/complete_checkout.py:1188` | `complete_checkout_post_payment_part()` 异常处理 |
| 订单创建失败（直接创建） | `saleor/checkout/complete_checkout.py:1668` | `create_order_from_checkout()` 异常处理 |
| 支付状态检查 | `saleor/payment/models.py:443` | `Payment.can_void()`, `Payment.can_refund()` |
| 交易后处理（失败跳过） | `saleor/payment/utils.py:562` | `gateway_postprocess()` |
| 过期预占清理 | `saleor/warehouse/tasks.py:27` | `delete_expired_reservations_task()` |
| 订单已创建检查 | `saleor/checkout/complete_checkout.py:1844` | `Order.objects.get_by_checkout_token()` |

---

### 9.8 设计要点总结

1. **事务分层设计**：
   - 支付前准备、支付处理使用 `transaction_with_commit_on_errors`，确保状态持久化
   - 订单创建使用 `traced_atomic_transaction`，确保原子性
   - 支付在独立事务中执行，避免长时间锁定库存行

2. **库存预占的最终一致性**：
   - 失败时不主动删除临时预占，依赖过期机制
   - 简化了失败处理逻辑，避免分布式事务问题
   - 代价是库存可能被"锁定"一段时间（`RESERVE_DURATION`）
   - ⚠️ **收敛**：这是**最终一致**而非**强一致**，失败后短时间内库存仍显示被占用

3. **事件驱动的最终一致性**：
   - 所有事件和通知通过 `transaction.on_commit` 注册
   - 确保只有订单真正创建成功后才触发
   - 避免了"订单创建成功但事件未发"或"事件已发但订单回滚"的不一致

4. **幂等性设计**：
   - `checkout.completing_started_at` 防止重复处理
   - `checkout.is_voucher_usage_increased` 防止优惠券重复增加
   - `Order.objects.get_by_checkout_token()` 防止订单重复创建
   - `payment_refund_or_void` 检查已有交易，防止重复退款

5. **⚠️ 失败处理的不一致性（新发现）**：
   - `_complete_checkout_fail_handler` 的参数传递不统一
   - 支付失败（`_process_payment` 内）时**不退款也不回滚优惠券**
   - 直接创建订单（`OrderCreateFromCheckout` mutation）失败时**不退款**
   - 支付前准备后期失败时**不回滚优惠券**
   - 这些可能是代码缺陷，需要注意

6. **⚠️ 支付退款/取消的前置条件（新发现）**：
   - 即使传入 `payment`，也不一定能退款/取消
   - `can_void()` 需要 `not_charged and is_authorized`
   - `can_refund()` 需要特定的 `charge_status`
   - 如果支付在授权阶段就失败，两个条件都不满足，什么也不做
   - 失败交易的支付状态不会更新（`gateway_postprocess` 对失败交易直接 return）
