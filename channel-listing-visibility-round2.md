# Saleor Channel 价格过滤与列表/详情可见性 - 深度补充分析

## 一、价格区间过滤：未设置价格记录的处理链路

### 1.1 核心问题：`price` 过滤器有意包含 `price_amount IS NULL` 的记录

**代码位置**：`saleor/graphql/product/filters/product_helpers.py:34`

```python
def filter_products_by_variant_price(qs, channel_slug, price_lte=None, price_gte=None):
    channels = Channel.objects.using(qs.db).filter(slug=channel_slug).values("pk")
    product_variant_channel_listings = ProductVariantChannelListing.objects.using(
        qs.db
    ).filter(Exists(channels.filter(pk=OuterRef("channel_id"))))
    
    if price_lte:
        product_variant_channel_listings = product_variant_channel_listings.filter(
            Q(price_amount__lte=price_lte) | Q(price_amount__isnull=True)  # 关键：包含未定价
        )
    if price_gte:
        product_variant_channel_listings = product_variant_channel_listings.filter(
            Q(price_amount__gte=price_gte) | Q(price_amount__isnull=True)  # 关键：包含未定价
        )
    # ...
```

**设计意图**：未设置价格的变体在价格过滤中被视为"匹配所有区间"，这样产品不会因为某个变体未定价而被排除。但这个设计有重要的边界条件。

### 1.2 两种价格过滤器的行为差异

Saleor 有两套价格过滤逻辑，行为不同：

| 过滤器 | 调用路径 | 处理 `price_amount IS NULL` |
|--------|---------|---------------------------|
| `filter.price` (ObjectTypeFilter) | `ProductFilter.filter_variant_price` → `filter_variant_price` → `filter_products_by_variant_price` | ✅ 包含（`Q(...) \| Q(price_amount__isnull=True)`） |
| `where.price` (WhereInput) | `ProductWhere.filter_variant_price` → `filter_where_by_numeric_field` | ❌ 排除（直接 `price_amount__lte/gte`） |
| `filter.minimal_price` | `ProductFilter.filter_minimal_price` → `filter_minimal_price` → `filter_products_by_minimal_price` | ❌ 排除（仅比较 `discounted_price_amount`） |
| `where.minimal_price` | `ProductWhere.filter_minimal_price` → `filter_where_by_numeric_field` | ❌ 排除（直接 `discounted_price_amount__lte/gte`） |

### 1.3 `filter_where_by_numeric_field` 的严格过滤

**代码位置**：`saleor/graphql/utils/filters.py:147`

```python
def filter_where_by_numeric_field(qs, field, value):
    if not value:
        return qs.none()
    
    if "eq" in value:
        # allow filtering by `None` value
        return qs.filter(**{field: value["eq"]})  # 可以显式查询 null
    
    if range and isinstance(range, dict):
        lte = range.get("lte")
        gte = range.get("gte")
        if lte is not None:
            qs = qs.filter(**{f"{field}__lte": lte})  # 不包含 null
        if gte is not None:
            qs = qs.filter(**{f"{field}__gte": gte})  # 不包含 null
        return qs
```

### 1.4 边界场景分析

假设某产品 P 在渠道 C 有三个变体：
- V1: `price_amount = 50`
- V2: `price_amount = 200`
- V3: `price_amount = NULL`（未设置价格）

**场景 1：使用 `filter: { price: { gte: 100 } }`**
```graphql
query {
  products(filter: { price: { gte: 100 }, channel: "default-channel" }) {
    edges { node { id name } }
  }
}
```
**结果**：✅ 产品 P 会被返回
- V1: 50 < 100 → 不匹配
- V2: 200 >= 100 → 匹配
- V3: NULL → 通过 `Q(price_amount__isnull=True)` 匹配
- 只要有一个变体匹配，产品就被返回

**场景 2：使用 `filter: { price: { lte: 100 } }`**
**结果**：✅ 产品 P 会被返回
- V1: 50 <= 100 → 匹配
- V2: 200 > 100 → 不匹配
- V3: NULL → 匹配

**场景 3：使用 `where: { price: { range: { gte: 100 } } }`**
```graphql
query {
  products(where: { price: { range: { gte: 100 } } }, channel: "default-channel") {
    edges { node { id name } }
  }
}
```
**结果**：⚠️ 产品 P 仍会被返回（但原因不同）
- V1: 50 < 100 → 不匹配
- V2: 200 >= 100 → 匹配
- V3: NULL → 不匹配（where 过滤不包含 null）
- 但 V2 匹配，所以产品仍被返回

**场景 4：产品只有未定价变体**
产品 Q 只有一个变体 V4: `price_amount = NULL`
- `filter: { price: { gte: 100 } }` → ✅ 返回（V4 通过 `IS NULL` 匹配）
- `where: { price: { range: { gte: 100 } } }` → ❌ 不返回（V4 不匹配）

