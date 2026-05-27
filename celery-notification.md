# Saleor Celery Task 与通知事件生成链路分析

## 1. 整体架构概览

Saleor 使用 Celery 作为异步任务队列系统，结合 Webhook 插件实现事件通知机制。整个链路包含以下核心环节：

```
业务操作 → 事件触发 → WebhookPlugin 处理 → 任务入队 → Worker 执行 → 重试保障 → 结果清理
```

**核心模块位置：**
- Celery 配置: `saleor/celeryconf.py`
- 核心任务基类: `saleor/core/tasks.py`
- Webhook 异步传输: `saleor/webhook/transport/asynchronous/transport.py`
- Webhook 插件: `saleor/plugins/webhook/plugin.py`
- 传输工具: `saleor/webhook/transport/utils.py`

---

## 2. 事件触发机制

### 2.1 事件触发入口: `call_event`

**文件位置**: `saleor/core/utils/events.py:19`

事件触发的标准方式是通过 `call_event` 函数，它确保在数据库事务内时，事件会在事务提交后触发。

```python
def call_event(func_obj, *func_args, **func_kwargs):
    """Call webhook event with given args.

    Ensures that in atomic transaction event is called on_commit.
    """
    connection = transaction.get_connection()
    if connection.in_atomic_block:
        transaction.on_commit(lambda: func_obj(*func_args, **func_kwargs))
    else:
        func_obj(*func_args, **func_kwargs)
```

**设计意图**: 确保数据库事务提交后再发送 Webhook，避免发送了 Webhook 但事务回滚导致数据不一致。

### 2.2 插件管理器: `PluginsManager`

**文件位置**: `saleor/plugins/manager.py`

`PluginsManager` 负责管理所有插件（包括 WebhookPlugin），并在业务逻辑中调用相应的事件方法。

核心方法 `__run_method_on_plugins` 遍历所有激活的插件并执行相应方法：

```python
def __run_method_on_plugins(
    self,
    method_name: str,
    default_value: Any,
    *args,
    channel_slug: str | None,
    plugin_ids: list[str] | None = None,
    **kwargs,
):
    """Try to run a method with the given name on each declared active plugin."""
    value = default_value
    plugins = self.get_plugins(
        channel_slug=channel_slug,
        active_only=True,
        plugin_ids=plugin_ids,
    )
    for plugin in plugins:
        value = self.__run_method_on_single_plugin(
            plugin, method_name, value, *args, **kwargs
        )
    return value
```

### 2.3 WebhookPlugin: 事件处理核心

**文件位置**: `saleor/plugins/webhook/plugin.py`

`WebhookPlugin` 是所有 Webhook 事件的处理入口，为每种事件类型提供专门的处理方法：

```python
class WebhookPlugin(BasePlugin):
    PLUGIN_ID = "mirumee.webhooks"
    PLUGIN_NAME = "Webhooks"
    DEFAULT_ACTIVE = True

    def order_created(self, order: "Order", previous_value: None, webhooks=None) -> None:
        if not self.active:
            return previous_value
        event_type = WebhookEventAsyncType.ORDER_CREATED
        if webhooks := self._get_webhooks_for_channel_events(
            event_type, order.channel.slug, webhooks
        ):
            order_data_generator = partial(
                generate_order_payload, order, self.requestor
            )
            self.trigger_webhooks_async(
                None,
                event_type,
                webhooks,
                order,
                self.requestor,
                legacy_data_generator=order_data_generator,
                queue=settings.ORDER_WEBHOOK_EVENTS_CELERY_QUEUE_NAME,
            )
        return previous_value
```

---

## 3. 任务入队与按渠道分发

### 3.1 触发异步 Webhook: `trigger_webhooks_async`

**文件位置**: `saleor/webhook/transport/asynchronous/transport.py:477`

