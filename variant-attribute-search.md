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

## 二、属性归集逻辑（带执行顺序和代码证据）

### 2.0 属性保存到数据库的完整流程（批量创建为例）

**执行顺序**：
```
Step 1: GraphQL Mutation 入口
        ↓ product_variant_bulk_create.py:886
Step 2: AttributeAssignmentMixin.save()
        ↓ attribute_assignment.py:368-389
Step 3: pre_save_values() 准备批量操作
        ↓ attribute_assignment.py:299-365
Step 4: _bulk_create_pre_save_values() 执行批量数据库操作
        ↓ attribute_assignment.py:425-446
Step 5: associate_attribute_values_to_instance() 建立关联
        ↓ attribute/utils/__init__.py
Step 6: 标记 search_index_dirty = True
        ↓ product_variant_bulk_create.py:919-920
```

**代码证据 Step 1 - 批量创建入口**：
`saleor/graphql/product/bulk_mutations/product_variant_bulk_create.py:878-886`
```python
if attributes := cleaned_input.get("attributes"):
    attributes_to_save.append((variant, attributes))

if not variant.name:
    cls.set_variant_name(variant, cleaned_input)

models.ProductVariant.objects.bulk_create(variants_to_create)

for variant, attributes in attributes_to_save:
    AttributeAssignmentMixin.save(variant, attributes)  # ← 属性保存入口
```

**代码证据 Step 2 - AttributeAssignmentMixin.save()**：
`saleor/graphql/attribute/utils/attribute_assignment.py:368-389`
```python
@classmethod
def save(cls, instance: T_INSTANCE, cleaned_input: T_INPUT_MAP,
         pre_save_bulk: T_PRE_SAVE_BULK | None = None):
    if pre_save_bulk is None:
        pre_save_bulk = cls.pre_save_values(instance, cleaned_input)
    attribute_and_values = cls._bulk_create_pre_save_values(pre_save_bulk)

    attr_val_map = defaultdict(list)
    clean_assignment_pks = []
    for attribute, values in attribute_and_values.items():
        if not values:
            clean_assignment_pks.append(attribute.pk)
        else:
            attr_val_map[attribute.pk].extend(values)

    associate_attribute_values_to_instance(instance, attr_val_map)  # ← 建立关联
    cls._clean_assignments(instance, clean_assignment_pks)
```

**代码证据 Step 3 - pre_save_values()**：
`saleor/graphql/attribute/utils/attribute_assignment.py:299-365`
```python
@classmethod
def pre_save_values(cls, instance: T_INSTANCE, cleaned_input: T_INPUT_MAP) -> T_PRE_SAVE_BULK:
    pre_save_bulk: T_PRE_SAVE_BULK = defaultdict(lambda: defaultdict(list))
    for attribute, values_input in cleaned_input:
        # 根据input_type选择处理器
        handler_class = cls.HANDLER_MAPPING.get(attribute.input_type)
        handler = handler_class(attribute, values_input)
        prepared_values = handler.pre_save_value(instance)  # ← 各类型处理器准备值
        
        for action, value_data in prepared_values:
            pre_save_bulk[action][attribute].append(value_data)
    return pre_save_bulk
```

**处理器映射**（attribute_assignment.py:54-67）：
```python
HANDLER_MAPPING = {
    AttributeInputType.DROPDOWN: SelectableAttributeHandler,
    AttributeInputType.SWATCH: SelectableAttributeHandler,
    AttributeInputType.MULTISELECT: MultiSelectableAttributeHandler,
    AttributeInputType.FILE: FileAttributeHandler,
    AttributeInputType.REFERENCE: ReferenceAttributeHandler,
    AttributeInputType.SINGLE_REFERENCE: ReferenceAttributeHandler,
    AttributeInputType.RICH_TEXT: RichTextAttributeHandler,
    AttributeInputType.PLAIN_TEXT: PlainTextAttributeHandler,
    AttributeInputType.NUMERIC: NumericAttributeHandler,
    AttributeInputType.DATE: DateTimeAttributeHandler,
    AttributeInputType.DATE_TIME: DateTimeAttributeHandler,
    AttributeInputType.BOOLEAN: BooleanAttributeHandler,
}
```

**代码证据 Step 4 - _bulk_create_pre_save_values()**：
`saleor/graphql/attribute/utils/attribute_assignment.py:425-446`
```python
@classmethod
def _bulk_create_pre_save_values(cls, pre_save_bulk):
    results: dict[attribute_models.Attribute, list[AttributeValue]] = defaultdict(list)
    for action, attribute_data in pre_save_bulk.items():
        for attribute, values in attribute_data.items():
            if action == AttributeValueBulkActionEnum.CREATE:
                values = AttributeValue.objects.bulk_create(values)
            elif action == AttributeValueBulkActionEnum.UPDATE_OR_CREATE:
                values = AttributeValue.objects.bulk_update_or_create(values)
            elif action == AttributeValueBulkActionEnum.GET_OR_CREATE:
                values = AttributeValue.objects.bulk_get_or_create(values)
            results[attribute].extend(values)
    return results
```

**代码证据 Step 5 - associate_attribute_values_to_instance()**：
`saleor/attribute/utils/__init__.py`
```python
def associate_attribute_values_to_instance(
    instance: Union[Product, ProductVariant, Page],
    attribute_values: dict[int, list[AttributeValue]],
):
    """将属性值关联到实例。对于Variant，创建AssignedVariantAttribute和AssignedVariantAttributeValue记录。"""
```

**代码证据 Step 6 - 标记脏标**：
`saleor/graphql/product/bulk_mutations/product_variant_bulk_create.py:909-920`
```python
@classmethod
def post_save_actions(cls, info, instances, product):
    # ... 省略其他逻辑
    product.search_index_dirty = True
    product.save(update_fields=["search_index_dirty"])  # ← 标记需要刷新索引
```

### 2.1 搜索向量生成入口（索引刷新时调用）

**执行顺序**：
```
Step 1: update_products_search_vector_task() 定时任务触发
        ↓ product/tasks.py:352-360
Step 2: update_products_search_vector() 处理批次
        ↓ product/search.py:52-74
Step 3: _prep_product_search_vector_index() 预取并生成向量
        ↓ product/search.py:32-49
Step 4: prepare_product_search_vector_value() 组装各部分向量
        ↓ product/search.py:77-96
Step 5: generate_variants_search_vector_value() 归集Variant属性
        ↓ product/search.py:99-118
Step 6: generate_attributes_search_vector_value_with_assignment() 归集属性值
        ↓ product/search.py:155-172
Step 7: get_search_vectors_for_attribute_values() 按类型生成向量
        ↓ attribute/search.py:11-78
```

