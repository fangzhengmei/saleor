# Saleor 产品属性链路分析

## 概述

Saleor 的产品属性系统构成了一条从**定义 → 绑定 → 搜索过滤**的完整数据链路。本文档梳理各模块间的衔接关系、属性类型差异、索引更新触发时机，以及属性变更后的兼容处理机制。

---

## 1. 属性定义模型与类型系统

### 1.1 核心模型

| 模型 | 文件位置 | 职责 |
|------|----------|------|
| `Attribute` | `saleor/attribute/models/base.py:116` | 属性元数据定义，包含类型、输入类型、过滤配置等 |
| `AttributeValue` | `saleor/attribute/models/base.py:365` | 属性值存储，支持多种数据类型字段 |
| `AttributeProduct` | `saleor/attribute/models/product.py:35` | 属性与产品类型的关联表（产品级属性） |
| `AttributeVariant` | `saleor/attribute/models/product_variant.py:55` | 属性与产品类型的关联表（变体级属性） |

### 1.2 属性类型分类

定义于 `saleor/attribute/__init__.py`:

**AttributeType（大类）**:
- `PRODUCT_TYPE` - 产品类型属性
- `PAGE_TYPE` - 页面类型属性

**AttributeInputType（输入类型）**:
| 类型 | 可用于变体选择 | 支持选项 | 唯一值 | 可翻译 |
|------|--------------|--------|--------|--------|
| `DROPDOWN` | ✅ | ✅ | ❌ | ❌ |
| `MULTISELECT` | ❌ | ✅ | ❌ | ❌ |
| `SWATCH` | ✅ | ✅ | ❌ | ❌ |
| `BOOLEAN` | ✅ | ❌ | ✅ | ❌ |
| `NUMERIC` | ✅ | ❌ | ✅ | ❌ |
| `FILE` | ❌ | ❌ | ✅ | ❌ |
| `REFERENCE` | ❌ | ❌ | ✅ | ❌ |
| `SINGLE_REFERENCE` | ❌ | ❌ | ✅ | ❌ |
| `RICH_TEXT` | ❌ | ❌ | ✅ | ✅ |
| `PLAIN_TEXT` | ❌ | ❌ | ✅ | ✅ |
| `DATE` | ❌ | ❌ | ✅ | ❌ |
| `DATE_TIME` | ❌ | ❌ | ✅ | ❌ |

**属性配置能力**（`ATTRIBUTE_PROPERTIES_CONFIGURATION`）:
- `filterable_in_storefront` - 可在 Storefront 过滤
- `filterable_in_dashboard` - 可在 Dashboard 过滤
- `available_in_grid` - 可在网格视图显示
- `storefront_search_position` - 搜索优先级

### 1.3 AttributeValue 多字段设计

`AttributeValue` 模型为不同输入类型设计了专用存储字段：
- `name` - 显示名称（通用）
- `value` - 颜色值（#RRGGBBAA 格式）
- `slug` - 唯一标识
- `file_url` / `content_type` - 文件类型
- `rich_text` / `plain_text` - 文本类型
- `boolean` - 布尔类型
- `date_time` - 日期时间类型
- `numeric` - 数值类型
- `reference_product` / `reference_variant` / `reference_page` / `reference_category` / `reference_collection` - 引用类型

---

## 2. 属性与商品/变体的绑定关系

### 2.1 绑定模型

| 模型 | 文件位置 | 说明 |
|------|----------|------|
| `AssignedProductAttributeValue` | `saleor/attribute/models/product.py:9` | 产品级属性值绑定 |
| `AssignedVariantAttribute` | `saleor/attribute/models/product_variant.py:35` | 变体级属性绑定（中间层） |
| `AssignedVariantAttributeValue` | `saleor/attribute/models/product_variant.py:8` | 变体级属性值绑定 |

### 2.2 绑定关系图

