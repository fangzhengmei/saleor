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

**⚠️ 重要修正**: 此函数**被业务代码广泛调用**，是设置刷新标记的主要方式。业务代码通过三种方式设置刷新标记：
1. **调用 `invalidate_order_prices()`** - 主要方式，共 13 处调用（详见第 4 节）
2. **直接设置 `order.should_refresh_prices = True`** - 特殊场景（如 `TaxExemptionManage`）
3. **批量 `update()` / `bulk_update()`** - 异步任务批量处理时使用

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

### 4.0 `invalidate_order_prices` 调用方式详解

**函数签名**:
```python
def invalidate_order_prices(order: Order, *, save: bool = False) -> None:
```

**两种调用模式**:
1. **`invalidate_order_prices(order)`**（`save=False`，共 10 处）
   - 仅设置内存属性，需后续手动调用 `order.save()`（9 处）或 `bulk_update()`（1 处）
2. **`invalidate_order_prices(order, save=True)`**（共 3 处）
   - 设置属性后立即保存到数据库

**调用分布（共 13 处非测试调用）**:
- GraphQL Mutations: 12 处
  - `save=False`: 9 处（7 处直接 mutation + 2 处 Mixin 方法）
  - `save=True`: 3 处（折扣相关）
- 异步任务: 1 处
  - `save=False`: 1 处（后续 `bulk_update()`）

**分组边界说明（详见 4.1-4.3 节）**:
- 第一类：GraphQL Mutations 调用 - 共 13 个入口（12 处调用 `invalidate_order_prices()` + 1 处直接赋值）
- 第二类：异步任务调用 - 共 3 个入口（1 处调用 `invalidate_order_prices()` + 2 处直接赋值/批量更新）
- 第三类：直接赋值/批量更新 - 共 3 处（分散在 GraphQL 和异步任务中，单独汇总）

---

### 4.1 第一类：GraphQL Mutations 调用（共 13 个入口）

**分组定义**：所有位于 `saleor/graphql/` 目录下的 mutation 代码，包括直接 mutation 和 Mixin 工具方法。

#### 1.1 通过 `invalidate_order_prices()` 函数（共 12 处）

| 序号 | 触发场景 | 文件位置 | 行号 | 调用方式 | 后续保存方式 |
|------|----------|----------|------|----------|-------------|
| 1 | **DraftOrderCreate** | `saleor/graphql/order/mutations/draft_order_create.py` | 439 | `invalidate_order_prices(instance)` | `instance.save()` 更新多个字段 |
| 2 | **DraftOrderUpdate** | `saleor/graphql/order/mutations/draft_order_update.py` | 286 | `invalidate_order_prices(instance)` | `instance.save()` 更新多个字段 |
| 3 | **OrderLinesCreate** | `saleor/graphql/order/mutations/order_lines_create.py` | 219 | `invalidate_order_prices(order)` | `order.save()` 更新多个字段 |
| 4 | **OrderLineUpdate** | `saleor/graphql/order/mutations/order_line_update.py` | 112 | `invalidate_order_prices(order)` | `order.save(update_fields=["should_refresh_prices", ...])` |
| 5 | **OrderLineDelete** | `saleor/graphql/order/mutations/order_line_delete.py` | 105 | `invalidate_order_prices(order)` | `order.save(update_fields=["should_refresh_prices", ...])` |
| 6 | **OrderDiscountDelete** | `saleor/graphql/order/mutations/order_discount_delete.py` | 69 | `invalidate_order_prices(order)` | `order.save(update_fields=["should_refresh_prices", ...])` |
| 7 | **OrderUpdate** | `saleor/graphql/order/mutations/order_update.py` | 156 | `invalidate_order_prices(instance)` | `_save_order_instance()` 统一保存 |
| 8 | **OrderDiscountAdd** | `saleor/graphql/order/mutations/order_discount_add.py` | 85 | `invalidate_order_prices(order, save=True)` | 函数内部自动保存 |
| 9 | **OrderLineDiscountUpdate** | `saleor/graphql/order/mutations/order_line_discount_update.py` | 85 | `invalidate_order_prices(order, save=True)` | 函数内部自动保存 |
| 10 | **OrderLineDiscountRemove** | `saleor/graphql/order/mutations/order_line_discount_remove.py` | 62 | `invalidate_order_prices(order, save=True)` | 函数内部自动保存 |
| 11 | **update_shipping_method** (Mixin) | `saleor/graphql/order/mutations/utils.py` | 134 | `invalidate_order_prices(order)` | 调用方后续负责保存 |
| 12 | **clear_shipping_method_from_order** (Mixin) | `saleor/graphql/order/mutations/utils.py` | 116 | `invalidate_order_prices(order)` | 调用方后续负责保存 |

#### 1.2 直接赋值（共 1 处）

