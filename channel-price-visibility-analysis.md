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
```

### 1.2 PublishableModel 基类 (`saleor/core/models.py:63-77`)

`ProductChannelListing` 继承自此基类，提供发布状态管理：

```python
class PublishableModel(models.Model):
    published_at = models.DateTimeField(blank=True, null=True)
    is_published = models.BooleanField(default=False)

    @property
    def is_visible(self):
        """产品在渠道中是否可见：已发布且发布时间已过"""
        return self.is_published and (
            self.published_at is None
            or self.published_at <= datetime.datetime.now(tz=datetime.UTC)
        )
```

### 1.3 商品渠道关联模型

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
        unique_together = [["product", "channel"]]

    def is_available_for_purchase(self):
        """是否达到可购买时间：available_for_purchase_at 不为空且已过期"""
        return (
            self.available_for_purchase_at is not None
            and datetime.datetime.now(tz=datetime.UTC) >= self.available_for_purchase_at
        )
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

### 1.4 上下文传递模型 (`saleor/graphql/core/context.py:88`)

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

### 2.1 两层职责划分

**重要：schema 层与 resolvers 层职责严格分离。**

| 层级 | 职责 | 文件 |
|------|------|------|
| **Schema 层** | 参数接收、默认渠道注入、Channel 对象加载、封装 ChannelContext | `saleor/graphql/product/schema.py` |
| **Resolvers 层** | 业务查询、可见性过滤（调用 visible_to_user）、返回 ChannelQsContext | `saleor/graphql/product/resolvers.py` |

### 2.2 Schema 层：默认渠道注入 (`saleor/graphql/product/schema.py:430-515`)

以 `resolve_products` 为例，展示 schema 层的完整处理流程：

```python
@staticmethod
@traced_resolver
def resolve_products(_root, info: ResolveInfo, *, channel=None, **kwargs):
    validate_and_apply_search_rank_sorting(...)
    search = kwargs.get("search")

    requestor = get_user_or_app_from_context(info.context)
    has_required_permissions = has_one_of_permissions(
        requestor, ALL_PRODUCTS_PERMISSIONS
    )

    # ========== Schema 层职责 1：参数标记 ==========
    # limited_channel_access 标记用户是否显式传了 channel
    limited_channel_access = False if channel is None else True

    # ========== Schema 层职责 2：默认渠道注入 ==========
    # 仅当普通用户未传 channel 时，注入默认渠道
    if channel is None and not has_required_permissions:
        channel = get_default_channel_slug_or_graphql_error(
            allow_replica=info.context.allow_replica
        )

    # ========== Schema 层职责 3：加载 Channel 对象并委托 resolvers ==========
    def _resolve_products(channel_obj):
        # 调用 resolvers 层进行业务查询
        qs = resolve_products(info, requestor, channel_obj, limited_channel_access)

        if search:
            qs = ChannelQsContext(
                qs=prefix_search(qs.qs, search), channel_slug=channel
            )
        kwargs["channel"] = channel
        qs = filter_connection_queryset(
            qs, kwargs, allow_replica=info.context.allow_replica
        )
        return create_connection_slice(qs, info, kwargs, ProductCountableConnection)

    if channel:
        return (
            ChannelBySlugLoader(info.context)
            .load(str(channel))
            .then(_resolve_products)
        )
    return _resolve_products(None)
```

**默认渠道注入的关键规则：**
- 仅对**普通用户**且**未传 channel**时触发
- 管理员用户不传 channel 时允许跨渠道查询
- 注入默认渠道后，`limited_channel_access` 仍为 `False`（因为用户没传）

### 2.3 Resolvers 层：可见性过滤 (`saleor/graphql/product/resolvers.py:116-139`)

**重要：visible_to_user 只负责可见性过滤，不做价格、库存等其他判断。**

```python
def resolve_products(info, requestor, channel, limited_channel_access) -> ChannelQsContext:
    connection_name = get_database_connection_name(info.context)

    # ========== Resolvers 层唯一职责：调用 visible_to_user 做可见性过滤 ==========
    qs = models.Product.objects.using(connection_name).visible_to_user(
        requestor, channel, limited_channel_access
    )

    # 普通用户额外过滤：只显示在列表中可见的商品（visible_in_listings）
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

    channel_slug = channel.slug if channel else None
    return ChannelQsContext(qs=qs, channel_slug=channel_slug)
