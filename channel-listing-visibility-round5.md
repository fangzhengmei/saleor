# Saleor Channel 可见性深度分析（第五轮 - 一致性修正版）

## 修正说明

本轮仅针对上一轮（round4）文档中**场景 11（管理员按价格过滤变体**的冲突结论进行一致性修正，其他内容保持不变。

### 冲突点定位

round4 第 746 行与第 749 行存在矛盾：
- ❌ 第 746 行（错误）："变体列表包含 V2 (price=200)，但不包含 V1 (price=100)"
- ✅ 第 749 行（正确）："变体列表返回该产品的所有变体（因为 `productVariants` 内部没有价格过滤）"

矛盾在于：前一句说"不包含 V1"，后一句又说"返回所有变体"。

---

## 一、`visible_in_listings` 开关的数据层级与生效位置

### 1.1 数据层级：产品级，而非变体级

**明确定义**：`visible_in_listings` 是 **产品级** 字段，定义在 `ProductChannelListing` 模型中。

**代码位置**：`saleor/product/models.py:319`

```python
class ProductChannelListing(PublishableModel):
    product = models.ForeignKey(Product, ...)
    channel = models.ForeignKey(Channel, ...)
    visible_in_listings = models.BooleanField(default=False)  # ← 产品级开关
    
    class Meta:
        unique_together = [["product", "channel"]]  # 每个产品在每个渠道只有一条记录
```

**结论**：不存在"变体级别的 `visible_in_listings`"。一个产品在某个渠道要么在列表中可见，要么不可见，对该产品的所有变体生效。

### 1.2 生效位置全景图

`visible_in_listings` 检查仅出现在**列表类查询**中，**详情类查询**完全不检查。

| 查询类型 | GraphQL 示例 | 是否检查 `visible_in_listings` | 代码位置 |
|---------|------------|--------------------------|---------|
| **产品列表查询** | `products(channel: "...") { ... }` | ✅ 是 | `resolvers.py:126` 普通用户额外过滤 |
| **独立变体列表查询** | `productVariants(channel: "...") { ... }` | ✅ 是 | `managers.py:354` `visible_to_user` 内检查 |
| **产品内分页变体** | `product { productVariants { ... } }` | ✅ 是 | 同上，通过 `product_id` 内部调用 |
| **产品详情查询** | `product(id: "...", channel: "...") { ... }` | ❌ 否 | `resolvers.py:88` 仅调用 `visible_to_user` |
| **单个变体查询** | `productVariant(sku: "...", channel: "...") { ... }` | ❌ 否 | `resolvers.py:157` 仅检查 `available_in_channel` |
| **产品内简单变体** | `product { variants { ... } }` (已废弃) | ❌ 否 | Dataloader 仅检查价格存在 |

### 1.3 生效位置 1：产品列表查询 (`resolve_products`)

**代码位置**：`saleor/graphql/product/resolvers.py:116`

```python
@traced_resolver
def resolve_products(info, requestor, channel, limited_channel_access):
    qs = models.Product.objects.using(connection_name).visible_to_user(
        requestor, channel, limited_channel_access
    )
    
    # ↓↓↓ 关键：visible_in_listings 过滤在此处添加 ↓↓↓
    if not has_one_of_permissions(requestor, ALL_PRODUCTS_PERMISSIONS):
        if channel:
            product_channel_listings = (
                models.ProductChannelListing.objects.using(connection_name)
                .filter(channel_id=channel.id, visible_in_listings=True)
                .values("id")
            )
            qs = qs.filter(
                Exists(product_channel_listings.filter(product_id=OuterRef("pk")))
```

### 1.4 生效位置 2：变体列表查询 (`ProductVariantQueryset.visible_to_user`)

**代码位置**：`saleor/product/managers.py:316`