**代码证据 Step 1 - 定时任务**：
`saleor/product/tasks.py:348-360`
```python
@app.task(queue=settings.UPDATE_SEARCH_VECTOR_INDEX_QUEUE_NAME)
def update_products_search_vector_task():
    products = (
        Product.objects.using(settings.DATABASE_CONNECTION_REPLICA_NAME)
        .filter(search_index_dirty=True)
        .order_by("updated_at")[:PRODUCTS_BATCH_SIZE]
        .values_list("id", flat=True)
    )
    with allow_writer():
        update_products_search_vector(products)
```

**代码证据 Step 2 - 批次处理**：
`saleor/product/search.py:52-74`
```python
def update_products_search_vector(product_ids: Iterable[int]):
    db_conn = settings.DATABASE_CONNECTION_REPLICA_NAME
    products = Product.objects.using(db_conn).filter(pk__in=product_ids).order_by("pk")
    for product_pks in queryset_in_batches(products, PRODUCTS_BATCH_SIZE):
        # 预取关联的Page title（用于reference类型属性）
        value_ids = (
            AssignedProductAttributeValue.objects.using(db_conn)
            .filter(product_id__in=product_pks)
            .values_list("value_id", flat=True)
        )
        value_to_page_id = (
            AttributeValue.objects.using(db_conn)
            .filter(id__in=value_ids, reference_page_id__isnull=False)
            .values_list("id", "reference_page_id")
        )
        page_id_to_title_map = dict(
            Page.objects.using(db_conn)
            .filter(id__in=[page_id for _, page_id in value_to_page_id])
            .values_list("id", "title")
        )
        products_batch = list(Product.objects.using(db_conn).filter(id__in=product_pks))
        _prep_product_search_vector_index(products_batch, page_id_to_title_map)
```

**代码证据 Step 3 - 预取并生成向量**：
`saleor/product/search.py:32-49`
```python
def _prep_product_search_vector_index(products, page_id_to_title_map=None):
    prefetch_related_objects(products, *PRODUCT_FIELDS_TO_PREFETCH)  # ← 关键预取
    
    for product in products:
        product.search_vector = FlatConcatSearchVector(
            *prepare_product_search_vector_value(
                product, already_prefetched=True,
                page_id_to_title_map=page_id_to_title_map,
            )
        )
        product.search_index_dirty = False
    
    Product.objects.bulk_update(
        products, ["search_vector", "updated_at", "search_index_dirty"]
    )
```

**预取字段定义**（product/search.py:19-24）：
```python
PRODUCT_FIELDS_TO_PREFETCH = [
    "variants__attributes__values",                    # Variant属性值
    "variants__attributes__assignment__attribute",     # Variant属性定义
    "attributevalues__value",                          # Product属性值
    "product_type__attributeproduct__attribute",       # Product属性定义
]
```

**代码证据 Step 4 - 组装搜索向量**：
`saleor/product/search.py:77-96`
```python
def prepare_product_search_vector_value(product, *, already_prefetched=False,
                                        page_id_to_title_map=None):
    search_vectors = [
        NoValidationSearchVector(Value(product.name), config="simple", weight="A"),
        NoValidationSearchVector(
            Value(product.description_plaintext), config="simple", weight="C"
        ),
        *generate_attributes_search_vector_value(
            product, page_id_to_title_map=page_id_to_title_map
        ),
        *generate_variants_search_vector_value(product),  # ← Variant属性归集
    ]
    return search_vectors
```

**代码证据 Step 5 - Variant属性归集**：
`saleor/product/search.py:99-118`
```python
def generate_variants_search_vector_value(product):
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

**代码证据 Step 6 - Variant属性值向量生成**：
`saleor/product/search.py:155-172`
```python
def generate_attributes_search_vector_value_with_assignment(assigned_attributes):
    search_vectors = []
    for assigned_attribute in assigned_attributes:
        attribute = assigned_attribute.assignment.attribute  # ← 通过assignment获取Attribute
        values = assigned_attribute.values.all()[
            : settings.PRODUCT_MAX_INDEXED_ATTRIBUTE_VALUES
        ]
        search_vectors += get_search_vectors_for_attribute_values(
            attribute, values, weight="B"
        )
    return search_vectors
```

**代码证据 Step 7 - 按类型生成搜索向量**：
`saleor/attribute/search.py:11-78`
```python
def get_search_vectors_for_attribute_values(attribute, values, page_id_to_title_map=None, weight="B"):
    search_vectors = []
    input_type = attribute.input_type
    
    if input_type in [AttributeInputType.DROPDOWN, AttributeInputType.MULTISELECT]:
        search_vectors += [
            NoValidationSearchVector(Value(value.name), config="simple", weight=weight)
            for value in values
        ]
    elif input_type == AttributeInputType.RICH_TEXT:
        search_vectors += [
            NoValidationSearchVector(
                Value(editorjs_to_text(value.rich_text)),
                config="simple", weight=weight,
            )
            for value in values
        ]
    elif input_type == AttributeInputType.PLAIN_TEXT:
        search_vectors += [
            NoValidationSearchVector(
                Value(value.plain_text), config="simple", weight=weight
            )
            for value in values
        ]
    elif input_type == AttributeInputType.NUMERIC:
        unit = attribute.unit
        search_vectors += [
            NoValidationSearchVector(
                Value(value.name + " " + unit if unit else value.name),
                config="simple", weight=weight,
            )
            for value in values
        ]
    elif input_type in [AttributeInputType.DATE, AttributeInputType.DATE_TIME]:
        search_vectors += [
            NoValidationSearchVector(
                Value(value.date_time.strftime("%Y-%m-%d %H:%M:%S")),
                config="simple", weight=weight,
            )
            for value in values
        ]
    elif input_type in [AttributeInputType.REFERENCE, AttributeInputType.SINGLE_REFERENCE]:
        search_vectors += [
            NoValidationSearchVector(
                Value(get_reference_attribute_search_value(
                    value, page_id_to_title_map=page_id_to_title_map
                )),
                config="simple", weight=weight,
            )
            for value in values
            if value.reference_page_id is not None
        ]
    return search_vectors
