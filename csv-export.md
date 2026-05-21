# Saleor CSV 导出流程异步任务串接逻辑分析

## 一、整体架构概览

Saleor 的数据导出流程采用 **"请求登记 → 异步执行 → 产物投递"** 三段式接力架构，涉及多个模块协同工作：

```
用户请求 → GraphQL Mutation → 创建 ExportFile 记录 → 触发 Celery 任务
                                                              ↓
                                                         后台执行导出
                                                              ↓
                                                         保存文件到存储
                                                              ↓
                                                         触发通知事件
                                                              ↓
                                         Plugin Manager → Admin Email Plugin → 发送邮件
                                                              ↓
                                                         Webhook 通知外部系统
```

## 二、第一阶段：导出请求登记

### 2.1 数据模型基础

核心模型定义在 `saleor/csv/models.py`：

**ExportFile 模型**（继承自 Job 抽象模型）：
```python
class ExportFile(Job):
    user = models.ForeignKey(User, ...)      # 发起导出的用户
    app = models.ForeignKey(App, ...)        # 发起导出的应用（可选）
    content_file = models.FileField(...)     # 导出的文件
```

**Job 基类**（`saleor/core/models.py:141`）：
```python
class Job(models.Model):
    status = models.CharField(choices=JobStatus.CHOICES, default=JobStatus.PENDING)
    message = models.CharField(max_length=255, blank=True, null=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

**Job 状态流转**（`saleor/core/__init__.py:7`）：
- `PENDING` - 待处理（初始状态）
- `SUCCESS` - 成功
- `FAILED` - 失败
- `DELETED` - 已删除

**ExportEvent 模型** - 用于审计追踪导出生命周期的每个事件：
```python
class ExportEvent(models.Model):
    date = models.DateTimeField(default=timezone.now)
    type = models.CharField(choices=ExportEvents.CHOICES)  # 事件类型
    parameters = JSONField(blank=True, default=dict)       # 事件参数
    export_file = models.ForeignKey(ExportFile, ...)
    user = models.ForeignKey(User, ...)
    app = models.ForeignKey(App, ...)
```

### 2.2 GraphQL Mutation 入口

三种导出类型对应的 Mutation：
1. **商品导出** - `ExportProducts` (`saleor/graphql/csv/mutations/export_products.py`)
2. **礼品卡导出** - `ExportGiftCards` (`saleor/graphql/csv/mutations/export_gift_cards.py`)
3. **优惠券代码导出** - `ExportVoucherCodes` (`saleor/graphql/csv/mutations/export_voucher_codes.py`)

所有导出 Mutation 都继承自 `BaseExportMutation`（`saleor/graphql/csv/mutations/base_export.py`）。

### 2.3 请求登记流程（以商品导出为例）

代码位置：`saleor/graphql/csv/mutations/export_products.py:108-132`

```python
@classmethod
def perform_mutation(cls, _root, info: ResolveInfo, /, input):
    # 1. 解析导出范围（IDS / FILTER / ALL）
    scope = cls.get_scope(input, Product)
    
    # 2. 解析导出配置（字段、属性、仓库、渠道）
    export_info_input = input.get("export_info") or {}
    export_info = cls.get_export_info(export_info_input)
    file_type = input["file_type"]
    
    # 3. 特殊处理：如果使用渠道相关过滤器，需注入 channel slug
    if "filter" in scope:
        scope = cls.add_channel_to_filter_scope(scope, input.get("filter", {}), export_info)
    
    # 4. 获取当前请求上下文（App / User）
    app = get_app_promise(info.context).get()
    
    # 5. ★ 关键点：创建 ExportFile 记录（登记请求）
    export_file = csv_models.ExportFile.objects.create(
        app=app, user=info.context.user
    )
    
    # 6. ★ 记录导出开始事件（审计追踪）
    export_started_event(
        export_file=export_file, app=app, user=info.context.user
    )
    
    # 7. ★ 触发异步任务（接力给后台）
    export_products_task.delay(export_file.pk, scope, export_info, file_type)
    
    # 8. 返回刚创建的 ExportFile 对象给前端
    export_file.refresh_from_db()
    return cls(export_file=export_file)
```

**关键设计要点**：
- 数据库记录创建和事件记录在 **当前请求线程** 中同步完成
- 使用 `export_file.pk` 作为任务参数传递，而不是整个对象（避免序列化问题和数据一致性问题）
- 立即返回响应，用户无需等待导出完成
- `export_started_event` 记录 `EXPORT_PENDING` 类型事件

事件创建代码（`saleor/csv/events.py:13-22`）：
```python
@allow_writer()
def export_started_event(*, export_file: "ExportFile", user=None, app=None) -> None:
    ExportEvent.objects.create(
        export_file=export_file, user=user, app=app, type=ExportEvents.EXPORT_PENDING
    )
```

## 三、关键细节深度剖析

### 3.1 成功通知与任务状态回写的先后顺序

这是一个非常容易混淆的执行顺序问题。让我们从代码层面精确核对：

**执行时序（以 `export_products` 为例）：**

```
export_products_task 函数
│
├─► 读取 ExportFile
├─► 调用 export_products(export_file, scope, ...)
│   │
│   ├─► 生成文件名
│   ├─► 构建查询集
│   ├─► 分批导出数据到临时文件
│   ├─► 【步骤 1】save_csv_file_in_export_file()  # 保存文件到 content_file
│   ├─► 关闭临时文件
│   └─► 【步骤 2】send_export_download_link_notification()  # 触发通知
│
├─► export_products_task 函数返回
│
└─► Celery 框架自动调用 on_success() 回调
    ├─► 【步骤 3】更新 ExportFile.status = SUCCESS
    └─► 创建 ExportEvent (type=EXPORT_SUCCESS)
