# Saleor Channel 商品上架与价格可见性规则分析

## 1. 核心数据模型

Saleor 通过两级 ChannelListing 模型实现商品在不同渠道的独立配置：

### 1.1 ProductChannelListing (`saleor/product/models.py:304`)

产品级别的渠道配置，控制产品在特定渠道的可见性：

```python
class ProductChannelListing(PublishableModel):
    product = models.ForeignKey(Product, ...)          # 关联产品
    channel = models.ForeignKey(Channel, ...)          # 关联渠道
    visible_in_listings = models.BooleanField(default=False)  # 列表可见性
    available_for_purchase_at = models.DateTimeField(...)     # 可购买时间
    currency = models.CharField(...)                   # 渠道货币
    discounted_price_amount = models.DecimalField(...) # 最低折扣价（预计算）
    discounted_price = MoneyField(...)
    discounted_price_dirty = models.BooleanField(default=False) # 价格脏标记

    def is_available_for_purchase(self):
        return (
            self.available_for_purchase_at is not None
            and datetime.datetime.now(tz=datetime.UTC) >= self.available_for_purchase_at
        )
```

**索引**：`published_at`、`discounted_price_amount` 建有 BTree 索引用于加速查询过滤。

### 1.2 ProductVariantChannelListing (`saleor/product/models.py:477`)

变体级别的渠道配置，控制具体 SKU 在渠道的价格：

```python
class ProductVariantChannelListing(models.Model):
    variant = models.ForeignKey(ProductVariant, ...)
    channel = models.ForeignKey(Channel, ...)
    currency = models.CharField(...)
    
    # 三种价格字段
    price_amount = models.DecimalField(...)            # 基准价（列表价）
    price = MoneyField(...)
    
    cost_price_amount = models.DecimalField(...)       # 成本价（仅管理后台可见）
    cost_price = MoneyField(...)
    
    prior_price_amount = models.DecimalField(...)      # 原价（用于显示划线价）
    prior_price = MoneyField(...)
    
    discounted_price_amount = models.DecimalField(...) # 折扣价（含促销）
    discounted_price = MoneyField(...)
    
    promotion_rules = models.ManyToManyField(PromotionRule, through="...")  # 关联的促销规则
    preorder_quantity_threshold = models.IntegerField(...) # 预售库存阈值
```

### 1.3 PublishableModel 基类 (`saleor/core/models.py:63`)

`ProductChannelListing` 继承自该抽象基类，提供发布状态管理：

```python
class PublishableModel(models.Model):
    published_at = models.DateTimeField(blank=True, null=True)  # 发布时间
    is_published = models.BooleanField(default=False)           # 发布标记

    @property
    def is_visible(self):
        return self.is_published and (
            self.published_at is None
            or self.published_at <= datetime.datetime.now(tz=datetime.UTC)
        )
```

---

## 2. 可见性控制逻辑

### 2.1 产品可见性判断 (`ProductsQueryset.visible_to_user`)

位置：`saleor/product/managers.py:75`

```python
def visible_to_user(self, requestor, channel, limited_channel_access):
    if has_one_of_permissions(requestor, ALL_PRODUCTS_PERMISSIONS):
        # 管理员权限
        if limited_channel_access:
            if channel:
                # 仅显示分配到该渠道的产品
                return self.filter(
                    Exists(ProductChannelListing.objects.filter(
                        channel_id=channel.id,
                        product_id=OuterRef("pk")
                    ))
                )
            return self.none()
        return self.all()  # 显示所有产品
    
    # 普通用户/匿名用户
    if not channel:
        return self.none()
    return self.published_with_variants(channel)
```

**关键权限** (`ALL_PRODUCTS_PERMISSIONS`):
- `OrderPermissions.MANAGE_ORDERS`
- `DiscountPermissions.MANAGE_DISCOUNTS`
- `ProductPermissions.MANAGE_PRODUCTS`

### 2.2 已发布产品查询 (`ProductsQueryset.published`)

位置：`saleor/product/managers.py:30`

```python
def published(self, channel: Channel):
    if not channel.is_active:
        return self.none()
    today = datetime.datetime.now(tz=datetime.UTC)
    channel_listings = ProductChannelListing.objects.filter(
        Q(published_at__lte=today) | Q(published_at__isnull=True),
        channel_id=channel.id,
        is_published=True,
    ).values("id")
    return self.filter(Exists(channel_listings.filter(product_id=OuterRef("pk"))))
```

**发布条件**（需同时满足）：
1. 渠道 `channel.is_active = True`
2. `ProductChannelListing.is_published = True`
3. `published_at` 为空 **或** `published_at <= 当前时间`

