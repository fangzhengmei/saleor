# 外部承运商运费 Webhook 计算路径分析

## 概述

外部承运商运费计算通过同步 Webhook 机制实现，整个流程分为多个阶段：**触发条件识别**、**缓存命中判断**、**本地/外部并行获取**、**可用性校验**、**已选方式刷新**、**响应处理与总价计算**。外部承运商返回的运费选项与本地（built-in）配送方式并列展示给用户。

---

## 一、触发条件识别

### 1.1 触发入口

外部承运商运费计算的触发主要发生在以下场景：

| 场景 | 触发点 | 关键文件 |
|------|--------|----------|
| 用户更新 checkout 配送方式 | `CheckoutShippingMethodUpdate` mutation | `saleor/graphql/checkout/mutations/checkout_shipping_method_update.py` |
| 查询可用配送方式 | `get_or_fetch_checkout_deliveries` | `saleor/checkout/delivery_context.py` |
| checkout 价格过期刷新 | `fetch_checkout_data` | `saleor/checkout/calculations.py` |

### 1.2 触发流程

```
用户操作 (GraphQL Mutation / 查询)
    ↓
CheckoutShippingMethodUpdate.perform_mutation() [checkout_shipping_method_update.py:141]
    ↓
get_checkout_delivery() [checkout_shipping_method_update.py:108]
    ↓
get_or_fetch_checkout_deliveries() [delivery_context.py:732]
    ↓
┌─ 是否触发外部 webhook? ─┐
│  是 → fetch_shipping_methods_for_checkout()
│  否 → 直接返回 CheckoutDelivery 缓存
└─────────────────────────┘
    ↓
可用性校验: 查找目标 shipping_method_id 且 active=True
    ↓
assign_delivery_method_to_checkout()
```

### 1.3 关键判断逻辑

在 `get_or_fetch_checkout_deliveries` [delivery_context.py:732-757] 中：

```python
if (
    checkout.delivery_methods_stale_at is None          # 从未获取过
    or checkout.delivery_methods_stale_at <= timezone.now()  # 已过期
) and allow_sync_webhooks:                               # 允许同步调用
    return fetch_shipping_methods_for_checkout(checkout_info, requestor=requestor)
```

- `delivery_methods_stale_at` 控制缓存有效期（默认 24 小时）
- `allow_sync_webhooks` 标志可禁用同步 webhook 调用

### 1.4 缓存命中 vs 重新获取：时序与来源澄清

**重要纠正**：缓存命中时，候选来源是 `CheckoutDelivery` 表，而非重新计算的本地方式。

#### 1.4.1 重新获取路径（缓存过期 + 允许同步 webhook）

**时序**：
1. 先获取本地内置配送方式（同步执行）
2. 并行获取外部承运商配送方式（异步 Promise）
3. 两者都返回后合并
4. 转换为 CheckoutDelivery 并批量 upsert 到数据库
5. 刷新缓存时间 `delivery_methods_stale_at`

**文件**：`saleor/checkout/delivery_context.py:587-709`

```python
def fetch_shipping_methods_for_checkout(...):
    # 第一步：同步获取本地内置方式
    built_in_shipping_methods_dict = {
        int(shipping_method.id): shipping_method
        for shipping_method in get_available_built_in_shipping_methods_for_checkout_info(...)
    }

    def with_external_methods(external_shipping_methods):
        # 第三步：外部方式返回后合并
        external_shipping_methods_dict = {...}
        all_methods = list(built_in_shipping_methods_dict.values()) + list(
            external_shipping_methods_dict.values()
        )
        # ... 后续处理

    # 第二步：异步获取外部方式
    return fetch_external_shipping_methods_for_checkout_info(...).then(with_external_methods)
```

#### 1.4.2 缓存命中路径（缓存有效 或 不允许同步 webhook）

**时序**：
1. 直接查询 `CheckoutDelivery` 表中 `is_valid=True` 的记录
2. 这些记录是之前 `fetch_shipping_methods_for_checkout` 成功执行后存储的
3. 包含本地和外部承运商的配送方式快照

```python
# 直接返回缓存，不重新获取本地或外部方式
return Promise.resolve(
    list(
        CheckoutDelivery.objects.using(
            checkout_info.database_connection_name
        ).filter(
            checkout_id=checkout.pk,
            is_valid=True,
        )
    )
)
```

#### 1.4.3 两种路径对比

| 维度 | 重新获取路径 | 缓存命中路径 |
|------|------------|------------|
| 本地方式来源 | 实时调用 `get_available_built_in_shipping_methods_for_checkout_info` | CheckoutDelivery 表缓存 |
| 外部方式来源 | 实时调用 webhook | CheckoutDelivery 表缓存 |
| 运费价格 | 实时最新 | 缓存时的价格 |
| 方式可用性 | 实时校验 | 缓存时的状态 |
| 性能 | 较慢（含网络 IO） | 快（仅数据库查询） |
| 触发条件 | 缓存过期 + `allow_sync_webhooks=True` | 缓存有效 或 `allow_sync_webhooks=False` |

---

## 二、不触发外部 Webhook 的判断分支

外部承运商 webhook 调用并非在所有场景下都会执行，系统设计了多层判断来避免不必要的同步调用。以下是所有会跳过外部 webhook 的分支路径。

### 2.1 `allow_sync_webhooks = False`：全局禁用同步 Webhook

#### 2.1.1 配送方式查询分支

**文件**：`saleor/checkout/delivery_context.py:732-757`