```

**关键代码证据**：

1. **`export_products` 内部顺序**（`saleor/csv/utils/export.py:59-61`）：
```python
save_csv_file_in_export_file(export_file, temporary_file, file_name)  # 第59行
temporary_file.close()                                                 # 第60行
send_export_download_link_notification(export_file, "products")        # 第61行
```

2. **`on_success` 回调时机**（`saleor/csv/tasks.py:49-57`）：
```python
def on_success(self, retval, task_id, args, kwargs):
    # 这个方法由 Celery 框架在任务函数返回后自动调用
    export_file_id = args[0]
    export_file = ExportFile.objects.get(pk=export_file_id)
    export_file.status = JobStatus.SUCCESS  # 这里才更新状态
    ...
```

**重要结论**：
- ✅ **通知发送时，ExportFile.status 仍然是 PENDING**
- ✅ 通知 payload 中的 `status` 字段值为 `PENDING`，而非 `SUCCESS`
- ✅ `EXPORT_SUCCESS` 事件在通知之后才记录
- ✅ 这是**设计使然**：通知负责投递下载链接，状态回写是框架层面的职责

**通知 payload 生成时的状态**（`saleor/csv/notifications.py:13-26`）：
```python
def get_default_export_payload(export_file: "ExportFile") -> dict:
    return {
        ...
        "status": export_file.status,  # 此时仍为 PENDING
        ...
    }
```

---

### 3.2 失败路径仍触发完成类 webhook 的历史遗留问题深度剖析

#### 3.2.1 历史背景与设计意图

**事实部分**：

1. **问题代码**（`saleor/csv/notifications.py:67-73`）：
   ```python
   def send_export_failed_info(export_file: "ExportFile", data_type: str):
       ...
       manager.notify(NotifyEventType.CSV_EXPORT_FAILED, payload_func=handler.payload)
       # 👇 即使失败，仍然调用 *export_completed 方法
       if data_type == "gift cards":
           manager.gift_card_export_completed(export_file)
       if data_type == "products":
           manager.product_export_completed(export_file)
       if data_type == "voucher codes":
           manager.voucher_code_export_completed(export_file)
   ```

2. **接口层面的 deprecated 标记**（`saleor/plugins/base_plugin.py:809-811, 1348-1350, 1654-1656`）：
   ```python
   # Note: This method is deprecated and will be removed in a future release.
   # Webhook-related functionality will be moved from the plugin to core modules.
   gift_card_export_completed: Callable[["ExportFile", None], None]
   ```

3. **成功与失败路径的对称性**（事实）：
   - 成功路径（`send_export_download_link_notification`，`saleor/csv/notifications.py:36-51`）：先 `notify(CSV_EXPORT_SUCCESS)`，再调用 `*_export_completed`
   - 失败路径（`send_export_failed_info`，`saleor/csv/notifications.py:67-73`）：先 `notify(CSV_EXPORT_FAILED)`，再调用 `*_export_completed`
   - 代码结构完全对称

4. **测试用例的刻意设计**（`saleor/csv/tests/test_notifications.py:58-99`）：
   ```python
   def test_send_export_failed_info(...):
       # given
       file_mock = mock.MagicMock(spec=File)
       file_mock.name = "temp_file.csv"
       user_export_file.content_file = file_mock  # 手动设置 content_file
       user_export_file.save()

       # when
       notifications.send_export_failed_info(user_export_file, data_type)

       # then
       mocked_gift_card_export_completed.assert_called_once_with(user_export_file)
   ```
   - 测试手动设置了 `content_file`，没有覆盖真实失败场景（`content_file = None`）
   - 该测试的断言明确验证了失败时会调用 `*_export_completed` 方法

**推断部分**：

5. **设计意图推测**：
   - 这种对称性设计表明：`*_export_completed` 代表"导出流程已终结"，无论成功失败
   - 可能是为了保持向后兼容，让依赖 `*_EXPORT_COMPLETED` 事件的外部系统在失败时也能收到通知
   - 外部系统需要通过 payload 中的 `status` 字段判断实际结果

6. **测试缺陷的原因**：
   - 编写测试时可能只关注"是否调用了方法"，而忽略了真实场景下的参数状态
   - 或者是已知 bug 但暂时通过 mock 绕过，等待后续修复

---

#### 3.2.2 Webhook 事件拼装的问题代码

**Webhook Plugin 的实现**（`saleor/plugins/webhook/plugin.py:739-755`）：
```python
def _trigger_export_event(self, event_type: str, export: "ExportFile"):
    if webhooks := get_webhooks_for_event(event_type):
        payload = self._serialize_payload({
            "id": graphene.Node.to_global_id("ExportFile", export.id),
            "export": get_default_export_payload(export),
            "csv_link": build_absolute_uri(export.content_file.url),  # ⚠️ 无 null 检查
            "recipient_email": export.user.email if export.user else None,
        })
        self.trigger_webhooks_async(payload, event_type, webhooks, export, self.requestor)
```

---

#### 3.2.3 GraphQL URL 解析与 Webhook 拼装逻辑的对比

**关键差异点**：

| 场景 | 代码位置 | 处理逻辑 | content_file 为空时的行为 |
|------|---------|---------|-------------------------|
| GraphQL 查询 | `saleor/graphql/csv/types.py:91-95` | 有 null 检查 | 返回 `None`（正常） |
| Webhook 导出事件 | `saleor/plugins/webhook/plugin.py:745` | 无 null 检查 | 抛出 `ValueError`（崩溃） |

**GraphQL 安全实现**（事实）：
```python
@staticmethod
def resolve_url(root: models.ExportFile, _info: ResolveInfo):
    content_file = root.content_file
    if not content_file:       # ✅ 显式 null 检查
        return None            # ✅ 优雅返回 None
    return build_absolute_uri(content_file.url)
