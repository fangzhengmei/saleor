# 商品规格(ProductVariant)与属性(Attribute)搜索索引实现分析

## 一、模型定义与关联关系

### 1.1 核心模型结构

#### Product 模型（搜索索引主体）
`saleor/product/models.py:174-265`

```python
class Product(SeoModel, ModelWithMetadata, ModelWithExternalReference):
    search_vector = SearchVectorField(blank=True, null=True)      # 全文搜索向量
    search_index_dirty = models.BooleanField(default=False, db_index=True)  # 索引脏标记
    search_document = models.TextField(blank=True, default="")   # Trigram搜索文档
    
    class Meta:
        indexes = [
            GinIndex(name="product_search_gin", fields=["search_document"], opclasses=["gin_trgm_ops"]),
            GinIndex(name="product_tsearch", fields=["search_vector"]),
            GinIndex(name="product_gin", fields=["name", "slug"], opclasses=["gin_trgm_ops"] * 2),
        ]
```

**关键点**：
- `search_vector` 存储PostgreSQL tsvector类型的全文搜索向量
- `search_index_dirty` 标记是否需要刷新索引
- 使用GIN索引提高搜索性能

#### ProductVariant 与 Attribute 关联模型
`saleor/attribute/models/product_variant.py:1-78`

三级关联结构：
```
ProductVariant
    ↓ attributes (ManyToMany)
AssignedVariantAttribute
    ↓ assignment (ForeignKey)
AttributeVariant  (ProductType与Attribute的关联)
    ↓ attribute (ForeignKey)
Attribute
    ↓ values (ManyToMany)
AttributeValue
    ↓ 通过 AssignedVariantAttributeValue 中间表关联
AssignedVariantAttributeValue
```

**关键模型**：
1. `AttributeVariant` - 定义ProductType与Attribute的关联，支持`variant_selection`标记
2. `AssignedVariantAttribute` - Variant与AttributeVariant的关联实例
3. `AssignedVariantAttributeValue` - 存储Variant具体选择的AttributeValue

---

## 二、属性归集逻辑

### 2.1 搜索向量生成入口

`saleor/product/search.py:77-96`

```python
def prepare_product_search_vector_value(
    product: "Product",
    *,
    already_prefetched=False,
    page_id_to_title_map: dict[int, str] | None = None,
) -> list[NoValidationSearchVector]:
    search_vectors = [
        NoValidationSearchVector(Value(product.name), config="simple", weight="A"),
        NoValidationSearchVector(Value(product.description_plaintext), config="simple", weight="C"),
        *generate_attributes_search_vector_value(product, page_id_to_title_map=page_id_to_title_map),
        *generate_variants_search_vector_value(product),  # Variant属性归集入口
    ]
    return search_vectors
```

### 2.2 Variant属性归集

`saleor/product/search.py:99-118`

```python
def generate_variants_search_vector_value(
    product: "Product",
) -> list[NoValidationSearchVector]:
    variants = list(product.variants.all()[: settings.PRODUCT_MAX_INDEXED_VARIANTS])
    
    # 1. 归集Variant的sku和name（权重A）
    search_vectors = [
        NoValidationSearchVector(
            Value(variant.sku), Value(variant.name), config="simple", weight="A"
        )
        if variant.sku
        else NoValidationSearchVector(Value(variant.name), config="simple", weight="A")
        for variant in variants
        if variant.sku or variant.name
    ]
    
    # 2. 归集Variant的属性值（权重B）
    if search_vectors:
        for variant in variants:
            search_vectors += generate_attributes_search_vector_value_with_assignment(
                variant.attributes.all()[: settings.PRODUCT_MAX_INDEXED_ATTRIBUTES]
            )
    return search_vectors
```

**配置限制**：
- `PRODUCT_MAX_INDEXED_VARIANTS` - 最多索引的Variant数量
- `PRODUCT_MAX_INDEXED_ATTRIBUTES` - 最多索引的属性数量
- `PRODUCT_MAX_INDEXED_ATTRIBUTE_VALUES` - 每个属性最多索引的值数量

