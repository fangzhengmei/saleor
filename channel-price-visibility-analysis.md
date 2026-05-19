# Saleor 渠道筛选三层贯通机制分析

## 概述

渠道（Channel）是 Saleor 电商系统中的核心概念，决定了商品的**价格**、**库存**与**上架可见性**。渠道筛选机制在三个层次上贯通运作：

1. **GraphQL 入参层**：接收并验证 `channel` 参数，将其封装为上下文对象向下传递
2. **价格计算层**：基于渠道信息获取定价、计算折扣
3. **可购买性判定层**：基于渠道信息判断可见性、库存可用性

---

## 一、核心数据模型

### 1.1 Channel 模型 (`saleor/channel/models.py:12`)

```python
class Channel(ModelWithMetadata):
    name = models.CharField(max_length=250)
    is_active = models.BooleanField(default=False)
    slug = models.SlugField(max_length=255, unique=True)
    currency_code = models.CharField(max_length=...)
    default_country = CountryField()
    allocation_strategy = models.CharField(...)  # 库存分配策略
    # ... 其他配置字段
```

### 1.2 商品渠道关联模型

**ProductChannelListing** (`saleor/product/models.py:304`) - 产品级渠道配置：
```python
class ProductChannelListing(PublishableModel):
    product = models.ForeignKey(Product, ...)
    channel = models.ForeignKey(Channel, ...)
    visible_in_listings = models.BooleanField(default=False)  # 列表页可见性
    available_for_purchase_at = models.DateTimeField(...)    # 可购买时间
    currency = models.CharField(...)
    discounted_price_amount = models.DecimalField(...)       # 产品级折扣价
    discounted_price_dirty = models.BooleanField(default=False)

    class Meta:
        unique_together = [["product", "channel"]]  # 联合唯一约束
```

**ProductVariantChannelListing** (`saleor/product/models.py:477`) - 变体级渠道配置：
```python
class ProductVariantChannelListing(models.Model):
    variant = models.ForeignKey(ProductVariant, ...)
    channel = models.ForeignKey(Channel, ...)
    currency = models.CharField(...)
    price_amount = models.DecimalField(...)          # 基准价
    cost_price_amount = models.DecimalField(...)     # 成本价
    prior_price_amount = models.DecimalField(...)    # 历史价
    discounted_price_amount = models.DecimalField(...)  # 折扣价

    class Meta:
        unique_together = [["variant", "channel"]]
```

### 1.3 上下文传递模型 (`saleor/graphql/core/context.py:88`)

```python
@dataclass
class ChannelContext[N]:
    node: N              # 业务实体（Product/ProductVariant 等）
    channel_slug: str | None  # 渠道标识

@dataclass
class ChannelQsContext[M]:
    qs: QuerySet[M]       # 查询集
    channel_slug: str | None
```

---

## 二、第一层：GraphQL 入参层

### 2.1 参数接收 (`saleor/graphql/product/schema.py:206-277`)

**查询定义：**
```graphql
product(
    id: ID
    slug: String
    channel: String  # 渠道 slug 参数
): Product

products(
    channel: String  # 渠道 slug 参数
    filter: ProductFilterInput
    sortBy: ProductOrder
    search: String
): ProductCountableConnection
```

### 2.2 参数处理流程

以 `resolve_product` 为例 (`saleor/graphql/product/schema.py:430-477`)：

```python
@staticmethod
def resolve_product(
    _root, info: ResolveInfo, *, id=None, slug=None, channel=None, ...
):
    # 1. 参数验证
    validate_one_of_args_is_in_query("id", id, "slug", slug, ...)
    requestor = get_user_or_app_from_context(info.context)
    has_required_permissions = has_one_of_permissions(requestor, ALL_PRODUCTS_PERMISSIONS)

    # 2. 渠道默认值处理
    limited_channel_access = False if channel is None else True
    if channel is None and not has_required_permissions:
        # 普通用户未传 channel 时使用默认渠道
        channel = get_default_channel_slug_or_graphql_error(...)

    # 3. 异步加载 Channel 对象
    def _resolve_product(channel_obj):
        product = resolve_product(
            info, ..., channel=channel_obj,
            limited_channel_access=limited_channel_access,
            requestor=requestor
        )
        # 4. 封装为 ChannelContext 向下传递
        return ChannelContext(node=product, channel_slug=channel) if product else None

    if channel:
        return ChannelBySlugLoader(info.context).load(str(channel)).then(_resolve_product)
    return _resolve_product(None)
```

