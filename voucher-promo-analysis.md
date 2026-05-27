# Saleor 优惠券(Voucher)与促销(Promotion)叠加与冲突分析

> **文档更新历史**：
> - **2026-05-27 第四次修订**：明确 `create_order_discount_objects_for_order_promotions` 中的双重门禁逻辑（`voucher_code` OR `DiscountType.MANUAL`），补充可回切/不可回切判定矩阵与状态图，修正 Checkout 侧/Order 侧门禁差异，补充测试断言验证。
> - **2026-05-27 第三次修订**：纠正 Draft Order 优惠券移除后的回切链路描述。原描述"仅清除 voucher_id"错误，实际通过 `clean_voucher`/`clean_voucher_code` 双向清理同步清空 `voucher` 和 `voucher_code` 两个字段，订单促销可正常回切。
> - **2026-05-27 第二次修订**：增加优惠券失效/移除后的状态转换分析，以及 ENTIRE_ORDER 与 apply_once_per_order 的类型分类和计价层级差异分析。
> - **2026-05-27 初版**：基础分析文档，涵盖折扣类型、适用判定、价格叠加与冲突仲裁。

---

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

这是最关键的冲突检测点。**核心原则：所有类型的优惠券与订单促销（ORDER_PROMOTION）均不共存。**

#### 判定依据：`voucher_code` 字段

`Checkout.voucher_code` 和 `Order.voucher_code` 都是 `CharField(max_length=255, null=True)` 类型的简单字符串字段。**无论优惠券是什么类型**（整单、运费、行级、apply_once_per_order），只要优惠券被应用，该字段就会被设置。订单促销的跳过逻辑直接检查此字段：

```python
if checkout.voucher_code:          # checkout.py:227
if order.voucher_code or ...:      # order.py:189
```

这是一个类型无关的检查——仅凭 `voucher_code` 非空就跳过订单促销，不区分优惠券的具体类型。

#### Checkout 场景

在 `saleor/checkout/utils.py:747` `add_voucher_to_checkout`：
```python
# 当添加优惠券时，显式删除订单促销折扣和赠品
# 此操作对所有优惠券类型都执行
checkout.voucher_code = voucher_code.code          # ← 所有类型都设置此字段
...
CheckoutDiscount.objects.filter(
    checkout=checkout_info.checkout,
    type=DiscountType.ORDER_PROMOTION,
).delete()
delete_gift_line(checkout_info.checkout, lines)
```

在 `saleor/discount/utils/checkout.py:214` `create_checkout_discount_objects_for_order_promotions`：
```python
# 订单促销只在未设置优惠券时才生效
# 检查的是 checkout.voucher_code，不区分优惠券类型
if checkout.voucher_code:
    _clear_checkout_discount(checkout_info, lines_info, save)
    return None
```

在 `saleor/checkout/utils.py:626` `recalculate_checkout_discount`：
```python
# 先处理优惠券（所有类型），再尝试订单促销——但订单促销会因 voucher_code 已设置而跳过
if voucher := checkout_info.voucher:
    ...
    checkout.save(...)
# 此调用会因 checkout.voucher_code 已被设置而直接返回 None
create_checkout_discount_objects_for_order_promotions(checkout_info, lines, save=True)
```

#### Order 场景

在 `saleor/discount/utils/order.py:181` `create_order_discount_objects_for_order_promotions`：
```python
# 如果设置了优惠券（任何类型）或人工折扣，则跳过订单促销
# voucher_code 字段在所有优惠券类型下均非空
if order.voucher_code or order.discounts.filter(type=DiscountType.MANUAL):
    _clear_order_discount(order, lines_info)
    return
```

在 `saleor/checkout/complete_checkout.py:741` 结账转订单时：
```python
# _process_voucher_data_for_order 对所有优惠券类型都返回 voucher_code
# 该值随后通过 **order_data 写入 Order.voucher_code
order_data.update(_process_voucher_data_for_order(checkout_info))
# _process_voucher_data_for_order 返回 {"voucher": voucher, "voucher_code": voucher_code.code}
# 不区分优惠券类型
```

#### 各类型优惠券对订单促销的影响总结

| 优惠券类型 | `voucher_code` 是否设置 | 是否阻断订单促销 | 代码依据 |
|-----------|----------------------|----------------|---------|
| `ENTIRE_ORDER`（非 apply_once_per_order） | ✅ 是 | ✅ 阻断 | `checkout/utils.py:767` 设置 `voucher_code` |
| `ENTIRE_ORDER` + `apply_once_per_order=True` | ✅ 是 | ✅ 阻断 | 同上，`voucher_code` 照常设置 |
| `SHIPPING` | ✅ 是 | ✅ 阻断 | 同上 |
| `SPECIFIC_PRODUCT` | ✅ 是 | ✅ 阻断 | 同上 |

**结论：不存在任何类型的优惠券可以与订单促销并存。**

