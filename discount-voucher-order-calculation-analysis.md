# 折扣与优惠券订单计算逻辑深度分析报告

## 一、两条计算路径：Checkout vs Draft Order

### 1.1 Checkout 计算路径

#### 1.1.1 触发时机
客户在前台结账过程中，价格重新计算时触发。

#### 1.1.2 入口函数
```
create_or_update_discount_objects_from_promotion_for_checkout()
  ├─ create_checkout_line_discount_objects_for_catalogue_promotions()
  └─ create_checkout_discount_objects_for_order_promotions()
```

#### 1.1.3 互斥与覆盖条件
**代码位置**：`saleor/discount/utils/checkout.py:226-229`

| 已存在 | 操作 | 说明 |
|--------|------|------|
| `checkout.voucher_code != None` | 清除订单级促销 | 调用 `_clear_checkout_discount()`，删除所有 `ORDER_PROMOTION` 类型折扣，清除赠品行 |
| 行级手动折扣 (`MANUAL`) | 移除行级目录促销 | 手动折扣与其他行级折扣不叠加 |
| 赠品行 (`is_gift=True`) | 移除所有折扣 | 赠品不参与任何折扣计算 |

#### 1.1.4 订单级促销资格判定
**代码位置**：`saleor/discount/utils/checkout.py:264-281`

1. 先调用 `_set_checkout_base_prices()` 设置基础价格
2. `base_subtotal` 仅包含目录级促销（`include_voucher=False`）
3. 使用 `base_subtotal` 作为订单级促销资格判定的基准
4. 选择匹配的订单级促销中**折扣金额最大**的规则应用

#### 1.1.5 订单级折扣分摊
**代码位置**：`saleor/checkout/base_calculations.py:264-327`

整单折扣（订单级促销或整单券）按各行基础价格比例分摊：
- 单行：折扣全部应用于该行
- 多行：按各行 `base_line_total_price` 占比分摊
- 最后一行吸收分摊余数，确保折扣总额精确

---

### 1.2 Draft Order 计算路径

#### 1.2.1 触发时机
后台管理员编辑草稿订单时触发。

#### 1.2.2 入口函数
```
refresh_order_prices()
  ├─ create_or_update_voucher_discount_objects_for_order()
  │   ├─ create_or_update_discount_object_from_order_level_voucher()
  │   └─ create_or_update_line_discount_objects_from_voucher()
  └─ handle_order_promotion()
      └─ create_order_discount_objects_for_order_promotions()
```

#### 1.2.3 互斥与覆盖条件
**代码位置**：`saleor/discount/utils/order.py:188-191`

| 已存在 | 操作 | 说明 |
|--------|------|------|
| `order.voucher_code != None` | 跳过订单级促销 | 直接 `return`，不处理订单级促销 |
| 存在 `MANUAL` 类型订单级折扣 | 跳过订单级促销 | 手动订单折扣优先级最高 |
| 行级手动折扣 | 移除行级优惠券折扣 | **代码位置**：`saleor/discount/utils/voucher.py:541-550` |
| 赠品行 (`is_gift=True`) | 移除行级优惠券折扣 | **代码位置**：`saleor/discount/utils/voucher.py:545` |

#### 1.2.4 订单级优惠券处理
**代码位置**：`saleor/discount/utils/voucher.py:387-482`

整单/运费优惠券折扣对象创建逻辑：
1. 判定是否应删除现有订单级优惠券折扣
   ```python
   should_delete_order_level_voucher_discount = (
       not order.voucher_id
       or (is_order_voucher and is_manual_discount)  # 手动折扣覆盖优惠券
       or is_line_level_voucher  # 行级券不与整单券共存
   )
   ```
2. 整单券基于 `order.subtotal.net` 计算折扣
3. 运费券基于 `order.undiscounted_base_shipping_price` 计算折扣

