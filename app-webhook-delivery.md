# Saleor Webhook 投递与失败重试策略解析

## 整体架构概览

Saleor 的 webhook 系统采用 **同步 + 异步** 混合模式，通过 Celery 任务队列实现异步投递，配合完善的签名、重试、断路器机制确保可靠性。核心协作流程如下：

```
业务事件 → WebhookPlugin.trigger_webhooks_async()
    ↓
call_event() → 事务提交后触发
    ↓
create_deliveries_for_subscriptions() → 生成 EventDelivery (PENDING)
    ↓
         ┌───────────────────────────────────┐
         │  两条独立的异步投递路径              │
         └─────────────┬─────────────────────┘
                       │
    ┌──────────────────┴────────────────────┐
    │                                       │
┌───▼───────────────────────────┐   ┌───────▼──────────────────────────────┐
│ 路径一：单条任务路径          │   │ 路径二：按应用批量路径               │
│ (当前默认)                    │   │ (未来替代方案，TODO: 待去重机制完善) │
│ send_webhook_request_async    │   │ send_webhooks_async_for_app         │
│ • 每个 delivery 一个任务      │   │ • 按 app 批量处理 PENDING delivery  │
│ • Celery 内置重试             │   │ • 自调度轮询，无 Celery 重试         │
│ • 指数退避：约 5 分钟耗尽     │   │ • 无退避：约 10 秒耗尽               │
│ • 3xx/4xx → 立即 FAILED     │   │ • 所有错误 → 保持 PENDING              │
│ • 5xx/网络错误 → 保持 PENDING  │   │ • 不检查 is_active                  │
│ • 检查 is_active              │   │ • attempt_count >= 5 → 死信          │
│ • MaxRetriesExceeded → 死信  │   │                                      │
└──────────────┬───────────────┘   └───────────────┬──────────────────────┘
               │                                    │
               └──────────────────┬─────────────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │  send_webhook_using_http()  │
                    │  send_webhook_using_aws_sqs() │
                    │  send_webhook_using_google_cloud_pubsub() │
                    └─────────────┬──────────────┘
                                  │
                ┌─────────────────┴─────────────────┐
                │                                   │
        ┌───────▼─────────┐                ┌────────▼──────────┐
        │  成功投递         │                │  失败投递          │
        │  clear_successful_│                │  status = FAILED   │
        │  delivery / _    │                │  保留所有记录      │
        │  deliveries      │                │  attempts 列表     │
        └───────────────────┘                └───────────────────┘
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

### 3.5 异步投递的两条路径：单条任务 vs 按应用批量

Saleor 异步 webhook 存在 **两条独立的投递路径**，它们的重试计数、失败标记和成功清理机制存在显著差异。

#### 3.5.1 路径一：单条任务路径 (`send_webhook_request_async`)

**当前默认使用的路径**，每个 EventDelivery 对应一个独立的 Celery 任务。

```python
# [saleor/webhook/transport/asynchronous/transport.py:761](saleor/webhook/transport/asynchronous/transport.py#L761)
@app.task(
    queue=settings.WEBHOOK_CELERY_QUEUE_NAME,
    bind=True,
    retry_backoff=10,
    retry_kwargs={"max_retries": 5},
)
def send_webhook_request_async(self, event_delivery_id, ...):
    delivery, not_found = get_delivery_for_webhook(event_delivery_id)
    attempt = create_attempt(delivery, self.request.id)  # 立即创建并保存 attempt
    
    # ... 投递逻辑 ...
    
    if response.status == FAILED:
        attempt_update(attempt, response)  # 1. attempt 先保存（总是执行）
        if retry_on_failure:
            handle_webhook_retry(self, webhook, response, delivery, attempt)  # 2. 可能抛出异常
        delivery_update(delivery, FAILED)  # ⚠️ 触发重试时这行不会执行！
    elif SUCCESS:
        delivery.status = SUCCESS
        attempt_update(attempt, response, with_save=False)
    
    clear_successful_delivery(delivery)  # 单条清理
```

#### 3.5.1.1 Retry 抛出后的关键时序

**⚠️ 重要修正**：之前的理解存在严重偏差。实际执行时序如下：

```
response.status == FAILED
    ↓
attempt_update(attempt, response)  ✅ 执行，attempt 落盘（状态=FAILED）
    ↓
retry_on_failure == True
    ↓
handle_webhook_retry()
    ├─ 检查状态码：3xx/4xx → return False → 继续执行 delivery_update
    └─ 5xx/网络错误 → celery_task.retry()
           ├─ 抛出 Retry 异常
           ├─ 捕获后上报可观测性
           └─ 重新抛出 Retry 异常 → 函数终止 ⚠️
    ↓