```
Attribute (定义)
    ├─── AttributeProduct (M2M through) ──── ProductType
    │                                          └─── Product
    │                                               └─── AssignedProductAttributeValue ──── AttributeValue
    │
    └─── AttributeVariant (M2M through) ──── ProductType
                                               └─── Product
                                                    └─── ProductVariant
                                                         └─── AssignedVariantAttribute
                                                              └─── AssignedVariantAttributeValue ──── AttributeValue
```

### 2.3 属性赋值流程

核心入口：`AttributeAssignmentMixin` (`saleor/graphql/attribute/utils/attribute_assignment.py:51`)

**赋值流程**:
1. **输入解析** (`clean_input`):
   - 解析 GraphQL 输入中的属性 ID/外部引用
   - 验证属性存在性和权限
   - 根据 `input_type` 选择对应的类型处理器

2. **类型处理器映射** (`HANDLER_MAPPING`):
   ```python
   DROPDOWN/SWATCH → SelectableAttributeHandler
   MULTISELECT → MultiSelectableAttributeHandler
   FILE → FileAttributeHandler
   REFERENCE/SINGLE_REFERENCE → ReferenceAttributeHandler
   RICH_TEXT → RichTextAttributeHandler
   PLAIN_TEXT → PlainTextAttributeHandler
   NUMERIC → NumericAttributeHandler
   DATE/DATE_TIME → DateTimeAttributeHandler
   BOOLEAN → BooleanAttributeHandler
   ```

3. **预保存处理** (`pre_save_values`):
   - 根据输入类型决定批量操作类型（CREATE/UPDATE_OR_CREATE/GET_OR_CREATE）
   - 对唯一值类型属性生成唯一 slug

4. **批量操作执行** (`_bulk_create_pre_save_values`):
   - 执行 `bulk_create` / `bulk_update_or_create` / `bulk_get_or_create`
   - 使用 `AttributeValueManager` 中的自定义批量方法

5. **绑定到实例** (`associate_attribute_values_to_instance`):
   - 调用 `saleor/attribute/utils.py:32` 中的核心绑定函数
   - 先删除旧的绑定，再创建新的绑定
   - 维护 `sort_order` 排序

---

## 3. 搜索索引构建机制

### 3.1 索引触发点

`Product.search_index_dirty` 字段 (`saleor/product/models.py:184`) 作为索引更新标记。

**设置 dirty 的场景**:

| 操作 | 文件位置 |
|------|----------|
| 产品创建 | `saleor/graphql/product/mutations/product/product_create.py:172` |
| 产品更新 | `saleor/graphql/product/mutations/product/product_update.py:113` |
| 变体创建 | `saleor/graphql/product/mutations/product_variant/product_variant_create.py:247` |
| 变体更新 | `saleor/graphql/product/mutations/product_variant/product_variant_update.py:216` |
| 变体删除 | `saleor/graphql/product/mutations/product_variant/product_variant_delete.py:53` |
| 变体批量创建 | `saleor/graphql/product/bulk_mutations/product_variant_bulk_create.py:919` |
| 变体批量更新 | `saleor/graphql/product/bulk_mutations/product_variant_bulk_update.py:771` |
| 属性删除 | `saleor/graphql/attribute/mutations/attribute_delete.py:79` |
| 属性值删除 | `saleor/graphql/attribute/mutations/attribute_value_delete.py:66` |

### 3.2 索引更新流程

**定时任务触发**: `update_products_search_vector_task` (`saleor/product/tasks.py:352`)
- 队列: `settings.UPDATE_SEARCH_VECTOR_INDEX_QUEUE_NAME`
- 批量处理: 每次最多 300 个产品（按 `updated_at` 排序）
- 调用 `update_products_search_vector()` 执行实际更新

**索引构建逻辑** (`saleor/product/search.py`):

1. **主函数** `_prep_product_search_vector_index()`:
   - 预加载关联数据（变体、属性、属性值等）
   - 调用 `prepare_product_search_vector_value()` 构建搜索向量
   - 批量更新 `search_vector` 并清除 `search_index_dirty` 标记

