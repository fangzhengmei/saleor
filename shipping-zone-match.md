# 下单地址校验与配送区匹配协作流程

## 一、整体架构概览

整个流程分为四个核心层次，按调用顺序依次衔接：

```
GraphQL Mutation 入口层
        ↓
地址规整与校验层（I18n + Form）
        ↓
配送区匹配核心层（ShippingMethodQueryset + PostalCode）
        ↓
Checkout 生命周期管理层（地址变更 → 配送方法选择 → 下单最终校验）
```

---

## 二、数据模型层

### 2.1 核心模型关系

**文件**：`saleor/shipping/models.py`

| 模型 | 关键字段 | 说明 |
|------|----------|------|
| `ShippingZone` | `countries` (多国家), `channels`, `default` | 配送区域，包含多个国家 |
| `ShippingMethod` | `shipping_zone_id`, `type` (PRICE_BASED/WEIGHT_BASED), `min/max_order_weight`, `excluded_products` | 配送方法，归属于某个配送区 |
| `ShippingMethodPostalCodeRule` | `shipping_method_id`, `start`, `end`, `inclusion_type` (INCLUDE/EXCLUDE) | 邮编规则，精确到某配送方法 |
| `ShippingMethodChannelListing` | `shipping_method_id`, `channel_id`, `min/max_order_price`, `price` | 渠道维度的价格与订单金额限制 |

**关键关联**：
- `ShippingZone.countries` 使用 `CountryField(multiple=True)`，支持多国家
- `ShippingMethod` 通过外键关联到 `ShippingZone`
- 邮编规则是 `ShippingMethod` 的子对象，通过 `postal_code_rules` 反向关联

---

## 三、地址规整与校验层

### 3.1 模块归属澄清（关键修正）

**存在两个 `i18n.py` 文件，各司其职，互不包含**：

| 文件 | 归属层级 | 核心内容 |
|------|----------|----------|
| `saleor/account/i18n.py` | **业务层** | `CountryAwareAddressForm`、`get_address_form_class()`、地址规范化逻辑 |
| `saleor/graphql/account/i18n.py` | **GraphQL 层** | `I18nMixin`、`validate_address()` 类方法、跳过校验权限逻辑 |

**调用关系**：
```
GraphQL 层 I18nMixin [graphql/account/i18n.py]
    ↓ 调用
account/forms.py [get_address_form()]
    ↓ 调用
业务层表单 [account/i18n.py: CountryAwareAddressForm]
```

### 3.2 调用链总览

```
GraphQL Mutation
    ↓
I18nMixin.validate_address() [graphql/account/i18n.py:155]
    ↓
I18nMixin._validate_address_form() [graphql/account/i18n.py:73]
    ↓
get_address_form() [account/forms.py:6]
    ↓
get_address_form_class() → CountryAwareAddressForm [account/i18n.py:251]
    ↓
CountryAwareAddressForm.validate_address() [account/i18n.py:196]
    ↓
i18naddress.normalize_address() [第三方库]
```

### 3.3 关键节点详解

#### 3.3.1 `I18nMixin.validate_address()` [graphql/account/i18n.py:155]

**参数控制**：
- `format_check`: 是否校验字段格式（默认 `True`）
- `required_check`: 是否校验必填字段（默认 `True`）
- `enable_normalization`: 是否启用地址规范化（默认 `True`）
- `preserve_all_address_fields`: 是否保留非标准字段（从 site settings 读取）

**核心逻辑**：
```python
# 1. 检查 country 必填（不受任何参数影响，始终校验）
if address_data.get("country") is None:
    raise ValidationError(...)

# 2. 处理 skip_validation 权限（🔴 关键：只设置 format_check = False）
if address_data.get("skip_validation"):
    cls.can_skip_address_validation(info)
    format_check = False  # 不影响 required_check

# 3. 调用表单验证
address_form = cls._validate_address_form(...)
address_data = address_form.cleaned_data

# 4. 构造 Address 实例
if not instance:
    instance = Address()
cls.construct_instance(instance, address_data)
```

**关键发现**：`skip_validation` 只影响 `format_check`，`required_check` 始终保持默认 `True`，没有任何调用方传入 `required_check=False`。

#### 3.3.2 `CountryAwareAddressForm.validate_address()` [account/i18n.py:196]

**地址规整四步走**：

1. **字段映射**：将 Saleor 的字段映射为 i18naddress 库的字段
   ```python
   I18N_MAPPING = [
       ("name", ["first_name", "last_name"]),
       ("street_address", ["street_address_1", "street_address_2"]),
       ("city_area", ["city_area"]),
       ("country_area", ["country_area"]),
       ("postal_code", ["postal_code"]),
       ("city", ["city"]),
       ("country_code", ["country"]),
   ]
   ```

2. **无效值替换**：`substitute_invalid_values()` 将不规范的地区名映射为标准值（如 "加州" → "CA"）

3. **规范化**：`i18naddress.normalize_address(data)` 调用第三方库进行地址标准化

4. **非标准字段恢复**：`_restore_non_allowed_fields()` 恢复被规范化过滤掉的、但原始数据中存在的字段

### 3.4 skip_validation 与 required_check 的行为界限

#### 3.4.1 参数交互逻辑

```python
# graphql/account/i18n.py:176-178
if address_data.get("skip_validation"):
    cls.can_skip_address_validation(info)
    format_check = False  # ← 只修改 format_check，required_check 保持 True
```

#### 3.4.2 错误过滤矩阵（实际行为）

