# Saleor 优惠券(Voucher)与促销(Promotion)叠加与冲突分析

## 一、核心数据模型概览

### 1.1 折扣类型枚举 `DiscountType`

定义于 `saleor/discount/__init__.py:18`：

| 类型 | 值 | 说明 |
|------|-----|------|
| `SALE` | `sale` | 旧版Sale（已弃用，迁移为Promotion） |
| `PROMOTION` | `promotion` | 目录级促销（Catalogue Promotion），作用于商品行 |
| `ORDER_PROMOTION` | `order_promotion` | 订单级促销，作用于整个订单 |
| `VOUCHER` | `voucher` | 优惠券 |
| `MANUAL` | `manual` | 人工折扣（Staff手工添加） |

### 1.2 优惠券类型 `VoucherType`

定义于 `saleor/discount/__init__.py:34`：

| 类型 | 说明 |
|------|------|
| `ENTIRE_ORDER` | 整单折扣，可设置 `apply_once_per_order` 变为只减最便宜商品 |
| `SHIPPING` | 运费折扣 |
| `SPECIFIC_PRODUCT` | 指定商品/分类/集合/变体折扣 |

### 1.3 促销类型 `PromotionType` & 奖励类型

- `PromotionType.CATALOGUE` — 目录促销，作用于商品行单价
- `PromotionType.ORDER` — 订单促销，作用于订单小计
- `RewardType.SUBTOTAL_DISCOUNT` — 小计折扣奖励
- `RewardType.GIFT` — 赠品奖励

### 1.4 折扣模型继承链

```
BaseDiscount (抽象)
  ├── OrderDiscount          ← 订单级折扣 (VOUCHER, ORDER_PROMOTION, MANUAL)
  ├── OrderLineDiscount      ← 行级折扣 (PROMOTION, VOUCHER, MANUAL)
  ├── CheckoutDiscount       ← 结账级折扣
  └── CheckoutLineDiscount   ← 结账行级折扣
```

关键字段：
- `unique_type` — 行级折扣的唯一约束字段，确保同一行每种折扣类型只存一条记录
- `promotion_rule` — 关联促销规则
- `voucher` / `voucher_code` — 关联优惠券
- `amount_value` — 实际折扣金额（数量 × 单价折扣）
- `value` / `value_type` — 折扣值和类型（fixed/percentage）

---

## 二、适用判定逻辑

### 2.1 优惠券资格验证

入口函数 `validate_voucher`（`saleor/discount/utils/voucher.py:327`）：

1. **最低消费验证** `validate_min_spent` — 小计 >= `voucher_channel_listing.min_spent`
2. **最低商品数量验证** `validate_min_checkout_items_quantity` — 商品数量 >= `min_checkout_items_quantity`
3. **每人限用一次验证** `validate_once_per_customer` — `VoucherCustomer` 表中无该邮箱记录
4. **仅限员工验证** `validate_only_for_staff` — 用户为 staff

对于 Checkout 场景（`saleor/checkout/utils.py:280`），上述验证发生在 `validate_voucher_for_checkout` 中，其中 `subtotal` 的计算已包含目录促销折扣（通过 `base_calculations.base_checkout_subtotal`）。

### 2.2 促销规则资格验证

#### 目录促销（Catalogue Promotion）

- **时间有效性**：`Promotion.objects.active(date)` 过滤 `start_date <= now <= end_date`
- **渠道匹配**：通过 `PromotionRule.channels` M2M 关联渠道
- **商品匹配**：通过 `PromotionRule.variants` M2M 关联变体，或通过 `catalogue_predicate` JSON 字段动态匹配

获取最佳目录促销折扣的核心函数 `get_best_promotion_discount`（`saleor/discount/utils/promotion.py:116`）：
- 遍历所有适用规则，计算每条规则的折扣金额
- 选择折扣金额最大的那条规则（`max` over `price - discount(price)`）
- **每条变体只应用一条最优目录促销规则**（非叠加）

#### 订单促销（Order Promotion）

入口函数 `create_discount_objects_for_order_promotions`（`saleor/discount/utils/promotion.py:718`）：

1. **获取适用规则**：`fetch_promotion_rules_for_checkout_or_order` 遍历所有 `order_predicate` 非空的规则，用 `filter_qs_by_predicate` 匹配订单/结账对象
2. **选择最优规则**：`get_best_rule` 比较所有适用规则（包括赠品规则），选择折扣金额最大的
3. **赠品规则特殊处理**：`_get_best_gift_reward` 检查库存可用性，选最贵的可用赠品
4. **订单级促销也只应用一条最优规则**

