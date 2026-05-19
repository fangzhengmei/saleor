# Saleor 基于 Subscription 的 Webhook 投递机制分析

## 概述

Saleor 的订阅式 Webhook 投递系统与传统同步 Webhook 不同，它通过**事件订阅查询**、**GraphQL 子集载荷构造**和**异步重试发送**三个核心阶段的协作，实现灵活、可扩展的事件通知机制。

---

## 一、订阅匹配阶段

### 1.1 Webhook 分组

入口函数 `group_webhooks_by_subscription()` [transport.py:290-299] 将 Webhook 分为两类：

```python
def group_webhooks_by_subscription(webhooks: Sequence["Webhook"]) -> tuple[list["Webhook"], list["Webhook"]]:
    subscription = [webhook for webhook in webhooks if webhook.subscription_query]
    legacy = [webhook for webhook in webhooks if not webhook.subscription_query]
    return legacy, subscription
```

- **Subscription Webhook**：带有 `subscription_query` 字段，使用 GraphQL 订阅查询生成载荷
- **Legacy Webhook**：无订阅查询，使用预定义的完整 JSON 载荷

### 1.2 事件类型可订阅性检查

在 `create_deliveries_for_multiple_subscription_objects()` [transport.py:111-115] 中验证事件类型：

```python
if event_type not in WEBHOOK_TYPES_MAP:
    logger.info("Skipping subscription webhook. Event %s is not subscribable.", event_type)
    return Promise.resolve([])
```

`WEBHOOK_TYPES_MAP` [subscription_types.py] 定义了所有可订阅的事件类型及其对应的 GraphQL 类型，例如：
- `OrderCreated`, `ProductUpdated`, `CustomerCreated` 等
- 每个事件类型通过 `SubscriptionObjectType` 定义，包含 `root_type` 和 `interfaces`

### 1.3 请求上下文初始化

为每个 App 初始化独立的 `SaleorContext` 请求上下文 [transport.py:128-139]：

```python
request = request_map.get(webhook.app_id)
if not request:
    request = initialize_request(
        app=webhook.app,
        requestor=requestor,
        sync_event=is_sync_event,
        event_type=event_type,
        allow_replica=allow_replica,
        request_time=request_time,
        dataloaders=dataloaders,
    )
    request_map[webhook.app_id] = request
```

**关键优化**：
- 同一 App 的多个 Webhook 共享同一个请求上下文
- 跨 Webhook 共享 `dataloaders` 字典，避免重复数据库查询

---

## 二、GraphQL 子集载荷构造阶段

### 2.1 核心函数：generate_payload_promise_from_subscription()

定义在 [subscription_payload.py:102-186]，使用 GraphQL 引擎执行订阅查询：

```python
def generate_payload_promise_from_subscription(
    event_type: str,
    subscribable_object,
    subscription_query: str,
    request: SaleorContext,
) -> Promise[dict[str, Any] | None]:
    graphql_backend = get_default_backend()
    ast = parse(subscription_query)
    document = graphql_backend.document_from_string(schema, ast)
    
    results_promise = document.execute(
        allow_subscriptions=True,
        root=(event_type, subscribable_object),
        context=get_context_value(request),
        return_promise=True,
    )
```

**执行流程**：
1. 解析订阅查询字符串为 AST
2. 使用 Saleor GraphQL Schema 编译文档
3. 传入 `(event_type, subscribable_object)` 作为根值
4. 异步执行查询，返回 Promise

### 2.2 根值解析机制

Subscription 类型通过 `resolve_type` [subscription_types.py:136-138] 将 `(event_type, subscribable_object)` 元组映射到具体的订阅类型：

```python
@classmethod
def resolve_type(cls, instance, info: ResolveInfo):
    type_str, _ = instance
    return cls.get_type(type_str)
```

每个订阅类型（如 `OrderCreated`）定义了如何从根值中提取数据：

```python
class OrderCreated(SubscriptionObjectType, OrderBase):
    @staticmethod
    def resolve_order(root, info: ResolveInfo):
        _, order = root
        return SyncWebhookControlContext(order)
```

### 2.3 载荷后处理