```

### 2.4 visible_to_user 的完整分支逻辑 (`saleor/product/managers.py:75-113`)

`visible_to_user` 有 **6 种分支**，仅做可见性过滤：

```python
def visible_to_user(self, requestor, channel, limited_channel_access):
    if has_one_of_permissions(requestor, ALL_PRODUCTS_PERMISSIONS):
        # ========== 管理员分支 ==========
        if limited_channel_access:
            # 分支 A：管理员 + 传了 channel
            # → 只返回该渠道下有 ProductChannelListing 的产品
            if channel:
                channel_listings = ProductChannelListing.objects.filter(
                    channel_id=channel.id
                ).values("id")
                return self.filter(
                    Exists(channel_listings.filter(product_id=OuterRef("pk")))
                )
            # 分支 B：管理员 + limited_channel_access=True 但 channel=None
            # → 返回空（理论不会发生）
            return self.none()
        # 分支 C：管理员 + 未传 channel
        # → 返回所有产品（跨渠道查询）
        return self.all()
    else:
        # ========== 普通用户分支 ==========
        # 分支 D：普通用户 + channel=None
        # → 返回空（普通用户不允许跨渠道）
        if not channel:
            return self.none()
        # 分支 E：普通用户 + 传了 channel（或默认渠道已注入）
        # → 调用 published_with_variants
        return self.published_with_variants(channel)
```

**分支对比表：**

| 请求者 | channel 参数 | limited_channel_access | 触发场景 | visible_to_user 返回 |
|--------|-------------|------------------------|----------|---------------------|
| 管理员 | 有值        | True                   | 管理员指定渠道查询 | 该渠道下有 listing 的产品 |
| 管理员 | None        | False                  | 管理员跨渠道查询 | 所有产品 |
| 管理员 | None        | True                   | 理论不会出现 | 空 |
| 普通用户 | 有值      | True                   | 用户指定渠道查询 | 已发布 + 有变体检索到 |
| 普通用户 | None      | False                  | Schema 层注入默认渠道 | 已发布 + 有变体检索到 |
| 普通用户 | None      | True                   | 理论不会出现 | 空 |

### 2.5 `published_with_variants` 完整逻辑 (`saleor/product/managers.py:55-73`)

```python
def published_with_variants(self, channel: Channel):
    from .models import ProductVariant, ProductVariantChannelListing

    # 1. 渠道必须激活
    if not channel.is_active:
        return self.none()

    # 2. 变体在该渠道下有定价
    variant_channel_listings = (
        ProductVariantChannelListing.objects.using(self.db)
        .filter(channel_id=channel.id, price_amount__isnull=False)
        .values("id")
    )
    variants = ProductVariant.objects.using(self.db).filter(
        Exists(variant_channel_listings.filter(variant_id=OuterRef("pk")))
    )

    # 3. 产品已发布（is_published=True 且 published_at <= now）
    return self.published(channel).filter(
        Exists(variants.filter(product_id=OuterRef("pk")))
    )