### 2.3 查询集过滤 (`saleor/graphql/product/resolvers.py:116-139`)

```python
def resolve_products(info, requestor, channel, limited_channel_access) -> ChannelQsContext:
    connection_name = get_database_connection_name(info.context)

    # 1. 基础可见性过滤
    qs = models.Product.objects.using(connection_name).visible_to_user(
        requestor, channel, limited_channel_access
    )

    # 2. 普通用户额外过滤：只显示在列表中可见的商品
    if not has_one_of_permissions(requestor, ALL_PRODUCTS_PERMISSIONS):
        if channel:
            product_channel_listings = (
                models.ProductChannelListing.objects.using(connection_name)
                .filter(channel_id=channel.id, visible_in_listings=True)
                .values("id")
            )
            qs = qs.filter(
                Exists(product_channel_listings.filter(product_id=OuterRef("pk")))
            )
        else:
            qs = models.Product.objects.none()

    # 3. 返回 ChannelQsContext 供后续解析使用
    channel_slug = channel.slug if channel else None
    return ChannelQsContext(qs=qs, channel_slug=channel_slug)
```

### 2.4 可见性判定逻辑 (`saleor/product/managers.py:75-120`)

```python
def visible_to_user(self, requestor, channel, limited_channel_access):
    if has_one_of_permissions(requestor, ALL_PRODUCTS_PERMISSIONS):
        # 管理员权限
        if limited_channel_access:
            if channel:
                # 限定渠道：只返回该渠道下的产品
                channel_listings = ProductChannelListing.objects.filter(
                    channel_id=channel.id
                ).values("id")
                return self.filter(
                    Exists(channel_listings.filter(product_id=OuterRef("pk")))
                )
            else:
                return self.none()
        else:
            return self.all()
    else:
        # 普通用户：必须有渠道且产品已发布
        if not channel:
            return self.none()
        channel_listings = ProductChannelListing.objects.filter(
            channel_id=channel.id,
            published_at__isnull=False,  # 已发布
            published_at__lte=now()
        ).values("id")
        return self.filter(
            Exists(channel_listings.filter(product_id=OuterRef("pk")))
        )
```

---

## 三、第二层：价格计算层

### 3.1 定价解析入口 (`saleor/graphql/product/types/products.py:685-788`)

**ProductVariant.pricing 字段解析：**

```python
@staticmethod
def resolve_pricing(root: ChannelContext[models.ProductVariant], info, *, address=None):
    if not root.channel_slug:
        return None

    channel_slug = str(root.channel_slug)
    context = info.context

    # 1. 批量加载所需数据（DataLoader 模式）
    product_channel_listing = ProductChannelListingByProductIdAndChannelSlugLoader(
        context
    ).load((root.node.product_id, channel_slug))

    variant_channel_listing = VariantChannelListingByVariantIdAndChannelSlugLoader(
        context
    ).load((root.node.id, channel_slug))

    channel = ChannelBySlugLoader(context).load(channel_slug)
    tax_class_id_loader = TaxClassIdByProductIdLoader(context).load(root.node.product_id)

    # 2. 组合数据并计算价格
    def load_tax_configuration(data):
        (channel, variant_channel_listing, product_channel_listing, tax_class_id) = data
        # ... 税务配置加载 ...
        return get_variant_availability(
            variant_channel_listing=variant_channel_listing,
            product_channel_listing=product_channel_listing,
            prices_entered_with_tax=...,
            tax_calculation_strategy=...,
            tax_rate=...,
        )

    return (
        Promise.all([channel, variant_channel_listing, product_channel_listing, ...])
        .then(load_tax_configuration)
    )
```

### 3.2 价格可用性计算 (`saleor/product/utils/availability.py:178-228`)