```python
def visible_to_user(self, requestor, channel, limited_channel_access):
    if has_one_of_permissions(requestor, ALL_PRODUCTS_PERMISSIONS):
        return self.all()  # 管理员豁免
    
    if not channel or not channel.is_active:
        return self.none()
    
    # 普通用户：变体必须有价格
    variants = self.filter(
        channel_listings__channel_id=channel.id,
        channel_listings__price_amount__isnull=False,
    )
    
    today = datetime.datetime.now(tz=datetime.UTC)
    # ↓↓↓ 关键：visible_in_listings 过滤在此处 ↓↓↓
    variants = variants.filter(
        Q(product__channel_listings__published_at__lte=today)
        | Q(product__channel_listings__published_at__isnull=True),
        product__channel_listings__is_published=True,
        product__channel_listings__channel_id=channel.id,
        product__channel_listings__visible_in_listings=True,  # 明确要求 True
    )
    return variants
```

**这个方法被两个入口调用**：
1. 顶级 `productVariants` 查询（`resolve_product_variants` 解析器）
2. 产品详情内 `product { productVariants { ... } }` 分页查询（通过内部 `product_id` 参数调用）

### 1.5 不生效的位置：Dataloader 变体查询

**代码位置**：`saleor/graphql/product/dataloaders/products.py:212`

```python
class ProductVariantsByProductIdAndChannel(DataLoader):
    def get_variants_filter(self, channel_id: int):
        variant_channel_listings = ProductVariantChannelListing.objects.filter(
            channel_id=channel_id
        )
        return Q(Exists(variant_channel_listings.filter(variant_id=OuterRef("id"))))
        # ↑↑↑ 仅检查变体有 channel listing，不检查 visible_in_listings ↑↑↑

class AvailableProductVariantsByProductIdAndChannel(
    ProductVariantsByProductIdAndChannel
):
    def get_variants_filter(self, channel_id: int):
        variant_channel_listings = ProductVariantChannelListing.objects.filter(
            channel_id=channel_id, price_amount__isnull=False  # 仅增加价格存在检查
        )
        return Q(Exists(variant_channel_listings.filter(variant_id=OuterRef("id"))))
        # ↑↑↑ 仍然不检查 visible_in_listings ↑↑↑
```

**被调用场景**：
- `product { variants { ... } }`（已废弃的简单变体列表）
- `product { isAvailable }` 内部检查变体可用性

---

## 二、产品详情内变体解析 vs 独立变体列表查询

### 2.1 三个变体查询入口的完整对比

Saleor 有三种方式查询产品变体，它们的可见性检查逻辑和过滤能力不同：

| 查询方式 | GraphQL 示例 | 解析入口 | `visible_in_listings` 检查 | 可过滤字段 |
|---------|------------|---------|--------------------------|-----------|
| **简单变体列表** (已废弃) | `product { variants { id name } }` | `Product.resolve_variants` → Dataloader | ❌ 不检查 | 无（固定返回所有有价格的变体） |
| **产品内分页变体** | `product { productVariants(first: 10) { edges { ... } }` | `Product.resolve_product_variants` → `resolve_product_variants(product_id=...)` → `visible_to_user` | ✅ 检查 | `search`, `sku`, `is_preorder`, `updated_at`, `attributes` (where) |
| **独立变体列表** | `productVariants(channel: "...") { edges { ... } }` | `resolve_product_variants()` → `visible_to_user` | ✅ 检查 | 同上，另加 `ids` 参数 |

### 2.2 分流点 1：简单变体列表 (`product { variants }`)

**代码位置**：`saleor/graphql/product/types/products.py:1521`

```python
@staticmethod
def resolve_variants(root: ChannelContext[models.Product], info):
    requestor = get_user_or_app_from_context(info.context)
    has_required_permissions = has_one_of_permissions(
        requestor, ALL_PRODUCTS_PERMISSIONS
    )
    # ...
    else:
        # ↓↓↓ 普通用户走这个分支 ↓↓↓
        variants = AvailableProductVariantsByProductIdAndChannel(info.context).load(
            (root.node.id, root.channel_slug)
```

**普通用户检查逻辑**：
```python
def get_variants_filter(self, channel_id: int):
    variant_channel_listings = ProductVariantChannelListing.objects.filter(
        channel_id=channel_id, 
        price_amount__isnull=False  # 唯一条件：变体价格存在
    )
    return Q(Exists(variant_channel_listings.filter(variant_id=OuterRef("id"))))
```