```python
def trigger_webhooks_async(
    data,  # deprecated
    event_type,
    webhooks,
    subscribable_object=None,
    requestor=None,
    legacy_data_generator=None,
    allow_replica=False,
    pre_save_payloads=None,
    request_time=None,
    queue=None,
):
    """Trigger async webhooks - both regular and subscription."""
    trigger_webhooks_async_for_multiple_objects(
        event_type=event_type,
        webhooks=webhooks,
        webhook_payloads_data=[
            WebhookPayloadData(
                subscribable_object=subscribable_object,
                legacy_data_generator=legacy_data_generator,
                data=data,
            )
        ],
        requestor=requestor,
        allow_replica=allow_replica,
        pre_save_payloads=pre_save_payloads,
        request_time=request_time,
        queue=queue,
    )
```

### 3.2 多对象批量处理: `trigger_webhooks_async_for_multiple_objects`

**文件位置**: `saleor/webhook/transport/asynchronous/transport.py:332`

该函数处理核心逻辑：

1. **Webhook 分组**: 区分 legacy webhooks 和 subscription webhooks
2. **延迟 Payload 判断**: 检查是否需要延迟生成 payload
3. **创建 EventDelivery**: 批量创建交付记录
4. **任务入队**: 根据 Webhook 类型分发到不同队列

```python
def trigger_webhooks_async_for_multiple_objects(
    event_type,
    webhooks,
    webhook_payloads_data: list[WebhookPayloadData],
    requestor=None,
    allow_replica=False,
    pre_save_payloads=None,
    request_time=None,
    queue=None,
):
    # 1. 分组 Webhooks
    legacy_webhooks, subscription_webhooks = group_webhooks_by_subscription(webhooks)

    # 2. 判断是否延迟生成 payload
    is_deferred_payload = WebhookEventAsyncType.EVENT_MAP.get(event_type, {}).get(
        "is_deferred_payload", False
    )

    # 3. 处理 Legacy Webhooks
    for webhook_payload_detail in webhook_payloads_data:
        if legacy_webhooks:
            # 创建 EventPayload 和 EventDelivery
            with transaction.atomic():
                payload = EventPayload.objects.create_with_payload_file(data)
                deliveries.extend(
                    create_event_delivery_list_for_webhooks(...)
                )

    # 4. 处理 Subscription Webhooks
    if subscription_webhooks:
        if is_deferred_payload:
            # 延迟生成 payload：先创建 delivery，后续在 task 中生成 payload
            deferred_deliveries_per_object = (
                create_deliveries_for_deferred_payload_subscriptions(...)
            )
            # 入队延迟 payload 任务
            generate_deferred_payloads.apply_async(...)
        else:
            # 立即生成 payload 并创建 delivery
            delivery_promise = create_deliveries_for_multiple_subscription_objects(...)

    # 5. 发送任务到对应队列
    def process_deliveries(deliveries_list):
        for delivery in deliveries_list:
            send_webhook_request_async.apply_async(
                kwargs={"event_delivery_id": delivery.pk, ...},
                queue=get_queue_name_for_webhook(delivery.webhook, default_queue=queue),
                MessageGroupId=message_group_id,
            )
```

### 3.3 队列分发策略: `get_queue_name_for_webhook`

**文件位置**: `saleor/webhook/transport/asynchronous/transport.py:322`

根据 Webhook 的 target_url scheme 分发到不同队列：

```python
def get_queue_name_for_webhook(webhook, default_queue):
    return {
        WebhookSchemes.AWS_SQS: settings.WEBHOOK_SQS_CELERY_QUEUE_NAME,
        WebhookSchemes.GOOGLE_CLOUD_PUBSUB: settings.WEBHOOK_PUBSUB_CELERY_QUEUE_NAME,
    }.get(
        urlparse(webhook.target_url).scheme.lower(),
        default_queue,
    )
```