```

---

## 三、第二层：价格计算层

### 3.1 两条独立的定价解析链路

**重要：产品（Product）和变体（ProductVariant）的定价解析是两条完全独立的链路。**

#### 链路 A：Product.pricing (`saleor/graphql/product/types/products.py:1232-1336`)

```python
@staticmethod
def resolve_pricing(root: ChannelContext[models.Product], info, *, address=None):
    if not root.channel_slug:
        return None

    channel_slug = str(root.channel_slug)
    context = info.context

    # 使用 Product + 变体列表
    channel = ChannelBySlugLoader(context).load(channel_slug)
    product_channel_listing = ProductChannelListingByProductIdAndChannelSlugLoader(
        context
    ).load((root.node.id, channel_slug))
    # 关键：加载该产品在该渠道下的 ALL 变体 listings
    variants_channel_listing = (
        VariantsChannelListingByProductIdAndChannelSlugLoader(context).load(
            (root.node.id, channel_slug)
        )
    )
    tax_class_id_loader = TaxClassIdByProductIdLoader(context).load(root.node.id)

    def load_tax_configuration(data):
        (channel, product_channel_listing, variants_channel_listing, tax_class_id) = data
        if not variants_channel_listing:
            return None
        # ... 税务配置加载 ...

        def calculate_pricing_info(data):
            # ... 税率计算 ...
            availability = get_product_availability(
                product_channel_listing=product_channel_listing,
                variants_channel_listing=variants_channel_listing,  # 列表
                ...
            )
            return ProductPricingInfo(**asdict(availability))

        # ... 异步加载 ...

    return Promise.all([...]).then(load_tax_configuration)
```

#### 链路 B：ProductVariant.pricing (`saleor/graphql/product/types/products.py:685-789`)

```python
@staticmethod
def resolve_pricing(root: ChannelContext[models.ProductVariant], info, *, address=None):
    if not root.channel_slug:
        return None

    channel_slug = str(root.channel_slug)
    context = info.context

    # 使用单个 Variant
    product_channel_listing = ProductChannelListingByProductIdAndChannelSlugLoader(
        context
    ).load((root.node.product_id, channel_slug))
    # 关键：只加载当前这一个变体的 listing
    variant_channel_listing = VariantChannelListingByVariantIdAndChannelSlugLoader(
        context
    ).load((root.node.id, channel_slug))
    channel = ChannelBySlugLoader(context).load(channel_slug)
    tax_class_id_loader = TaxClassIdByProductIdLoader(context).load(
        root.node.product_id
    )

    def load_tax_configuration(data):
        (product_channel_listing, variant_channel_listing, channel, tax_class_id) = data
        if not variant_channel_listing or not product_channel_listing:
            return None
        # ... 税务配置加载 ...

        def calculate_pricing_info(data):
            # ... 税率计算 ...
            availability = get_variant_availability(
                variant_channel_listing=variant_channel_listing,  # 单个
                product_channel_listing=product_channel_listing,
                ...
            )
            return VariantPricingInfo(**asdict(availability)) if availability else None

        # ... 异步加载 ...

    return Promise.all([...]).then(load_tax_configuration)
```

### 3.2 is_visible 对 on_sale 与价格字段的真实影响

**关键：is_visible 是价格返回的总开关，决定了 on_sale 的值，但不影响价格本身的计算。**

**get_product_availability 核心逻辑 (`saleor/product/utils/availability.py:119-175`)：**

```python
def get_product_availability(
    *,
    product_channel_listing,
    variants_channel_listing,  # 列表
    ...
):
    # ========== 步骤 1：价格计算（不受 is_visible 影响） ==========
    undiscounted = _calculate_product_price_with_taxes_range(
        "price", variants_channel_listing, ...
    )
    discounted = _calculate_product_price_with_taxes_range(
        "discounted_price", variants_channel_listing, ...
    )
    prior = _calculate_product_price_with_taxes_range(
        "prior_price", variants_channel_listing, ...
    )
    discount = _get_total_discount_from_range(undiscounted, discounted)

    # ========== 步骤 2：is_visible 判定（总开关） ==========
    # is_visible 来自 PublishableModel.is_visible 属性
    # = is_published AND (published_at IS NULL OR published_at <= now)
    is_visible = (
        product_channel_listing is not None and product_channel_listing.is_visible
    )

    # ========== 步骤 3：on_sale 依赖 is_visible ==========
    # 即使有折扣，如果产品不可见，on_sale 也为 False
    is_on_sale = is_visible and discount is not None

    # ========== 步骤 4：价格字段原样返回（不受 is_visible 影响） ==========
    return ProductAvailability(
        on_sale=is_on_sale,              # 受 is_visible 影响
        price_range=discounted,          # 原样返回，不受 is_visible 影响
        price_range_undiscounted=undiscounted,  # 原样返回
        price_range_prior=prior,         # 原样返回
        discount=discount,               # 原样返回
        discount_prior=...,
    )