**不检查**：
- ❌ 不检查 `visible_in_listings`
- ❌ 不支持任何用户指定的过滤条件

### 2.3 分流点 2：产品内分页变体 (`product { productVariants }`)

**代码位置**：`saleor/graphql/product/types/products.py:1546`

```python
@staticmethod
def resolve_product_variants(root: ChannelContext[models.Product], info, **kwargs):
    requestor = get_user_or_app_from_context(info.context)
    
    def _resolve_product_variants(channel_obj):
        limited_channel_access = False if channel_slug is None else True
        # ↓↓↓ 调用 resolve_product_variants，传入内部 product_id 参数 ↓↓↓
        qs = resolve_product_variants(
            info,
            channel=channel_obj,
            product_id=root.node.pk,  # ← 内部参数，不对外暴露
            limited_channel_access=limited_channel_access,
            requestor=requestor,
        )
        kwargs["channel"] = qs.channel_slug
        qs = filter_connection_queryset(qs, kwargs, allow_replica=...)
        return create_connection_slice(qs, info, kwargs, ProductVariantCountableConnection)
```

这个方法调用 `resolve_product_variants` 解析器函数，最终调用 `ProductVariantQueryset.visible_to_user`，**包含** `visible_in_listings` 检查。

### 2.4 分流点 3：独立变体列表 (`productVariants`)

**代码位置**：`saleor/graphql/product/resolvers.py:188`

```python
@traced_resolver
def resolve_product_variants(
    info, requestor, ids=None, channel=None, product_id=None, limited_channel_access=False
):
    connection_name = get_database_connection_name(info.context)
    
    # ↓↓↓ 调用 visible_to_user，包含 visible_in_listings 检查 ↓↓↓
    qs = models.ProductVariant.objects.using(connection_name).visible_to_user(
        requestor, channel, limited_channel_access
    )
    
    if ids:
        db_ids = [from_global_id_or_error(node_id, "ProductVariant")[1] for node_id in ids]
        qs = qs.filter(pk__in=db_ids)
    
    if product_id:  # ← 仅内部调用时使用，外部无法通过 GraphQL 指定
        qs = qs.filter(product_id=product_id)
    
    return ChannelQsContext(qs=qs, channel_slug=channel.slug if channel else None)
```

### 2.5 分流逻辑总结图

```
GraphQL 变体查询
        │
        ├─────────────────────────────────────────────────┐
        │                                                 │
        ▼                                                 ▼
product { variants { ... } }                  product { productVariants { ... } }
（简单列表，已废弃）                             （支持过滤/分页）
        │                                                 │
        │                                                 ├─ 内部调用 resolve_product_variants
        │                                                 │   (product_id = root.node.pk)
        │                                                 │
        │                                                 ├─ visible_to_user
        │                                                 │   (managers.py:316)
        │                                                 │   - is_published = True
        │                                                 │   - published_at <= now
        │                                                 │   - visible_in_listings = True  ✅
        │                                                 │   - price_amount IS NOT NULL
        │                                                 │
        │                                                 ▼
        └─ AvailableProductVariantsByProductIdAndChannel         结果集
           (dataloader)
           - 仅检查 price_amount IS NOT NULL
           - 不检查 visible_in_listings  ❌
           - 不支持用户过滤
        │
        ▼
      结果集
        │
        │
        └─────────────────────────────────────────────────┐
                                                          │
                                                          ▼
                                            productVariants(channel: "...") { ... }
                                            （独立变体列表查询）
                                                          │
                                                          ├─ resolve_product_variants()
                                                          │   (无 product_id 参数)
                                                          │
                                                          ├─ visible_to_user
                                                          │   (同上，包含 visible_in_listings ✅)
                                                          │
                                                          ▼
                                                        结果集
                                                          │
                                                          ├─ 支持过滤：ids, search, sku, 
                                                          │  is_preorder, updated_at, attributes
                                                          │
                                                          └─ ❌ 不支持：product, price 过滤
```

---

## 三、"查询特定产品的变体"的正确方式