2. **搜索向量组成** (`prepare_product_search_vector_value()`):
   - 产品名称（权重 A）
   - 产品描述（权重 C）
   - 产品属性值（权重 B）- `generate_attributes_search_vector_value()`
   - 变体 SKU/名称/属性（权重 A/B）- `generate_variants_search_vector_value()`

3. **属性值向量化** (`get_search_vectors_for_attribute_values()`):
   | 输入类型 | 索引内容 |
   |----------|----------|
   | DROPDOWN/MULTISELECT | `value.name` |
   | RICH_TEXT | `editorjs_to_text(value.rich_text)` |
   | PLAIN_TEXT | `value.plain_text` |
   | NUMERIC | `value.name + " " + unit` |
   | DATE/DATE_TIME | `value.date_time.strftime("%Y-%m-%d %H:%M:%S")` |
   | REFERENCE | 仅支持 Page 类型，索引页面标题 |

**索引限制**:
- 最多索引 `PRODUCT_MAX_INDEXED_ATTRIBUTES` 个属性
- 每个属性最多索引 `PRODUCT_MAX_INDEXED_ATTRIBUTE_VALUES` 个值
- 每个产品最多索引 `PRODUCT_MAX_INDEXED_VARIANTS` 个变体

---

## 4. 属性过滤机制

### 4.1 过滤入口

核心函数: `filter_products_by_attributes()` (`saleor/graphql/product/filters/product_attributes.py:976`)

**过滤流程**:
1. 检查输入格式，判断是否使用旧版过滤语法
2. 新版过滤使用 `value` 字段，支持多种过滤方式
3. 旧版过滤使用 `slug` + `values`/`values_range`/`boolean`/`date`/`date_time`

### 4.2 新版过滤能力

| 过滤方式 | 说明 |
|----------|------|
| `slug` + `name` | 按属性值 slug 或名称过滤 |
| `numeric` | 数值范围过滤（支持 `gte`/`lte`/`eq` 等） |
| `boolean` | 布尔值过滤 |
| `date` / `date_time` | 日期范围过滤 |
| `reference` | 引用类型过滤，支持: |
| - `referenced_ids` | 按引用对象 ID 过滤 |
| - `page_slugs` / `product_slugs` | 按引用对象 slug 过滤 |
| - `product_variant_skus` | 按变体 SKU 过滤 |
| - `category_slugs` / `collection_slugs` | 按分类/合集 slug 过滤 |

引用类型支持 `contains_all`（全部匹配）和 `contains_any`（任一匹配）逻辑。

### 4.3 过滤实现原理

以产品属性为例，过滤通过多层 `Exists` 子查询实现：

```
ProductQuerySet
  └── Exists(AssignedProductAttributeValue
        ├── filter(value_id__in=匹配的属性值ID)
        └── filter(product_id=OuterRef("pk"))
      )
  OR
  └── Exists(ProductVariant
        └── Exists(AssignedVariantAttribute
              └── Exists(AssignedVariantAttributeValue
                    └── filter(value_id__in=匹配的属性值ID)
                  )
            )
      )
```

---

## 5. 属性变更与删除的兼容处理

### 5.1 属性删除处理

**入口**: `AttributeDelete.perform_mutation()` (`saleor/graphql/attribute/mutations/attribute_delete.py:58`)

**处理流程**:
1. **收集受影响的产品/页面 ID**:
   - `get_product_ids_to_search_index_update()` - 通过产品类型关联找到所有相关产品
   - `get_page_ids_to_search_index_update()` - 通过页面类型关联找到所有相关页面

2. **原子删除**:
   - 使用 `attribute_value_qs_select_for_update()` 锁定属性值防止并发修改
   - 先删除所有 `AttributeValue`（级联删除绑定关系）
   - 再删除 `Attribute` 本身

3. **索引更新**:
   - 调用 `mark_products_search_vector_as_dirty_in_batches()` 标记产品需要重建索引
   - 分批处理（每批 1000 个），通过 Celery 任务异步执行

### 5.2 属性值删除处理