delivery_update(delivery, FAILED)  ❌ 触发重试时永远不会执行！
```

**关键结论**：
- 当 **触发重试**（5xx/网络错误）时：`delivery.status` 始终保持 `PENDING`，只有 `attempt` 被标记为 `FAILED`
- 当 **不触发重试**（3xx/4xx）时：`handle_webhook_retry()` 返回 `False`，继续执行 `delivery_update(delivery, FAILED)`
- 当 **超过最大重试**：`handle_webhook_retry()` 捕获 `MaxRetriesExceededError` 后返回 `False`，继续执行 `delivery_update(delivery, FAILED)`

**核心机制**：

| 机制 | 实现方式 | 代码位置 |
|-----|---------|---------|
| **重试计数** | Celery 内置 `self.request.retries`，存储在 Celery 消息元数据中 | [transport.py:793](saleor/webhook/transport/asynchronous/transport.py#L793) |
| **重试触发** | `handle_webhook_retry()` → `celery_task.retry()` 抛出 `Retry` 异常，由 Celery 调度器重入队 | [utils.py:451](saleor/webhook/transport/utils.py#L451) |
| **重试节奏** | 指数退避 `countdown = 10 * (2^retries)`，共 5 次重试 | [utils.py:451](saleor/webhook/transport/utils.py#L451) |
| **失败标记** | 3xx/4xx 立即标记 FAILED；5xx/网络错误触发重试时保持 PENDING，重试耗尽后才标记 FAILED | [transport.py:830](saleor/webhook/transport/asynchronous/transport.py#L830) |
| **死信判定** | 捕获 `MaxRetriesExceededError` 后返回 False，继续执行 `delivery_update(delivery, FAILED)` | [utils.py:457](saleor/webhook/transport/utils.py#L457) |
| **成功清理** | `clear_successful_delivery(delivery)` 单条删除 | [transport.py:843](saleor/webhook/transport/asynchronous/transport.py#L843) |
| **Attempt 创建** | `create_attempt(delivery, task_id)` 立即单条保存 | [transport.py:780](saleor/webhook/transport/asynchronous/transport.py#L780) |
| **禁用 webhook/app 处理** | `get_multiple_deliveries_for_webhooks()` 中检查 `is_active`，非活跃的立即标记 FAILED，**不创建 attempt，不触发重试** | [utils.py:529](saleor/webhook/transport/utils.py#L529) |
| **重试时 delivery 状态** | 始终保持 PENDING，直到重试耗尽或成功 | 代码时序分析 |

#### 3.5.2 路径二：按应用批量路径 (`send_webhooks_async_for_app`)

**未来替代路径**（代码中已有 TODO 注释说明待去重机制完善后切换），按应用批量处理 PENDING 状态的投递。

```python
# [saleor/webhook/transport/asynchronous/transport.py:852](saleor/webhook/transport/asynchronous/transport.py#L852)
@app.task(
    queue=settings.WEBHOOK_CELERY_QUEUE_NAME,
    bind=True,
    # 注意：没有 retry_backoff 和 retry_kwargs！
)
def send_webhooks_async_for_app(self, app_id, ...):
    # 查询时就统计了 attempt 数量
    deliveries = get_deliveries_for_app(app_id, WEBHOOK_ASYNC_BATCH_SIZE)
    # deliveries 包含 attempts_count = Count("attempts", distinct=True)
    
    attempts_for_deliveries = create_attempts_for_deliveries(deliveries, self.request.id)
    
    for delivery_id, delivery_with_count in deliveries.items():
        delivery = delivery_with_count.delivery
        attempt_count = delivery_with_count.count  # 数据库中已有的 attempt 数
        attempt = attempts_for_deliveries[delivery_id]
        
        # ... 投递逻辑 ...
        
        if FAILED:
            failed_deliveries_attempts.append((delivery, attempt, attempt_count))
        elif SUCCESS:
            delivery.status = SUCCESS
    
    # 批量处理失败
    process_failed_deliveries(failed_deliveries_attempts, MAX_WEBHOOK_RETRIES)
    clear_successful_deliveries(successful_deliveries)  # 批量清理
    
    # 自调度，持续轮询
    send_webhooks_async_for_app.apply_async(kwargs={"app_id": app_id, ...})
```

#### 3.5.2.1 4xx 场景下的重试与终态判断

**⚠️ 重要修正**：之前的理解存在严重偏差。`process_failed_deliveries()` **完全不检查 HTTP 状态码**！

```python
# [saleor/webhook/transport/utils.py:647](saleor/webhook/transport/utils.py#L647)
def process_failed_deliveries(failed_deliveries_attempts, max_webhook_retries):
    for delivery, attempt, attempt_count in failed_deliveries_attempts:
        # ⚠️ 没有任何状态码检查！4xx 和 5xx 一视同仁
        if attempt_count >= max_webhook_retries:
            delivery.status = EventDeliveryStatus.FAILED
            deliveries_to_update.append(delivery)
        deliveries_attempts_to_update.append(attempt)