由于 `productVariants` 查询不支持 `product` 过滤参数，要查询特定产品的变体有两种合法方式：

### 方式 1：产品详情内嵌套查询（推荐）

```graphql
query {
  product(id: "UHJvZHVjdDox", channel: "default-channel") {
    id name
    productVariants(first: 10) {  # ← 自动限定当前产品
      edges {
        node { id name sku }
      }
    }
  }
}
```

**特点**：
- ✅ 自动限定产品，无需额外参数
- ✅ 支持过滤、排序、分页
- ✅ 包含 `visible_in_listings` 检查
- ✅ 语义清晰，符合 GraphQL 嵌套查询设计

### 方式 2：通过 SKU 过滤（如果知道 SKU 模式）

```graphql
query {
  productVariants(
    channel: "default-channel",
    filter: { sku: ["SKU-V1", "SKU-V2"] }  # ← 精确指定变体SKU列表
  ) {
    edges { node { id name sku } }
  }
}
```

或者使用 `where` 参数：
```graphql
query {
  productVariants(
    channel: "default-channel",
    where: { sku: { oneOf: ["SKU-V1", "SKU-V2"] }
  ) {
    edges { node { id name sku } }
  }
}
```

### 方式 3：通过 IDs 精确查询

```graphql
query {
  productVariants(
    channel: "default-channel",
    ids: ["UHJvZHVjdFZhcmlhbnQ6MTAw", "UHJvZHVjdFZhcmlhbnQ6MTAx"]  # ← 变体的GlobalID
  ) {
    edges { node { id name sku } }
  }
}
```

### 方式 4：通过 `search` 模糊匹配（不精确）

```graphql
query {
  productVariants(
    channel: "default-channel",
    filter: { search: "产品P" }  # ← 搜索变体名称、SKU或所属产品名称
  ) {
    edges { node { id name sku } }
  }
}
```

**注意**：`search` 会匹配变体名称、SKU 和**所属产品的名称**，可能返回其他产品的变体。

---

## 四、"按价格过滤变体"的正确方式

`productVariants` 查询**不支持**直接按价格过滤。要实现按价格过滤变体，需要通过产品层间接实现：

### 方式 1：先过滤产品，再查询变体（推荐）

```graphql
query {
  products(
    channel: "default-channel",
    filter: { price: { gte: 150 } }  # ← 产品级价格过滤
    first: 10
  ) {
    edges {
      node {
        id name
        productVariants {  # ← 再查询该产品的所有变体
          edges { node { id name sku } }
        }
      }
    }
  }
}
```

**说明**：产品级 `price` 过滤检查的是**变体基准价** (`price_amount`)，只要产品有一个变体的价格在范围内就返回该产品。

### 方式 2：按产品最低折扣价过滤

```graphql
query {
  products(
    channel: "default-channel",
    filter: { minimal_price: { gte: 150 } }  # ← 按产品最低折扣价过滤
    first: 10
  ) {
    edges {
      node {
        id name
        productVariants {
          edges { node { id name sku } }
        }
      }
    }
  }
}
```

**说明**：`minimal_price` 检查的是 `ProductChannelListing.discounted_price_amount`（产品所有变体折扣价的最小值）。

### 方式 3：使用 `where` 精确价格过滤

```graphql
query {
  products(
    channel: "default-channel",
    where: { price: { range: { gte: 150, lte: 300 } } }  # ← 不含 NULL 的严格过滤
    first: 10
  ) {
    edges {
      node {
        id name
        productVariants {
          edges { node { id name sku } }
        }
      }
    }
  }
}
```

**区别**：`where.price` 不包含 `price_amount IS NULL` 的记录，而 `filter.price` 包含。

---

## 五、边界场景示例（修正版）

### 场景设定

产品 P 在渠道 C 中的配置：
- `ProductChannelListing.is_published = True`
- `ProductChannelListing.published_at = NULL`（立即可见）
- `ProductChannelListing.visible_in_listings = False`  ← 关键：不在列表中显示
- 变体 V1: `id=100`, `sku="V1"`, `price_amount = 100`
- 变体 V2: `id=101`, `sku="V2"`, `price_amount = 200`

