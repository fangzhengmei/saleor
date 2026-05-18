# 折扣与优惠券订单计算逻辑深度分析报告（v2 校正版）

> 本版本基于实际代码逻辑进行了关键校正，所有结论均以当前代码为准。

---

## 一、voucher_code 存在时跳过订单级促销的触发条件

### 1.1 Checkout 路径

#### 触发点
**代码位置**：`saleor/discount/utils/checkout.py:226-229`

```python
def create_checkout_discount_objects_for_order_promotions(checkout_info, lines_info, *, save=False):
    _set_checkout_base_prices(checkout_info, lines_info)
    checkout = checkout_info.checkout

    # Discount from order rules is applied only when the voucher is not set
    if checkout.voucher_code:
        _clear_checkout_discount(checkout_info, lines_info, save)
        return None
```

#### 触发条件
- **判断依据**：`checkout.voucher_code` 是否为真值（非空字符串）
- **动作**：
  1. 调用 `_clear_checkout_discount()` 清除所有 `ORDER_PROMOTION` 类型折扣
  2. 删除赠品行（如果有）
  3. 清除 `checkout_info.discounts` 中的订单级促销
  4. 重置 `checkout.discount_amount` / `discount_name` 等字段
  5. 立即 `return None`，不执行后续促销匹配逻辑

#### 关键说明
- **不校验优惠券有效性**：只要 `voucher_code` 字段有值，无论该优惠券是否有效、是否已过期、是否适用，都会跳过订单级促销
- **执行时机**：在促销规则匹配**之前**，是第一道关卡
- **影响范围**：仅影响订单级促销（`ORDER_PROMOTION`），目录级促销（`PROMOTION`）不受影响

---

### 1.2 Draft Order 路径

#### 触发点
**代码位置**：`saleor/discount/utils/order.py:188-191`

```python
def create_order_discount_objects_for_order_promotions(order, lines_info):
    # If voucher is set or manual discount applied, then skip order promotions
    if order.voucher_code or order.discounts.filter(type=DiscountType.MANUAL):
        _clear_order_discount(order, lines_info)
        return
```

#### 触发条件（满足任一即触发）
1. `order.voucher_code` 为真值（非空字符串）
2. 订单存在 `MANUAL` 类型的折扣对象

#### 动作
1. 调用 `_clear_order_discount()` 清除所有 `ORDER_PROMOTION` 类型折扣
2. 删除赠品行（如果有）
3. 立即 `return`，不执行后续促销匹配逻辑

---

### 1.3 两条路径对比

| 维度 | Checkout 路径 | Draft Order 路径 |
|------|--------------|------------------|
| 触发函数 | `create_checkout_discount_objects_for_order_promotions()` | `create_order_discount_objects_for_order_promotions()` |
| 触发条件 | `checkout.voucher_code` 非空 | `order.voucher_code` 非空 **或** 存在手动订单折扣 |
| 清除动作 | `_clear_checkout_discount()` | `_clear_order_discount()` |
| 清除内容 | ORDER_PROMOTION 折扣 + 赠品行 | ORDER_PROMOTION 折扣 + 赠品行 |
| 前置操作 | 先执行 `_set_checkout_base_prices()` | 先判断，符合条件立即清除后 return |

---

## 二、apply_once_per_order 金额计算真实步骤

### 2.1 核心函数
**代码位置**：`saleor/discount/utils/voucher.py:622-661`

```python
def calculate_line_discount_amount_from_voucher(line_info, total_price):
    """
    Args:
        line_info: Order/Checkout line data.
        total_price: Total price of the line, should be already reduced by
                     catalogue discounts if any applied.
    """
```

**重要**：`total_price` 是**该行的总价**（单价 × 数量，已减目录促销），不是直接输入用于计算折扣的基准。

---

### 2.2 apply_once_per_order = True 时的计算流程

```
输入: total_price (该行总价, 已减目录促销)
       line_info (含 voucher, channel, quantity)

Step 1: 计算单价
  unit_price = total_price / quantity

Step 2: 基于单价计算优惠券折扣
  voucher_unit_discount_amount = voucher.get_discount_amount_for(
      unit_price, channel=channel
  )
  注: get_discount_amount_for 根据折扣类型计算:
    - FIXED:    min(固定折扣金额, unit_price)
    - PERCENTAGE: unit_price × 折扣百分比

Step 3: 约束折扣不超过单价
  discount_amount = min(voucher_unit_discount_amount, unit_price)

输出: discount_amount (单件折扣金额, 不乘以数量)
```