**队列配置** (`saleor/settings.py:1037-1054`):
- `WEBHOOK_CELERY_QUEUE_NAME`: 默认 Webhook 队列
- `WEBHOOK_DEFERRED_PAYLOAD_QUEUE_NAME`: 延迟 payload 生成队列
- `WEBHOOK_SQS_CELERY_QUEUE_NAME`: AWS SQS 专用队列
- `WEBHOOK_PUBSUB_CELERY_QUEUE_NAME`: Google Cloud Pub/Sub 专用队列
- `CHECKOUT_WEBHOOK_EVENTS_CELERY_QUEUE_NAME`: Checkout 事件专用队列
- `ORDER_WEBHOOK_EVENTS_CELERY_QUEUE_NAME`: Order 事件专用队列

---

## 4. Worker 执行流程

### 4.1 Celery Task 基类: `RestrictWriterDBTask`

**文件位置**: `saleor/core/tasks.py:19`

所有 Celery Task 的基类，用于限制对写库的非预期访问：

```python
class RestrictWriterDBTask(Task):
    """Celery task that checks usage of unrestricted queries to the writer database."""

    def __call__(self, *args, **kwargs):
        func_path = settings.CELERY_RESTRICT_WRITER_METHOD
        if not func_path:
            return None
        wrapper_fun = import_string(func_path)
        with connections[settings.DATABASE_CONNECTION_DEFAULT_NAME].execute_wrapper(
            wrapper_fun
        ):
            return super().__call__(*args, **kwargs)
```

### 4.2 延迟 Payload 生成任务: `generate_deferred_payloads`

**文件位置**: `saleor/webhook/transport/asynchronous/transport.py:535`

对于标记为 `is_deferred_payload` 的事件类型，payload 在 Celery Task 中生成：

```python
@app.task(
    bind=True,
    max_retries=12,
)
@allow_writer()
@task_with_telemetry_context
def generate_deferred_payloads(
    self,
    event_delivery_ids: list,
    deferred_payload_data: dict,
    send_webhook_queue: str | None = None,
    *,
    telemetry_context: TelemetryTaskContext,
):
    # 1. 检查 EventDelivery 是否在 replica 中可用（处理主从延迟）
    available_delivery_pks, missing_delivery_pks = (
        confirm_event_delivery_availability(event_delivery_ids, db_connection_name)
    )

    # 2. 如果全部缺失，重试等待主从同步
    if not available_delivery_pks:
        retry_backoff = 1
        countdown = retry_backoff * (2**self.request.retries)
        raise self.retry(countdown=countdown, ...)

    # 3. 重新构建 subscribable_object 并生成 payload
    _generate_deferred_payloads(...)

    # 4. payload 生成完成后，入队发送任务
    send_webhook_request_async.apply_async(...)
```

### 4.3 Webhook 发送任务: `send_webhook_request_async`

**文件位置**: `saleor/webhook/transport/asynchronous/transport.py:761`

实际执行 Webhook HTTP 请求的任务：

```python
@app.task(
    queue=settings.WEBHOOK_CELERY_QUEUE_NAME,
    bind=True,
    retry_backoff=10,
    retry_kwargs={"max_retries": 5},
)
@allow_writer()
@task_with_telemetry_context
def send_webhook_request_async(
    self, event_delivery_id, *, telemetry_context: TelemetryTaskContext
) -> None:
    # 1. 获取 delivery 和 webhook
    delivery, not_found = get_delivery_for_webhook(event_delivery_id)

    # 2. 创建发送尝试记录
    attempt = create_attempt(delivery, self.request.id)

    # 3. 发送 Webhook 请求
    response = send_webhook_using_scheme_method(
        webhook.target_url,
        domain,
        webhook.secret_key,
        delivery.event_type,
        data,
        webhook.custom_headers,
    )

    # 4. 处理结果
    if response.status == EventDeliveryStatus.FAILED:
        attempt_update(attempt, response)
        if retry_on_failure:
            handle_webhook_retry(self, webhook, response, delivery, attempt)
        delivery_update(delivery, EventDeliveryStatus.FAILED)
    elif response.status == EventDeliveryStatus.SUCCESS:
        delivery.status = EventDeliveryStatus.SUCCESS
        clear_successful_delivery(delivery)
```