| 序号 | 触发场景 | 文件位置 | 行号 | 设置方式 | 后续保存方式 |
|------|----------|----------|------|----------|-------------|
| 13 | **TaxExemptionManage** | `saleor/graphql/tax/mutations/tax_exemption_manage.py` | 111 | `order.should_refresh_prices = True` | `order.save()` |

---

### 4.2 第二类：异步任务调用（共 3 个入口）

**分组定义**：所有位于 `tasks.py` 中的 Celery 异步任务代码。

#### 2.1 通过 `invalidate_order_prices()` 函数（共 1 处）

| 序号 | 触发场景 | 文件位置 | 行号 | 调用方式 | 后续保存方式 |
|------|----------|----------|------|----------|-------------|
| 14 | **recalculate_orders_task** | `saleor/order/tasks.py` | 44 | `invalidate_order_prices(order)` | `bulk_update()` 批量保存 |

#### 2.2 直接赋值/批量更新（共 2 处）

| 序号 | 触发场景 | 文件位置 | 行号 | 设置方式 | 后续保存方式 |
|------|----------|----------|------|----------|-------------|
| 15 | **drop_invalid_shipping_methods_relations_for_given_channels** | `saleor/shipping/tasks.py` | 33 | 直接 `update(..., should_refresh_prices=True)` | QuerySet.update() 直接执行 |
| 16 | **disconnect_voucher_codes_from_draft_orders_task** | `saleor/discount/tasks.py` | 315 | `order.should_refresh_prices = True` | `bulk_update()` 批量保存 |

---

### 4.3 第三类：直接赋值/批量更新方式汇总（共 3 处）

**分组定义**：不调用 `invalidate_order_prices()` 函数，直接设置属性的场景（分散在 GraphQL 和异步任务中）。

| 序号 | 触发场景 | 所属分组 | 文件位置 | 行号 | 设置方式 |
|------|----------|----------|----------|------|----------|
| G1 | **TaxExemptionManage** | GraphQL | `saleor/graphql/tax/mutations/tax_exemption_manage.py` | 111 | `order.should_refresh_prices = True` → `order.save()` |
| T1 | **drop_invalid_shipping_methods_relations_for_given_channels** | 异步任务 | `saleor/shipping/tasks.py` | 33 | `update(..., should_refresh_prices=True)` |
| T2 | **disconnect_voucher_codes_from_draft_orders_task** | 异步任务 | `saleor/discount/tasks.py` | 315 | `order.should_refresh_prices = True` → `bulk_update()` |

---

### 4.4 分类统计表（全文档数据统一）

#### 4.4.1 按业务入口分组（总计 16 个入口）

| 业务分组 | 入口数 | 通过 `invalidate_order_prices()` | 直接赋值/批量更新 |
|----------|--------|----------------------------------|-------------------|
| **GraphQL Mutations** | 13 | 12 | 1 |
| **异步任务** | 3 | 1 | 2 |
| **总计** | **16** | **13** | **3** |

#### 4.4.2 按调用函数分组（`invalidate_order_prices()` 共 13 处调用）

| 调用模式 | 数量 | 所属分组 | 典型场景 |
|----------|------|----------|----------|
| `invalidate_order_prices()` + 手动 `save()` | 9 | GraphQL（7+2 Mixin） | 大部分 mutations（需要同时更新其他字段） |
| `invalidate_order_prices(save=True)` | 3 | GraphQL | 折扣相关 mutation（仅需更新刷新标记） |
| `invalidate_order_prices()` + `bulk_update()` | 1 | 异步任务 | `recalculate_orders_task` |
| **小计** | **13** | | |

#### 4.4.3 按代码位置分组（`invalidate_order_prices()` 共 13 处调用）

| 代码位置 | 数量 | 具体文件 |
|----------|------|----------|
| `saleor/graphql/order/mutations/` | 12 | `draft_order_create.py`、`draft_order_update.py`、`order_lines_create.py`、`order_line_update.py`、`order_line_delete.py`、`order_discount_add.py`、`order_discount_delete.py`、`order_line_discount_update.py`、`order_line_discount_remove.py`、`order_update.py`、`utils.py` (2 处) |
| `saleor/order/tasks.py` | 1 | `recalculate_orders_task` |
| **小计** | **13** | |

---

### 4.5 触发入口汇总表（16 个入口，逐一核对）