```python
def get_or_fetch_checkout_deliveries(
    checkout_info: "CheckoutInfo",
    requestor: Union["App", "User", None],
    allow_sync_webhooks: bool = True,
) -> Promise[list[CheckoutDelivery]]:
    checkout = checkout_info.checkout
    if (
        checkout.delivery_methods_stale_at is None
        or checkout.delivery_methods_stale_at <= timezone.now()
    ) and allow_sync_webhooks:  # ← 关键判断
        return fetch_shipping_methods_for_checkout(checkout_info, requestor=requestor)
    # 直接返回缓存的 CheckoutDelivery
    return Promise.resolve(
        list(
            CheckoutDelivery.objects.using(
                checkout_info.database_connection_name
            ).filter(
                checkout_id=checkout.pk,
                is_valid=True,
            )
        )
    )
```

**行为**：
- 当 `allow_sync_webhooks=False` 时，即使 `delivery_methods_stale_at` 已过期，也不会调用 `fetch_shipping_methods_for_checkout`
- 直接从数据库查询已缓存的 `CheckoutDelivery`（`is_valid=True` 的记录）
- 这些缓存记录中可能包含之前查询到的外部承运商配送方式

#### 2.1.2 价格计算分支（Tax App 场景）

**文件**：`saleor/checkout/calculations.py:395-399`

```python
tax_calculation_strategy = get_tax_calculation_strategy_for_checkout(
    checkout_info, database_connection_name=database_connection_name
)

if (
    tax_calculation_strategy == TaxCalculationStrategy.TAX_APP
    and not allow_sync_webhooks
):
    return Promise.resolve((checkout_info, lines))
```

**行为**：
- 当税费计算策略为 `TAX_APP` 且 `allow_sync_webhooks=False` 时，直接返回，不调用税 App webhook
- 这会影响运费的税费计算，但不会影响配送方式列表本身

#### 2.1.3 典型调用场景

`allow_sync_webhooks=False` 通常用于以下场景：
- 后台任务或非用户交互流程，避免阻塞
- 已知缓存仍然有效时的性能优化
- 特定 API 端点设计为只使用本地数据

### 2.2 `delivery_methods_stale_at` 未过期：缓存命中

#### 2.2.1 缓存有效期配置

**文件**：`saleor/settings.py:969-971`

```python
CHECKOUT_DELIVERY_OPTIONS_TTL = datetime.timedelta(
    seconds=parse(os.environ.get("CHECKOUT_DELIVERY_OPTIONS_TTL", "24 hours"))
)
```

- 默认有效期：**24 小时**
- 可通过环境变量 `CHECKOUT_DELIVERY_OPTIONS_TTL` 调整

#### 2.2.2 缓存判断逻辑

**文件**：`saleor/checkout/delivery_context.py:743-747`

```python
if (
    checkout.delivery_methods_stale_at is None      # 从未获取过
    or checkout.delivery_methods_stale_at <= timezone.now()  # 已过期
) and allow_sync_webhooks:
    return fetch_shipping_methods_for_checkout(...)  # 重新获取
```

**真值表**：

| `delivery_methods_stale_at` | `allow_sync_webhooks` | 是否触发外部 webhook |
|----------------------------|----------------------|---------------------|
| `None`（首次）             | `True`               | ✅ 是              |
| `<= now()`（过期）         | `True`               | ✅ 是              |
| `> now()`（有效）          | `True`               | ❌ 否（走缓存）   |
| 任意值                     | `False`              | ❌ 否（走缓存）   |

#### 2.2.3 缓存更新时机

在成功获取配送方式后，缓存时间会被刷新：

**文件**：`saleor/checkout/delivery_context.py:659-662`

```python
checkout.delivery_methods_stale_at = (
    timezone.now() + settings.CHECKOUT_DELIVERY_OPTIONS_TTL
)
checkout.save(update_fields=["delivery_methods_stale_at"])
```

#### 2.2.4 缓存回退行为

当使用缓存时（不触发外部 webhook）：
- 直接查询 `CheckoutDelivery` 表中 `is_valid=True` 的记录
- 这些记录包含之前获取的本地和外部承运商配送方式
- 外部承运商的运费价格是缓存时的价格，不会实时更新

### 2.3 无可用 Webhook：`get_webhooks_for_event` 返回空

#### 2.3.1 外部配送方式列表查询分支

**文件**：`saleor/checkout/webhooks/list_shipping_methods.py:31-33`

```python
event_type = WebhookEventSyncType.SHIPPING_LIST_METHODS_FOR_CHECKOUT
webhooks = get_webhooks_for_event(event_type)
if not webhooks:  # ← 无可用 webhook
    return Promise.resolve(methods)  # 返回空列表
```

`get_webhooks_for_event` 查询条件 [webhook/utils.py:69-91]：
- Webhook 必须是激活状态（`is_active=True`）
- 关联的 App 必须是激活状态（`app.is_active=True`）
- Webhook 必须订阅了目标事件类型

#### 2.3.2 配送方式过滤分支

**文件**：`saleor/checkout/webhooks/exclude_shipping.py:38-42`

```python
webhooks = get_webhooks_for_event(
    WebhookEventSyncType.CHECKOUT_FILTER_SHIPPING_METHODS
)
if not webhooks:
    return Promise.resolve([])  # 不过滤任何方式
```

#### 2.3.3 无 Webhook 时的回退行为

| 场景 | 无可用 webhook 时的行为 |
|------|------------------------|
| `SHIPPING_LIST_METHODS_FOR_CHECKOUT` | 返回空的外部配送方式列表，只展示本地配送方式 |
| `CHECKOUT_FILTER_SHIPPING_METHODS` | 返回空的排除列表，所有配送方式都标记为可用 |