```python
def get_variant_availability(
    *,
    variant_channel_listing: ProductVariantChannelListing,
    product_channel_listing: ProductChannelListing | None,
    prices_entered_with_tax: bool,
    tax_calculation_strategy: str,
    tax_rate: Decimal,
) -> VariantAvailability | None:

    if variant_channel_listing.price is None:
        return None

    # 1. 计算含税价格
    discounted_price_taxed = _calculate_product_price_with_taxes(
        variant_channel_listing.discounted_price, tax_rate, ...
    )
    undiscounted_price_taxed = _calculate_product_price_with_taxes(
        variant_channel_listing.price, tax_rate, ...
    )

    # 2. 计算折扣金额
    discount = _get_total_discount(undiscounted_price_taxed, discounted_price_taxed)

    # 3. 可见性判定（关键：依赖 product_channel_listing）
    is_visible = (
        product_channel_listing is not None
        and product_channel_listing.is_visible
    )
    is_on_sale = is_visible and discount is not None

    return VariantAvailability(
        on_sale=is_on_sale,
        price=discounted_price_taxed,
        price_undiscounted=undiscounted_price_taxed,
        price_prior=prior_price_taxed,
        discount=discount,
        discount_prior=discount_prior,
    )
```

### 3.3 折扣价格预计算 (`saleor/product/utils/variant_prices.py:29-119`)

系统会异步预计算折扣价格，存储在 `ProductVariantChannelListing.discounted_price` 中：

```python
def update_discounted_prices_for_promotion(products, only_dirty_products=False):
    # 1. 获取变体与促销规则的映射
    rules_info_per_variant = get_variants_to_promotion_rules_map(variant_qs)

    # 2. 按产品+渠道分组处理
    for product_channel_listing in product_channel_listings:
        product_id = product_channel_listing.product_id
        channel_id = product_channel_listing.channel_id
        variant_listings = product_to_variant_listings_per_channel_map[product_id][channel_id]

        # 3. 计算每个变体的折扣价
        (
            discounted_variants_price,
            variant_listings_to_update,
            ...
        ) = _get_discounted_variants_prices_for_promotions(
            variant_listings,
            rules_info_per_variant,
            product_channel_listing.channel,  # 渠道信息用于促销规则匹配
            ...
        )

        # 4. 产品级折扣价 = 最低变体折扣价
        product_discounted_price = min(discounted_variants_price)
```

---

## 四、第三层：可购买性判定层

### 4.1 产品可用性解析 (`saleor/graphql/product/types/products.py:1341-1400`)

```python
@staticmethod
def resolve_is_available(root: ChannelContext[models.Product], info, site, *, address=None):
    if not root.channel_slug:
        return None

    channel_slug = str(root.channel_slug)
    country_code = address.country if address is not None else None

    # 1. 加载该渠道下的所有变体
    def load_variants_availability(variants):
        if include_shipping_zones:
            # 按渠道+国家筛选库存
            country_keys = [(variant.id, country_code, channel_slug) for variant in variants]
            return VariantAvailableQuantityByVariantIdCountryCodeAndChannelSlugLoader(
                info.context
            ).load_many(country_keys)
        else:
            # 按渠道筛选库存
            channel_keys = [(variant.id, channel_slug) for variant in variants]
            return VariantAvailableQuantityByVariantIdAndChannelSlugLoader(
                info.context
            ).load_many(channel_keys)

    # 2. 只要有一个变体可用，则产品可用
    def calculate_is_available(quantities):
        for qty in quantities:
            if qty > 0:
                return True
        return False

    return (
        ProductVariantsByProductIdLoader(info.context)
        .load(root.node.id)
        .then(load_variants_availability)
        .then(calculate_is_available)
    )
```

### 4.2 可购买时间判定 (`saleor/graphql/product/types/products.py:1635-1649`)

```python
@staticmethod
def resolve_is_available_for_purchase(root: ChannelContext[models.Product], info):
    if not root.channel_slug:
        return None
    channel_slug = str(root.channel_slug)

    def calculate_is_available_for_purchase(product_channel_listing):
        if not product_channel_listing:
            return None
        # 调用 ProductChannelListing 的方法
        return product_channel_listing.is_available_for_purchase()

    return (
        ProductChannelListingByProductIdAndChannelSlugLoader(info.context)
        .load((root.node.id, channel_slug))
        .then(calculate_is_available_for_purchase)
    )
```

**ProductChannelListing 中的实现 (`saleor/product/models.py:341-345`)：**
```python
def is_available_for_purchase(self):
    return (
        self.available_for_purchase_at is not None
        and datetime.datetime.now(tz=datetime.UTC) >= self.available_for_purchase_at
    )
```

