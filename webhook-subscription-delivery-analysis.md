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

## 六、文件索引

| 模块 | 文件 | 核心职责 |
|------|------|----------|
| 异步传输 | `saleor/webhook/transport/asynchronous/transport.py` | 投递触发、任务定义、延迟 payload |
| 传输工具 | `saleor/webhook/transport/utils.py` | HTTP/SQS/PubSub 发送、重试逻辑、状态管理 |
| 订阅载荷 | `saleor/graphql/webhook/subscription_payload.py` | GraphQL 查询执行、payload 生成 |
| 订阅类型 | `saleor/graphql/webhook/subscription_types.py` | 订阅类型定义、字段解析 |