#### 关键校正点
- **之前的错误理解**：认为 `total_price` 直接作为折扣计算输入，折扣 = min(优惠券总额, 该行总价)
- **实际逻辑**：先从总价算出 `unit_price`，基于单价计算折扣，**最终折扣金额是单件折扣，不乘数量**
- **影响**：对于固定金额优惠券 + apply_once_per_order，折扣金额受单价约束，与数量无关

---

### 2.3 与 apply_once_per_order = False 的对比

| 场景 | 计算方式 | 折扣约束 |
|------|---------|---------|
| **apply_once_per_order = True** | 基于 unit_price 计算 | `min(voucher_unit_discount, unit_price)` |
| **apply_once_per_order = False (FIXED)** | 基于 unit_price 计算后 × 数量 | `min(voucher_unit_discount × quantity, total_price)` |
| **apply_once_per_order = False (PERCENTAGE)** | 基于 total_price 直接计算 | `min(voucher_discount, total_price)` |

---

### 2.4 数值示例校正

假设：
- 商品 A：单价 $100（目录促销后），数量 2
- 优惠券：$30 FIXED，`apply_once_per_order = True`，未指定适用范围

**v1 错误计算**：
- 该行总价 = $100 × 2 = $200
- 折扣 = min($30, $200) = $30 ❌

**v2 正确计算（基于实际代码）**：
- 该行总价 = $100 × 2 = $200
- unit_price = $200 / 2 = $100
- voucher_unit_discount = min($30, $100) = $30
- discount_amount = min($30, $100) = $30 ✅（单件折扣）
- 该行最终折扣总额 = $30（因为只应用于最便宜的一行，且是单件折扣）

**最终行价格**：$100 × 2 - $30 = $170

---

### 2.5 多数量场景对比

假设：
- 商品 A：单价 $50，数量 3
- 优惠券：$40 FIXED，`apply_once_per_order = True`

**计算过程**：
- 该行总价 = $50 × 3 = $150
- unit_price = $150 / 3 = $50
- voucher_unit_discount = min($40, $50) = $40
- discount_amount = min($40, $50) = $40
- 该行最终折扣总额 = $40（不是 $40 × 3 = $120！）

**结论**：`apply_once_per_order` 的折扣是**单件折扣**，仅应用一次，与行内数量无关。

---

## 三、gift line 排除边界详解

gift line（`is_gift = True`）的排除发生在**两个不同阶段**，边界和逻辑不同。

---

### 3.1 第一阶段：范围筛选（get_discounted_lines）

**代码位置**：`saleor/discount/utils/voucher.py:220-253`

```python
def get_discounted_lines(lines, voucher_info):
    discounted_lines = []
    if (voucher_info.product_pks or voucher_info.collection_pks or
        voucher_info.category_pks or voucher_info.variant_pks):
        # 指定了适用范围
        for line_info in lines:
            if line_info.line.is_gift:  # 🔴 排除 gift line
                continue
            # ... 匹配 variant / product / category / collection ...
            if 匹配成功:
                discounted_lines.append(line_info)
    else:
        # 🟢 未指定任何范围：所有 lines 都加入，包括 gift line！
        discounted_lines.extend(lines)
    return discounted_lines
```

#### 边界说明
| 场景 | gift line 是否被排除 |
|------|---------------------|
| 指定了 products/variants/categories/collections 任一 | ✅ 排除 |
| 未指定任何范围 | ❌ **不排除**，包含在候选列表中 |

> **关键点**：`get_discounted_lines` 只在 `voucher.type == SPECIFIC_PRODUCT` 时被调用（见 `attach_voucher_to_line_info` 第 206-210 行）。对于 `ENTIRE_ORDER + apply_once_per_order` 的优惠券，不经过此函数，直接使用所有 lines_info。

---

### 3.2 第二阶段：挂券后创建折扣对象

#### Checkout 路径（目录促销折扣对象创建）
**代码位置**：`saleor/discount/utils/checkout.py:157-160`

```python
# delete all existing discounts if the line is not discounted or it is a gift
if not discounted_line or line.is_gift:
    line_discounts_to_remove.extend(discounts_to_update)
    continue
```

**说明**：创建目录促销折扣对象时，如果行是 gift，移除该行所有折扣。

#### Draft Order 路径（优惠券折扣对象创建）
**代码位置**：`saleor/discount/utils/voucher.py:543-550`

```python
if (
    (not voucher and use_denormalized_data is False)
    or line.is_gift  # 🔴 无条件排除 gift line
    or manual_line_discount
):
    if discount_to_update:
        line_discounts_to_remove.append(discount_to_update)
    continue
```

**说明**：无论优惠券是否指定了适用范围，无论优惠券类型是什么，**只要 `line.is_gift = True`，就移除该行的优惠券折扣**。

---

### 3.3 gift line 完整处理流程