### 4.3 库存可用性检查 (`saleor/warehouse/availability.py:63-100`)

```python
def check_stock_and_preorder_quantity(
    variant, country_code, channel_slug, quantity, *,
    include_shipping_zones, checkout_lines, check_reservations, ...
):
    """验证变体在指定渠道是否有足够库存"""
    if variant.is_preorder_active():
        # 预购模式：检查预购配额
        check_preorder_threshold_in_orders(variant, quantity, channel_slug, ...)
    else:
        # 普通模式：检查实际库存
        check_stock_quantity(
            variant, country_code, channel_slug, quantity,
            include_shipping_zones=include_shipping_zones, ...
        )
```

### 4.4 库存查询 - 按渠道筛选 (`saleor/warehouse/models.py:440-452`)

```python
class StockQuerySet:
    def for_channel(self, channel_slug: str):
        """返回指定渠道下所有仓库的库存"""
        WarehouseChannel = Channel.warehouses.through
        channels = Channel.objects.filter(slug=channel_slug).values("pk")
        warehouse_channels = WarehouseChannel.objects.filter(
            Exists(channels.filter(pk=OuterRef("channel_id")))
        ).values("warehouse_id")

        return self.select_related("product_variant").filter(
            Exists(warehouse_channels.filter(warehouse_id=OuterRef("warehouse_id")))
        )

    def for_channel_and_country(self, channel_slug, country_code, ...):
        """返回指定渠道+国家配送区域的库存"""
        # 通过 ShippingZone → Warehouse 关联筛选
        # 确保仓库在该渠道的配送范围内
```

### 4.5 库存获取 (`saleor/warehouse/models.py:474-487`)

```python
def get_variant_stocks(
    self, channel_slug, product_variant, country_code=None, *, include_shipping_zones
):
    if include_shipping_zones:
        return self.for_channel_and_country(channel_slug, country_code).filter(
            product_variant=product_variant
        )
    return self.for_channel(channel_slug).filter(product_variant=product_variant)
```

---

## 五、三层贯通数据流

### 5.1 完整调用链示例：查询产品详情

```
GraphQL 请求
  ↓
POST /graphql
  ↓
schema.py: resolve_product()         [入参层]
  ├─ 接收 channel 参数
  ├─ 权限检查与默认渠道处理
  ├─ 加载 Channel 对象
  ├─ 调用 resolvers.resolve_product()
  │   └─ managers.ProductQuerySet.visible_to_user()
  │       └─ 按 channel 过滤 ProductChannelListing
  └─ 封装为 ChannelContext(node=product, channel_slug=channel)
  ↓
types/products.py: resolve_pricing() [价格计算层]
  ├─ 从 root.channel_slug 读取渠道
  ├─ 加载 ProductChannelListing + ProductVariantChannelListing
  └─ 调用 availability.get_variant_availability()
      ├─ 从 channel_listing 读取价格
      ├─ 计算含税价格
      └─ 检查 product_channel_listing.is_visible
  ↓
types/products.py: resolve_is_available() [可购买性层]
  ├─ 从 root.channel_slug 读取渠道
  ├─ 加载变体库存数据（按 channel 过滤）
  └─ 检查库存数量 > 0
  ↓
GraphQL 响应
```

### 5.2 数据流向图