### 2.3 已发布且有可售变体 (`ProductsQueryset.published_with_variants`)

位置：`saleor/product/managers.py:55`

```python
def published_with_variants(self, channel: Channel):
    if not channel.is_active:
        return self.none()
    variant_channel_listings = ProductVariantChannelListing.objects.filter(
        channel_id=channel.id,
        price_amount__isnull=False,  # 必须设置价格
    ).values("id")
    variants = ProductVariant.objects.filter(
        Exists(variant_channel_listings.filter(variant_id=OuterRef("pk")))
    )
    return self.published(channel).filter(
        Exists(variants.filter(product_id=OuterRef("pk")))
    )
```

**额外条件**：产品至少有一个变体在该渠道设置了价格（`price_amount IS NOT NULL`）。

### 2.4 变体检出可见性 (`ProductVariantQueryset.visible_to_user`)

位置：`saleor/product/managers.py:316`

```python
def visible_to_user(self, requestor, channel, limited_channel_access):
    if has_one_of_permissions(requestor, ALL_PRODUCTS_PERMISSIONS):
        if limited_channel_access:
            if channel:
                return self.filter(product__channel_listings__channel_id=channel.id)
            return self.none()
        return self.all()
    
    if not channel or not channel.is_active:
        return self.none()
    
    # 普通用户：变体必须有价格，且产品已发布且在列表中可见
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

**普通用户可见条件**（需同时满足）：
1. 渠道激活
2. 变体在该渠道有价格设置
3. 产品在该渠道已发布（`is_published=True` 且 `published_at` 满足）
4. 产品在该渠道 `visible_in_listings=True`

### 2.5 可见性三字段区别

| 字段 | 定义位置 | 用途 |
|------|---------|------|
| `is_published` | `PublishableModel` | 产品在该渠道的发布开关 |
| `published_at` | `PublishableModel` | 定时发布时间，`null` 表示立即可见 |
| `visible_in_listings` | `ProductChannelListing` | 是否在产品列表/搜索结果中显示，设为 `False` 时仅可通过直接链接访问 |
| `available_for_purchase_at` | `ProductChannelListing` | 可购买时间，控制下单时间窗口 |

---

## 3. 价格选取逻辑

### 3.1 变体价格获取 (`ProductVariant.get_price`)

位置：`saleor/product/models.py:406`

```python
def get_price(
    self,
    channel_listing: "ProductVariantChannelListing",
    price_override: Optional["Decimal"] = None,
    promotion_rules: Iterable["PromotionRule"] | None = None,
) -> "Money":
    if price_override is None:
        # 优先使用预计算的折扣价，否则回退到基准价
        return channel_listing.discounted_price or channel_listing.price
    
    # 自定义价格：需要重新计算促销折扣
    price: Money = self.get_base_price(channel_listing, price_override)
    rules = promotion_rules or []
    return calculate_discounted_price_for_rules(
        price=price, rules=rules, currency=channel_listing.currency
    )
```

**价格优先级**：
1. `discounted_price`（预计算的促销价）→ **优先**
2. 若 `discounted_price` 为 `null`，使用 `price`（基准价）

### 3.2 折扣价预计算 (`update_discounted_prices_for_promotion`)

位置：`saleor/product/utils/variant_prices.py:29`

```python
def update_discounted_prices_for_promotion(
    products: ProductsQueryset, only_dirty_products: bool = False
) -> list[VariantDiscountedPriceChange]:
    # 1. 获取每个变体适用的促销规则
    rules_info_per_variant = get_variants_to_promotion_rules_map(variant_qs)
    
    # 2. 按产品+渠道分组获取变体渠道列表
    product_to_variant_listings_per_channel_map = (
        _get_product_to_variant_channel_listings_per_channel_map(variant_qs)
    )
    
    for product_channel_listing in product_channel_listings:
        # 3. 计算每个变体的折扣价
        discounted_variants_price, variant_listings_to_update, ... = (
            _get_discounted_variants_prices_for_promotions(...)
        )
        
        # 4. 产品折扣价 = 所有变体折扣价中的最小值
        product_discounted_price = min(discounted_variants_price)
        
        # 5. 批量更新
        ProductChannelListing.objects.bulk_update(
            changed_products_listings_to_update, ["discounted_price_amount"]
        )
        ProductVariantChannelListing.objects.bulk_update(
            changed_variants_listings_to_update, ["discounted_price_amount"]
        )