| 错误类型 | format_check | required_check | 结果 |
|----------|-------------|----------------|------|
| 格式错误（邮编/电话格式不对） | True | 任意 | 抛出错误 |
| 格式错误（邮编/电话格式不对） | False | 任意 | **跳过，保留原始值到 cleaned_data** |
| 必填错误（字段为空） | 任意 | True | 抛出错误 |
| 必填错误（字段为空） | 任意 | False | 有值则接受，无值则忽略 |

#### 3.4.3 实际生效组合

| 场景 | format_check | required_check | 行为 |
|------|-------------|----------------|------|
| **默认校验** | True | True | 格式+必填全部校验 |
| **skip_validation=True** | False | True | 跳过格式校验，但必填字段仍然检查 |
| **理论上的宽松模式** | False | False | 全部跳过（目前无调用方使用） |

**重要结论**：`skip_validation=True` 并不意味着完全跳过校验，只是跳过格式校验，必填字段（如 country、postal_code 等）仍然会检查。`country` 字段在 `validate_address()` 入口处单独检查，不受任何参数影响。

---

## 四、配送区匹配核心层

### 4.1 总调用链

```
get_valid_internal_shipping_methods_for_checkout_info() [checkout/delivery_context.py:176]
    ↓
ShippingMethodQueryset.applicable_shipping_methods_for_instance() [shipping/models.py:170]
    ↓
ShippingMethodQueryset.applicable_shipping_methods() [shipping/models.py:140]
    ↓ 价格/重量过滤
filter_shipping_methods_by_postal_code_rules() [shipping/postal_codes.py:117]
    ↓
is_shipping_method_applicable_for_postal_code() [shipping/postal_codes.py:96]
    ↓
check_postal_code_in_range() [shipping/postal_codes.py:75]
```

### 4.2 `applicable_shipping_methods_for_instance()` [shipping/models.py:170]

**参数**：
- `instance`: Checkout 或 Order 对象
- `channel_id`: 渠道 ID
- `price`: 订单小计金额（Money 对象）
- `shipping_address`: 配送地址
- `country_code`: 国家代码（可从地址自动提取）
- `lines`: 订单行，用于提取产品 ID 和计算重量

**核心逻辑**：

```python
def applicable_shipping_methods_for_instance(...):
    # 1. 无地址直接返回 None
    if not shipping_address:
        return None

    # 2. 提取国家代码
    if not country_code:
        country_code = shipping_address.country.code

    # 3. 提取产品 ID（用于排除规则）
    instance_product_ids = {
        line.variant.product_id for line in lines if line.variant
    }

    # 4. 计算重量
    if isinstance(instance, Checkout):
        weight = calculate_checkout_weight(lines)
    else:
        weight = instance.weight

    # 5. 调用基础匹配方法
    applicable_methods = self.applicable_shipping_methods(
        price=price,
        channel_id=channel_id,
        weight=weight,
        country_code=country_code,
        product_ids=instance_product_ids,
    ).prefetch_related("postal_code_rules")

    # 6. 邮编规则过滤
    return filter_shipping_methods_by_postal_code_rules(
        applicable_methods, shipping_address
    )
```

### 4.3 `applicable_shipping_methods()` [shipping/models.py:140]

**四层过滤**：

| 过滤层级 | 代码位置 | 说明 |
|----------|----------|------|
| 国家 + 渠道 | L148-153 | `shipping_zone__countries__contains=country_code` AND `shipping_zone__channels__id=channel_id` |
| 产品排除 | L160-161 | 排除 `excluded_products` 中包含的产品 |
| 价格区间 | L162-164 | 调用 `_applicable_price_based_methods()` |
| 重量区间 | L165 | 调用 `_applicable_weight_based_methods()` |

### 4.4 邮编规则匹配 [shipping/postal_codes.py]

#### 4.4.1 `check_postal_code_in_range()` [L75]

**国家特殊处理**：
```python
country_func_map = {
    "GB": check_uk_postal_code,      # 英国：BH20 2BC
    "IM": check_uk_postal_code,      # 马恩岛
    "GG": check_uk_postal_code,      # 根西岛
    "JE": check_uk_postal_code,      # 泽西岛
    "IE": check_irish_postal_code,   # 爱尔兰：A65 2F0A
}
```

**UK 邮编解析** [L45-54]：
- 正则：`r"^([A-Z]{1,2})([0-9]+)([A-Z]?) ?([0-9][A-Z]{2})$"`
- 分组：(字母)(数字)(可选字母) (数字+字母)
- 第二组转 int 后进行区间比较

#### 4.4.2 `is_shipping_method_applicable_for_postal_code()` [L96]

**规则判定逻辑**：
```python
results = {rule: check_postal_code_in_range(...)}

if not results:
    return True  # 无规则 → 适用

# 全 INCLUDE 规则：任一命中即适用
if all(rule.inclusion_type == INCLUDE):
    return any(results.values())

# 全 EXCLUDE 规则：任一命中即不适用
if all(rule.inclusion_type == EXCLUDE):
    return not any(results.values())

# 混合规则 → 不支持，返回 False
return False
```

---

## 五、Checkout 生命周期管理层

### 5.1 场景一：更新配送地址

**Mutation**：`CheckoutShippingAddressUpdate` [graphql/checkout/mutations/checkout_shipping_address_update.py]

```mermaid
sequenceDiagram
    participant Client
    participant Mutation
    participant I18n
    participant CheckoutUtil
    participant DeliveryCtx

    Client->>Mutation: checkoutShippingAddressUpdate
    Note over Mutation: 检查是否为自提（collection_point_id），是则拒绝
    Mutation->>I18n: validate_address()
    I18n-->>Mutation: 返回规范化后的 Address 实例
    Mutation->>CheckoutUtil: change_shipping_address_in_checkout()
    Note over CheckoutUtil: 更新 checkout.shipping_address
    Mutation->>CheckoutUtil: mark_checkout_deliveries_as_stale_if_needed()
    Note over CheckoutUtil: 设置 delivery_methods_stale_at = 过去时间
    Mutation->>CheckoutUtil: invalidate_checkout()
    Note over CheckoutUtil: 设置 price_expiration，触发价格重算
    Mutation-->>Client: 返回 checkout
```

