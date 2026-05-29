# 商品 CSV 导入导出代码链路深度分析

---

## ⚠️ 核心发现（已验证代码事实）

### 代码证据：Saleor csv 模块只有导出，没有导入！

| 验证项 | 代码位置 | 事实 |
|--------|---------|------|
| Mutation 注册 | `saleor/graphql/csv/schema.py:45-54` | `CsvMutations` 只注册了 `export_products`, `export_gift_cards`, `export_voucher_codes` |
| Mutation 导出列表 | `saleor/graphql/csv/mutations/__init__.py:1-5` | 只导出了 3 个 Export* 类，没有 Import* 类 |
| petl 库使用 | `saleor/csv/utils/export.py:6,184,269` | 只有 `etl.tocsv()`, `etl.io.csv.appendcsv()`，没有 `fromcsv`/`read_csv` |
| 事件类型 | `saleor/csv/__init__.py:1-24` | `ExportEvents` 只有 `EXPORT_PENDING`, `EXPORT_SUCCESS`, `EXPORT_FAILED` 等，没有 IMPORT_* |
| 数据模型 | `saleor/csv/models.py:12-36` | `ExportFile` 模型只有 `content_file` 存储导出文件，没有导入文件字段 |
| 全局搜索 | 全代码库 | 没有找到 `parse_csv`, `DictReader`, `fromcsv`, `read_csv` 等 CSV 解析代码 |

### 推断说明（无直接代码，基于架构的逻辑推断）

> 以下内容基于代码架构的合理推断，没有直接代码证据：

1. **真实的"CSV 导入"工作流**：
   ```
   用户导出 CSV → 编辑 CSV → 自行转换为 GraphQL JSON → 调用 productBulkCreate mutation
   ```

2. **为什么没有 CSV 文件导入？**（推断）
   - CSV 解析需要处理太多边缘情况（格式、编码、数据清洗、空值处理）
   - 字段映射不匹配（导出是 slug/名称，导入需要 ID）
   - 留给用户/前端/外部系统处理：导出 CSV → Excel 编辑 → 自定义脚本转 GraphQL

3. **用户需要自行处理的转换**：
   - slug → ID 转换（分类、商品类型、合集等）
   - 带单位的字符串解析（如 `"100 g"` → 数字 100 + 单位）
   - 纯文本描述转 EditorJS JSON 格式
   - 多值字段的分隔符处理（如多个合集 slug）

---

## 一、完整数据流总览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          导出方向 (Database → CSV)                       │
├─────────────────────────────────────────────────────────────────────────┤
│  GraphQL (ExportProducts)                                               │
│       ↓  [对象: ExportProductsInput]                                    │
│  Celery Task (export_products_task)                                     │
│       ↓  [对象: scope(dict), export_info(dict)]                         │
│  字段映射 (ProductExportFields) → ORM 查询路径                          │
│       ↓  [对象: export_fields(list), file_headers(list)]                │
│  批次处理 (BATCH_SIZE=1000) → queryset_in_batches                      │
│       ↓  [对象: batch_pks(list)]                                        │
│  数据处理 (get_products_data) → 关系数据补充                            │
│       ↓  [对象: products_with_variants_data(list[dict])]                │
│  petl 库 → 生成 CSV/XLSX 文件                                           │
│       ↓  [对象: NamedTemporaryFile]                                     │
│  ExportFile.content_file → 存储 + 通知                                  │
└─────────────────────────────────────────────────────────────────────────┘
                              ↕ 用户手动转换（推断）
┌─────────────────────────────────────────────────────────────────────────┐
│                          导入方向 (JSON → Database)                      │
├─────────────────────────────────────────────────────────────────────────┤
│  GraphQL (ProductBulkCreate)                                            │
│       ↓  [对象: ProductBulkCreateInput[]]                               │
│  输入验证 (clean_products) → index_error_map 逐行记录错误               │
│       ↓  [对象: index_error_map(defaultdict(list))]                     │
│  错误策略检查 (REJECT_EVERYTHING / REJECT_FAILED_ROWS)                  │
│       ↓  [对象: instances_data_with_errors_list(list)]                  │
│  实例构建 (不保存) → construct_instance                                  │
│       ↓  [对象: Product 实例 (未保存)]                                   │
│  批量保存 → bulk_create() 分别写入多张表                                │
│       ↓  [对象: Product[], ProductMedia[], ...]                         │
│  M2M 关系保存 → CollectionProduct.bulk_create()                         │
│       ↓  [对象: CollectionProduct[]]                                    │
│  后置操作 → Webhook + 促销规则标记                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 二、字段映射：双向数据流串联（已校转变体重量字段）

### 2.1 变体重量字段完整映射路径（已校对）

#### 导出方向完整链路 (Database → CSV)

```
ProductFieldEnum.VARIANT_WEIGHT = "variant weight"
    │  文件: saleor/graphql/csv/enums.py:46
    ↓
HEADERS_TO_FIELDS_MAPPING["fields"]["variant weight"] = "variant_weight"
    │  文件: saleor/csv/utils/__init__.py:17
    ↓  ⚠️  "variant_weight" 不是真实数据库字段！是 annotate 生成的计算字段
get_products_data() → annotate()
    │  文件: saleor/csv/utils/products_data.py:60-67
    ↓
    variant_weight=Case(
        When(
            variants__weight__isnull=False,           # 读取真实字段: variants__weight
            then=Concat("variants__weight", V(" g")),  # 拼接单位: "100 g"
        ),
        default=V(""),
        output_field=CharField(),
    )
    │
    ↓  values(*product_export_fields)
    ↓  distinct("pk", "variants__pk")
    ↓  [data: {"variant_weight": "100 g", ...}]
    ↓
petl.fromdicts() → etl.io.csv.appendcsv()
    │
    ↓
CSV 输出: "100 g" (字符串，带单位)
```

