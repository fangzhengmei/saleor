# Saleor Webhook 投递与失败重试策略解析

## 整体架构概览

Saleor 的 webhook 系统采用 **同步 + 异步** 混合模式，通过 Celery 任务队列实现异步投递，配合完善的签名、重试、断路器机制确保可靠性。核心协作流程如下：

```
业务事件 → WebhookPlugin.trigger_webhooks_async()
    ↓
call_event() → 事务提交后触发
    ↓
create_deliveries_for_subscriptions() → 生成 EventDelivery
    ↓
send_webhook_request_async.apply_async() → 提交到 Celery 队列
    ↓
send_webhook_using_http() / send_webhook_using_aws_sqs() / send_webhook_using_google_cloud_pubsub()
    ↓
成功 → 清理 EventDelivery
失败 → handle_webhook_retry() → 指数退避重试
超过最大重试 → 标记 FAILED（死信）
```

---

## 1. 事件分发机制

### 1.1 触发入口：WebhookPlugin

`WebhookPlugin` ([saleor/plugins/webhook/plugin.py](saleor/plugins/webhook/plugin.py)) 是所有 webhook 事件的统一分发入口。每个业务事件（如 `order_created`、`product_updated`）都会调用 plugin 对应的方法。

核心分发流程：

```python
# WebhookPlugin 中的典型调用
def order_created(self, order, previous_value, webhooks=None):
    if webhooks := self._get_webhooks_for_channel_events(
        event_type, order.channel.slug, webhooks
    ):
        self.trigger_webhooks_async(
            None, event_type, webhooks, order, self.requestor,
            legacy_data_generator=order_data_generator,
            queue=settings.ORDER_WEBHOOK_EVENTS_CELERY_QUEUE_NAME,
        )
```

### 1.2 事务安全：call_event