#### 1.2.5 草稿订单特有逻辑
- 支持 `use_denormalized_data=True` 模式：基于优惠券应用时的快照数据计算
  **代码位置**：`saleor/discount/utils/voucher.py:552-566`
- 适用于订单已支付后重新计算价格的场景，保证价格一致性

---

### 1.3 两条路径差异对比表

| 维度 | Checkout 路径 | Draft Order 路径 |
|------|--------------|------------------|
| 触发方 | 前端客户操作 | 后台管理员操作 |
| 手动折扣处理 | 行级手动折扣移除目录促销 | 行级手动折扣移除优惠券；订单级手动折扣跳过所有促销 |
| 优惠券折扣对象 | 不创建独立 Discount 对象，直接计算 | 创建独立 `OrderLineDiscount` / `OrderDiscount` 对象持久化 |
| 基础价格设置 | `_set_checkout_base_prices()` | `_set_order_base_prices()` |
| 反归一化数据 | 不支持 | 支持 `use_denormalized_data` 模式 |
| 折扣分摊时机 | 计算时动态分摊 | 分摊结果持久化到数据库 |

---

## 二、apply_once_per_order 完整判定过程

### 2.1 概念说明
`apply_once_per_order` 是优惠券的一个布尔字段，当设为 `True` 时，优惠券折扣仅应用于订单中**最便宜的一个符合条件的商品行**，而不是所有符合条件的商品行。

### 2.2 完整判定流程

```
Step 1: 判定优惠券是否为 line-level
  ↓ (voucher.type == SPECIFIC_PRODUCT OR voucher.apply_once_per_order)
Step 2: 筛选符合适用范围的商品行 (get_discounted_lines)
  ├─ 如果指定了 products/variants/categories/collections
  │   └─ 仅保留匹配范围内的行
  └─ 如果未指定任何范围
      └─ 所有非赠品行都符合条件
  ↓
Step 3: 从筛选结果中排除赠品行 (is_gift=True)
  ↓
Step 4: 如果 apply_once_per_order=True
  ├─ 在筛选出的行中找到单价最低的行 (get_the_cheapest_line)
  │   比较 key: line_info.variant_discounted_price
  └─ 仅保留这一行作为折扣目标
  ↓
Step 5: 将 voucher 关联到目标行的 line_info
  ↓
Step 6: 计算该行折扣金额
  └─ 如果 apply_once_per_order=True
      └─ 折扣 = min(总折扣额, 该行总价)  # 不超过该行价格
```

### 2.3 关键代码详解

#### 2.3.1 适用范围筛选
**代码位置**：`saleor/discount/utils/voucher.py:220-253`

```python
def get_discounted_lines(lines, voucher_info):
    if voucher_info.product_pks or voucher_info.collection_pks or \
       voucher_info.category_pks or voucher_info.variant_pks:
        for line_info in lines:
            if line_info.line.is_gift:  # 排除赠品
                continue
            # 匹配 variant / product / category / collection
            if (line_variant.pk in voucher_info.variant_pks
                or line_product.pk in voucher_info.product_pks
                or line_category.pk in voucher_info.category_pks
                or line_collections.intersection(voucher_info.collection_pks)):
                discounted_lines.append(line_info)
    else:
        # 未指定范围，所有产品都适用（排除赠品）
        discounted_lines.extend(lines)
    return discounted_lines
```

**关键点**：
- 只要指定了任何一个范围字段，就进行严格匹配
- 未指定任何范围字段时，**所有非赠品**都适用
- 赠品行 (`is_gift=True`) 始终被排除

#### 2.3.2 最便宜行选取
**代码位置**：`saleor/discount/utils/voucher.py:256-261`

```python
def get_the_cheapest_line(lines_info):
    if not lines_info:
        return None
    return min(lines_info, key=lambda line_info: line_info.variant_discounted_price)
```

**比较基准**：`variant_discounted_price` — 已应用目录级促销后的单价