**关键函数**：
- `change_shipping_address_in_checkout()` [checkout/utils.py:402]：保存地址并返回需要更新的字段
- `mark_checkout_deliveries_as_stale_if_needed()`：地址变更后，配送方法缓存失效

### 5.2 场景二：选择配送方法

**Mutation**：`CheckoutShippingMethodUpdate` [graphql/checkout/mutations/checkout_shipping_method_update.py]

```mermaid
sequenceDiagram
    participant Client
    participant Mutation
    participant DeliveryCtx

    Client->>Mutation: checkoutShippingMethodUpdate
    Mutation->>DeliveryCtx: get_or_fetch_checkout_deliveries()
    alt 缓存有效
        DeliveryCtx-->>Mutation: 返回已缓存的 CheckoutDelivery 列表
    else 缓存失效
        DeliveryCtx->>DeliveryCtx: fetch_shipping_methods_for_checkout()
        Note over DeliveryCtx: 调用 get_available_built_in_shipping_methods_for_checkout_info()
        Note over DeliveryCtx: → applicable_shipping_methods_for_instance()
        Note over DeliveryCtx: → 邮编规则过滤
        Note over DeliveryCtx: 调用外部 webhook 获取额外配送方法
        Note over DeliveryCtx: 调用 CHECKOUT_FILTER_SHIPPING_METHODS webhook 过滤
        DeliveryCtx-->>Mutation: 返回 CheckoutDelivery 列表
    end
    Mutation->>Mutation: 检查 shipping_method_id 是否在有效列表中
    Mutation->>DeliveryCtx: assign_delivery_method_to_checkout()
    Note over DeliveryCtx: 更新 checkout.assigned_delivery_id
    Mutation-->>Client: 返回 checkout
```

**关键函数**：
- `get_or_fetch_checkout_deliveries()` [checkout/delivery_context.py:732]：带 TTL 缓存，`CHECKOUT_DELIVERY_OPTIONS_TTL` 控制有效期
- `fetch_shipping_methods_for_checkout()` [checkout/delivery_context.py:587]：完整获取流程

### 5.3 场景三：下单最终校验

**入口**：`complete_checkout_pre_payment_part()` → `_prepare_checkout_with_payment()` [checkout/complete_checkout.py:1088]

```mermaid
sequenceDiagram
    participant Client
    participant Complete
    participant Cleaner
    participant DeliveryCtx

    Client->>Complete: checkoutComplete
    Complete->>Complete: fetch_checkout_data()
    Note over Complete: 触发价格、税金、配送方法的全量重算
    Complete->>Cleaner: clean_checkout_shipping()
    Note over Cleaner: 三重校验
    alt 校验不通过
        Cleaner-->>Complete: 抛出 ValidationError
    else 校验通过
        Complete->>Complete: 继续支付与订单创建
    end
```

**`clean_checkout_shipping()` 三重校验** [checkout/checkout_cleaner.py:25]：

| 校验项 | 代码 | 错误码 |
|--------|------|--------|
| 配送方法是否设置 | L37-44 | `SHIPPING_METHOD_NOT_SET` |
| 配送地址是否设置 | L46-54 | `SHIPPING_ADDRESS_NOT_SET` |
| 配送方法是否对地址有效 | L55-64 | `INVALID_SHIPPING_METHOD` |

**第三重校验核心**：
```python
if not delivery_method_info.is_method_in_valid_methods(checkout_info):
    if checkout_info.checkout.collection_point_id:
        clear_cc_delivery_method(checkout_info)
    raise ValidationError(
        "Delivery method is not valid for your shipping address",
        code=error_code.INVALID_SHIPPING_METHOD.value,
    )
```

`is_method_in_valid_methods()` 对于 `ShippingMethodInfo` 实现 [checkout/delivery_context.py:97-98]：
```python
def is_method_in_valid_methods(self, checkout_info) -> bool:
    return self.delivery_method.active
```

即：配送方法数据对象的 `active` 字段决定是否有效。该字段在 `initialize_shipping_method_active_status()` [shipping/utils.py:130] 中根据排除规则设置。

---

## 六、关键衔接点总结

### 6.1 地址 → 配送区 的数据流转

| 步骤 | 数据来源 | 处理 | 输出 |
|------|----------|------|------|
| 1 | 用户输入地址 | I18n 规范化 | `Address` 实例 |
| 2 | `Address.country.code` | `ShippingZone.countries__contains` 查询 | 匹配国家的 `ShippingZone` 列表 |
| 3 | `ShippingZone.id` | 关联查询 `ShippingMethod` | 候选配送方法列表 |
| 4 | `Address.postal_code` | 邮编规则逐行校验 | 最终可用配送方法 |

### 6.2 缓存失效时机

| 操作 | 缓存处理 | 位置 |
|------|----------|------|
| 更新配送地址 | `mark_checkout_deliveries_as_stale_if_needed()` 设置 `delivery_methods_stale_at` 为过去时间 | `checkout_shipping_address_update.py:205` |
| 更新商品 | `invalidate_checkout()` 触发价格失效，间接导致配送方法重算 | `checkout/utils.py:76` |
| 优惠券变更 | 同上 | - |
| TTL 过期 | `delivery_methods_stale_at <= now` 时重新获取 | `checkout/delivery_context.py:743` |