#### 导入方向完整链路 (CSV → Database)

```
用户手动解析: "100 g" → 提取数字 100 + 单位 "g"
    │  ⚠️  无代码支持，需用户自行处理
    ↓
WeightScalar.parse_value()
    │  文件: saleor/graphql/core/scalars.py:95-126
    ├─ 如果是 {"unit": "g", "value": 100} → Weight(g=100)
    └─ 如果是 100 → 使用默认单位 → Weight(g=100)
    ↓
clean_base_fields() → 验证 weight.value >= 0
    │  文件: saleor/graphql/product/bulk_mutations/product_bulk_create.py:239-248
    ↓
construct_instance() → ProductVariant.weight = Weight(g=100)
    │
    ↓
ProductVariant.objects.bulk_create()
    │
    ↓
数据库存储: MeasurementField(measurement=Weight)
    │  文件: saleor/product/models.py:132-136
    ↓
真实字段: ProductVariant.weight (Weight 对象，含 value 和 unit)
```

### 2.2 导出方向字段映射 (Database → CSV)

**文件**: `saleor/csv/utils/__init__.py:1-93`

```python
class ProductExportFields:
    HEADERS_TO_FIELDS_MAPPING = {
        "fields": {
            "id": "id",                                   # 商品 ID
            "name": "name",                               # 商品名称
            "description": "description_as_str",          # 商品描述 (annotate 生成)
            "category": "category__slug",                 # 分类 slug
            "product type": "product_type__name",         # 商品类型名称
            "product weight": "product_weight",           # 商品重量 (annotate 生成，带单位)
            "variant id": "variants__id",                 # 变体 ID
            "variant sku": "variants__sku",               # 变体 SKU
            "variant weight": "variant_weight",           # 变体重量 (annotate 生成，带单位)
            "variant is preorder": "variants__is_preorder",
            "variant preorder global threshold": "variants__preorder_global_threshold",
            "variant preorder end date": "variants__preorder_end_date",
        },
        "product_many_to_many": {
            "collections": "collections__slug",           # 合集 slug (多值，逗号分隔)
            "product media": "media__image",              # 商品图片 URL (多值)
        },
        "variant_many_to_many": {
            "variant media": "variants__media__image"     # 变体图片 URL (多值)
        }
    }
```

**动态表头生成流程** (`saleor/csv/utils/product_headers.py:13-31`):
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

### 2.3 导入方向字段映射 (JSON → Database)

**文件**: `saleor/graphql/product/bulk_mutations/product_bulk_create.py:117-176`

```python
class ProductBulkCreateInput(ProductCreateInput):
    name = graphene.String()                           # → Product.name
    slug = graphene.String()                           # → Product.slug
    description = JSONString()                         # → Product.description (EditorJS JSON)
    category = graphene.ID()                           # → Product.category_id (需要 ID，不是 slug!)
    product_type = graphene.ID(required=True)          # → Product.product_type_id (需要 ID!)
    collections = NonNullList(graphene.ID)             # → CollectionProduct M2M (需要 ID 列表!)
    weight = WeightScalar()                            # → Product.weight (Weight 对象)
    attributes = NonNullList(AttributeValueInput)      # → 单独的属性赋值表
    media = NonNullList(MediaInput)                    # → ProductMedia
    channel_listings = NonNullList(ProductChannelListingCreateInput)
    variants = NonNullList(ProductVariantBulkCreateInput)
```

### 2.4 导出 ↔ 导入 字段对应表（完整校对，含 variant media 校准）

| CSV 导出表头 | 导出 ORM 路径 | 导入 GraphQL 字段 | 存在性 | 类型差异 | 转换需求 |
|-------------|--------------|------------------|--------|---------|---------|
| id | `id` | (自动生成) | - | - | 导入时不提供 |
| name | `name` | `name` | ✅ 存在 | 字符串 → 字符串 | ✅ 直接对应 |
| description | `description_as_str` | `description` | ✅ 存在 | 纯文本 → EditorJS JSON | ⚠️ 需转换格式 |
| category | `category__slug` | `category` | ✅ 存在 | slug → ID | ⚠️ 需查询转换 |
| product type | `product_type__name` | `productType` | ✅ 存在 | 名称 → ID | ⚠️ 需查询转换 |
| product weight | `product_weight` | `weight` | ✅ 存在 | 字符串 "100 g" → Weight 对象 | ⚠️ 需解析单位 |
| collections | `collections__slug` | `collections` | ✅ 存在 | slug 列表 → ID 列表 | ⚠️ 需查询+拆分 |
| product media | `media__image` | `media` | ✅ 存在 | URL 字符串 → MediaInput | ⚠️ 需构造对象 |
| variant id | `variants__id` | `variants[].id` | - | - | 导入时不提供 |
| variant sku | `variants__sku` | `variants[].sku` | ✅ 存在 | 字符串 → 字符串 | ✅ 直接对应 |
| variant weight | `variant_weight` | `variants[].weight` | ✅ 存在 | 字符串 "100 g" → Weight 对象 | ⚠️ 需解析单位 |
| **variant media** | `variants__media__image` | **`variants[].media` (❌ 不存在)** | ❌ 不存在 | - | **见下方替代方案** |

### 2.5 Variant Media 字段：不存在时的处理方式与替代映射（代码证据）

#### 关键发现：Variant Media 在导入侧**没有直接字段**