```

**关键结论**：
- 批量路径中 **没有针对 4xx 错误的特殊处理**
- 4xx 错误和 5xx 错误一视同仁，都会被重试
- 直到 `attempt_count >= 5` 才会被标记为死信
- 这意味着批量路径下 4xx 错误也会产生 **6 个 attempt 记录**，约 10 秒耗尽重试

**核心机制**：

| 机制 | 实现方式 | 代码位置 |
|-----|---------|---------|
| **重试计数** | 数据库查询 `Count("attempts", distinct=True)`，统计已有的 attempt 记录数 | [utils.py:489](saleor/webhook/transport/utils.py#L489) |
| **重试触发** | 不使用 Celery 重试，通过自调度持续轮询 `PENDING` 状态的 delivery | [transport.py:939](saleor/webhook/transport/asynchronous/transport.py#L939) |
| **重试节奏** | **无指数退避**，任务执行完立即重新调度，重试间隔取决于任务执行时间 | [transport.py:939](saleor/webhook/transport/asynchronous/transport.py#L939) |
| **失败标记** | 不立即标记 FAILED，保持 PENDING，`process_failed_deliveries()` 中当 `attempt_count >= 5` 才批量标记 | [utils.py:654](saleor/webhook/transport/utils.py#L654) |
| **死信判定** | `attempt_count >= MAX_WEBHOOK_RETRIES` 时批量标记为 `FAILED`，**不区分 4xx/5xx** | [utils.py:654](saleor/webhook/transport/utils.py#L654) |
| **成功清理** | `clear_successful_deliveries(deliveries)` 批量删除 | [transport.py:937](saleor/webhook/transport/asynchronous/transport.py#L937) |
| **Attempt 创建** | `create_attempts_for_deliveries()` 批量创建，最后 bulk_update | [utils.py:677](saleor/webhook/transport/utils.py#L677) |
| **禁用 webhook/app 处理** | `get_deliveries_for_app()` **不检查** `is_active`，非活跃的仍会被选中，**正常创建 attempt，正常发送请求，失败后正常重试**，直到 `attempt_count >= 5` 才标记死信 | [utils.py:482](saleor/webhook/transport/utils.py#L482) |
| **4xx 错误处理** | **与 5xx 完全相同**，都会重试 5 次（共 6 次尝试） | [utils.py:654](saleor/webhook/transport/utils.py#L654) |

#### 3.5.3 两条路径的失败处理流程对比

**`process_failed_deliveries` 批量处理逻辑** ([utils.py:647](saleor/webhook/transport/utils.py#L647))：

```python
def process_failed_deliveries(failed_deliveries_attempts, max_webhook_retries):
    for delivery, attempt, attempt_count in failed_deliveries_attempts:
        # 只有当数据库中已有的 attempt 数 >= 最大重试次数时，才标记为死信
        if attempt_count >= max_webhook_retries:
            delivery.status = EventDeliveryStatus.FAILED
            deliveries_to_update.append(delivery)
        deliveries_attempts_to_update.append(attempt)
    
    # 批量更新
    EventDelivery.objects.bulk_update(deliveries_to_update, ["status"])
    EventDeliveryAttempt.objects.bulk_update(deliveries_attempts_to_update, update_fields)
```

#### 3.5.4 两条路径的重试节奏对比

**单条任务路径（区分错误类型）**：

| 错误类型 | 尝试次数 | 重试次数 | 等待时间 | 累计时间 | 最终状态 |
|---------|---------|---------|---------|---------|---------|
| 4xx/3xx | 1 | 0 | 0 | 0 | FAILED（不重试） |
| 5xx/网络错误 | 1 | 0 | 0 | 0 | PENDING |
| 5xx/网络错误 | 2 | 1 | 10 秒 | 10 秒 | PENDING |
| 5xx/网络错误 | 3 | 2 | 20 秒 | 30 秒 | PENDING |
| 5xx/网络错误 | 4 | 3 | 40 秒 | 70 秒 | PENDING |
| 5xx/网络错误 | 5 | 4 | 80 秒 | 150 秒 | PENDING |
| 5xx/网络错误 | 6 | 5 | 160 秒 | 310 秒 | FAILED（重试耗尽） |

**批量路径（不区分错误类型）**：

假设每次任务执行耗时 2 秒，批量大小 100：

| 错误类型 | 尝试次数 | attempt_count | 等待时间 | 累计时间 | 最终状态 |
|---------|---------|---------------|---------|---------|---------|
| 4xx/3xx/5xx | 1 | 0 | ~0 | ~0 | PENDING |
| 4xx/3xx/5xx | 2 | 1 | ~2 秒 | ~2 秒 | PENDING |
| 4xx/3xx/5xx | 3 | 2 | ~2 秒 | ~4 秒 | PENDING |
| 4xx/3xx/5xx | 4 | 3 | ~2 秒 | ~6 秒 | PENDING |
| 4xx/3xx/5xx | 5 | 4 | ~2 秒 | ~8 秒 | PENDING |
| 4xx/3xx/5xx | 6 | 5 | ~2 秒 | ~10 秒 | FAILED（attempt_count >= 5） |

> **重要差异**：
> - 单条路径：4xx/3xx 立即死信（1 次尝试），5xx 约 5 分钟死信（6 次尝试）
> - 批量路径：所有错误类型都约 10 秒死信（6 次尝试），**不区分 4xx/5xx**

#### 3.5.5 禁用 webhook/app 的完整流转对比

这是两条路径最关键的差异之一，也是当前最容易产生理解偏差的地方。

##### 单条任务路径的完整流转

```
send_webhook_request_async(event_delivery_id)
    ↓