**前提**：产品 P 可以通过直接链接访问（因为 `is_published = True`）。
**变体 GlobalID**：`UHJvZHVjdFZhcmlhbnQ6MTAw` (V1), `UHJvZHVjdFZhcmlhbnQ6MTAx` (V2)

---

### 场景 1：普通用户查询产品列表

```graphql
query {
  products(channel: "default-channel", first: 10) {
    edges { node { id name } }
  }
}
```

**结果**：❌ 产品 P 不返回
- `resolve_products` 额外过滤 `visible_in_listings=True` 不满足

---

### 场景 2：普通用户通过 ID 访问产品详情

```graphql
query {
  product(id: "UHJvZHVjdDox", channel: "default-channel") {
    id name
  }
}
```

**结果**：✅ 产品 P 返回
- `resolve_product` 仅调用 `visible_to_user`，不检查 `visible_in_listings`

---

### 场景 3：普通用户在产品详情页查询简单变体列表（已废弃）

```graphql
query {
  product(id: "UHJvZHVjdDox", channel: "default-channel") {
    id name
    variants {  # ← 简单变体列表（已废弃）
      id name sku
    }
  }
}
```

**结果**：✅ V1、V2 都返回
- 使用 `AvailableProductVariantsByProductIdAndChannel` dataloader
- 仅检查 `price_amount__isnull=False`，**不检查** `visible_in_listings`

---

### 场景 4：普通用户在产品详情页查询分页变体列表

```graphql
query {
  product(id: "UHJvZHVjdDox", channel: "default-channel") {
    id name
    productVariants(first: 10) {  # ← 分页变体列表
      edges { node { id name sku } }
    }
  }
}
```

**结果**：❌ 变体列表为空
- 内部调用 `resolve_product_variants(product_id=product.pk)` → `visible_to_user`
- `visible_to_user` 要求 `product__channel_listings__visible_in_listings=True`，不满足

---

### 场景 5：普通用户通过独立变体列表查询特定产品的变体（修正版）

**错误写法**（上一轮的错误）：
```graphql
# ❌ 错误：ProductVariantFilter 没有 product 字段
query {
  productVariants(
    channel: "default-channel", 
    filter: { product: "UHJvZHVjdDox" }
  ) {
    edges { node { id name sku } }
  }
}
```

**正确写法 1**（推荐，使用嵌套查询）：
```graphql
# ✅ 正确：通过产品详情嵌套查询，自动限定产品
query {
  product(id: "UHJvZHVjdDox", channel: "default-channel") {
    productVariants(first: 10) {
      edges { node { id name sku } }
    }
  }
}
```

**正确写法 2**（通过 SKU 精确过滤）：
```graphql
# ✅ 正确：通过 SKU 列表过滤
query {
  productVariants(
    channel: "default-channel",
    filter: { sku: ["V1", "V2"] }
  ) {
    edges { node { id name sku } }
  }
}
```

**正确写法 3**（通过 IDs 精确过滤）：
```graphql
# ✅ 正确：通过变体 GlobalID 列表过滤
query {
  productVariants(
    channel: "default-channel",
    ids: ["UHJvZHVjdFZhcmlhbnQ6MTAw", "UHJvZHVjdFZhcmlhbnQ6MTAx"]
  ) {
    edges { node { id name sku } }
  }
}
```

**结果**（无论哪种正确写法）：❌ 变体列表为空
- 都走 `visible_to_user` 检查，要求 `visible_in_listings=True`，不满足

---

### 场景 6：普通用户通过 SKU 直接查询单个变体

```graphql
query {
  productVariant(sku: "V1", channel: "default-channel") {
    id name sku
  }
}
```

**结果**：✅ V1 返回
- `resolve_variant` 使用 `Product.visible_to_user` + `available_in_channel`
- `available_in_channel` 仅检查 `price_amount__isnull=False`，**不检查** `visible_in_listings`

---

### 场景 7：管理员执行上述所有查询

**结果**：✅ 所有查询都返回完整数据
- 管理员豁免 `visible_in_listings` 检查
- 管理员豁免 `is_published` 检查

