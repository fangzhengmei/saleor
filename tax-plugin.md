# 税务插件接入点与商品订单价格重算触发关系分析

## 一、核心概念

### 税务计算策略（TaxCalculationStrategy）
`saleor/tax/__init__.py:1-6`
```python
class TaxCalculationStrategy:
    FLAT_RATES = "FLAT_RATES"    # 固定税率
    TAX_APP = "TAX_APP"          # 税务插件/应用
```

---

### 三种插件协作策略

#### 1. 覆盖策略（Override）- Webhook 路径
**⚠️ 重要修正**: `__run_tax_method_until_first_success` 方法定义在 `saleor/plugins/manager.py:632-657`，但**实际未被任何业务代码调用**，属于死代码。

**真正的覆盖策略实现**: `saleor/tax/webhooks/shared.py`

**工作原理**:
- 通过同步 Webhook 调用外部税务应用获取税务数据
- **第一个成功返回有效数据的 Webhook 生效**，后续结果被忽略
- 两种调用模式:

**模式 A - 指定税务应用（推荐）**:
```python
get_taxes_for_app_identifier(
    event_type=event_type,
    app_identifier=app_identifier,
    ...
)
```
- 根据 `app_identifier` 查找指定的税务应用
- 只调用该应用注册的 webhook（如果有）
- 应用不存在或 webhook 不存在时抛出 `TaxDataError`
- 调用链: `_get_promised_taxes_for_order()` → `order_calculate_taxes.get_taxes()` → `shared.get_taxes()` → `get_taxes_for_app_identifier()`

**模式 B - 遍历所有 Webhook（兼容模式，已废弃警告）**:
```python
get_taxes_from_all_webhooks(
    event_type=event_type,
    ...
)
```
- 遍历所有订阅了税务计算事件的 webhook
- `Promise.all()` 并发调用所有 webhook
- 在 `process_responses` 中遍历响应，**第一个成功解析为 `TaxData` 的生效**
- 解析失败的响应被跳过，继续尝试下一个
- 调用链: 当 `tax_app_identifier` 为空时触发此路径

**边界条件**:
- 指定 `app_identifier` 但应用不存在 → `Promise.reject(TaxDataError)`
- 指定 `app_identifier` 但应用无对应 webhook → `Promise.reject(TaxDataError)`
- Webhook 返回数据无法解析为 `TaxData` → 跳过，继续下一个
- 所有 Webhook 都失败 → 返回 `None`

**前置条件**:
- `tax_calculation_strategy = TAX_APP`
- `tax_app_identifier` **不以** `PLUGIN_IDENTIFIER_PREFIX` 开头
- `allow_sync_webhooks=True`（默认为 True）

---

#### 2. 追加策略（Append）- 插件路径
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

**调用链**:
- `_recalculate_with_plugins()` → 对每个订单行/运费调用 manager 方法
- `manager.calculate_order_line_unit()` → `__run_method_on_plugins()`
- 遍历插件列表，链式传递 `previous_value`

**边界条件**:
- 插件方法不存在 → 返回 `previous_value`
- 插件方法返回 `NotImplemented` → 返回 `previous_value`
- 插件方法返回有效值 → 作为下一个插件的 `previous_value`

**前置条件**:
- `tax_calculation_strategy = TAX_APP`
- `tax_app_identifier` **以** `PLUGIN_IDENTIFIER_PREFIX` 开头
- 对应插件已激活且配置正确

---

#### 3. 回退策略（Fallback）- 插件内部检查
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

**前置条件**:
- 在追加策略遍历过程中，每个插件执行前调用
- `previous_value` 必须是 `TaxedMoney` 类型

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
1. 订单状态在 `ORDER_EDITABLE_STATUS` 中（`DRAFT` 或 `UNCONFIRMED`）
2. 满足以下任一条件（OR）:
   - `force_update=True`
   - `order.should_refresh_prices=True`
   - `expired_line_ids` 非空（行价格过期）
3. 税务策略检查:
   - 如果是 `TAX_APP` 策略且 `allow_sync_webhooks=False` → **不刷新**