但注意：优惠券与**目录促销（Catalogue Promotion）**可以叠加，因为目录促销作用于行级单价，而订单促销作用于订单小计，两者不共享互斥检查逻辑。

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
| **行级优惠券** | ✅ 叠加 | - | ✅ 叠加 | ✅ 叠加 | ❌ 互斥 | ❌ 互斥 | ✅ 叠加 |
| **整单优惠券** | ✅ 叠加 | ✅ 叠加 | - | ✅ 叠加 | ❌ 互斥 | ✅ 叠加 | ❌ 互斥 |
| **运费优惠券** | ✅ 叠加 | ✅ 叠加 | ✅ 叠加 | - | ❌ 互斥 | ✅ 叠加 | ✅ 叠加 |
| **订单促销** | ✅ 叠加 | ❌ 互斥 | ❌ 互斥 | ❌ 互斥 | - | ✅ 叠加 | ❌ 互斥 |
| **人工行折扣** | ❌ 互斥 | ❌ 互斥 | ✅ 叠加 | ✅ 叠加 | ✅ 叠加 | - | ✅ 叠加 |
| **人工整单折扣** | ✅ 叠加 | ✅ 叠加 | ❌ 互斥 | ✅ 叠加 | ❌ 互斥 | ✅ 叠加 | - |

**互斥依据说明：**

- **任意优惠券 ❌ 订单促销**：`checkout.voucher_code` / `order.voucher_code` 在所有优惠券类型下均被设置，订单促销计算前统一检查此字段并跳过（`checkout.py:227`、`order.py:189`）
- **人工整单折扣 ❌ 订单促销**：Order 侧双重门禁 `order.discounts.filter(type=DiscountType.MANUAL)` 存在时跳过订单促销（`order.py:189`），Checkout 侧无此门禁
- **人工整单折扣 ❌ 整单优惠券**：添加人工折扣时 `order.discounts.exclude(voucher__type=VoucherType.SHIPPING).delete()` 删除优惠券折扣（`order/utils.py:762`）；添加优惠券时 `is_manual_discount` 检查跳过优惠券折扣（`voucher.py:398`）
- **目录促销 ✅ 所有优惠券**：目录促销作用于行级单价（通过 `VariantChannelListingPromotionRule.discounted_price_amount` 预计算），与订单促销的订单小计级检查不冲突；行级优惠券在 `calculate_base_line_total_price` 中基于已扣目录促销的价格继续扣减
- **人工行折扣 ❌ 所有行级折扣**：`unique_type` 唯一约束 + 显式删除逻辑

### 4.2 冲突仲裁的实现机制

#### 机制一：双重门禁（订单促销专属）

Order 侧 `create_order_discount_objects_for_order_promotions`（`order.py:189`）：
```python
if order.voucher_code or order.discounts.filter(type=DiscountType.MANUAL):
    _clear_order_discount(order, lines_info)
    return
```

Checkout 侧 `create_checkout_discount_objects_for_order_promotions`（`checkout.py:227`）：
```python
if checkout.voucher_code:
    _clear_checkout_discount(checkout_info, lines_info, save)
    return None
```

| 门禁条件 | Checkout 侧 | Order 侧 |
|---------|------------|---------|
| `voucher_code` 非空 | ✅ 阻断 | ✅ 阻断 |
| MANUAL 订单折扣存在 | 不适用 | ✅ 阻断 |

#### 机制二：显式删除（添加时清除竞争方）

- **添加优惠券 → 删除订单促销**：`checkout/utils.py:791` 添加优惠券时删除所有 `ORDER_PROMOTION` 类型的 CheckoutDiscount 和赠品行
- **添加人工整单折扣 → 删除所有其他订单折扣**：`order/utils.py:762` `order.discounts.exclude(voucher__type=VoucherType.SHIPPING).delete()` 删除 VOUCHER、ORDER_PROMOTION、旧 MANUAL 折扣，仅保留运费优惠券
- **添加人工行折扣 → 删除同行的目录促销和优惠券行折扣**：`order/utils.py:856` `_remove_invalid_discounts_for_adding_manual` 删除同行所有非 MANUAL 折扣

#### 机制三：条件跳过（计算前判断资格）

- **订单促销计算前检查双重门禁**：`order.py:189` 检查 `voucher_code` 和 MANUAL 折扣
- **Checkout 侧订单促销计算前检查优惠券**：`checkout.py:227` 仅检查 `voucher_code`
- **订单级优惠券计算前检查人工折扣**：`voucher.py:398` `is_manual_discount` 为 True 时删除优惠券折扣

#### 机制四：行级唯一约束

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
  │     可与所有类型优惠券叠加
  │
  ├── ② 行级优惠券（行级，基于已扣目录促销的价格再扣）
  │     可与目录促销叠加，但阻断订单促销
  │
  ├── ③ 人工行折扣（如存在，会清除①和②，独立计算）
  │
  ├── ④ 整单级折扣（三选一，互斥，有优先级）
  │     ├─ 整单优惠券 (ENTIRE_ORDER)      ──┐
  │     ├─ 订单促销 (ORDER_PROMOTION)      ├── 三者互斥
  │     └─ 人工整单折扣 (MANUAL)          ──┘
  │     优先级：整单优惠券 > 订单促销（voucher_code 检查使订单促销跳过）
  │             人工整单折扣 > 整单优惠券（manual 检查使整单优惠券被删除）
  │
  └── ⑤ 运费优惠券（基于原始运费）
        可与目录促销、行级优惠券叠加，但阻断订单促销
