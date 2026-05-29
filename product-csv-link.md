# 商品 CSV 导入导出代码链路分析

## 整体架构概览

```
GraphQL API 层
    ↓
任务调度层 (Celery)
    ↓
批次处理层 (BATCH_SIZE = 1000)
    ↓
数据处理层 (字段映射、关系解析)
    ↓
文件生成层 (petl 库)
    ↓
存储与通知层
```

---

## 一、字段映射逻辑

### 1.1 字段映射核心定义

**文件**: `saleor/csv/utils/__init__.py:1-93`

`ProductExportFields` 类定义了完整的字段映射关系，分为以下几类：

#### 基础字段映射 (`HEADERS_TO_FIELDS_MAPPING`)

```python
"fields": {
    "id": "id",
    "name": "name",
    "description": "description_as_str",
    "category": "category__slug",
    "product type": "product_type__name",
    "product weight": "product_weight",
    "variant id": "variants__id",
    "variant sku": "variants__sku",
    "variant weight": "variant_weight",
    "variant is preorder": "variants__is_preorder",
    ...
}
```

- **键**：CSV 表头名称（用户可见）
- **值**：Django ORM 查询路径（双下划线 `__` 表示跨表关联）

#### 多对多关系字段映射

```python
"product_many_to_many": {
    "collections": "collections__slug",
    "product media": "media__image",
},
"variant_many_to_many": {
    "variant media": "variants__media__image"
}
```

#### 属性字段映射

| 类型 | 映射字典 | 说明 |
|------|---------|------|
| 商品属性 | `PRODUCT_ATTRIBUTE_FIELDS` | 30-47 行，定义了 16 种属性字段的 ORM 路径 |
| 变体属性 | `VARIANT_ATTRIBUTE_FIELDS` | 66-84 行，定义变体属性的 ORM 路径 |

#### 渠道与仓库字段映射

| 类型 | 映射字典 | 说明 |
|------|---------|------|
| 商品渠道列表 | `PRODUCT_CHANNEL_LISTING_FIELDS` | 49-58 行，包含货币代码、发布状态、发布日期等 |
| 变体渠道列表 | `VARIANT_CHANNEL_LISTING_FIELDS` | 86-93 行，包含价格、成本价、预购阈值等 |
| 仓库库存 | `WAREHOUSE_FIELDS` | 60-64 行，包含仓库 slug 和库存数量 |

### 1.2 字段映射流程

**入口函数**: `get_product_export_fields_and_headers_info()`  
**文件**: `saleor/csv/utils/product_headers.py:13-31`

```
1. get_product_export_fields_and_headers(export_info)
   ├─ 解析 export_info["fields"]
   ├─ 使用 ChainMap 合并 HEADERS_TO_FIELDS_MAPPING
   ├─ 返回 (export_fields, file_headers)
   │
2. get_attributes_headers(export_info)
   ├─ 查询 Attribute 表
   ├─ 按 product_types / product_variant_types 分类
   ├─ 生成表头格式: "slug-value (product attribute)"
   │
3. get_warehouses_headers(export_info)
   └─ 生成格式: "slug-value (warehouse quantity)"
   │
4. get_channels_headers(export_info)
   └─ 生成格式: "slug-value (channel currency code)"
```

### 1.3 GraphQL 枚举到字段名的映射

**文件**: `saleor/graphql/csv/enums.py:35-48`

`ProductFieldEnum` 定义了 GraphQL API 暴露的字段枚举，与 `HEADERS_TO_FIELDS_MAPPING["fields"]` 的键一一对应：

```python
class ProductFieldEnum(BaseEnum):
    NAME = "name"                    # → "name"
    DESCRIPTION = "description"      # → "description"
    PRODUCT_TYPE = "product type"    # → "product type"
    CATEGORY = "category"            # → "category"
    PRODUCT_WEIGHT = "product weight"
    COLLECTIONS = "collections"
    ...
```

---

## 二、批次处理机制

### 2.1 批次处理核心工具

**文件**: `saleor/core/utils/batches.py:4-17`

```python
def queryset_in_batches(queryset: QuerySet, batch_size: int):
    start_pk = 0
    while True:
        qs = queryset.filter(pk__gt=start_pk).order_by("pk")[:batch_size]
        pks = list(qs.values_list("pk", flat=True))
        if not pks:
            break
        yield pks
        start_pk = pks[-1]
```

**关键特性**:
- 使用 `pk__gt` 游标式分页，避免 offset 性能问题
- 按主键排序，确保数据完整性
- 每次只返回主键列表，减少内存占用

### 2.2 导出批次处理流程

**文件**: `saleor/csv/utils/export.py:192-223`

**常量**: `BATCH_SIZE = 1000` (第 26 行)