**代码证据**：
| 验证项 | 代码位置 | 事实 |
|--------|---------|------|
| ProductVariantInput 定义 | `saleor/graphql/product/mutations/product_variant/product_variant_create.py:48-83` | `ProductVariantInput` 包含 `attributes`, `sku`, `name`, `weight`, `preorder` 等，但**没有 `media` 字段** |
| ProductVariantBulkCreateInput 定义 | `saleor/graphql/product/bulk_mutations/product_variant_bulk_create.py:195-215` | 继承自 `ProductVariantInput`，新增 `stocks`, `channel_listings`，但**仍然没有 `media` 字段** |
| ProductBulkCreate 的 variants 字段 | `saleor/graphql/product/bulk_mutations/product_bulk_create.py:195-215` | `variants` 是 `NonNullList(ProductVariantBulkCreateInput)`，继承了同样的字段限制 |
| VariantMedia 分配方式 | `saleor/graphql/product/mutations/product_variant/variant_media_assign.py:17-74` | 需要单独调用 `variantMediaAssign` mutation，传入 `media_id` 和 `variant_id` |

#### Variant Media 的替代映射方案（两步走）

**方案：先创建 Product + Media，再单独分配给 Variant**

```
步骤 1: 在 productBulkCreate 中创建 Product 和 ProductMedia（共享媒体池）
    ├─ products[].media[] = [
    │     {"media_url": "https://example.com/img1.jpg", "alt": "Image 1"},
    │     {"media_url": "https://example.com/img2.jpg", "alt": "Image 2"}
    │   ]
    └─ 返回: products[].media[].id (如 "UHJvZHVjdE1lZGlhOjE=")

步骤 2: 调用 variantMediaAssign mutation 分配给特定 Variant
    mutation {
      variantMediaAssign(
        mediaId: "UHJvZHVjdE1lZGlhOjE=",  # 步骤 1 返回的 media ID
        variantId: "UHJvZHVjdFZhcmlhbnQ6MQ=="  # 步骤 1 返回的 variant ID
      ) {
        productVariant { id }
        media { id }
      }
    }
```

**注意**：
- 媒体必须先属于产品（通过 `product.media` 创建），才能分配给变体
- 可以批量创建产品后，再循环调用 `variantMediaAssign` 为每个变体分配媒体
- 如果媒体已分配给变体，会报错 "This media is already assigned"

**关键不匹配点总结**（代码证据）：
1. **slug/名称 vs ID**：导出的是人类可读的 slug 或名称，导入需要数据库 ID
   - 代码证据：`HEADERS_TO_FIELDS_MAPPING` 使用 `__slug` 或 `__name`，而 `ProductBulkCreateInput` 使用 `graphene.ID()`

2. **字符串带单位 vs Weight 对象**：导出是 `"100 g"` 字符串，导入需要 `WeightScalar`（数字或 `{unit, value}` 对象）
   - 代码证据：`annotate` 用 `Concat("variants__weight", V(" g"))` 生成字符串，`WeightScalar.parse_value()` 需要数字或字典

3. **纯文本 vs EditorJS JSON**：导出是纯文本，导入需要 `JSONString` 类型的 EditorJS 格式
   - 代码证据：导出用 `Cast("description", CharField())`，导入用 `JSONString(description=...)`

4. **Variant Media 无直接导入字段**：导出有 `variants__media__image`，导入 `ProductVariantBulkCreateInput` 没有 media 字段
   - 代码证据：`ProductVariantInput` 定义中没有 `media` 字段，需通过 `variantMediaAssign` 单独分配

---

### 2.6 从导出 CSV 到 productBulkCreate 输入的最小转换规则

#### 前置准备（必须先做）

1. **建立 ID 映射表**：
   - 查询所有 Category: `{ slug: "electronics", id: "Q2F0ZWdvcnk6MQ==" }`
   - 查询所有 ProductType: `{ name: "Physical", id: "UHJvZHVjdFR5cGU6MQ==" }`
   - 查询所有 Collection: `{ slug: "summer", id: "Q29sbGVjdGlvbjox" }`
   - 查询所有 Channel: `{ slug: "default", id: "Q2hhbm5lbDox" }`
   - 查询所有 Warehouse: `{ slug: "main", id: "V2FyZWhvdXNlOjE=" }`

2. **解析 CSV 行**：注意导出是**每变体一行**（同一产品可能有多行，每个变体一行）

#### 最小转换规则表（可落地代码）

| CSV 列名 | 转换规则 | 示例输入 → 输出 |
|---------|---------|----------------|
| **必填字段** | | |
| name | 直接使用 | `"T-Shirt"` → `"T-Shirt"` |
| product type | 用 name → product_type_id 映射表查询 | `"Physical"` → `"UHJvZHVjdFR5cGU6MQ=="` |
| variants[].sku | 直接使用 | `"TS-001-S"` → `"TS-001-S"` |
| variants[].attributes | 从属性列（如 "size (variant attribute)"）解析 | "S" → `[{"id": "QXR0cmlidXRlOjE=", "values": [{"name": "S"}]}]` |
| **推荐字段** | | |
| slug | 自动生成或留空（会自动从 name 生成） | `"t-shirt"` → `"t-shirt"` |
| category | 用 slug → category_id 映射表查询 | `"apparel"` → `"Q2F0ZWdvcnk6MQ=="` |
| product weight | 正则解析 `"(\d+) (\w+)"` → 构造 Weight 对象 | `"100 g"` → `{"unit": "g", "value": 100}` |
| collections | 按逗号拆分 slug → 查映射表得 ID 列表 | `"summer,sale"` → `["Q29sbGVjdGlvbjox", "Q29sbGVjdGlvbjoy"]` |
| media | 按逗号拆分 URL → 构造 MediaInput 数组 | `"https://a.jpg,https://b.jpg"` → `[{"media_url": "https://a.jpg", "alt": ""}, ...]` |
| description | 包装成 EditorJS 格式 | `"A great t-shirt"` → `'{"blocks": [{"type": "paragraph", "data": {"text": "A great t-shirt"}}]}'` |
| **变体字段** | | |
| variants[].weight | 正则解析 → Weight 对象 | `"50 g"` → `{"unit": "g", "value": 50}` |
| **variant media（特殊处理）** | | |
| variant media | 第一步：先放到 product.media 共享池<br>第二步：创建成功后调用 variantMediaAssign | 见下方示例 |
| **渠道字段** | | |
| channel_listings | 从渠道列解析 → 构造 channel_listing 对象 | (见下方示例) |
| variant channel_listings | 从变体渠道列解析 → 构造价格对象 | (见下方示例) |
| **库存字段** | | |
| variants[].stocks | 从仓库列解析 → 构造 stock 对象 | "main: 100" → `[{"warehouse": "V2FyZWhvdXNlOjE=", "quantity": 100}]` |

