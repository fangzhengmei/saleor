# Saleor Channel 可见性深度分析（第三轮）

## 一、`visible_in_listings` 开关的数据层级与生效位置

### 1.1 数据层级：产品级，而非变体级

**明确定义**：`visible_in_listings` 是 **产品级** 字段，定义在 `ProductChannelListing` 模型中。

**代码位置**：`saleor/product/models.py:319`

```python
class ProductChannelListing(PublishableModel):
    product = models.ForeignKey(Product, ...)
    channel = models.ForeignKey(Channel, ...)
    visible_in_listings = models.BooleanField(default=False)  # ← 产品级开关
    # ...
```

**数据库约束**：`unique_together = [["product", "channel"]]`，确保每个产品在每个渠道只有一条 `ProductChannelListing` 记录。

**结论**：不存在"变体级别的 `visible_in_listings`"。一个产品在某个渠道要么在列表中可见，要么不可见，对该产品的所有变体生效。

### 1.2 生效位置全景图

`visible_in_listings` 检查仅出现在**列表类查询**中，**详情类查询**完全不检查。

| 查询类型 | 代码位置 | 是否检查 `visible_in_listings` | 检查逻辑 |
|---------|---------|--------------------------|---------|
| **产品列表查询** `products` | `saleor/graphql/product/resolvers.py:126` | ✅ 是 | 普通用户额外过滤 `visible_in_listings=True` |
| **变体列表查询** `productVariants` | `saleor/product/managers.py:354` | ✅ 是 | `product__channel_listings__visible_in_listings=True` |
| **产品详情查询** `product` | `saleor/graphql/product/resolvers.py:88` | ❌ 否 | 仅调用 `visible_to_user`，无额外过滤 |
| **单个变体查询** `productVariant` | `saleor/graphql/product/resolvers.py:157` | ❌ 否 | 仅检查 `available_in_channel`（价格存在） |
| **产品详情内变体** `product { variants }` | `AvailableProductVariantsByProductIdAndChannel` | ❌ 否 | dataloader 仅检查变体价格存在 |
| **产品详情内变体（分页）** `product { productVariants }` | `resolve_product_variants` | ✅ 是 | 调用 `visible_to_user`，包含列表可见检查 |

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
                .filter(channel_id=channel.id, visible_in_listings=True)  # 明确要求 True
                .values("id")
            )
            qs = qs.filter(
                Exists(product_channel_listings.filter(product_id=OuterRef("pk")))
            )
        else:
            qs = models.Product.objects.none()
    
    return ChannelQsContext(qs=qs, channel_slug=channel.slug if channel else None)
```

**关键点**：
- 这是在 `visible_to_user` 基础上**额外**添加的过滤
- 仅对**普通用户**生效，管理员豁免
- 检查的是 `ProductChannelListing.visible_in_listings` 字段

### 1.4 生效位置 2：变体列表查询 (`ProductVariantQueryset.visible_to_user`)

**代码位置**：`saleor/product/managers.py:316`

```python
def visible_to_user(self, requestor, channel, limited_channel_access):
    if has_one_of_permissions(requestor, ALL_PRODUCTS_PERMISSIONS):
        # 管理员逻辑...
        return self.all()
    
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

**关键点**：
- 通过 `product__channel_listings__visible_in_listings` 跨表关联检查
- 检查的仍然是**产品级**的 `visible_in_listings` 字段
- 这个方法被两个入口调用：
  1. 顶级 `productVariants` 查询（`resolve_product_variants` 解析器）
  2. 产品详情内 `product { productVariants { ... } }` 分页查询

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

Saleor 有三种方式查询产品变体，它们的可见性检查逻辑不同：

| 查询方式 | GraphQL 示例 | 解析入口 | 普通用户可见性检查 |
|---------|------------|---------|------------------|
| **简单变体列表** | `product { variants { id name } }` | `Product.resolve_variants` | 变体有价格即可，**不检查** `visible_in_listings` |
| **产品内分页变体** | `product { productVariants(first: 10) { edges { node { ... } } } }` | `Product.resolve_product_variants` → `resolve_product_variants` → `visible_to_user` | 检查 `visible_in_listings=True` |
| **独立变体列表** | `productVariants(filter: { product: "..." }) { edges { ... } }` | `resolve_product_variants` → `visible_to_user` | 检查 `visible_in_listings=True` |

### 2.2 分流点 1：简单变体列表 (`product { variants }`)

**代码位置**：`saleor/graphql/product/types/products.py:1521`

```python
@staticmethod
def resolve_variants(root: ChannelContext[models.Product], info):
    requestor = get_user_or_app_from_context(info.context)
    has_required_permissions = has_one_of_permissions(
        requestor, ALL_PRODUCTS_PERMISSIONS
    )
    if has_required_permissions and not root.channel_slug:
        variants = ProductVariantsByProductIdLoader(info.context).load(root.node.id)
    elif has_required_permissions and root.channel_slug:
        variants = ProductVariantsByProductIdAndChannel(info.context).load(
            (root.node.id, root.channel_slug)
        )
    else:
        # ↓↓↓ 普通用户走这个分支 ↓↓↓
        variants = AvailableProductVariantsByProductIdAndChannel(info.context).load(
            (root.node.id, root.channel_slug)
        )
    
    def map_channel_context(variants):
        return [
            ChannelContext(node=variant, channel_slug=root.channel_slug)
            for variant in variants
        ]
    
    return variants.then(map_channel_context)
```