#### 2.3.3 attach_voucher_to_line_info
**代码位置**：`saleor/discount/utils/voucher.py:194-217`

```python
def attach_voucher_to_line_info(voucher_info, lines_info):
    voucher = voucher_info.voucher
    discounted_lines_by_voucher = []
    lines_included_in_discount = lines_info

    if voucher.type == VoucherType.SPECIFIC_PRODUCT:
        discounted_lines_by_voucher.extend(get_discounted_lines(lines_info, voucher_info))
        lines_included_in_discount = discounted_lines_by_voucher

    if voucher.apply_once_per_order:
        if cheapest_line := get_the_cheapest_line(lines_included_in_discount):
            discounted_lines_by_voucher = [cheapest_line]  # 仅保留最便宜的一行

    for line_info in lines_info:
        if line_info in discounted_lines_by_voucher:
            line_info.voucher = voucher
            line_info.voucher_code = voucher_info.voucher_code
```

### 2.4 与优惠券类型的组合关系

| 优惠券类型 | apply_once_per_order | 折扣应用方式 |
|-----------|---------------------|-------------|
| `ENTIRE_ORDER` | `False` | 整单折扣，基于 subtotal.net 计算，分摊到各行 |
| `ENTIRE_ORDER` | `True` | **行级处理**，仅应用于最便宜的一行 |
| `SPECIFIC_PRODUCT` | `False` | 应用于所有符合范围的行，每行单独计算 |
| `SPECIFIC_PRODUCT` | `True` | 应用于符合范围的行中最便宜的一行 |
| `SHIPPING` | N/A | 仅影响运费，不涉及商品行 |

**重要**：当 `ENTIRE_ORDER` + `apply_once_per_order=True` 时，优惠券被视为 **line-level voucher**（见 `is_line_level_voucher()` 函数），**不与订单级促销互斥**（因为互斥判断只检查 `is_order_level_voucher()`）。

### 2.5 折扣金额计算
**代码位置**：`saleor/discount/utils/voucher.py:622-658`

```python
def calculate_line_discount_amount_from_voucher(line_info, total_price):
    voucher = line_info.voucher
    if voucher.apply_once_per_order:
        # 整个优惠券金额只应用于这一行
        voucher_discount_amount = voucher.get_discount_amount_for(
            total_price, line_info.channel
        )
        discount_amount = min(voucher_discount_amount, total_price)
    else:
        # SPECIFIC_PRODUCT 类型，按行计算
        unit_price = line_info.variant_discounted_price
        unit_discount = voucher.get_discount_amount_for(unit_price, line_info.channel)
        discount_amount = unit_discount * line_info.line.quantity
    return discount_amount
```

### 2.6 对最终价格的影响示例

假设订单：
- 商品 A：单价 $100，数量 1
- 商品 B：单价 $50，数量 1
- 优惠券：$30 固定金额，`apply_once_per_order=True`，未指定适用范围

**计算过程**：
1. 范围筛选：两行都符合（未指定范围，且都不是赠品）
2. 最便宜行：商品 B（$50）
3. 折扣金额：min($30, $50) = $30
4. 最终：商品 A $100，商品 B $20，小计 $120

如果 `apply_once_per_order=False`（整单券）：
- 折扣 $30 基于 subtotal $150 计算
- 按比例分摊：商品 A 分摊 $20，商品 B 分摊 $10
- 最终：商品 A $80，商品 B $40，小计 $120

两种方式订单小计相同，但**行级价格分布不同**，影响退款等后续操作。

---

## 三、min_spent 在含税与未含税渠道下的校验差异

### 3.1 核心判定逻辑
**代码位置**：`saleor/discount/utils/voucher.py:307-324`

```python
def validate_voucher_in_order(order, lines, channel):
    subtotal = order.subtotal
    tax_configuration = channel.tax_configuration
    prices_entered_with_tax = tax_configuration.prices_entered_with_tax
    value = subtotal.gross if prices_entered_with_tax else subtotal.net
    validate_voucher(order.voucher, value, quantity, customer_email, channel, order.user)
```

