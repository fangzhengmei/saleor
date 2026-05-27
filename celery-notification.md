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

---

## 9. 按业务渠道筛选 Webhook

**文件位置**: `saleor/webhook/utils.py:202`

### 9.1 触发条件

| 触发条件 | 说明 |
|---------|------|
| Webhook 存在 `subscription_query` | 说明是 subscription webhook，可能包含 channel 过滤条件 |
| 事件关联具体 channel | 如 order.created 事件关联 order.channel.slug |
| `filterable_channel_slugs` 数组已填充 | Webhook 保存时解析 subscription_query 提取的 channel 列表 |

### 9.2 处理动作

```python
def filter_webhooks_for_channel(
    webhooks: Iterable[Webhook],
    channel_slug: str,
) -> list[Webhook]:
    for webhook in webhooks:
        # 1. Legacy webhook（无 subscription_query）直接通过
        if not webhook.subscription_query:
            filtered.append(webhook)
            continue
        # 2. 有 subscription_query 但无 channel 过滤条件
        filterable_channel_slugs = list(webhook.filterable_channel_slugs)
        if not filterable_channel_slugs:
            filtered.append(webhook)
            continue
        # 3. 有 channel 过滤条件，匹配当前 channel
        if channel_slug in filterable_channel_slugs:
            filtered.append(webhook)
```

**筛选流程**:
```
webhook
  ├─ 无 subscription_query → 通过
  ├─ 有 subscription_query
  │    ├─ filterable_channel_slugs 为空 → 通过
  │    └─ filterable_channel_slugs 非空
  │         ├─ 当前 channel 在列表中 → 通过
  │         └─ 当前 channel 不在列表中 → 过滤
```

### 9.3 结果

- **通过筛选**: Webhook 保留，后续会为其创建 EventDelivery 并入队
- **被过滤**: Webhook 被排除，不触发任何通知
- **设计意图**: 允许用户通过 subscription query 精确控制哪些 channel 的事件需要通知，避免不必要的 Webhook 调用

---

## 10. 队列决策顺序与回退机制

**关键函数**: `get_queue_name_for_webhook` (`saleor/webhook/transport/asynchronous/transport.py:322`)

### 10.1 触发条件

| 触发条件 | 说明 |
|---------|------|
| 事件触发时传入 `queue` 参数 | WebhookPlugin 中为不同事件类型指定专用队列 |
| Webhook.target_url scheme | 决定是否使用协议专用队列 |
| Settings 中各队列的环境变量配置 | 决定各队列是否有独立配置 |

### 10.2 处理动作

**队列决策优先级（从高到低）**:

```
1. 协议专用队列（目标地址驱动）
   ├─ scheme == "awssqs" → WEBHOOK_SQS_CELERY_QUEUE_NAME
   └─ scheme == "gcpubsub" → WEBHOOK_PUBSUB_CELERY_QUEUE_NAME

2. 事件专用队列（业务类型驱动，作为协议队列的 default_queue）
   ├─ ORDER_* → ORDER_WEBHOOK_EVENTS_CELERY_QUEUE_NAME
   └─ CHECKOUT_* → CHECKOUT_WEBHOOK_EVENTS_CELERY_QUEUE_NAME

3. 默认队列回退（最终兜底）
   └─ 所有未配置的队列 → WEBHOOK_CELERY_QUEUE_NAME
```

**代码中的实际调用**:
```python
# 在 process_deliveries 中
send_webhook_request_async.apply_async(
    kwargs={"event_delivery_id": delivery.pk, ...},
    queue=get_queue_name_for_webhook(
        delivery.webhook,
        # default_queue 优先级：传入的 queue > 默认队列
        default_queue=queue or settings.WEBHOOK_CELERY_QUEUE_NAME,
    ),
)
```

**Settings 中的回退链** (`saleor/settings.py:1043-1055`):
```python
# SQS 队列默认回退到 WEBHOOK_CELERY_QUEUE_NAME
WEBHOOK_SQS_CELERY_QUEUE_NAME = os.environ.get(
    "WEBHOOK_SQS_CELERY_QUEUE_NAME", WEBHOOK_CELERY_QUEUE_NAME
)
# PUBSUB 队列默认回退到 WEBHOOK_CELERY_QUEUE_NAME
WEBHOOK_PUBSUB_CELERY_QUEUE_NAME = os.environ.get(
    "WEBHOOK_PUBSUB_CELERY_QUEUE_NAME", WEBHOOK_CELERY_QUEUE_NAME
)
# CHECKOUT 事件队列默认回退到 WEBHOOK_CELERY_QUEUE_NAME
CHECKOUT_WEBHOOK_EVENTS_CELERY_QUEUE_NAME = os.environ.get(
    "CHECKOUT_WEBHOOK_EVENTS_CELERY_QUEUE_NAME", WEBHOOK_CELERY_QUEUE_NAME
)
# ORDER 事件队列默认回退到 WEBHOOK_CELERY_QUEUE_NAME
ORDER_WEBHOOK_EVENTS_CELERY_QUEUE_NAME = os.environ.get(
    "ORDER_WEBHOOK_EVENTS_CELERY_QUEUE_NAME", WEBHOOK_CELERY_QUEUE_NAME
)
```