### 2.3 Variant属性值向量生成

`saleor/product/search.py:155-172`

```python
def generate_attributes_search_vector_value_with_assignment(
    assigned_attributes: "QuerySet",
) -> list[NoValidationSearchVector]:
    search_vectors = []
    for assigned_attribute in assigned_attributes:
        attribute = assigned_attribute.assignment.attribute
        values = assigned_attribute.values.all()[
            : settings.PRODUCT_MAX_INDEXED_ATTRIBUTE_VALUES
        ]
        search_vectors += get_search_vectors_for_attribute_values(
            attribute, values, weight="B"
        )
    return search_vectors
```

### 2.4 属性值类型处理

`saleor/attribute/search.py:11-78`

根据不同的`AttributeInputType`生成不同的搜索向量：

| 输入类型 | 处理方式 | 权重 |
|---------|---------|------|
| DROPDOWN / MULTISELECT | 使用 `value.name` | B |
| RICH_TEXT | 使用 `editorjs_to_text(value.rich_text)` | B |
| PLAIN_TEXT | 使用 `value.plain_text` | B |
| NUMERIC | 使用 `value.name + " " + unit` | B |
| DATE / DATE_TIME | 格式化为 `YYYY-MM-DD HH:MM:SS` | B |
| REFERENCE / SINGLE_REFERENCE | 使用关联Page的title | B |

---

## 三、索引刷新机制

### 3.1 脏标记触发点

索引刷新采用"标记-异步更新"模式。以下场景会标记`search_index_dirty=True`：

#### 场景1：Variant创建/更新
`saleor/graphql/product/mutations/product_variant/product_variant_create.py:247-248`

```python
instance.product.search_index_dirty = True
instance.product.save(update_fields=["search_index_dirty"])
```

**相关文件**：
- `product_variant_create.py` - 创建Variant时
- `product_variant_update.py` - 更新Variant时
- `product_variant_bulk_update.py` - 批量更新时
- `product_variant_bulk_delete.py` - 批量删除时（直接同步更新search_vector）

#### 场景2：属性值更新
`saleor/graphql/attribute/mutations/attribute_value_update.py:86-110`

```python
@classmethod
def _mark_products_search_index_dirty(cls, instance):
    variants = product_models.ProductVariant.objects.filter(
        Exists(instance.variantassignments.filter(variant_id=OuterRef("id")))
    )
    products = product_models.Product.objects.filter(
        Q(search_index_dirty=False)
        & (
            Q(Exists(instance.productvalueassignment.filter(product_id=OuterRef("id"))))
            | Q(Exists(variants.filter(product_id=OuterRef("id"))))
        )
    ).order_by("pk")
    mark_products_search_vector_as_dirty_in_batches(
        list(products.values_list("id", flat=True))
    )
```

**触发时机**：当AttributeValue的name、value等字段更新时，所有使用该属性值的Product都会被标记为脏。

#### 场景3：属性分配变更
`saleor/graphql/product/mutations/attributes.py:348-353`

```python
product_ids = list(
    models.Product.objects.filter(product_type=product_type).values_list("id", flat=True)
)
mark_products_search_vector_as_dirty_in_batches(product_ids)
```

**触发时机**：从ProductType中取消分配属性时。

#### 场景4：ProductType更新
`saleor/graphql/product/mutations/product_type/product_type_update.py:51-54`

```python
if has_variants_changed and not instance.has_variants:
    product_ids = list(instance.products.values_list("id", flat=True))
    if product_ids:
        mark_products_search_vector_as_dirty_in_batches(product_ids)
```

### 3.2 批量标记工具

`saleor/product/utils/search_helpers.py:7-11`