**入口**: `AttributeValueDelete.perform_mutation()` (`saleor/graphql/attribute/mutations/attribute_value_delete.py:54`)

**处理流程**:
1. **收集受影响 ID**:
   - `get_product_ids_to_search_index_update_for_attribute_values()` - 查找使用该属性值的产品/变体
   - 逻辑：`AssignedVariantAttributeValue` → `AssignedVariantAttribute` → `ProductVariant` → `Product`
   - 加上直接通过 `AssignedProductAttributeValue` 关联的产品

2. **执行删除**:
   - 调用父类 `ModelDeleteMutation` 执行删除
   - 级联删除所有 `Assigned*AttributeValue` 绑定

3. **索引更新**:
   - 同样使用批量 dirty 标记机制

### 5.3 属性值更新处理

**索引触发**: `get_product_ids_to_search_index_update_for_attribute_values()` 同样适用于属性值更新场景。

### 5.4 批量操作优化

**批量更新/创建** (`AttributeValueManager`):
- `bulk_get_or_create()` - 批量获取或创建
- `bulk_update_or_create()` - 批量更新或创建
- 使用 `select_for_update` 锁定行防止并发冲突（`saleor/attribute/models/base.py:338-345`）

**属性赋值优化**:
- `associate_attribute_values_to_instance()` 使用 `bulk_create(ignore_conflicts=True)` 避免重复绑定
- 先删除旧绑定，再批量创建新绑定

---

## 6. 关键数据流时序

### 6.1 产品创建时的属性链路

```
GraphQL productCreate 突变
    ↓
AttributeAssignmentMixin.clean_input()
    ├── 验证属性存在性
    └── 根据 input_type 选择处理器
    ↓
AttributeAssignmentMixin.pre_save_values()
    ├── 为唯一值类型生成 AttributeValue
    └── 执行批量数据库操作
    ↓
AttributeAssignmentMixin.save()
    └── associate_attribute_values_to_instance()
        ├── 删除旧的 AssignedProductAttributeValue
        └── 批量创建新的绑定关系
    ↓
设置 product.search_index_dirty = True
    ↓
Celery 定时任务 update_products_search_vector_task
    └── 构建 search_vector 并清除 dirty 标记
```

### 6.2 属性删除时的索引更新链路

```
GraphQL attributeDelete 突变
    ↓
get_product_ids_to_search_index_update()
    ├── 找到关联的 ProductType
    └── 找到所有使用该类型的 Product
    ↓
删除 Attribute（级联删除 AttributeValue 和 Assigned*）
    ↓
mark_products_search_vector_as_dirty_in_batches()
    └── 分批调用 Celery 任务 mark_products_search_vector_as_dirty
        └── 批量设置 Product.search_index_dirty = True
    ↓
Celery 定时任务处理 dirty 产品，重建索引
```

---

## 7. 核心代码位置速查表

| 功能 | 文件位置 |
|------|----------|
| 属性模型定义 | `saleor/attribute/models/base.py` |
| 产品属性绑定 | `saleor/attribute/models/product.py` |
| 变体属性绑定 | `saleor/attribute/models/product_variant.py` |
| 属性赋值核心逻辑 | `saleor/graphql/attribute/utils/attribute_assignment.py` |
| 属性绑定到实例 | `saleor/attribute/utils.py` |
| 产品搜索索引构建 | `saleor/product/search.py` |
| 属性搜索向量化 | `saleor/attribute/search.py` |
| 产品搜索索引任务 | `saleor/product/tasks.py` |
| 属性过滤逻辑 | `saleor/graphql/product/filters/product_attributes.py` |
| 属性删除突变 | `saleor/graphql/attribute/mutations/attribute_delete.py` |
| 属性值删除突变 | `saleor/graphql/attribute/mutations/attribute_value_delete.py` |
| 索引更新辅助函数 | `saleor/graphql/attribute/mutations/utils.py` |
| 批量标记 dirty | `saleor/product/utils/search_helpers.py` |