```

| 输入类型 | 处理方式 | 权重 |
|---------|---------|------|
| DROPDOWN / MULTISELECT | 使用 `value.name` | B |
| RICH_TEXT | 使用 `editorjs_to_text(value.rich_text)` | B |
| PLAIN_TEXT | 使用 `value.plain_text` | B |
| NUMERIC | 使用 `value.name + " " + unit` | B |
| DATE / DATE_TIME | 格式化为 `YYYY-MM-DD HH:MM:SS` | B |
| REFERENCE / SINGLE_REFERENCE | 使用关联Page的title | B |

---

## 三、索引刷新机制（带执行顺序和代码证据）

### 3.1 脏标记触发点汇总

**执行顺序（以批量更新为例）**：
```
Step 1: productVariantBulkUpdate Mutation
        ↓ product_variant_bulk_update.py:821-883
Step 2: clean_variants() 验证输入
        ↓ product_variant_bulk_update.py:469-558
Step 3: update_variants() 构造更新对象
        ↓ product_variant_bulk_update.py:561-610
Step 4: save_variants() 执行数据库更新
        ↓ product_variant_bulk_update.py:666-753
Step 5: AttributeAssignmentMixin.save() 保存属性（如有变更）
        ↓ product_variant_bulk_update.py:700-701
Step 6: post_save_actions() 标记脏标
        ↓ product_variant_bulk_update.py:771-772
```

**代码证据 Step 4-5 - 保存属性变更**：
`saleor/graphql/product/bulk_mutations/product_variant_bulk_update.py:700-701`
```python
if attributes := cleaned_input.get("attributes"):
    AttributeAssignmentMixin.save(variant, attributes)  # ← 属性变更时调用
```

**代码证据 Step 6 - 标记脏标**：
`saleor/graphql/product/bulk_mutations/product_variant_bulk_update.py:771-772`
```python
product.search_index_dirty = True
product.save(update_fields=["search_index_dirty"])
```

### 3.2 各场景脏标记触发点

#### 场景1：Variant创建/更新/删除

**单个创建** - `saleor/graphql/product/mutations/product_variant/product_variant_create.py:247-248`
```python
instance.product.search_index_dirty = True
instance.product.save(update_fields=["search_index_dirty"])
```

**批量创建** - `saleor/graphql/product/bulk_mutations/product_variant_bulk_create.py:919-920`
```python
product.search_index_dirty = True
product.save(update_fields=["search_index_dirty"])
```

**批量更新** - `saleor/graphql/product/bulk_mutations/product_variant_bulk_update.py:771-772`
```python
product.search_index_dirty = True
product.save(update_fields=["search_index_dirty"])
```

**批量删除（特殊 - 同步更新）** - `saleor/graphql/product/bulk_mutations/product_variant_bulk_delete.py:146-156`
```python
# 批量删除时直接同步更新search_vector，不经过异步流程
for product in products:
    product.search_vector = FlatConcatSearchVector(
        *prepare_product_search_vector_value(product)
    )
    product.default_variant = product.variants.first()
    product.save(
        update_fields=[
            "default_variant",
            "search_vector",
            "updated_at",
        ]
    )
```

#### 场景2：属性值更新
`saleor/graphql/attribute/mutations/attribute_value_update.py:86-110`
```python
@classmethod
def _mark_products_search_index_dirty(cls, instance):
    # 查找使用该AttributeValue的所有ProductVariant
    variants = product_models.ProductVariant.objects.filter(
        Exists(instance.variantassignments.filter(variant_id=OuterRef("id")))
    )
    # 查找关联的所有Product（且search_index_dirty=False）
    products = product_models.Product.objects.filter(
        Q(search_index_dirty=False)
        & (
            Q(Exists(instance.productvalueassignment.filter(product_id=OuterRef("id"))))
            | Q(Exists(variants.filter(product_id=OuterRef("id"))))
        )
    ).order_by("pk")
    # 批量标记
    mark_products_search_vector_as_dirty_in_batches(
        list(products.values_list("id", flat=True))
    )
```

#### 场景3：属性分配变更
`saleor/graphql/product/mutations/attributes.py:348-353`
```python
product_ids = list(
    models.Product.objects.filter(product_type=product_type).values_list("id", flat=True)
)
mark_products_search_vector_as_dirty_in_batches(product_ids)
```

#### 场景4：ProductType更新
`saleor/graphql/product/mutations/product_type/product_type_update.py:51-54`
```python
if has_variants_changed and not instance.has_variants:
    product_ids = list(instance.products.values_list("id", flat=True))
    if product_ids:
        mark_products_search_vector_as_dirty_in_batches(product_ids)
```

### 3.3 批量标记工具

**执行顺序**：
```
Step 1: mark_products_search_vector_as_dirty_in_batches() 分批
        ↓ product/utils/search_helpers.py:7-11
Step 2: mark_products_search_vector_as_dirty.delay() 异步任务
        ↓ product/tasks.py:337-345
Step 3: 使用select_for_update行锁更新
        ↓ product/lock_objects.py:6-7
```

**代码证据 Step 1 - 分批**：
`saleor/product/utils/search_helpers.py:7-11`
```python
MARK_SEARCH_VECTOR_DIRTY_BATCH_SIZE = 1000

def mark_products_search_vector_as_dirty_in_batches(product_ids: list[int]):
    for i in range(0, len(product_ids), MARK_SEARCH_VECTOR_DIRTY_BATCH_SIZE):
        batch_ids = product_ids[i : i + MARK_SEARCH_VECTOR_DIRTY_BATCH_SIZE]
        mark_products_search_vector_as_dirty.delay(batch_ids)  # ← 每批1000个
```

**代码证据 Step 2-3 - 行锁更新**：
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

`saleor/product/lock_objects.py:6-7`
```python
def product_qs_select_for_update() -> QuerySet[Product]:
    return Product.objects.order_by("pk").select_for_update(of=(["self"]))  # ← 行锁
```

### 3.4 定时更新任务

**执行顺序**：
```
Step 1: Celery Beat 定时触发
        ↓ 配置在 settings.py 中
Step 2: update_products_search_vector_task()
        ↓ product/tasks.py:348-360