**决策示例**:
```
场景1: Order 事件 + HTTP Webhook
  → 传入 queue = ORDER_WEBHOOK_EVENTS_CELERY_QUEUE_NAME
  → scheme = "https"，不在协议队列映射中
  → 最终队列 = ORDER_WEBHOOK_EVENTS_CELERY_QUEUE_NAME

场景2: Order 事件 + SQS Webhook
  → 传入 queue = ORDER_WEBHOOK_EVENTS_CELERY_QUEUE_NAME
  → scheme = "awssqs"，匹配协议队列
  → 最终队列 = WEBHOOK_SQS_CELERY_QUEUE_NAME

场景3: Product 事件（无专用队列）+ HTTP Webhook
  → 传入 queue = None
  → scheme = "https"，不在协议队列映射中
  → 最终队列 = WEBHOOK_CELERY_QUEUE_NAME
```

### 10.3 结果

- **协议专用队列**: SQS/PubSub Webhook 使用独立队列，避免与 HTTP Webhook 竞争资源
- **事件专用队列**: Order/Checkout 高频事件使用独立队列，隔离核心业务流量
- **默认队列兜底**: 所有未明确配置的场景统一使用默认队列，确保系统可用性
- **可扩展性**: 每个队列都可通过环境变量独立配置，支持按需拆分

---

## 11. Delivery 暂不可见时的快速重试

**文件位置**: `saleor/webhook/transport/asynchronous/transport.py:772-776`

### 11.1 触发条件

| 触发条件 | 说明 |
|---------|------|
| `send_webhook_request_async` 任务执行 | Worker 从队列中取出任务开始执行 |
| `get_delivery_for_webhook` 返回 `(None, True)` | delivery 在主库已创建但从库还未同步（主从延迟） |

```python
def get_delivery_for_webhook(event_delivery_id):
    delivery, inactive_delivery_ids = get_multiple_deliveries_for_webhooks(...)
    not_found = False
    # delivery 不存在且不在 inactive 列表中 → 主从延迟
    if not delivery and event_delivery_id not in inactive_delivery_ids:
        not_found = True
    return delivery, not_found
```

### 11.2 处理动作

```python
def send_webhook_request_async(self, event_delivery_id, ...):
    delivery, not_found = get_delivery_for_webhook(event_delivery_id)
    if not delivery:
        if not_found:
            # 快速重试：countdown=1，1秒后重试
            raise self.retry(countdown=1)
        return
```

**重试特点**:
- **快速重试**: `countdown=1`，仅等待 1 秒
- **无退避**: 不使用指数退避，因为主从延迟通常是短暂的
- **复用重试预算**: 使用 `send_webhook_request_async` 自身的 `max_retries=5` 重试次数
- **重试上下文**: 保留原始 task_id，便于追踪

### 11.3 结果

- **成功**: 1秒后 delivery 在从库可见，任务正常执行
- **失败**: 如果多次重试后仍不可见，最终会因 `MaxRetriesExceededError` 标记为失败
- **设计意图**: 针对主从复制延迟的常见场景，用最小代价（1秒等待）解决大部分临时性不可见问题

---

## 12. Deferred Payload 部分缺失时的补排队逻辑

**文件位置**: `saleor/webhook/transport/asynchronous/transport.py:567-600`

### 12.1 触发条件

| 触发条件 | 说明 |
|---------|------|
| `generate_deferred_payloads` 任务执行 | 处理延迟 payload 生成 |
| `confirm_event_delivery_availability` 返回部分缺失 | 某些 delivery 在从库已同步，某些还未同步 |

```python
def confirm_event_delivery_availability(
    event_delivery_ids: list[int],
    db_connection_name: str,
) -> tuple[set[int], set[int]]:
    available_delivery_pks = set(
        EventDelivery.objects.using(db_connection_name)
        .filter(pk__in=event_delivery_ids)
        .values_list("pk", flat=True)
    )
    missing_delivery_pks = set(event_delivery_ids) - available_delivery_pks
    return available_delivery_pks, missing_delivery_pks
```

### 12.2 处理动作