`_process_payload_instance_async()` [subscription_payload.py:70-86] 标准化输出格式：

```python
def _process_payload_instance_async(payload_instance):
    ((key, value),) = payload_instance.data.items()
    def process_single_payload(data: dict[str, Any] | None) -> dict[str, Any]:
        if "event" == key:
            return data or {}
        return {"data": {key: data}}
```

### 2.4 延迟载荷模式 (Deferred Payload)

对于高并发场景，Saleor 提供延迟载荷生成模式 [transport.py:255-287]：

```python
def create_deliveries_for_deferred_payload_subscriptions(...):
    # 先创建 EventDelivery（无 payload）
    deliveries_to_create = []
    for webhook in webhooks:
        delivery = EventDelivery(
            status=EventDeliveryStatus.PENDING,
            event_type=event_type,
            webhook=webhook,
        )
        deliveries_to_create.append(delivery)
    
    EventDelivery.objects.bulk_create(deliveries_to_create)
    
    # 触发异步 payload 生成任务
    generate_deferred_payloads.apply_async(...)
```

**优势**：
- 主流程快速返回，不阻塞
- Payload 生成可以在 replica 数据库上执行
- 支持跨节点负载均衡

延迟任务 `generate_deferred_payloads` [transport.py:535-608]：
1. 检查 replica 数据库上 EventDelivery 的可用性
2. 如不可用则指数退避重试（最多 12 次）
3. 重建 subscribable_object（支持 Django model 和 dataclass）
4. 调用 `generate_payload_promise_from_subscription` 生成 payload
5. 更新 EventDelivery 并触发投递任务

---

## 三、异步重试发送阶段

### 3.1 投递任务：send_webhook_request_async()

定义在 [transport.py:761-844]，Celery 任务配置：

```python
@app.task(
    queue=settings.WEBHOOK_CELERY_QUEUE_NAME,
    bind=True,
    retry_backoff=10,
    retry_kwargs={"max_retries": 5},
)
def send_webhook_request_async(self, event_delivery_id, ...):
```

**执行流程**：
1. 从数据库获取 EventDelivery 和 Webhook
2. 创建 EventDeliveryAttempt 记录
3. 根据 target_url scheme 选择发送方式：
   - HTTP/HTTPS：`send_webhook_using_http()`
   - AWS SQS：`send_webhook_using_aws_sqs()`
   - Google Cloud Pub/Sub：`send_webhook_using_google_cloud_pubsub()`

### 3.2 重试策略

`handle_webhook_retry()` [utils.py:406-466] 实现智能重试：

```python
def handle_webhook_retry(celery_task, webhook, response, delivery, delivery_attempt):
    # 3xx 和 4xx 错误不重试
    if response.response_status_code and 300 <= response.response_status_code < 500:
        return False
    
    # 指数退避：retry_backoff * (2^retries)
    countdown = celery_task.retry_backoff * (2**celery_task.request.retries)
    celery_task.retry(countdown=countdown, **celery_task.retry_kwargs)
```

**重试参数**：
- 初始退避：10 秒
- 指数因子：2
- 最大重试：5 次
- 最大延迟：10 * 2^4 = 160 秒

### 3.3 投递状态管理

| 状态 | 说明 |
|------|------|
| `PENDING` | 待投递 |
| `SUCCESS` | 投递成功 |
| `FAILED` | 投递失败（已耗尽重试） |

**成功清理**：`clear_successful_deliveries()` [utils.py:612-643] 自动删除成功的 EventDelivery 和关联的 EventPayload，避免数据库膨胀。

### 3.4 批量投递优化

`send_webhooks_async_for_app()` [transport.py:846-944] 支持按 App 批量投递：

```python
def send_webhooks_async_for_app(self, app_id, ...):
    deliveries = get_deliveries_for_app(app_id, WEBHOOK_ASYNC_BATCH_SIZE)
    # 批量创建 attempts
    # 批量发送请求
    # 批量更新状态
```

**自调度模式**：任务完成后自动调度下一次执行，实现连续处理。

---

## 四、三段协作完整流程

