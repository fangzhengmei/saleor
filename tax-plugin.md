# 税务插件接入点与商品订单价格重算触发关系分析

## 一、核心概念

### 税务计算策略（TaxCalculationStrategy）
`saleor/tax/__init__.py:1-6`
```python
class TaxCalculationStrategy:
    FLAT_RATES = "FLAT_RATES"    # 固定税率
    TAX_APP = "TAX_APP"          # 税务插件/应用
```

### 三种插件协作策略

#### 1. 覆盖策略（Override）- `__run_tax_method_until_first_success`
**位置**: `saleor/plugins/manager.py:632-657`

**工作原理**:
- 遍历所有激活的插件，一旦有一个插件返回有效的 `tax_data`（非 None），立即返回该结果
- 第一个成功返回税务数据的插件会**覆盖**所有其他插件的结果
- 如果插件抛出 `TaxDataError`，记录错误并继续尝试下一个插件
- 如果所有插件都失败，抛出最后一个错误

**适用场景**:
- 用于获取税务类型选择列表等只需一个有效响应的场景
- 核心方法: `get_tax_rate_type_choices`

**边界条件**:
- 插件返回 `None` 时继续下一个
- 插件抛出 `TaxDataError` 时继续下一个
- 插件返回非 `None` 值时立即终止遍历

---

#### 2. 追加策略（Append）- `__run_method_on_plugins`
**位置**: `saleor/plugins/manager.py:211-255`

**工作原理**:
- 遍历所有激活的插件，每个插件接收前一个插件的输出作为 `previous_value`
- 插件可以选择修改 `previous_value` 并返回，或返回 `NotImplemented` 跳过
- 多个插件可以**依次追加/修改**结果，形成链式处理

**核心税务计算方法使用此策略**:
- `calculate_checkout_total` / `calculate_order_total`
- `calculate_checkout_shipping` / `calculate_order_shipping`
- `calculate_checkout_line_total` / `calculate_order_line_total`
- `calculate_checkout_line_unit_price` / `calculate_order_line_unit`
- `get_checkout_line_tax_rate` / `get_order_line_tax_rate`
- `get_checkout_shipping_tax_rate` / `get_order_shipping_tax_rate`

**边界条件**:
- 插件方法不存在 → 返回 `previous_value`
- 插件方法返回 `NotImplemented` → 返回 `previous_value`
- 插件方法返回有效值 → 作为下一个插件的 `previous_value`

---

#### 3. 回退策略（Fallback）- `_skip_plugin`
**位置**: `saleor/plugins/avatax/plugin.py:191-209`

**工作原理**:
- 每个插件在执行计算前检查 `previous_value`
- 如果 `previous_value.net != previous_value.gross`，说明**前一个插件已经计算了税费**
- 当前插件跳过自己的逻辑，直接返回 `previous_value`
- 这是一种回退机制：如果前面的插件已成功计算税费，后面的插件不再重复计算

**检查条件**:
```python
def _skip_plugin(self, previous_value):
    # 1. 插件未正确配置（用户名/密码缺失）
    if not (self.config.username_or_account and self.config.password_or_license):
        return True
    
    # 2. 插件未激活
    if not self.active:
        return True
    
    # 3. 前一个插件已计算税费（net != gross）
    if isinstance(previous_value, TaxedMoney):
        return previous_value.net != previous_value.gross
```

**边界条件**:
- 插件配置不完整 → 跳过
- 插件未激活 → 跳过
- 前序插件已完成税费计算 → 跳过

---

## 二、订单价格重算触发机制

### 触发入口函数

#### 1. `should_refresh_prices` - 判断是否需要重算
**位置**: `saleor/order/calculations.py:70-87`

```python
def should_refresh_prices(
    order: Order,
    force_update: bool,
    expired_line_ids: list[UUID],
    allow_sync_webhooks: bool = True,
) -> bool:
```

**触发条件（AND 逻辑）**:
1. 订单状态在 `ORDER_EDITABLE_STATUS` 中
2. 满足以下任一条件（OR）:
   - `force_update=True`
   - `order.should_refresh_prices=True`
   - `expired_line_ids` 非空（行价格过期）
3. 税务策略检查:
   - 如果是 `TAX_APP` 策略且 `allow_sync_webhooks=False` → **不刷新**