```
export_products_in_batches()
    ↓
for batch_pks in queryset_in_batches(queryset, BATCH_SIZE):
    ├─ 1. 预取关联数据 (prefetch_related)
    │   ├─ attributevalues
    │   ├─ variants
    │   ├─ collections
    │   ├─ media
    │   ├─ product_type
    │   └─ category
    │
    ├─ 2. get_products_data() - 数据处理
    │   ├─ annotate 计算字段 (product_weight, variant_weight)
    │   ├─ values() 获取基础字段
    │   ├─ get_products_relations_data() - 商品关系数据
    │   │   ├─ 多对多字段 (collections, media)
    │   │   ├─ 商品属性数据
    │   │   └─ 商品渠道列表数据
    │   └─ get_variants_relations_data() - 变体关系数据
    │       ├─ 变体属性数据
    │       ├─ 变体渠道列表数据
    │       └─ 仓库库存数据
    │
    └─ 3. append_to_file() - 写入文件
        └─ 使用 petl.fromdicts() 转换并追加
```

### 2.3 批量导入处理流程 (ProductBulkCreate)

**文件**: `saleor/graphql/product/bulk_mutations/product_bulk_create.py:938-972`

```
perform_mutation()
    ↓
1. clean_products() - 数据清洗与验证
   ├─ 预加载所有 Warehouse 和 Channel
   ├─ 检测重复 SKU
   └─ 对每个产品:
      ├─ clean_base_fields() - 基础字段验证
      ├─ clean_attributes() - 属性验证
      ├─ clean_media() - 媒体验证
      ├─ clean_product_channel_listings() - 渠道列表验证
      └─ clean_variants() - 变体验证 (委托给 ProductVariantBulkCreate)
    ↓
2. create_products() - 构建实例 (不保存)
   ├─ construct_instance() 构建 Product 实例
   ├─ 创建变体实例 (不保存)
   └─ 收集所有关联数据
    ↓
3. 错误策略检查 (error_policy)
   ├─ REJECT_EVERYTHING: 任何错误全部拒绝
   └─ REJECT_FAILED_ROWS: 只拒绝错误行
    ↓
4. save() - 批量保存
   ├─ Product.objects.bulk_create()
   ├─ ProductMedia.objects.bulk_create()
   ├─ ProductChannelListing.objects.bulk_create()
   ├─ AttributeAssignmentMixin.save() - 保存属性
   └─ save_variants() - 批量保存变体
    ↓
5. _save_m2m() - 保存多对多关系
   └─ CollectionProduct.objects.bulk_create()
    ↓
6. post_save_actions() - 后置操作
   ├─ 触发 webhook 事件 (product_created)
   └─ 标记促销规则需要重新计算
```

### 2.4 批量导入的错误处理策略

**文件**: `saleor/graphql/product/bulk_mutations/product_bulk_create.py:949-958`

```python
if any(index_error_map.values()):
    if error_policy == ErrorPolicyEnum.REJECT_EVERYTHING.value:
        results = get_results(instances_data_with_errors_list, True)
        return ProductBulkCreate(count=0, results=results)
    
    if error_policy == ErrorPolicyEnum.REJECT_FAILED_ROWS.value:
        for data in instances_data_with_errors_list:
            if data["errors"] and data["instance"]:
                data["instance"] = None
```

**index_error_map 结构**:
```python
defaultdict(list)
{
    product_index: [
        ProductBulkCreateError(
            path="variants.0.sku",
            message="SKU already exists",
            code="DUPLICATED",
            ...
        ),
        ...
    ],
    ...
}
```

---

## 三、错误反馈机制

### 3.1 导出错误处理

**文件**: `saleor/csv/tasks.py:28-47`

```python
class ExportTask(RestrictWriterDBTask):
    def on_failure(self, exc, task_id, args, kwargs, einfo):
        export_file_id = args[0]
        export_file = ExportFile.objects.get(pk=export_file_id)
        
        # 1. 更新状态
        export_file.content_file = None
        export_file.status = JobStatus.FAILED
        export_file.save(update_fields=["status", "updated_at", "content_file"])
        
        # 2. 记录错误事件
        events.export_failed_event(
            export_file=export_file,
            user=export_file.user,
            app=export_file.app,
            message=str(exc),
            error_type=str(einfo.type),
        )
        
        # 3. 发送通知
        send_export_failed_info(export_file, data_type)
```

### 3.2 事件记录系统

**文件**: `saleor/csv/events.py:13-82`

| 事件类型 | 触发时机 | 参数 |
|---------|---------|------|
| `EXPORT_PENDING` | 导出任务启动时 | - |
| `EXPORT_SUCCESS` | 导出成功时 | - |
| `EXPORT_FAILED` | 导出失败时 | `message`, `error_type` |
| `EXPORT_DELETED` | 导出文件删除时 | - |
| `EXPORTED_FILE_SENT` | 导出文件通知已发送 | - |
| `EXPORT_FAILED_INFO_SENT` | 失败通知已发送 | - |