| 序号 | 业务分组 | 触发场景 | 设置方式 | 具体代码 | 适用订单状态 |
|------|----------|----------|----------|----------|-------------|
| 1 | GraphQL | **创建草稿订单** | `invalidate_order_prices()` + 手动 save | `invalidate_order_prices(instance)` → `instance.save()` | DRAFT |
| 2 | GraphQL | **更新草稿订单** | `invalidate_order_prices()` + 手动 save | `invalidate_order_prices(instance)` → `instance.save()` | DRAFT |
| 3 | GraphQL | **添加订单行** | `invalidate_order_prices()` + 手动 save | `invalidate_order_prices(order)` → `order.save()` | DRAFT / UNCONFIRMED |
| 4 | GraphQL | **更新订单行** | `invalidate_order_prices()` + 手动 save | `invalidate_order_prices(order)` → `order.save(update_fields=...)` | DRAFT / UNCONFIRMED |
| 5 | GraphQL | **删除订单行** | `invalidate_order_prices()` + 手动 save | `invalidate_order_prices(order)` → `order.save(update_fields=...)` | DRAFT / UNCONFIRMED |
| 6 | GraphQL | **添加订单折扣** | `invalidate_order_prices(save=True)` | `invalidate_order_prices(order, save=True)` | DRAFT / UNCONFIRMED |
| 7 | GraphQL | **删除订单折扣** | `invalidate_order_prices()` + 手动 save | `invalidate_order_prices(order)` → `order.save(update_fields=...)` | DRAFT / UNCONFIRMED |
| 8 | GraphQL | **更新行折扣** | `invalidate_order_prices(save=True)` | `invalidate_order_prices(order, save=True)` | DRAFT / UNCONFIRMED |
| 9 | GraphQL | **移除行折扣** | `invalidate_order_prices(save=True)` | `invalidate_order_prices(order, save=True)` | DRAFT / UNCONFIRMED |
| 10 | GraphQL | **更新配送方式** (Mixin) | `invalidate_order_prices()` + 调用方保存 | `invalidate_order_prices(order)` | DRAFT / UNCONFIRMED |
| 11 | GraphQL | **清除配送方式** (Mixin) | `invalidate_order_prices()` + 调用方保存 | `invalidate_order_prices(order)` | DRAFT / UNCONFIRMED |
| 12 | GraphQL | **更新订单信息** | `invalidate_order_prices()` + 统一保存 | `invalidate_order_prices(instance)` → `_save_order_instance()` | DRAFT / UNCONFIRMED |
| 13 | GraphQL | **税务豁免管理** | 直接赋值 + save | `order.should_refresh_prices = True` → `order.save()` | DRAFT / UNCONFIRMED |
| 14 | 异步任务 | **订单重算任务** | `invalidate_order_prices()` + 批量保存 | `invalidate_order_prices(order)` → `bulk_update()` | DRAFT / UNCONFIRMED |
| 15 | 异步任务 | **配送方法失效** | 直接批量 `update()` | `update(..., should_refresh_prices=True)` | DRAFT / UNCONFIRMED |
| 16 | 异步任务 | **凭证码失效** | 直接赋值 + `bulk_update()` | `order.should_refresh_prices = True` → `bulk_update()` | DRAFT |

---

### 4.6 关键调用链说明

**典型 GraphQL 调用链 - `save=False` + 手动 save（共 9 处）**:
```
OrderLineUpdate.save()
    ↓
change_order_line_quantity()  # 修改订单行数量
    ↓
invalidate_order_prices(order)  # save=False，仅设置内存属性
    ↓
order.save(update_fields=["should_refresh_prices", "weight", "updated_at"])  # 手动保存
    ↓
后续 GraphQL 解析时触发 fetch_order_prices_if_expired() 实际重算
```

**典型 GraphQL 调用链 - `save=True` 模式（共 3 处）**:
```
OrderDiscountAdd.perform_mutation()
    ↓
create_manual_order_discount()  # 创建折扣
    ↓
order_discount_added_event()  # 记录事件
    ↓
invalidate_order_prices(order, save=True)  # save=True，内部自动保存
    ↓
后续 GraphQL 解析时触发 fetch_order_prices_if_expired() 实际重算
```

**典型异步任务调用链 - `invalidate_order_prices()` + `bulk_update()`（共 1 处）**:
```
recalculate_orders_task(order_ids)
    ↓
for order in orders:
    invalidate_order_prices(order)  # save=False，仅设置内存属性
    ↓
Order.objects.bulk_update(orders, ["should_refresh_prices"])  # 批量保存
    ↓
后续订单访问时触发实际重算
```

**典型直接赋值调用链 - 异步任务批量 update（共 1 处）**:
```
drop_invalid_shipping_methods_relations_for_given_channels()
    ↓
Order.objects.filter(...).update(
    shipping_method=None,
    should_refresh_prices=True  # 直接在 QuerySet.update() 中设置
)
    ↓
后续订单访问时触发实际重算
```

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
   **修正**: 函数实际名为 `invalidate_order_prices`。

3. **错误**: `invalidate_order_prices` 未被业务代码直接调用
   **修正**: 该函数**被业务代码广泛调用**，共 13 处非测试调用，是设置刷新标记的主要方式。另有 3 处直接设置属性或批量更新。