```

**Webhook 危险实现**（事实）：
```python
def _trigger_export_event(self, event_type: str, export: "ExportFile"):
    if webhooks := get_webhooks_for_event(event_type):
        payload = self._serialize_payload({
            ...
            "csv_link": build_absolute_uri(export.content_file.url),  # ⚠️ 无检查！
            ...
        })
```

**Django FileField 行为**（事实）：
- Django `FileField` 使用描述符模式，当值为 `None` 时，访问 `.url` 会抛出：
  ```
  ValueError: The 'content_file' attribute has no file associated with it.
  ```
- 这是 Django `FieldFile` 描述符的标准行为，而非 Python 原生的 `AttributeError`

**为何两者行为不一致**（推断）：
- GraphQL 解析器是用户侧 API，需要考虑各种边界情况，包括导出失败的场景
- Webhook 导出事件最初只设计为在**成功路径**调用，因此没有考虑 `content_file` 为空的情况
- 后来为了保持向后兼容，在失败路径也调用了 `*_export_completed`，但没有更新 webhook 的实现

---

#### 3.2.3 异常类型、触发条件与传播路径

**事实部分**：

1. **异常类型**：`ValueError`（不是 `AttributeError`）
   - Django `FileField` 描述符在值为 `None` 时，访问 `.url` 属性抛出标准的 `ValueError`
   - 代码证据：`django/db/models/fields/files.py` 中的 `FieldFile.__get__` 方法

2. **触发条件**（必须同时满足）：
   - ✅ 导出任务执行失败（抛出未捕获的异常）
   - ✅ Celery 调用 `on_failure` 回调，执行 `export_file.content_file = None`（`saleor/csv/tasks.py:32`）
   - ✅ 存在至少一个 **激活的** Webhook 订阅了 `*_EXPORT_COMPLETED` 事件
   - ✅ WebhookPlugin 处于激活状态

3. **精确的异常传播路径**：
   ```
   on_failure()
   ├─► export_file.content_file = None                    # saleor/csv/tasks.py:32
   ├─► export_file.status = FAILED                        # saleor/csv/tasks.py:35
   ├─► export_file.save(update_fields=["status", ...])    # saleor/csv/tasks.py:36
   ├─► export_failed_event(...)                           # saleor/csv/tasks.py:38-43
   └─► send_export_failed_info(export_file, data_type)    # saleor/csv/tasks.py:45
       ├─► manager.notify(CSV_EXPORT_FAILED)              # ✅ 正常执行
       └─► manager.product_export_completed(export_file)  # saleor/csv/notifications.py:71
           └─► __run_method_on_plugins()
               └─► WebhookPlugin.product_export_completed()
                   └─► _trigger_export_event()
                       ├─► get_webhooks_for_event() → 返回非空
                       └─► export.content_file.url        # 💥 ValueError!
   ```

4. **export_* 任务不具备自动重试功能**（事实）：
   - 代码证据 1：`saleor/csv/tasks.py:60, 77, 93` - `@app.task` 装饰器未配置任何重试参数
   - 代码证据 2：`saleor/csv/tasks.py:20` - `ExportTask` 基类未配置 `autoretry_for`、`max_retries` 等
   - 代码证据 3：`saleor/core/tasks.py:19-51` - `RestrictWriterDBTask` 基类仅做数据库访问限制，无重试逻辑
   - 代码证据 4：`saleor/celeryconf.py:32` - 默认 task_cls 配置为 `RestrictWriterDBTask`，无重试配置
   - 代码证据 5：`saleor/settings.py` - 无 `CELERY_TASK_DEFAULT_RETRY_DELAY` 等全局重试配置
   - **结论**：export_* 任务失败后**不会自动重试**

**推断部分**：

5. **异常后果**：
   - `__run_method_on_single_plugin` 没有 try-catch 保护（`saleor/plugins/manager.py:233-255`）
   - 异常向上传播，导致 `on_failure` 回调异常终止
   - 虽然 export_* 任务不会自动重试，但 `on_failure` 回调的异常终止会导致：
     - `EXPORT_FAILED` 事件已记录（因为在通知之前调用）
     - 但 `CSV_EXPORT_FAILED` webhook 通知（以及邮件通知）可能已发送（取决于执行顺序）
     - `*_EXPORT_COMPLETED` webhook 通知完全失败
     - 最终 `ExportFile.status` 已设置为 `FAILED`（因为在通知之前保存）
   - **注意**：`on_failure` 是 Celery 回调，其内部的异常不会触发任务重试，只会被 Celery 框架记录为错误日志

6. **为何测试未发现**（推断）：
   - 测试用例 `test_send_export_failed_info` 手动设置 `content_file = file_mock`（`saleor/csv/tests/test_notifications.py:62-66`），绕过了真实失败场景
   - 这是典型的测试用例设计缺陷，没有覆盖真实的执行路径

---

#### 3.2.4 对下游订阅方的影响及判定方法

**事实部分**：

1. **影响分类**（事实）：

| 订阅类型 | 影响程度 | 说明 |
|---------|---------|------|
| `CSV_EXPORT_FAILED` | 🔵 无影响 | payload 不访问 `content_file.url`，通知正常发送 |
| `CSV_EXPORT_SUCCESS` | 🔵 无影响 | 仅在成功路径触发，此时 `content_file` 已赋值 |
| `*_EXPORT_COMPLETED` | 🔴 严重 | 失败时触发会导致 `ValueError`，webhook 投递完全失败 |

2. **payload 结构**（事实）：
   - 成功时的 `*_EXPORT_COMPLETED` 事件 payload 包含 `csv_link` 字段
   - `CSV_EXPORT_SUCCESS` 事件 payload 包含 `csv_link` 字段
   - `CSV_EXPORT_FAILED` 事件 payload **不包含** `csv_link` 字段

3. **通知时序**（事实）：
   - 成功路径：通知发送时 `ExportFile.status` 为 `PENDING`，`on_success` 回调在通知之后才更新为 `SUCCESS`（已在 3.1 节验证）

**推断/建议部分**：

4. **下游系统判定方法**（推断）：

   对于订阅了 `*_EXPORT_COMPLETED` 事件的外部系统，需要通过 payload 中的字段判定结果：

   ```python
   # 成功时的 payload（简化）
   {
       "export": {
           "id": "RXhwb3J0RmlsZToxMjM=",
           "status": "success",  # ✅ SUCCESS
           "message": null,
           ...
       },
       "csv_link": "https://.../export_files/product_data_...csv",
       "recipient_email": "user@example.com",
       ...
   }

   # 失败时，如果 webhook 代码被修复后能正常发送，payload 会是：
   {
       "export": {
           "id": "RXhwb3J0RmlsZToxMjM=",
           "status": "failed",   # ❌ FAILED
           "message": "Export failed due to ...",
           ...
       },
       "csv_link": null,  # ⚠️ 修复后应该为 null，而不是崩溃
       ...
   }
   ```

5. **推荐的下游判定逻辑**（建议）：
   ```python
   def handle_export_completed(payload):
       export_info = payload.get("export", {})
       status = export_info.get("status")
       
       if status == "success":
           # 处理成功：下载 csv_link
           download_url = payload.get("csv_link")
           if download_url:
               process_export_file(download_url)
           else:
               # 异常情况：status 为 success 但无下载链接
               handle_missing_download_link(export_info)
       elif status == "failed":
           # 处理失败：记录错误，通知用户
           error_message = export_info.get("message")
           handle_export_failure(error_message)
       else:
           # status 为 "pending"（通知先于状态回写）
           # 需要轮询或等待后续事件
           handle_pending_state(export_info)
   ```

6. **注意事项**（事实 + 推断）：
   - 如 3.1 节所述，即使是成功路径，通知发送时 `status` 也可能是 `PENDING`，因为 `on_success` 回调在通知之后才更新状态
   - 如果 bug 未修复，下游系统永远收不到失败时的 `*_EXPORT_COMPLETED` 事件
   - 建议下游系统同时订阅 `CSV_EXPORT_FAILED` 事件作为失败场景的可靠通知源

---

#### 3.2.5 为何 Admin Email Plugin 不受影响

**事实部分**：

1. Admin Email Plugin 的失败通知处理完全独立于 `*_export_completed` 方法（`saleor/plugins/admin_email/notify_events.py:42-65`）：
   ```python
   # AdminEmailPlugin 只响应 notify() 调用，不响应 *_export_completed 调用
   def send_csv_export_failed(payload_func, config, plugin):
       template = get_email_template_or_default(...)
       if not template:
           return
       payload = payload_func()  # ✅ payload 不包含 csv_link，不会崩溃
       recipient_email = payload.get("recipient_email")
       ...
       send_export_failed_email_task.delay(...)
   ```

2. payload 结构差异（事实）：
   - ✅ `CSV_EXPORT_FAILED` payload：**不包含** `csv_link` 字段（`saleor/csv/notifications.py:58-61`）
   - ✅ `CSV_EXPORT_SUCCESS` payload：**包含** `csv_link` 字段（`saleor/csv/notifications.py:18-26`）
   - ⚠️ `*_EXPORT_COMPLETED` webhook payload：**总是包含** `csv_link` 字段（`saleor/plugins/webhook/plugin.py:745`）

**推断部分**：

3. 设计意图（推断）：
   - `CSV_EXPORT_FAILED` 通知只负责告知用户导出失败，不需要提供下载链接
   - `*_EXPORT_COMPLETED` webhook 最初设计为仅在成功时调用，因此总是假设 `content_file` 存在

---

### 3.3 Scope 参数从 Mutation 到 Celery 再到 Filter 的完整链路

这是一个经过多层校验、转换和反序列化的复杂链路。让我们逐层追踪：

#### 3.3.1 第一层：GraphQL Mutation 校验

**ExportScope 枚举**（`saleor/graphql/csv/enums.py:14-17`）：
```python
class ExportScope(BaseEnum):
    ALL = "all"
    IDS = "ids"
    FILTER = "filter"