**普通用户检查逻辑**（`AvailableProductVariantsByProductIdAndChannel`）：
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
- ❌ 不检查产品 `is_published`（但产品本身已经通过 `visible_to_user` 检查）

### 2.3 分流点 2：产品内分页变体 (`product { productVariants }`)

**代码位置**：`saleor/graphql/product/types/products.py:1546`

```python
@staticmethod
def resolve_product_variants(root: ChannelContext[models.Product], info, **kwargs):
    requestor = get_user_or_app_from_context(info.context)
    
    def _resolve_product_variants(channel_obj):
        limited_channel_access = False if channel_slug is None else True
        # ↓↓↓ 调用独立变体列表查询的同一个解析器 ↓↓↓
        qs = resolve_product_variants(
            info,
            channel=channel_obj,
            product_id=root.node.pk,
            limited_channel_access=limited_channel_access,
            requestor=requestor,
        )
        # ... 过滤、分页 ...
        return create_connection_slice(
            qs, info, kwargs, ProductVariantCountableConnection
        )
    
    if channel_slug := root.channel_slug:
        return (
            ChannelBySlugLoader(info.context)
            .load(channel_slug)
            .then(_resolve_product_variants)
        )
    return _resolve_product_variants(None)
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
    
    if product_id:
        qs = qs.filter(product_id=product_id)  # 可选：限定产品
    
    # ...
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
        │                                                 ├─ 调用 resolve_product_variants
        │                                                 │   (resolvers.py:188)
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
        │
        ▼
      结果集
```

---

## 三、边界场景示例

### 场景设定

产品 P 在渠道 C 中的配置：
- `ProductChannelListing.is_published = True`
- `ProductChannelListing.published_at = NULL`（立即可见）
- `ProductChannelListing.visible_in_listings = False`  ← 关键：不在列表中显示
- 变体 V1: `sku="V1"`, `price_amount = 100`
- 变体 V2: `sku="V2"`, `price_amount = 200`

**前提**：产品 P 可以通过直接链接访问（因为 `is_published = True`）。

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

### 场景 3：普通用户在产品详情页查询简单变体列表

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
- 调用 `resolve_product_variants` → `visible_to_user`
- `visible_to_user` 要求 `product__channel_listings__visible_in_listings=True`，不满足

---

### 场景 5：普通用户直接查询独立变体列表（限定产品）

```graphql
query {
  productVariants(
    channel: "default-channel", 
    filter: { product: "UHJvZHVjdDox" }
  ) {
    edges { node { id name sku } }
  }
}
```

**结果**：❌ 变体列表为空
- 同场景 4，调用 `visible_to_user`，检查 `visible_in_listings`

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

### 场景 10：变体价格过滤 + `visible_in_listings = False`

```graphql
query {
  productVariants(
    channel: "default-channel",
    filter: { price: { gte: 150 } }
  ) {
    edges { node { id name sku } }
  }
}
```

**结果**：❌ 不返回 V2
- 虽然 V2 的价格 `200 >= 150` 且 `filter.price` 包含 `IS NULL` 逻辑
- 但 `visible_to_user` 先执行，要求 `visible_in_listings=True`，在价格过滤之前就被过滤掉了

**注意**：`visible_in_listings` 检查在查询链的**最上游**，优先级高于价格过滤。

---

## 四、查询链路优先级与关键注意点

### 4.1 过滤优先级（从先到后）

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
4. 列表可见检查（visible_in_listings）← 仅列表查询
    │
    ▼
5. 变体价格存在检查（price_amount IS NOT NULL）
    │
    ▼
6. 用户指定的过滤器（price, stock, 等）
    │
    ▼
  结果集
```

### 4.2 关键注意点

1. **`visible_in_listings` 是产品级开关**，不是变体级。一个产品在某个渠道要么在列表中可见，要么不可见。

2. **`visible_in_listings` 仅影响列表类查询**：
   - ✅ 影响：`products`、`productVariants`、`product { productVariants }`
   - ❌ 不影响：`product`、`productVariant`、`product { variants }`

3. **两个变体字段的区别**：
   - `product { variants }`（简单列表，已废弃）：不检查 `visible_in_listings`
   - `product { productVariants }`（分页列表）：检查 `visible_in_listings`

4. **`visible_in_listings` 检查在查询链最上游**，优先级高于价格过滤、库存过滤等用户指定的过滤条件。

5. **产品详情可见，但变体列表可能不可见**：当 `visible_in_listings=False` 时，用户可以访问产品详情页，但 `product { productVariants }` 分页查询会返回空。

---

## 五、关键代码位置速查表

| 功能 | 文件位置 |
|------|---------|
| `visible_in_listings` 字段定义 | `saleor/product/models.py:319` |
| 产品列表可见性过滤 | `saleor/graphql/product/resolvers.py:126` |
| 变体列表可见性过滤 | `saleor/product/managers.py:354` |
| 产品详情内简单变体解析 | `saleor/graphql/product/types/products.py:1521` |
| 产品详情内分页变体解析 | `saleor/graphql/product/types/products.py:1546` |
| 独立变体列表解析器 | `saleor/graphql/product/resolvers.py:188` |
| 变体检出 Dataloader（不检查列表可见） | `saleor/graphql/product/dataloaders/products.py:259` |
| 变体检出（仅检查价格） | `saleor/product/managers.py:297` |