### 4.4 多传输协议支持: `send_webhook_using_scheme_method`

**文件位置**: `saleor/webhook/transport/utils.py:375`

根据 URL scheme 选择不同的传输方式：

```python
def send_webhook_using_scheme_method(
    target_url,
    domain,
    secret,
    event_type,
    data,
    custom_headers=None,
) -> WebhookResponse:
    scheme_matrix: dict[WebhookSchemes, Callable] = {
        WebhookSchemes.HTTP: send_webhook_using_http,
        WebhookSchemes.HTTPS: send_webhook_using_http,
        WebhookSchemes.AWS_SQS: send_webhook_using_aws_sqs,
        WebhookSchemes.GOOGLE_CLOUD_PUBSUB: send_webhook_using_google_cloud_pubsub,
    }
    # ... 调用对应方法
```

---

## 5. 失败重试保障机制

### 5.1 重试处理: `handle_webhook_retry`

**文件位置**: `saleor/webhook/transport/utils.py:406`

```python
def handle_webhook_retry(
    celery_task: Task,
    webhook: Webhook,
    response: WebhookResponse,
    delivery: EventDelivery,
    delivery_attempt: EventDeliveryAttempt,
) -> bool:
    # 1. 3xx 和 4xx 状态码不重试
    if response.response_status_code and 300 <= response.response_status_code < 500:
        return False

    # 2. 指数退避重试: countdown = retry_backoff * (2^retries)
    try:
        countdown = celery_task.retry_backoff * (2**celery_task.request.retries)
        celery_task.retry(countdown=countdown, **celery_task.retry_kwargs)
    except Retry as retry_error:
        next_retry = observability.task_next_retry_date(retry_error)
        observability.report_event_delivery_attempt(delivery_attempt, next_retry)
        raise retry_error
    except MaxRetriesExceededError:
        # 超过最大重试次数，标记为失败
        return False
```

**重试参数配置**:
- `retry_backoff=10`: 基础退避时间 10 秒
- `max_retries=5`: 最大重试 5 次
- 退避策略: 指数退避 `10s, 20s, 40s, 80s, 160s`

### 5.2 熔断器机制: `BreakerBoard`

**文件位置**: `saleor/webhook/circuit_breaker/breaker_board.py`

对于同步 Webhook 事件，Saleor 实现了熔断器模式防止级联故障：

**状态流转**:
```
CLOSED → OPEN → HALF_OPEN → CLOSED
          ↓           ↑
          └───────────┘
```

**核心参数**:
- `BREAKER_BOARD_TTL_SECONDS = 300`: 统计时间窗口 5 分钟
- `BREAKER_BOARD_FAILURE_MIN_COUNT = 100`: 最小失败次数阈值
- `BREAKER_BOARD_FAILURE_THRESHOLD_PERCENTAGE = 35`: 失败率阈值 35%
- `BREAKER_BOARD_COOLDOWN_SECONDS = 120`: 冷却时间 2 分钟
- `BREAKER_BOARD_SUCCESS_COUNT_RECOVERY = 50`: 恢复所需成功次数

```python
def update_breaker_state(self, app: "App") -> str:
    state, changed_at = self.storage.get_app_state(app.id)
    total = self.storage.get_event_count(app.id, "total") or 1
    errors = self.storage.get_event_count(app.id, "error")

    # CLOSED → OPEN: 超过错误阈值
    if state == CircuitBreakerState.CLOSED and self.exceeded_error_threshold(...):
        return self.set_breaker_state(app, CircuitBreakerState.OPEN, ...)

    # OPEN → HALF_OPEN: 冷却时间过后
    if state == CircuitBreakerState.OPEN and changed_at < (time.time() - cooldown):
        return self.set_breaker_state(app, CircuitBreakerState.HALF_OPEN, ...)

    # HALF_OPEN → OPEN / CLOSED: 恢复阶段判断
    if state == CircuitBreakerState.HALF_OPEN:
        if self.exceeded_error_threshold(...):
            return self.set_breaker_state(app, CircuitBreakerState.OPEN, ...)
        if self.reached_half_open_target_success_count(...):
            return self.set_breaker_state(app, CircuitBreakerState.CLOSED, ...)
    return state
```