Step 3: update_products_search_vector() 批次处理
        ↓ product/search.py:52-74
Step 4: _prep_product_search_vector_index() 生成向量并保存
        ↓ product/search.py:32-49
```

**代码证据 Step 2 - 定时任务**：
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
        .order_by("updated_at")[:PRODUCTS_BATCH_SIZE]  # 每次取300个
        .values_list("id", flat=True)
    )
    with allow_writer():
        update_products_search_vector(products)
```

**代码证据 Step 3-4 - 批次处理并保存**：
`saleor/product/search.py:32-49`
```python
def _prep_product_search_vector_index(products, page_id_to_title_map=None):
    prefetch_related_objects(products, *PRODUCT_FIELDS_TO_PREFETCH)
    
    for product in products:
        product.search_vector = FlatConcatSearchVector(
            *prepare_product_search_vector_value(
                product, already_prefetched=True,
                page_id_to_title_map=page_id_to_title_map,
            )
        )
        product.search_index_dirty = False  # ← 清除脏标记
    
    Product.objects.bulk_update(
        products, ["search_vector", "updated_at", "search_index_dirty"]
    )
```

---

## 四、查询匹配逻辑（带执行顺序和代码证据）

### 4.1 全文搜索（使用search_vector）

**执行顺序**：
```
Step 1: GraphQL Query (products) 入口
        ↓ graphql/product/schema.py
Step 2: resolve_products() 获取基础QuerySet
        ↓ graphql/product/resolvers.py
Step 3: ProductFilter 应用过滤
        ↓ graphql/product/filters/product.py:94-231
Step 4: filter_search() 调用前缀搜索
        ↓ graphql/product/filters/product_helpers.py:324-325
Step 5: prefix_search() 核心搜索算法
        ↓ core/search.py:144-179
Step 6: parse_search_query() 解析搜索语法
        ↓ core/search.py:89-141
Step 7: 返回带search_rank注解的QuerySet
```

**代码证据 Step 3 - ProductFilter定义**：
`saleor/graphql/product/filters/product.py:140`
```python
search = django_filters.CharFilter(method=filter_search)
```

**代码证据 Step 4 - filter_search()**：
`saleor/graphql/product/filters/product_helpers.py:324-325`
```python
def filter_search(qs, _, value):
    return prefix_search(qs, value)  # ← 委托给核心搜索函数
```

**代码证据 Step 5 - prefix_search()**：
`saleor/core/search.py:144-179`
```python
def prefix_search(qs: "QuerySet", value: str) -> "QuerySet":
    if not value:
        return qs.annotate(search_rank=Value(0))
    
    value = strip_accents(value)  # ← 去除重音
    
    parsed_query = parse_search_query(value)  # ← 解析搜索语法
    if not parsed_query:
        return qs.annotate(search_rank=Value(0)).none()
    
    # 前缀查询 - 用于过滤（ broad match ）
    prefix_query = SearchQuery(parsed_query, search_type="raw", config="simple")
    
    # 精确查询 - 仅用于排序（ higher rank for exact matches ）
    exact_query = SearchQuery(value, search_type="websearch", config="simple")
    
    qs = qs.filter(search_vector=prefix_query).annotate(  # ← 使用search_vector过滤
        prefix_rank=SearchRank(F("search_vector"), prefix_query),
        exact_rank=SearchRank(F("search_vector"), exact_query),
        search_rank=F("exact_rank") * 2 + F("prefix_rank"),  # ← 精确匹配权重翻倍
    )
    
    return qs
```

**代码证据 Step 6 - parse_search_query()**：
`saleor/core/search.py:89-141`
```python
def parse_search_query(value: str) -> str | None:
    tokens = _tokenize(value)  # ← 分词处理
    if not tokens:
        return None
    
    parts: list[str] = []
    pending_connector = " & "
    
    for token in tokens:
        if token["type"] == "or":
            pending_connector = " | "
            continue
        
        if parts:
            parts.append(pending_connector)
        pending_connector = " & "
        
        neg = "!" if token["negated"] else ""
        
        if token["type"] == "word":
            parts.append(f"{neg}{token['word']}:*")  # ← 前缀匹配
        
        elif token["type"] == "phrase":
            words = token["words"]
            if len(words) == 1:
                parts.append(f"{neg}{words[0]}")
            else:
                phrase_tsquery = " <-> ".join(words)  # ← 短语匹配（followed-by）
                parts.append(f"!({phrase_tsquery})" if neg else f"({phrase_tsquery})")
    
    return "".join(parts) if parts else None
```

**搜索语法支持**：
- 多词隐式AND：`"coffee shop"` → `coffee:* & shop:*`
- OR运算符：`"coffee OR tea"` → `coffee:* | tea:*`
- 否定：`"-decaf"` → `!decaf:*`
- 引号短语：`'"green tea"'` → `(green <-> tea)`

**评分规则**：
- 精确匹配(websearch)：2x权重
- 前缀匹配：1x权重

### 4.2 属性过滤（不使用search_vector，直接联表）

**执行顺序（新版属性过滤）**：
```
Step 1: ProductFilter.filter_attributes() 入口
        ↓ graphql/product/filters/product.py:164-167
Step 2: filter_products_by_attributes() 路由到新旧版
        ↓ graphql/product/filters/product_attributes.py:976-984
Step 3: _filter_products_by_attributes() 新版逻辑
        ↓ graphql/product/filters/product_attributes.py:851-931
Step 4: 根据值类型分发到具体过滤函数
        ↓ product_attributes.py:893-928
Step 5: filter_by_slug_or_name() / filter_by_numeric_attribute() 等
        ↓ product_attributes.py:339-416
Step 6: _get_assigned_product_attribute_for_attribute_value() 构建Exists查询
        ↓ product_attributes.py:325-336
Step 7: 返回过滤后的QuerySet
```

**代码证据 Step 1 - 入口**：
`saleor/graphql/product/filters/product.py:164-167`
```python
def filter_attributes(self, queryset, name, value):
    if not value:
        return queryset
    return filter_products_by_attributes(queryset, value)
```

**代码证据 Step 2 - 路由新旧版**：
`saleor/graphql/product/filters/product_attributes.py:976-984`
```python
def filter_products_by_attributes(qs, value):
    if not value:
        return qs.none()
    
    # 检测是否使用旧版格式（包含slug和value以外的字段）
    if set(value[0].keys()).difference({"slug", "value"}):
        return deprecated_filter_attributes(qs, value)  # ← 旧版
    return _filter_products_by_attributes(qs, value)    # ← 新版
```