---

#### 2. `mark_order_as_priced` - 标记需要重算
**位置**: `saleor/order/utils.py:93-108`

```python
order.should_refresh_prices = True
order.save(update_fields=["should_refresh_prices", "updated_at"])
```

---

#### 3. `get_expired_line_ids` - 获取过期行
**位置**: `saleor/order/calculations.py:242-253`

- 仅对 `DRAFT` 状态订单生效
- 检查 `draft_base_price_expire_at < now` 的行

---

### 完整重算流程

#### `fetch_order_prices_if_expired` - 主入口
**位置**: `saleor/order/calculations.py:192-239`

```
1. 调用 should_refresh_prices() 判断是否需要重算
   ↓ 不需要 → 返回 (order, lines)
   ↓ 需要
2. 调用 prepare_order_lines_for_refresh() 准备订单行
   - 如有过期行 → 调用 refresh_order_base_prices_and_discounts()
   - 否则 → 调用 fetch_draft_order_lines_info()
   ↓
3. 调用 process_order_promotion() 处理订单促销
   ↓
4. 调用 process_order_prices() 计算价格和税费
```

---

#### `process_order_prices` - 价格计算核心
**位置**: `saleor/order/calculations.py:117-189`

```
1. 调用 calculate_prices() 计算基础价格（不含税）
   ↓
2. 获取税务策略 tax_strategy
   ↓
3. 调用 calculate_taxes() 计算税费
   ↓
4. 保存结果（通过 optimistic lock 检查 updated_at）
```

---

#### `calculate_taxes` - 税费计算调度
**位置**: `saleor/order/calculations.py:288-336`

```
获取税务配置:
- prices_entered_with_tax: 价格是否含税
- charge_taxes: 是否征税
- tax_exemption: 是否免税
- tax_app_identifier: 税务应用标识

分支逻辑:
1. prices_entered_with_tax = True
   → 调用 promise_calculate_taxes_with_error_handling()
   → 无论成功失败，最后调用 remove_tax_if_needed()
   
2. prices_entered_with_tax = False 且不应征税
   → 直接 remove_tax()（gross = net）
   
3. prices_entered_with_tax = False 且应征税
   → 调用 promise_calculate_taxes_with_error_handling()
```

---

## 三、税务插件与价格重算的触发关系

### 税务计算分支（`_calculate_and_add_tax`）
**位置**: `saleor/order/calculations.py:339-386`

```
根据 tax_calculation_strategy 分支:

1. FLAT_RATES 策略
   → 调用 update_order_prices_with_flat_rates()
   → 使用数据库中配置的固定税率

2. TAX_APP 策略（有 tax_app_identifier）
   → 调用 _call_plugin_or_tax_app()
   ├─ 以 PLUGIN_IDENTIFIER_PREFIX 开头 → 走插件路径
   │   → 调用 _recalculate_with_plugins()（追加策略）
   └─ 否则 → 走 Webhook 路径
       → 调用 _get_promised_taxes_for_order()（覆盖策略）

3. 兼容模式（无 tax_app_identifier，已废弃）
   → _recalculate_with_plugins() + _get_promised_taxes_for_order()
```

---

### 插件路径深入（`_call_plugin_or_tax_app`）
**位置**: `saleor/order/calculations.py:389-423`

```
tax_app_identifier 以 PLUGIN_IDENTIFIER_PREFIX 开头时:

1. 提取 plugin_ids（去除前缀）
2. 检查插件是否存在且激活
3. 调用 _recalculate_with_plugins(plugin_ids=[...])
   → 使用 __run_method_on_plugins（追加策略）
   → 仅调用指定 plugin_ids 的插件
   → 每个插件内部用 _skip_plugin（回退策略）
```

---

### Webhook 路径深入（`_get_promised_taxes_for_order`）
**位置**: `saleor/order/calculations.py:426-457`

```
调用 order_calculate_taxes.get_taxes()
→ 通过同步 Webhook 获取税务数据
→ 调用 _apply_tax_data() 应用税务数据
→ 覆盖策略：第一个成功响应的 Webhook 生效
```

---

### `_recalculate_with_plugins` - 插件税费计算
**位置**: `saleor/order/calculations.py:460-532`