```
有优惠券，需要应用到行
    ↓
Step 1: attach_voucher_to_line_info()
    ├─ SPECIFIC_PRODUCT 类型 → 调用 get_discounted_lines()
    │   ├─ 指定了范围 → 排除 gift line
    │   └─ 未指定范围 → 包含 gift line（暂时）
    └─ ENTIRE_ORDER + apply_once_per_order → 不调用 get_discounted_lines()
       └─ 所有 lines_info 都在候选中，包括 gift line
    ↓
Step 2: 将 voucher 关联到候选行的 line_info
    ↓
Step 3: 创建/更新折扣对象（prepare_line_discount_objects_for_voucher）
    └─ 遍历所有行，只要 line.is_gift → 移除折扣，不创建
    ↓
最终结果：gift line 不会有任何优惠券折扣
```

---

### 3.4 边界场景说明

#### 场景 1：ENTIRE_ORDER + apply_once_per_order，订单含 gift line
- 候选列表包含 gift line
- 选最便宜行时，gift line 的 `variant_discounted_price` 通常为 $0，可能被选中
- 但在创建折扣对象阶段，gift line 被排除
- **结果**：优惠券可能无法应用（如果 gift line 是最便宜的），或者应用到次便宜的行

#### 场景 2：SPECIFIC_PRODUCT，未指定范围，订单含 gift line
- `get_discounted_lines` 返回所有行，包括 gift line
- 但在创建折扣对象阶段，gift line 被排除
- **结果**：gift line 不享受折扣

#### 场景 3：SPECIFIC_PRODUCT，指定范围包含 gift line 对应的产品
- `get_discounted_lines` 中直接排除 gift line
- 候选列表中没有 gift line
- **结果**：gift line 不享受折扣

---

## 四、校正结论小结

### 4.1 校正点汇总

| 校正项 | v1 理解 | v2 实际代码 | 影响 |
|--------|--------|-------------|------|
| voucher_code 跳过订单级促销 | 隐含校验优惠券有效性 | 只要 `voucher_code` 非空就跳过，不校验有效性 | 无效/过期优惠券也会导致订单级促销无法应用 |
| apply_once_per_order 金额计算 | 基于该行总价 | 先算单价，基于单价计算折扣，结果是单件折扣 | 固定金额优惠券受单价约束，与数量无关 |
| gift line 排除 | 全局始终排除 | 两阶段排除：范围筛选阶段仅在指定范围时排除，创建折扣对象阶段无条件排除 | 未指定范围时 gift line 会进入候选列表，可能影响最便宜行选取 |

### 4.2 对订单折扣结果理解的影响

1. **优惠券码输入即阻断订单级促销**：
   - 只要用户输入了优惠券码，无论该码是否能用，订单级促销都不会生效
   - 业务上需注意：用户输入无效优惠券后又删除，需重新触发价格计算才能恢复订单级促销

2. **apply_once_per_order 是"单件折扣"不是"整行折扣"**：
   - 优惠券 $50 FIXED + apply_once_per_order，应用于单价 $30 的商品，最多折扣 $30，不是 $50
   - 该行数量再多，折扣也只有 $30（单件）

3. **gift line 可能干扰最便宜行选取**：
   - ENTIRE_ORDER + apply_once_per_order 时，gift line（$0）会参与最便宜行比较
   - 如果 gift line 被选中，在创建折扣对象阶段又被排除，可能导致优惠券无法应用
   - 业务上需注意：包含赠品的订单，使用 apply_once_per_order 优惠券可能出现预期外的行为

4. **两条路径的清除动作时机不同**：
   - Checkout 路径先设 base_price，再判断 voucher_code
   - Draft Order 路径先判断 voucher_code，符合条件直接清除后 return
   - 对性能和执行路径有细微影响

---

## 五、关键代码位置速查表

| 逻辑点 | 文件 | 行号 |
|--------|------|------|
| Checkout 跳过订单级促销 | `saleor/discount/utils/checkout.py` | 226-229 |
| Draft Order 跳过订单级促销 | `saleor/discount/utils/order.py` | 188-191 |
| apply_once_per_order 金额计算 | `saleor/discount/utils/voucher.py` | 622-661 |
| get_discounted_lines 范围筛选 | `saleor/discount/utils/voucher.py` | 220-253 |
| attach_voucher_to_line_info 挂券 | `saleor/discount/utils/voucher.py` | 194-217 |
| 准备行级优惠券折扣对象 | `saleor/discount/utils/voucher.py` | 512-619 |
| Checkout 清除订单级促销 | `saleor/discount/utils/checkout.py` | 284-317 |
| Draft Order 清除订单级促销 | `saleor/discount/utils/order.py` | 243-251 |