```

**get_variant_availability 核心逻辑 (`saleor/product/utils/availability.py:178-228`)：**

```python
def get_variant_availability(
    *,
    variant_channel_listing,  # 单个
    product_channel_listing,
    ...
):
    if variant_channel_listing.price is None:
        return None

    # ========== 步骤 1：价格计算（不受 is_visible 影响） ==========
    discounted_price_taxed = _calculate_product_price_with_taxes(
        variant_channel_listing.discounted_price, ...
    )
    undiscounted_price_taxed = _calculate_product_price_with_taxes(
        variant_channel_listing.price, ...
    )
    discount = _get_total_discount(undiscounted_price_taxed, discounted_price_taxed)

    # ========== 步骤 2：is_visible 判定（总开关） ==========
    is_visible = (
        product_channel_listing is not None and product_channel_listing.is_visible
    )

    # ========== 步骤 3：on_sale 依赖 is_visible ==========
    is_on_sale = is_visible and discount is not None

    # ========== 步骤 4：价格字段原样返回 ==========
    return VariantAvailability(
        on_sale=is_on_sale,              # 受 is_visible 影响
        price=discounted_price_taxed,    # 原样返回
        price_undiscounted=undiscounted_price_taxed,  # 原样返回
        price_prior=prior_price_taxed,   # 原样返回
        discount=discount,               # 原样返回
        discount_prior=...,
    )
```

**is_visible 影响总结：**

| 字段 | 受 is_visible 影响？ | 说明 |
|------|---------------------|------|
| `on_sale` | ✅ 是 | `is_on_sale = is_visible AND discount IS NOT NULL` |
| `price_range` / `price` | ❌ 否 | 即使 is_visible=False，价格仍会计算并返回 |
| `price_range_undiscounted` / `price_undiscounted` | ❌ 否 | 原样返回 |
| `discount` | ❌ 否 | 原样返回 |

**设计意图：** 价格信息可以用于后台展示或预览，即使产品未发布也能看到定价；但 `on_sale` 标志明确告诉前端"这个产品现在是否在促销"。

### 3.3 价格计算的两个函数对比

| 维度 | get_product_availability | get_variant_availability |
|------|--------------------------|--------------------------|
| 输入 | variants_channel_listing（列表） | variant_channel_listing（单个） |
| 价格范围 | 计算所有变体的 min/max 价格范围 | 只返回单个变体的价格 |
| 输出 | ProductAvailability（含 price_range） | VariantAvailability（含 price） |
| 位置 | saleor/product/utils/availability.py:119 | saleor/product/utils/availability.py:178 |

### 3.4 折扣价格预计算 (`saleor/product/utils/variant_prices.py:29-119`)

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
            product_channel_listing.channel,
            ...
        )

        # 4. 产品级折扣价 = 最低变体折扣价
        product_discounted_price = min(discounted_variants_price)
```

---

## 四、第三层：可购买性判定层

### 4.1 三个字段的语义边界

**重要：is_available、is_available_for_purchase、available_for_purchase 是三个完全不同的字段，语义边界清晰。**

| 字段 | 类型 | 语义 | 判定逻辑 | 使用场景 |
|------|------|------|----------|----------|
| **is_available** | Boolean | 产品现在能否下单购买（综合判定） | 1. is_available_for_purchase() = True<br>2. 任一变体库存 > 0 | 购物车/下单前检查 |
| **is_available_for_purchase** | Boolean | 产品是否设置为可购买状态（时间门槛） | `available_for_purchase_at IS NOT NULL AND now() >= available_for_purchase_at` | 展示"即将开售"标签 |
| **available_for_purchase** | Date（已废弃） | 可购买日期 | 直接返回 `available_for_purchase_at` 的日期部分 | 历史兼容 |
| **available_for_purchase_at** | DateTime | 可购买时间点 | 直接返回 `available_for_purchase_at` | 展示具体开售时间 |