### 2.4 合并逻辑：本地候选的复用与并列返回

无论是否触发外部 webhook，本地配送方式始终会被获取和展示：

**文件**：`saleor/checkout/delivery_context.py:607-621`

```python
# 第一步：始终获取本地内置配送方式
built_in_shipping_methods_dict: dict[int, ShippingMethodData] = {
    int(shipping_method.id): shipping_method
    for shipping_method in get_available_built_in_shipping_methods_for_checkout_info(
        checkout_info=checkout_info
    )
}

def with_external_methods(external_shipping_methods: list[ShippingMethodData]):
    # 第二步：合并本地 + 外部（外部可能为空）
    external_shipping_methods_dict: dict[str, ShippingMethodData] = {
        shipping_method.id: shipping_method
        for shipping_method in external_shipping_methods
    }
    all_methods = list(built_in_shipping_methods_dict.values()) + list(
        external_shipping_methods_dict.values()
    )
```

#### 2.4.1 不同场景下的返回结果

| 场景 | 本地配送方式 | 外部配送方式 | 最终返回 |
|------|------------|------------|----------|
| 首次请求 + 有 webhook | ✅ 获取 | ✅ webhook 返回 | 本地 + 外部 并列 |
| 缓存有效 + 有 webhook | ✅ 获取（但可能走 CheckoutDelivery 缓存） | ❌ 不调用 | CheckoutDelivery 缓存中的本地 + 外部 |
| `allow_sync_webhooks=False` | ✅ 获取（走 CheckoutDelivery 缓存） | ❌ 不调用 | CheckoutDelivery 缓存中的本地 + 外部 |
| 首次请求 + 无 webhook | ✅ 获取 | ❌ 空列表 | 仅本地 |
| 缓存过期 + 无 webhook | ✅ 获取 | ❌ 空列表 | 仅本地 |

#### 2.4.2 CheckoutDelivery 缓存的作用

当不触发外部 webhook 时，系统依赖 `CheckoutDelivery` 表中的缓存记录：

```
CheckoutDelivery 表
├─ built_in_shipping_method_id (本地方式)
├─ external_shipping_method_id (外部方式，base64 编码)
├─ is_external (标记)
├─ price_amount (缓存的运费)
├─ is_valid (是否有效)
└─ active (是否可用)
```

- 这些记录是之前调用外部 webhook 时存储的
- 即使外部 webhook 暂时不可用或被禁用，用户仍然能看到之前的配送选项
- 但运费价格可能不是实时最新的

### 2.5 分支判断总览

```
get_or_fetch_checkout_deliveries()
    │
    ├─ allow_sync_webhooks == False?
    │   ├─ 是 → 直接返回 CheckoutDelivery 缓存（不触发任何 webhook）
    │   └─ 否 → 继续判断
    │
    ├─ delivery_methods_stale_at > now?
    │   ├─ 是 → 直接返回 CheckoutDelivery 缓存（不触发任何 webhook）
    │   └─ 否 → 调用 fetch_shipping_methods_for_checkout()
    │
fetch_shipping_methods_for_checkout()
    │
    ├─ [始终执行] 获取本地内置配送方式
    │
    └─ fetch_external_shipping_methods_for_checkout_info()
         │
         └─ list_shipping_methods_for_checkout()
              │
              ├─ get_webhooks_for_event(SHIPPING_LIST_METHODS_FOR_CHECKOUT)
              │   ├─ 无 webhook → 返回空列表（只展示本地方式）
              │   └─ 有 webhook → 触发同步 webhook 调用
              │
              └─ 合并本地 + 外部方式 → 转换为 CheckoutDelivery → 存储 → 返回
```

---

## 三、用户提交配送方式后的可用性校验与拒绝路径

当用户在前端选择配送方式并提交时，系统会经过多层校验。任何一层校验失败都会拒绝该配送方式的选择。

### 3.1 提交入口与全局 ID 解析

**文件**：`saleor/graphql/checkout/mutations/checkout_shipping_method_update.py:87-106`

```python
@staticmethod
def _resolve_delivery_method_id(id_) -> str | None:
    if id_ is None:
        return None

    possible_types = ("ShippingMethod", APP_ID_PREFIX)
    type_, id_ = from_global_id_or_error(id_)
    str_type = str(type_)

    if str_type not in possible_types:
        raise ValidationError(
            {
                "shipping_method_id": ValidationError(
                    "ID does not belong to known shipping methods",
                    code=CheckoutErrorCode.INVALID.value,
                )
            }
        )

    return id_
```

**第一层校验：ID 类型校验**

| 校验项 | 失败行为 | 错误码 |
|--------|---------|-------|
| GraphQL ID 类型必须是 `ShippingMethod` 或 `app` | 抛出 ValidationError | `INVALID` |

### 3.2 候选列表获取与可用性校验

**文件**：`saleor/graphql/checkout/mutations/checkout_shipping_method_update.py:108-139`

```python
@classmethod
def get_checkout_delivery(
    cls,
    checkout_info: CheckoutInfo,
    shipping_method_id: str | None,
    requestor: Union["App", "User", None],
) -> CheckoutDelivery | None:
    if shipping_method_id is None:
        return None
    
    # 第二步：获取候选列表（可能触发外部 webhook，也可能走缓存）
    checkout_deliveries = get_or_fetch_checkout_deliveries(
        checkout_info, requestor=requestor
    ).get()
    
    # 第三步：解析 ID
    internal_shipping_method_id = cls._resolve_delivery_method_id(
        shipping_method_id
    )
    if internal_shipping_method_id is None:
        return None

    # 第四步：遍历候选列表查找匹配项
    for method in checkout_deliveries:
        if not method.active:  # ← 关键校验：必须是 active
            continue
        if method.shipping_method_id == internal_shipping_method_id:
            return method

    # 第五步：未找到匹配项，拒绝
    raise ValidationError(
        {
            "shipping_method": ValidationError(
                "This shipping method is not applicable.",
                code=CheckoutErrorCode.SHIPPING_METHOD_NOT_APPLICABLE.value,
            )
        }
    )
```