```
事件触发
    ↓
[订阅匹配阶段]
    ├─ 按 subscription_query 分组 Webhook
    ├─ 检查事件类型可订阅性
    └─ 初始化请求上下文（共享 dataloaders）
    ↓
[载荷构造阶段]
    ├─ 模式选择：即时 / 延迟
    │   ├─ 即时：Promise.all 并行执行所有订阅查询
    │   └─ 延迟：先创建 EventDelivery，异步生成 payload
    ├─ GraphQL 执行：使用订阅查询生成子集载荷
    ├─ 预保存比较：如 payload 未变化则跳过投递
    └─ 批量创建 EventPayload 和 EventDelivery
    ↓
[重试发送阶段]
    ├─ 触发 send_webhook_request_async 任务
    ├─ 按 scheme 发送（HTTP/SQS/PubSub）
    ├─ 成功：清理 delivery 和 payload
    └─ 失败：指数退避重试，最多 5 次
```

---

## 五、关键技术点

### 5.1 Dataloader 共享

跨 Webhook 共享 dataloaders [transport.py:118, 127]，避免 N+1 查询问题：
```python
dataloaders: dict[str, type[DataLoader]] = {}  # 所有 Webhook 共享
request = initialize_request(..., dataloaders=dataloaders)
```

### 5.2 预保存 payload 比较

`generate_pre_save_payloads()` [subscription_payload.py:263-309] 在保存前生成 payload，保存后比较，如无变化则跳过投递，减少无效 Webhook 调用。

### 5.3 数据库事务一致性

EventPayload 和 EventDelivery 的创建在同一事务中 [transport.py:195-204]，避免数据库状态不一致。

### 5.4 可观测性

- OpenTelemetry tracing：`webhooks_otel_trace()`
- 指标收集：`record_external_request()`, `record_first_delivery_attempt_delay()`
- 结构化日志：包含 webhook_id, event_type, duration 等字段

---

## 七、同步 Webhook 与 Subscription Webhook 协作差异

### 7.1 同步事件的两类执行路径

Saleor 的同步事件（`WebhookEventSyncType`）并非统一执行路径，而是根据事件类型分为**交易类同步事件**和**普通同步事件**两类，二者在重试机制、执行线程、payload 处理上存在显著差异。

#### 7.1.1 交易类同步事件（走 Celery 队列重试）

**事件类型**：
- `TRANSACTION_CHARGE_REQUESTED`
- `TRANSACTION_REFUND_REQUESTED`
- `TRANSACTION_CANCELATION_REQUESTED`

**执行入口**：`trigger_transaction_request()` [synchronous/transport.py:451-516]

```python
def trigger_transaction_request(transaction_data, event_type, requestor):
    # 1. 创建 delivery（同步生成 payload）
    if webhook.subscription_query:
        delivery = create_delivery_for_subscription_sync_event(...)
    else:
        payload = generate_transaction_action_request_payload(...)
        delivery = EventDelivery.objects.create(...)
    
    # 2. 关键：投递到 Celery 队列，异步执行
    call_event(
        handle_transaction_request_task.delay,
        delivery.id,
        transaction_data.event.id,
    )
```

**关键特征**：
- **执行线程**：创建 delivery 后立即返回，实际发送由 `handle_transaction_request_task` Celery 任务执行
- **重试机制**：Celery 任务配置 `max_retries=5` + `retry_backoff=10`，5xx 错误指数退避重试
- **时效性**：不阻塞当前请求线程，异步完成
- **持久化**：EventDelivery 和 EventPayload 均持久化到数据库

**Celery 任务执行流程** [synchronous/transport.py:69-100]：
```python
@app.task(bind=True, retry_backoff=10, retry_kwargs={"max_retries": 5})
def handle_transaction_request_task(self, delivery_id, request_event_id):
    delivery, _ = get_delivery_for_webhook(delivery_id)
    attempt = create_attempt(delivery, self.request.id)
    response, response_data = _send_webhook_request_sync(delivery, attempt=attempt)
    
    # 仅 5xx 错误重试
    if response.response_status_code and response.response_status_code >= 500:
        handle_webhook_retry(self, delivery.webhook, response, delivery, attempt)
```

#### 7.1.2 普通同步事件（请求线程内直接发送，无重试）

