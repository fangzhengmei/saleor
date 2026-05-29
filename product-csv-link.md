# 商品 CSV 导入导出代码链路深度分析

## ⚠️ 核心发现

**Saleor 的 `csv/` 模块只有导出功能，没有直接的 CSV 文件导入！**

| 方向 | 代码入口 | 输入格式 |
|------|---------|---------|
| 导出 | `saleor/csv/` 模块 | 数据库 → CSV/XLSX 文件 |
| 导入 | `saleor/graphql/product/bulk_mutations/product_bulk_create.py` | GraphQL JSON 数组 (不是 CSV 文件) |

**真实的"CSV 导入"工作流**：
```
用户导出 CSV → 编辑 CSV → 自行转换为 GraphQL JSON → 调用 productBulkCreate mutation
```

---

## 一、完整数据流总览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          导出方向 (Database → CSV)                       │
├─────────────────────────────────────────────────────────────────────────┤
│  GraphQL (ExportProducts)                                               │
│       ↓                                                                 │
│  Celery Task (export_products_task)                                     │
│       ↓                                                                 │
│  字段映射 (ProductExportFields) → ORM 查询路径                          │
│       ↓                                                                 │
│  批次处理 (BATCH_SIZE=1000) → queryset_in_batches                      │
│       ↓                                                                 │
│  数据处理 (get_products_data) → 关系数据补充                            │
│       ↓                                                                 │
│  petl 库 → 生成 CSV/XLSX 文件                                           │
│       ↓                                                                 │
│  ExportFile.content_file → 存储 + 通知                                  │
└─────────────────────────────────────────────────────────────────────────┘
                              ↕ 用户手动转换
┌─────────────────────────────────────────────────────────────────────────┐
│                          导入方向 (JSON → Database)                      │
├─────────────────────────────────────────────────────────────────────────┤
│  GraphQL (ProductBulkCreate)                                            │
│       ↓                                                                 │
│  输入验证 (clean_products) → index_error_map 逐行记录错误               │
│       ↓                                                                 │
│  错误策略检查 (REJECT_EVERYTHING / REJECT_FAILED_ROWS)                  │
│       ↓                                                                 │
│  实例构建 (不保存) → construct_instance                                  │
│       ↓                                                                 │
│  批量保存 → bulk_create() 分别写入多张表                                │
│       ↓                                                                 │
│  M2M 关系保存 → CollectionProduct.bulk_create()                         │
│       ↓                                                                 │
│  后置操作 → Webhook + 促销规则标记                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 二、字段映射：双向数据流串联

### 2.1 导出方向字段映射 (Database → CSV)

**文件**: `saleor/csv/utils/__init__.py:1-93`

```python
class ProductExportFields:
    HEADERS_TO_FIELDS_MAPPING = {
        "fields": {
            "id": "id",                                   # 商品 ID
            "name": "name",                               # 商品名称
            "description": "description_as_str",          # 商品描述
            "category": "category__slug",                 # 分类 slug
            "product type": "product_type__name",         # 商品类型名称
            "product weight": "product_weight",           # 商品重量 (带单位)
            "variant id": "variants__id",                 # 变体 ID
            "variant sku": "variants__sku",               # 变体 SKU
            "variant weight": "variant_weight",           # 变体重量
        },
        "product_many_to_many": {
            "collections": "collections__slug",           # 合集 slug (多值)
            "product media": "media__image",              # 商品图片 (多值)
        },
        "variant_many_to_many": {
            "variant media": "variants__media__image"     # 变体图片 (多值)
        }
    }
```

**表头生成流程**:
```
get_product_export_fields_and_headers_info(export_info)
    ├─ get_product_export_fields_and_headers()
    │   └─ ChainMap 合并 HEADERS_TO_FIELDS_MAPPING
    ├─ get_attributes_headers()
    │   └─ "slug-value (product attribute)"
    ├─ get_warehouses_headers()
    │   └─ "slug-value (warehouse quantity)"
    └─ get_channels_headers()
        └─ "slug-value (channel currency code)"
```

### 2.2 导入方向字段映射 (JSON → Database)

**文件**: `saleor/graphql/product/bulk_mutations/product_bulk_create.py:117-176`