**第二层至第五层校验**

| 层级 | 校验项 | 失败行为 | 错误码 |
|------|-------|---------|-------|
| 第二层 | `get_or_fetch_checkout_deliveries` 成功获取候选列表 | 依赖缓存或重新获取 | - |
| 第三层 | ID 解析成功 | 返回 None（不抛错，但会导致后续找不到） | - |
| 第四层 | 配送方式 `active=True` | 跳过该方式 | - |
| 第五层 | 在候选列表中找到匹配的 `shipping_method_id` | 抛出 ValidationError | `SHIPPING_METHOD_NOT_APPLICABLE` |

### 3.3 `active` 状态的设置逻辑

`active` 字段在 `fetch_shipping_methods_for_checkout` 中通过过滤 webhook 设置：

**文件**：`saleor/checkout/delivery_context.py:627-634`

```python
@allow_writer()
def with_excluded_methods(excluded_methods: list[ExcludedShippingMethod]):
    initialize_shipping_method_active_status(all_methods, excluded_methods)
    # ...
```

- `CHECKOUT_FILTER_SHIPPING_METHODS` webhook 返回的排除列表会将对应方式标记为 `active=False`
- 无过滤 webhook 时，所有方式默认为 `active=True`
- 外部承运商返回的方式默认 `active=True`，除非被过滤 webhook 排除

### 3.4 前置校验：是否需要配送

**文件**：`saleor/graphql/checkout/mutations/checkout_shipping_method_update.py:177-186`

```python
if use_legacy_error_flow_for_checkout and not is_shipping_required(lines):
    raise ValidationError(
        {
            "shipping_method": ValidationError(
                ERROR_DOES_NOT_SHIP,
                code=CheckoutErrorCode.SHIPPING_NOT_REQUIRED.value,
            )
        }
    )
```

| 校验项 | 失败行为 | 错误码 |
|--------|---------|-------|
| checkout 中的商品是否需要配送 | 抛出 ValidationError | `SHIPPING_NOT_REQUIRED` |

### 3.5 完整校验流程图

```
用户提交 shipping_method_id (GraphQL ID)
    │
    ├─ 解析 GraphQL ID → 验证类型是 ShippingMethod 或 app
    │   └─ 失败 → ValidationError(INVALID)
    │
    ├─ 获取候选列表 get_or_fetch_checkout_deliveries()
    │   ├─ 可能触发外部 webhook（缓存过期 + 允许同步）
    │   └─ 可能直接返回 CheckoutDelivery 缓存
    │
    ├─ 遍历候选列表
    │   ├─ 跳过 active=False 的方式
    │   └─ 匹配 shipping_method_id
    │       └─ 未找到 → ValidationError(SHIPPING_METHOD_NOT_APPLICABLE)
    │
    └─ 校验通过 → 返回 CheckoutDelivery → assign_delivery_method_to_checkout()
```

---

## 四、已选配送方式刷新与总价重算触发机制

当配送方式列表刷新（缓存过期触发 `fetch_shipping_methods_for_checkout`）时，已选配送方式可能发生变化。系统设计了两种策略来处理这种情况。

### 4.1 两种刷新策略：`overwrite_assigned_delivery`

**文件**：`saleor/checkout/delivery_context.py:587-603`

```python
def fetch_shipping_methods_for_checkout(
    checkout_info: "CheckoutInfo",
    requestor: Union["App", "User", None],
    overwrite_assigned_delivery: bool = True,  # ← 策略开关
) -> Promise[list[CheckoutDelivery]]:
```

| `overwrite_assigned_delivery` | 行为 | 适用场景 |
|----------------------------|------|---------|
| `True` | 用刷新后的数据覆盖已选配送方式 | 常规刷新，用户正在选择配送方式时 |
| `False` | 保留已选配送方式，如变化则标记为无效 | 后台刷新或订单确认前的刷新 |

### 4.2 策略一：覆盖模式（`overwrite_assigned_delivery=True`）

**文件**：`saleor/checkout/delivery_context.py:372-407`

```python
def _overwrite_assigned_delivery(
    checkout_info: "CheckoutInfo",
    assigned_delivery: CheckoutDelivery | None,
    refreshed_delivery: CheckoutDelivery | None,
):
    if not assigned_delivery:
        return

    # 更新当前已选配送方式的详情，或标记为无效
    if refreshed_delivery:
        _create_or_update_checkout_deliveries([refreshed_delivery])
    else:
        _invalidate_assigned_delivery(assigned_delivery)

    # 关键：检查价格/税费变化，触发总价重算
    if _refreshed_assigned_delivery_has_impact_on_prices(
        assigned_delivery, refreshed_delivery
    ):
        from .utils import invalidate_checkout
        invalidate_checkout(
            checkout_info=checkout_info,
            lines=checkout_info.lines,
            manager=checkout_info.manager,
            recalculate_discount=True,
            save=True,
        )
```

#### 4.2.1 价格影响判断

**文件**：`saleor/checkout/delivery_context.py:562-585`