```python
MARK_SEARCH_VECTOR_DIRTY_BATCH_SIZE = 1000

def mark_products_search_vector_as_dirty_in_batches(product_ids: list[int]):
    for i in range(0, len(product_ids), MARK_SEARCH_VECTOR_DIRTY_BATCH_SIZE):
        batch_ids = product_ids[i : i + MARK_SEARCH_VECTOR_DIRTY_BATCH_SIZE]
        mark_products_search_vector_as_dirty.delay(batch_ids)
```

### 3.3 脏标记任务

`saleor/product/tasks.py:337-345`

```python
@app.task
@allow_writer()
def mark_products_search_vector_as_dirty(product_ids: list[int]):
    if not product_ids:
        return
    with transaction.atomic():
        ids = product_qs_select_for_update().filter(pk__in=product_ids).values("id")
        Product.objects.filter(id__in=ids).update(search_index_dirty=True)
```

**关键点**：使用`select_for_update`行锁避免并发更新冲突。

### 3.4 定时更新任务

`saleor/product/tasks.py:348-360`

```python
@app.task(
    queue=settings.UPDATE_SEARCH_VECTOR_INDEX_QUEUE_NAME,
    expires=settings.BEAT_UPDATE_SEARCH_EXPIRE_AFTER_SEC,
)
def update_products_search_vector_task():
    products = (
        Product.objects.using(settings.DATABASE_CONNECTION_REPLICA_NAME)
        .filter(search_index_dirty=True)
        .order_by("updated_at")[:PRODUCTS_BATCH_SIZE]  # 300
        .values_list("id", flat=True)
    )
    with allow_writer():
        update_products_search_vector(products)
```

**调度**：由Celery Beat定时触发，轮询处理脏数据。

### 3.5 索引更新核心逻辑

`saleor/product/search.py:32-49`

```python
def _prep_product_search_vector_index(
    products, page_id_to_title_map: dict[int, str] | None = None
):
    prefetch_related_objects(products, *PRODUCT_FIELDS_TO_PREFETCH)
    
    for product in products:
        product.search_vector = FlatConcatSearchVector(
            *prepare_product_search_vector_value(
                product,
                already_prefetched=True,
                page_id_to_title_map=page_id_to_title_map,
            )
        )
        product.search_index_dirty = False
    
    Product.objects.bulk_update(
        products, ["search_vector", "updated_at", "search_index_dirty"]
    )
```

**预取字段**：
```python
PRODUCT_FIELDS_TO_PREFETCH = [
    "variants__attributes__values",
    "variants__attributes__assignment__attribute",
    "attributevalues__value",
    "product_type__attributeproduct__attribute",
]
```

**批量处理**：`PRODUCTS_BATCH_SIZE = 100`，每批处理100个产品。

---

## 四、查询匹配逻辑

### 4.1 全文搜索（使用search_vector）

#### 搜索过滤入口
`saleor/graphql/product/filters/product_helpers.py:324-325`

```python
def filter_search(qs, _, value):
    return prefix_search(qs, value)
```

#### 核心搜索算法
`saleor/core/search.py:144-179`

```python
def prefix_search(qs: "QuerySet", value: str) -> "QuerySet":
    value = strip_accents(value)
    
    parsed_query = parse_search_query(value)
    if not parsed_query:
        return qs.annotate(search_rank=Value(0)).none()
    
    # 前缀查询 - 用于过滤
    prefix_query = SearchQuery(parsed_query, search_type="raw", config="simple")
    
    # 精确查询 - 仅用于排序
    exact_query = SearchQuery(value, search_type="websearch", config="simple")
    
    qs = qs.filter(search_vector=prefix_query).annotate(
        prefix_rank=SearchRank(F("search_vector"), prefix_query),
        exact_rank=SearchRank(F("search_vector"), exact_query),
        search_rank=F("exact_rank") * 2 + F("prefix_rank"),  # 精确匹配权重翻倍
    )
    
    return qs
```