**场景 5：产品所有变体价格都不匹配，但有未定价变体**
产品 R 有变体：V5(price=30), V6(price=NULL)
- `filter: { price: { gte: 100 } }` → ✅ 返回（V6 通过 `IS NULL` 匹配）
- `where: { price: { range: { gte: 100 } } }` → ❌ 不返回

### 1.5 与 `published_with_variants` 的交互

注意：`visible_to_user` 对于普通用户会调用 `published_with_variants`，该方法要求：

**代码位置**：`saleor/product/managers.py:55`

```python
def published_with_variants(self, channel: Channel):
    variant_channel_listings = ProductVariantChannelListing.objects.filter(
        channel_id=channel.id,
        price_amount__isnull=False,  # 关键：必须有价格
    ).values("id")
    # ...
    return self.published(channel).filter(Exists(variants.filter(...)))
```

**重要边界**：产品必须至少有一个变体设置了价格（`price_amount IS NOT NULL`）才能被普通用户看到。

所以**场景 4 实际上不会发生**，因为产品 Q 在 `visible_to_user` 阶段就被过滤掉了。

但**场景 5 仍然可能发生**：产品 R 有 V5(price=30) 满足 `price_amount__isnull=False`，所以通过了 `published_with_variants`，但在价格过滤时 V6(price=NULL) 可能影响结果。

### 1.6 完整价格过滤链路图

```
用户查询（普通用户）
    │
    ▼
visible_to_user → published_with_variants
    │  条件：
    │  - 至少有一个变体 price_amount IS NOT NULL
    │
    ▼
┌─────────────────────────────────────────────┐
│  价格过滤器分支                              │
│                                             │
│  ┌───────────────────────────────────────┐  │
│  │  filter.price (ObjectTypeFilter)      │  │
│  │  条件：                                │  │
│  │  (price_amount >= X AND IS NOT NULL)  │  │
│  │  OR                                  │  │
│  │  (price_amount <= Y AND IS NOT NULL)  │  │
│  │  OR                                  │  │
│  │  price_amount IS NULL                │  │
│  └───────────────────────────────────────┘  │
│                                             │
│  ┌───────────────────────────────────────┐  │
│  │  where.price (WhereInput)             │  │
│  │  条件：                                │  │
│  │  price_amount >= X (不含 NULL)        │  │
│  │  AND/OR                              │  │
│  │  price_amount <= Y (不含 NULL)        │  │
│  └───────────────────────────────────────┘  │
│                                             │
│  ┌───────────────────────────────────────┐  │
│  │  filter.minimal_price / where.minimal │  │
│  │  条件：                                │  │
│  │  直接比较 ProductChannelListing       │  │
│  │  .discounted_price_amount             │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
    │
    ▼
  结果集
```

---

## 二、列表可见与详情可见的分流条件

### 2.1 分流点：`visible_in_listings` 字段的应用时机

`visible_in_listings` 是控制产品是否出现在**列表/搜索/分类**结果中的关键字段，但**不影响**通过直接链接访问产品详情页。

### 2.2 列表查询链路：强制要求 `visible_in_listings = True`

**代码位置**：`saleor/graphql/product/resolvers.py:116`

```python
@traced_resolver
def resolve_products(
    info: ResolveInfo,
    requestor,
    channel: Channel | None,
    limited_channel_access: bool,
) -> ChannelQsContext:
    connection_name = get_database_connection_name(info.context)
    qs = models.Product.objects.using(connection_name).visible_to_user(
        requestor, channel, limited_channel_access
    )
    
    # 普通用户额外过滤：仅显示 visible_in_listings=True 的产品
    if not has_one_of_permissions(requestor, ALL_PRODUCTS_PERMISSIONS):
        if channel:
            product_channel_listings = (
                models.ProductChannelListing.objects.using(connection_name)
                .filter(channel_id=channel.id, visible_in_listings=True)  # 关键
                .values("id")
            )
            qs = qs.filter(
                Exists(product_channel_listings.filter(product_id=OuterRef("pk")))
            )
        else:
            qs = models.Product.objects.none()
    
    return ChannelQsContext(qs=qs, channel_slug=channel.slug if channel else None)
```

**注意**：`visible_in_listings` 过滤是在 `visible_to_user` 之后**额外**添加的。

### 2.3 详情查询链路：不要求 `visible_in_listings`

**代码位置**：`saleor/graphql/product/resolvers.py:88`

```python
def resolve_product(
    info: ResolveInfo,
    id, slug, slug_language_code, external_reference,
    channel: Channel | None,
    limited_channel_access: bool,
    requestor,
):
    database_connection_name = get_database_connection_name(info.context)
    qs = models.Product.objects.using(database_connection_name).visible_to_user(
        requestor, channel, limited_channel_access
    )
    # 没有 visible_in_listings 过滤！
    
    if id:
        _type, id = from_global_id_or_error(id, "Product")
        return qs.filter(id=id).first()
    # ... slug / external_reference 同理
```