```

**优先级规则总结：**

| 冲突场景 | 仲裁结果 | 代码位置 |
|---------|---------|---------|
| 任意优惠券 vs 订单促销 | **优惠券胜**，订单促销被跳过/删除 | `checkout.py:227`、`order.py:189`、`checkout/utils.py:791` |
| 整单优惠券 vs 人工整单折扣 | **人工胜**，优惠券订单折扣被删除 | `voucher.py:398` |
| 行级优惠券 vs 目录促销 | **叠加**，行级优惠券基于已扣目录促销的价格计算 | `base_calculations.py:38` |
| 人工行折扣 vs 目录促销/行级优惠券 | **人工胜**，行级折扣被清除 | `checkout.py:143`、`order.py:147` |
| 多条目录促销规则 | **只取最优**，折扣金额最大的那条 | `promotion.py:116` |
| 多条订单促销规则 | **只取最优**，折扣金额最大的那条 | `promotion.py:718` |

---

## 五、优惠券失效/移除后的状态转换与订单促销回切

### 5.1 Checkout 侧：自动回切流程

当优惠券因资格校验失败（最低消费不满足、优惠券过期、商品不再适用等）而被移除时，系统会自动切换到订单促销。这是一个**无状态自动回切**机制。

#### 状态转换链路

```
触发重算 (recalculate_checkout_discounts)
  │
  ├─ Step 1: create_checkout_line_discount_objects_for_catalogue_promotions
  │     └── 目录促销行折扣独立处理，不受优惠券影响
  │
  └─ Step 2: recalculate_checkout_discount
        │
        ├─ 有 voucher? ──→ check_voucher_for_checkout
        │                     │
        │                     ├─ 校验通过 → 保留优惠券
        │                     │
        │                     └─ 校验失败 (NotApplicable)
        │                           │
        │                           └─ remove_voucher_from_checkout
        │                                 ├─ checkout.voucher_code = None
        │                                 ├─ checkout.discount_name = None
        │                                 ├─ checkout.discount_amount = Decimal(0)
        │                                 └─ checkout.save(update_fields=[...])
        │
        └─ 无 voucher → remove_voucher_from_checkout (幂等)
              │
              └─ Step 3: create_checkout_discount_objects_for_order_promotions
                    │
                    ├─ checkout.voucher_code == None → 不再跳过
                    │
                    ├─ _set_checkout_base_prices (设置基准价格，含目录促销)
                    │
                    ├─ create_discount_objects_for_order_promotions
                    │     └── 计算并应用订单促销折扣
                    │
                    └── 保存折扣信息到 checkout
```

#### 关键代码路径

**`check_voucher_for_checkout`**（`saleor/checkout/utils.py:603`）：
```python
def check_voucher_for_checkout(voucher, manager, checkout_info, lines):
    checkout = checkout_info.checkout
    address = checkout_info.shipping_address or checkout_info.billing_address
    try:
        discount = get_voucher_discount_for_checkout(
            manager, voucher, checkout_info, lines, address,
        )
        return discount
    except NotApplicable:
        remove_voucher_from_checkout(checkout)   # ← 清空 voucher_code
        checkout_info.voucher = None
        return None
```

**`remove_voucher_from_checkout`**（`saleor/checkout/utils.py:839`）：
```python
def remove_voucher_from_checkout(checkout: Checkout):
    checkout.voucher_code = None
    checkout.discount_name = None
    checkout.translated_discount_name = None
    checkout.discount_amount = Decimal(0)
    checkout.save(update_fields=[
        "voucher_code", "discount_name", "translated_discount_name",
        "discount_amount", "currency", "last_change",
    ])
```

**`recalculate_checkout_discount`**（`saleor/checkout/utils.py:626`）：
```python
def recalculate_checkout_discount(manager, checkout_info, lines):
    checkout = checkout_info.checkout
    if voucher := checkout_info.voucher:
        discount = check_voucher_for_checkout(voucher, manager, checkout_info, lines)
        if discount:
            # ... 设置 checkout.discount 并保存
            checkout.save(...)
    else:
        remove_voucher_from_checkout(checkout)

    # 无论优惠券是否成功，始终尝试计算订单促销
    # 如果优惠券刚被移除，voucher_code 已为 None，订单促销会生效
    create_checkout_discount_objects_for_order_promotions(
        checkout_info, lines, save=True
    )