```

**`get_scope` 分派逻辑**（`saleor/graphql/csv/mutations/base_export.py:24-30`）：
```python
@classmethod
def get_scope(cls, input, only_type) -> Mapping[str, list | dict | str]:
    scope = input["scope"]
    if scope == ExportScope.IDS.value:
        return cls.clean_ids(input, only_type)    # 校验并转换 ID
    if scope == ExportScope.FILTER.value:
        return cls.clean_filter(input)            # 校验 filter 非空
    return {"all": ""}
```

**`clean_ids` - Global ID 转换**（`saleor/graphql/csv/mutations/base_export.py:33-45`）：
```python
@classmethod
def clean_ids(cls, input, only_type) -> dict[str, list[str]]:
    ids = input.get("ids", [])
    if not ids:
        raise ValidationError(...)
    # 调用 get_global_ids_or_error 将 GraphQL global ID 转为数据库主键
    pks = cls.get_global_ids_or_error(ids, only_type=only_type, field="ids")
    return {"ids": pks}
```

**`get_global_ids_or_error` 实现**（`saleor/graphql/core/mutations.py:385-399`）：
```python
@classmethod
def get_global_ids_or_error(cls, ids, only_type=None, field="ids"):
    try:
        _nodes_type, pks = resolve_global_ids_to_primary_keys(
            ids, only_type, raise_error=True
        )
    except GraphQLError as e:
        raise ValidationError(...) from e
    return pks