get_delivery_for_webhook() → get_multiple_deliveries_for_webhooks()
    ↓
✓ 检查 webhook.is_active
✓ 检查 webhook.app.is_active
   (APP_DELETED / APP_STATUS_CHANGED 事件例外)
    ↓
┌────────────────────────┬───────────────────────────┐
│        活跃             │         非活跃             │
├────────────────────────┼───────────────────────────┤
│ 继续执行                │ 加入 inactive_delivery_ids │
│ create_attempt()        │ EventDelivery.objects     │
│ 发送请求                │   .filter(...).update(     │
│ 失败则 handle_webhook_  │     status=FAILED)        │
│   retry()               │ 不创建 attempt            │
│                        │ 不触发重试                │
└────────────────────────┴───────────────────────────┘
```

**关键代码** ([utils.py:529](saleor/webhook/transport/utils.py#L529))：
```python
should_deliver = delivery.webhook.is_active and (
    delivery.webhook.app.is_active or bypass_inactive_check
)
if not should_deliver:
    inactive_delivery_ids.add(delivery.pk)
# ...
if inactive_delivery_ids:
    EventDelivery.objects.filter(id__in=inactive_delivery_ids).update(
        status=EventDeliveryStatus.FAILED
    )
```

##### 按应用批量路径的完整流转

```
send_webhooks_async_for_app(app_id)
    ↓
get_deliveries_for_app(app_id, batch_size)
    ↓
✗ 不检查 webhook.is_active
✗ 不检查 webhook.app.is_active
✓ 只过滤 status=PENDING 和 webhook__app_id=app_id
    ↓
┌──────────────────────────────────────────────┐
│    活跃/非活跃 一视同仁，全部选中处理           │
├──────────────────────────────────────────────┤
│ create_attempts_for_deliveries()             │
│ send_webhook_using_scheme_method()           │
│   ↓                                          │
│ 如果 webhook 配置无效/目标 URL 不存在 → 失败   │
│   ↓                                          │
│ 进入 failed_deliveries_attempts              │
│ process_failed_deliveries()                  │
│   attempt_count >= 5 → 标记 FAILED           │
│   否则 → 保持 PENDING，下一轮继续             │
└──────────────────────────────────────────────┘
```

> **⚠️ 严重偏差提醒**：批量路径中完全没有 `is_active` 检查。如果 webhook 或 app 被禁用，但其 EventDelivery 仍然是 PENDING 状态，那么：
> 1. 它会被反复选中处理（最多 6 次）
> 2. 每次都会创建新的 EventDeliveryAttempt 记录
> 3. 每次都会实际尝试发送 HTTP 请求（可能失败）
> 4. 直到 attempt_count >= 5 才会被标记为死信
> 5. 整个过程约 10 秒完成（无退避）

##### 对重试计数和死信判定的影响

| 维度 | 单条任务路径 | 按应用批量路径 |
|-----|-------------|-------------|
| **禁用后 attempt 数量** | 0 个（不创建） | 6 个（完整重试周期） |
| **禁用后死信标记时机** | 立即 | 约 10 秒后（attempt_count 累积到 5） |
| **禁用后实际请求次数** | 0 次 | 6 次（每次都尝试发送） |
| **重试计数方式** | 不涉及（无重试） | `Count("attempts")` 数据库统计 |
| **死信判定依据** | `MaxRetriesExceededError`（不会触发） | `attempt_count >= 5` |
| **对应用的影响** | 无感知，不会收到任何请求 | 收到 6 次失败请求，可能触发告警风暴 |

##### 应用侧的最佳实践

1. **禁用 webhook 前先清理队列**：如果需要禁用 webhook 或 app，建议先将所有 PENDING 状态的 EventDelivery 手动标记为 FAILED，避免批量路径继续处理
2. **监控 attempt 数量激增**：如果发现某个 delivery 的 attempt 数量在短时间内快速增长，可能是 webhook 被禁用但批量路径仍在处理
3. **幂等性保护**：即使 webhook 被禁用，应用侧仍需做好幂等性处理，防止批量路径的重复请求
4. **定期清理死信**：定期查询 `status=FAILED` 或 `attempts_gte=5` 的投递记录，确认是否为禁用导致的

---

## 4. 死信处理

### 4.1 两条路径的死信判定差异

#### 单条任务路径的死信流程

```
第1次尝试失败 → delivery.status = FAILED → 触发 Celery 重试
    ↓