**事件存储模型**: `ExportEvent`  
**文件**: `saleor/csv/models.py:22-36`

```python
class ExportEvent(models.Model):
    date = models.DateTimeField(default=timezone.now)
    type = models.CharField(max_length=255, choices=ExportEvents.CHOICES)
    parameters = JSONField(blank=True, default=dict)  # 存储错误详情
    export_file = models.ForeignKey(ExportFile, ...)
    user = models.ForeignKey(User, ...)
    app = models.ForeignKey(App, ...)
```

### 3.3 通知系统

**文件**: `saleor/csv/notifications.py:29-73`

#### 成功通知
```python
def send_export_download_link_notification(export_file, data_type):
    payload = {
        "export": get_default_export_payload(export_file),
        "csv_link": build_absolute_uri(export_file.content_file.url),
        "recipient_email": export_file.user.email,
        "data_type": data_type,
        **get_site_context(),
    }
    manager.notify(NotifyEventType.CSV_EXPORT_SUCCESS, payload_func=handler.payload)
    manager.product_export_completed(export_file)  # webhook
```

#### 失败通知
```python
def send_export_failed_info(export_file, data_type):
    payload = {
        "export": get_default_export_payload(export_file),
        "recipient_email": export_file.user.email,
        "data_type": data_type,
        **get_site_context(),
    }
    manager.notify(NotifyEventType.CSV_EXPORT_FAILED, payload_func=handler.payload)
```

### 3.4 GraphQL 输入验证错误

**文件**: `saleor/graphql/csv/mutations/export_products.py:134-175`

```python
def add_channel_to_filter_scope(cls, scope, filter_input, export_info):
    used_channel_filters = set(filter_input.keys()) & CHANNEL_REQUIRED_FILTERS
    if used_channel_filters:
        channel_pks = export_info.get("channels") or []
        if len(channel_pks) != 1:
            raise ValidationError({
                "channels": ValidationError(
                    "Exactly one channel must be provided...",
                    code=ExportErrorCode.REQUIRED.value,
                )
            })
```

**错误码定义**: `ExportErrorCode`  
**文件**: `saleor/csv/error_codes.py:1-8`

```python
class ExportErrorCode(Enum):
    GRAPHQL_ERROR = "graphql_error"
    INVALID = "invalid"
    NOT_FOUND = "not_found"
    REQUIRED = "required"
```

---

## 四、完整代码链路衔接

### 4.1 导出完整链路

```
GraphQL 入口: ExportProducts.perform_mutation()
│   文件: saleor/graphql/csv/mutations/export_products.py:109-132
│
├─ 1. get_scope() - 解析导出范围
│   ├─ ALL: 导出全部
│   ├─ IDS: 导出指定 ID
│   └─ FILTER: 按过滤条件导出
│
├─ 2. get_export_info() - 解析导出字段
│   ├─ fields: 基础字段列表
│   ├─ attributes: 属性 ID 列表
│   ├─ warehouses: 仓库 ID 列表
│   └─ channels: 渠道 ID 列表
│
├─ 3. add_channel_to_filter_scope() - 渠道过滤器验证
│
├─ 4. 创建 ExportFile 记录
│   export_file = csv_models.ExportFile.objects.create(...)
│
├─ 5. 记录启动事件
│   events.export_started_event(...)
│
└─ 6. 提交异步任务
    export_products_task.delay(export_file.pk, scope, export_info, file_type)
    
    ↓ (Celery 异步执行)
    
Celery 任务: export_products_task()
│   文件: saleor/csv/tasks.py:60-74
│
└─ export_products()
    │   文件: saleor/csv/utils/export.py:29-61
    │
    ├─ 1. get_queryset() - 获取查询集
    │   ├─ 按 ID 过滤 或 按 filter 过滤
    │   └─ 使用 replica 数据库读取
    │
    ├─ 2. get_product_export_fields_and_headers_info()
    │   └─ 字段映射 → 表头生成
    │
    ├─ 3. create_file_with_headers()
    │   └─ 创建临时文件，写入表头
    │
    ├─ 4. export_products_in_batches()
    │   └─ 批次处理 (BATCH_SIZE = 1000)
    │       ├─ queryset_in_batches()
    │       ├─ get_products_data()
    │       └─ append_to_file()
    │
    ├─ 5. save_csv_file_in_export_file()
    │   └─ 保存文件到 ExportFile.content_file
    │
    └─ 6. send_export_download_link_notification()
        ├─ manager.notify(CSV_EXPORT_SUCCESS)
        └─ manager.product_export_completed()
```

