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

### 7.1 事件类型分流机制

Saleor 从事件类型层面就将 webhook 分为两条独立链路，通过 `WebhookEventAsyncType` 和 `WebhookEventSyncType` 两个枚举类 [event_types.py] 进行分流：

**异步事件类型**（`WebhookEventAsyncType`）：
- 用于通知类场景，如 `ORDER_CREATED`, `PRODUCT_UPDATED`, `CUSTOMER_CREATED` 等
- 触发后立即返回，不等待 webhook 响应
- 支持指数退避重试机制

**同步事件类型**（`WebhookEventSyncType`）：
- 用于需要响应结果的场景，如：
  - 支付类：`PAYMENT_AUTHORIZE`, `PAYMENT_CAPTURE`, `PAYMENT_REFUND`
  - 税费计算：`CHECKOUT_CALCULATE_TAXES`, `ORDER_CALCULATE_TAXES`
  - 交易类：`TRANSACTION_CHARGE_REQUESTED`, `TRANSACTION_REFUND_REQUESTED`
  - 配送方式：`SHIPPING_LIST_METHODS_FOR_CHECKOUT`, `CHECKOUT_FILTER_SHIPPING_METHODS`
  - 支付会话：`TRANSACTION_INITIALIZE_SESSION`, `TRANSACTION_PROCESS_SESSION`
- 触发后阻塞等待 webhook 响应，响应结果直接影响业务流程
- 仅支持 HTTP/HTTPS scheme，不支持 SQS/PubSub

### 7.2 同步链路跳过的订阅载荷渲染步骤

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

**同步 subscription 模式跳过的步骤**：
1. **跳过 Webhook 分组**：同步事件在触发时已确定是同步链路，不经过 `group_webhooks_by_subscription()`
2. **跳过批量处理**：同步事件单次只处理一个 webhook，不进行批量创建
3. **跳过预保存 payload 比较**：同步事件无 `pre_save_payloads` 机制
4. **跳过延迟载荷模式**：同步事件必须即时生成 payload，不支持 deferred payload
5. **跳过 dataloader 跨 webhook 共享**：同步事件通常只有一个 webhook，无共享优化
6. **跳过 Celery 任务队列**：同步事件在当前线程直接执行，不经过任务队列
7. **跳过自动清理**：失败的 delivery 会调用 `save_unsuccessful_delivery_attempt()` 保留记录用于排查

### 7.3 Payload 生成失败/为空的处理策略

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
  - 调用方收到 `None` 后通常会创建失败事件，如 `create_failed_transaction_event()`
- **重试**：无重试，同步事件失败直接影响业务流程

**同步 legacy webhook**：
- **行为**：使用预先生成的 `static_payload`，不会出现 payload 为空的情况
- **短路时机**：仅在 HTTP 请求失败时短路

### 7.4 重试机制差异对比

| 维度 | 异步 Subscription | 同步 Subscription | 同步 Legacy |
|------|------------------|------------------|------------|
| **重试触发** | 5xx 错误、网络超时 | 5xx 错误（仅交易类任务） | 5xx 错误（仅交易类任务） |
| **最大重试** | 5 次 | 5 次（仅 `handle_transaction_request_task`） | 5 次（仅 `handle_transaction_request_task`） |
| **退避策略** | 指数退避（10s * 2^n） | 指数退避（10s * 2^n） | 指数退避（10s * 2^n） |
| **3xx/4xx 重试** | 不重试 | 不重试 | 不重试 |
| **payload 失败重试** | 不重试（根本不创建 delivery） | 不重试（返回 None 短路） | 不存在此场景 |
| **缓存机制** | 无 | 有（`SYNC_WEBHOOK_FAILURE_SENTINEL` 防止重复失败请求） | 有 |

### 7.5 两条链路的完整分流流程

```
事件触发
    ↓
判断事件类型 ∈ WebhookEventSyncType.ALL ?
    ├─ 是 → 同步链路
    │    └─ 判断 webhook.subscription_query ?
    │         ├─ 是 → 同步 Subscription 模式
    │         │    ├─ 调用 generate_payload_from_subscription()
    │         │    ├─ 如 payload 为空 → return None → 短路
    │         │    ├─ 创建 EventDelivery（可选保存）
    │         │    └─ 同步发送 HTTP 请求 → 等待响应 → 返回结果
    │         └─ 否 → 同步 Legacy 模式
    │              ├─ 使用预生成 static_payload
    │              ├─ 创建 EventDelivery（内存对象）
    │              └─ 同步发送 HTTP 请求 → 等待响应 → 返回结果
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