**事件类型**：
- **支付类（旧API）**：`PAYMENT_AUTHORIZE`, `PAYMENT_CAPTURE`, `PAYMENT_REFUND`, `PAYMENT_VOID`, `PAYMENT_CONFIRM`, `PAYMENT_PROCESS`
- **税费计算**：`CHECKOUT_CALCULATE_TAXES`, `ORDER_CALCULATE_TAXES`
- **配送方式**：`SHIPPING_LIST_METHODS_FOR_CHECKOUT`, `CHECKOUT_FILTER_SHIPPING_METHODS`, `ORDER_FILTER_SHIPPING_METHODS`
- **支付会话**：`TRANSACTION_INITIALIZE_SESSION`, `TRANSACTION_PROCESS_SESSION`, `PAYMENT_GATEWAY_INITIALIZE_SESSION`
- **支付方式管理**：`LIST_STORED_PAYMENT_METHODS`, `STORED_PAYMENT_METHOD_DELETE_REQUESTED`
- **支付令牌化**：`PAYMENT_GATEWAY_INITIALIZE_TOKENIZATION_SESSION`, `PAYMENT_METHOD_INITIALIZE_TOKENIZATION_SESSION`, `PAYMENT_METHOD_PROCESS_TOKENIZATION_SESSION`

**执行入口**：`trigger_webhook_sync_promise()` [synchronous/transport.py:398-442]

**关键特征**：
- **执行线程**：在当前请求线程内同步执行，阻塞等待响应
- **重试机制**：**无任何重试**，一次发送失败即返回 None
- **时效性**：直接影响当前请求响应时间
- **持久化**：
  - Subscription 模式：根据 `with_save` 参数决定是否保存（默认 `True`）
  - Legacy 模式：EventDelivery 仅为内存对象，不持久化到数据库
  - 失败时调用 `save_unsuccessful_delivery_attempt()` 保存失败记录

### 7.2 两类同步事件的核心差异对比

| 维度 | 交易类同步事件 | 普通同步事件 |
|------|--------------|------------|
| **执行方式** | Celery 任务异步执行 | 当前线程同步执行 |
| **重试机制** | 5 次指数退避（仅 5xx） | 无重试 |
| **是否阻塞请求** | 否 | 是 |
| **Delivery 持久化** | 总是持久化 | Subscription 可选，Legacy 不持久化 |
| **调用方处理** | 触发后立即返回，无需等待 | 等待响应结果，直接用于业务逻辑 |
| **失败处理** | 重试耗尽后创建失败交易事件 | 返回 None，调用方自行处理 |

### 7.3 同步链路跳过的订阅载荷渲染步骤

同步 webhook 虽然也支持 `subscription_query`，但其执行链路与异步有显著差异：

**触发入口分流** [synchronous/transport.py:398-442]：
```python
def trigger_webhook_sync_promise(*, event_type, webhook, ...):
    if webhook.subscription_query:
        # 同步 subscription 模式
        delivery_promise = create_promise_delivery_for_subscription_sync_event(...)
    else:
        # 同步 legacy 模式 - 直接使用静态 payload
        delivery_promise = Promise.resolve(
            EventDelivery(
                status=EventDeliveryStatus.PENDING,
                event_type=event_type,
                payload=EventPayload(payload=static_payload),
                webhook=webhook,
            )
        )
    return delivery_promise.then(trigger_sync_for_delivery)
```

**同步 subscription 模式跳过的步骤**（普通同步事件）：
1. **跳过 Webhook 分组**：同步事件在触发时已确定是同步链路，不经过 `group_webhooks_by_subscription()`
2. **跳过批量处理**：同步事件单次只处理一个 webhook，不进行批量创建
3. **跳过预保存 payload 比较**：同步事件无 `pre_save_payloads` 机制
4. **跳过延迟载荷模式**：同步事件必须即时生成 payload，不支持 deferred payload
5. **跳过 dataloader 跨 webhook 共享**：同步事件通常只有一个 webhook，无共享优化
6. **跳过 Celery 任务队列**：普通同步事件在当前线程直接执行，不经过任务队列
7. **跳过自动清理**：失败的 delivery 会调用 `save_unsuccessful_delivery_attempt()` 保留记录用于排查