```python
def _refreshed_assigned_delivery_has_impact_on_prices(
    assigned_delivery: CheckoutDelivery,
    refreshed_delivery: CheckoutDelivery | None,
) -> bool:
    if not refreshed_delivery:
        return True  # 已选方式消失，必须重算

    # 比较关键字段
    return (
        assigned_delivery.price_amount != refreshed_delivery.price_amount
        or assigned_delivery.tax_class_id != refreshed_delivery.tax_class_id
        or assigned_delivery.currency != refreshed_delivery.currency
    )
```

#### 4.2.2 触发总价重算的机制

**文件**：`saleor/checkout/utils.py:76-89`

```python
def invalidate_checkout(
    checkout_info: "CheckoutInfo",
    lines: list["CheckoutLineInfo"],
    manager: "PluginsManager",
    *,
    recalculate_discount: bool = True,
    save: bool,
) -> list[str]:
    if recalculate_discount:
        recalculate_checkout_discounts(checkout_info, lines, manager)

    updated_fields = invalidate_checkout_prices(checkout_info, save=save)
    return updated_fields
```

```python
def invalidate_checkout_prices(
    checkout_info: "CheckoutInfo",
    *,
    save: bool,
) -> list[str]:
    checkout = checkout_info.checkout
    price_expiration = timezone.now()
    checkout.price_expiration = price_expiration
    checkout.discount_expiration = price_expiration
    updated_fields = ["price_expiration", "discount_expiration", "last_change"]
    if save:
        checkout.save(update_fields=updated_fields)
    return updated_fields
```

**重算触发点**：
- 设置 `checkout.price_expiration = now()` 标记价格过期
- 下一次查询总价时会重新计算
- 同时重新计算折扣（如运费优惠）

### 4.3 策略二：保留模式（`overwrite_assigned_delivery=False`）

**文件**：`saleor/checkout/delivery_context.py:432-463`

```python
def _preserve_assigned_delivery(
    checkout: Checkout,
    assigned_delivery: CheckoutDelivery | None,
    refreshed_delivery: CheckoutDelivery | None,
):
    if not assigned_delivery:
        return

    # 情况1：刷新后已选方式消失 → 标记为无效
    if not refreshed_delivery:
        _invalidate_assigned_delivery(assigned_delivery)
        return

    # 情况2：检查配送方式是否变化
    delivery_changed = is_delivery_changed(assigned_delivery, refreshed_delivery)
    
    # 情况2a：未变化且当前有效 → 什么都不做
    if not delivery_changed and assigned_delivery.is_valid:
        return

    # 情况2b：未变化但当前无效 → 恢复为有效
    if not delivery_changed and not assigned_delivery.is_valid:
        _restore_assigned_delivery_as_valid(
            checkout=checkout,
            assigned_delivery=assigned_delivery,
        )
    
    # 情况2c：已变化且当前有效 → 标记原方式无效，插入新方式
    if delivery_changed and assigned_delivery.is_valid:
        _invalidate_assigned_delivery(assigned_delivery)
        _create_or_update_checkout_deliveries([refreshed_delivery])
```

#### 4.3.1 配送方式变化判断

**文件**：`saleor/checkout/delivery_context.py:359-369`

```python
def is_delivery_changed(
    first: CheckoutDelivery,
    second: CheckoutDelivery,
) -> bool:
    return (
        first.name != second.name
        or first.price != second.price  # 价格变化
        or first.tax_class_id != second.tax_class_id
        or first.maximum_delivery_days != second.maximum_delivery_days
        or first.minimum_delivery_days != second.minimum_delivery_days
    )
```

#### 4.3.2 保留模式下的总价重算

保留模式不直接调用 `invalidate_checkout`，但：
- 原已选方式被标记为 `is_valid=False` 后，用户必须重新选择
- 新方式通过 `_create_or_update_checkout_deliveries` 插入数据库
- 用户重新选择时会触发 `assign_delivery_method_to_checkout`，进而触发 `invalidate_checkout`

### 4.4 CheckoutDelivery 刷新：清理与保留

**文件**：`saleor/checkout/delivery_context.py:466-493`

```python
def _refresh_checkout_deliveries(
    checkout: "Checkout",
    assigned_delivery: CheckoutDelivery | None,
    checkout_deliveries: list["CheckoutDelivery"],
    built_in_shipping_methods_dict: dict[int, ShippingMethodData],
    external_shipping_methods_dict: dict[str, ShippingMethodData],
):
    # 构建不删除的条件：
    exclude_from_delete = Q(
        built_in_shipping_method_id__in=list(built_in_shipping_methods_dict.keys())
    ) | Q(external_shipping_method_id__in=list(external_shipping_methods_dict.keys()))

    if assigned_delivery:
        # 关键：始终保留已选配送方式，即使它不再可用
        exclude_from_delete |= Q(pk=assigned_delivery.pk)

    # 删除不再可用的方式（除了已选的）
    CheckoutDelivery.objects.filter(
        checkout_id=checkout.pk,
    ).exclude(exclude_from_delete).delete()

    # 批量 upsert 最新的方式
    if checkout_deliveries:
        _create_or_update_checkout_deliveries(checkout_deliveries)
```

### 4.5 刷新后的状态转移图

```
刷新前已选方式 (assigned_delivery)
    │
    ├─ 刷新后仍存在 (refreshed_delivery 存在)
    │   │
    │   ├─ 方式未变化 (is_delivery_changed=False)
    │   │   ├─ 当前有效 → 无操作，价格不重算
    │   │   └─ 当前无效 → 恢复有效 (is_valid=True)
    │   │
    │   └─ 方式已变化 (is_delivery_changed=True)
    │       ├─ overwrite=True → 覆盖原方式，价格变化则 invalidate_checkout
    │       └─ overwrite=False → 原方式标记无效，插入新方式，用户需重新选择
    │
    └─ 刷新后不存在 (refreshed_delivery 为 None)
        ├─ overwrite=True → 标记原方式无效，invalidate_checkout
        └─ overwrite=False → 标记原方式无效
```