### 3.2 两种渠道模式对比

#### 3.2.1 含税渠道 (`prices_entered_with_tax = True`)
- 商品价格在录入时已包含税费（典型场景：B2C 电商，面向最终消费者）
- `min_spent` 校验使用 **`subtotal.gross`**（含税小计）
- 客户看到的价格就是最终支付价格，校验门槛与客户感知一致

#### 3.2.2 未含税渠道 (`prices_entered_with_tax = False`)
- 商品价格在录入时不含税费（典型场景：B2B 电商，面向企业客户）
- `min_spent` 校验使用 **`subtotal.net`**（未税小计）
- 税费在结算时单独计算，校验门槛基于商品本身价值

### 3.3 Checkout 与 Order 的差异

#### 3.3.1 Checkout 路径
**代码位置**：`saleor/discount/utils/voucher.py:280-304`

```python
def validate_voucher_for_checkout(manager, voucher, checkout_info, lines):
    subtotal = base_calculations.base_checkout_subtotal(
        lines, checkout_info.channel, checkout_info.checkout.currency
    )
    validate_voucher(voucher, subtotal, quantity, customer_email, checkout_info.channel, checkout_info.user)
```

**关键点**：`base_checkout_subtotal()` 返回的是 **未税价格**（Money 类型，非 TaxedMoney），**未区分**含税/未税渠道。

#### 3.3.2 Order 路径
**代码位置**：`saleor/discount/utils/voucher.py:307-324`

```python
def validate_voucher_in_order(order, lines, channel):
    subtotal = order.subtotal  # TaxedMoney 类型
    tax_configuration = channel.tax_configuration
    prices_entered_with_tax = tax_configuration.prices_entered_with_tax
    value = subtotal.gross if prices_entered_with_tax else subtotal.net
    validate_voucher(order.voucher, value, quantity, customer_email, channel, order.user)
```

**关键点**：根据渠道配置动态选择 `gross` 或 `net`。

### 3.4 不一致性说明

> **重要差异**：Checkout 路径的 `min_spent` 校验**始终使用未税价格**，而 Order 路径根据渠道配置选择含税或未税价格。

这意味着在含税渠道下：
- 同一订单，Checkout 阶段和 Order 阶段的 `min_spent` 校验基准可能不同
- 如果订单含税小计刚好超过门槛，但未税小计未达门槛，可能出现：
  - Checkout 阶段校验失败（使用 net）
  - 但按业务逻辑（含税价格）应该通过

### 3.5 对最终折扣结果的影响

#### 场景示例
- 渠道配置：`prices_entered_with_tax = True`（含税），税率 20%
- 优惠券 `min_spent = $120`
- 订单商品：未税 $100，含税 $120

| 阶段 | 校验基准 | 金额 | 结果 |
|------|---------|------|------|
| Checkout | `base_checkout_subtotal`（net） | $100 | ❌ 未达 $120 门槛 |
| Order | `subtotal.gross` | $120 | ✅ 达到门槛 |

**业务影响**：
- 客户在结账页面看到"优惠券不满足最低消费条件"的错误
- 但实际上订单含税金额已满足条件
- 可能导致客户流失或客诉

#### 对折扣计算的直接影响
`min_spent` 校验失败会：
1. 抛出 `NotApplicable` 异常
2. 优惠券无法应用
3. 订单不享受该优惠券折扣
4. 订单最终价格 = 原价 - 其他适用折扣

### 3.6 Voucher 模型层校验
**代码位置**：`saleor/discount/models.py:178-186`

```python
def validate_min_spent(self, value: Money, channel: Channel):
    voucher_channel_listing = self.channel_listings.filter(channel=channel).first()
    if not voucher_channel_listing:
        raise NotApplicable("This voucher is not assigned to this channel")
    min_spent = voucher_channel_listing.min_spent
    if min_spent and value < min_spent:
        target = min_spent.quantize()
        msg = f"This offer is only valid for orders over {target.amount} {target.currency}."
        raise NotApplicable(msg, min_spent=min_spent)
```