### 2.3 优惠券与促销的前置互斥判定

这是最关键的冲突检测点。**核心原则：订单级促销与优惠券不共存。**

#### Checkout 场景

在 `saleor/checkout/utils.py:747` `add_voucher_to_checkout`：
```python
# 当添加优惠券时，显式删除订单促销折扣和赠品
CheckoutDiscount.objects.filter(
    checkout=checkout_info.checkout,
    type=DiscountType.ORDER_PROMOTION,
).delete()
delete_gift_line(checkout_info.checkout, lines)
```

在 `saleor/discount/utils/checkout.py:227` `create_checkout_discount_objects_for_order_promotions`：
```python
# 订单促销只在未设置优惠券时才生效
if checkout.voucher_code:
    _clear_checkout_discount(checkout_info, lines_info, save)
    return None
```

#### Order 场景

在 `saleor/discount/utils/order.py:189` `create_order_discount_objects_for_order_promotions`：
```python
# 如果设置了优惠券或人工折扣，则跳过订单促销
if order.voucher_code or order.discounts.filter(type=DiscountType.MANUAL):
    _clear_order_discount(order, lines_info)
    return
```

---

## 三、价格计算与叠加顺序

### 3.1 Checkout 行价格计算流程

核心函数 `calculate_base_line_total_price`（`saleor/checkout/base_calculations.py:38`）：

```
1. total_price = undiscounted_unit_price × quantity
2. 遍历 line_info.discounts 中的每个折扣对象:
     total_price -= discount.amount_value          ← 目录促销折扣（先扣）
3. 如果 line_info.voucher 存在:
     total_price -= calculate_line_discount_amount_from_voucher(
         line_info, total_price                     ← 注意：基于已扣目录促销后的价格
     )                                              ← 优惠券折扣（后扣）
4. 返回 quantize(total_price)
```

**关键：优惠券折扣基于已叠加目录促销后的价格计算**，两者是叠加关系。

### 3.2 优惠券行级折扣的具体计算

函数 `calculate_line_discount_amount_from_voucher`（`saleor/discount/utils/voucher.py:622`）：

| 优惠券类型 | 计算方式 |
|-----------|----------|
| `SPECIFIC_PRODUCT` + 非 `apply_once_per_order` + percentage | 对 total_price 算百分比折扣，不超过 total_price |
| `SPECIFIC_PRODUCT` + 非 `apply_once_per_order` + fixed | 对单价算固定折扣，乘以数量，不超过 total_price |
| `apply_once_per_order` (任何类型) | 只对单价应用一次（不乘数量），不超过单价 |

### 3.3 Checkout 小计与总计

#### 小计（含行级折扣，不含整单折扣）

```
base_checkout_subtotal = Σ calculate_base_line_total_price(line)
```

#### 总计（含所有折扣）

`checkout_total`（`saleor/checkout/base_calculations.py:199`）：
```
1. subtotal = base_checkout_subtotal           ← 含目录促销+行级优惠券
2. shipping_price = base_checkout_delivery_price ← 含运费优惠券
3. discount = checkout.discount                 ← 整单优惠券或订单促销
4. 如果存在订单促销或整单优惠券:
     subtotal = max(0, subtotal - discount)     ← 整单级折扣最后扣
5. total = subtotal + shipping_price
```

### 3.4 Order 行价格计算

Order 行的基础价格存储在 `OrderLine.base_unit_price_amount` 上，该值已经过目录促销和行级优惠券扣减。订单促销/整单优惠券通过 `propagate_order_discount_on_order_prices` 按比例分摊到每行。

### 3.5 计算管道的完整执行顺序

**Checkout 重新计算管道**（`recalculate_checkout_discounts`，`saleor/checkout/utils.py:92`）：

```
Step 1: create_checkout_line_discount_objects_for_catalogue_promotions(lines)
        └── 处理目录促销行折扣（增删改）
Step 2: recalculate_checkout_discount(manager, checkout_info, lines)
        ├── 验证优惠券资格
        ├── 计算优惠券折扣金额
        └── 设置 checkout.discount
Step 3: create_checkout_discount_objects_for_order_promotions(checkout_info, lines, save=True)
        └── 如果无优惠券，计算并设置订单促销折扣
```

**Order 重新计算管道**（`fetch_order_prices_if_expired`，`saleor/order/calculations.py:192`）：