**代码证据 Step 3 - 新版过滤逻辑**：
`saleor/graphql/product/filters/product_attributes.py:851-931`
```python
def _filter_products_by_attributes(qs: QuerySet[Product], value: list[dict]):
    attribute_slugs = {attr_filter["slug"] for attr_filter in value if "slug" in attr_filter}
    attributes_map = {
        attr.slug: attr
        for attr in Attribute.objects.using(qs.db).filter(slug__in=attribute_slugs)
    }
    
    attr_filter_expression = Q()
    
    # 处理没有value的属性（只要有值即可）
    attr_without_values_input = []
    for attr_filter in value:
        if "slug" in attr_filter and "value" not in attr_filter:
            attr_without_values_input.append(attributes_map[attr_filter["slug"]])
    
    if attr_without_values_input:
        atr_value_qs = AttributeValue.objects.using(qs.db).filter(
            attribute_id__in=[attr.id for attr in attr_without_values_input]
        )
        attr_filter_expression = _get_assigned_product_attribute_for_attribute_value(
            atr_value_qs, qs.db
        )
    
    # 处理带value的属性（按类型分发）
    for attr_filter in value:
        attr_value = attr_filter.get("value")
        if not attr_value:
            continue
        
        attr_id = attributes_map[attr_filter["slug"]].id if "slug" in attr_filter else None
        
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
    
    return qs.filter(attr_filter_expression) if attr_filter_expression != Q() else qs.none()
```

**代码证据 Step 5-6 - 具体过滤函数和Exists查询**：
`saleor/graphql/product/filters/product_attributes.py:339-352`
```python
def filter_by_slug_or_name(attr_id, attr_value, db_connection_name):
    attribute_values = get_attribute_values_by_slug_or_name_value(
        attr_id=attr_id, attr_value=attr_value,
        db_connection_name=db_connection_name,
    )
    return _get_assigned_product_attribute_for_attribute_value(
        attribute_values=attribute_values,
        db_connection_name=db_connection_name,
    )
```

`saleor/graphql/product/filters/product_attributes.py:325-336`
```python
def _get_assigned_product_attribute_for_attribute_value(
    attribute_values: QuerySet[AttributeValue], db_connection_name: str,
):
    return Q(
        Exists(
            AssignedProductAttributeValue.objects.using(db_connection_name).filter(
                Exists(attribute_values.filter(id=OuterRef("value_id"))),  # ← 子查询匹配值
                product_id=OuterRef("id"),  # ← 关联到Product
            )
        )
    )
```

**旧版属性过滤（同时支持Product和Variant属性）**：
`saleor/graphql/product/filters/product_attributes.py:200-229`
```python
def filter_products_by_attributes_values(qs, queries):
    filters = []
    for values in queries.values():
        # Product属性过滤
        assigned_product_attribute_values = AssignedProductAttributeValue.objects.using(
            qs.db
        ).filter(value_id__in=values)
        product_attribute_filter = Q(
            Exists(assigned_product_attribute_values.filter(product_id=OuterRef("pk")))
        )
        
        # Variant属性过滤（三层嵌套Exists）
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
        
        filters.append(product_attribute_filter | variant_attribute_filter)  # ← OR合并
    
    return qs.filter(*filters)
```

**关键点**：旧版过滤同时查询Product和Variant的属性，使用`OR`条件合并。新版过滤（`_filter_products_by_attributes`）目前只查Product属性，不查Variant属性。

---

## 五、完整流程时序图（带代码证据）

### 5.1 批量创建Variant时索引更新流程

```
Step 1: GraphQL Mutation (productVariantBulkCreate)
        ↓ saleor/graphql/product/bulk_mutations/product_variant_bulk_create.py:930-981
        perform_mutation()
        ├─ clean_variants() 验证输入 [line 940-942]
        ├─ create_variants() 构造对象 [line 943-945]
        ├─ save_variants() 保存到数据库 [line 963]
        │   ├─ bulk_create(variants_to_create) [line 883]
        │   ├─ AttributeAssignmentMixin.save(variant, attributes) [line 885-886]
        │   └─ 设置default_variant [line 891-893]
        └─ post_save_actions() [line 970]
            └─ product.search_index_dirty = True [line 919-920]
                ↓
Step 2: Celery Beat定时任务
        ↓ saleor/product/tasks.py:348-360
        update_products_search_vector_task()
        ├─ 查询search_index_dirty=True的Product（按updated_at排序，取300个）
        └─ 调用update_products_search_vector(products)
            ↓
Step 3: update_products_search_vector()
        ↓ saleor/product/search.py:52-74
        ├─ 按100个一批处理
        ├─ 预取关联Page的title
        └─ 调用_prep_product_search_vector_index()
            ↓
Step 4: _prep_product_search_vector_index()
        ↓ saleor/product/search.py:32-49
        ├─ prefetch_related_objects() 预取关联数据
        ├─ prepare_product_search_vector_value() 生成搜索向量
        │   ├─ Product.name (权重A)
        │   ├─ Product.description_plaintext (权重C)
        │   ├─ Product属性值 (权重B)
        │   └─ generate_variants_search_vector_value()
        │       ├─ Variant.sku / Variant.name (权重A)
        │       └─ Variant属性值 (权重B)
        ├─ 设置product.search_index_dirty = False
        └─ bulk_update() 保存到数据库
```

### 5.2 属性值更新时索引更新流程

```
Step 1: GraphQL Mutation (attributeValueUpdate)
        ↓ saleor/graphql/attribute/mutations/attribute_value_update.py
        perform_mutation()
        ├─ 更新AttributeValue
        └─ post_save_action()
            ↓
Step 2: _mark_products_search_index_dirty(instance)
        ↓ saleor/graphql/attribute/mutations/attribute_value_update.py:86-110
        ├─ 查找使用该AttributeValue的所有ProductVariant
        ├─ 查找关联的所有Product（且search_index_dirty=False）
        └─ mark_products_search_vector_as_dirty_in_batches()
            ↓
Step 3: mark_products_search_vector_as_dirty_in_batches()
        ↓ saleor/product/utils/search_helpers.py:7-11
        └─ 按1000个一批，调用mark_products_search_vector_as_dirty.delay()
            ↓
Step 4: mark_products_search_vector_as_dirty() 任务执行
        ↓ saleor/product/tasks.py:337-345
        └─ 使用select_for_update行锁，设置search_index_dirty=True
            ↓
Step 5: （后续流程同上，由定时任务处理）
```