### 6.3 错误码汇总

| 错误码 | 触发时机 | 所在模块 |
|--------|----------|----------|
| `SHIPPING_CHANGE_FORBIDDEN` | 自提订单尝试修改配送地址 | `checkout_shipping_address_update.py:163` |
| `SHIPPING_METHOD_NOT_APPLICABLE` | 选择的配送方法不在有效列表中 | `checkout_shipping_method_update.py:135` |
| `SHIPPING_NOT_REQUIRED` | 不需要配送的商品尝试设置配送方法 | `checkout_shipping_method_update.py:182` |
| `SHIPPING_METHOD_NOT_SET` | 下单时未选择配送方法 | `checkout_cleaner.py:41` |
| `SHIPPING_ADDRESS_NOT_SET` | 下单时未设置配送地址 | `checkout_cleaner.py:49` |
| `INVALID_SHIPPING_METHOD` | 配送方法对当前地址无效 | `checkout_cleaner.py:61` |

---

## 七、核心文件清单

| 文件 | 职责 |
|------|------|
| `saleor/account/i18n.py` | 业务层：地址规范化表单（CountryAwareAddressForm）、表单构造 |
| `saleor/graphql/account/i18n.py` | GraphQL 层：I18nMixin、validate_address() 类方法、跳过校验权限 |
| `saleor/account/forms.py` | 地址表单工厂 |
| `saleor/shipping/models.py` | 配送区/方法数据模型 + QuerySet 匹配逻辑 |
| `saleor/shipping/postal_codes.py` | 邮编区间匹配算法 |
| `saleor/shipping/utils.py` | 配送方法数据转换与状态初始化 |
| `saleor/checkout/delivery_context.py` | 配送上下文、缓存管理、方法获取 |
| `saleor/checkout/checkout_cleaner.py` | 下单前最终校验 |
| `saleor/checkout/complete_checkout.py` | 完整下单流程编排 |
| `saleor/graphql/checkout/mutations/checkout_shipping_address_update.py` | 更新配送地址 Mutation |
| `saleor/graphql/checkout/mutations/checkout_shipping_method_update.py` | 选择配送方法 Mutation |

---

## 八、附录 A：I18nMixin 归属与继承体系

### 8.1 双 i18n.py 模块边界澄清（与第三章结论统一）

**核心发现**：存在**两个独立的 `i18n.py` 文件**，归属不同层级，各司其职：

| 文件 | 归属层级 | 核心职责 | 关键类/函数 |
|------|----------|----------|------------|
| `saleor/account/i18n.py` | **业务层** | 地址表单定义、地址规范化逻辑 | `CountryAwareAddressForm`、`get_address_form_class()`、`validate_address()` 实例方法 |
| `saleor/graphql/account/i18n.py` | **GraphQL 层** | Mutation 地址校验入口、权限控制、错误处理 | `I18nMixin`、`validate_address()` 类方法、`SKIP_ADDRESS_VALIDATION_PERMISSION_MAP` |

**分层设计意图**：
1. **业务层 `account/i18n.py`**：不感知 GraphQL，可独立被其他业务模块调用
2. **GraphQL 层 `graphql/account/i18n.py`**：纯横向能力注入，为 Mutation 提供统一的地址校验入口

**调用流向**：
```
Mutation（继承 I18nMixin）
    ↓ [graphql/account/i18n.py]
I18nMixin.validate_address() 类方法
    ↓
_validate_address_form()
    ↓ [account/forms.py]
get_address_form()
    ↓ [account/i18n.py]
CountryAwareAddressForm.validate_address() 实例方法
    ↓
i18naddress.normalize_address() [第三方库]
```

**I18nMixin 精确定位**：
- **位置**：`saleor/graphql/account/i18n.py:58`
- **性质**：纯 GraphQL 层工具类，与业务模型层解耦
- **设计意图**：为 Mutation 提供地址校验的横向能力

### 8.2 继承关系图

```
graphene.Mutation
    ↓
BaseMutation [graphql/core/mutations.py:142]
    ↓  (多继承)
CheckoutShippingAddressUpdate(AddressMetadataMixin, BaseMutation, I18nMixin)
                                  ↑
                        地址校验能力注入点
```

**继承顺序说明**：
1. `AddressMetadataMixin`：提供元数据处理能力
2. `BaseMutation`：提供基础 Mutation 框架（错误处理、权限检查等）
3. `I18nMixin`：注入地址校验方法

### 8.3 使用 I18nMixin 的 Mutation 清单

| Mutation | 用途 |
|----------|------|
| `CheckoutShippingAddressUpdate` | 更新结账配送地址 |
| `CheckoutBillingAddressUpdate` | 更新结账账单地址 |
| `CheckoutCreate` | 创建结账 |
| `CheckoutComplete` | 完成结账（下单） |
| `CheckoutPaymentCreate` | 创建支付 |
| `AddressCreate` | 创建用户地址 |
| `BaseAddressUpdate` | 更新地址基类 |
| `BaseCustomerCreate` | 创建客户基类 |
| `CustomerBulkUpdate` | 批量更新客户 |
| `OrderUpdate` | 更新订单 |
| `DraftOrderCreate` | 创建草稿订单 |
| `DraftOrderUpdate` | 更新草稿订单 |
| `OrderBulkCreate` | 批量创建订单 |
| `WarehouseShippingZoneAssign` | 仓库配送区分配 |
| `WarehouseShippingZoneUnassign` | 仓库配送区解除 |
| `ShopAddressUpdate` | 更新店铺地址 |

