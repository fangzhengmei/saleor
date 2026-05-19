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

## 二、出站请求与签名

### 2.1 外部方法获取流程

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

### 2.2 同步 Webhook 调用链

**文件**：`saleor/webhook/transport/synchronous/transport.py`

#### 2.2.1 创建 Delivery 对象

```python
trigger_webhook_sync_promise()
    ↓
create_promise_delivery_for_subscription_sync_event()
    ↓
generate_payload_promise_from_subscription()  # 生成 payload
    ↓
创建 EventDelivery 和 EventPayload
```

#### 2.2.2 发送 HTTP 请求

```python
send_webhook_request_sync() [transport.py:195]
    ↓
_send_webhook_request_sync() [transport.py:103]
    ↓
1. 计算签名: signature_for_payload(message, webhook.secret_key)
2. 发送 HTTP POST 请求: send_webhook_using_http()
3. 解析 JSON 响应
```

### 2.3 签名机制

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

### 2.4 请求头信息

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

## 三、响应处理与总价计算

### 3.1 响应解析

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

### 3.2 ID 编码机制

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

### 3.3 与本地方式合并展示

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

### 3.4 存储到 CheckoutDelivery

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

### 3.5 分配到 Checkout

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

### 3.6 运费并入 Checkout 总价

#### 3.6.1 基础运费计算

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

#### 3.6.2 总价计算流程

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

#### 3.6.3 价格字段关系

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

## 四、完整调用链路图

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

## 五、关键代码文件索引

| 模块 | 文件路径 | 核心功能 |
|------|----------|----------|
| 触发入口 | `saleor/graphql/checkout/mutations/checkout_shipping_method_update.py` | Mutation 入口 |
| 配送上下文 | `saleor/checkout/delivery_context.py` | 配送方式获取、合并、分配 |
| 外部方法获取 | `saleor/checkout/webhooks/list_shipping_methods.py` | 调用外部承运商 webhook |
| 过滤排除 | `saleor/checkout/webhooks/exclude_shipping.py` | 排除不可用配送方式 |
| 同步传输 | `saleor/webhook/transport/synchronous/transport.py` | 同步 webhook 请求发送 |
| 签名 | `saleor/webhook/transport/__init__.py` | 请求签名计算 |
| 响应解析 | `saleor/webhook/response_schemas/shipping.py` | Pydantic 响应验证 |
| 工具函数 | `saleor/webhook/transport/shipping_helpers.py` | ID 编码 |
| 数据转换 | `saleor/shipping/utils.py` | ShippingMethodData ↔ CheckoutDelivery 转换 |
| 价格计算 | `saleor/checkout/base_calculations.py` | 基础运费计算 |
| 总价计算 | `saleor/checkout/calculations.py` | Checkout 总价计算 |
| 数据模型 | `saleor/checkout/models.py` | Checkout、CheckoutDelivery 模型 |
| 接口定义 | `saleor/shipping/interface.py` | ShippingMethodData 数据类 |

---

## 六、核心设计特点

1. **并行获取**：本地和外部配送方式并行获取，通过 Promise 异步处理
2. **统一抽象**：`ShippingMethodData` 统一表示本地和外部配送方式
3. **缓存机制**：`delivery_methods_stale_at` 避免频繁调用 webhook
4. **ID 编码**：base64 编码区分本地/外部配送方式
5. **过滤机制**：`CHECKOUT_FILTER_SHIPPING_METHODS` webhook 支持二次过滤
6. **原子更新**：使用 `bulk_create` + `update_conflicts` 批量更新 CheckoutDelivery
7. **价格解耦**：运费存储在 CheckoutDelivery，计算时直接读取