---

#### 2. `invalidate_order_prices` - 标记需要重算
**⚠️ 重要修正**: 函数实际名为 `invalidate_order_prices`，不是 `mark_order_as_priced`。

**位置**: `saleor/order/utils.py:93-108`

```python
def invalidate_order_prices(order: Order, *, save: bool = False) -> None:
    if order.status not in ORDER_EDITABLE_STATUS:
        return
    order.should_refresh_prices = True
    if save:
        order.save(update_fields=["should_refresh_prices", "updated_at"])
```

**⚠️ 注意**: 此函数定义了，但**未被业务代码直接调用**。业务代码主要通过两种方式设置刷新标记：
1. 直接设置 `order.should_refresh_prices = True` 然后保存
2. 批量 `update()` 语句

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
   │   → 调用 _recalculate_with_plugins()（追加策略 + 回退策略）
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
→ 调用 shared.get_taxes()
   ├─ 有 app_identifier → get_taxes_for_app_identifier()（单 Webhook 调用）
   └─ 无 app_identifier → get_taxes_from_all_webhooks()（多 Webhook，首个成功生效）
→ 调用 _apply_tax_data() 应用税务数据
→ 覆盖策略：第一个成功响应并解析有效 TaxData 的 Webhook 生效
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

## 四、业务入口：设置 `should_refresh_prices=True` 的场景

### 4.1 GraphQL Mutations（通过 `invalidate_order_prices` 或直接设置）

#### 订单行相关操作
| Mutation | 文件位置 | 触发场景 |
|----------|----------|----------|
| `DraftOrderCreate` | `saleor/graphql/order/mutations/draft_order_create.py` | 创建草稿订单时 |
| `DraftOrderUpdate` | `saleor/graphql/order/mutations/draft_order_update.py` | 更新草稿订单时 |
| `OrderLinesCreate` | `saleor/graphql/order/mutations/order_lines_create.py` | 添加订单行时 |
| `OrderLineUpdate` | `saleor/graphql/order/mutations/order_line_update.py` | 更新订单行（数量、价格）时 |
| `OrderLineDelete` | `saleor/graphql/order/mutations/order_line_delete.py` | 删除订单行时 |

#### 折扣相关操作
| Mutation | 文件位置 | 触发场景 |
|----------|----------|----------|
| `OrderDiscountAdd` | `saleor/graphql/order/mutations/order_discount_add.py` | 添加订单折扣时 |
| `OrderDiscountDelete` | `saleor/graphql/order/mutations/order_discount_delete.py` | 删除订单折扣时 |
| `OrderLineDiscountUpdate` | `saleor/graphql/order/mutations/order_line_discount_update.py` | 更新订单行折扣时 |
| `OrderLineDiscountRemove` | `saleor/graphql/order/mutations/order_line_discount_remove.py` | 移除订单行折扣时 |

#### 配送方式相关操作
| Mixin 方法 | 文件位置 | 触发场景 |
|------------|----------|----------|
| `ShippingMethodUpdateMixin.update_shipping_method` | `saleor/graphql/order/mutations/utils.py:119-134` | 更新配送方式时 |
| `ShippingMethodUpdateMixin.clear_shipping_method_from_order` | `saleor/graphql/order/mutations/utils.py:105-116` | 清除配送方式时 |

#### 其他订单操作
| Mutation | 文件位置 | 触发场景 |
|----------|----------|----------|
| `OrderUpdate` | `saleor/graphql/order/mutations/order_update.py` | 更新订单地址、用户等信息时 |
| `TaxExemptionManage` | `saleor/graphql/tax/mutations/tax_exemption_manage.py:111` | 设置/取消税务豁免时（直接设置 `should_refresh_prices=True`） |

---

### 4.2 异步任务（直接批量更新）

| 任务函数 | 文件位置 | 触发场景 |
|----------|----------|----------|
| `drop_invalid_shipping_methods_relations_for_given_channels` | `saleor/shipping/tasks.py:29-33` | 配送方法从渠道移除时，批量更新使用了该配送方法的可编辑订单 |
| `disconnect_voucher_codes_from_draft_orders_task` | `saleor/discount/tasks.py:304-324` | 凭证码与草稿订单断开连接时，批量标记重算 |