**GraphQL 类型定义 (`saleor/graphql/product/types/products.py:969-1111`)：**

```graphql
type Product {
    # 综合可用性：可购买时间已过 + 有库存
    is_available(
        address: AddressInput
    ): Boolean
    description: "Whether the product is in stock, set as available for purchase in the given channel, and published."

    # 可购买状态：仅检查时间门槛
    is_available_for_purchase: Boolean
    description: """
        Refers to a state that can be set by admins to control whether a product
        is available for purchase in storefronts. This does not guarantee the
        availability of stock. When set to `False`, this product is still visible
        to customers, but it cannot be purchased.
    """

    # 可购买时间（已废弃）
    available_for_purchase: Date @deprecated(reason: "Use availableForPurchaseAt")

    # 可购买时间
    available_for_purchase_at: DateTime
}
```

### 4.2 is_available 的完整判定链 (`saleor/graphql/product/types/products.py:1341-1408`)

**is_available 是两层门槛的 AND 关系：时间门槛 AND 库存门槛**

```python
@staticmethod
@traced_resolver
@load_site_callback
def resolve_is_available(
    root: ChannelContext[models.Product], info, site, *, address=None
):
    if not root.channel_slug:
        return None

    channel_slug = str(root.channel_slug)
    country_code = address.country if address is not None else None
    requestor = get_user_or_app_from_context(info.context)
    has_required_permissions = has_one_of_permissions(
        requestor, ALL_PRODUCTS_PERMISSIONS
    )

    # ========== 门槛 1：可购买时间检查 ==========
    def check_is_available_for_purchase(product_channel_listing):
        # 如果没有产品渠道 listing，直接不可用
        if product_channel_listing:
            # 调用 ProductChannelListing.is_available_for_purchase()
            # 检查 available_for_purchase_at <= now
            if product_channel_listing.is_available_for_purchase():
                # 时间门槛通过，进入库存检查
                return check_variant_availability()
        # 时间门槛未通过，直接返回 False
        return False

    # ========== 门槛 2：库存检查 ==========
    def check_variant_availability():
        # 根据权限选择不同的变体加载器
        if has_required_permissions and not channel_slug:
            # 管理员跨渠道：加载所有变体
            variants = ProductVariantsByProductIdLoader(info.context).load(
                root.node.id
            )
        elif has_required_permissions and channel_slug:
            # 管理员指定渠道：加载该渠道下的所有变体（含未定价的）
            variants = ProductVariantsByProductIdAndChannel(info.context).load(
                (root.node.id, channel_slug)
            )
        else:
            # 普通用户：只加载该渠道下有定价的可用变体
            variants = AvailableProductVariantsByProductIdAndChannel(
                info.context
            ).load((root.node.id, channel_slug))

        return variants.then(load_variants_availability).then(
            calculate_is_available
        )

    # 加载变体的库存可用性
    def load_variants_availability(variants):
        include_shipping_zones = (
            site.settings.use_legacy_shipping_zone_stock_availability
        )
        if include_shipping_zones:
            # 按渠道+国家筛选库存
            country_keys = [(variant.id, country_code, channel_slug) for variant in variants]
            return AvailableQuantityByProductVariantIdCountryCodeAndChannelSlugLoader(
                info.context
            ).load_many(country_keys)
        # 按渠道筛选库存
        channel_keys = [(variant.id, channel_slug) for variant in variants]
        return AvailableQuantityByProductVariantIdAndChannelSlugLoader(
            info.context
        ).load_many(channel_keys)

    # 只要有一个变体可用，则产品可用
    def calculate_is_available(quantities):
        for qty in quantities:
            if qty > 0:
                return True
        return False

    # 从加载 product_channel_listing 开始整个判定链
    return (
        ProductChannelListingByProductIdAndChannelSlugLoader(info.context)
        .load((root.node.id, channel_slug))
        .then(check_is_available_for_purchase)  # 先过时间门槛
    )
```

### 4.3 is_available_for_purchase 的独立解析 (`saleor/graphql/product/types/products.py:1635-1666`)