#### 最小可工作的转换示例

**CSV 输入行**：
```csv
name,product type,category,product weight,variant sku,variant weight,size (variant attribute)
T-Shirt,Physical,apparel,100 g,TS-001-S,50 g,S
```

**转换后的 GraphQL 输入**：
```graphql
mutation {
  productBulkCreate(
    products: [{
      name: "T-Shirt",
      productType: "UHJvZHVjdFR5cGU6MQ==",  # "Physical" 对应的 ID
      category: "Q2F0ZWdvcnk6MQ==",           # "apparel" 对应的 ID
      weight: { unit: "g", value: 100 },
      variants: [{
        sku: "TS-001-S",
        weight: { unit: "g", value: 50 },
        attributes: [{
          id: "QXR0cmlidXRlOjE=",            # "size" 属性的 ID
          values: [{ name: "S" }]
        }]
      }]
    }],
    errorPolicy: REJECT_FAILED_ROWS
  ) {
    count
    results {
      product { id }
      errors { path message code }
    }
  }
}
```

#### Variant Media 完整转换流程

**CSV 输入**：
```csv
name,variant sku,product media,variant media
T-Shirt,TS-001,"https://a.jpg","https://a.jpg,https://b.jpg"
```

**转换步骤**：
```
步骤 1: 收集所有媒体到 product.media（去重）
  product.media = [
    { media_url: "https://a.jpg", alt: "Product image" },
    { media_url: "https://b.jpg", alt: "Variant image" }
  ]

步骤 2: 执行 productBulkCreate，记录返回的 media ID 和 variant ID
  返回:
    product.media[0].id = "UHJvZHVjdE1lZGlhOjE="  (a.jpg)
    product.media[1].id = "UHJvZHVjdE1lZGlhOjI="  (b.jpg)
    product.variants[0].id = "UHJvZHVjdFZhcmlhbnQ6MQ=="  (TS-001)

步骤 3: 为变体分配媒体（循环调用 variantMediaAssign）
  为 TS-001 分配 a.jpg:
    mutation { variantMediaAssign(mediaId: "UHJvZHVjdE1lZGlhOjE=", variantId: "UHJvZHVjdFZhcmlhbnQ6MQ==") { ... } }
  为 TS-001 分配 b.jpg:
    mutation { variantMediaAssign(mediaId: "UHJvZHVjdE1lZGlhOjI=", variantId: "UHJvZHVjdFZhcmlhbnQ6MQ==") { ... } }
```

---

## 三、对象流串联：从字段映射到失败反馈边界

### 3.1 导出方向完整对象流