```

#### 回切时机

| 触发场景 | 触发函数 | 回切行为 |
|---------|---------|---------|
| 修改商品行（增删改） | `checkout/utils.py:76` `invalidate_checkout` | ✅ 自动重算并回切 |
| 修改配送地址 | `checkout/utils.py:402` `change_shipping_address_in_checkout` | ✅ 自动重算并回切 |
| 修改配送方式 | `checkout/utils.py` 配送相关函数 | ✅ 自动重算并回切 |
| 添加/移除优惠券 | `checkout/utils.py:682` `add_promo_code_to_checkout` | ✅ 直接操作后回切 |
| 优惠券过期 | `check_voucher_for_checkout` 中 `NotApplicable` | ✅ 校验失败自动回切 |

### 5.2 Order 侧：双重门禁逻辑与回切

与 Checkout 侧不同，Order 侧的订单促销被**两个独立条件**阻断，任意一个满足即跳过：

```python
# saleor/discount/utils/order.py:189
if order.voucher_code or order.discounts.filter(type=DiscountType.MANUAL):
    _clear_order_discount(order, lines_info)
    return
```

- **门禁 1**：`order.voucher_code` 非空 — 任意类型优惠券阻断
- **门禁 2**：`order.discounts` 中存在 `type=MANUAL` 的折扣对象 — 人工整单折扣阻断

两个条件是 **OR** 关系，必须**同时为假**时订单促销才能生效。

#### Checkout 侧 vs Order 侧的门禁差异

| 维度 | Checkout 侧 | Order 侧 |
|------|------------|---------|
| 门禁条件 | `checkout.voucher_code` 非空 | `order.voucher_code` 非空 **OR** 存在 MANUAL 订单折扣 |
| MANUAL 折扣是否阻断 | ❌ 不阻断（Checkout 无 MANUAL 订单折扣） | ✅ 阻断 |
| 门禁数量 | 1 个 | 2 个（双重门禁） |
| 代码位置 | `checkout.py:227` | `order.py:189` |

> **⚠️ 文档更正说明**：之前版本仅关注 `voucher_code` 门禁，忽略了 `DiscountType.MANUAL` 门禁。实际上，即使 `voucher_code` 为 None，只要存在人工整单折扣对象，订单促销同样无法回切。

#### 双重门禁下的状态空间

```
                        voucher_code
                     None      非空
                 ┌──────────┬──────────┐
  MANUAL折扣不存在 │ ✅ 可回切  │ ❌ 被门禁1阻断│
                 ├──────────┼──────────┤
  MANUAL折扣存在   │ ❌ 被门禁2阻断│ ❌ 被双重门禁阻断│
                 └──────────┴──────────┘
```

> 注意：`voucher_code` 非空 + MANUAL 折扣存在这个组合在实际中不太可能出现，
> 因为添加 MANUAL 折扣时会删除 VOUCHER 折扣对象（但不清除 `voucher_code`）。

#### MANUAL 整单折扣的阻断机制

**创建时：删除所有其他订单折扣对象**

`create_manual_order_discount`（`order/utils.py:749`）：
```python
def create_manual_order_discount(order, reason, value_type, value, ...):
    with transaction.atomic():
        # Manual order discount does not stack with other order-level discounts
        order.discounts.exclude(voucher__type=VoucherType.SHIPPING).delete()
        # ↑ 删除所有订单折扣对象（VOUCHER、ORDER_PROMOTION、MANUAL），仅保留运费优惠券
        ...
        order_discount = OrderDiscount.objects.create(
            type=DiscountType.MANUAL, ...
        )
```

**删除时：回切到优惠券（如有），否则回切到订单促销**

`remove_order_discount_from_order`（`order/utils.py:787`）：
```python
def remove_order_discount_from_order(order, order_discount):
    order_discount.delete()    # ← 删除 MANUAL 折扣对象，门禁2解除

    # Manual discounts take precedence over vouchers, overriding them when applied.
    # However, this does not entirely dissociate the voucher from the order.
    # If the manual discount is removed, the voucher is reevaluated.
    if order.voucher:
        create_or_update_discount_object_from_order_level_voucher(order)
        # ↑ 优惠券仍在 → 恢复优惠券折扣 → 但门禁1仍阻断订单促销
```

删除 MANUAL 折扣后通过 `invalidate_order_prices` 触发 `fetch_order_prices_if_expired`，完整链路如下：
```
OrderDiscountDelete mutation
  → remove_order_discount_from_order()
      ├─ 删除 MANUAL OrderDiscount → 门禁2解除
      └─ 如果 order.voucher 存在 → 恢复 VOUCHER 折扣对象
  → invalidate_order_prices() → should_refresh_prices = True
  → fetch_order_prices_if_expired()
      → create_order_discount_objects_for_order_promotions()
          ├─ 如果 order.voucher_code 非空 → 门禁1仍阻断 → ❌ 订单促销不回切
          └─ 如果 order.voucher_code 为空 → 门禁1解除 → ✅ 订单促销回切
```

#### 测试断言验证

**场景 1：删除 MANUAL 折扣 → 回切到订单促销**（`test_order_discount.py:1021`）
```python
# 前置：order 有 MANUAL 折扣，无优惠券，存在 order_promotion_rule
manual_discount = order.discounts.get()      # MANUAL 类型

# 调用 OrderDiscountDelete mutation
response = staff_api_client.post_graphql(ORDER_DISCOUNT_DELETE, variables)