```python
class ProductBulkCreateInput(ProductCreateInput):
    name = graphene.String()                           # → Product.name
    slug = graphene.String()                           # → Product.slug
    description = JSONString()                         # → Product.description
    category = graphene.ID()                           # → Product.category_id (需要 ID，不是 slug!)
    product_type = graphene.ID(required=True)          # → Product.product_type_id
    collections = NonNullList(graphene.ID)             # → CollectionProduct M2M
    attributes = NonNullList(AttributeValueInput)      # → 单独的属性赋值表
    media = NonNullList(MediaInput)                    # → ProductMedia
    channel_listings = NonNullList(ProductChannelListingCreateInput)
    variants = NonNullList(ProductVariantBulkCreateInput)
```

### 2.3 导出 ↔ 导入 字段对应表

| CSV 导出表头 | 导出 ORM 路径 | 导入 GraphQL 字段 | 注意事项 |
|-------------|--------------|------------------|---------|
| id | id | (自动生成) | 导入时不提供，系统生成 |
| name | name | name | 直接对应 |
| description | description_as_str | description | 导出是纯文本，导入需要 EditorJS JSON |
| category | category__slug | category | ⚠️ 导出是 slug，导入需要 ID！需自行转换 |
| product type | product_type__name | productType | ⚠️ 导出是名称，导入需要 ID！需自行转换 |
| product weight | product_weight | weight | 导出带单位 ("100 g")，导入是数字 |
| collections | collections__slug | collections | ⚠️ 导出是 slug 列表，导入需要 ID 列表！ |
| product media | media__image | media | ⚠️ 导出是 URL，导入需要 media_url 或本地文件 |
| variant id | variants__id | variants[].id | 导入时不提供 |
| variant sku | variants__sku | variants[].sku | 直接对应 |
| variant weight | variants__weight | variants[].weight | 同上，单位问题 |

**关键不匹配点**：导出的是人类可读的 slug/名称，导入需要的是数据库 ID！用户必须自行进行 ID 转换。

---

## 三、批次处理：双向机制对比

### 3.1 导出批次处理 (Database → CSV)

**文件**: `saleor/csv/utils/export.py:192-223`

```python
BATCH_SIZE = 1000

def export_products_in_batches(queryset, ...):
    for batch_pks in queryset_in_batches(queryset, BATCH_SIZE):
        # 1. 按主键批量预取关联数据
        product_batch = Product.objects.filter(pk__in=batch_pks).prefetch_related(
            "attributevalues", "variants", "collections",
            "media", "product_type", "category"
        )
        # 2. 处理当前批次数据
        export_data = get_products_data(product_batch, ...)
        # 3. 追加写入文件
        append_to_file(export_data, ...)
```

**批次工具**: `saleor/core/utils/batches.py:4-17`

```python
def queryset_in_batches(queryset: QuerySet, batch_size: int):
    start_pk = 0
    while True:
        qs = queryset.filter(pk__gt=start_pk).order_by("pk")[:batch_size]
        pks = list(qs.values_list("pk", flat=True))
        if not pks:
            break
        yield pks
        start_pk = pks[-1]  # 游标式分页，避免 offset 性能问题
```

**导出批次特性**:
- ✅ 使用 `pk__gt` 游标式分页，性能稳定
- ✅ 每批独立预取关联数据，避免 N+1
- ✅ 每批直接写入文件，内存占用低
- ❌ 没有事务，失败后需重新导出

### 3.2 导入批次处理 (JSON → Database)

**文件**: `saleor/graphql/product/bulk_mutations/product_bulk_create.py:938-972`