```
┌─────────────────────────────────────────────────────────────────────────┐
│  阶段 1: GraphQL 入口 → 任务提交                                         │
├─────────────────────────────────────────────────────────────────────────┤
│  ExportProducts.perform_mutation()                                       │
│  文件: saleor/graphql/csv/mutations/export_products.py:109-132           │
│                                                                          │
│  输入对象:                                                               │
│  {                                                                       │
│    scope: "all" | {"ids": [...]} | {"filter": {...}},                    │
│    export_info: {                                                        │
│      fields: [ProductFieldEnum.NAME, ProductFieldEnum.VARIANT_WEIGHT],    │
│      attributes: ["attr_id_1", ...],                                     │
│      warehouses: ["warehouse_id_1", ...],                                │
│      channels: ["channel_id_1", ...]                                     │
│    },                                                                    │
│    file_type: "csv" | "xlsx"                                             │
│  }                                                                       │
│                                                                          │
│  输出对象:                                                               │
│  ExportFile(                                                             │
│    id=123,                                                               │
│    status=JobStatus.PENDING,                                             │
│    user=User(...),                                                       │
│    app=None                                                              │
│  )                                                                       │
│                                                                          │
│  异步任务参数:                                                           │
│  export_products_task.delay(                                             │
│    export_file_id=123,                                                   │
│    scope={"all": ""},                                                    │
│    export_info={"fields": [...], ...},                                   │
│    file_type="csv"                                                       │
│  )                                                                       │
└─────────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  阶段 2: 字段映射 → 表头生成                                             │
├─────────────────────────────────────────────────────────────────────────┤
│  get_product_export_fields_and_headers_info(export_info)                 │
│  文件: saleor/csv/utils/product_headers.py:13-31                         │
│                                                                          │
│  输入对象: export_info (dict)                                            │
│  输出对象:                                                               │
│  (                                                                       │
│    export_fields: ["id", "name", "variant_weight", ...],  # ORM 查询字段 │
│    file_headers: ["id", "name", "variant weight", ...],   # CSV 表头     │
│    data_headers: ["id", "name", "variant_weight", ...]    # 数据键名     │
│  )                                                                       │
│                                                                          │
│  中间对象: ProductExportFields.HEADERS_TO_FIELDS_MAPPING (dict)          │
└─────────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  阶段 3: 批次处理 → 数据生成                                             │
├─────────────────────────────────────────────────────────────────────────┤
│  export_products_in_batches()                                            │
│  文件: saleor/csv/utils/export.py:192-223                                │
│                                                                          │
│  批次循环对象:                                                           │
│  for batch_pks in queryset_in_batches(queryset, BATCH_SIZE=1000):         │
│    batch_pks = [1, 2, 3, ..., 1000]  (list[int])                        │
│                                                                          │
│    ↓                                                                     │
│    product_batch = Product.objects.filter(pk__in=batch_pks)              │
│      .prefetch_related(                                                  │
│        "attributevalues", "variants", "collections",                     │
│        "media", "product_type", "category"                               │
│      )  (QuerySet)                                                       │
│                                                                          │
│    ↓                                                                     │
│    get_products_data(                                                    │
│      queryset=product_batch,                                             │
│      export_fields=set(...),                                             │
│      attribute_ids=[...],                                                │
│      warehouse_ids=[...],                                                │
│      channel_ids=[...]                                                   │
│    )                                                                     │
│                                                                          │
│    输出对象: products_with_variants_data (list[dict])                    │
│    [                                                                     │
│      {                                                                   │
│        "id": "UHJvZHVjdDox",                                             │
│        "name": "Product 1",                                              │
│        "variant_weight": "100 g",                                        │
│        "collections__slug": "summer, sale",                              │
│        "size (variant attribute)": "M, L",                               │
│        "main (warehouse quantity)": 100,                                 │
│        "default-channel (channel currency code)": "USD",                 │
│        ...                                                               │
│      },                                                                  │
│      ... (每变体一行)                                                    │
│    ]                                                                     │
└─────────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  阶段 4: 文件写入 → 失败反馈边界                                         │
├─────────────────────────────────────────────────────────────────────────┤
│  append_to_file()                                                        │
│  文件: saleor/csv/utils/export.py:259-271                                │
│                                                                          │
│  输入对象:                                                               │
│  export_data = [{"id": ..., "name": ..., ...}, ...]                      │
│  headers = ["id", "name", "variant weight", ...]                         │
│  temporary_file = NamedTemporaryFile(...)                                │
│                                                                          │
│  处理对象:                                                               │
│  table = etl.fromdicts(export_data, header=headers, missing="")          │
│                                                                          │
│  输出: etl.io.csv.appendcsv(table, temp_file.name, delimiter=",")         │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│  失败反馈边界: ExportTask.on_failure()                                   │
│  文件: saleor/csv/tasks.py:28-47                                         │
│                                                                          │
│  触发条件: Celery 任务抛出异常                                            │
│                                                                          │
│  处理对象流:                                                             │
│  Exception exc                                                           │
│      ↓                                                                   │
│  ExportFile.objects.get(pk=export_file_id) → export_file                 │
│      ↓                                                                   │
│  export_file.content_file = None                                         │
│  export_file.status = JobStatus.FAILED                                   │
│  export_file.save()                                                      │
│      ↓                                                                   │
│  export_failed_event(                                                    │
│    export_file=export_file,                                              │
│    message=str(exc),                                                     │
│    error_type=str(einfo.type)                                            │
│  ) → ExportEvent(...) (存入审计日志)                                      │
│      ↓                                                                   │
│  send_export_failed_info(export_file, data_type)                         │
│      ↓                                                                   │
│  NotifyEventType.CSV_EXPORT_FAILED + Webhook                             │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 导入方向完整对象流

```
┌─────────────────────────────────────────────────────────────────────────┐
│  阶段 1: GraphQL 入口 → 初始化                                           │
├─────────────────────────────────────────────────────────────────────────┤
│  ProductBulkCreate.perform_mutation()                                    │
│  文件: saleor/graphql/product/bulk_mutations/product_bulk_create.py:938-972│
│  装饰器: @traced_atomic_transaction()  # 整个操作在事务中                 │
│                                                                          │
│  输入对象:                                                               │
│  {                                                                       │
│    products: [                                                           │
│      {                                                                   │
│        name: "Product 1",                                                │
│        description: '{"blocks": [...]}',  # EditorJS JSON                │
│        category: "Q2F0ZWdvcnk6MQ==",     # Global ID                     │
│        productType: "UHJvZHVjdFR5cGU6MQ==",                              │
│        weight: {"unit": "g", "value": 100},  # WeightScalar 格式         │
│        variants: [{"sku": "SKU-001", "weight": 50, ...}],                │
│        ...                                                               │
│      },                                                                  │
│      ... (更多产品)                                                      │
│    ],                                                                    │
│    error_policy: "REJECT_EVERYTHING" | "REJECT_FAILED_ROWS"              │
│  }                                                                       │
│                                                                          │
│  初始化对象:                                                             │
│  index_error_map = defaultdict(list)  # 按产品索引收集错误                │
└─────────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  阶段 2: 全量验证 → 错误收集                                             │
├─────────────────────────────────────────────────────────────────────────┤
│  clean_products(info, products_data, index_error_map)                    │
│  文件: saleor/graphql/product/bulk_mutations/product_bulk_create.py:619-654│
│                                                                          │
│  预加载对象 (避免 N+1):                                                  │
│  warehouse_global_id_to_instance_map = {                                 │
│    "V2FyZWhvdXNlOjE=": Warehouse(...),                                    │
│    ...                                                                   │
│  }                                                                       │
│  channel_global_id_to_instance_map = {                                   │
│    "Q2hhbm5lbDox": Channel(...),                                         │
│    ...                                                                   │
│  }                                                                       │
│  duplicated_sku = {"DUPLICATE-123", ...}  # 全局去重检测                 │
│                                                                          │
│  逐产品验证循环:                                                          │
│  for product_index, product_data in enumerate(products_data):            │
│                                                                          │
│    ↓  clean_base_fields()                                                │
│    cleaned_input["description_plaintext"] = editorjs_to_text(description)│
│    weight = cleaned_input.get("weight")                                  │
│    if weight and weight.value < 0:                                       │
│      index_error_map[product_index].append(                              │
│        ProductBulkCreateError(                                           │
│          path="weight",                                                  │
│          message="Product can't have negative weight.",                  │
│          code=ProductBulkCreateErrorCode.INVALID.value                   │
│        )                                                                 │
│      )                                                                   │
│                                                                          │
│    ↓  clean_attributes()                                                 │
│    AttributeAssignmentMixin.clean_input() → ValidationError              │
│    → add_indexes_to_errors() → index_error_map[product_index]            │
│                                                                          │
│    ↓  clean_media()                                                      │
│    clean_image_file() / probe_media_url() → ValidationError              │
│    → index_error_map[product_index]                                      │
│                                                                          │
│    ↓  clean_product_channel_listings()                                   │
│    验证 channel_id 存在、不重复、发布状态合法                              │
│    → index_error_map[product_index]                                      │
│                                                                          │
│    ↓  clean_variants()                                                   │
│    委托给 ProductVariantBulkCreate.clean_variant()                        │
│    → variant_index_error_map → index_error_map[product_index]            │
│                                                                          │
│  输出对象:                                                               │
│  cleaned_inputs_map = {0: cleaned_input_0, 1: None, 2: cleaned_input_2, ...}│
└─────────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  阶段 3: 实例构建 → 错误策略检查                                         │
├─────────────────────────────────────────────────────────────────────────┤
│  create_products(info, cleaned_inputs_map, index_error_map)              │
│  文件: saleor/graphql/product/bulk_mutations/product_bulk_create.py:657-726│
│                                                                          │
│  循环构建实例 (不保存):                                                  │
│  for index, cleaned_input in cleaned_inputs_map.items():                 │
│    if not cleaned_input:                                                 │
│      instances_data_and_errors_list.append(                              │
│        {"instance": None, "errors": index_error_map[index]}              │
│      )                                                                   │
│      continue                                                            │
│                                                                          │
│    instance = models.Product()                                           │
│    instance = cls.construct_instance(instance, cleaned_input)            │
│    cls.clean_instance(info, instance)  # full_clean() 验证                │
│    instance.search_index_dirty = True                                    │
│                                                                          │
│    → variants_instances_data = [] (ProductVariant 实例，未保存)           │
│    → media_to_create = [] (ProductMedia 数据，未保存)                     │
│    → listings_to_create = [] (ProductChannelListing 数据，未保存)        │
│                                                                          │
│  输出对象:                                                               │
│  instances_data_with_errors_list = [                                     │
│    {                                                                     │
│      "instance": Product(name="Product 1", ...),  # 未保存               │
│      "errors": [],                                                       │
│      "cleaned_input": {"variants": [...], "media": [...], ...}            │
│    },                                                                    │
│    {                                                                     │
│      "instance": None,                                                   │
│      "errors": [ProductBulkCreateError(path="weight", ...)]              │
│    },                                                                    │
│    ...                                                                   │
│  ]                                                                       │
├─────────────────────────────────────────────────────────────────────────┤
│  错误策略检查 (失败反馈边界)                                             │
│  文件: saleor/graphql/product/bulk_mutations/product_bulk_create.py:949-958│
│                                                                          │
│  if any(index_error_map.values()):                                       │
│                                                                          │
│    # 策略 1: 全部拒绝                                                    │
│    if error_policy == "REJECT_EVERYTHING":                               │
│      results = get_results(instances_data_with_errors_list, True)         │
│      return ProductBulkCreate(count=0, results=results)  # 直接返回       │
│                                                                          │
│    # 策略 2: 只拒绝错误行                                                │
│    if error_policy == "REJECT_FAILED_ROWS":                              │
│      for data in instances_data_with_errors_list:                        │
│        if data["errors"] and data["instance"]:                           │
│          data["instance"] = None  # 标记为不保存                         │
└─────────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  阶段 4: 批量保存 → M2M 关系 → 后置操作                                  │
├─────────────────────────────────────────────────────────────────────────┤
│  save(info, instances_data_with_errors_list)                              │
│  文件: saleor/graphql/product/bulk_mutations/product_bulk_create.py:778-827│
│                                                                          │
│  批量写入顺序 (按表 bulk_create):                                        │
│                                                                          │
│  1. Product.objects.bulk_create(products_to_create)                      │
│     → products_to_create = [Product(...), Product(...), ...] (已保存)    │
│                                                                          │
│  2. ProductMedia.objects.bulk_create(media_to_create)                    │
│     → media_to_create = [ProductMedia(product=product_1, ...), ...]      │
│                                                                          │
│  3. ProductChannelListing.objects.bulk_create(listings_to_create)        │
│     → listings_to_create = [ProductChannelListing(product=product_1, ...)]│
│                                                                          │
│  4. 逐个保存属性 (不能 bulk_create)                                      │
│     for product, attributes in attributes_to_save:                       │
│       AttributeAssignmentMixin.save(product, attributes)                 │
│                                                                          │
│  5. 批量保存变体 (委托给 ProductVariantBulkCreate.save_variants())        │
│     → variants = [ProductVariant(...), ...] (已保存)                     │
│                                                                          │
│  ↓                                                                       │
│  _save_m2m()  # 保存多对多关系                                            │
│  CollectionProduct.objects.bulk_create([                                 │
│    CollectionProduct(product=product_1, collection=collection_1),        │
│    ...                                                                   │
│  ])                                                                      │
│                                                                          │
│  ↓                                                                       │
│  post_save_actions()  # 后置操作                                          │
│  manager.product_created(product, webhooks=webhooks)  # Webhook          │
│  manager.product_variant_created(variant, webhooks=webhooks)  # Webhook  │
│  mark_active_catalogue_promotion_rules_as_dirty(channel_ids)  # 促销标记  │
│                                                                          │
│  输出对象:                                                               │
│  ProductBulkCreate(                                                      │
│    count=3,  # 成功创建数量                                              │
│    results=[                                                             │
│      ProductBulkResult(                                                  │
│        product=ChannelContext(node=Product(...), channel_slug=None),      │
│        errors=[]                                                         │
│      ),                                                                  │
│      ProductBulkResult(                                                  │
│        product=None,                                                     │
│        errors=[ProductBulkCreateError(path="weight", ...)]               │
│      ),                                                                  │
│      ...                                                                 │
│    ]                                                                     │
│  )                                                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 批次处理：双向机制对比