**搜索语法支持**（`parse_search_query`函数）：
- 多词隐式AND：`"coffee shop"` → `coffee:* & shop:*`
- OR运算符：`"coffee OR tea"` → `coffee:* | tea:*`
- 否定：`"-decaf"` → `!decaf:*`
- 引号短语：`'"green tea"'` → `(green <-> tea)`

**评分规则**：
- 精确匹配(websearch)：2x权重
- 前缀匹配：1x权重

### 4.2 属性过滤（不使用search_vector）

属性过滤采用直接联表查询方式，不依赖search_vector索引。

#### 属性过滤入口
`saleor/graphql/product/filters/product_attributes.py:976-984`

```python
def filter_products_by_attributes(
    qs: QuerySet[Product], value: list[dict[str, str | dict | list | bool]]
) -> QuerySet[Product]:
    if not value:
        return qs.none()
    
    if set(value[0].keys()).difference({"slug", "value"}):
        return deprecated_filter_attributes(qs, value)
    return _filter_products_by_attributes(qs, value)
```

#### 新属性过滤逻辑
`saleor/graphql/product/filters/product_attributes.py:851-931`

```python
def _filter_products_by_attributes(
    qs: QuerySet[Product], value: list[dict]
) -> QuerySet[Product]:
    attr_filter_expression = Q()
    
    for attr_filter in value:
        attr_value = attr_filter.get("value")
        attr_slug = attr_filter.get("slug")
        attr_id = attributes_map[attr_slug].id if attr_slug else None
        
        # 根据值类型选择不同的过滤函数
        if "slug" in attr_value or "name" in attr_value:
            attr_filter_expression &= filter_by_slug_or_name(attr_id, attr_value, qs.db)
        elif "numeric" in attr_value:
            attr_filter_expression &= filter_by_numeric_attribute(attr_id, attr_value["numeric"], qs.db)
        elif "boolean" in attr_value:
            attr_filter_expression &= filter_by_boolean_attribute(attr_id, attr_value["boolean"], qs.db)
        elif "date" in attr_value:
            attr_filter_expression &= filter_by_date_attribute(attr_id, attr_value["date"], qs.db)
        elif "date_time" in attr_value:
            attr_filter_expression &= filter_by_date_time_attribute(attr_id, attr_value["date_time"], qs.db)
        elif "reference" in attr_value:
            attr_filter_expression &= filter_objects_by_reference_attributes(attr_id, attr_value["reference"], qs.db)
    
    if attr_filter_expression != Q():
        return qs.filter(attr_filter_expression)
    return qs.none()
```

#### Product属性值查询
`saleor/graphql/product/filters/product_attributes.py:325-336`

```python
def _get_assigned_product_attribute_for_attribute_value(
    attribute_values: QuerySet[AttributeValue],
    db_connection_name: str,
):
    return Q(
        Exists(
            AssignedProductAttributeValue.objects.using(db_connection_name).filter(
                Exists(attribute_values.filter(id=OuterRef("value_id"))),
                product_id=OuterRef("id"),
            )
        )
    )
```

#### 旧版属性过滤（同时支持Product和Variant属性）
`saleor/graphql/product/filters/product_attributes.py:200-229`

```python
def filter_products_by_attributes_values(qs, queries: T_PRODUCT_FILTER_QUERIES):
    filters = []
    for values in queries.values():
        # Product属性过滤
        assigned_product_attribute_values = AssignedProductAttributeValue.objects.using(
            qs.db
        ).filter(value_id__in=values)
        product_attribute_filter = Q(
            Exists(assigned_product_attribute_values.filter(product_id=OuterRef("pk")))
        )
        
        # Variant属性过滤
        assigned_variant_attribute_values = AssignedVariantAttributeValue.objects.using(
            qs.db
        ).filter(value_id__in=values)
        assigned_variant_attributes = AssignedVariantAttribute.objects.using(
            qs.db
        ).filter(
            Exists(
                assigned_variant_attribute_values.filter(assignment_id=OuterRef("pk"))
            )
        )
        product_variants = ProductVariant.objects.using(qs.db).filter(
            Exists(assigned_variant_attributes.filter(variant_id=OuterRef("pk")))
        )
        variant_attribute_filter = Q(
            Exists(product_variants.filter(product_id=OuterRef("pk")))
        )
        
        filters.append(product_attribute_filter | variant_attribute_filter)
    
    return qs.filter(*filters)
```