```

**核心逻辑**：
- 遍历产品的所有变体，对每个变体计算最佳促销折扣
- 产品级 `discounted_price` = 所有变体 `discounted_price` 中的最小值
- 通过 `discounted_price_dirty` 标记支持增量更新

### 3.3 单变体促销价格计算 (`_get_discounted_variants_prices_for_promotions`)

位置：`saleor/product/utils/variant_prices.py:254`

```python
def _get_discounted_variants_prices_for_promotions(...):
    for variant_listing in variant_listings:
        # 获取最佳促销折扣（最大优惠）
        applied_discount = calculate_discounted_price_for_promotions(
            price=variant_listing.price,
            rules_info_per_variant=rules_info_per_variant,
            channel=channel,
            variant_id=variant_listing.variant_id,
        )
        
        discounted_variant_price = variant_listing.price
        if applied_discount:
            rule_id, discount = applied_discount
            discounted_variant_price -= discount
            discounted_variant_price = max(
                discounted_variant_price, zero_money(discounted_variant_price.currency)
            )
        
        # 价格变化检测
        if variant_listing.discounted_price != discounted_variant_price:
            variant_listing.discounted_price_amount = discounted_variant_price.amount
            variants_listings_to_update.append(variant_listing)
```

### 3.4 最佳促销规则选择 (`get_best_promotion_discount`)

位置：`saleor/discount/utils/promotion.py:116`

```python
def get_best_promotion_discount(
    price: Money,
    rules_info_for_variant: list[PromotionRuleInfo],
    channel: "Channel",
) -> tuple[UUID, Money] | None:
    available_discounts = []
    for rule_id, discount in get_product_promotion_discounts(...):
        available_discounts.append((rule_id, discount))
    
    if available_discounts:
        # 选择折扣金额最大的促销
        return max(available_discounts, key=lambda x: x[1])
    return None
```

**促销价格规则**：
- 多个促销规则同时适用时，**选择折扣金额最大**的那个（不叠加）
- 折扣后价格不能低于 0

### 3.5 可用性价格计算 (`get_variant_availability`)

位置：`saleor/product/utils/availability.py:178`

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
    
    # 折扣价（含税）
    discounted_price_taxed = _calculate_product_price_with_taxes(
        variant_channel_listing.discounted_price, ...
    )
    # 基准价（含税）
    undiscounted_price_taxed = _calculate_product_price_with_taxes(
        variant_channel_listing.price, ...
    )
    # 原价（含税，用于划线价显示）
    prior_price_taxed = _calculate_product_price_with_taxes(
        variant_channel_listing.prior_price, ...
    )
    
    discount = _get_total_discount(undiscounted_price_taxed, discounted_price_taxed)
    
    is_visible = (
        product_channel_listing is not None and product_channel_listing.is_visible
    )
    is_on_sale = is_visible and discount is not None  # 促销中标记
```

**促销中 (`on_sale`) 判断**：
1. 产品在该渠道可见（`is_visible = True`）
2. 折扣价 < 基准价（存在实际折扣）

### 3.6 价格字段总结

| 字段 | 存储位置 | 说明 |
|------|---------|------|
| `price` | `ProductVariantChannelListing` | 基准价/列表价 |
| `cost_price` | `ProductVariantChannelListing` | 成本价，仅管理员可见 |
| `prior_price` | `ProductVariantChannelListing` | 原价，用于显示划线价（如 ¥199 → ¥99） |
| `discounted_price` | `ProductVariantChannelListing` | 折扣价（含促销），预计算 |
| `discounted_price` | `ProductChannelListing` | 产品级折扣价 = 所有变体折扣价最小值 |

---

## 4. GraphQL 查询过滤逻辑

### 4.1 产品列表查询入口 (`resolve_products`)

位置：`saleor/graphql/product/resolvers.py:116`

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
                .filter(channel_id=channel.id, visible_in_listings=True)
                .values("id")
            )
            qs = qs.filter(
                Exists(product_channel_listings.filter(product_id=OuterRef("pk")))
            )
        else:
            qs = models.Product.objects.none()
    
    return ChannelQsContext(qs=qs, channel_slug=channel.slug if channel else None)
```

**关键**：普通用户查询产品列表时，额外增加 `visible_in_listings=True` 过滤，确保不在列表中显示的产品不会出现在搜索/分类结果中。

### 4.2 可用过滤器 (`ProductFilter`)

位置：`saleor/graphql/product/filters/product.py:94`

```python
class ProductFilter(MetadataFilterBase):
    is_published = django_filters.BooleanFilter(method="filter_is_published")
    published_from = ObjectTypeFilter(...)  # 按发布日期过滤
    is_available = django_filters.BooleanFilter(method="filter_is_available")
    available_from = ObjectTypeFilter(...)  # 按可购买日期过滤
    is_visible_in_listing = django_filters.BooleanFilter(method="filter_listed")
    price = ObjectTypeFilter(input_class=PriceRangeInput, method="filter_variant_price")
    minimal_price = ObjectTypeFilter(...)  # 按产品最低折扣价过滤
    stock_availability = EnumFilter(...)   # 按库存状态过滤
    # ... 其他过滤器