### 8.4 Skip Validation 机制完整流程

#### 8.4.1 权限映射表 [graphql/account/i18n.py:20-55]

每个 Mutation 有独立的跳过校验权限配置：

```python
SKIP_ADDRESS_VALIDATION_PERMISSION_MAP = {
    "checkoutCreate": [
        CheckoutPermissions.HANDLE_CHECKOUTS,
        AuthorizationFilters.AUTHENTICATED_APP,
    ],
    "checkoutShippingAddressUpdate": [
        CheckoutPermissions.HANDLE_CHECKOUTS,
        AuthorizationFilters.AUTHENTICATED_APP,
    ],
    # ... 其他 Mutation
}
```

#### 8.4.2 执行时序

```mermaid
sequenceDiagram
    participant Client
    participant Mutation
    participant I18nMixin
    participant Permission

    Client->>Mutation: 提交地址 + skipValidation: true
    Mutation->>I18nMixin: validate_address(address_data, skip_validation=True)
    
    Note over I18nMixin: 第一步：权限检查
    I18nMixin->>Permission: can_skip_address_validation(info)
    Note over Permission: 根据 info.field_name（Mutation 名）<br/>查询 SKIP_ADDRESS_VALIDATION_PERMISSION_MAP
    
    alt 权限不足
        Permission-->>I18nMixin: 抛出 PermissionDenied
        I18nMixin-->>Mutation: 异常终止
        Mutation-->>Client: 返回权限错误
    else 权限通过
        Permission-->>I18nMixin: 验证通过
        Note over I18nMixin: format_check = False<br/>required_check = True（保持默认）
    end
    
    Note over I18nMixin: 第二步：表单验证
    I18nMixin->>I18nMixin: _validate_address_form()
    Note over I18nMixin: 遍历 errors_dict<br/>• 格式错误 → 因 format_check=False 跳过<br/>• 必填错误 → 因 required_check=True 仍然抛出
    
    alt 存在必填错误（如 postal_code/city 为空）
        I18nMixin-->>Mutation: 抛出 ValidationError（code="required"）
        Mutation-->>Client: 返回必填字段错误
    else 仅格式错误或无错误
        Note over I18nMixin: 第三步：标记状态
        I18nMixin->>I18nMixin: cleaned_data["validation_skipped"] = True
        
        Note over I18nMixin: 第四步：持久化标记
        I18nMixin-->>Mutation: 返回 Address 实例<br/>(validation_skipped=True)
        
        Mutation->>DB: 保存 Address，validation_skipped 字段写入数据库
        Mutation-->>Client: 返回成功响应
    end
```

**时序图代码验证说明**：

```python
# graphql/account/i18n.py:176-178
if address_data.get("skip_validation"):
    cls.can_skip_address_validation(info)
    format_check = False  # ← 只修改 format_check
    # required_check 保持默认 True，未被修改！

# graphql/account/i18n.py:145-150
# 错误过滤逻辑：
# • error.code != "required": if values_check (即 format_check)
# • error.code == "required": if required_check (始终为 True)
```

#### 8.4.2.1 skip_validation 与 required_check 的行为界限（关键澄清）

**核心代码** [graphql/account/i18n.py:176-178]：
```python
if address_data.get("skip_validation"):
    cls.can_skip_address_validation(info)
    format_check = False  # ← 只修改 format_check，required_check 保持默认 True
```

**行为对照表**：

| 参数 | 含义 | 受 skip_validation 影响？ | 默认值 |
|------|------|--------------------------|--------|
| `format_check` | 是否校验字段格式（邮编、电话格式等） | ✅ 是，设为 `False` | `True` |
| `required_check` | 是否校验必填字段 | ❌ 否，保持不变 | `True` |

**实际生效组合**：

| 场景 | format_check | required_check | 行为 |
|------|-------------|----------------|------|
| **默认校验** | `True` | `True` | 格式+必填全部校验 |
| **`skip_validation=True`** | `False` | `True` | 跳过格式校验，但必填字段仍然检查 |
| **理论宽松模式** | `False` | `False` | 全部跳过（无调用方使用） |

**错误过滤矩阵**：

| 错误类型 | format_check | required_check | 结果 |
|----------|-------------|----------------|------|
| 格式错误（邮编/电话格式不对） | `True` | 任意 | 抛出错误 |
| 格式错误（邮编/电话格式不对） | `False` | 任意 | 跳过，保留原始值到 `cleaned_data` |
| 必填错误（字段为空） | 任意 | `True` | 抛出错误 |
| 必填错误（字段为空） | 任意 | `False` | 有值则接受，无值则忽略 |

**重要结论**：
1. `skip_validation=True` 并不意味着完全跳过校验，只是跳过格式校验
2. 必填字段（如 country、postal_code、city 等）仍然会检查
3. `country` 字段在 `validate_address()` 入口处单独检查，**不受任何参数影响**，始终校验

#### 8.4.3 validation_skipped 标记的下游影响

| 模块 | 检测点 | 行为 |
|------|--------|------|
| `Address.as_data()` | `account/models.py:118` | 若 `validation_skipped=True`，跳过电话号码 E.164 格式化 |
| `Avatax Plugin` | `plugins/avatax/plugin.py:364` | 税金计算前输出警告日志 |
| `Order Calculations` | `order/calculations.py:442` | 税金计算前输出警告日志 |
| `CheckoutComplete` | `graphql/checkout/mutations/checkout_complete.py:205,228` | 若未跳过校验，则调用 I18n 重新校验地址 |