第2次尝试失败 → delivery.status = FAILED → 触发 Celery 重试
    ↓
...（共 5 次重试，6 次尝试）
    ↓
第6次尝试失败 → MaxRetriesExceededError → delivery.status = FAILED（最终状态）
```

**死信特征**：`EventDelivery.status == FAILED` 且 `attempts` 数量为 6。

#### 批量路径的死信流程

```
第1次尝试失败 → delivery.status 保持 PENDING → attempt_count = 1
    ↓
任务自调度 → 第2次尝试失败 → delivery.status 保持 PENDING → attempt_count = 2
    ↓
...（共 5 次 attempt，6 次尝试）
    ↓
第6次尝试失败 → attempt_count = 5 → process_failed_deliveries 标记为 FAILED
```

**死信特征**：`EventDelivery.status == FAILED` 且 `attempts` 数量为 6。

> **关键盲区**：在批量路径中，前 5 次尝试失败时 `delivery.status` 始终保持 `PENDING`，应用通过 `status: FAILED` 过滤时会漏掉这些"正在失败但尚未标记死信"的投递。

#### 特殊情况：禁用 webhook/app 的死信流程

**单条任务路径 - 禁用 webhook/app**：
```
任务启动 → get_multiple_deliveries_for_webhooks()
    ↓
检查 webhook.is_active = false
    ↓
立即标记 delivery.status = FAILED
    ↓
不创建 attempt
不触发重试
任务直接返回
```

**死信特征**：`EventDelivery.status == FAILED` 且 `attempts` 数量为 **0**。

**批量路径 - 禁用 webhook/app**：
```
任务启动 → get_deliveries_for_app()
    ↓
不检查 is_active，直接选中 PENDING delivery
    ↓
第1次尝试失败 → attempt_count = 0 → 保持 PENDING
    ↓
第2次尝试失败 → attempt_count = 1 → 保持 PENDING
    ↓
...（共 6 次尝试）
    ↓
第6次尝试失败 → attempt_count = 5 → 标记 FAILED
```

**死信特征**：`EventDelivery.status == FAILED` 且 `attempts` 数量为 **6**。

> **⚠️ 重大差异**：禁用 webhook/app 时，两条路径的死信特征完全不同：
> - 单条路径：0 个 attempt，立即死信
> - 批量路径：6 个 attempt，约 10 秒后死信
> 应用侧不能通过 attempts 数量是否为 0 来判断是否为禁用导致的死信，需要结合 webhook.is_active 字段综合判断。

### 4.2 对扩展应用判断重试节奏和死信状态的影响

#### 4.2.1 重试节奏判断的不确定性

由于两条路径并存且重试机制不同，扩展应用无法准确预测重试节奏：

1. **单条路径**：有约 5 分钟的重试窗口，指数退避给了应用充足的恢复时间
2. **批量路径**：仅约 10 秒就耗尽重试次数，应用可能来不及恢复
3. **路径切换风险**：代码 TODO 注释表明未来可能切换到批量路径，应用的重试处理逻辑可能在升级后失效

#### 4.2.2 死信状态判断的正确姿势

**❌ 错误方式**：仅通过 `EventDelivery.status == FAILED` 判断死信
- 单条路径：每次失败都会临时标记为 FAILED，但可能还在重试周期内
- 批量路径：前 5 次失败都保持 PENDING，直到第 6 次才标记为 FAILED
- 禁用 webhook/app：单条路径会立即标记 FAILED 但 attempts=0，容易与其他情况混淆

**✅ 正确方式**：结合 `status`、`attempts` 数量和 `webhook.isActive` 综合判断

```graphql
query GetDeadLetterDeliveries {
  eventDeliveries(
    filters: {
      or: [
        { status: FAILED },
        { status: PENDING, attempts_Gte: 5 }  # 捕获批量路径中即将成为死信的投递
      ]
    }
  ) {
    id
    status
    createdAt
    eventType
    attempts {
      id
      responseStatusCode
      response
      createdAt
    }
    webhook {
      id
      isActive
      app {
        isActive
      }
    }
  }
}
```

**死信类型判断逻辑**：
| status | attempts 数量 | webhook.isActive | app.isActive | 首次 attempt 状态码 | 死信类型 |
|--------|--------------|-----------------|--------------|----------------|----------|
| FAILED | 0 | false | - | - | webhook 被禁用导致 |
| FAILED | 0 | true | false | - | app 被禁用导致 |
| FAILED | 1 | true | true | 3xx/4xx | 客户端错误（单条路径，不重试） |
| FAILED | 6 | true | true | - | 重试耗尽导致（单条路径 5xx/网络错误） |
| FAILED | 6 | true | true | - | 重试耗尽导致（批量路径，不区分错误类型） |
| PENDING | >= 5 | true | true | - | 即将成为死信（批量路径） |

> **判断技巧**：
> - `attempts == 1 且 `status == FAILED` 且状态码 3xx/4xx → 单条路径的客户端错误
> - `attempts == 6` 且 `status == FAILED` → 重试耗尽（需结合状态码和路径判断
> - `attempts` 列表中首次尝试的状态码可以帮助区分错误类型