**is_available_for_purchase 只检查时间门槛，与库存无关：**

```python
@staticmethod
def resolve_is_available_for_purchase(root: ChannelContext[models.Product], info):
    if not root.channel_slug:
        return None
    channel_slug = str(root.channel_slug)

    def calculate_is_available_for_purchase(product_channel_listing):
        if not product_channel_listing:
            return None
        # 仅调用 is_available_for_purchase() 方法
        return product_channel_listing.is_available_for_purchase()

    return (
        ProductChannelListingByProductIdAndChannelSlugLoader(info.context)
        .load((root.node.id, channel_slug))
        .then(calculate_is_available_for_purchase)
    )

@staticmethod
def resolve_available_for_purchase_at(root: ChannelContext[models.Product], info):
    if not root.channel_slug:
        return None
    channel_slug = str(root.channel_slug)

    def calculate_available_for_purchase_at(product_channel_listing):
        if not product_channel_listing:
            return None
        # 直接返回字段值，不做任何判断
        return product_channel_listing.available_for_purchase_at

    return (
        ProductChannelListingByProductIdAndChannelSlugLoader(info.context)
        .load((root.node.id, channel_slug))
        .then(calculate_available_for_purchase_at)
    )
```

### 4.4 判定流程可视化

```
resolve_is_available()
    │
    ▼
加载 ProductChannelListing
    │
    ▼
check_is_available_for_purchase()
    ├─ 存在 listing？
    │   ├─ 否 → 返回 False
    │   └─ 是 → 检查 is_available_for_purchase()
    │       ├─ available_for_purchase_at 已过？
    │       │   ├─ 否 → 返回 False
    │       │   └─ 是 → 进入 check_variant_availability()
    │       └─ available_for_purchase_at 为空？
    │           └─ 返回 False
    └─ 无 listing → 返回 False
                        │
                        ▼
                check_variant_availability()
                    │
                    ▼
                根据权限选择变体加载器
                    │
                    ▼
                load_variants_availability()
                    │
                    ▼
                calculate_is_available()
                    └─ 任一变体库存 > 0 → 返回 True
```

### 4.5 可购买时间判定 (`saleor/product/models.py:341-345`)

```python
def is_available_for_purchase(self):
    """只有 available_for_purchase_at 不为空且已过期，才返回 True"""
    return (
        self.available_for_purchase_at is not None
        and datetime.datetime.now(tz=datetime.UTC) >= self.available_for_purchase_at
    )
```

### 4.6 库存可用性检查 (`saleor/warehouse/availability.py:63-100`)

```python
def check_stock_and_preorder_quantity(
    variant, country_code, channel_slug, quantity, *,
    include_shipping_zones, checkout_lines, check_reservations, ...
):
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

### 4.7 库存查询 - 按渠道筛选 (`saleor/warehouse/models.py:440-452`)

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
```

---

## 五、三层贯通数据流

### 5.1 完整调用链示例：查询产品详情