```

**所有 channel 相关过滤器都通过 `get_channel_slug_from_filter_data(self.data)` 获取当前 channel**。

### 4.3 按发布状态过滤 (`filter_products_is_published`)

位置：`saleor/graphql/product/filters/product_helpers.py:216`

```python
def filter_products_is_published(qs, _, value, channel_slug):
    channel = Channel.objects.filter(slug=channel_slug).values("pk")
    
    # 产品渠道列表匹配
    product_channel_listings = ProductChannelListing.objects.filter(
        Exists(channel.filter(pk=OuterRef("channel_id"))),
        is_published=value,
    ).values("product_id")
    
    # 额外校验：至少有一个变体设置了价格
    variant_channel_listings = ProductVariantChannelListing.objects.filter(
        Exists(channel.filter(pk=OuterRef("channel_id"))),
        price_amount__isnull=False,
    ).values("id")
    variants = ProductVariant.objects.filter(
        Exists(variant_channel_listings.filter(variant_id=OuterRef("pk")))
    ).values("product_id")
    
    return qs.filter(
        Exists(product_channel_listings.filter(product_id=OuterRef("pk"))),
        Exists(variants.filter(product_id=OuterRef("pk"))),
    )
```

### 4.4 按列表可见性过滤 (`filter_products_visible_in_listing`)

位置：`saleor/graphql/product/filters/product_helpers.py:288`

```python
def filter_products_visible_in_listing(qs, _, value, channel_slug):
    channel = Channel.objects.filter(slug=channel_slug).values("pk")
    product_channel_listings = ProductChannelListing.objects.filter(
        Exists(channel.filter(pk=OuterRef("channel_id"))),
        visible_in_listings=value,
    ).values("product_id")
    return qs.filter(Exists(product_channel_listings.filter(product_id=OuterRef("pk"))))
```

### 4.5 按价格范围过滤

#### 按变体价格过滤 (`filter_variant_price`)

位置：`saleor/graphql/product/filters/product.py:417`

```python
def filter_variant_price(self, qs, _, value):
    channel_slug = get_channel_slug_from_filter_data(self.data)
    channel_id = Channel.objects.filter(slug=channel_slug).values("pk")
    
    variant_listing = ProductVariantChannelListing.objects.filter(
        Exists(channel_id.filter(pk=OuterRef("channel_id")))
    )
    variant_listing = filter_where_by_numeric_field(
        variant_listing, "price_amount", value  # 按基准价过滤
    )
```

#### 按最低折扣价过滤 (`filter_minimal_price`)

位置：`saleor/graphql/product/filters/product.py:429`

```python
def filter_minimal_price(self, qs, _, value):
    channel_slug = get_channel_slug_from_filter_data(self.data)
    channel = Channel.objects.filter(slug=channel_slug).first()
    
    product_listing = ProductChannelListing.objects.filter(
        channel_id=channel.id
    )
    product_listing = filter_where_by_numeric_field(
        product_listing, "discounted_price_amount", value  # 按产品折扣价过滤
    )
```

**价格过滤区别**：
- `price` 过滤器：按 **变体基准价** (`price_amount`) 过滤，只要有一个变体价格在范围内即返回产品
- `minimal_price` 过滤器：按 **产品最低折扣价** (`discounted_price_amount`) 过滤

### 4.6 按库存状态过滤 (`filter_products_by_stock_availability`)

位置：`saleor/graphql/product/filters/product_helpers.py:100`

```python
def filter_products_by_stock_availability(qs, stock_availability, channel_slug):
    # 获取渠道关联的仓库
    warehouse_pks = get_available_warehouse_pks_for_product(qs, channel_slug)
    
    # 可用库存 = 实际库存 - 已分配 - 已预留
    stocks = Stock.objects.filter(
        warehouse_id__in=warehouse_pks,
        quantity__gt=Coalesce(allocated_subquery, 0) + Coalesce(reservation_subquery, 0),
    ).values("product_variant_id")
    
    variants = ProductVariant.objects.filter(
        Exists(stocks.filter(product_variant_id=OuterRef("pk")))
    ).values("product_id")
    
    if stock_availability == StockAvailability.IN_STOCK.value:
        qs = qs.filter(Exists(variants.filter(product_id=OuterRef("pk"))))
    if stock_availability == StockAvailability.OUT_OF_STOCK.value:
        qs = qs.filter(~Exists(variants.filter(product_id=OuterRef("pk"))))
    return qs