**关键点**：旧版过滤同时查询Product和Variant的属性，使用`OR`条件合并。

---

## 五、完整流程时序图

### 5.1 Variant创建时索引更新流程

```
GraphQL Mutation (productVariantCreate)
    ↓
save() 方法
    ├─ 保存ProductVariant
    ├─ 保存属性值 (AttributeAssignmentMixin.save)
    ├─ 生成Variant名称
    └─ 标记Product.search_index_dirty = True
        ↓
Celery Beat定时任务
    ↓
update_products_search_vector_task()
    ├─ 查询search_index_dirty=True的Product（按updated_at排序）
    └─ 调用update_products_search_vector(products)
        ↓
update_products_search_vector()
    ├─ 批量获取关联数据（Prefetch）
    ├─ 生成search_vector（包含Product和Variant属性）
    └─ 批量更新Product.search_vector和search_index_dirty=False
```

### 5.2 属性值更新时索引更新流程

```
GraphQL Mutation (attributeValueUpdate)
    ↓
post_save_action()
    ↓
mark_search_index_dirty(instance)
    ├─ 查找使用该AttributeValue的所有ProductVariant
    ├─ 查找关联的所有Product（且search_index_dirty=False）
    └─ 批量调用mark_products_search_vector_as_dirty.delay()
        ↓
Celery任务执行
    ↓
mark_products_search_vector_as_dirty()
    └─ 使用select_for_update行锁更新Product.search_index_dirty=True
        ↓
（后续流程同上，由定时任务处理）
```

### 5.3 搜索查询流程

```
GraphQL Query (products)
    ↓
resolve_products() → 获取基础QuerySet
    ↓
ProductFilter.filter_search()
    ↓
prefix_search(qs, value)
    ├─ strip_accents() - 去除重音
    ├─ parse_search_query() - 解析搜索语法
    ├─ 构建prefix_query（前缀匹配，用于过滤）
    ├─ 构建exact_query（精确匹配，用于排序）
    └─ 返回带search_rank注解的QuerySet
        ↓
（可选）ProductFilter.filter_attributes()
    ↓
filter_products_by_attributes()
    └─ 直接联表查询AssignedProductAttributeValue/AssignedVariantAttributeValue
        ↓
返回最终结果
```

---

## 六、关键设计特点

### 6.1 读写分离
- 写入：标记`search_index_dirty=True`，快速返回
- 读取：定时任务异步批量更新`search_vector`
- 查询：直接使用`search_vector`进行全文搜索

### 6.2 并发控制
- 使用`select_for_update`行锁避免并发标记冲突
- 批量处理减少数据库压力
- 队列机制（Celery）削峰填谷

### 6.3 索引分层
- **全文搜索**：使用`search_vector`（tsvector + GIN索引），支持复杂语法
- **属性过滤**：直接联表查询，不依赖search_vector，保证实时性
- **Trigram搜索**：使用`search_document` + gin_trgm_ops索引，支持模糊匹配

### 6.4 权重设计
| 内容 | 权重 | 说明 |
|-----|------|------|
| Product.name | A | 最高优先级 |
| Variant.sku / Variant.name | A | 最高优先级 |
| 属性值（所有类型） | B | 中等优先级 |
| Product.description_plaintext | C | 最低优先级 |

### 6.5 性能优化
- 预取关联数据（`prefetch_related_objects`）减少N+1查询
- 批量更新（`bulk_update`）减少数据库交互
- 只读副本查询（`using(db_conn)`）降低主库压力
- 合理的批量大小（100-1000）平衡内存和性能