**警告日志示例**：
```python
# checkout/utils.py:947-956
def log_address_if_validation_skipped_for_checkout(checkout_info, logger):
    address = get_address_for_checkout_taxes(checkout_info)
    if address and address.validation_skipped:
        logger.warning(
            "Fetching tax data for checkout with address validation skipped. "
            "Address ID: %s",
            address.id,
        )
```

---

## 九、附录 B：校验失败原因传递路径

### 9.1 错误消息完整链路

```
Django Form Errors (dict)
        ↓ [attach_params_to_address_form_errors]
    按 format_check/required_check 过滤
        ↓
ValidationError(error_dict)
        ↓ [BaseMutation.mutate 捕获]
handle_errors(e) → validation_error_to_error_type()
        ↓
ErrorType 实例列表 (含 field/message/code)
        ↓
GraphQL Response { errors: [...] }
```

### 9.2 关键节点详解

#### 9.2.1 Django Form → ValidationError 转换 [graphql/account/i18n.py:73-122]

```python
def _validate_address_form(cls, address_data, ...):
    # 1. 执行表单验证
    if not address_form.is_valid():
        validation_skipped = True
        
        # 2. 错误处理入口
        errors = cls.attach_params_to_address_form_errors(
            address_form, params, format_check, required_check
        )
        
        # 3. 有错误才抛出
        if errors:
            raise ValidationError(errors)
```

#### 9.2.2 错误过滤逻辑 [graphql/account/i18n.py:125-152]

`attach_params_to_address_form_errors()` 实现了条件性错误抛出：

```python
def attach_params_to_address_form_errors(...):
    address_errors_dict = address_form.errors.as_data()
    
    for field, errors in address_errors_dict.items():
        for error in errors:
            
            # 情形 A：格式错误（code != "required"）
            if error.code != "required":
                if values_check:  # 即 format_check
                    errors_dict[field] = errors  # 加入错误
                else:
                    # 跳过校验：保留原始值到 cleaned_data
                    address_form.cleaned_data[field] = address_form.data[field]
            
            # 情形 B：必填错误（code == "required"）
            if error.code == "required":
                field_value = address_form.data.get(field)
                if required_check:
                    errors_dict[field] = errors  # 加入错误
                elif field_value is not None:
                    # 有值则接受：即使不符合表单要求
                    address_form.cleaned_data[field] = field_value
    
    return errors_dict
```

**过滤矩阵**：

| 错误类型 | format_check | required_check | 结果 |
|----------|-------------|----------------|------|
| 格式错误 | True | 任意 | 抛出错误 |
| 格式错误 | False | 任意 | 忽略，保留原始值 |
| 必填错误 | 任意 | True | 抛出错误 |
| 必填错误 | 任意 | False | 有值则接受，无值则忽略 |

#### 9.2.3 ValidationError → GraphQL Error 转换 [graphql/core/mutations.py:87-118]

`validation_error_to_error_type()` 负责将 Django 异常转换为 GraphQL 错误对象：

```python
def validation_error_to_error_type(validation_error: ValidationError, error_type_class):
    err_list = []
    error_class_fields = set(error_type_class._meta.fields.keys())
    
    if hasattr(validation_error, "error_dict"):
        # 字段级错误
        for field_label, field_errors in validation_error.error_dict.items():
            # 转换字段名：snake_case → camelCase
            field = snake_to_camel_case(field_label) if field_label != NON_FIELD_ERRORS else None
            
            for err in field_errors:
                error = error_type_class(
                    field=field,
                    message=err.messages[0],  # 取第一条消息
                    code=get_error_code_from_error(err),
                )
                # 附加额外参数（如 address_type）
                attach_error_params(error, err.params, error_class_fields)
                err_list.append(error)
    else:
        # 非字段错误
        for err in validation_error.error_list:
            error = error_type_class(
                message=err.messages[0],
                code=get_error_code_from_error(err),
            )
            attach_error_params(error, err.params, error_class_fields)
            err_list.append(error)
    
    return err_list
```

#### 9.2.4 BaseMutation 异常捕获机制 [graphql/core/mutations.py:518-547]

```python
def mutate(cls, root, info: ResolveInfo, **data):
    disallow_replica_in_context(info.context)
    setup_context_user(info.context)

    if not cls.check_permissions(info.context, data=data):
        raise PermissionDenied(permissions=cls._meta.permissions)

    try:
        # 执行具体 Mutation 逻辑
        response = cls.perform_mutation(root, info, **data)
        if response.errors is None:
            response.errors = []
        return response
    except ValidationError as e:
        # 统一转换为 GraphQL 错误格式
        return cls.handle_errors(e)

@classmethod
def handle_errors(cls, error: ValidationError, **extra):
    error_list = validation_error_to_error_type(error, cls._meta.error_type_class)
    return cls.handle_typed_errors(error_list, **extra)

@classmethod
def handle_typed_errors(cls, errors: list, **extra):
    if cls._meta.error_type_field is not None:
        extra.update({cls._meta.error_type_field: errors})
    # 返回包含 errors 字段的响应，而非抛出异常
    return cls(errors=errors, **extra)
```

### 9.3 典型错误响应示例

**请求**（邮编格式错误）：
```graphql
mutation {
  checkoutShippingAddressUpdate(
    id: "Q2hlY2tvdXQ6MQ=="
    shippingAddress: {
      country: US
      postalCode: "INVALID"
      streetAddress1: "123 Main St"
      city: "New York"
    }
  ) {
    checkout {
      id
    }
    errors {
      field
      message
      code
    }
  }
}
```

**响应**（校验失败）：
```json
{
  "data": {
    "checkoutShippingAddressUpdate": {
      "checkout": null,
      "errors": [
        {
          "field": "postalCode",
          "message": "Invalid postal code format.",
          "code": "INVALID"
        }
      ]
    }
  }
}
```