```

**`resolve_global_ids_to_primary_keys` 实现**（`saleor/graphql/utils/__init__.py:60-90`）：
```python
def resolve_global_ids_to_primary_keys(ids, graphene_type=None, raise_error=False):
    pks = []
    for graphql_id in ids:
        node_type, _id = from_global_id_or_error(graphql_id)
        # 校验 ID 类型是否匹配
        if used_type and str(used_type) != str(node_type):
            raise GraphQLError(f"Must receive {str(used_type)} id: {graphql_id}.")
        pks.append(_id)
    return used_type, pks
```

#### 3.3.2 第二层：商品导出特有 - Channel 注入

**`add_channel_to_filter_scope`**（`saleor/graphql/csv/mutations/export_products.py:135-175`）：
```python
@classmethod
def add_channel_to_filter_scope(cls, scope: dict, filter_input: dict, export_info: dict):
    # 检查是否使用了渠道相关的过滤器
    used_channel_filters = set(filter_input.keys()) & CHANNEL_REQUIRED_FILTERS
    if not used_channel_filters:
        return scope
    
    channel_pks = export_info.get("channels") or []
    # 必须恰好指定一个渠道
    if len(channel_pks) != 1:
        raise ValidationError(
            "Exactly one channel must be provided in export_info.channels "
            f"when using channel-dependent filters: {sorted(used_channel_filters)}."
        )
    
    # 将 channel slug 注入到 filter dict 中
    updated_filter = {**scope["filter"], "channel": channel.slug}
    return {"filter": updated_filter}
```

**渠道相关过滤器**（`saleor/graphql/csv/mutations/export_products.py:27-38`）：
```python
CHANNEL_REQUIRED_FILTERS = frozenset([
    "is_published", "published_from", "is_available",
    "available_from", "is_visible_in_listing",
    "price", "minimal_price", "stock_availability",
])
```

#### 3.3.3 第三层：Celery 序列化 - 类型丢失问题

Scope 通过 Celery 传递时会经过 **JSON 序列化**，这会导致：
- `datetime` 对象 → ISO 格式字符串
- `date` 对象 → ISO 格式字符串
- 所有非基本类型都会被字符串化

**问题根源**（`saleor/csv/utils/export.py:153` 注释）：
```python
def parse_input(data: Any) -> dict[str, str | dict]:
    """Parse input into correct data types.

    Scope coming from Celery will be passed as strings.  # 👈 关键注释
    """
```

#### 3.3.4 第四层：反序列化 - 类型恢复

**`parse_input` 实现**（`saleor/csv/utils/export.py:150-176`）：
```python
def parse_input(data: Any) -> dict[str, str | dict]:
    if "attributes" in data:
        serialized_attributes = []
        for attr in data.get("attributes") or []:
            # 恢复 date_time 类型
            if "date_time" in attr:
                if gte := attr["date_time"].get("gte"):
                    attr["date_time"]["gte"] = datetime.datetime.fromisoformat(gte)
                if lte := attr["date_time"].get("lte"):
                    attr["date_time"]["lte"] = datetime.datetime.fromisoformat(lte)
            # 恢复 date 类型
            if "date" in attr:
                if gte := attr["date"].get("gte"):
                    attr["date"]["gte"] = datetime.date.fromisoformat(gte)
                if lte := attr["date"].get("lte"):
                    attr["date"]["lte"] = datetime.date.fromisoformat(lte)
            serialized_attributes.append(attr)
        if serialized_attributes:
            data["attributes"] = serialized_attributes
    return data
```

**测试验证**（`saleor/csv/tests/export/test_export.py:936-972`）：
```python
def test_parse_input():
    data = {
        "attributes": [
            {
                "slug": "release-date-time",
                "date_time": {
                    "gte": "2019-08-08T00:00:00+02:00",  # 字符串
                    "lte": "2021-08-08T00:00:00+02:00",  # 字符串
                },
            },
        ],
    }
    parsed_data = parse_input(data)
    # 断言类型已恢复
    assert isinstance(parsed_data["attributes"][0]["date_time"]["gte"], datetime.datetime)
    assert isinstance(parsed_data["attributes"][0]["date_time"]["lte"], datetime.datetime)
```

#### 3.3.5 第五层：Filter 层应用

**`get_queryset` 实现**（`saleor/csv/utils/export.py:136-147`）：
```python
def get_queryset(model, filter, scope: dict) -> "QuerySet":
    queryset = model.objects.using(settings.DATABASE_CONNECTION_REPLICA_NAME).all()
    if "ids" in scope:
        # 直接使用主键过滤
        queryset = model.objects.using(...).filter(pk__in=scope["ids"])
    elif "filter" in scope:
        # 先反序列化，再应用过滤器
        queryset = filter(data=parse_input(scope["filter"]), queryset=queryset).qs
    queryset = queryset.order_by("pk")
    return queryset
```

#### 3.3.6 Scope 完整流转图

```
GraphQL 输入 (前端)
    │
    ▼