### 5.3 批量重试任务: `send_webhooks_async_for_app`

**文件位置**: `saleor/webhook/transport/asynchronous/transport.py:846`

按 App 批量处理待发送的 Webhook：

```python
@app.task(
    queue=settings.WEBHOOK_CELERY_QUEUE_NAME,
    bind=True,
)
def send_webhooks_async_for_app(self, app_id, telemetry_context, ...) -> None:
    # 1. 获取该 App 待发送的 deliveries（批量 100 条）
    deliveries = get_deliveries_for_app(app_id, WEBHOOK_ASYNC_BATCH_SIZE)

    # 2. 批量创建尝试记录
    attempts_for_deliveries = create_attempts_for_deliveries(deliveries, ...)

    # 3. 逐个发送
    for delivery_id, delivery_with_count in deliveries.items():
        # 发送请求...

    # 4. 处理失败的 deliveries
    process_failed_deliveries(failed_deliveries_attempts, MAX_WEBHOOK_RETRIES)

    # 5. 自调用继续处理下一批
    send_webhooks_async_for_app.apply_async(kwargs={"app_id": app_id, ...})
```

---

## 6. 核心数据模型

### 6.1 EventPayload: 事件 Payload 存储

**文件位置**: `saleor/core/models.py:171`

```python
class EventPayload(models.Model):
    payload = models.TextField(default="")  # 小 payload 直接存储
    payload_file = models.FileField(...)      # 大 payload 存文件
    created_at = models.DateTimeField(auto_now_add=True)

    def get_payload(self):
        if self.payload_file:
            with self.payload_file.open("rb") as f:
                return f.read().decode("utf-8")
        return self.payload
```

**批量创建优化**:
```python
@transaction.atomic
def bulk_create_with_payload_files(self, objs, payloads):
    created_objs = self.bulk_create(objs)
    for obj, payload_data in zip(created_objs, payloads):
        obj.save_payload_file(payload_data, save_instance=False)
    self.bulk_update(created_objs, ["payload_file"])
    return created_objs
```

### 6.2 EventDelivery: 事件交付记录

**文件位置**: `saleor/core/models.py:206`

```python
class EventDelivery(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    status = models.CharField(  # PENDING / SUCCESS / FAILED
        max_length=255,
        choices=EventDeliveryStatus.CHOICES,
        default=EventDeliveryStatus.PENDING,
    )
    event_type = models.CharField(max_length=255)
    payload = models.ForeignKey(EventPayload, ...)
    webhook = models.ForeignKey("webhook.Webhook", ...)
```

### 6.3 EventDeliveryAttempt: 发送尝试记录

**文件位置**: `saleor/core/models.py:223`

```python
class EventDeliveryAttempt(models.Model):
    delivery = models.ForeignKey(EventDelivery, ...)
    created_at = models.DateTimeField(auto_now_add=True)
    task_id = models.CharField(max_length=255, null=True)
    duration = models.FloatField(null=True)
    response = models.TextField(null=True)
    response_headers = models.TextField(null=True)
    response_status_code = models.PositiveSmallIntegerField(null=True)
    request_headers = models.TextField(null=True)
    status = models.CharField(
        max_length=255,
        choices=EventDeliveryStatus.CHOICES,
        default=EventDeliveryStatus.PENDING,
    )
```

---

## 7. 成功后清理机制