---

### 场景 8：用户搜索产品（带关键词）

```graphql
query {
  products(
    channel: "default-channel",
    filter: { search: "产品P" }
  ) {
    edges { node { id name } }
  }
}
```

**结果**：❌ 产品 P 不返回
- 搜索属于列表查询范畴，走 `resolve_products` 入口
- 包含 `visible_in_listings=True` 检查

---

### 场景 9：产品 `visible_in_listings = True`，但某个变体未设置价格

产品配置同上，但 V2 的 `price_amount = NULL`。

```graphql
query {
  product(id: "UHJvZHVjdDox", channel: "default-channel") {
    variants { id name sku }
  }
}
```

**结果**：✅ 仅返回 V1
- `AvailableProductVariantsByProductIdAndChannel` 检查 `price_amount__isnull=False`
- V2 被过滤掉，因为没有价格

---

### 场景 10：变体价格过滤 + `visible_in_listings = False`（修正版）

**错误写法**（上一轮的错误）：
```graphql
# ❌ 错误：ProductVariantFilter 没有 price 字段
query {
  productVariants(
    channel: "default-channel",
    filter: { price: { gte: 150 } }
  ) {
    edges { node { id name sku } }
  }
}
```

**正确写法**（先过滤产品，再查变体）：
```graphql
# ✅ 正确：通过产品级价格过滤，再查询变体
query {
  products(
    channel: "default-channel",
    filter: { price: { gte: 150 } }  # ← 产品级价格过滤
  ) {
    edges {
      node {
        id name
        productVariants {
          edges { node { id name sku price { amount } }
        }
      }
    }
  }
}
```

**结果**：❌ 产品 P 不返回，因此也没有变体
- 产品级 `filter.price` 在 `resolve_products` 中执行
- `resolve_products` 先检查 `visible_in_listings=True`，不满足，在价格过滤之前就被过滤掉了

**注意**：`visible_in_listings` 检查在查询链的**最上游**，优先级高于价格过滤。

---

### 场景 11：管理员按价格过滤变体（一致性修正版）

```graphql
query {
  products(
    channel: "default-channel",
    filter: { price: { gte: 150 } }
  ) {
    edges {
      node {
        id name
        productVariants {
          edges { node { id name sku price { amount } }
        }
      }
    }
  }
}
```

**结果**：✅ 产品 P 返回，变体列表**同时包含 V1 (price=100) 和 V2 (price=200)
- 管理员豁免 `visible_in_listings` 检查
- 产品级价格过滤：V2 的价格 `200 >= 150` 匹配，因此产品 P 通过产品级过滤被返回
- 变体列表返回该产品的**所有**变体（因为 `productVariants` 内部没有价格过滤）

**代码链路推导说明**：

1. **产品级过滤链路**（`products(filter: { price: { gte: 150 } })`）：
   - `resolve_products` → `ProductsQueryset.visible_to_user` → 管理员豁免 `visible_in_listings` 检查
   - 产品级 `filter.price`（`filter_products_by_variant_price`）：
     ```python
     if price_gte:
         product_variant_channel_listings = product_variant_channel_listings.filter(
             Q(price_amount__gte=price_gte) | Q(price_amount__isnull=True)
         )
     ```
   - V2 的 `price_amount = 200 >= 150` 匹配，因此产品 P 被返回

2. **变体列表链路**（`product { productVariants { ... } }`）：
   - `Product.resolve_product_variants` → `resolve_product_variants(product_id=product.pk)` → `ProductVariantQueryset.visible_to_user`
   - 管理员在 `visible_to_user` 中走管理员分支（`managers.py:326`）：
     ```python
     if has_one_of_permissions(requestor, ALL_PRODUCTS_PERMISSIONS):
         if limited_channel_access:
             if channel:
                 return self.filter(product__channel_listings__channel_id=channel.id)
     ```
   - 这只是过滤"该产品在该渠道有 channel listing 的变体，**没有任何价格过滤**
   - 只要变体在该渠道有 channel listing（V1 和 V2 都有），就会被返回
   - 因此 V1 (price=100) 和 V2 (price=200) 都会被返回