```
GraphQL 请求
  ↓
POST /graphql
  ↓
schema.py: resolve_product()         [Schema 层]
  ├─ 接收 channel 参数
  ├─ 设置 limited_channel_access = (channel is not None)
  ├─ 普通用户无 channel 时注入默认渠道
  ├─ 加载 Channel 对象
  ├─ 调用 resolvers.resolve_product()
  │   └─ managers.ProductQuerySet.visible_to_user()  [Resolvers 层]
  │       ├─ 仅做可见性过滤
  │       ├─ 管理员分支：6种情况的权限判断
  │       └─ 普通用户分支：published_with_variants
  │           ├─ channel.is_active?
  │           ├─ ProductChannelListing 已发布?
  │           └─ 至少一个变体有定价?
  └─ 封装为 ChannelContext(node=product, channel_slug=channel)
  ↓
types/products.py: resolve_pricing() [价格计算层]
  ├─ Product.pricing:
  │   ├─ 从 root.channel_slug 读取渠道
  │   ├─ 加载 ProductChannelListing + ALL 变体 listings
  │   └─ 调用 get_product_availability() → 返回价格范围
  │       ├─ 价格计算（不受 is_visible 影响）
  │       └─ on_sale = is_visible AND discount IS NOT NULL
  └─ ProductVariant.pricing:
      ├─ 从 root.channel_slug 读取渠道
      ├─ 加载 ProductChannelListing + 单个变体 listing
      └─ 调用 get_variant_availability() → 返回单个价格
          ├─ 价格计算（不受 is_visible 影响）
          └─ on_sale = is_visible AND discount IS NOT NULL
  ↓
types/products.py: resolve_is_available() [可购买性层]
  ├─ 从 root.channel_slug 读取渠道
  ├─ 门槛 1：加载 ProductChannelListing
  │   └─ 检查 is_available_for_purchase()（时间门槛）
  ├─ 门槛 2：检查库存
  │   ├─ 根据权限选择变体加载器
  │   ├─ 加载变体库存（按 channel 过滤 Stock）
  │   └─ 检查库存数量 > 0
  └─ 两层门槛都通过才返回 True
  ↓
GraphQL 响应
```

### 5.2 可购买性判定的两层门槛

```
┌─────────────────────────────────────────────────────────────┐
│                可购买性判定（两层门槛）                       │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              门槛 1：时间门槛                          │  │
│  │  is_available_for_purchase                           │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │  ProductChannelListing                        │  │  │
│  │  │  - available_for_purchase_at = 2026-06-01     │  │  │
│  │  │  - 当前时间 = 2026-05-19                      │  │  │
│  │  └────────────────────┬───────────────────────────┘  │  │
│  │                       │                              │  │
│  │                       ▼                              │  │
│  │  is_available_for_purchase() = False                 │  │
│  │                       │                              │  │
│  │                       ├─ 是 → 进入库存检查           │  │
│  │                       └─ 否 → 返回 False             │  │
│  └──────────────────────────────────────────────────────┘  │
│                                 │                           │
│                                 ▼ （通过）                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              门槛 2：库存门槛                          │  │
│  │  is_available                                        │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │  Stock.objects.for_channel("default")         │  │  │
│  │  │  筛选出该渠道关联仓库的库存                     │  │  │
│  │  └────────────────────┬───────────────────────────┘  │  │
│  │                       │                              │  │
│  │                       ▼                              │  │
│  │  任一变体库存数量 > 0?                                │  │
│  │                       │                              │  │
│  │                       ├─ 是 → 返回 True              │  │
│  │                       └─ 否 → 返回 False             │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 价格计算的两条链路对比

```
┌─────────────────────────────────────────────────────────────┐
│                    价格计算的两条链路                         │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Product.pricing 链路                     │  │
│  │  输入：ChannelContext[Product]                       │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │  DataLoader: VariantsChannelListingBy...      │  │  │
│  │  │  加载该产品在该渠道下的 ALL 变体 listings       │  │  │
│  │  └────────────────────┬───────────────────────────┘  │  │
│  │                       │                              │  │
│  │                       ▼                              │  │
│  │  函数：get_product_availability()                   │  │
│  │  计算所有变体的价格范围 [min_price, max_price]       │  │
│  │  返回：ProductAvailability                         │  │
│  │  字段：price_range（TaxedMoneyRange）               │  │
│  │  on_sale = is_visible AND discount IS NOT NULL      │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           ProductVariant.pricing 链路                 │  │
│  │  输入：ChannelContext[ProductVariant]                │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │  DataLoader: VariantChannelListingBy...       │  │  │
│  │  │  只加载当前这一个变体的 listing                 │  │  │
│  │  └────────────────────┬───────────────────────────┘  │  │
│  │                       │                              │  │
│  │                       ▼                              │  │
│  │  函数：get_variant_availability()                   │  │
│  │  只计算单个变体的价格                                │  │
│  │  返回：VariantAvailability                         │  │
│  │  字段：price（TaxedMoney）                          │  │
│  │  on_sale = is_visible AND discount IS NOT NULL      │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 六、关键设计模式与技术要点