### 5.3 搜索查询流程

```
Step 1: GraphQL Query (products)
        ↓ saleor/graphql/product/schema.py
        resolve_products() → 获取基础QuerySet
            ↓
Step 2: ProductFilter 应用过滤条件
        ↓ saleor/graphql/product/filters/product.py:94-231
        ├─ filter_search() → prefix_search() [line 140]
        │   ↓ saleor/core/search.py:144-179
        │   ├─ strip_accents() - 去除重音
        │   ├─ parse_search_query() - 解析搜索语法（AND/OR/NOT/短语）
        │   ├─ 构建prefix_query（前缀匹配，用于过滤）
        │   ├─ 构建exact_query（精确匹配，用于排序）
        │   ├─ qs.filter(search_vector=prefix_query) - 使用索引过滤
        │   └─ annotate(search_rank=...) - 计算评分
        │
        └─ (可选) filter_attributes() [line 164-167]
            ↓ saleor/graphql/product/filters/product_attributes.py:976-984
            filter_products_by_attributes()
            ├─ 检测新旧版格式
            ├─ 新版: _filter_products_by_attributes() - 仅查Product属性
            └─ 旧版: filter_products_by_attributes_values() - 查Product+Variant属性
                ↓
Step 3: 返回最终结果（按search_rank排序）
```

### 5.4 批量删除Variant时索引更新流程（特殊 - 同步更新）

```
Step 1: GraphQL Mutation (productVariantBulkDelete)
        ↓ saleor/graphql/product/bulk_mutations/product_variant_bulk_delete.py:73-159
        perform_mutation()
        ├─ 验证参数
        ├─ select_for_update() 锁定Product和Variant [line 97-114]
        ├─ 删除属性值 [line 116]
        ├─ 删除Channel Listing [line 117-119]
        ├─ 删除Variant [line 120]
        └─ 同步更新search_vector [line 145-156]
            ├─ 对每个Product:
            │   ├─ product.search_vector = FlatConcatSearchVector(
            │   │   *prepare_product_search_vector_value(product)
            │   │ )  ← 同步计算新的搜索向量
            │   ├─ 设置新的default_variant
            │   └─ product.save() 立即保存
            └─ post_save_actions() - 触发webhook
```

**特殊说明**：批量删除时不经过异步流程，而是同步计算并保存search_vector，确保删除后搜索结果立即反映变化。

---

## 六、关键设计特点

### 6.1 读写分离
- **写入路径**：标记`search_index_dirty=True`，快速返回（O(1)单字段更新）
- **更新路径**：定时任务异步批量更新`search_vector`（O(n)批量处理）
- **查询路径**：直接使用`search_vector`进行全文搜索（O(log n)GIN索引）

### 6.2 并发控制
- 使用`select_for_update`行锁避免并发标记冲突 [saleor/product/lock_objects.py:6-7]
- 批量处理减少数据库压力（100-1000条/批）
- 队列机制（Celery）削峰填谷，避免高并发时数据库压力过大

### 6.3 索引分层设计
| 索引类型 | 实现方式 | 用途 | 实时性 |
|---------|---------|------|--------|
| 全文搜索 | `search_vector` + tsvector + GIN索引 | 关键词搜索（名称、描述、属性值） | 最终一致（异步更新） |
| 属性过滤 | 直接联表查询 `AssignedProductAttributeValue` | 按属性值精确筛选 | 强一致（实时查询） |
| Trigram搜索 | `search_document` + gin_trgm_ops索引 | 模糊匹配、拼写容错 | 最终一致 |

### 6.4 权重设计
| 内容 | 权重 | 说明 |
|-----|------|------|
| Product.name | A | 最高优先级 |
| Variant.sku / Variant.name | A | 最高优先级 |
| 属性值（所有类型） | B | 中等优先级 |
| Product.description_plaintext | C | 最低优先级 |

### 6.5 性能优化手段
1. **预取关联数据**：`prefetch_related_objects()` 减少N+1查询 [saleor/product/search.py:35]
2. **批量更新**：`bulk_update()` 减少数据库交互次数 [saleor/product/search.py:47-48]
3. **只读副本**：`using(db_conn)` 查询从副本，降低主库压力 [saleor/product/search.py:53-54]
4. **合理批次大小**：100（索引更新）、1000（脏标记）平衡内存和性能
5. **Exists子查询**：属性过滤使用多层Exists而非JOIN，提高查询效率
6. **行锁保护**：避免并发标记导致的更新丢失

---

## 七、搜索索引一致性边界问题分析

### 7.1 default_variant为空时批量删除同步更新search_vector的条件

#### 问题背景
批量删除Variant时有一个特殊的同步更新逻辑，但只有在特定条件下才会触发。

#### 条件分析
**触发同步更新的条件**：`default_variant__isnull=True`

**代码证据**：
`saleor/graphql/product/bulk_mutations/product_variant_bulk_delete.py:141-156`
```python
# set new product default variant if any has been removed
products = models.Product.objects.order_by("pk").filter(
    pk__in=product_pks, default_variant__isnull=True  # ← 关键条件
)
for product in products:
    product.search_vector = FlatConcatSearchVector(
        *prepare_product_search_vector_value(product)
    )
    product.default_variant = product.variants.first()
    product.save(
        update_fields=[
            "default_variant",
            "search_vector",
            "updated_at",
        ]
    )
```

**条件解读**：
1. 只有当被删除的Variant是该Product的`default_variant`时，`default_variant__isnull`才会为True
2. 如果删除的不是default_variant，**不会触发同步更新**，也**不会标记search_index_dirty**
3. 这意味着批量删除非default_variant时，`search_vector`不会被更新

**对比单个删除的处理**：
`saleor/graphql/product/mutations/product_variant/product_variant_delete.py:52-58`
```python
product = models.Product.objects.get(id=instance.product_id)
product.search_index_dirty = True  # ← 单个删除总是标记脏标
product.save(update_fields=["search_index_dirty"])
# if the product default variant has been removed set the new one
if not product.default_variant:
    product.default_variant = product.variants.first()
    product.save(update_fields=["default_variant", "updated_at"])
```