```python
@traced_atomic_transaction()  # 整个操作在单个事务中！
def perform_mutation(cls, root, info, **data):
    index_error_map: dict = defaultdict(list)
    
    # ── 阶段 1: 全量验证 (不保存) ──
    cleaned_inputs_map = cls.clean_products(
        info, data["products"], index_error_map
    )
    
    # ── 阶段 2: 构建实例 (不保存) ──
    instances_data_with_errors_list = cls.create_products(
        info, cleaned_inputs_map, index_error_map
    )
    
    # ── 阶段 3: 错误策略检查 ──
    if any(index_error_map.values()):
        if error_policy == "REJECT_EVERYTHING":
            # 全部回滚，返回所有错误
            return ProductBulkCreate(count=0, results=get_results(..., True))
        if error_policy == "REJECT_FAILED_ROWS":
            # 标记错误行为 None，只保存正确的
            for data in instances_data_with_errors_list:
                if data["errors"] and data["instance"]:
                    data["instance"] = None
    
    # ── 阶段 4: 批量写入数据库 ──
    variants, updated_channels = cls.save(
        info, instances_data_with_errors_list
    )
    
    # ── 阶段 5: 保存多对多关系 ──
    cls._save_m2m(info, instances_data_with_errors_list)
    
    # ── 阶段 6: 后置操作 ──
    cls.post_save_actions(info, products, variants, updated_channels)
```

**save() 方法的批量写入顺序**:
```python
@classmethod
def save(cls, info, product_data_with_errors_list):
    # 1. 批量写入 Product 主表
    models.Product.objects.bulk_create(products_to_create)
    
    # 2. 批量写入 ProductMedia
    models.ProductMedia.objects.bulk_create(media_to_create)
    
    # 3. 批量写入 ProductChannelListing
    models.ProductChannelListing.objects.bulk_create(listings_to_create)
    
    # 4. 逐个保存属性 (不能 bulk_create，因为要处理赋值)
    for product, attributes in attributes_to_save:
        AttributeAssignmentMixin.save(product, attributes)
    
    # 5. 批量写入变体 (委托给 ProductVariantBulkCreate)
    if variants_input_data:
        variants = cls.save_variants(info, variants_input_data)
```

**导入批次特性**:
- ✅ 先全量验证，再批量写入，减少数据库 IO
- ✅ 整个操作在 `@traced_atomic_transaction()` 事务中
- ✅ 支持两种错误处理策略
- ❌ 没有分片，所有数据一次性加载到内存

---

## 四、失败反馈：完整链路串联

### 4.1 导出失败反馈链路

**文件**: `saleor/csv/tasks.py:20-57`

```
Celery Task Exception
    ↓
ExportTask.on_failure() 回调
    ├─ 1. 更新 ExportFile 状态
    │   ├─ content_file = None
    │   ├─ status = JobStatus.FAILED
    │   └─ save()
    │
    ├─ 2. 记录错误事件 (审计日志)
    │   └─ events.export_failed_event()
    │       └─ ExportEvent.objects.create(
    │              type=EXPORT_FAILED,
    │              parameters={"message": str(exc), "error_type": str(einfo.type)}
    │          )
    │
    └─ 3. 发送通知
        └─ send_export_failed_info()
            ├─ manager.notify(CSV_EXPORT_FAILED)  # 邮件通知
            └─ manager.product_export_completed()  # Webhook
```

**导出错误码** (`saleor/csv/error_codes.py`):
```python
class ExportErrorCode(Enum):
    GRAPHQL_ERROR = "graphql_error"    # GraphQL 层面错误
    INVALID = "invalid"                # 参数无效
    NOT_FOUND = "not_found"            # 资源不存在
    REQUIRED = "required"              # 缺少必填字段
```

### 4.2 导入失败反馈链路

**文件**: `saleor/graphql/product/bulk_mutations/product_bulk_create.py:264-283`

**核心数据结构**: `index_error_map`

```python
index_error_map = defaultdict(list)
# 结构:
{
    0: [  # 第 0 个产品的错误
        ProductBulkCreateError(
            path="weight",           # 错误字段路径 (camelCase)
            message="Product can't have negative weight.",
            code="INVALID",
        ),
        ProductBulkCreateError(
            path="variants.0.sku",   # 嵌套路径: 第 0 个变体的 sku
            message="SKU already exists.",
            code="DUPLICATED",
            values=["DUPLICATE_SKU_123"],
        ),
    ],
    2: [  # 第 2 个产品的错误
        ...
    ]
}
```

**错误收集点分布**:

| 阶段 | 函数 | 错误类型 |
|------|------|---------|
| 基础字段验证 | `clean_base_fields()` | weight < 0 |
| 属性验证 | `clean_attributes()` | AttributeAssignmentMixin 错误 |
| 媒体验证 | `clean_media()` | 图片格式/URL 无效 |
| 渠道列表验证 | `clean_product_channel_listings()` | channel_id 不存在、重复 |
| 变体验证 | `clean_variants()` | 委托给 ProductVariantBulkCreate |
| 实例构建 | `create_products()` | full_clean() 验证失败 |

**错误策略执行**:
```python
# 文件: saleor/graphql/product/bulk_mutations/product_bulk_create.py:949-958

if any(index_error_map.values()):
    # 策略 1: 有任何错误，全部拒绝
    if error_policy == ErrorPolicyEnum.REJECT_EVERYTHING.value:
        results = get_results(instances_data_with_errors_list, True)
        return ProductBulkCreate(count=0, results=results)  # count=0！
    
    # 策略 2: 只拒绝错误行，继续保存正确的
    if error_policy == ErrorPolicyEnum.REJECT_FAILED_ROWS.value:
        for data in instances_data_with_errors_list:
            if data["errors"] and data["instance"]:
                data["instance"] = None  # 标记为不保存
```

### 4.3 导入错误响应格式

```graphql
mutation {
  productBulkCreate(
    products: [...],
    errorPolicy: REJECT_FAILED_ROWS
  ) {
    count          # 成功创建的数量
    results {      # 每个产品的结果 (与输入顺序一致)
      product {    # 成功创建的产品对象，失败则为 null
        id
        name
      }
      errors {     # 该产品的错误列表
        path       # 如 "variants.0.sku"
        message    # 人类可读消息
        code       # 错误码枚举
        values     # 相关值 (如重复的 SKU)
      }
    }
  }
}
```

---

## 五、完整代码链路：从导出到导入

### 5.1 导出完整执行路径

```
GraphQL Mutation: ExportProducts.perform_mutation()
│   文件: saleor/graphql/csv/mutations/export_products.py:109-132
│
├─ get_scope(input, Product)
│   ├─ ALL: {"all": ""}
│   ├─ IDS: 解析 global_id → 转为 pk
│   └─ FILTER: 传递 filter 参数
│
├─ get_export_info(export_info_input)
│   ├─ fields: [ProductFieldEnum.NAME, ...]
│   ├─ attributes: [attribute_pk, ...]  (global_id → pk)
│   ├─ warehouses: [warehouse_pk, ...]
│   └─ channels: [channel_pk, ...]
│
├─ add_channel_to_filter_scope()  # 渠道相关过滤器验证
│
├─ 创建 ExportFile 记录
│   export_file = ExportFile.objects.create(app=..., user=...)
│
├─ 记录启动事件
│   events.export_started_event(export_file=...)
│
└─ 提交异步任务
    export_products_task.delay(export_file.pk, scope, export_info, file_type)
    
    ↓ (Celery Worker 异步执行)
    
Celery Task: export_products_task()
│   文件: saleor/csv/tasks.py:60-74
│
└─ export_products()
    │   文件: saleor/csv/utils/export.py:29-61
    │
    ├─ get_queryset()  # 使用 replica 数据库
    │   ├─ ids 过滤: filter(pk__in=scope["ids"])
    │   └─ filter 过滤: ProductFilter(data=..., queryset=...).qs
    │
    ├─ get_product_export_fields_and_headers_info()
    │   └─ 字段映射 → 表头生成
    │
    ├─ create_file_with_headers()
    │   └─ NamedTemporaryFile + petl.tocsv()
    │
    ├─ export_products_in_batches()
    │   └─ BATCH_SIZE = 1000
    │       ├─ queryset_in_batches()
    │       ├─ get_products_data()
    │       └─ append_to_file()
    │
    ├─ save_csv_file_in_export_file()
    │   └─ export_file.content_file.save(file_name, temp_file)
    │
    └─ send_export_download_link_notification()
        ├─ CSV_EXPORT_SUCCESS 通知
        └─ product_export_completed webhook
```

### 5.2 导入完整执行路径