### 4.6 分配配送方式时的总价重算

当用户明确选择配送方式时（`assign_delivery_method_to_checkout`），始终会触发总价重算：

**文件**：`saleor/checkout/delivery_context.py:760-800`

```python
def assign_delivery_method_to_checkout(
    checkout_info: "CheckoutInfo",
    lines_info: list["CheckoutLineInfo"],
    manager: "PluginsManager",
    delivery_method: CheckoutDelivery | Warehouse | None,
):
    fields_to_update = []
    checkout = checkout_info.checkout
    with transaction.atomic():
        if delivery_method is None:
            fields_to_update = remove_delivery_method_from_checkout(...)
        elif isinstance(delivery_method, CheckoutDelivery):
            fields_to_update = assign_shipping_method_to_checkout(
                checkout, delivery_method
            )
        # ...

        if not fields_to_update:
            return

        # 关键：始终触发 invalidate_checkout
        invalidate_prices_updated_fields = invalidate_checkout(
            checkout_info, lines_info, manager, save=False
        )
        checkout.save(update_fields=fields_to_update + invalidate_prices_updated_fields)
```

**`assign_shipping_method_to_checkout` 内部逻辑** [delivery_context.py:272-290]：

```python
def assign_shipping_method_to_checkout(
    checkout: Checkout, checkout_delivery: CheckoutDelivery
) -> list[str]:
    fields_to_update = []
    fields_to_update += remove_click_and_collect_from_checkout(checkout)
    fields_to_update += _assign_undiscounted_base_shipping_price_to_checkout(
        checkout, checkout_delivery
    )
    if checkout.assigned_delivery_id != checkout_delivery.id:
        checkout.assigned_delivery = checkout_delivery
        fields_to_update.append("assigned_delivery_id")
    # ...
    return fields_to_update
```

```python
def _assign_undiscounted_base_shipping_price_to_checkout(
    checkout: Checkout, checkout_delivery: CheckoutDelivery
) -> list[str]:
    current_shipping_price = quantize_price(
        checkout.undiscounted_base_shipping_price, checkout.currency
    )
    new_shipping_price = quantize_price(checkout_delivery.price, checkout.currency)
    if current_shipping_price != new_shipping_price:
        checkout.undiscounted_base_shipping_price_amount = new_shipping_price.amount
        return ["undiscounted_base_shipping_price_amount"]
    return []
```

---

## 五、出站请求与签名

### 5.1 外部方法获取流程

```
fetch_shipping_methods_for_checkout()
    ↓
1. 获取本地内置配送方式
   get_available_built_in_shipping_methods_for_checkout_info()
    ↓
2. 并行获取外部承运商配送方式
   fetch_external_shipping_methods_for_checkout_info()
    ↓
   list_shipping_methods_for_checkout() [webhooks/list_shipping_methods.py:23]
    ↓
   trigger_webhook_sync_promise() [transport/synchronous/transport.py:398]
```

### 5.2 同步 Webhook 调用链

**文件**：`saleor/webhook/transport/synchronous/transport.py`

#### 5.2.1 创建 Delivery 对象

```python
trigger_webhook_sync_promise()
    ↓
create_promise_delivery_for_subscription_sync_event()
    ↓
generate_payload_promise_from_subscription()  # 生成 payload
    ↓
创建 EventDelivery 和 EventPayload
```

#### 5.2.2 发送 HTTP 请求

```python
send_webhook_request_sync() [transport.py:195]
    ↓
_send_webhook_request_sync() [transport.py:103]
    ↓
1. 计算签名: signature_for_payload(message, webhook.secret_key)
2. 发送 HTTP POST 请求: send_webhook_using_http()
3. 解析 JSON 响应
```

### 5.3 签名机制

**文件**：`saleor/webhook/transport/__init__.py:7-11`

```python
def signature_for_payload(body: bytes, secret_key: str | None):
    if not secret_key:
        return get_jwt_manager().jws_encode(body)
    hash = hmac.new(bytes(secret_key, "utf-8"), body, hashlib.sha256)
    return hash.hexdigest()
```

- **签名算法**：HMAC-SHA256
- **密钥**：Webhook 的 `secret_key`
- **无密钥时**：使用 JWS 编码

### 5.4 请求头信息

**文件**：`saleor/webhook/transport/utils.py:226-236`

```python
headers = {
    "Content-Type": "application/json",
    # 新版标准头
    "Saleor-Event-Type": event_type,
    "Saleor-Domain": domain,
    "Saleor-Signature": signature,
    "Saleor-Api-Url": api_url,
    # 旧版兼容头 (X- 前缀，将在 4.0 弃用)
    "X-Saleor-Event-Type": event_type,
    "X-Saleor-Domain": domain,
    "X-Saleor-Signature": signature,
}
```

---

## 四、响应处理与总价计算

### 4.1 响应解析

**文件**：`saleor/checkout/webhooks/list_shipping_methods.py:64-91`

```python
_parse_list_shipping_methods_response()
    ↓
ListShippingMethodsSchema.model_validate()  # Pydantic 验证
    ↓
转换为 ShippingMethodData 对象列表
```