**不一致风险分析**：
| 场景 | 单个删除 | 批量删除 | 一致性风险 |
|-----|---------|---------|-----------|
| 删除default_variant | 标记脏标 + 同步设置新default | 同步更新search_vector + 设置新default | 低（批量删除更及时） |
| 删除非default_variant | 标记脏标（异步更新） | **无任何操作** | **高**（search_vector包含已删除Variant的信息） |

**边界案例**：
- Product有3个Variant：V1(default)、V2、V3
- 批量删除V2和V3（都不是default）
- 结果：search_vector仍然包含V2和V3的sku/name/属性值，直到下次索引刷新

---

### 7.2 sku或name为空时变体属性归集的分支行为

#### 问题背景
变体的属性归集逻辑中存在一个条件分支，当sku和name都为空时会产生特殊行为。

#### 代码证据
`saleor/product/search.py:99-118`
```python
def generate_variants_search_vector_value(product: "Product"):
    variants = list(product.variants.all()[: settings.PRODUCT_MAX_INDEXED_VARIANTS])
    
    search_vectors = [
        NoValidationSearchVector(
            Value(variant.sku), Value(variant.name), config="simple", weight="A"
        )
        if variant.sku
        else NoValidationSearchVector(Value(variant.name), config="simple", weight="A")
        for variant in variants
        if variant.sku or variant.name  # ← 条件1：过滤掉sku和name都为空的variant
    ]
    if search_vectors:  # ← 条件2：只有当search_vectors非空时才归集属性
        for variant in variants:  # ← 遍历所有variant（包括sku/name为空的）
            search_vectors += generate_attributes_search_vector_value_with_assignment(
                variant.attributes.all()[: settings.PRODUCT_MAX_INDEXED_ATTRIBUTES]
            )
    return search_vectors
```

#### 分支行为分析

**分支1：variant.sku 或 variant.name 非空**
- 该variant会被加入`search_vectors`列表（包含sku和name的搜索向量）
- 由于`search_vectors`非空，会进入属性归集循环
- **所有variant**的属性都会被归集（包括sku/name为空的variant）

**分支2：variant.sku 和 variant.name 都为空**
- 该variant会被`if variant.sku or variant.name`过滤掉
- 但如果有其他variant满足条件1，该variant的**属性仍然会被归集**
- 只有当**所有variant**的sku和name都为空时，`search_vectors`为空，才不会归集任何属性

**边界案例矩阵**：

| 案例 | V1 | V2 | V3 | 结果 |
|-----|----|----|----|------|
| 案例1 | sku="A" | sku="B" | sku="C" | 所有V的sku/name和属性都被归集 |
| 案例2 | sku="A" | (空) | (空) | V1的sku被归集，**所有V的属性都被归集** |
| 案例3 | (空) | (空) | (空) | **无任何数据被归集**（包括所有V的属性） |

**代码行为溯源**：
```python
# 注意：属性归集循环遍历的是 variants（原始列表），不是过滤后的列表
if search_vectors:
    for variant in variants:  # ← 遍历原始列表，包含sku/name为空的variant
        search_vectors += generate_attributes_search_vector_value_with_assignment(...)
```

#### 潜在问题
1. **一致性问题**：案例2中，V2和V3的属性被归集，但它们的sku/name没有被归集
2. **静默丢失**：案例3中，所有属性都静默丢失，没有任何警告或错误
3. **业务影响**：如果业务允许sku和name为空，那么这些Variant的属性无法被搜索到

---

### 7.3 脏标聚合与定时刷新对查询时效性的影响

#### 问题背景
搜索索引采用"脏标标记 + 定时刷新"的异步更新模式，这种模式在不同场景下对查询时效性有不同影响。

#### 配置参数澄清（关键修正）

**⚠️ 重要发现：实际调度间隔是60秒，不是20秒**

| 参数 | 实际用途 | 默认值 | 代码位置 |
|-----|---------|-------|---------|
| `initial_timedelta=60` | **实际调度间隔** - 每60秒检查一次脏数据 | 60秒 | `saleor/core/schedules.py:259` |
| `BEAT_UPDATE_SEARCH_FREQUENCY` | 仅用于设置`expires`（任务过期时间），不控制调度频率 | 20秒 | `saleor/settings.py:651` |
| `PRODUCTS_BATCH_SIZE` | 每次定时任务处理的Product数量 | 300 | `saleor/product/tasks.py:55` |

**代码证据1 - 实际调度间隔定义**：
`saleor/core/schedules.py:258-275`
```python
class product_search_update_schedule(TimeBaseSchedule):
    def __init__(self, initial_timedelta=60, nowfun=None, app=None):  # ← 默认60秒
        # initial_timedelta defaults to 60 seconds, as referencing settings.py variables
        # would require rebuilding the schedule. settings depends on this class instance,
        # leading to a circular import if accessed directly.
        import_path = "saleor.core.schedules.initiated_product_search_update_schedule"
        super().__init__(import_path, initial_timedelta, nowfun, app)

    def are_dirty(self) -> bool:
        from django.conf import settings
        from ..product.models import Product
        return (
            Product.objects.using(settings.DATABASE_CONNECTION_REPLICA_NAME)
            .filter(search_index_dirty=True)
            .exists()
        )
```

**代码证据2 - TimeBaseSchedule调度逻辑**：
`saleor/core/schedules.py:171-185`
```python
def is_due(self, last_run_at):
    last_run_at = self.maybe_make_aware(last_run_at)
    rem_delta = self.remaining_estimate(last_run_at)
    remaining_s = max(rem_delta.total_seconds(), 0)
    if remaining_s == 0:
        remaining_s = self.initial_timedelta.total_seconds()  # ← 60秒

    are_marked_as_dirty = self.are_dirty()
    # 只有当时间到了 AND 有脏数据时才执行
    return schedstate(is_due=are_marked_as_dirty, next=remaining_s)
```

**代码证据3 - BEAT_UPDATE_SEARCH_FREQUENCY仅用于expires**：
`saleor/product/tasks.py:348-350`
```python
@app.task(
    queue=settings.UPDATE_SEARCH_VECTOR_INDEX_QUEUE_NAME,
    expires=settings.BEAT_UPDATE_SEARCH_EXPIRE_AFTER_SEC,  # ← 仅用于过期时间
)
def update_products_search_vector_task():
```