# 验证：MANUAL 折扣已删除，订单促销折扣生效
with pytest.raises(manual_discount._meta.model.DoesNotExist):
    manual_discount.refresh_from_db()
order_discount = order.discounts.get()       # ORDER_PROMOTION 类型
assert order_discount.value == reward_value
```

**场景 2：删除 MANUAL 折扣 → 回切到优惠券（不回切到订单促销）**（`test_order_discount.py:1107`）
```python
# 前置：order 有 MANUAL 折扣 + voucher_code 非空 + voucher 存在
order.voucher_code = code
order.voucher = voucher
order.save()

# 调用 OrderDiscountDelete mutation
response = staff_api_client.post_graphql(ORDER_DISCOUNT_DELETE, variables)

# 验证：MANUAL 折扣已删除，优惠券折扣恢复，订单促销被门禁1阻断
voucher_discount = order.discounts.get()
assert voucher_discount.type == DiscountType.VOUCHER
```

**场景 3：删除 MANUAL 折扣 → 回切到赠品促销**（`test_order_discount.py:1063`）
```python
# 前置：order 有 MANUAL 折扣，无优惠券，存在 gift_promotion_rule
manual_discount = order.discounts.get()

# 调用 OrderDiscountDelete mutation
response = staff_api_client.post_graphql(ORDER_DISCOUNT_DELETE, variables)

# 验证：MANUAL 折扣已删除，赠品行恢复，无 OrderDiscount
assert not order.discounts.exists()
gift_line = order.lines.filter(is_gift=True).first()
assert gift_line is not None
```

#### 通过 draftOrderUpdate 移除优惠券时的回切

> **文档更正说明**：之前版本错误地描述为"仅清除 voucher_id"。实际通过 `clean_voucher`/`clean_voucher_code` 双向清理机制同步清空两个字段。

`draft_order_cleaner.py:46-105` 中实现了双向同步：
```python
def clean_voucher(voucher, channel, cleaned_input):
    if voucher is None:
        cleaned_input["voucher_code"] = None   # ← 同步清空
        return

def clean_voucher_code(voucher_code, channel, cleaned_input):
    if voucher_code is None:
        cleaned_input["voucher"] = None        # ← 双向同步
        return
```

`construct_instance(instance, cleaned_input)` 将两个字段都设为 None → 门禁1解除 → 订单促销可回切。

测试断言（`test_draft_order_update.py:2402`）：
```python
assert order.voucher_code is None
assert order.voucher is None
```

#### Order 侧回切的完整判定矩阵

| # | `voucher_code` | MANUAL 折扣 | `order.voucher` | 订单促销状态 | 回切路径 |
|---|---------------|------------|----------------|------------|---------|
| 1 | None | 不存在 | 任意 | ✅ 生效 | — |
| 2 | 非空 | 不存在 | 非空 | ❌ 门禁1阻断 | 通过 draftOrderUpdate 设置 `voucher=null` 或 `voucherCode=null` |
| 3 | None | 存在 | None | ❌ 门禁2阻断 | 通过 OrderDiscountDelete mutation 删除 MANUAL 折扣 |
| 4 | None | 存在 | 非空 | ❌ 门禁2阻断 | 通过 OrderDiscountDelete 删除 MANUAL → 恢复优惠券折扣（但门禁1仍阻断订单促销） |
| 5 | 非空 | 存在 | 非空 | ❌ 双重阻断 | 先 OrderDiscountDelete → 再 draftOrderUpdate 移除优惠券 |
| 6 | 非空 | 不存在 | None | ❌ 门禁1阻断 | 需显式清空 `voucher_code`（`voucher_id` 为 None 不够） |

#### 回切路径状态图

```
                  ┌───────────────────────────────┐
                  │   订单促销被阻断（不可回切）      │
                  │   条件: voucher_code 非空        │
                  │         OR MANUAL 折扣存在       │
                  └───────────────┬───────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
    ┌─────────▼─────────┐  ┌─────▼──────┐  ┌────────▼────────┐
    │ 仅门禁1阻断         │  │ 双重阻断    │  │ 仅门禁2阻断       │
    │ voucher_code 非空   │  │            │  │ MANUAL 折扣存在   │
    │ MANUAL 不存在       │  │            │  │ voucher_code 为空│
    └─────────┬─────────┘  └─────┬──────┘  └────────┬────────┘
              │                   │                   │
    draftOrderUpdate    ① OrderDiscountDelete  OrderDiscountDelete
    voucher=null         ② draftOrderUpdate      → 删除 MANUAL
    或 voucherCode=null    voucher=null          → 若voucher存在:
              │                   │              恢复 VOUCHER 折扣
              │                   │              (进入门禁1状态)
              ▼                   ▼                   │
    ┌─────────────────────────────────────────────────┘
    │
    ▼ (当 voucher_code=None 且 MANUAL 不存在时)
  ┌───────────────────┐
  │ ✅ 订单促销可回切    │
  │ fetch_order_prices │
  │ → create_order_    │
  │   discount_objects │
  │   _for_order_      │
  │   promotions()     │
  │ → 两个门禁均为假    │
  └───────────────────┘
