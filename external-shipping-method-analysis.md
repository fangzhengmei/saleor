# 外部承运商运费 Webhook 计算路径分析

## 概述

外部承运商运费计算通过同步 Webhook 机制实现，整个流程分为三个核心阶段：**触发条件识别**、**出站请求与签名**、**响应处理与总价计算**。外部承运商返回的运费选项与本地（built-in）配送方式并列展示给用户。

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
CheckoutShippingMethodUpdate.perform_mutation()
    ↓
get_or_fetch_checkout_deliveries() [delivery_context.py:732]
    ↓
检查 delivery_methods_stale_at 是否已过期
    ↓ （如过期或首次请求）
fetch_shipping_methods_for_checkout() [delivery_context.py:587]
```

### 1.3 关键判断逻辑

在 `get_or_fetch_checkout_deliveries` [delivery_context.py:732-757] 中：

```python
if (
    checkout.delivery_methods_stale_at is None
    or checkout.delivery_methods_stale_at <= timezone.now()
) and allow_sync_webhooks:
    return fetch_shipping_methods_for_checkout(checkout_info, requestor=requestor)
```

- `delivery_methods_stale_at` 控制缓存有效期（默认 `CHECKOUT_DELIVERY_OPTIONS_TTL`）
- `allow_sync_webhooks` 标志可禁用同步 webhook 调用

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

## 三、出站请求与签名

### 3.1 外部方法获取流程

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

### 3.2 同步 Webhook 调用链

**文件**：`saleor/webhook/transport/synchronous/transport.py`

#### 3.2.1 创建 Delivery 对象

```python
trigger_webhook_sync_promise()
    ↓
create_promise_delivery_for_subscription_sync_event()
    ↓
generate_payload_promise_from_subscription()  # 生成 payload
    ↓
创建 EventDelivery 和 EventPayload
```

#### 3.2.2 发送 HTTP 请求

```python
send_webhook_request_sync() [transport.py:195]
    ↓
_send_webhook_request_sync() [transport.py:103]
    ↓
1. 计算签名: signature_for_payload(message, webhook.secret_key)
2. 发送 HTTP POST 请求: send_webhook_using_http()
3. 解析 JSON 响应
```

### 3.3 签名机制

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

### 3.4 请求头信息

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

### 4.3 与本地方式合并展示

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

### 4.4 存储到 CheckoutDelivery

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

### 4.5 分配到 Checkout

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

### 4.6 运费并入 Checkout 总价

#### 4.6.1 基础运费计算

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

#### 4.6.2 总价计算流程

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

#### 4.6.3 价格字段关系

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

## 五、完整调用链路图

```
GraphQL: checkoutShippingMethodUpdate
    │
    ├─→ get_or_fetch_checkout_deliveries()
    │    │
    │    └─→ fetch_shipping_methods_for_checkout()
    │         │
    │         ├─→ [本地] get_available_built_in_shipping_methods_for_checkout_info()
    │         │
    │         └─→ [外部] fetch_external_shipping_methods_for_checkout_info()
    │              │
    │              └─→ list_shipping_methods_for_checkout()
    │                   │
    │                   ├─→ get_webhooks_for_event(SHIPPING_LIST_METHODS_FOR_CHECKOUT)
    │                   │
    │                   ├─→ generate_checkout_payload()
    │                   │
    │                   └─→ trigger_webhook_sync_promise()
    │                        │
    │                        ├─→ [签名] signature_for_payload()
    │                        ├─→ send_webhook_using_http()
    │                        └─→ _parse_list_shipping_methods_response()
    │
    ├─→ 合并 all_methods = built_in + external
    │
    ├─→ excluded_shipping_methods_for_checkout()  # 可选过滤
    │
    ├─→ convert_shipping_method_data_to_checkout_delivery()
    │
    ├─→ _create_or_update_checkout_deliveries()  # 批量 upsert
    │
    └─→ assign_delivery_method_to_checkout()
         │
         ├─→ 设置 checkout.assigned_delivery
         ├─→ 设置 undiscounted_base_shipping_price
         └─→ invalidate_checkout()  # 触发重新计算总价
```

---

## 六、关键代码文件索引

| 模块 | 文件路径 | 核心功能 |
|------|----------|----------|
| 触发入口 | `saleor/graphql/checkout/mutations/checkout_shipping_method_update.py` | Mutation 入口 |
| 配送上下文 | `saleor/checkout/delivery_context.py` | 配送方式获取、合并、分配、缓存判断 |
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
| 数据模型 | `saleor/checkout/models.py` | Checkout、CheckoutDelivery 模型 |
| 接口定义 | `saleor/shipping/interface.py` | ShippingMethodData 数据类 |
| 配置 | `saleor/settings.py` | `CHECKOUT_DELIVERY_OPTIONS_TTL` 等配置 |

---

## 七、核心设计特点

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