**代码证据4 - CELERY_BEAT_SCHEDULE配置**：
`saleor/settings.py:707-712`
```python
"update-products-search-vectors": {
    "task": "saleor.product.tasks.update_products_search_vector_task",
    # Scheduled task that runs every 60 seconds to check for products
    # requiring a search index rebuild.
    "schedule": initiated_product_search_update_schedule,  # ← 使用60秒间隔的schedule
},
```

---

#### 脏标聚合机制

**聚合场景1：多次修改同一Product**
```
T0: Product.search_index_dirty = False
T1: Variant创建 → search_index_dirty = True（第1次标记）
T2: Variant更新 → search_index_dirty = True（第2次标记，值已为True）
T3: 属性值更新 → search_index_dirty = True（第3次标记，值仍为True）
T4: 定时任务执行 → search_index_dirty = False，search_vector更新

结果：3次标记只触发1次索引更新
```

**代码证据 - 幂等标记**：
`saleor/product/tasks.py:337-345`
```python
def mark_products_search_vector_as_dirty(product_ids: list[int]):
    if not product_ids:
        return
    with transaction.atomic():
        ids = product_qs_select_for_update().filter(pk__in=product_ids).values("id")
        Product.objects.filter(id__in=ids).update(search_index_dirty=True)
```
- 由于是布尔字段，多次UPDATE为True不会产生额外效果
- `select_for_update`确保并发标记时不会丢失更新

**聚合场景2：属性值更新影响大量Product**
```
T0: AttributeValue.name = "红色"
T1: 更新AttributeValue.name = "中国红"
T2: 查询所有使用该属性值的Product（可能1000+个）
T3: 分批标记为脏标（每批1000个）
T4: 定时任务每次处理300个
T5: 需要5轮才能处理完所有Product
```

**代码证据 - 分批标记**：
`saleor/product/utils/search_helpers.py:7-11`
```python
MARK_SEARCH_VECTOR_DIRTY_BATCH_SIZE = 1000

def mark_products_search_vector_as_dirty_in_batches(product_ids: list[int]):
    for i in range(0, len(product_ids), MARK_SEARCH_VECTOR_DIRTY_BATCH_SIZE):
        batch_ids = product_ids[i : i + MARK_SEARCH_VECTOR_DIRTY_BATCH_SIZE]
        mark_products_search_vector_as_dirty.delay(batch_ids)
```

---

#### 时效性分析（修正后）

**场景1：单个Variant创建/更新**
```
时间线：
T=0s:    标记search_index_dirty=True（快速返回，<100ms）
T=1s:    用户搜索 → 命中旧的search_vector（不包含新Variant）
T=60s:   定时任务检查（第1次）
         ↓ 如果刚好在检查后才标记，需要等下一轮
T=120s:  定时任务检查（第2次），发现脏数据，处理300个
T=120.5s: search_vector更新完成
T=121s:  用户搜索 → 命中新的search_vector（包含新Variant）

时效性：延迟约60-120秒（取决于定时任务检查时机）
```

**场景2：批量创建100个Variant（不同Product）**
```
时间线：
T=0s:    批量操作，标记100个Product为脏标
T=60s:   定时任务检查，发现脏数据，处理这100个Product
T=60.2s: 全部更新完成（因为100 < 300，1轮处理完）

时效性：延迟约60-120秒
```

**场景3：属性值更新影响1500个Product**
```
时间线：
T=0s:     属性值更新，分批标记1500个Product为脏标
T=60s:    定时任务触发，处理前300个
T=120s:   定时任务触发，处理300个（累计600）
T=180s:   定时任务触发，处理300个（累计900）
T=240s:   定时任务触发，处理300个（累计1200）
T=300s:   定时任务触发，处理300个（累计1500，全部完成）

时效性：延迟约60-300秒（取决于Product在批次中的位置）
```

**场景4：高并发，积压超过300个**
```
时间线：
T=0s:     积压500个脏Product
T=60s:    定时任务触发，处理前300个（按updated_at排序）
T=120s:   定时任务触发，处理剩余200个 + 新增的300个中的前100个
T=180s:   定时任务触发，处理剩余200个
...

时效性：延迟时间 = 积压数量 / 300 * 60秒
```

---

#### 时效性影响矩阵（修正后）

| 操作类型 | 影响Product数 | 最佳延迟 | 最差延迟 | 备注 |
|---------|--------------|---------|---------|------|
| 单个Variant创建 | 1 | ~60s | ~120s | 正常场景 |
| 批量创建Variant | 100 | ~60s | ~120s | 1轮处理完 |
| 属性值更新 | 1500 | ~60s | ~300s | 需5轮处理 |
| 高并发积压 | 10000 | ~60s | ~2000s | 持续积压 |

---

#### 与属性过滤的对比
| 查询方式 | 时效性 | 一致性 |
|---------|-------|-------|
| 全文搜索（search_vector） | 最终一致（60s+延迟） | 异步更新，可能短暂不一致 |
| 属性过滤（直接联表） | 强一致（实时） | 实时查询，总是一致 |

---

#### 优化建议（修正后）

1. **缩短刷新间隔（需要修改代码）**：
   - 当前间隔硬编码在 `saleor/core/schedules.py:259` 的 `initial_timedelta=60`
   - 改为 `initial_timedelta=10` 或 `initial_timedelta=30`
   - ⚠️ 注意：`BEAT_UPDATE_SEARCH_FREQUENCY` 环境变量**不影响**调度频率

2. **增加批次大小**：
   - 将`PRODUCTS_BATCH_SIZE`从300增加到500-1000（需评估内存使用）

3. **批量删除补全**：
   - 在批量删除非default_variant时，也应标记`search_index_dirty=True`
   - 当前只有删除default_variant时才同步更新search_vector

4. **sku/name空值处理**：
   - 在属性归集前，应确保至少有一个可搜索字段，或单独处理属性归集

5. **监控积压**：
   - 监控`search_index_dirty=True`的Product数量，设置告警阈值
   - 超过阈值时考虑临时增加处理批次或频率

6. **让调度间隔可配置**：
   - 修改`product_search_update_schedule`类，支持从环境变量读取间隔
   - 避免硬编码60秒