```
Step 1: refresh_order_base_prices_and_discounts(order, line_ids, lines)
        ├── refresh_order_line_discount_objects_for_catalogue_promotions
        │     └── 刷新目录促销行折扣
        ├── refresh_manual_line_discount_object
        │     └── 刷新人工行折扣
        ├── reattach_apply_once_per_order_voucher_info (如果需要)
        ├── create_or_update_line_discount_objects_from_voucher
        │     └── 刷新优惠券行折扣
        └── update_unit_discount_data_on_order_lines_info
              └── 汇总行折扣到 OrderLine.unit_discount_*
Step 2: process_order_promotion(order, lines_info)
        └── handle_order_promotion → create_order_discount_objects_for_order_promotions
              └── 如果无优惠券，计算并设置订单促销折扣
Step 3: calculate_prices(order, lines)
        ├── base_order_subtotal (含行级折扣)
        ├── base_order_total (含行级折扣+运费)
        └── calculate_prices (应用整单折扣，计算税金)
```

---

## 四、冲突仲裁规则总结

### 4.1 互斥矩阵

| 折扣A ↓ \ 折扣B → | 目录促销 | 行级优惠券 | 整单优惠券 | 运费优惠券 | 订单促销 | 人工行折扣 | 人工整单折扣 |
|---------------------|---------|-----------|-----------|-----------|---------|-----------|-------------|
| **目录促销** | - | ✅ 叠加 | ✅ 叠加 | ✅ 叠加 | ✅ 叠加 | ❌ 互斥 | ✅ 叠加 |
| **行级优惠券** | ✅ 叠加 | - | ✅ 叠加 | ✅ 叠加 | ✅ 叠加 | ❌ 互斥 | ✅ 叠加 |
| **整单优惠券** | ✅ 叠加 | ✅ 叠加 | - | ✅ 叠加 | ❌ 互斥 | ✅ 叠加 | ❌ 互斥 |
| **运费优惠券** | ✅ 叠加 | ✅ 叠加 | ✅ 叠加 | - | ✅ 叠加 | ✅ 叠加 | ✅ 叠加 |
| **订单促销** | ✅ 叠加 | ✅ 叠加 | ❌ 互斥 | ✅ 叠加 | - | ✅ 叠加 | ❌ 互斥 |
| **人工行折扣** | ❌ 互斥 | ❌ 互斥 | ✅ 叠加 | ✅ 叠加 | ✅ 叠加 | - | ✅ 叠加 |
| **人工整单折扣** | ✅ 叠加 | ✅ 叠加 | ❌ 互斥 | ✅ 叠加 | ❌ 互斥 | ✅ 叠加 | - |

### 4.2 冲突仲裁的实现机制

#### 机制一：显式删除（添加时清除竞争方）

- **添加优惠券 → 删除订单促销**：`checkout/utils.py:791` 添加优惠券时删除所有 `ORDER_PROMOTION` 类型的 CheckoutDiscount 和赠品行
- **添加人工折扣 → 删除优惠券订单折扣**：`discount/utils/voucher.py:398` 如果人工折扣存在，删除优惠券订单折扣

#### 机制二：条件跳过（计算前判断资格）

- **订单促销计算前检查优惠券**：`discount/utils/order.py:189` 和 `discount/utils/checkout.py:227` 计算订单促销前检查 `voucher_code` 是否已设置
- **订单级优惠券计算前检查人工折扣**：`discount/utils/voucher.py:398` 如果人工折扣存在，跳过整单优惠券

#### 机制三：行级唯一约束

- `OrderLineDiscount` 的 `unique_orderline_discount_type` 约束确保：同一行同一 `unique_type` 只有一条记录
- `CheckoutLineDiscount` 的 `unique_checkoutline_discount_type` 约束同理
- 这防止同一行出现多条目录促销或多条优惠券折扣

#### 机制四：只取最优（Best-of-N 策略）

- **目录促销**：`get_best_promotion_discount` 取折扣金额最大的规则
- **订单促销**：`get_best_rule` 取折扣金额最大的规则（赠品按变体价格比较）
- **apply_once_per_order 优惠券**：`get_the_cheapest_line` 取最便宜的一行

### 4.3 叠加顺序中的优先级

当多种折扣同时存在时，**扣减顺序**决定了最终价格：