`call_event` ([saleor/core/utils/events.py:19](saleor/core/utils/events.py#L19)) 确保事件在数据库事务提交后才触发，避免投递任务读取到未提交的数据。

```python
def call_event(func_obj, *func_args, **func_kwargs):
    connection = transaction.get_connection()
    if connection.in_atomic_block:
        transaction.on_commit(lambda: func_obj(*func_args, **func_kwargs))
    else:
        func_obj(*func_args, **func_kwargs)
```

### 1.3 同步 vs 异步 Webhook

| 特性 | 同步 Webhook | 异步 Webhook |
|------|-------------|-------------|
| 用途 | 支付网关、税务计算等需要实时响应的场景 | 通知类事件（订单创建、商品更新等） |
| 执行方式 | 阻塞等待响应 | 后台 Celery 任务 |
| 超时设置 | `WEBHOOK_SYNC_TIMEOUT` (18s) | `WEBHOOK_TIMEOUT` (18s) |
| 重试次数 | 5次 | 5次 |
| 断路器 | 支持 | 不支持 |
| 缓存 | 支持响应缓存（5分钟） | 不支持 |

**同步 Webhook 入口**：`trigger_webhook_sync_promise()` ([saleor/webhook/transport/synchronous/transport.py:398](saleor/webhook/transport/synchronous/transport.py#L398))

**异步 Webhook 入口**：`trigger_webhooks_async()` ([saleor/webhook/transport/asynchronous/transport.py:477](saleor/webhook/transport/asynchronous/transport.py#L477))

### 1.4 延迟 Payload 生成 (Deferred Payload)

对于计算成本高的事件（如库存变动、价格变动），支持先创建 EventDelivery 再异步生成 payload，避免阻塞主业务流程。

关键代码：`create_deliveries_for_deferred_payload_subscriptions()` ([saleor/webhook/transport/asynchronous/transport.py:255](saleor/webhook/transport/asynchronous/transport.py#L255))

```
触发事件 → 创建 EventDelivery（无 payload）
    ↓
generate_deferred_payloads 任务 → 延迟生成 payload
    ↓
send_webhook_request_async 任务 → 实际投递
```

延迟 payload 任务本身最多重试 **12 次**，采用指数退避。

---

## 2. 签名机制

### 2.1 签名生成

`signature_for_payload` ([saleor/webhook/transport/__init__.py:7](saleor/webhook/transport/__init__.py#L7)) 实现签名逻辑：

```python
def signature_for_payload(body: bytes, secret_key: str | None):
    if not secret_key:
        return get_jwt_manager().jws_encode(body)
    hash = hmac.new(bytes(secret_key, "utf-8"), body, hashlib.sha256)
    return hash.hexdigest()
```

**两种签名模式**：
1. **HMAC-SHA256**：当 webhook 配置了 `secret_key` 时使用，生成十六进制字符串
2. **JWS 签名**：未配置 `secret_key` 时使用 Saleor 的 JWT 密钥签名

### 2.2 HTTP 请求头

`send_webhook_using_http()` ([saleor/webhook/transport/utils.py:205](saleor/webhook/transport/utils.py#L205)) 会同时发送新旧两种格式的头部：

```python
headers = {
    "Content-Type": "application/json",
    # 旧格式（将在 Saleor 4.0 废弃）
    "X-Saleor-Event": event_type,
    "X-Saleor-Domain": domain,
    "X-Saleor-Signature": signature,
    # 新格式
    "Saleor-Event": event_type,
    "Saleor-Domain": domain,
    "Saleor-Signature": signature,
    "Saleor-Api-Url": api_url,
}
```

### 2.3 其他协议的签名传递

- **AWS SQS**：签名放在 `MessageAttributes.Signature` 中
- **Google Cloud Pub/Sub**：签名放在消息属性 `signature` 中

---

## 3. 重试策略

### 3.1 异步 Webhook 重试配置

`send_webhook_request_async` 任务 ([saleor/webhook/transport/asynchronous/transport.py:761](saleor/webhook/transport/asynchronous/transport.py#L761))：

```python
@app.task(
    queue=settings.WEBHOOK_CELERY_QUEUE_NAME,
    bind=True,
    retry_backoff=10,
    retry_kwargs={"max_retries": 5},
)
def send_webhook_request_async(self, event_delivery_id, ...):
```

### 3.2 重试触发逻辑

`handle_webhook_retry` ([saleor/webhook/transport/utils.py:406](saleor/webhook/transport/utils.py#L406)) 控制重试行为：

```python
def handle_webhook_retry(celery_task, webhook, response, delivery, delivery_attempt):
    # 3xx 和 4xx 状态码不重试
    if response.response_status_code and 300 <= response.response_status_code < 500:
        return False
    
    # 指数退避：countdown = 10 * (2^retries)
    countdown = celery_task.retry_backoff * (2**celery_task.request.retries)
    celery_task.retry(countdown=countdown, **celery_task.retry_kwargs)
```

### 3.3 重试节奏（指数退避）

假设 `retry_backoff=10`，`max_retries=5`：

| 重试次数 | 等待时间 | 累计时间 |
|---------|---------|---------|
| 第1次重试 | 10 秒 | 10 秒 |
| 第2次重试 | 20 秒 | 30 秒 |
| 第3次重试 | 40 秒 | 70 秒 |
| 第4次重试 | 80 秒 | 150 秒 |
| 第5次重试 | 160 秒 | 310 秒 |

**总最大耗时：约 5 分钟 10 秒**

### 3.4 同步 Webhook 重试

交易类同步 webhook (`handle_transaction_request_task`) 同样配置：
- `retry_backoff=10`, `max_retries=5`
- 只有 `response_status_code >= 500` 才重试

---

## 4. 死信处理

### 4.1 死信判定

Saleor **没有专门的死信队列**，而是通过状态标记实现死信处理：

1. 超过 `max_retries`（5次）后，捕获 `MaxRetriesExceededError`
2. 将 `EventDelivery.status` 设置为 `EventDeliveryStatus.FAILED`
3. 记录错误日志，任务结束

关键代码 ([saleor/webhook/transport/utils.py:457](saleor/webhook/transport/utils.py#L457))：

```python
except MaxRetriesExceededError:
    is_success = False
    task_logger.info(
        "[Webhook ID: %r] Failed request to %r: exceeded retry limit. Delivery ID: %r",
        webhook.id, webhook.target_url, delivery.id,
    )
```

### 4.2 数据保留策略

**成功投递**：`clear_successful_delivery()` ([saleor/webhook/transport/utils.py:607](saleor/webhook/transport/utils.py#L607)) 会立即删除：
- `EventDelivery` 记录
- 关联的 `EventPayload`（如果没有其他 delivery 引用）
- 存储的 payload 文件

**失败投递**：保留在数据库中，包括：
- `EventDelivery` (status = FAILED)
- `EventDeliveryAttempt`（每次尝试的详细信息）
- `EventPayload`（payload 被转存为文件）

可以通过 GraphQL API 查询失败的投递：`eventDeliveries` 查询，过滤 `status: FAILED`

### 4.3 可观测性

每次投递尝试都会：
1. 创建 `EventDeliveryAttempt` 记录，包含：
   - 响应状态码、响应内容（截断到配置大小）
   - 请求/响应头部
   - 耗时
   - 状态
2. 上报 metrics：`record_external_request()`
3. 触发可观测性 webhook（如果配置了）

---

## 5. 断路器 (Circuit Breaker)

### 5.1 适用范围

仅对 **同步 Webhook** 生效，防止下游服务故障时雪崩。

`BreakerBoard` ([saleor/webhook/circuit_breaker/breaker_board.py:68](saleor/webhook/circuit_breaker/breaker_board.py#L68)) 实现断路器模式。

### 5.2 状态转换

```
          错误率 > 35% 且 错误数 > 100
    CLOSED ──────────────────────────────────→ OPEN
      ↑                                            │
      │ 成功数 > 50                      冷却 2分钟 │
      │                                            ↓
      └────────── HALF_OPEN ←────────────────── HALF_OPEN
          错误率 > 30% 且 错误数 > 20
```

### 5.3 关键参数

| 参数 | 默认值 | 说明 |
|-----|-------|------|
| `BREAKER_BOARD_TTL_SECONDS` | 300 | 统计时间窗口（5分钟） |
| `BREAKER_BOARD_FAILURE_MIN_COUNT` | 100 | 打开断路器的最小错误数 |
| `BREAKER_BOARD_FAILURE_THRESHOLD_PERCENTAGE` | 35 | 打开断路器的错误率阈值 |
| `BREAKER_BOARD_COOLDOWN_SECONDS` | 120 | OPEN 状态冷却时间 |
| `BREAKER_BOARD_SUCCESS_COUNT_RECOVERY` | 50 | HALF_OPEN → CLOSED 需要的成功数 |

### 5.4 OPEN 状态行为

当断路器打开时：
- 非 dry-run 事件：直接返回 `None`，不发起实际请求
- dry-run 事件：继续执行但忽略结果（用于测试）

---

## 6. 核心数据模型

### 6.1 Webhook ([saleor/webhook/models.py:18](saleor/webhook/models.py#L18))

```python
class Webhook(models.Model):
    name = models.CharField(max_length=255)
    app = models.ForeignKey(App, on_delete=models.CASCADE)
    target_url = WebhookURLField(max_length=255)  # http/https/awssqs/gcpubsub
    is_active = models.BooleanField(default=True)
    secret_key = models.CharField(max_length=255)  # 用于 HMAC 签名
    subscription_query = models.TextField()       # GraphQL 订阅查询
    custom_headers = models.JSONField()           # 自定义请求头
    filterable_channel_slugs = ArrayField(...)    # 渠道过滤
```

### 6.2 EventDelivery ([saleor/core/models.py:206](saleor/core/models.py#L206))

```python
class EventDelivery(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    status = models.CharField(choices=EventDeliveryStatus.CHOICES)  # PENDING/SUCCESS/FAILED
    event_type = models.CharField(max_length=255)
    payload = models.ForeignKey(EventPayload, on_delete=models.CASCADE)
    webhook = models.ForeignKey(Webhook, on_delete=models.CASCADE)
```

### 6.3 EventDeliveryAttempt ([saleor/core/models.py:223](saleor/core/models.py#L223))

```python
class EventDeliveryAttempt(models.Model):
    delivery = models.ForeignKey(EventDelivery, related_name="attempts")
    task_id = models.CharField(max_length=255)  # Celery task ID
    duration = models.FloatField()              # 请求耗时（秒）
    response = models.TextField()               # 响应内容（截断）
    response_status_code = models.PositiveSmallIntegerField()
    request_headers = models.TextField()
    response_headers = models.TextField()
    status = models.CharField(choices=EventDeliveryStatus.CHOICES)
```

### 6.4 EventPayload ([saleor/core/models.py:171](saleor/core/models.py#L171))

```python
class EventPayload(models.Model):
    payload = models.TextField(default="")                    # 小 payload 直接存
    payload_file = models.FileField(storage=private_storage)  # 大 payload 存文件
    created_at = models.DateTimeField(auto_now_add=True)
```

Payload 超过一定大小时自动转存文件（`save_as_file()`），避免数据库膨胀。

---

## 7. 投递协议支持

### 7.1 HTTP/HTTPS

使用 `send_webhook_using_http()`，通过 `HTTPClient` 发送 POST 请求：
- 禁止重定向 (`allow_redirects=False`)
- IP 过滤（防止 SSRF）
- 连接超时 + 读取超时配置

### 7.2 AWS SQS

使用 `send_webhook_using_aws_sqs()`：
- 从 URL 解析 region、access key、secret key
- FIFO 队列自动设置 `MessageGroupId`
- 签名放在 MessageAttributes 中

### 7.3 Google Cloud Pub/Sub

使用 `send_webhook_using_google_cloud_pubsub()`：
- 使用应用默认凭据
- 签名放在消息属性中

---

## 8. 关键配置项

```python
# settings.py 中的关键配置
WEBHOOK_TIMEOUT = (3.05, 18)  # (连接超时, 读取超时)
WEBHOOK_SYNC_TIMEOUT = (3.05, 18)
WEBHOOK_WAITING_FOR_RESPONSE_TIMEOUT = 18

# 队列配置
WEBHOOK_CELERY_QUEUE_NAME = "webhooks"
WEBHOOK_SQS_CELERY_QUEUE_NAME = "webhooks_sqs"
WEBHOOK_PUBSUB_CELERY_QUEUE_NAME = "webhooks_pubsub"
WEBHOOK_DEFERRED_PAYLOAD_QUEUE_NAME = "webhooks_deferred_payload"
ORDER_WEBHOOK_EVENTS_CELERY_QUEUE_NAME = "order_webhooks"

# 断路器
BREAKER_BOARD_ENABLED = False
BREAKER_BOARD_SYNC_EVENTS = [...]
BREAKER_BOARD_DRY_RUN_SYNC_EVENTS = [...]

# 缓存
WEBHOOK_CACHE_DEFAULT_TTL = 5 * 60  # 5分钟
SYNC_WEBHOOK_FAILURE_CACHE_TTL = 1  # 1秒

# 响应大小限制
EVENT_DELIVERY_ATTEMPT_RESPONSE_SIZE_LIMIT = 100000
```

---

## 9. 典型失败场景处理

| 场景 | 处理方式 |
|-----|---------|
| 网络连接错误 | 标记 FAILED，触发重试 |
| 5xx 服务器错误 | 触发指数退避重试 |
| 4xx 客户端错误（如 404、403） | 不重试，直接标记 FAILED |
| 3xx 重定向 | 不重试，直接标记 FAILED（禁止重定向） |
| 无效 IP 地址（SSRF 防护） | 不重试，标记 FAILED |
| payload 解析失败 | 不重试，标记 FAILED |
| webhook/app 被禁用 | 不重试，标记 FAILED |
| 数据库事务未提交 | `call_event` 自动延迟到 `on_commit` |
| 主从延迟（延迟 payload） | 最多 12 次重试等待从库同步 |

---

## 10. 关键代码文件索引

| 文件 | 核心职责 |
|-----|---------|
| [saleor/webhook/transport/utils.py](saleor/webhook/transport/utils.py) | HTTP/SQS/PubSub 投递、重试逻辑、attempt 管理 |
| [saleor/webhook/transport/asynchronous/transport.py](saleor/webhook/transport/asynchronous/transport.py) | 异步 webhook 任务、延迟 payload、批量处理 |
| [saleor/webhook/transport/synchronous/transport.py](saleor/webhook/transport/synchronous/transport.py) | 同步 webhook、交易请求、缓存逻辑 |
| [saleor/webhook/circuit_breaker/breaker_board.py](saleor/webhook/circuit_breaker/breaker_board.py) | 断路器实现 |
| [saleor/webhook/transport/__init__.py](saleor/webhook/transport/__init__.py) | 签名生成 |
| [saleor/core/utils/events.py](saleor/core/utils/events.py) | 事务安全事件触发 |
| [saleor/core/models.py](saleor/core/models.py) | EventDelivery、EventDeliveryAttempt、EventPayload 模型 |
| [saleor/plugins/webhook/plugin.py](saleor/plugins/webhook/plugin.py) | WebhookPlugin 事件分发入口 |