#### 导出批次处理 (Database → CSV)

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

**导出批次特性**：
- ✅ 使用 `pk__gt` 游标式分页，性能稳定，避免大 offset
- ✅ 每批独立预取关联数据，避免 N+1 查询
- ✅ 每批直接写入文件，内存占用低（流式处理）
- ❌ 没有事务保护，失败后需重新导出

#### 导入批次处理 (JSON → Database)

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

**导入批次特性**：
- ✅ 先全量验证，再批量写入，减少数据库 IO 次数
- ✅ 整个操作在 `@traced_atomic_transaction()` 事务中，数据一致性有保障
- ✅ 支持两种错误处理策略，灵活应对不同场景
- ❌ 没有分片，所有数据一次性加载到内存（输入限制需外部控制）

---

## 四、失败反馈：完整链路串联

### 4.1 导出失败反馈链路

**文件**: `saleor/csv/tasks.py:20-57`

```
Celery Task Exception
    ↓  (exc: Exception, einfo: ExceptionInfo)
ExportTask.on_failure(exc, task_id, args, kwargs, einfo)
    │
    ├─ 1. 更新 ExportFile 状态
    │   export_file_id = args[0]
    │   export_file = ExportFile.objects.get(pk=export_file_id)
    │   export_file.content_file = None
    │   export_file.status = JobStatus.FAILED
    │   export_file.save(update_fields=["status", "updated_at", "content_file"])
    │
    ├─ 2. 记录错误事件 (审计日志)
    │   events.export_failed_event(
    │       export_file=export_file,
    │       user=export_file.user,
    │       app=export_file.app,
    │       message=str(exc),
    │       error_type=str(einfo.type)
    │   )
    │   ↓
    │   ExportEvent.objects.create(
    │       export_file=export_file,
    │       type=ExportEvents.EXPORT_FAILED,
    │       parameters={"message": str(exc), "error_type": str(einfo.type)},
    │       user=export_file.user,
    │       app=export_file.app
    │   )
    │
    └─ 3. 发送通知
        send_export_failed_info(export_file, data_type)
        │
        ├─ manager.notify(NotifyEventType.CSV_EXPORT_FAILED, payload_func=...)
        │  → 邮件通知用户
        │
        └─ manager.product_export_completed(export_file)
           → Webhook 通知
```