### 4.2 导入完整链路 (ProductBulkCreate)

```
GraphQL 入口: ProductBulkCreate.perform_mutation()
│   文件: saleor/graphql/product/bulk_mutations/product_bulk_create.py:938-972
│   装饰器: @traced_atomic_transaction() - 整个操作在事务中
│
├─ 1. clean_products() - 数据清洗与验证
│   │
│   ├─ 预加载所有 Warehouse 和 Channel (避免 N+1)
│   ├─ 检测重复 SKU (全局去重)
│   │
│   └─ 对每个产品:
│       ├─ clean_base_fields()
│       │   ├─ 验证 weight >= 0
│       │   ├─ 生成 description_plaintext
│       │   └─ 自动生成 slug (如果未提供)
│       │
│       ├─ clean_attributes()
│       │   └─ AttributeAssignmentMixin.clean_input()
│       │
│       ├─ clean_media()
│       │   ├─ 本地文件: clean_image_file()
│       │   └─ 远程 URL: probe_media_url()
│       │
│       ├─ clean_product_channel_listings()
│       │   ├─ 验证 channel_id 存在
│       │   ├─ 检查重复 channel_id
│       │   └─ 验证发布状态与日期的一致性
│       │
│       └─ clean_variants()
│           └─ 委托给 ProductVariantBulkCreate.clean_variant()
│
├─ 2. create_products() - 构建实例
│   ├─ construct_instance() 构建 Product (不保存)
│   ├─ 构建 ProductVariant 实例 (不保存)
│   └─ 收集所有关联数据
│
├─ 3. 错误策略检查
│   ├─ REJECT_EVERYTHING: 有任何错误全部返回
│   └─ REJECT_FAILED_ROWS: 标记错误行为 None，继续处理
│
├─ 4. save() - 批量保存
│   ├─ Product.objects.bulk_create(products_to_create)
│   ├─ ProductMedia.objects.bulk_create(media_to_create)
│   ├─ ProductChannelListing.objects.bulk_create(listings_to_create)
│   ├─ AttributeAssignmentMixin.save()
│   └─ ProductVariantBulkCreate.save_variants()
│
├─ 5. _save_m2m() - 保存多对多关系
│   └─ CollectionProduct.objects.bulk_create()
│
└─ 6. post_save_actions() - 后置操作
    ├─ manager.product_created() - 触发 webhook
    ├─ manager.product_variant_created() - 触发 webhook
    └─ mark_active_catalogue_promotion_rules_as_dirty()
```

### 4.3 数据流向关系图

```
导出方向:
  GraphQL Input (ExportInfoInput)
       ↓
  ProductFieldEnum → HEADERS_TO_FIELDS_MAPPING
       ↓
  ORM Query Paths (category__slug, variants__sku, ...)
       ↓
  Database Query (annotate + values + distinct)
       ↓
  Relation Data (attributes, channels, warehouses)
       ↓
  petl Table → CSV/XLSX File
       ↓
  ExportFile.content_file → S3/本地存储
       ↓
  Notification (Email + Webhook)

导入方向:
  GraphQL Input (ProductBulkCreateInput[])
       ↓
  Validation (index_error_map)
       ↓
  Model Instance Construction (不保存)
       ↓
  Error Policy Check
       ↓
  bulk_create() 批量写入
       ↓
  M2M Relations bulk_create()
       ↓
  Webhook Events + Promotion Rules
```

---

## 五、关键文件索引

| 模块 | 文件路径 | 核心职责 |
|------|---------|---------|
| 字段映射 | `saleor/csv/utils/__init__.py` | `ProductExportFields` 定义 |
| 表头生成 | `saleor/csv/utils/product_headers.py` | 字段到表头的转换 |
| 数据处理 | `saleor/csv/utils/products_data.py` | 产品/变体关系数据提取 |
| 导出核心 | `saleor/csv/utils/export.py` | 批次处理、文件生成 |
| 任务调度 | `saleor/csv/tasks.py` | Celery 任务定义、错误回调 |
| 事件记录 | `saleor/csv/events.py` | 导出生命周期事件 |
| 通知系统 | `saleor/csv/notifications.py` | 成功/失败通知 |
| 数据模型 | `saleor/csv/models.py` | `ExportFile`, `ExportEvent` |
| 错误码 | `saleor/csv/error_codes.py` | `ExportErrorCode` |
| GraphQL 导出 | `saleor/graphql/csv/mutations/export_products.py` | 导出 mutation |
| GraphQL 导入 | `saleor/graphql/product/bulk_mutations/product_bulk_create.py` | 批量创建 mutation |
| 批次工具 | `saleor/core/utils/batches.py` | `queryset_in_batches` |