#### 4.2.3 应用侧的最佳实践

1. **不要依赖状态判断重试阶段**：通过 `attempts` 列表的长度和创建时间判断实际重试进度
2. **预留足够的幂等性**：批量路径可能在短时间内快速重试多次，即使 webhook 被禁用也可能收到请求
3. **监控 attempt 数量**：当 `attempts` 数量接近 5 时触发告警，提前介入
4. **禁用前先清理**：禁用 webhook 或 app 前，先手动将所有 PENDING 的 delivery 标记为 FAILED
5. **区分路径处理**：
   - 单条路径：利用 5 分钟窗口进行故障恢复
   - 批量路径：快速失败，考虑人工介入或降级处理

### 4.3 死信判定

Saleor **没有专门的死信队列**，而是通过状态标记实现死信处理。死信判定有 **三种不同的触发场景**：

#### 场景一：重试耗尽（普通失败）

**单条任务路径**：
1. 超过 `max_retries`（5次）后，捕获 `MaxRetriesExceededError`
2. 将 `EventDelivery.status` 设置为 `EventDeliveryStatus.FAILED`
3. 记录错误日志，任务结束
4. **死信特征**：`status=FAILED`, `attempts=6`

**批量路径**：
1. 当 `attempt_count >= MAX_WEBHOOK_RETRIES`（5次）时
2. `process_failed_deliveries()` 批量将状态设置为 `FAILED`
3. 任务继续自调度处理其他投递
4. **死信特征**：`status=FAILED`, `attempts=6`