```
GraphQL Mutation: ProductBulkCreate.perform_mutation()
│   文件: saleor/graphql/product/bulk_mutations/product_bulk_create.py:938-972
│   装饰器: @traced_atomic_transaction()
│
├─ 初始化错误映射
│   index_error_map = defaultdict(list)
│
├─ 阶段 1: 全量数据清洗验证
│   clean_products(info, products_data, index_error_map)
│   │
│   ├─ 预加载所有 Warehouse 和 Channel (避免 N+1)
│   ├─ 全局检测重复 SKU
│   │
│   └─ 对每个产品 (按索引):
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
│       │   └─ 验证发布状态与日期一致性
│       │
│       └─ clean_variants()
│           └─ 委托给 ProductVariantBulkCreate.clean_variant()
│
├─ 阶段 2: 构建模型实例 (不保存)
│   create_products(info, cleaned_inputs_map, index_error_map)
│   │
│   ├─ construct_instance() 构建 Product 实例
│   ├─ 构建 ProductVariant 实例 (不保存)
│   └─ 收集所有关联数据
│
├─ 阶段 3: 错误策略检查
│   if any(index_error_map.values()):
│       ├─ REJECT_EVERYTHING: 返回 count=0，所有错误
│       └─ REJECT_FAILED_ROWS: 标记错误行为 None
│
├─ 阶段 4: 批量写入数据库
│   save(info, instances_data_with_errors_list)
│   │
│   ├─ Product.objects.bulk_create(products_to_create)
│   ├─ ProductMedia.objects.bulk_create(media_to_create)
│   ├─ ProductChannelListing.objects.bulk_create(listings_to_create)
│   ├─ AttributeAssignmentMixin.save()  # 逐个保存
│   └─ ProductVariantBulkCreate.save_variants()
│
├─ 阶段 5: 保存多对多关系
│   _save_m2m(info, instances_data)
│   └─ CollectionProduct.objects.bulk_create()
│
└─ 阶段 6: 后置操作
    post_save_actions(info, products, variants, channels)
    ├─ manager.product_created() webhook
    ├─ manager.product_variant_created() webhook
    └─ mark_active_catalogue_promotion_rules_as_dirty()
```

---

## 六、关键文件索引

| 模块 | 文件路径 | 核心职责 |
|------|---------|---------|
| 字段映射定义 | `saleor/csv/utils/__init__.py` | `ProductExportFields` 导出字段映射 |
| 表头生成 | `saleor/csv/utils/product_headers.py` | 导出字段到 CSV 表头转换 |
| 导出数据处理 | `saleor/csv/utils/products_data.py` | 产品/变体关系数据提取 |
| 导出核心逻辑 | `saleor/csv/utils/export.py` | 批次处理、petl 文件生成 |
| 导出任务调度 | `saleor/csv/tasks.py` | Celery 任务、错误回调 |
| 导出事件记录 | `saleor/csv/events.py` | 导出生命周期事件审计 |
| 导出通知 | `saleor/csv/notifications.py` | 成功/失败邮件 + Webhook |
| 导入核心逻辑 | `saleor/graphql/product/bulk_mutations/product_bulk_create.py` | 批量创建 mutation |
| 变体批量导入 | `saleor/graphql/product/bulk_mutations/product_variant_bulk_create.py` | 变体批量创建 |
| 批次工具 | `saleor/core/utils/batches.py` | `queryset_in_batches` 游标分页 |
| 导出错误码 | `saleor/csv/error_codes.py` | `ExportErrorCode` 枚举 |
| 导入错误码 | `saleor/product/error_codes.py` | `ProductBulkCreateErrorCode` 枚举 |

---

## 七、设计洞察

1. **不对称设计**：导出有完整的 CSV 处理，导入只有 GraphQL JSON 接口
   - 可能原因：CSV 解析需要处理太多边缘情况（格式、编码、数据清洗）
   - 留给用户/前端处理：导出 CSV → Excel 编辑 → 自定义脚本转 GraphQL

2. **导出优化为读性能**：
   - 使用 replica 数据库读取
   - 游标式分页避免大 offset
   - 分批写入文件降低内存

3. **导入优化为数据一致性**：
   - 全量验证后再写入
   - 整个操作在事务中
   - 灵活的错误策略支持

4. **错误反馈分层**：
   - 导出：Celery 回调 → 事件日志 → 通知
   - 导入：`index_error_map` 逐行追踪 → 两种错误策略 → 结构化响应