```

**仓库关联逻辑**：
- 新逻辑：通过 `Warehouse.for_channel(channel.id)` 直接关联
- 旧逻辑（兼容）：通过 `ShippingZone` → `Warehouse` 关联
- 由 `Shop.useLegacyShippingZoneStockAvailability` 配置控制

### 4.7 按可购买状态过滤 (`filter_products_is_available`)

位置：`saleor/graphql/product/filters/product_helpers.py:245`

```python
def filter_products_is_available(qs, _, value, channel_slug):
    now = datetime.datetime.now(tz=datetime.UTC)
    if value:
        product_channel_listings = ProductChannelListing.objects.filter(
            ...,
            available_for_purchase_at__lte=now,  # 可购买时间已到
        ).values("product_id")
    else:
        product_channel_listings = ProductChannelListing.objects.filter(
            ...,
            Q(available_for_purchase_at__gt=now)  # 可购买时间未到
            | Q(available_for_purchase_at__isnull=True),  # 未设置可购买时间
        ).values("product_id")
    return qs.filter(Exists(product_channel_listings.filter(product_id=OuterRef("pk"))))
```

---

## 5. 完整可见性判断流程图

```
用户查询产品列表
        │
        ▼
┌─────────────────────────┐
│  获取当前 channel        │
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│  用户权限判断            │
│  - 管理员？ → 可见所有（可按channel过滤）
│  - 普通用户？ → 继续     │
└─────────────────────────┘
        │
        ▼ 普通用户
┌──────────────────────────────────────────┐
│  visible_to_user → published_with_variants │
│  条件：                                     │
│  1. channel.is_active = True               │
│  2. product_channel_listing:               │
│     - is_published = True                  │
│     - published_at <= now OR null          │
│  3. 至少有一个变体在该渠道有价格           │
└──────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────┐
│  额外过滤：resolve_products      │
│  visible_in_listings = True      │
└──────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────┐
│  用户指定的过滤器（可选）         │
│  - is_published                  │
│  - is_available                  │
│  - is_visible_in_listing         │
│  - price / minimal_price         │
│  - stock_availability            │
│  - ... 其他                      │
└──────────────────────────────────┘
        │
        ▼
      结果集
```

---

## 6. 价格计算完整流程

```
变体基准价 (price_amount)
        │
        ▼
┌──────────────────────────┐
│  促销规则匹配            │
│  获取所有适用的促销规则   │
└──────────────────────────┘
        │
        ▼
┌──────────────────────────┐
│  选择最大折扣            │
│  get_best_promotion_discount │
└──────────────────────────┘
        │
        ▼
┌──────────────────────────┐
│  计算折扣价              │
│  discounted_price =      │
│  price - max(discounts)  │
└──────────────────────────┘
        │
        ▼
┌──────────────────────────┐
│  预计算存储              │
│  ProductVariantChannelListing.discounted_price_amount │
└──────────────────────────┘
        │
        ▼
┌──────────────────────────┐
│  产品级聚合              │
│  ProductChannelListing.discounted_price_amount = │
│  min(所有变体折扣价)     │
└──────────────────────────┘
        │
        ▼
┌──────────────────────────┐
│  查询时计算税费          │
│  get_variant_availability │
│  - 含税价格              │
│  - on_sale 标记          │
│  - 折扣金额              │
└──────────────────────────┘
        │
        ▼
      最终展示价格
```

---

## 7. 关键代码位置速查表

| 功能 | 文件位置 |
|------|---------|
| 产品渠道列表模型 | `saleor/product/models.py:304` |
| 变体渠道列表模型 | `saleor/product/models.py:477` |
| 发布基类 | `saleor/core/models.py:63` |
| 产品可见性查询 | `saleor/product/managers.py:75` |
| 变体检出查询 | `saleor/product/managers.py:316` |
| 折扣价预计算 | `saleor/product/utils/variant_prices.py:29` |
| 可用性价格计算 | `saleor/product/utils/availability.py:178` |
| 产品列表解析器 | `saleor/graphql/product/resolvers.py:116` |
| 产品过滤器 | `saleor/graphql/product/filters/product.py:94` |
| 过滤辅助函数 | `saleor/graphql/product/filters/product_helpers.py` |
| 最佳促销选择 | `saleor/discount/utils/promotion.py:116` |