**响应**（skipValidation + 权限通过）：
```json
{
  "data": {
    "checkoutShippingAddressUpdate": {
      "checkout": {
        "id": "Q2hlY2tvdXQ6MQ==",
        "shippingAddress": {
          "postalCode": "INVALID",
          "validationSkipped": true
        }
      },
      "errors": []
    }
  }
}
```

---

## 十、附录 C：配送方法失效原因到错误码映射链路

### 10.1 链路总览

```
Webhook 返回 excluded_methods
        ↓ [shipping/webhooks/shared.py]
get_excluded_shipping_data() → 合并多 webhook reason
        ↓ [checkout/delivery_context.py:633-634]
initialize_shipping_method_active_status() → 设置 active=False + message=reason
        ↓ [两种校验场景]
        ├─ 选配送时 → SHIPPING_METHOD_NOT_APPLICABLE
        └─ 下单时 → INVALID_SHIPPING_METHOD
```

### 10.2 第一段：Excluded Reason 写入 active/message

#### 10.2.1 数据结构定义 [shipping/interface.py:16-59]

```python
@dataclass
class ShippingMethodData:
    id: str
    price: Money
    name: str | None = None
    # ... 其他字段
    active: bool = True       # 标记是否可用
    message: str = ""         # 失效原因描述

@dataclass
class ExcludedShippingMethod:
    id: str                    # 配送方法 ID
    reason: str | None         # 排除原因
```

#### 10.2.2 Webhook 响应解析流程 [shipping/webhooks/shared.py]

**四步处理链**：

```
Webhook 响应 JSON
    ↓
FilterShippingMethodsSchema.model_validate() → Pydantic 校验
    ↓
_get_excluded_shipping_methods_from_response() → 提取 excluded_methods
    ↓
_parse_excluded_shipping_methods() → 转为 {method_id: [ExcludedShippingMethod]}
    ↓
merge_excluded_methods_map() → 合并多 webhook reason（空格连接）
```

**合并逻辑** [shipping/webhooks/shared.py:164-173]：
```python
def merge_excluded_methods_map(excluded_methods_map):
    excluded_methods = []
    for method_id, methods in excluded_methods_map.items():
        reason = None
        if reasons := [m.reason for m in methods if m.reason]:
            reason = " ".join(reasons)  # 多原因空格连接
        excluded_methods.append(ExcludedShippingMethod(id=method_id, reason=reason))
    return excluded_methods
```

#### 10.2.3 写入 ShippingMethodData [shipping/utils.py:130-141]

```python
def initialize_shipping_method_active_status(
    shipping_methods: list["ShippingMethodData"],
    excluded_methods: list["ExcludedShippingMethod"],
):
    reason_map = {str(method.id): method.reason for method in excluded_methods}
    for instance in shipping_methods:
        instance.active = True      # 默认激活
        instance.message = ""
        reason = reason_map.get(str(instance.id))
        if reason is not None:
            instance.active = False  # 被排除 → 标记为失效
            instance.message = reason  # 写入失效原因
```

**调用点** [checkout/delivery_context.py:632-634]：
```python
def with_excluded_methods(excluded_methods: list[ExcludedShippingMethod]):
    initialize_shipping_method_active_status(all_methods, excluded_methods)
    # ... 后续处理 CheckoutDelivery
```

### 10.3 第二段：选配送方法时报错

**Mutation**：`checkoutShippingMethodUpdate` [graphql/checkout/mutations/checkout_shipping_method_update.py]

#### 10.3.1 校验流程

```mermaid
sequenceDiagram
    participant Client
    participant Mutation
    participant DeliveryCtx

    Client->>Mutation: checkoutShippingMethodUpdate(shipping_method_id)
    Mutation->>DeliveryCtx: get_or_fetch_checkout_deliveries()
    Note over DeliveryCtx: 返回 CheckoutDelivery 列表<br/>(仅 is_valid=True)
    
    Mutation->>Mutation: 遍历 checkout_deliveries
    Note over Mutation: 只考虑 method.active == True 的方法
    
    alt 找到匹配的 active 方法
        Mutation->>DeliveryCtx: assign_delivery_method_to_checkout()
        Mutation-->>Client: 返回成功
    else 未找到匹配
        Mutation-->>Client: 抛出 ValidationError<br/>code=SHIPPING_METHOD_NOT_APPLICABLE
    end
```

#### 10.3.2 核心校验代码 [graphql/checkout/mutations/checkout_shipping_method_update.py:109-139]

```python
@classmethod
def get_checkout_delivery(cls, checkout_info, shipping_method_id, requestor):
    if shipping_method_id is None:
        return None
    
    # 1. 获取有效配送列表（含 active 标记）
    checkout_deliveries = get_or_fetch_checkout_deliveries(
        checkout_info, requestor=requestor
    ).get()
    
    internal_shipping_method_id = cls._resolve_delivery_method_id(shipping_method_id)
    
    # 2. 只在 active=True 的方法中查找
    for method in checkout_deliveries:
        if not method.active:  # 🔴 关键点：跳过失效方法
            continue
        if method.shipping_method_id == internal_shipping_method_id:
            return method  # ✅ 找到有效方法
    
    # 3. 未找到 → 抛出错误
    raise ValidationError(
        {
            "shipping_method": ValidationError(
                "This shipping method is not applicable.",
                code=CheckoutErrorCode.SHIPPING_METHOD_NOT_APPLICABLE.value,
            )
        }
    )
```

**关键特性**：
- 错误消息："This shipping method is not applicable."
- 错误码：`SHIPPING_METHOD_NOT_APPLICABLE`
- 不暴露具体失效原因（message 字段不传递到错误响应）