4. **错误**: 未说明哪些业务入口会设置刷新标记
   **修正**: 补充了 4.1-4.5 节，详细列出了所有 16 个业务入口及其具体调用方式。

5. **错误**: `invalidate_order_prices()` 调用分类错误（GraphQL 11 次、异步任务 2 次）
   **修正**: 实际为 GraphQL 12 次、异步任务 1 次。`discount/tasks.py` 和 `shipping/tasks.py` 不调用 `invalidate_order_prices()`，而是直接赋值或批量更新。

6. **错误**: 未区分 `invalidate_order_prices()` 的两种调用模式
   **修正**: 明确了 `save=True`（3 处，折扣相关）和 `save=False`（10 处，含 9 处手动 save 和 1 处 bulk_update）两种调用模式的区别和使用场景。

7. **错误**: 分组口径不统一，GraphQL 调用明细中包含异步任务入口
   **修正**: 重新梳理分组边界，明确三类入口的定义：
   - 第一类：GraphQL Mutations 调用 - 所有 `graphql/` 目录下的代码（共 13 个入口）
   - 第二类：异步任务调用 - 所有 `tasks.py` 中的 Celery 任务（共 3 个入口）
   - 第三类：直接赋值/批量更新 - 不调用 `invalidate_order_prices()` 的场景（共 3 处，分散在上述两类中）
   修正了 4.1-4.6 节的所有分组标题、行项归类、统计总数和示例说明，确保四方面完全一致。

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

### 9.1 核心策略与流程

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

### 9.2 `invalidate_order_prices` 调用位置（共 13 处，按分组排列）

#### 9.2.1 GraphQL Mutations（共 12 处）

| 序号 | 调用场景 | 文件位置 | 行号 | 调用方式 |
|------|---------|----------|------|----------|
| 1 | 创建草稿订单 | `saleor/graphql/order/mutations/draft_order_create.py` | 439 | `invalidate_order_prices(instance)` |
| 2 | 更新草稿订单 | `saleor/graphql/order/mutations/draft_order_update.py` | 286 | `invalidate_order_prices(instance)` |
| 3 | 添加订单行 | `saleor/graphql/order/mutations/order_lines_create.py` | 219 | `invalidate_order_prices(order)` |
| 4 | 更新订单行 | `saleor/graphql/order/mutations/order_line_update.py` | 112 | `invalidate_order_prices(order)` |
| 5 | 删除订单行 | `saleor/graphql/order/mutations/order_line_delete.py` | 105 | `invalidate_order_prices(order)` |
| 6 | 删除订单折扣 | `saleor/graphql/order/mutations/order_discount_delete.py` | 69 | `invalidate_order_prices(order)` |
| 7 | 更新订单信息 | `saleor/graphql/order/mutations/order_update.py` | 156 | `invalidate_order_prices(instance)` |
| 8 | 添加订单折扣 | `saleor/graphql/order/mutations/order_discount_add.py` | 85 | `invalidate_order_prices(order, save=True)` |
| 9 | 更新行折扣 | `saleor/graphql/order/mutations/order_line_discount_update.py` | 85 | `invalidate_order_prices(order, save=True)` |
| 10 | 移除行折扣 | `saleor/graphql/order/mutations/order_line_discount_remove.py` | 62 | `invalidate_order_prices(order, save=True)` |
| 11 | 更新配送方式 (Mixin) | `saleor/graphql/order/mutations/utils.py` | 134 | `invalidate_order_prices(order)` |
| 12 | 清除配送方式 (Mixin) | `saleor/graphql/order/mutations/utils.py` | 116 | `invalidate_order_prices(order)` |

#### 9.2.2 异步任务（共 1 处）

| 序号 | 调用场景 | 文件位置 | 行号 | 调用方式 |
|------|---------|----------|------|----------|
| 13 | 订单重算任务 | `saleor/order/tasks.py` | 44 | `invalidate_order_prices(order)` |

### 9.3 直接赋值/批量更新位置（共 3 处，按分组排列）

#### 9.3.1 GraphQL Mutations（共 1 处）

| 序号 | 调用场景 | 文件位置 | 行号 | 设置方式 |
|------|---------|----------|------|----------|
| G1 | 税务豁免管理 | `saleor/graphql/tax/mutations/tax_exemption_manage.py` | 111 | 直接 `order.should_refresh_prices = True` |

#### 9.3.2 异步任务（共 2 处）

| 序号 | 调用场景 | 文件位置 | 行号 | 设置方式 |
|------|---------|----------|------|----------|
| T1 | 配送方法失效处理 | `saleor/shipping/tasks.py` | 33 | 批量 `update(..., should_refresh_prices=True)` |
| T2 | 凭证码失效处理 | `saleor/discount/tasks.py` | 315 | 直接赋值 + `bulk_update()` |