### 7.4 Payload 生成失败/为空的处理策略

**异步 subscription webhook** [asynchronous/transport.py:156-179]：
```python
for (subscribable_object, webhook), data in zip(...):
    if not data:
        logger.info("No payload was generated with subscription for event: %s", event_type)
        continue  # 静默跳过，不创建 EventDelivery
```
- **行为**：payload 为空时直接 `continue`，不创建 EventDelivery 和 EventPayload
- **重试**：无重试，因为根本没有创建 delivery
- **日志**：仅记录 info 级别日志，不视为错误

**同步 subscription webhook** [synchronous/transport.py:263-326]：
```python
def create_delivery_for_subscription_sync_event(...):
    data = generate_payload_from_subscription(...)
    if not data:
        logger.info("No payload was generated with subscription for event: %s", event_type)
        return None  # 返回 None，短路后续流程
```
- **行为**：payload 为空时返回 `None`，不创建 EventDelivery
- **短路时机**：
  - 在 `trigger_webhook_sync_promise` 中，`delivery` 为 `None` 时直接返回 `None` [synchronous/transport.py:413-415]
  - 交易类事件：调用方收到 `None` 后创建失败交易事件 `create_failed_transaction_event()`
  - 普通同步事件：调用方收到 `None` 后按业务逻辑处理（如税费计算失败返回错误）
- **重试**：无论交易类还是普通同步事件，payload 生成都不重试

**同步 legacy webhook**：
- **行为**：使用预先生成的 `static_payload`，不会出现 payload 为空的情况
- **短路时机**：仅在 HTTP 请求失败时短路

### 7.5 缓存哨兵（SYNC_WEBHOOK_FAILURE_SENTINEL）的作用范围

**定义** [const.py]：
```python
SYNC_WEBHOOK_FAILURE_SENTINEL = object()  # 标记失败的缓存值
SYNC_WEBHOOK_FAILURE_CACHE_TTL = 30  # 失败缓存 30 秒
```

**仅作用于 `trigger_webhook_sync_promise_if_not_cached()`** [synchronous/transport.py:202-260]：

```python
def trigger_webhook_sync_promise_if_not_cached(...):
    cache_key = generate_cache_key_for_webhook(...)
    response_data = cache.get(cache_key)
    
    # 关键：如果缓存中是失败哨兵，直接跳过
    if response_data == const.SYNC_WEBHOOK_FAILURE_SENTINEL:
        logger.warning("Skipping request to %s for event %s due to previous failure.", ...)
        return Promise.resolve(None)
    
    if response_data is not None:
        return Promise.resolve(response_data)  # 命中成功缓存
    
    # 未命中缓存，实际发送请求
    return trigger_webhook_sync_promise(...).then(process_response_data)

def process_response_data(response_data):
    if response_data is not None:
        cache.set(cache_key, response_data, timeout=cache_timeout)  # 成功缓存
    else:
        cache.set(cache_key, const.SYNC_WEBHOOK_FAILURE_SENTINEL, 
                  timeout=const.SYNC_WEBHOOK_FAILURE_CACHE_TTL)  # 失败缓存
    return response_data
```

**实际使用场景**：
1. **配送方式过滤** [shipping/webhooks/shared.py:106]：
   - `CHECKOUT_FILTER_SHIPPING_METHODS`
   - `ORDER_FILTER_SHIPPING_METHODS`
   - 缓存 3 分钟（`CACHE_EXCLUDED_SHIPPING_TIME = 60 * 3`）

2. **列出已保存支付方式** [plugins/webhook/plugin.py:2800]：
   - `LIST_STORED_PAYMENT_METHODS`
   - 缓存默认 TTL（`WEBHOOK_CACHE_DEFAULT_TTL`）

**不使用缓存哨兵的场景**：
- 所有异步事件
- 交易类同步事件（`TRANSACTION_*_REQUESTED`）
- 税费计算（`CHECKOUT_CALCULATE_TAXES`, `ORDER_CALCULATE_TAXES`）
- 支付会话初始化/处理（`TRANSACTION_INITIALIZE_SESSION`, `TRANSACTION_PROCESS_SESSION`）
- 旧支付 API（`PAYMENT_AUTHORIZE`, `PAYMENT_CAPTURE` 等）
- 支付令牌化相关事件