**响应 Schema**（`saleor/webhook/response_schemas/shipping.py`）：
- `id`: 承运商内部配送方式 ID
- `name`: 配送方式名称
- `amount`: 运费金额
- `currency`: 货币代码（必须与 checkout 货币一致）
- `maximum_delivery_days`: 最长配送天数
- `minimum_delivery_days`: 最短配送天数
- `description`: 描述
- `metadata`: 元数据

### 4.2 ID 编码机制

**文件**：`saleor/webhook/transport/shipping_helpers.py:7-11`

```python
def to_shipping_app_id(app: App, shipping_method_id: str) -> str:
    app_identifier = app.identifier or app.id
    return base64.b64encode(
        str.encode(f"{APP_ID_PREFIX}:{app_identifier}:{shipping_method_id}")
    ).decode("utf-8")
```

- 外部承运商返回的 ID 会被编码为：`base64("app:<app_identifier>:<method_id>")`
- 用于区分本地配送方式（整数 ID）和外部配送方式（base64 编码字符串）

### 6.3 与本地方式合并展示

**文件**：`saleor/checkout/delivery_context.py:587-709`

```python
fetch_shipping_methods_for_checkout()
    ↓
built_in_methods = get_available_built_in_shipping_methods_for_checkout_info()
external_methods = fetch_external_shipping_methods_for_checkout_info()
    ↓
all_methods = built_in_methods + external_methods  # 合并
    ↓
excluded_shipping_methods_for_checkout()  # 过滤排除
    ↓
initialize_shipping_method_active_status()  # 设置 active 状态
    ↓
转换为 CheckoutDelivery 并存储
```

**关键合并逻辑** [delivery_context.py:614-621]：
```python
def with_external_methods(external_shipping_methods: list[ShippingMethodData]):
    external_shipping_methods_dict: dict[str, ShippingMethodData] = {
        shipping_method.id: shipping_method
        for shipping_method in external_shipping_methods
    }
    all_methods = list(built_in_shipping_methods_dict.values()) + list(
        external_shipping_methods_dict.values()
    )
```

### 6.4 存储到 CheckoutDelivery

**文件**：`saleor/shipping/utils.py:91-127`

```python
convert_shipping_method_data_to_checkout_delivery()
    ↓
CheckoutDelivery(
    external_shipping_method_id=... if is_external else None,
    built_in_shipping_method_id=... if not is_external else None,
    is_external=shipping_method_is_external,
    price_amount=shipping_method_data.price.amount,
    currency=shipping_method_data.price.currency,
    ...
)
```

**CheckoutDelivery 模型关键字段** [checkout/models.py:38-108]：
- `external_shipping_method_id`: 外部承运商配送方式 ID（base64 编码）
- `built_in_shipping_method_id`: 本地配送方式 ID
- `is_external`: 标记是否为外部承运商
- `price_amount`: 运费金额
- `currency`: 货币
- `is_valid`: 是否有效
- `active`: 是否可用（可能被过滤 webhook 排除）

### 6.5 分配到 Checkout

**文件**：`saleor/checkout/delivery_context.py:760-800`

```python
assign_delivery_method_to_checkout()
    ↓
assign_shipping_method_to_checkout()
    ↓
1. 设置 checkout.assigned_delivery = checkout_delivery
2. 设置 checkout.shipping_method_name = delivery.name
3. 设置 checkout.undiscounted_base_shipping_price_amount
4. invalidate_checkout()  # 标记价格需要重新计算
5. 触发 CHECKOUT_UPDATED 异步 webhook
```

### 6.6 运费并入 Checkout 总价

#### 6.6.1 基础运费计算

**文件**：`saleor/checkout/base_calculations.py:89-156`

```python
base_checkout_delivery_price()
    ↓
base_checkout_undiscounted_delivery_price()
    ↓
calculate_base_price_for_shipping_method()
    ↓
return shipping_method.price  # 直接使用 ShippingMethodData.price
```

#### 6.6.2 总价计算流程

**文件**：`saleor/checkout/calculations.py:165-191`

```python
calculate_checkout_total()
    ↓
fetch_checkout_data()  # 确保价格最新
    ↓
_calculate_checkout_total()
    ↓
total = checkout.subtotal + checkout.shipping_price
```

#### 6.6.3 价格字段关系

```
CheckoutDelivery.price_amount
    ↓ (assign 时赋值)
Checkout.undiscounted_base_shipping_price_amount
    ↓ (base_calculations)
Checkout.shipping_price (TaxedMoney: net + gross)
    ↓ (checkout_total)
Checkout.total = subtotal + shipping_price - discount
```

---

## 七、完整调用链路图