```
对每个订单行:
  1. manager.calculate_order_line_unit()  → 追加策略
  2. manager.calculate_order_line_total() → 追加策略
  3. manager.get_order_line_tax_rate()    → 追加策略
  每个调用内部都会执行 _skip_plugin() 检查（回退策略）

对订单运费:
  1. manager.calculate_order_shipping()   → 追加策略
  2. manager.get_order_shipping_tax_rate() → 追加策略

最后:
  manager.calculate_order_total()         → 追加策略
```

---

## 四、三种策略协作关系图

```
订单价格重算触发
    ↓
calculate_taxes()
    ↓
tax_calculation_strategy = TAX_APP
    ↓
┌─────────────────────────────────────────────────────┐
│ 追加策略 (__run_method_on_plugins)                  │
│ 遍历插件列表，链式传递 previous_value               │
│                                                     │
│  Plugin A               Plugin B               Plugin C
│      │                    │                    │
│      ├─ _skip_plugin()    ├─ _skip_plugin()    ├─ _skip_plugin()
│      │  检查前序结果     │  检查前序结果     │  检查前序结果
│      │                    │                    │
│      ├─ 计算税费          ├─ 前序已计算?       ├─ 前序已计算?
│      │  返回新值          │  是→返回原值       │  是→返回原值
│      │                    │  否→计算返回新值   │  否→计算返回新值
│      ↓                    ↓                    ↓
│ previous_value ────► result_A ────► result_A ────► result_A
└─────────────────────────────────────────────────────┘
    ↓
结果应用到订单和订单行
```

---

## 五、边界条件总结表

| 策略 | 方法 | 触发条件 | 终止条件 | 错误处理 |
|------|------|----------|----------|----------|
| **覆盖** | `__run_tax_method_until_first_success` | 所有激活插件 | 第一个返回非 `None` 的插件 | 捕获 `TaxDataError`，继续下一个，最后抛出 |
| **追加** | `__run_method_on_plugins` | 所有激活插件 | 遍历完所有插件 | 插件返回 `NotImplemented` 则跳过 |
| **回退** | `_skip_plugin` | 每个插件执行前 | 前序插件已计算税费（net != gross） | 直接返回 `previous_value` |

---

## 六、税务插件调用关键路径

### Checkout 税务计算路径
```
manager.calculate_checkout_*()
    ↓
__run_method_on_plugins(追加)
    ↓
插件.calculate_checkout_*(previous_value)
    ↓
_skip_plugin(回退) → 前序已计算? → 是: 返回 previous_value
                          ↓ 否
                    调用外部税务 API
                          ↓
                    返回计算后的 TaxedMoney
```

### Order 税务计算路径
```
fetch_order_prices_if_expired()
    ↓
should_refresh_prices() → 检查是否触发
    ↓
calculate_taxes()
    ↓
_calculate_and_add_tax()
    ├─ FLAT_RATES → update_order_prices_with_flat_rates()
    └─ TAX_APP → _call_plugin_or_tax_app()
        ├─ 插件路径 → _recalculate_with_plugins() → 追加+回退
        └─ Webhook 路径 → _get_promised_taxes_for_order() → 覆盖
```

---

## 七、关键代码位置索引

| 功能模块 | 文件位置 | 行号 |
|---------|----------|------|
| 税务策略枚举 | `saleor/tax/__init__.py` | 1-6 |
| 追加策略实现 | `saleor/plugins/manager.py` | 211-255 |
| 覆盖策略实现 | `saleor/plugins/manager.py` | 632-657 |
| 回退策略实现 | `saleor/plugins/avatax/plugin.py` | 191-209 |
| 价格重算判断 | `saleor/order/calculations.py` | 70-87 |
| 标记重算标志 | `saleor/order/utils.py` | 93-108 |
| 价格重算主入口 | `saleor/order/calculations.py` | 192-239 |
| 税务计算调度 | `saleor/order/calculations.py` | 288-336 |
| 插件路径税费计算 | `saleor/order/calculations.py` | 460-532 |
| 固定税率计算 | `saleor/tax/calculations/order.py` | 26-71 |
| 税务数据应用 | `saleor/order/calculations.py` | 553-612 |