模型层校验本身是**中性**的，只比较传入的 `value` 和 `min_spent`，不关心是 gross 还是 net。**差异完全由调用方传入的 value 决定**。

---

## 四、关键互斥关系总结

### 4.1 折扣类型互斥矩阵

| 已存在↓ \ 可叠加→ | 目录促销 | 订单级促销 | 优惠券(整单) | 优惠券(行级) | 手动折扣(行) | 手动折扣(单) |
|------------------|---------|-----------|-------------|-------------|-------------|-------------|
| 目录促销 | - | ✅ | ✅ | ✅ | ✅ | ✅ |
| 订单级促销 | ✅ | - | ❌ | ⚠️ | ❌ | ❌ |
| 优惠券(整单) | ✅ | ❌ | - | ❌ | ❌ | ❌ |
| 优惠券(行级) | ✅ | ⚠️ | ❌ | - | ❌ | ❌ |
| 手动折扣(行) | ❌ | ❌ | ❌ | ❌ | - | ❌ |
| 手动折扣(单) | ✅ | ❌ | ❌ | ❌ | ❌ | - |

符号说明：
- ✅ 可叠加
- ❌ 不可叠加（后者被清除/跳过）
- ⚠️ 视具体情况（`ENTIRE_ORDER` + `apply_once_per_order=True` 视为行级，不与订单级促销互斥）

### 4.2 清除/跳过逻辑触发条件

| 场景 | Checkout 路径 | Draft Order 路径 |
|------|--------------|------------------|
| 有优惠券时处理订单级促销 | `_clear_checkout_discount()` 清除 | 直接 `return` 跳过 |
| 行有手动折扣时处理行级优惠券 | N/A（Checkout 不处理优惠券折扣对象） | 移除该行的优惠券折扣 |
| 行有手动折扣时处理行级目录促销 | 移除该行的目录促销折扣 | N/A（Draft Order 刷新时目录促销已在价格中） |
| 行是赠品 | 移除该行所有折扣 | 移除该行优惠券折扣 |

---

## 五、计算流程完整决策树

```
开始计算订单价格
    │
    ├─ 应用目录级促销（始终应用，选最优规则）
    │
    ├─ 检查是否有手动折扣
    │   ├─ 行级手动折扣：移除该行其他所有折扣
    │   └─ 订单级手动折扣：跳过所有促销和优惠券
    │
    ├─ 检查是否有优惠券
    │   ├─ 有优惠券
    │   │   ├─ 判定优惠券类型
    │   │   │   ├─ 整单券 (ENTIRE_ORDER + !apply_once_per_order)
    │   │   │   │   └─ 基于 subtotal 计算折扣，分摊到各行
    │   │   │   ├─ 行级券 (SPECIFIC_PRODUCT 或 ENTIRE_ORDER + apply_once_per_order)
    │   │   │   │   ├─ 筛选适用范围（排除赠品）
    │   │   │   │   ├─ apply_once_per_order=True ?
    │   │   │   │   │   ├─ 是：选取最便宜的一行
    │   │   │   │   │   └─ 否：所有符合条件的行
    │   │   │   │   └─ 计算每行折扣
    │   │   │   └─ 运费券 (SHIPPING)
    │   │   │       └─ 基于运费计算折扣
    │   │   └─ 跳过订单级促销
    │   │
    │   └─ 无优惠券
    │       └─ 尝试应用订单级促销
    │           ├─ 基于 base_subtotal（仅含目录促销）判定资格
    │           ├─ 选最优规则（折扣最大或赠品价值最高）
    │           └─ 是赠品？添加赠品行 : 计算折扣并分摊
    │
    └─ 计算最终价格
结束
```