scope = ExportScope.IDS / FILTER / ALL
    │
    ├─► IDS: clean_ids()
    │    └─► get_global_ids_or_error()
    │         └─► resolve_global_ids_to_primary_keys()
    │              └─► {"ids": [1, 2, 3]}  # 数据库主键
    │
    ├─► FILTER: clean_filter()
    │    └─► 校验 filter 非空
    │         └─► {"filter": {...}}
    │              │
    │              └─► (商品导出) add_channel_to_filter_scope()
    │                   └─► 注入 channel slug
    │                        └─► {"filter": {..., "channel": "default-channel"}}
    │
    └─► ALL: {"all": ""}
    │
    ▼
Celery JSON 序列化
    │  ⚠️  datetime → "2024-01-01T00:00:00+00:00"
    │  ⚠️  date → "2024-01-01"
    │
    ▼
Celery Worker 接收
    │
    ▼
get_queryset()
    │
    ├─► ids: filter(pk__in=scope["ids"])
    │
    └─► filter: parse_input(scope["filter"])  # 恢复类型
              └─► datetime.fromisoformat() / date.fromisoformat()
                   └─► Filter(data=...).qs
```

---

## 四、第二阶段：后台执行编排

### 4.1 Celery 任务定义

任务定义在 `saleor/csv/tasks.py`。

**核心基类 ExportTask**：
```python
class ExportTask(RestrictWriterDBTask):
    TASK_NAME_TO_DATA_TYPE_MAPPING = {
        "export-products": "products",
        "export-gift-cards": "gift cards",
        "export-voucher-codes": "voucher codes",
    }

    def on_failure(self, exc, task_id, args, kwargs, einfo):
        """任务失败回调"""
        export_file_id = args[0]
        export_file = ExportFile.objects.get(pk=export_file_id)

        # 更新状态为失败
        export_file.content_file = None
        export_file.status = JobStatus.FAILED
        export_file.save(update_fields=["status", "updated_at", "content_file"])

        # 记录失败事件
        events.export_failed_event(
            export_file=export_file,
            user=export_file.user,
            app=export_file.app,
            message=str(exc),
            error_type=str(einfo.type),
        )

        # 发送失败通知
        data_type = ExportTask.TASK_NAME_TO_DATA_TYPE_MAPPING.get(self.name, "unknown data")
        send_export_failed_info(export_file, data_type)

    def on_success(self, retval, task_id, args, kwargs):
        """任务成功回调"""
        export_file_id = args[0]
        export_file = ExportFile.objects.get(pk=export_file_id)
        
        # 更新状态为成功
        export_file.status = JobStatus.SUCCESS
        export_file.save(update_fields=["status", "updated_at"])
        
        # 记录成功事件
        events.export_success_event(
            export_file=export_file, user=export_file.user, app=export_file.app
        )
```

**三个具体任务函数**：
```python
# 商品导出任务
@app.task(name="export-products", base=ExportTask)
def export_products_task(export_file_id: int, scope: dict, export_info: dict, 
                         file_type: str, delimiter: str = ","):
    with allow_writer():
        # 从主库读取（避免主从延迟导致数据不存在）
        export_file = ExportFile.objects.select_related("app", "user").get(pk=export_file_id)
    export_products(export_file, scope, export_info, file_type, delimiter)

# 礼品卡导出任务
@app.task(name="export-gift-cards", base=ExportTask)
def export_gift_cards_task(export_file_id: int, scope: dict, file_type: str, ...):
    ...

# 优惠券代码导出任务
@app.task(name="export-voucher-codes", base=ExportTask)
def export_voucher_codes_task(export_file_id: int, file_type: str, ...):
    ...
```

### 4.2 任务执行的关键设计

1. **`RestrictWriterDBTask` 基类** - 确保任务在只读模式下运行（除了明确标记的 `allow_writer()` 区域）
2. **主库读取** - 任务开始时用 `allow_writer()` + 主库读取 ExportFile，避免主从延迟
3. **状态更新通过回调** - `on_success` / `on_failure` 回调由 Celery 框架自动调用
4. **事件记录完整** - 每个状态变更都有对应的 ExportEvent 记录

### 4.3 数据导出核心逻辑

代码位置：`saleor/csv/utils/export.py`

**通用导出流程**（以 `export_products` 为例）：
```python
def export_products(export_file, scope, export_info, file_type, delimiter=","):
    # 1. 生成文件名
    file_name = get_filename("product", file_type)
    
    # 2. 根据 scope 构建查询集
    queryset = get_queryset(Product, ProductFilter, scope)
    
    # 3. 获取导出字段和表头
    export_fields, file_headers, data_headers = get_product_export_fields_and_headers_info(export_info)
    
    # 4. 创建临时文件并写入表头
    temporary_file = create_file_with_headers(file_headers, delimiter, file_type)
    
    # 5. ★ 分批导出数据（避免内存溢出）
    export_products_in_batches(
        queryset, export_info, set(export_fields), data_headers, 
        delimiter, temporary_file, file_type
    )
    
    # 6. ★ 保存文件到 ExportFile（接力给下一环）
    save_csv_file_in_export_file(export_file, temporary_file, file_name)
    temporary_file.close()
    
    # 7. ★ 触发下载链接通知（最后一棒）
    send_export_download_link_notification(export_file, "products")
```

**分批处理机制**：
```python
BATCH_SIZE = 1000

def export_products_in_batches(queryset, ...):
    for batch_pks in queryset_in_batches(queryset, BATCH_SIZE):
        # 每批 1000 条，使用 prefetch_related 优化查询
        product_batch = Product.objects.filter(pk__in=batch_pks).prefetch_related(...)
        export_data = get_products_data(product_batch, ...)
        append_to_file(export_data, headers, temporary_file, file_type, delimiter)
```

**文件保存逻辑**（`saleor/csv/utils/export.py:274-278`）：
```python
@allow_writer()
def save_csv_file_in_export_file(export_file, temporary_file, file_name):
    export_file.content_file.save(file_name, temporary_file)