### 2.4 四个查询入口的完整对比

| 查询入口 | 解析器 | visible_to_user 基础 | 额外可见性过滤 |
|---------|-------|---------------------|---------------|
| 产品列表 | `resolve_products` | `ProductsQueryset.visible_to_user` | 普通用户需 `visible_in_listings=True` |
| 产品详情 | `resolve_product` | `ProductsQueryset.visible_to_user` | ❌ 无 |
| 变体列表 | `resolve_product_variants` | `ProductVariantQueryset.visible_to_user` | 普通用户需 `visible_in_listings=True`（见下方） |
| 变体详情 | `resolve_variant` | `Product.visible_to_user` + `available_in_channel` | ❌ 无 |

### 2.5 变体查询的特殊处理

#### 变体列表查询 (`resolve_product_variants`)

**代码位置**：`saleor/graphql/product/resolvers.py:188`

```python
def resolve_product_variants(...):
    qs = models.ProductVariant.objects.using(connection_name).visible_to_user(
        requestor, channel, limited_channel_access
    )
```

调用的是 `ProductVariantQueryset.visible_to_user`，该方法对普通用户有额外要求：

**代码位置**：`saleor/product/managers.py:316`

```python
def visible_to_user(self, requestor, channel, limited_channel_access):
    # ... 管理员逻辑 ...
    
    # 普通用户
    if not channel or not channel.is_active:
        return self.none()
    
    variants = self.filter(
        channel_listings__channel_id=channel.id,
        channel_listings__price_amount__isnull=False,
    )
    
    today = datetime.datetime.now(tz=datetime.UTC)
    variants = variants.filter(
        Q(product__channel_listings__published_at__lte=today)
        | Q(product__channel_listings__published_at__isnull=True),
        product__channel_listings__is_published=True,
        product__channel_listings__channel_id=channel.id,
        product__channel_listings__visible_in_listings=True,  # 关键：列表可见
    )
    return variants
```

#### 单个变体查询 (`resolve_variant`)

**代码位置**：`saleor/graphql/product/resolvers.py:157`

```python
def resolve_variant(...):
    visible_products = (
        models.Product.objects.using(connection_name)
        .visible_to_user(requestor, channel, limited_channel_access)
        .values_list("pk", flat=True)
    )
    qs = models.ProductVariant.objects.using(connection_name).filter(
        product__id__in=visible_products
    )
    if not requestor_has_access_to_all:
        qs = qs.available_in_channel(channel)  # 仅要求 price_amount IS NOT NULL
```

调用的是 `available_in_channel`，只要求变体有价格，不要求 `visible_in_listings`：

**代码位置**：`saleor/product/managers.py:297`

```python
def available_in_channel(self, channel: Channel | None):
    if not channel:
        return self.none()
    channel_listings = (
        ProductVariantChannelListing.objects.using(self.db)
        .filter(price_amount__isnull=False, channel_id=channel.id)  # 仅要求有价格
        .values("id")
    )
    return self.filter(Exists(channel_listings.filter(variant_id=OuterRef("pk"))))
```

### 2.6 边界场景分析

假设产品 P 在渠道 C 中的配置：
- `is_published = True`
- `published_at = NULL`（立即可见）
- `visible_in_listings = False`（不在列表中显示）
- 变体 V1 有价格 `price_amount = 100`

**场景 1：普通用户查询产品列表**
```graphql
query {
  products(channel: "default-channel", first: 10) {
    edges { node { id name } }
  }
}
```
**结果**：❌ 产品 P 不返回
- 通过 `visible_to_user`（已发布且有可售变体）
- 但 `resolve_products` 额外过滤 `visible_in_listings=True` 不满足

**场景 2：普通用户通过 ID 查询产品详情**
```graphql
query {
  product(id: "UHJvZHVjdDox", channel: "default-channel") {
    id name
  }
}
```
**结果**：✅ 产品 P 返回
- 仅通过 `visible_to_user` 检查，不检查 `visible_in_listings`

**场景 3：普通用户查询产品的变体列表**
```graphql
query {
  product(id: "UHJvZHVjdDox", channel: "default-channel") {
    variants { id name }
  }
}
```
**结果**：✅ 变体 V1 返回
- 产品详情已通过可见性检查
- 变体解析使用 `VariantsByProductIdAndChannel` dataloader，内部逻辑需确认

**场景 4：普通用户直接查询变体列表**
```graphql
query {
  productVariants(channel: "default-channel", first: 10) {
    edges { node { id name } }
  }
}
```
**结果**：❌ 变体 V1 不返回
- `resolve_product_variants` 使用 `ProductVariantQueryset.visible_to_user`
- 该方法要求 `product__channel_listings__visible_in_listings=True`