```
GraphQL: checkoutShippingMethodUpdate
    │
    ├─→ 解析 shipping_method_id (GraphQL ID → 内部 ID)
    │    └─→ 验证类型是 ShippingMethod 或 app
    │
    ├─→ get_checkout_delivery()
    │    │
    │    └─→ get_or_fetch_checkout_deliveries()
    │         │
    │         ├─→ 检查缓存 delivery_methods_stale_at
    │         │    ├─→ 缓存有效 → 返回 CheckoutDelivery 缓存
    │         │    └─→ 缓存过期 → fetch_shipping_methods_for_checkout()
    │         │
    │         └─→ allow_sync_webhooks?
    │              ├─→ False → 返回 CheckoutDelivery 缓存
    │              └─→ True → fetch_shipping_methods_for_checkout()
    │
    ├─→ 遍历候选列表: 匹配 shipping_method_id + active=True
    │    └─→ 未找到 → ValidationError(SHIPPING_METHOD_NOT_APPLICABLE)
    │
    └─→ assign_delivery_method_to_checkout()
         │
         ├─→ assign_shipping_method_to_checkout()
         │    ├─→ 设置 checkout.assigned_delivery
         │    └─→ 设置 undiscounted_base_shipping_price
         │
         └─→ invalidate_checkout() → 标记 price_expiration = now()


fetch_shipping_methods_for_checkout()
    │
    ├─→ [同步] 获取本地内置配送方式
    │
    ├─→ [异步] fetch_external_shipping_methods_for_checkout_info()
    │    │
    │    └─→ list_shipping_methods_for_checkout()
    │         │
    │         ├─→ get_webhooks_for_event(SHIPPING_LIST_METHODS_FOR_CHECKOUT)
    │         │    ├─→ 无 webhook → 返回空列表
    │         │    └─→ 有 webhook → 同步调用外部承运商
    │         │
    │         ├─→ [签名] signature_for_payload()
    │         ├─→ send_webhook_using_http()
    │         └─→ 解析响应 → ShippingMethodData 列表
    │
    ├─→ 合并 all_methods = built_in + external
    │
    ├─→ CHECKOUT_FILTER_SHIPPING_METHODS (可选过滤)
    │
    ├─→ refresh_assigned_delivery
    │    ├─→ overwrite=True → _overwrite_assigned_delivery()
    │    │    └─→ 价格/税费变化 → invalidate_checkout()
    │    └─→ overwrite=False → _preserve_assigned_delivery()
    │         ├─→ 方式未变化 → 无操作
    │         └─→ 方式已变化 → 标记无效 + 插入新方式
    │
    ├─→ _refresh_checkout_deliveries()
    │    ├─→ 删除不再可用的方式（保留已选方式）
    │    └─→ 批量 upsert CheckoutDelivery
    │
    └─→ 设置 delivery_methods_stale_at = now() + TTL
```

---

## 八、关键代码文件索引

| 模块 | 文件路径 | 核心功能 |
|------|----------|----------|
| 触发入口 | `saleor/graphql/checkout/mutations/checkout_shipping_method_update.py` | Mutation 入口、可用性校验 |
| 配送上下文 | `saleor/checkout/delivery_context.py` | 配送方式获取、合并、分配、缓存判断、刷新策略 |
| 外部方法获取 | `saleor/checkout/webhooks/list_shipping_methods.py` | 调用外部承运商 webhook |
| 过滤排除 | `saleor/checkout/webhooks/exclude_shipping.py` | 排除不可用配送方式 |
| 同步传输 | `saleor/webhook/transport/synchronous/transport.py` | 同步 webhook 请求发送 |
| 签名 | `saleor/webhook/transport/__init__.py` | 请求签名计算 |
| 响应解析 | `saleor/webhook/response_schemas/shipping.py` | Pydantic 响应验证 |
| 工具函数 | `saleor/webhook/transport/shipping_helpers.py` | ID 编码 |
| Webhook 查询 | `saleor/webhook/utils.py` | `get_webhooks_for_event` 查询可用 webhook |
| 数据转换 | `saleor/shipping/utils.py` | ShippingMethodData ↔ CheckoutDelivery 转换 |
| 价格计算 | `saleor/checkout/base_calculations.py` | 基础运费计算 |
| 总价计算 | `saleor/checkout/calculations.py` | Checkout 总价计算、`allow_sync_webhooks` 处理 |
| 价格失效 | `saleor/checkout/utils.py` | `invalidate_checkout`、`invalidate_checkout_prices` |
| 数据模型 | `saleor/checkout/models.py` | Checkout、CheckoutDelivery 模型 |
| 接口定义 | `saleor/shipping/interface.py` | ShippingMethodData 数据类 |
| 配置 | `saleor/settings.py` | `CHECKOUT_DELIVERY_OPTIONS_TTL` 等配置 |

---

## 九、核心设计特点

1. **并行获取**：本地和外部配送方式并行获取，通过 Promise 异步处理
2. **统一抽象**：`ShippingMethodData` 统一表示本地和外部配送方式
3. **缓存机制**：`delivery_methods_stale_at` 避免频繁调用 webhook，默认 24 小时有效期
4. **ID 编码**：base64 编码区分本地/外部配送方式
5. **过滤机制**：`CHECKOUT_FILTER_SHIPPING_METHODS` webhook 支持二次过滤
6. **原子更新**：使用 `bulk_create` + `update_conflicts` 批量更新 CheckoutDelivery
7. **价格解耦**：运费存储在 CheckoutDelivery，计算时直接读取
8. **优雅降级**：多层回退机制确保外部 webhook 不可用时仍能正常工作
   - `allow_sync_webhooks=False`：全局禁用同步调用，直接使用缓存
   - `delivery_methods_stale_at` 未过期：使用 CheckoutDelivery 缓存
   - 无可用 webhook：只展示本地配送方式
9. **数据持久化**：外部承运商配送方式持久化到 CheckoutDelivery 表，即使 webhook 暂时不可用也能展示历史选项
10. **多层可用性校验**：
    - ID 类型校验（ShippingMethod 或 app）
    - 候选列表存在性校验
    - `active=True` 状态校验
    - 不需要配送的前置校验
11. **双策略刷新机制**：
    - 覆盖模式（`overwrite=True`）：刷新后数据覆盖已选方式，价格变化自动触发总价重算
    - 保留模式（`overwrite=False`）：保留已选方式，变化则标记为无效
12. **已选方式保护**：`_refresh_checkout_deliveries` 始终保留已选配送方式，即使它不再可用
13. **价格失效标记**：通过 `price_expiration` 标记价格过期，而非立即重算，实现懒加载重算
14. **折扣联动**：运费变化时自动重新计算折扣（如运费优惠）