### 6.1 DataLoader 批量数据加载

所有跨表查询都使用 DataLoader 模式，避免 N+1 问题：
- `ChannelBySlugLoader` - 按 slug 加载渠道
- `ProductChannelListingByProductIdAndChannelSlugLoader` - 加载产品渠道配置
- `VariantsChannelListingByProductIdAndChannelSlugLoader` - 加载产品下所有变体的渠道配置
- `VariantChannelListingByVariantIdAndChannelSlugLoader` - 加载单个变体的渠道配置
- `AvailableQuantityByProductVariantIdAndChannelSlugLoader` - 加载库存数量

### 6.2 上下文传递模式

通过 `ChannelContext` 和 `ChannelQsContext` 将渠道信息贯穿整个解析链：
- 封装业务实体和渠道标识
- 类型安全，支持泛型
- 解析器可直接从 `root.channel_slug` 获取渠道

### 6.3 Schema 层与 Resolvers 层的职责分离

- **Schema 层**：参数处理、默认值注入、类型转换、分页包装
- **Resolvers 层**：业务查询、数据过滤、返回原始数据

### 6.4 visible_to_user 的单一职责

visible_to_user 只做可见性过滤，不涉及：
- ❌ 价格计算
- ❌ 库存检查
- ❌ 可购买时间判断
- ✅ 仅基于发布状态和渠道关联进行过滤

### 6.5 is_visible 的双重角色

1. **发布状态标志**：产品是否已发布且发布时间已过
2. **on_sale 的总开关**：即使有折扣，产品不可见时 on_sale 也为 False

### 6.6 三个可购买性字段的语义边界

| 字段 | 检查内容 | 依赖库存？ | 典型使用场景 |
|------|----------|------------|-------------|
| `available_for_purchase_at` | 直接返回时间值 | ❌ | 展示"即将于 X 月 X 日开售" |
| `is_available_for_purchase` | 检查时间是否已过 | ❌ | 展示"即将开售"或"在售中"标签 |
| `is_available` | 时间已过 AND 有库存 | ✅ | 下单按钮启用/禁用 |

### 6.7 数据库查询优化

1. **联合唯一约束**：`ProductChannelListing` 和 `ProductVariantChannelListing` 都有 `(entity_id, channel_id)` 联合唯一约束
2. **Exists 子查询**：使用 `Exists(OuterRef)` 模式避免 JOIN 性能问题
3. **索引优化**：`discounted_price_amount` 等常用字段建有 BTree 索引
4. **读写分离**：通过 `get_database_connection_name(context)` 选择主从库

---

## 七、关键文件索引

| 层级 | 文件路径 | 主要职责 |
|------|---------|---------|
| 数据模型 | `saleor/channel/models.py` | Channel 模型定义 |
| 数据模型 | `saleor/core/models.py` | PublishableModel（is_visible 属性） |
| 数据模型 | `saleor/product/models.py` | ProductChannelListing, ProductVariantChannelListing |
| 上下文 | `saleor/graphql/core/context.py` | ChannelContext, ChannelQsContext |
| Schema 层 | `saleor/graphql/product/schema.py` | GraphQL 查询定义、默认渠道注入、参数处理 |
| Resolvers 层 | `saleor/graphql/product/resolvers.py` | 调用 visible_to_user 做可见性过滤 |
| 入参层 | `saleor/product/managers.py` | visible_to_user, published_with_variants |
| 价格层 | `saleor/graphql/product/types/products.py` | Product.pricing / ProductVariant.pricing 解析 |
| 价格层 | `saleor/product/utils/availability.py` | get_product_availability, get_variant_availability |
| 价格层 | `saleor/product/utils/variant_prices.py` | 折扣价格预计算 |
| 可购买层 | `saleor/graphql/product/types/products.py` | resolve_is_available, resolve_is_available_for_purchase |
| 库存层 | `saleor/warehouse/availability.py` | check_stock_and_preorder_quantity |
| 库存层 | `saleor/warehouse/models.py` | StockQuerySet.for_channel 等查询方法 |