---

### 4.3 触发入口汇总表

| 触发方式 | 触发场景 | 设置方式 | 适用订单状态 |
|----------|----------|----------|-------------|
| **创建草稿订单** | `DraftOrderCreate` mutation | `invalidate_order_prices()` | DRAFT |
| **更新草稿订单** | `DraftOrderUpdate` mutation | `invalidate_order_prices()` | DRAFT |
| **添加订单行** | `OrderLinesCreate` mutation | `invalidate_order_prices()` | DRAFT / UNCONFIRMED |
| **更新订单行** | `OrderLineUpdate` mutation | `invalidate_order_prices()` | DRAFT / UNCONFIRMED |
| **删除订单行** | `OrderLineDelete` mutation | `invalidate_order_prices()` | DRAFT / UNCONFIRMED |
| **添加订单折扣** | `OrderDiscountAdd` mutation | `invalidate_order_prices()` | DRAFT / UNCONFIRMED |
| **删除订单折扣** | `OrderDiscountDelete` mutation | `invalidate_order_prices()` | DRAFT / UNCONFIRMED |
| **更新行折扣** | `OrderLineDiscountUpdate` mutation | `invalidate_order_prices()` | DRAFT / UNCONFIRMED |
| **移除行折扣** | `OrderLineDiscountRemove` mutation | `invalidate_order_prices()` | DRAFT / UNCONFIRMED |
| **更新配送方式** | `update_shipping_method()` | `invalidate_order_prices()` | DRAFT / UNCONFIRMED |
| **清除配送方式** | `clear_shipping_method_from_order()` | `invalidate_order_prices()` | DRAFT / UNCONFIRMED |
| **更新订单信息** | `OrderUpdate` mutation | `invalidate_order_prices()` | DRAFT / UNCONFIRMED |
| **税务豁免管理** | `TaxExemptionManage` mutation | 直接设置 `= True` | DRAFT / UNCONFIRMED |
| **配送方法失效** | 异步任务 `drop_invalid_shipping_methods_...` | 批量 `update()` | DRAFT / UNCONFIRMED |
| **凭证码失效** | 异步任务 `disconnect_voucher_codes_...` | 批量 `bulk_update()` | DRAFT |

---

## 五、三种策略协作关系图

### 5.1 插件路径（追加 + 回退）
```
订单价格重算触发
    ↓
calculate_taxes()
    ↓
tax_calculation_strategy = TAX_APP
tax_app_identifier 以 PLUGIN_IDENTIFIER_PREFIX 开头
    ↓
┌─────────────────────────────────────────────────────┐
│ 追加策略 (__run_method_on_plugins)                  │
│ 遍历指定 plugin_ids 的插件，链式传递 previous_value │
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

### 5.2 Webhook 路径（覆盖）
```
订单价格重算触发
    ↓
calculate_taxes()
    ↓
tax_calculation_strategy = TAX_APP
tax_app_identifier 不以 PLUGIN_IDENTIFIER_PREFIX 开头
    ↓
┌─────────────────────────────────────────────────────┐
│ 覆盖策略 (shared.get_taxes)                         │
│                                                     │
│ 模式 A: 指定 app_identifier                         │
│  → 查找指定 App → 调用其 Webhook → 返回 TaxData     │
│     （无多 Webhook 竞争，只有一个 Webhook 被调用）   │
│                                                     │
│ 模式 B: 无 app_identifier（兼容模式）                │
│  → 并发调用所有 Webhook                              │
│  → Promise.all() 等待所有响应                        │
│  → 遍历响应，首个成功解析为 TaxData 的生效           │
│  → 后续响应被忽略                                   │
└─────────────────────────────────────────────────────┘
    ↓