关键代码 ([saleor/webhook/transport/utils.py:457](saleor/webhook/transport/utils.py#L457))：
```python
except MaxRetriesExceededError:
    is_success = False
    task_logger.info(
        "[Webhook ID: %r] Failed request to %r: exceeded retry limit. Delivery ID: %r",
        webhook.id, webhook.target_url, delivery.id,
    )
```

#### 场景二：禁用 webhook/app

**单条任务路径**：
1. `get_multiple_deliveries_for_webhooks()` 中检查 `is_active` 为 false
2. 立即批量更新 `status=FAILED`
3. **不创建 attempt，不触发重试**
4. **死信特征**：`status=FAILED`, `attempts=0`

关键代码 ([saleor/webhook/transport/utils.py:539](saleor/webhook/transport/utils.py#L539))：
```python
if inactive_delivery_ids:
    EventDelivery.objects.filter(id__in=inactive_delivery_ids).update(
        status=EventDeliveryStatus.FAILED
    )
```

**批量路径**：
1. `get_deliveries_for_app()` **不检查** `is_active`，仍然选中处理
2. 经过完整的 6 次尝试周期
3. `attempt_count >= 5` 时通过 `process_failed_deliveries()` 标记死信
4. **死信特征**：`status=FAILED`, `attempts=6`

#### 场景三：4xx 客户端错误

**⚠️ 重要修正**：两条路径的处理方式完全不同。

**单条任务路径**（不重试）：
1. `handle_webhook_retry()` 中检测到 `300 <= code < 500`
2. `return False` 不触发重试
3. 继续执行 `delivery_update(delivery, FAILED)`
4. 任务结束
5. **死信特征**：`status=FAILED`, `attempts=1`, 状态码 3xx/4xx

**批量路径**（与 5xx 完全相同，会重试）：
1. `process_failed_deliveries()` **不检查状态码**，所有失败一视同仁
2. 保持 PENDING，`attempt_count += 1`
3. 下一轮自调度继续选中处理
4. 直到 `attempt_count >= 5` 才标记死信
5. **死信特征**：`status=FAILED`, `attempts=6`, 状态码 3xx/4xx

### 4.4 数据保留策略

**成功投递**：`clear_successful_delivery()` ([saleor/webhook/transport/utils.py:607](saleor/webhook/transport/utils.py#L607)) 会立即删除：
- `EventDelivery` 记录
- 关联的 `EventPayload`（如果没有其他 delivery 引用）
- 存储的 payload 文件

**失败投递**：保留在数据库中，包括：
- `EventDelivery` (status = FAILED)
- `EventDeliveryAttempt`（每次尝试的详细信息）
- `EventPayload`（payload 被转存为文件）

可以通过 GraphQL API 查询失败的投递：`eventDeliveries` 查询，过滤 `status: FAILED`

### 4.5 可观测性

每次投递尝试都会：
1. 创建 `EventDeliveryAttempt` 记录，包含：
   - 响应状态码、响应内容（截断到配置大小）
   - 请求/响应头部
   - 耗时
   - 状态
2. 上报 metrics：`record_external_request()`
3. 触发可观测性 webhook（如果配置了）

### 4.6 两条路径协作关系总结

```
┌─────────────────────────────────────────────────────────────────┐
│                    触发异步 webhook 投递                          │
└─────────────────────────────────────┬───────────────────────────┘
                                      │
                      ┌───────────────▼───────────────┐
                      │  create_deliveries_for_subscriptions │
                      │  生成 EventDelivery (PENDING)     │
                      └───────────────┬───────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          │                           │                           │
┌─────────▼─────────┐     ┌───────────▼───────────┐               │
│  单条任务路径       │     │  按应用批量路径        │               │
│  (当前默认)         │     │  (未来替代方案)        │               │
├───────────────────┤     ├───────────────────────┤               │
│ send_webhook_     │     │ send_webhooks_        │               │
│ request_async     │     │ async_for_app         │               │
├───────────────────┤     ├───────────────────────┤               │
│ ✓ 检查 is_active    │     │ ✗ 不检查 is_active    │               │
│   禁用 → 立即 FAILED  │     │   禁用 → 继续处理        │               │
│ • Celery 重试      │     │ • 自调度轮询          │               │
│ • 指数退避 5分钟   │     │ • 无退避 ~10秒耗尽    │               │
│ • 失败立即标记     │     │ • 保持 PENDING        │               │
│ • attempt_count=6  │     │ • attempt_count=5    │               │
│   标记死信         │     │   标记死信            │               │
└─────────┬─────────┘     └───────────┬───────────┘               │
          │                           │                           │
          └───────────────┬───────────┘                           │
                          │                                       │
                  ┌───────▼───────┐                               │
                  │  成功投递       │                               │
                  │  clear_successful_   │                               │
                  │  delivery / _deliveries │                               │
                  │  删除 EventDelivery   │                               │
                  └───────┬───────┘                               │
                          │                                       │
                  ┌───────▼───────┐                               │
                  │  失败投递       │                               │
                  │  status=FAILED │                               │
                  │  保留所有记录   │                               │
                  └───────────────┘                               │
                                                                  │
                                              (TODO: 待去重机制完善后
                                               切换到批量路径)
```

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

```python
# 代码中定义的常量
# [saleor/webhook/transport/asynchronous/transport.py:74](saleor/webhook/transport/asynchronous/transport.py#L74)
MAX_WEBHOOK_RETRIES = 5

# 单条路径 Celery 配置 [transport.py:761](saleor/webhook/transport/asynchronous/transport.py#L761)
retry_backoff = 10
retry_kwargs = {"max_retries": 5}
# 指数退避公式: countdown = 10 * (2^retries)

# 批量路径批处理大小
WEBHOOK_ASYNC_BATCH_SIZE = 100
```

> **注意**：两条路径的最大重试次数都是 5，但计数方式不同：
> - 单条路径：`self.request.retries` 从 0 到 5，共 6 次尝试
> - 批量路径：`attempt_count` 从 0 到 5，第 6 次尝试时达到阈值标记死信

---

## 9. 典型失败场景处理

| 场景 | 单条任务路径处理 | 按应用批量路径处理 |
|-----|-----------------|-------------------|
| 网络连接错误 | 保持 PENDING，触发 Celery 指数退避重试 | 保持 PENDING，下一轮自调度重试 |
| 5xx 服务器错误 | 保持 PENDING，触发指数退避重试（10/20/40/80/160s） | 保持 PENDING，无退避立即重试（约2秒间隔） |
| **4xx 客户端错误** | ✅ 不重试，立即标记 FAILED，`attempts=1` | ❌ **与 5xx 完全相同**，保持 PENDING，重试 5 次（共 6 次尝试），`attempt_count >= 5` 才标记 FAILED |
| 3xx 重定向 | ✅ 不重试，立即标记 FAILED，`attempts=1` | ❌ **与 5xx 完全相同**，保持 PENDING，重试 5 次（共 6 次尝试），`attempt_count >= 5` 才标记 FAILED |
| 无效 IP 地址（SSRF） | ✅ 不重试，立即标记 FAILED，`attempts=1` | ❌ **与 5xx 完全相同**，保持 PENDING，重试 5 次（共 6 次尝试），`attempt_count >= 5` 才标记 FAILED |
| payload 解析失败 | ✅ 不重试，立即标记 FAILED，`attempts=1` | ❌ **与 5xx 完全相同**，保持 PENDING，重试 5 次（共 6 次尝试），`attempt_count >= 5` 才标记 FAILED |
| **webhook/app 被禁用** | ✅ `get_multiple_deliveries_for_webhooks()` 中检查 `is_active`，立即标记 FAILED，**不创建 attempt，不触发重试**，`attempts=0` | ❌ `get_deliveries_for_app()` **不检查** `is_active`，**仍会被选中**，正常创建 attempt，正常发送请求，失败后正常重试，直到 `attempt_count >= 5` 才标记死信，`attempts=6` |
| 数据库事务未提交 | `call_event` 自动延迟到 `on_commit` | `call_event` 自动延迟到 `on_commit` |
| 主从延迟（延迟 payload） | 最多 12 次重试等待从库同步 | 最多 12 次重试等待从库同步 |

> **⚠️ 关键差异总结**：
> - **单条路径**：智能区分错误类型，3xx/4xx/SSRF/payload 错误立即死信（1 次尝试），5xx/网络错误指数退避重试（6 次尝试，约 5 分钟）
> - **批量路径**：所有错误类型一视同仁，全部重试 5 次（共 6 次尝试），约 10 秒耗尽
> - **禁用 webhook/app**：单条路径立即死信（0 次尝试），批量路径完整重试 6 次
> - **批量路径风险**：4xx 错误（如 404、403）会产生 6 次重复请求，可能触发告警风暴

---

## 10. 关键代码文件索引

| 文件 | 核心职责 |
|-----|---------|
| [saleor/webhook/transport/utils.py](saleor/webhook/transport/utils.py) | HTTP/SQS/PubSub 投递、重试逻辑、attempt 管理、`handle_webhook_retry`、`process_failed_deliveries`、`get_deliveries_for_app` |
| [saleor/webhook/transport/asynchronous/transport.py](saleor/webhook/transport/asynchronous/transport.py) | 异步 webhook 任务、`send_webhook_request_async`（单条路径）、`send_webhooks_async_for_app`（批量路径）、延迟 payload、批量处理 |
| [saleor/webhook/transport/synchronous/transport.py](saleor/webhook/transport/synchronous/transport.py) | 同步 webhook、交易请求、缓存逻辑 |
| [saleor/webhook/circuit_breaker/breaker_board.py](saleor/webhook/circuit_breaker/breaker_board.py) | 断路器实现 |
| [saleor/webhook/transport/__init__.py](saleor/webhook/transport/__init__.py) | 签名生成 `signature_for_payload` |
| [saleor/core/utils/events.py](saleor/core/utils/events.py) | 事务安全事件触发 `call_event` |
| [saleor/core/models.py](saleor/core/models.py) | EventDelivery、EventDeliveryAttempt、EventPayload 模型 |
| [saleor/plugins/webhook/plugin.py](saleor/plugins/webhook/plugin.py) | WebhookPlugin 事件分发入口 |

---

## 11. 关键函数速查表

| 函数 | 所属文件 | 核心功能 |
|-----|---------|---------|
| `send_webhook_request_async` | [transport.py:761](saleor/webhook/transport/asynchronous/transport.py#L761) | 单条任务路径主函数，Celery 内置重试 |
| `send_webhooks_async_for_app` | [transport.py:852](saleor/webhook/transport/asynchronous/transport.py#L852) | 按应用批量路径主函数，自调度轮询 |
| `get_delivery_for_webhook` | [utils.py:469](saleor/webhook/transport/utils.py#L469) | 单条查询 delivery，调用 get_multiple_deliveries_for_webhooks |
| `get_multiple_deliveries_for_webhooks` | [utils.py:503](saleor/webhook/transport/utils.py#L503) | **单条路径的关键过滤函数**，检查 `is_active`，禁用则立即标记 FAILED |
| `get_deliveries_for_app` | [utils.py:482](saleor/webhook/transport/utils.py#L482) | 批量查询待处理投递，**不检查 is_active**，统计 attempt_count |
| `handle_webhook_retry` | [utils.py:406](saleor/webhook/transport/utils.py#L406) | 单条路径的重试触发，指数退避 |
| `process_failed_deliveries` | [utils.py:647](saleor/webhook/transport/utils.py#L647) | 批量路径的失败处理，死信标记 |
| `create_attempt` | [utils.py:554](saleor/webhook/transport/utils.py#L554) | 单条创建 attempt 记录 |
| `create_attempts_for_deliveries` | [utils.py:677](saleor/webhook/transport/utils.py#L677) | 批量创建 attempt 记录 |
| `clear_successful_delivery` | [utils.py:607](saleor/webhook/transport/utils.py#L607) | 单条清理成功投递 |
| `clear_successful_deliveries` | [utils.py:612](saleor/webhook/transport/utils.py#L612) | 批量清理成功投递 |
| `delivery_update` | [utils.py:539](saleor/webhook/transport/utils.py#L539) | 更新 delivery 状态 |
| `attempt_update` | [utils.py:574](saleor/webhook/transport/utils.py#L574) | 更新 attempt 记录 |