```

#### 特殊场景：voucher_code 非空但 voucher_id 为 None

| voucher_code | voucher_id | MANUAL 折扣 | 订单促销 | 说明 |
|-------------|-----------|------------|---------|------|
| None | None | 不存在 | ✅ 生效 | 正常状态 |
| 非空 | 非空 | 不存在 | ❌ 阻断 | 优惠券完整存在 |
| 非空 | None | 不存在 | ❌ 阻断 | `voucher_code` 字段本身即为门禁条件，与 `voucher_id` 无关 |
| None | 非空 | 不存在 | ✅ 生效 | 理论上不应出现（voucher 存在但 code 为空），但若出现则不阻断 |

### 5.3 状态转换的设计差异

| 维度 | Checkout 侧 | Order 侧 |
|------|------------|---------|
| 优惠券校验失败 | 自动移除并回切订单促销 | 仅记录错误，不自动回切 |
| 门禁条件 | 仅 `voucher_code` | `voucher_code` **OR** MANUAL 折扣 |
| 显式移除优惠券 | 通过 `remove_voucher_from_checkout` 清空 `voucher_code` | 通过 `clean_voucher`/`clean_voucher_code` 同步清空两个字段 |
| 移除 MANUAL 折扣后 | 不适用（Checkout 无 MANUAL 订单折扣） | 通过 `OrderDiscountDelete` 删除折扣对象，然后 `invalidate_order_prices` 触发重算 |
| 订单促销回切 | 自动、无状态 | 需要显式触发 mutation 清除门禁条件 |
| 设计目标 | 实时计算，对用户友好 | 保持订单历史完整性，防止意外变更 |

---

### 5.4 Draft Order 优惠券移除与订单促销回切的完整状态转换图

```
初始状态: order.voucher = V1, order.voucher_code = "CODE1"
         order.discounts 包含 type=VOUCHER 的折扣对象
         订单促销被阻断（voucher_code 非空）

                │
                ▼
    用户调用 draftOrderUpdate mutation
    输入: {"voucher": null} 或 {"voucherCode": null}
                │
                ▼
    clean_input() 阶段
    ┌─────────────────────────────────────┐
    │ draft_order_cleaner.clean_voucher() │
    │ 或 clean_voucher_code()             │
    │  → cleaned_input["voucher"] = None   │
    │  → cleaned_input["voucher_code"] = None │
    └─────────────────────────────────────┘
                │
                ▼
    construct_instance(instance, cleaned_input)
    ┌─────────────────────────────────────┐
    │ instance.voucher_id = None          │
    │ instance.voucher_code = None        │
    └─────────────────────────────────────┘
                │
                ▼
    handle_order_voucher()
    ┌─────────────────────────────────────┐
    │ release_voucher_code_usage()        │
    │ → 释放优惠券使用次数                 │
    │ create_or_update_voucher_discount_  │
    │   objects_for_order()               │
    │ → 删除 type=VOUCHER 的 OrderDiscount│
    └─────────────────────────────────────┘
                │
                ▼
    order.save() → 两个字段都保存为 None
                │
                ▼
    order.should_refresh_prices = True
                │
                ▼
    fetch_order_prices_if_expired()
    ┌─────────────────────────────────────┐
    │ prepare_order_lines_for_refresh()   │
    │ process_order_promotion()           │
    │   → create_order_discount_objects_  │
    │       for_order_promotions()        │
    │       → 检查 order.voucher_code     │
    │         → 为 None → ✅ 不跳过        │
    │         → 计算并应用订单促销折扣      │
    └─────────────────────────────────────┘
                │
                ▼
最终状态: order.voucher = None, order.voucher_code = None
         order.discounts 包含 type=ORDER_PROMOTION 的折扣对象
         订单促销成功回切
```

---

## 六、ENTIRE_ORDER 与 apply_once_per_order 的类型分类与计价层级

### 6.1 类型分类：is_order_level_voucher vs is_line_level_voucher

Saleor 使用三个分类函数来确定优惠券的计价层级（`saleor/discount/utils/voucher.py:51-66`）：

```python
def is_order_level_voucher(voucher):
    return bool(
        voucher
        and voucher.type == VoucherType.ENTIRE_ORDER
        and not voucher.apply_once_per_order    # ← 关键：必须 NOT apply_once_per_order
    )

def is_shipping_voucher(voucher):
    return bool(voucher and voucher.type == VoucherType.SHIPPING)

def is_line_level_voucher(voucher):
    return voucher and (
        voucher.type == VoucherType.SPECIFIC_PRODUCT
        or voucher.apply_once_per_order          # ← 关键：apply_once_per_order 归入行级
    )