**导出错误码** (`saleor/csv/error_codes.py`):
```python
class ExportErrorCode(Enum):
    GRAPHQL_ERROR = "graphql_error"    # GraphQL 层面错误 (如参数解析失败)
    INVALID = "invalid"                # 参数无效 (如格式错误)
    NOT_FOUND = "not_found"            # 资源不存在 (如 channel 不存在)
    REQUIRED = "required"              # 缺少必填字段
```

### 4.2 导入失败反馈链路

**文件**: `saleor/graphql/product/bulk_mutations/product_bulk_create.py:264-283`

**核心数据结构**: `index_error_map`

```python
index_error_map = defaultdict(list)

# 结构 (逐产品索引收集):
{
    0: [  # 第 0 个产品的错误
        ProductBulkCreateError(
            path="weight",              # 错误字段路径 (camelCase)
            message="Product can't have negative weight.",
            code="INVALID",
        ),
        ProductBulkCreateError(
            path="variants.0.sku",      # 嵌套路径: 第 0 个变体的 sku
            message="SKU already exists.",
            code="DUPLICATED",
            values=["DUPLICATE_SKU_123"],  # 相关值 (如重复的 SKU)
        ),
    ],
    2: [  # 第 2 个产品的错误
        ProductBulkCreateError(
            path="channelListings",
            message="Not existing channel ID.",
            code="NOT_FOUND",
            channels=["Q2hhbm5lbDox"],   # 相关 channel ID
        ),
    ],
}
```

**错误收集点分布** (代码证据):