### 7.6 完整分流流程（修正版）

```
事件触发
    ↓
判断事件类型 ∈ WebhookEventSyncType.ALL ?
    ├─ 是 → 同步链路
    │    ├─ 判断是否为交易类事件（TRANSACTION_*_REQUESTED）?
    │    │    ├─ 是 → 交易类同步事件路径
    │    │    │    ├─ 判断 webhook.subscription_query ?
    │    │    │    │    ├─ 是 → 调用 create_delivery_for_subscription_sync_event()
    │    │    │    │    │    ├─ 生成 payload
    │    │    │    │    │    ├─ 如 payload 为空 → return None → 短路 → 创建失败交易事件
    │    │    │    │    │    └─ 创建并持久化 EventDelivery + EventPayload
    │    │    │    │    └─ 否 → 使用预生成 payload，创建并持久化 EventDelivery
    │    │    │    └─ 触发 handle_transaction_request_task.delay() → 进入 Celery 队列
    │    │    │         └─ 任务执行：发送请求 → 5xx 错误指数退避重试（最多 5 次）
    │    │    └─ 否 → 普通同步事件路径（请求线程内执行）
    │    │         ├─ 判断是否使用缓存（trigger_webhook_sync_promise_if_not_cached）?
    │    │         │    ├─ 是 → 检查缓存
    │    │         │    │    ├─ 命中失败哨兵 → 直接返回 None（短路 30 秒）
    │    │         │    │    ├─ 命中成功缓存 → 返回缓存数据
    │    │         │    │    └─ 未命中 → 继续执行
    │    │         │    └─ 否 → 直接执行
    │    │         ├─ 判断 webhook.subscription_query ?
    │    │         │    ├─ 是 → 同步 Subscription 模式
    │    │         │    │    ├─ 调用 generate_payload_from_subscription()
    │    │         │    │    ├─ 如 payload 为空 → return None → 短路
    │    │         │    │    └─ 创建 EventDelivery（可选保存）
    │    │         │    └─ 否 → 同步 Legacy 模式
    │    │         │         ├─ 使用预生成 static_payload
    │    │         │         └─ 创建 EventDelivery（内存对象，不持久化）
    │    │         └─ 当前线程同步发送 HTTP 请求 → 等待响应 → 返回结果（无重试）
    └─ 否 → 异步链路
         └─ 调用 group_webhooks_by_subscription()
              ├─ subscription webhook → 异步 Subscription 模式
              │    ├─ 并行调用 generate_payload_promise_from_subscription()
              │    ├─ 如 payload 为空 → continue → 不创建 delivery
              │    ├─ 预保存 payload 比较（可选跳过）
              │    ├─ 批量创建 EventDelivery + EventPayload
              │    └─ 触发 send_webhook_request_async Celery 任务
              │         ├─ 发送请求
              │         ├─ 失败 → 指数退避重试（最多 5 次）
              │         └─ 成功 → 清理 delivery 和 payload
              └─ legacy webhook → 异步 Legacy 模式
                   ├─ 使用预生成 payload
                   ├─ 创建 EventDelivery + EventPayload
                   └─ 触发 send_webhook_request_async Celery 任务
```

---

## 八、文件索引

| 模块 | 文件 | 核心职责 |
|------|------|----------|
| 异步传输 | `saleor/webhook/transport/asynchronous/transport.py` | 投递触发、任务定义、延迟 payload |
| 同步传输 | `saleor/webhook/transport/synchronous/transport.py` | 同步 webhook 触发、交易请求处理 |
| 传输工具 | `saleor/webhook/transport/utils.py` | HTTP/SQS/PubSub 发送、重试逻辑、状态管理 |
| 事件类型 | `saleor/webhook/event_types.py` | 异步/同步事件类型定义、权限映射 |
| 订阅载荷 | `saleor/graphql/webhook/subscription_payload.py` | GraphQL 查询执行、payload 生成 |
| 订阅类型 | `saleor/graphql/webhook/subscription_types.py` | 订阅类型定义、字段解析 |