```

#### 分类结果矩阵

| `type` | `apply_once_per_order` | `is_order_level` | `is_shipping` | `is_line_level` | 实际计价层级 |
|--------|----------------------|-----------------|--------------|----------------|------------|
| `ENTIRE_ORDER` | `False` | ✅ | ❌ | ❌ | 订单级 |
| `ENTIRE_ORDER` | `True` | ❌ | ❌ | ✅ | 行级 |
| `SHIPPING` | 任意 | ❌ | ✅ | ❌ | 运费级 |
| `SPECIFIC_PRODUCT` | 任意 | ❌ | ❌ | ✅ | 行级 |

**核心结论**：`ENTIRE_ORDER + apply_once_per_order=True` 被分类为**行级**优惠券，与 `SPECIFIC_PRODUCT` 同属一类。

### 6.2 计价层级差异

#### 订单级 ENTIRE_ORDER（`apply_once_per_order=False`）

- **折扣对象存储**：`OrderDiscount`（订单级）或 `CheckoutDiscount`（结账级）
- **计算方式**：基于订单小计
- **计算入口**：`get_voucher_discount_for_checkout`（`saleor/checkout/utils.py:490`）
  ```python
  if voucher.type == VoucherType.ENTIRE_ORDER and not voucher.apply_once_per_order:
      subtotal = base_calculations.base_checkout_subtotal(lines, channel, currency)
      return voucher.get_discount_amount_for(subtotal, channel)
  ```
- **分摊方式**：通过 `propagate_order_discount_on_order_prices` 按比例分摊到每行
- **与订单促销的关系**：互斥（设置 `voucher_code`，订单促销跳过）

#### 行级 ENTIRE_ORDER（`apply_once_per_order=True`）

- **折扣对象存储**：`OrderLineDiscount`（行级）或 `CheckoutLineDiscount`（结账行级）
- **计算方式**：基于最便宜商品的**单价**（不是小计）
- **计算入口**：`get_voucher_discount_for_checkout`（`saleor/checkout/utils.py:498`）
  ```python
  if voucher.type == VoucherType.ENTIRE_ORDER and voucher.apply_once_per_order:
      prices = get_base_lines_prices(lines)
      return voucher.get_discount_amount_for(min(prices), channel)
  ```
- **行级应用**：通过 `attach_voucher_to_line_info`（`saleor/discount/utils/voucher.py:194`）
  ```python
  if voucher.apply_once_per_order:
      if cheapest_line := get_the_cheapest_line(lines_included_in_discount):
          discounted_lines_by_voucher = [cheapest_line]   # ← 只选最便宜的一行
  ```
- **行级折扣计算**：`calculate_line_discount_amount_from_voucher`（`saleor/discount/utils/voucher.py:622`）
  ```python
  if line_info.voucher.apply_once_per_order:
      unit_price = total_price / quantity
      voucher_unit_discount_amount = voucher.get_discount_amount_for(unit_price, channel)
      discount_amount = min(voucher_unit_discount_amount, unit_price)   # ← 只扣单价，不乘数量
  ```
- **与订单促销的关系**：仍互斥（设置 `voucher_code`，订单促销跳过）

#### SPECIFIC_PRODUCT 行级优惠券

- **折扣对象存储**：`OrderLineDiscount` / `CheckoutLineDiscount`
- **计算方式**：
  - 非 `apply_once_per_order` + percentage：基于行的 `total_price`（已扣目录促销）计算百分比
  - 非 `apply_once_per_order` + fixed：基于行的 `unit_price` × `quantity`
  - `apply_once_per_order`：与 ENTIRE_ORDER + apply_once_per_order 相同，只扣单价
- **商品范围**：通过 `voucher.products`、`voucher.collections`、`voucher.categories`、`voucher.variants` M2M 限制

### 6.3 apply_once_per_order 对两种类型的影响对比

| 维度 | ENTIRE_ORDER + apply_once_per_order | SPECIFIC_PRODUCT + apply_once_per_order |
|------|------------------------------------|----------------------------------------|
| **适用商品范围** | 所有商品（选最便宜的一行） | 指定商品/分类/集合/变体范围内选最便宜 |
| **折扣对象类型** | `OrderLineDiscount` / `CheckoutLineDiscount` | `OrderLineDiscount` / `CheckoutLineDiscount` |
| **计价基础** | 最便宜商品的**单价** | 指定范围内最便宜商品的**单价** |
| **是否乘数量** | ❌ 不乘（只扣单价一次） | ❌ 不乘（只扣单价一次） |
| **对订单促销** | 互斥（`voucher_code` 阻断） | 互斥（`voucher_code` 阻断） |
| **与目录促销** | 叠加（行级，基于已扣目录促销的价格） | 叠加（行级，基于已扣目录促销的价格） |

### 6.4 行级优惠券在 Order 重算中的特殊处理

在 `refresh_order_base_prices_and_discounts`（`saleor/order/calculations.py:707`）中：

```python
is_apply_once_per_order_voucher = (
    order.voucher and order.voucher.apply_once_per_order
)
if is_apply_once_per_order_voucher:
    # apply_once_per_order 优惠券可能影响其他行（因为只选最便宜的）
    # 如果最便宜行发生变化，需要重新分配
    reattach_apply_once_per_order_voucher_info(
        lines_info, initial_cheapest_line, order
    )
    create_or_update_line_discount_objects_from_voucher(lines_info)