### 7.1 清理成功交付: `clear_successful_deliveries`

**文件位置**: `saleor/webhook/transport/utils.py:612`

```python
def clear_successful_deliveries(deliveries: list["EventDelivery"]):
    # 1. 收集需要删除的 delivery 和 payload
    for delivery in deliveries:
        if delivery.status != EventDeliveryStatus.SUCCESS:
            continue
        delivery_ids_to_delete.append(delivery.id)
        if payload_id := delivery.payload_id:
            payload_ids_to_delete.append(payload_id)

    # 2. 删除 delivery
    if delivery_ids_to_delete:
        EventDelivery.objects.filter(pk__in=delivery_ids_to_delete).delete()

    # 3. 删除无引用的 payload 和文件
    if payload_ids_to_delete:
        payloads_to_delete = EventPayload.objects.filter(
            pk__in=payload_ids_to_delete, deliveries__isnull=True
        )
        files_to_delete = [ep.payload_file.name for ep in payloads_to_delete if ep.payload_file]
        payloads_to_delete.delete()
        delete_files_from_private_storage_task(files_to_delete)
```

### 7.2 定期清理过期 Payload: `delete_event_payloads_task`

**文件位置**: `saleor/core/tasks.py:66`

```python
@app.task
def delete_event_payloads_task(expiration_date=None):
    # 删除超过 EVENT_PAYLOAD_DELETE_PERIOD 且无 delivery 引用的 payload
    delete_period = timezone.now() - settings.EVENT_PAYLOAD_DELETE_PERIOD
    valid_deliveries = EventDelivery.objects.filter(created_at__gt=delete_period)
    payloads_to_delete = (
        EventPayload.objects
        .filter(~Exists(valid_deliveries.filter(payload_id=OuterRef("id"))))
        .order_by("-pk")
    )
    # 批量删除，自调用继续
    if ids:
        EventPayload.objects.filter(pk__in=ids).delete()
        delete_event_payloads_task.delay(expiration_date)
```

---

## 8. 关键调用链总结

### 8.1 订单创建事件完整链路

```
1. 业务代码: order_created mutation
   ↓
2. call_event(manager.order_created, order)
   ↓ (事务提交后)
3. PluginsManager.order_created(order)
   ↓
4. WebhookPlugin.order_created(order)
   ├─ get_webhooks_for_event(ORDER_CREATED)
   ├─ filter_webhooks_for_channel(webhooks, channel_slug)
   └─ trigger_webhooks_async(...)
      ↓
5. trigger_webhooks_async_for_multiple_objects(...)
   ├─ 区分 legacy/subscription webhooks
   ├─ 创建 EventPayload + EventDelivery
   └─ send_webhook_request_async.apply_async(queue=order_events_queue)
      ↓ (Celery Worker)
6. send_webhook_request_async(event_delivery_id)
   ├─ get_delivery_for_webhook()
   ├─ create_attempt()
   ├─ send_webhook_using_http()
   ├─ 成功: clear_successful_delivery()
   └─ 失败: handle_webhook_retry() → 指数退避重试
```

### 8.2 延迟 Payload 事件链路

```
1. 事件触发 (如 PRODUCT_VARIANT_DISCOUNTED_PRICE_UPDATED)
   ↓
2. trigger_webhooks_async()
   ↓ (is_deferred_payload=True)
3. create_deliveries_for_deferred_payload_subscriptions()
   └─ 只创建 EventDelivery，不创建 EventPayload
      ↓
4. generate_deferred_payloads.apply_async(queue=deferred_payload_queue)
   ↓ (Celery Worker)
5. generate_deferred_payloads()
   ├─ 等待主从同步（必要时重试）
   ├─ 重建 subscribable_object
   ├─ 执行 subscription query 生成 payload
   ├─ 创建 EventPayload 并关联
   └─ send_webhook_request_async.apply_async()
      ↓
6. 后续流程同上
```