**结论一致性验证**：产品级价格过滤仅影响"哪些产品被返回"，不影响"返回产品的哪些变体被返回"。变体级过滤需要额外的变体级过滤条件（但 `ProductVariantFilter` 没有 `price` 字段）。

**如果只想返回价格 >= 150 的变体**，需要在应用层过滤结果，或者使用更精确的查询方式。

---

## 六、查询链路优先级与关键注意点

### 6.1 过滤优先级（从先到后）

```
查询入口
    │
    ▼
1. 权限判断（管理员 vs 普通用户）
    │
    ▼
2. 渠道激活检查（channel.is_active）
    │
    ▼
3. 产品发布状态检查（is_published, published_at）
    │
    ▼
4. 列表可见检查（visible_in_listings）← 仅列表查询，优先级最高
    │
    ▼
5. 变体价格存在检查（price_amount IS NOT NULL）
    │
    ▼
6. 用户指定的过滤器（search, sku, is_preorder, attributes, 等）
    │
    ▼
  结果集
```

### 6.2 关键注意点

1. **`visible_in_listings` 是产品级开关**，不是变体级。一个产品在某个渠道要么在列表中可见，要么不可见。

2. **`visible_in_listings` 仅影响列表类查询**：
   - ✅ 影响：`products`、`productVariants`、`product { productVariants }`
   - ❌ 不影响：`product`、`productVariant`、`product { variants }`

3. **两个变体字段的区别**：
   - `product { variants }`（简单列表，已废弃）：不检查 `visible_in_listings`，不支持过滤
   - `product { productVariants }`（分页列表）：检查 `visible_in_listings`，支持过滤/排序/分页

4. **`productVariants` 独立查询不支持 `product` 和 `price` 过滤**：
   - 要查询特定产品的变体：使用 `product { productVariants }` 嵌套查询
   - 要按价格过滤变体：先通过 `products(filter: { price: ... })` 过滤产品，再查询变体

5. **`visible_in_listings` 检查在查询链最上游**，优先级高于价格过滤、搜索过滤等用户指定的过滤条件。

6. **产品详情可见，但变体列表可能不可见**：当 `visible_in_listings=False` 时，用户可以访问产品详情页，但 `product { productVariants }` 分页查询会返回空。

7. **`product_id` 是内部参数**：`resolve_product_variants` 函数有 `product_id` 参数，但仅用于 `product { productVariants }` 内部调用，不对外暴露在 GraphQL schema 中。

8. **产品级过滤与变体级过滤是分离的**：产品级价格过滤仅决定哪些产品被返回，变体列表返回产品的所有符合可见变体，不受产品级价格过滤的影响。

---

## 七、关键代码位置速查表

| 功能 | 文件位置 |
|------|---------|
| `visible_in_listings` 字段定义 | `saleor/product/models.py:319` |
| `ProductVariantFilter` 字段定义 | `saleor/graphql/product/filters/product_variant.py:697` |
| `ProductVariantWhere` 字段定义 | `saleor/graphql/product/filters/product_variant.py:720` |
| 产品列表可见性过滤 | `saleor/graphql/product/resolvers.py:126` |
| 变体列表可见性过滤 | `saleor/product/managers.py:354` |
| 产品详情内简单变体解析 | `saleor/graphql/product/types/products.py:1521` |
| 产品详情内分页变体解析 | `saleor/graphql/product/types/products.py:1546` |
| 独立变体列表解析器 | `saleor/graphql/product/resolvers.py:188` |
| 变体检出 Dataloader（不检查列表可见） | `saleor/graphql/product/dataloaders/products.py:259` |
| 变体检出（仅检查价格） | `saleor/product/managers.py:297` |
| 产品级价格过滤 | `saleor/graphql/product/filters/product_helpers.py:34` |
| 产品级最低价格过滤 | `saleor/graphql/product/filters/product_helpers.py:60` |
| 管理员变体可见性（`visible_to_user` 管理员分支） | `saleor/product/managers.py:326` |