else:
    # SPECIFIC_PRODUCT 非 apply_once_per_order 只影响指定行
    create_or_update_line_discount_objects_from_voucher(lines_info_to_update)
```

**`reattach_apply_once_per_order_voucher_info`**（`saleor/order/fetch.py:202`）的核心逻辑：
```python
def reattach_apply_once_per_order_voucher_info(lines_info, initial_cheapest_line_info, order):
    if get_the_cheapest_line(lines_info) == initial_cheapest_line_info:
        return  # 最便宜行没变，无需重新分配
    
    # 最便宜行变了 → 清除所有行的优惠券信息，重新分配
    for line_info in lines_info:
        line_info.voucher = None
        line_info.voucher_code = None
        line_info.voucher_denormalized_info = None
    
    attach_voucher_info(lines_info, order)  # 重新选最便宜的行
```

### 6.5 完整的优惠券类型-计价层级决策树

```
Voucher
  │
  ├── type == SHIPPING ──────────→ 运费级 ──→ 基于运费计算
  │
  ├── type == ENTIRE_ORDER
  │     │
  │     ├── apply_once_per_order == False ──→ 订单级 ──→ 基于订单小计
  │     │
  │     └── apply_once_per_order == True ──→ 行级 ──→ 选最便宜行，扣单价(不乘数量)
  │
  └── type == SPECIFIC_PRODUCT
        │
        ├── apply_once_per_order == False ──→ 行级 ──→ 对指定范围内每行扣(乘数量)
        │
        └── apply_once_per_order == True ──→ 行级 ──→ 指定范围选最便宜行，扣单价(不乘数量)
```

---

## 七、关键代码文件索引

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

## 八、特殊场景分析

### 8.1 订单完成时的优惠券冻结

当 Checkout 转换为 Order 时（`complete_checkout.py:_process_voucher_data_for_order`）：
- 优惠券数据被验证并增加使用计数
- 订单中的优惠券折扣信息被**反规范化**（denormalized），即使用当时的快照数据
- 后续即使优惠券条件变化（如折扣值修改），已完成订单中的折扣不会自动变化
- **但 `voucher_code` 仍保留在订单上**，持续阻断订单促销回切

### 8.2 草稿订单的价格刷新

草稿订单的 `should_refresh_prices` 标志触发价格重算：
- `refresh_order_base_prices_and_discounts` 会重新获取最新的渠道列表价格
- 目录促销折扣基于最新价格重新计算
- 优惠券折扣使用反规范化数据（`use_denormalized_data=True`）保持订单创建时的条件
- 订单促销重新基于最新条件评估（但 `voucher_code` 非空时跳过）

### 8.3 赠品促销与优惠券的交互

- 赠品行（`is_gift=True`）**不参与**行级优惠券折扣（`get_discounted_lines` 中跳过 `is_gift` 行）
- 添加优惠券时会删除现有赠品行（`delete_gift_line`）
- 优惠券优先于赠品促销

### 8.4 目录促销折扣的预计算

目录促销折扣通过 `VariantChannelListingPromotionRule` 预计算并存储在渠道列表上（`discounted_price_amount`），这意味着：
- 目录促销折扣不依赖于订单/结账上下文
- 当 `variants_dirty` 标志设置后，后台任务重新计算折扣价格
- 计算订单行折扣时直接使用预计算的 `discount_amount`

---

## 九、价格计算流程图示

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
│  │ 可与所有优惠券叠加    │                                        │
│  └─────────────────────┘                                        │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────────┐                                        │
│  │ ② 行级优惠券折扣     │  SPECIFIC_PRODUCT / apply_once_per_order│
│  │ (基于已扣①的价格)    │  通过 voucher.get_discount_amount_for   │
│  │ 阻断订单促销          │                                        │
│  └─────────────────────┘                                        │
│       │                                                         │
│       ▼                                                         │
│  行价格 → 汇总为小计 base_subtotal                               │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────────────────────┐                          │
│  │ ③ 整单级折扣（三选一，互斥，有优先级）│                          │
│  │   ├─ 整单优惠券 (ENTIRE_ORDER)   │ 优先级最高                │
│  │   ├─ 订单促销 (ORDER_PROMOTION)  │ 若有任何优惠券则跳过       │
│  │   └─ 人工整单折扣 (MANUAL)       │ 若有则删除整单优惠券       │
│  │ 基于小计按比例分摊到各行           │                          │
│  └──────────────────────────────────┘                          │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────────┐                                        │
│  │ ④ 运费优惠券折扣     │  基于原始运费计算                      │
│  │ 阻断订单促销          │  与目录促销/行级优惠券可共存          │
│  └─────────────────────┘                                        │
│       │                                                         │
│       ▼                                                         │
│  运费 + 小计（已扣③） → 总计                                       │
│                                                                 │
│ ─────────────────────────────────────                          │
│ 关键互斥：任意类型优惠券 → voucher_code 非空 → 订单促销完全跳过 │
│ 关键叠加：目录促销 → 行级优惠券（基于已扣价格继续扣）             │
└─────────────────────────────────────────────────────────────────┘
```