```python
def generate_deferred_payloads(self, event_delivery_ids, ...):
    # 1. 检查哪些 delivery 在从库可用
    available_delivery_pks, missing_delivery_pks = (
        confirm_event_delivery_availability(event_delivery_ids, db_connection_name)
    )

    # 2. 全部缺失：指数退避重试等待主从同步
    if not available_delivery_pks:
        retry_backoff = 1
        countdown = retry_backoff * (2**self.request.retries)
        raise self.retry(countdown=countdown, ...)

    # 3. 部分缺失：把缺失的单独重新入队（补排队）
    if missing_delivery_pks:
        request_kwargs = self.request.kwargs
        generate_deferred_payloads.apply_async(
            kwargs={
                **request_kwargs,
                "event_delivery_ids": list(missing_delivery_pks),
            },
            MessageGroupId=message_group_id,
        )

    # 4. 处理已可用的 delivery
    _generate_deferred_payloads(
        event_delivery_ids=available_delivery_pks,
        ...
    )
```

**全部缺失的退避策略**:
- `max_retries=12`: 最多重试 12 次
- 退避公式: `countdown = 1 * (2^retries)`
- 重试序列: `1s, 2s, 4s, 8s, 16s, 32s, 64s, 128s, 256s, 512s, 1024s, 2048s`
- 总等待时间可达约 68 分钟，应对极端主从延迟场景

**部分缺失的分流策略**:
```
输入: [1, 2, 3, 4, 5]
  ├─ available: [1, 2, 3] → 立即处理，生成 payload 并入队发送
  └─ missing: [4, 5] → 重新入队 generate_deferred_payloads
        ↓ (下次执行)
        ├─ available: [4] → 处理
        └─ missing: [5] → 再次入队
              ↓
              (最终处理或失败)
```

### 12.3 结果

- **已可用 delivery**: 立即生成 payload 并入队 `send_webhook_request_async`，不被缺失的 delivery 阻塞
- **缺失 delivery**: 单独重新入队，享受独立的重试预算，不会影响已可用的 delivery
- **全部缺失**: 指数退避重试，给予足够时间等待主从同步
- **设计意图**: 分批处理避免批量任务因个别 delivery 未同步而全部阻塞，提高整体吞吐量

---

## 13. 核心决策逻辑汇总（触发条件 → 处理动作 → 结果）

### 13.1 按渠道筛选 Webhook

| 触发条件 | 处理动作 | 结果 |
|---------|---------|------|
| Webhook 无 subscription_query | 直接加入结果列表 | Webhook 保留 |
| Webhook 有 subscription_query 但 filterable_channel_slugs 为空 | 直接加入结果列表 | Webhook 保留 |
| Webhook 有 subscription_query 且当前 channel 在 filterable_channel_slugs 中 | 加入结果列表 | Webhook 保留 |
| Webhook 有 subscription_query 且当前 channel 不在 filterable_channel_slugs 中 | 跳过 | Webhook 被过滤 |

### 13.2 队列决策

| 触发条件 | 处理动作 | 结果 |
|---------|---------|------|
| Webhook scheme 是 awssqs | 使用 WEBHOOK_SQS_CELERY_QUEUE_NAME | SQS 专用队列 |
| Webhook scheme 是 gcpubsub | 使用 WEBHOOK_PUBSUB_CELERY_QUEUE_NAME | Pub/Sub 专用队列 |
| Webhook scheme 是 http/https 且传入 queue=ORDER_* | 使用 ORDER_WEBHOOK_EVENTS_CELERY_QUEUE_NAME | Order 事件队列 |
| Webhook scheme 是 http/https 且传入 queue=CHECKOUT_* | 使用 CHECKOUT_WEBHOOK_EVENTS_CELERY_QUEUE_NAME | Checkout 事件队列 |
| Webhook scheme 是 http/https 且传入 queue=None | 使用 WEBHOOK_CELERY_QUEUE_NAME | 默认队列 |
| 以上队列未配置环境变量 | 回退到 WEBHOOK_CELERY_QUEUE_NAME | 默认队列兜底 |

### 13.3 Delivery 不可见重试

| 触发条件 | 处理动作 | 结果 |
|---------|---------|------|
| get_delivery_for_webhook 返回 (None, True) | raise self.retry(countdown=1) | 1秒后快速重试 |
| 重试后 delivery 可见 | 正常执行发送逻辑 | Webhook 发送成功或失败 |
| 重试超过 max_retries=5 次 | MaxRetriesExceededError | EventDelivery 标记为 FAILED |

### 13.4 Deferred Payload 补排队

| 触发条件 | 处理动作 | 结果 |
|---------|---------|------|
| 全部 delivery 缺失 | 指数退避重试 (1s, 2s, 4s, ...) | 等待主从同步 |
| 部分 delivery 缺失 | 缺失的重新入队，可用的立即处理 | 分批分流，不阻塞 |
| 重试超过 max_retries=12 次 | MaxRetriesExceededError | 记录错误日志，放弃 |
| delivery 最终可用 | 生成 payload 并入队发送 | Webhook 正常发送 |