```

## 五、第三阶段：下载产物投递

### 5.1 通知触发机制

代码位置：`saleor/csv/notifications.py`

**成功通知**：
```python
def send_export_download_link_notification(export_file: "ExportFile", data_type: str):
    def _generate_payload():
        payload = {
            "export": get_default_export_payload(export_file),
            "csv_link": build_absolute_uri(export_file.content_file.url),
            "recipient_email": export_file.user.email if export_file.user else None,
            "data_type": data_type,
            **get_site_context(),
        }
        return payload

    manager = get_plugins_manager(allow_replica=True)
    handler = NotifyHandler(_generate_payload)
    
    # 1. 触发通用 CSV_EXPORT_SUCCESS 通知事件
    manager.notify(NotifyEventType.CSV_EXPORT_SUCCESS, payload_func=handler.payload)
    
    # 2. 触发特定类型的 webhook 事件
    if data_type == "gift cards":
        manager.gift_card_export_completed(export_file)
    if data_type == "products":
        manager.product_export_completed(export_file)
    if data_type == "voucher codes":
        manager.voucher_code_export_completed(export_file)
```

**失败通知**：
```python
def send_export_failed_info(export_file: "ExportFile", data_type: str):
    def _generate_payload():
        payload = {
            "export": get_default_export_payload(export_file),
            "recipient_email": export_file.user.email if export_file.user else None,
            "data_type": data_type,
            **get_site_context(),
        }
        return payload

    manager = get_plugins_manager(allow_replica=True)
    handler = NotifyHandler(_generate_payload)
    manager.notify(NotifyEventType.CSV_EXPORT_FAILED, payload_func=handler.payload)
    # 同样触发特定类型的 webhook 事件
    ...
```

### 5.2 NotifyHandler 延迟加载设计

代码位置：`saleor/core/notify.py:5-19`

```python
class NotifyHandler:
    """payload 按需生成，只有当插件/webhook 真正需要时才生成"""
    
    def __init__(self, payload_func):
        self.generate_payload_func = payload_func

    @cache
    def payload(self):
        return self.generate_payload_func()
```

**设计亮点**：
- 使用 `functools.cache` 确保 payload 只生成一次
- 如果没有插件订阅该事件，payload 永远不会生成，避免不必要的计算

### 5.3 Admin Email Plugin 邮件投递

代码位置：`saleor/plugins/admin_email/notify_events.py:42-65`

```python
def send_csv_export_success(payload_func: Callable[[], dict], config: dict, plugin):
    # 1. 获取邮件模板
    template = get_email_template_or_default(
        plugin,
        constants.CSV_EXPORT_SUCCESS_TEMPLATE_FIELD,
        constants.CSV_EXPORT_SUCCESS_DEFAULT_TEMPLATE,
        ...
    )
    
    # 2. 模板为空表示不发送该通知
    if not template:
        return
    
    # 3. 生成 payload（此时才真正调用 payload_func）
    payload = payload_func()
    recipient_email = payload.get("recipient_email")
    if not recipient_email:
        return
    
    # 4. 获取邮件主题
    subject = get_email_subject(...)
    
    # 5. ★ 异步发送邮件（又是一次接力！）
    send_email_with_link_to_download_file_task.delay(
        recipient_email, payload, config, subject, template
    )
```

**事件映射**（`saleor/plugins/admin_email/plugin.py:36-43`）：
```python
def get_admin_event_map():
    return {
        ...
        AdminNotifyEvent.CSV_EXPORT_SUCCESS: send_csv_export_success,
        AdminNotifyEvent.CSV_EXPORT_FAILED: send_csv_export_failed,
    }
```

### 5.4 邮件发送任务

代码位置：`saleor/plugins/admin_email/tasks.py`

```python
@app.task
def send_email_with_link_to_download_file_task(
    recipient_email: str, payload: dict, config: dict, subject: str, template: str
):
    # 渲染邮件模板
    content = render_email_template(template, payload)
    # 发送邮件
    send_email(
        config, recipient_email, subject, content, 
        from_email=settings.DEFAULT_FROM_EMAIL
    )
    # 记录邮件已发送事件
    export_file_sent_event(
        export_file_id=payload["export"]["id"],  # 这里是 global_id，需要转换
        user_id=...
    )
```

### 5.5 Webhook 通知外部系统

除了邮件通知，PluginManager 还会触发 webhook：

```python
# 在 notifications.py 中
manager.product_export_completed(export_file)
```

这会触发 `WebhookEventAsyncType.PRODUCT_EXPORT_COMPLETED` 事件，通知所有订阅了该事件的外部系统。

## 六、完整时序图（接力关系）

```
HTTP 请求线程
│
├─► 创建 ExportFile (status=PENDING)
├─► 创建 ExportEvent (type=EXPORT_PENDING)
├─► 发送 Celery 任务: export_products_task.delay(pk, scope, ...)
└─► 返回 ExportFile 给前端

───────────────────────────────────────────────────────────────

Celery Worker 线程 (export_products_task)
│
├─► 从主库读取 ExportFile
├─► 分批查询数据并写入临时文件
│   ├─► 批次 1: 查询 1000 条 → 写入文件
│   ├─► 批次 2: 查询 1000 条 → 写入文件
│   └─► ...
├─► 保存文件到 content_file 字段
├─► 调用 send_export_download_link_notification()
│   └─► manager.notify(CSV_EXPORT_SUCCESS, payload_func)
│       ├─► AdminEmailPlugin.notify()
│       │   └─► send_csv_export_success()
│       │       └─► send_email_with_link_to_download_file_task.delay()
│       └─► WebhookPlugin.notify()
│           └─► 发送 HTTP 请求到外部系统
└─► 任务完成，Celery 自动调用 on_success()
    ├─► 更新 ExportFile.status = SUCCESS
    └─► 创建 ExportEvent (type=EXPORT_SUCCESS)