### 10.4 第三段：下单最终校验时报错

**入口**：`clean_checkout_shipping()` [checkout/checkout_cleaner.py:25-65]

#### 10.4.1 校验流程

```mermaid
sequenceDiagram
    participant Complete
    participant Cleaner
    participant ShippingMethodInfo

    Complete->>Cleaner: clean_checkout_shipping()
    Note over Cleaner: 第1重：delivery_method 是否设置？
    alt 未设置
        Cleaner-->>Complete: SHIPPING_METHOD_NOT_SET
    else 已设置
        Note over Cleaner: 第2重：is_valid_delivery_method()？
        alt 无效（无 shipping_address）
            Cleaner-->>Complete: SHIPPING_ADDRESS_NOT_SET
        else 有效
            Note over Cleaner: 第3重：is_method_in_valid_methods()？
            Cleaner->>ShippingMethodInfo: is_method_in_valid_methods(checkout_info)
            ShippingMethodInfo-->>Cleaner: return delivery_method.active
            alt active=False（已失效）
                Cleaner-->>Complete: INVALID_SHIPPING_METHOD
            else active=True（有效）
                Cleaner-->>Complete: 校验通过
            end
        end
    end
```

#### 10.4.2 三重校验代码 [checkout/checkout_cleaner.py:25-65]

```python
def clean_checkout_shipping(checkout_info, lines, error_code):
    delivery_method_info = checkout_info.get_delivery_method_info()

    if is_shipping_required(lines):
        # 第1重：配送方法是否设置
        if not delivery_method_info.delivery_method:
            raise ValidationError(
                {
                    "shipping_method": ValidationError(
                        "Shipping method is not set",
                        code=error_code.SHIPPING_METHOD_NOT_SET.value,
                    )
                }
            )
        
        # 第2重：配送地址是否设置
        if not delivery_method_info.is_valid_delivery_method():
            raise ValidationError(
                {
                    "shipping_address": ValidationError(
                        "Shipping address is not set",
                        code=error_code.SHIPPING_ADDRESS_NOT_SET.value,
                    )
                }
            )
        
        # 第3重：配送方法是否对当前地址有效 🔴
        if not delivery_method_info.is_method_in_valid_methods(checkout_info):
            if checkout_info.checkout.collection_point_id:
                clear_cc_delivery_method(checkout_info)
            raise ValidationError(
                {
                    "shipping_method": ValidationError(
                        "Delivery method is not valid for your shipping address",
                        code=error_code.INVALID_SHIPPING_METHOD.value,
                    )
                }
            )
```

#### 10.4.3 `is_method_in_valid_methods()` 实现

**ShippingMethodInfo** [checkout/delivery_context.py:97-98]：
```python
def is_method_in_valid_methods(self, checkout_info) -> bool:
    return self.delivery_method.active  # 直接读取 active 字段
```

**CollectionPointInfo** [checkout/delivery_context.py:158-162]（自提场景）：
```python
def is_method_in_valid_methods(self, checkout_info) -> bool:
    valid_delivery_methods = checkout_info.valid_pick_up_points
    return bool(
        valid_delivery_methods and self.delivery_method in valid_delivery_methods
    )
```

### 10.5 失效原因到错误码映射汇总

| 场景 | 触发条件 | 错误码 | 错误消息 |
|------|----------|--------|----------|
| **选配送时** | 配送方法在 excluded_methods 列表中 → active=False → 遍历跳过 → 未找到 | `SHIPPING_METHOD_NOT_APPLICABLE` | "This shipping method is not applicable." |
| **下单时** | 已选择的配送方法 active=False → is_method_in_valid_methods 返回 False | `INVALID_SHIPPING_METHOD` | "Delivery method is not valid for your shipping address" |
| **下单时** | 未选择任何配送方法 | `SHIPPING_METHOD_NOT_SET` | "Shipping method is not set" |
| **下单时** | 未设置配送地址 | `SHIPPING_ADDRESS_NOT_SET` | "Shipping address is not set" |

### 10.6 关键设计说明

#### 10.6.1 message 字段的作用

`ShippingMethodData.message` 存储了具体失效原因（来自 Webhook 的 reason），但**不会直接传递到错误响应中**，主要用于：
- 内部日志记录
- 前端展示可用配送方法列表时，标记为灰色并显示 tooltip 提示
- GraphQL 查询 `shippingMethods` 时返回给客户端展示

#### 10.6.2 两次校验的差异

| 维度 | 选配送时校验 | 下单时校验 |
|------|------------|-----------|
| 校验点 | `get_checkout_delivery()` | `clean_checkout_shipping()` |
| 检查目标 | 检查选择的方法是否在有效列表中 | 检查已选择的方法当前是否仍然有效 |
| 错误码 | `SHIPPING_METHOD_NOT_APPLICABLE` | `INVALID_SHIPPING_METHOD` |
| 触发时机 | 用户主动选择配送方法时 | 提交订单前的最终检查 |
| 防护目的 | 防止用户选择已失效的方法 | 防止地址/商品变化后原方法不再适用 |

#### 10.6.3 可能的失效场景

1. **邮编不匹配**：地址邮编不在配送方法的 INCLUDE 范围内或在 EXCLUDE 范围内
2. **价格超限**：订单小计超过 maximum_order_price 或低于 minimum_order_price
3. **重量超限**：订单重量超过 maximum_order_weight 或低于 minimum_order_weight
4. **产品排除**：订单包含配送方法排除的产品
5. **Webhook 排除**：外部 Webhook 返回该配送方法不可用（携带自定义 reason）