**场景 5：普通用户通过 SKU 查询单个变体**
```graphql
query {
  productVariant(sku: "SKU-001", channel: "default-channel") {
    id name
  }
}
```
**结果**：✅ 变体 V1 返回
- `resolve_variant` 使用 `Product.visible_to_user` + `available_in_channel`
- 不检查 `visible_in_listings`

### 2.7 完整分流流程图

```
查询入口
    │
    ├───────────────────────────────────────────┐
    │                                           │
    ▼                                           ▼
产品列表/搜索                            产品详情（ID/Slug）
resolve_products                          resolve_product
    │                                           │
    │                                           ├─ visible_to_user
    │                                           │  (published_with_variants)
    │                                           │  - is_published = True
    │                                           │  - published_at <= now
    │                                           │  - 至少一个变体有价格
    │                                           │
    │                                           ▼
    │                                         返回产品（无 visible_in_listings 检查）
    │
    ├─ visible_to_user
    │  (published_with_variants)
    │
    ├─ 额外过滤（普通用户）
    │  └─ visible_in_listings = True
    │
    ▼
  列表结果


变体列表                                   变体详情（ID/SKU）
resolve_product_variants                   resolve_variant
    │                                           │
    │                                           ├─ Product.visible_to_user
    │                                           │  （检查产品可见性）
    │                                           │
    │                                           ├─ available_in_channel
    │                                           │  - 变体 price_amount IS NOT NULL
    │                                           │
    │                                           ▼
    │                                         返回变体
    │
    ├─ ProductVariantQueryset.visible_to_user
    │  - 变体 price_amount IS NOT NULL
    │  - 产品 is_published = True
    │  - 产品 published_at <= now
    │  - 产品 visible_in_listings = True  ← 关键区别
    │
    ▼
  变体列表结果
```

---

## 三、管理员 vs 普通用户的权限边界

### 3.1 管理员权限豁免

拥有以下任一权限的用户视为管理员：
- `ProductPermissions.MANAGE_PRODUCTS`
- `OrderPermissions.MANAGE_ORDERS`
- `DiscountPermissions.MANAGE_DISCOUNTS`

**管理员豁免规则**：
1. `visible_to_user` 不调用 `published_with_variants`，可查看所有产品
2. `resolve_products` 不添加 `visible_in_listings=True` 过滤
3. `ProductVariantQueryset.visible_to_user` 不添加 `visible_in_listings=True` 过滤
4. 可查看 `cost_price`（成本价）

### 3.2 特殊场景：产品部分变体未定价

产品 P 有三个变体：
- V1: price = 100, visible_in_listings = False
- V2: price = 200, visible_in_listings = False
- V3: price = NULL, visible_in_listings = False

**普通用户查询 `filter: { price: { gte: 150 } }`**：
1. `visible_to_user` → 通过（V1、V2 有价格）
2. `visible_in_listings = True` → 通过（产品级别设置）
3. 价格过滤：
   - V1: 100 < 150 → 不匹配
   - V2: 200 >= 150 → 匹配
   - V3: NULL → 匹配（filter 模式）
4. ✅ 产品返回

**普通用户查询 `where: { price: { range: { gte: 150 } } }`**：
1-2. 同上
3. 价格过滤：
   - V1: 100 < 150 → 不匹配
   - V2: 200 >= 150 → 匹配
   - V3: NULL → 不匹配（where 模式）
4. ✅ 产品仍返回（因 V2 匹配）

**变体 V3 本身在 `visible_to_user` 中的处理**：
- 变体列表查询：V3 会被排除（`price_amount__isnull=False`）
- 单个变体查询：V3 会被排除（`available_in_channel` 要求 `price_amount__isnull=False`）
- 所以用户永远看不到未定价的变体，无论价格过滤如何

---

## 四、关键代码位置速查表（补充）

| 功能 | 文件位置 |
|------|---------|
| 变体价格过滤（含 null） | `saleor/graphql/product/filters/product_helpers.py:34` |
| Where 价格过滤（不含 null） | `saleor/graphql/utils/filters.py:147` |
| 最低价格过滤 | `saleor/graphql/product/filters/product_helpers.py:60` |
| 产品列表解析器（列表可见过滤） | `saleor/graphql/product/resolvers.py:116` |
| 产品详情解析器（无列表可见过滤） | `saleor/graphql/product/resolvers.py:88` |
| 变体列表解析器（列表可见过滤） | `saleor/graphql/product/resolvers.py:188` |
| 变体详情解析器（无列表可见过滤） | `saleor/graphql/product/resolvers.py:157` |
| 产品变体列表可见过滤 | `saleor/product/managers.py:316` |
| 变体检出（仅检查价格） | `saleor/product/managers.py:297` |