_apply_tax_data() 应用税务数据到订单和订单行
```

---

## 六、边界条件总结表

| 策略 | 实现位置 | 生效路径 | 触发条件 | 终止条件 | 错误处理 |
|------|----------|----------|----------|----------|----------|
| **覆盖** | `saleor/tax/webhooks/shared.py` | Webhook 路径 | `tax_app_identifier` 不以插件前缀开头 | 模式A: 仅调用指定 App 的 Webhook；模式B: 首个成功解析 `TaxData` 的响应 | 模式A: 应用/Webhook 不存在则 reject；模式B: 解析失败则跳过继续下一个 |
| **追加** | `saleor/plugins/manager.py:211-255` | 插件路径 | `tax_app_identifier` 以插件前缀开头 | 遍历完所有指定插件 | 插件返回 `NotImplemented` 则跳过 |
| **回退** | `saleor/plugins/avatax/plugin.py:191-209` | 插件路径（内部检查） | 追加策略遍历过程中每个插件执行前 | `previous_value.net != previous_value.gross` | 直接返回 `previous_value` |

---

## 七、关键修正说明

### 7.1 之前的错误
1. **错误**: 覆盖策略实现是 `__run_tax_method_until_first_success`
   **修正**: 该方法是死代码，从未被调用。真正的覆盖策略在 `tax/webhooks/shared.py` 中实现。

2. **错误**: `mark_order_as_priced` 函数标记重算
   **修正**: 函数实际名为 `invalidate_order_prices`，且**未被业务代码直接调用**。业务代码主要通过直接设置属性或批量更新来标记重算。

3. **错误**: 未说明哪些业务入口会设置刷新标记
   **修正**: 补充了 4.1-4.3 节，详细列出了所有 15 个业务入口。

### 7.2 三种策略实际生效位置

| 策略 | Checkout 税务计算 | Order 税务计算（插件路径） | Order 税务计算（Webhook 路径） |
|------|------------------|---------------------------|--------------------------------|
| 覆盖 | ❌ 不生效 | ❌ 不生效 | ✅ 生效（`shared.get_taxes`） |
| 追加 | ✅ 生效（`__run_method_on_plugins`） | ✅ 生效（`__run_method_on_plugins`） | ❌ 不生效 |
| 回退 | ✅ 生效（`_skip_plugin`） | ✅ 生效（`_skip_plugin`） | ❌ 不生效 |

---

## 八、税务插件调用关键路径

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
        ├─ 插件路径（PLUGIN_IDENTIFIER_PREFIX）
        │   → _recalculate_with_plugins() → 追加+回退
        └─ Webhook 路径
            → _get_promised_taxes_for_order() → shared.get_taxes() → 覆盖
```

---

## 九、关键代码位置索引

| 功能模块 | 文件位置 | 行号 |
|---------|----------|------|
| 税务策略枚举 | `saleor/tax/__init__.py` | 1-6 |
| 追加策略实现 | `saleor/plugins/manager.py` | 211-255 |
| ~~覆盖策略（死代码）~~ | `saleor/plugins/manager.py` | 632-657 |
| **覆盖策略（真正实现）** | `saleor/tax/webhooks/shared.py` | 24-139 |
| 回退策略实现 | `saleor/plugins/avatax/plugin.py` | 191-209 |
| 价格重算判断 | `saleor/order/calculations.py` | 70-87 |
| 标记重算标志 | `saleor/order/utils.py` | 93-108 |
| 价格重算主入口 | `saleor/order/calculations.py` | 192-239 |
| 税务计算调度 | `saleor/order/calculations.py` | 288-336 |
| 插件路径税费计算 | `saleor/order/calculations.py` | 460-532 |
| Webhook 路径税费计算 | `saleor/order/calculations.py` | 426-457 |
| 固定税率计算 | `saleor/tax/calculations/order.py` | 26-71 |
| 税务数据应用 | `saleor/order/calculations.py` | 553-612 |
| 税务豁免管理 | `saleor/graphql/tax/mutations/tax_exemption_manage.py` | 109-114 |
| 配送方式失效处理 | `saleor/shipping/tasks.py` | 29-33 |
| 凭证码失效处理 | `saleor/discount/tasks.py` | 313-316 |