| 阶段 | 函数 | 错误类型 | 代码位置 |
|------|------|---------|---------|
| 基础字段验证 | `clean_base_fields()` | weight < 0 | `product_bulk_create.py:239-248` |
| 属性验证 | `clean_attributes()` | AttributeAssignmentMixin 错误 | `product_bulk_create.py:286-311` |
| 媒体验证 | `clean_media()` | 图片格式/URL 无效 | `product_bulk_create.py:429-486` |
| 渠道列表验证 | `clean_product_channel_listings()` | channel_id 不存在、重复 | `product_bulk_create.py:367-426` |
| 变体验证 | `clean_variants()` | 委托给 ProductVariantBulkCreate | `product_bulk_create.py:488-551` |
| 实例构建 | `create_products()` | full_clean() 验证失败 | `product_bulk_create.py:712-724` |

**错误策略执行** (`product_bulk_create.py:949-958`):
```python
if any(index_error_map.values()):
    # 策略 1: 有任何错误，全部拒绝，返回 count=0
    if error_policy == ErrorPolicyEnum.REJECT_EVERYTHING.value:
        results = get_results(instances_data_with_errors_list, True)
        return ProductBulkCreate(count=0, results=results)
    
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
    products: [
      {name: "Product 1", weight: -1, productType: "UHJvZHVjdFR5cGU6MQ=="},
      {name: "Product 2", sku: "DUPLICATE", productType: "UHJvZHVjdFR5cGU6MQ=="},
      {name: "Product 3", productType: "UHJvZHVjdFR5cGU6MQ=="}
    ],
    errorPolicy: REJECT_FAILED_ROWS
  ) {
    count          # 成功创建的数量: 1 (只有第 3 个成功)
    results {      # 每个产品的结果 (与输入顺序一致)
      product {    # 成功创建的产品对象，失败则为 null
        id
        name
      }
      errors {     # 该产品的错误列表
        path       # 如 "weight", "variants.0.sku"
        message    # 人类可读消息
        code       # 错误码枚举 (INVALID, DUPLICATED, NOT_FOUND, ...)
        values     # 相关值 (如重复的 SKU)
        channels   # 相关 channel ID (如果是渠道错误)
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
│   ├─ fields: [ProductFieldEnum.NAME, ProductFieldEnum.VARIANT_WEIGHT, ...]
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
├─ 阶段 3: 错误策略检查 (失败反馈边界)
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
| 导出数据处理 | `saleor/csv/utils/products_data.py` | 产品/变体关系数据提取、annotate 计算 |
| 导出核心逻辑 | `saleor/csv/utils/export.py` | 批次处理、petl 文件生成 |
| 导出任务调度 | `saleor/csv/tasks.py` | Celery 任务、错误回调 |
| 导出事件记录 | `saleor/csv/events.py` | 导出生命周期事件审计 |
| 导出通知 | `saleor/csv/notifications.py` | 成功/失败邮件 + Webhook |
| 导入核心逻辑 | `saleor/graphql/product/bulk_mutations/product_bulk_create.py` | 批量创建 mutation |
| 变体批量导入 | `saleor/graphql/product/bulk_mutations/product_variant_bulk_create.py` | 变体批量创建 |
| 重量标量 | `saleor/graphql/core/scalars.py` | `WeightScalar` 解析和序列化 |
| 批次工具 | `saleor/core/utils/batches.py` | `queryset_in_batches` 游标分页 |
| 导出错误码 | `saleor/csv/error_codes.py` | `ExportErrorCode` 枚举 |
| 导入错误码 | `saleor/product/error_codes.py` | `ProductBulkCreateErrorCode` 枚举 |
| CSV Mutation 注册 | `saleor/graphql/csv/schema.py` | `CsvMutations` 只注册 3 个导出 mutation |

---

## 七、设计洞察（代码事实 + 推断）

### 代码事实

1. **不对称设计**：导出有完整的 CSV 处理，导入只有 GraphQL JSON 接口
   - 代码证据：`CsvMutations` 只有 3 个 export mutation，没有 import mutation
   - 代码证据：`petl` 库只有 `tocsv`/`appendcsv`，没有 `fromcsv`/`read_csv`

2. **导出优化为读性能**：
   - 代码证据：使用 `DATABASE_CONNECTION_REPLICA_NAME` 读取从库
   - 代码证据：`pk__gt` 游标式分页避免大 offset
   - 代码证据：每批处理后立即写入文件，降低内存占用

3. **导入优化为数据一致性**：
   - 代码证据：`@traced_atomic_transaction()` 包裹整个 mutation
   - 代码证据：先全量验证，再批量写入
   - 代码证据：两种错误策略支持（`REJECT_EVERYTHING` / `REJECT_FAILED_ROWS`）

4. **字段映射不匹配**：
   - 代码证据：导出的 `variant_weight` 是 annotate 生成的带单位字符串，导入的 `WeightScalar` 需要数字或对象
   - 代码证据：导出用 `__slug`/`__name`，导入用 `graphene.ID()`
   - 代码证据：导出用 `Cast("description", CharField())`，导入用 `JSONString`

### 推断说明（无直接代码）

1. **为什么没有 CSV 文件导入？**
   - CSV 解析需要处理太多边缘情况（格式、编码、数据清洗、空值处理）
   - 字段映射不匹配（slug/名称 vs ID，字符串 vs 对象）
   - 留给用户/前端/外部系统处理，Saleor 只提供结构化的 GraphQL 接口

2. **用户需要自行实现的转换层**：
   - CSV 解析库（如 Python `csv.DictReader` 或 `pandas`）
   - slug → ID 查询转换（批量查询分类、商品类型、合集等）
   - 单位字符串解析（正则提取数值和单位）
   - 纯文本 → EditorJS JSON 格式包装

3. **扩展性建议**：
   - 如果需要 CSV 文件导入，可以在 `saleor/csv/` 目录下新增 `import*.py` 模块
   - 参考导出的批次处理模式，实现导入的批次处理
   - 复用 `index_error_map` 模式处理导入错误