───────────────────────────────────────────────────────────────

Celery Worker 线程 (send_email_with_link_to_download_file_task)
│
├─► 渲染邮件模板
├─► 发送 SMTP 邮件
└─► 创建 ExportEvent (type=EXPORTED_FILE_SENT)
```

## 七、失败处理流程

```
Celery Worker 线程 (export_products_task)
│
└─► 任务抛出异常
    └─► Celery 自动调用 on_failure()
        ├─► 更新 ExportFile.status = FAILED
        ├─► 清空 content_file
        ├─► 创建 ExportEvent (type=EXPORT_FAILED, parameters={message, error_type})
        └─► 调用 send_export_failed_info()
            └─► manager.notify(CSV_EXPORT_FAILED, payload_func)
                ├─► AdminEmailPlugin 发送失败邮件
                │   └─► send_export_failed_email_task.delay()
                │       └─► 创建 ExportEvent (type=EXPORT_FAILED_INFO_SENT)
                └─► WebhookPlugin 通知外部系统
```

## 八、关键设计模式总结

### 8.1 接力棒传递模式

整个流程是典型的 **"接力棒"** 模式：
1. **第一棒（HTTP线程）**：创建记录 → 交棒给 Celery
2. **第二棒（导出任务）**：执行导出 → 保存文件 → 交棒给通知系统
3. **第三棒（通知任务）**：发送邮件/Webhook → 记录事件

每一步只负责自己的职责，通过数据库记录和消息队列传递状态。

### 8.2 数据库读写分离策略

- **读取**：优先使用从库（`settings.DATABASE_CONNECTION_REPLICA_NAME`）
- **写入**：使用 `@allow_writer()` 装饰器标记需要主库的操作
- **任务启动时**：强制从主库读取 ExportFile，避免主从延迟

### 8.3 事件溯源模式

每个状态变更都记录 `ExportEvent`，提供完整的审计追踪：
- `EXPORT_PENDING` - 请求已登记
- `EXPORT_SUCCESS` - 导出成功
- `EXPORT_FAILED` - 导出失败
- `EXPORTED_FILE_SENT` - 下载链接已发送
- `EXPORT_FAILED_INFO_SENT` - 失败通知已发送
- `EXPORT_DELETED` - 文件已删除

### 8.4 延迟加载优化

`NotifyHandler` 使用 `@cache` 装饰器实现 payload 懒加载，只有当真正有订阅者时才生成 payload。

### 8.5 旧文件清理

还有一个定时任务 `delete_old_export_files`（`saleor/csv/tasks.py:109-134`），定期清理超过 `settings.EXPORT_FILES_TIMEDELTA` 的导出文件。

## 九、核心文件索引

| 模块 | 文件路径 | 职责 |
|------|---------|------|
| 数据模型 | `saleor/csv/models.py` | ExportFile, ExportEvent 定义 |
| 基类模型 | `saleor/core/models.py:141` | Job 抽象基类 |
| 状态枚举 | `saleor/core/__init__.py:7` | JobStatus 定义 |
| Mutation 入口 | `saleor/graphql/csv/mutations/export_products.py` | 商品导出请求登记 |
| Mutation 入口 | `saleor/graphql/csv/mutations/export_gift_cards.py` | 礼品卡导出请求登记 |
| Mutation 入口 | `saleor/graphql/csv/mutations/export_voucher_codes.py` | 优惠券代码导出请求登记 |
| Mutation 基类 | `saleor/graphql/csv/mutations/base_export.py` | 通用导出逻辑 |
| 异步任务 | `saleor/csv/tasks.py` | Celery 任务定义与回调 |
| 导出核心逻辑 | `saleor/csv/utils/export.py` | 数据查询、分批导出、文件保存 |
| 通知触发 | `saleor/csv/notifications.py` | 通知事件触发 |
| 事件记录 | `saleor/csv/events.py` | ExportEvent 创建函数 |
| 通知类型 | `saleor/core/notify.py` | NotifyEventType, NotifyHandler |
| 邮件插件 | `saleor/plugins/admin_email/plugin.py` | AdminEmailPlugin 定义 |
| 邮件通知逻辑 | `saleor/plugins/admin_email/notify_events.py` | 邮件发送触发逻辑 |
| 邮件发送任务 | `saleor/plugins/admin_email/tasks.py` | 异步邮件发送 |
| Webhook 插件 | `saleor/plugins/webhook/plugin.py` | Webhook 通知实现 |
| 基础插件接口 | `saleor/plugins/base_plugin.py` | *export_completed 方法定义 |
| Global ID 转换 | `saleor/graphql/utils/__init__.py` | resolve_global_ids_to_primary_keys |
| Mutation 基类 | `saleor/graphql/core/mutations.py` | get_global_ids_or_error |
| 事件类型枚举 | `saleor/webhook/event_types.py` | Webhook 事件类型定义 |
| 范围枚举 | `saleor/graphql/csv/enums.py` | ExportScope 枚举定义 |
| 插件执行器 | `saleor/plugins/manager.py:233-255` | `__run_method_on_single_plugin` 无异常保护 |
| 异步传输 | `saleor/webhook/transport/asynchronous/transport.py:477-517` | `trigger_webhooks_async` 实现 |
| Webhook 查询 | `saleor/webhook/utils.py:69-84` | `get_webhooks_for_event` 实现 |