```
原始单价
  │
  ├── ① 目录促销（行级，固定或百分比，先扣）
  │
  ├── ② 行级优惠券（行级，基于已扣目录促销的价格再扣）
  │
  ├── ③ 人工行折扣（如存在，会清除①和②，独立计算）
  │
  ├── ④ 整单优惠券（基于前三步后的小计）
  │     或
  │     订单促销（基于前三步后的小计，与④互斥）
  │
  └── ⑤ 运费优惠券（基于原始运费）
        和
        人工整单折扣（基于小计+运费，与④互斥）
```

---

## 五、关键代码文件索引

| 功能 | 文件路径 |
|------|---------|
| 折扣类型与枚举 | `saleor/discount/__init__.py` |
| 折扣数据模型 | `saleor/discount/models.py` |
| 折扣接口(DiscountInfo等) | `saleor/discount/interface.py` |
| 优惠券工具函数 | `saleor/discount/utils/voucher.py` |
| 促销工具函数 | `saleor/discount/utils/promotion.py` |
| 订单折扣计算 | `saleor/discount/utils/order.py` |
| 结账折扣计算 | `saleor/discount/utils/checkout.py` |
| 共享工具 | `saleor/discount/utils/shared.py` |
| Checkout 基础价格计算 | `saleor/checkout/base_calculations.py` |
| Order 基础价格计算 | `saleor/order/base_calculations.py` |
| Checkout 折扣编排 | `saleor/checkout/utils.py` (`recalculate_checkout_discounts`, `add_voucher_to_checkout`) |
| Order 折扣编排 | `saleor/order/calculations.py` (`fetch_order_prices_if_expired`) |
| 结账→订单转换时的优惠券处理 | `saleor/checkout/complete_checkout.py` (`_process_voucher_data_for_order`) |

---

## 六、特殊场景分析

### 6.1 订单完成时的优惠券冻结

当 Checkout 转换为 Order 时（`complete_checkout.py:_process_voucher_data_for_order`）：
- 优惠券数据被验证并增加使用计数
- 订单中的优惠券折扣信息被**反规范化**（denormalized），即使用当时的快照数据
- 后续即使优惠券条件变化（如折扣值修改），已完成订单中的折扣不会自动变化

### 6.2 草稿订单的价格刷新

草稿订单的 `should_refresh_prices` 标志触发价格重算：
- `refresh_order_base_prices_and_discounts` 会重新获取最新的渠道列表价格
- 目录促销折扣基于最新价格重新计算
- 优惠券折扣使用反规范化数据（`use_denormalized_data=True`）保持订单创建时的条件
- 订单促销重新基于最新条件评估

### 6.3 赠品促销与优惠券的交互

- 赠品行（`is_gift=True`）**不参与**行级优惠券折扣（`get_discounted_lines` 中跳过 `is_gift` 行）
- 添加优惠券时会删除现有赠品行（`delete_gift_line`）
- 优惠券优先于赠品促销

### 6.4 目录促销折扣的预计算

目录促销折扣通过 `VariantChannelListingPromotionRule` 预计算并存储在渠道列表上（`discounted_price_amount`），这意味着：
- 目录促销折扣不依赖于订单/结账上下文
- 当 `variants_dirty` 标志设置后，后台任务重新计算折扣价格
- 计算订单行折扣时直接使用预计算的 `discount_amount`

---

## 七、价格计算流程图示

```
┌─────────────────────────────────────────────────────────────────┐
│                     Checkout 价格重算                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  原始单价 × 数量                                                 │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────────┐                                        │
│  │ ① 目录促销折扣       │  从 VariantChannelListingPromotionRule │
│  │ (每条变体只取最优)    │  读取预计算的 discounted_price_amount │
│  └─────────────────────┘                                        │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────────┐                                        │
│  │ ② 行级优惠券折扣     │  SPECIFIC_PRODUCT / apply_once_per_order│
│  │ (基于已扣①的价格)    │  通过 voucher.get_discount_amount_for   │
│  └─────────────────────┘                                        │
│       │                                                         │
│       ▼                                                         │
│  行价格 → 汇总为小计 base_subtotal                               │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────────────────────┐                          │
│  │ ③ 整单级折扣（三选一，互斥）       │                          │
│  │   ├─ 整单优惠券 (ENTIRE_ORDER)    │                          │
│  │   ├─ 订单促销 (ORDER_PROMOTION)   │                          │
│  │   └─ 人工整单折扣 (MANUAL)        │                          │
│  │ 基于小计按比例分摊到各行           │                          │
│  └──────────────────────────────────┘                          │
│       │                                                         │
│       ▼                                                         │
│  运费 + 小计（已扣③） → 总计                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