```
┌─────────────────────────────────────────────────────────────┐
│                    GraphQL 入参层                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Query.product(channel: "default-channel")          │  │
│  └─────────────────────┬────────────────────────────────┘  │
│                        │ channel_slug                       │
│                        ▼                                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  ChannelContext(node=Product, channel_slug="default")│  │
│  └─────────────────────┬────────────────────────────────┘  │
└────────────────────────│────────────────────────────────────┘
                         │
┌────────────────────────│────────────────────────────────────┐
│                    价格计算层                               │
│                        │                                   │
│  ┌─────────────────────▼────────────────────────────────┐  │
│  │  ProductChannelListing                              │  │
│  │  - channel_id = 1                                   │  │
│  │  - visible_in_listings = true                       │  │
│  │  - discounted_price_amount = 99.99                  │  │
│  └─────────────────────┬────────────────────────────────┘  │
│                        │                                   │
│  ┌─────────────────────▼────────────────────────────────┐  │
│  │  ProductVariantChannelListing                       │  │
│  │  - channel_id = 1                                   │  │
│  │  - price_amount = 100.00                            │  │
│  │  - discounted_price_amount = 80.00                  │  │
│  └─────────────────────┬────────────────────────────────┘  │
│                        │                                   │
│  ┌─────────────────────▼────────────────────────────────┐  │
│  │  VariantAvailability                                 │  │
│  │  - on_sale = true                                   │  │
│  │  - price = TaxedMoney(net=80, gross=88)             │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                         │
┌────────────────────────│────────────────────────────────────┐
│                  可购买性判定层                             │
│                        │                                   │
│  ┌─────────────────────▼────────────────────────────────┐  │
│  │  WarehouseChannel 关联表                            │  │
│  │  筛选出 channel=1 的仓库                            │  │
│  └─────────────────────┬────────────────────────────────┘  │
│                        │                                   │
│  ┌─────────────────────▼────────────────────────────────┐  │
│  │  Stock.objects.for_channel("default")               │  │
│  │  筛选出关联仓库的库存                                │  │
│  └─────────────────────┬────────────────────────────────┘  │
│                        │                                   │
│  ┌─────────────────────▼────────────────────────────────┐  │
│  │  is_available = true                                │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 六、关键设计模式与技术要点

### 6.1 DataLoader 批量数据加载

所有跨表查询都使用 DataLoader 模式，避免 N+1 问题：
- `ChannelBySlugLoader` - 按 slug 加载渠道
- `ProductChannelListingByProductIdAndChannelSlugLoader` - 加载产品渠道配置
- `VariantChannelListingByVariantIdAndChannelSlugLoader` - 加载变体渠道配置
- `VariantAvailableQuantityByVariantIdAndChannelSlugLoader` - 加载库存数量

### 6.2 上下文传递模式

通过 `ChannelContext` 和 `ChannelQsContext` 将渠道信息贯穿整个解析链：
- 封装业务实体和渠道标识
- 类型安全，支持泛型
- 解析器可直接从 `root.channel_slug` 获取渠道

### 6.3 数据库查询优化

1. **联合唯一约束**：`ProductChannelListing` 和 `ProductVariantChannelListing` 都有 `(entity_id, channel_id)` 联合唯一约束
2. **Exists 子查询**：使用 `Exists(OuterRef)` 模式避免 JOIN 性能问题
3. **索引优化**：`discounted_price_amount` 等常用字段建有 BTree 索引
4. **读写分离**：通过 `get_database_connection_name(context)` 选择主从库

### 6.4 价格计算的两层架构

1. **预计算层**：异步任务预计算折扣价格，写入 `discounted_price_amount`
2. **实时计算层**：查询时实时计算税费，生成最终含税价格

### 6.5 可见性的多维度判定

```
产品可见性 = 全局可见性 AND 渠道可见性
  ├─ 全局：Product.published_at ≤ now
  └─ 渠道：
      ├─ ProductChannelListing.published_at ≤ now
      ├─ ProductChannelListing.visible_in_listings = true（列表页）
      └─ ProductChannelListing.available_for_purchase_at ≤ now（购买）
```

---

## 七、关键文件索引

| 层级 | 文件路径 | 主要职责 |
|------|---------|---------|
| 数据模型 | `saleor/channel/models.py` | Channel 模型定义 |
| 数据模型 | `saleor/product/models.py` | ProductChannelListing, ProductVariantChannelListing |
| 上下文 | `saleor/graphql/core/context.py` | ChannelContext, ChannelQsContext |
| 入参层 | `saleor/graphql/product/schema.py` | GraphQL 查询定义与参数处理 |
| 入参层 | `saleor/graphql/product/resolvers.py` | 查询集过滤与渠道筛选 |
| 入参层 | `saleor/product/managers.py` | visible_to_user, available_in_channel |
| 价格层 | `saleor/graphql/product/types/products.py` | pricing, is_available 字段解析 |
| 价格层 | `saleor/product/utils/availability.py` | 价格可用性计算 |
| 价格层 | `saleor/product/utils/variant_prices.py` | 折扣价格预计算 |
| 库存层 | `saleor/warehouse/availability.py` | 库存可用性检查 |
| 库存层 | `saleor/warehouse/models.py` | StockQuerySet.for_channel 等查询方法 |
