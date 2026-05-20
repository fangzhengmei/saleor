# 礼品卡完整链路代码分析

## 概述

礼品卡（Gift Card）在 Saleor 系统中经历四个核心阶段的接力：**卡片生成 → 库存绑定 → 订单抵扣 → 余额冻结**。每个阶段都有明确的代码边界和数据流转逻辑。

---

## 第一阶段：卡片生成（签发）

### 1.1 数据模型

**文件**: `saleor/giftcard/models.py:45-151`

核心字段：
```python
class GiftCard(ModelWithMetadata):
    code = models.CharField(max_length=16, unique=True)  # 礼品卡兑换码，8-16位
    is_active = models.BooleanField(default=True)        # 是否激活
    initial_balance_amount = models.DecimalField(...)    # 初始余额
    current_balance_amount = models.DecimalField(...)    # 当前余额
    currency = models.CharField(...)                     # 币种
    expiry_date = models.DateField(null=True)            # 过期日期
    product = models.ForeignKey("product.Product", ...)  # 关联的产品（用于库存绑定）
    created_by = models.ForeignKey(User, ...)            # 创建人
    used_by = models.ForeignKey(User, ...)               # 使用人
    last_used_on = models.DateTimeField(null=True)       # 最后使用时间
```

### 1.2 创建逻辑

**文件**: `saleor/graphql/giftcard/mutations/gift_card_create.py`

#### 核心流程：
1. **clean_input** (第124-170行):
   - 验证礼品卡代码唯一性（`is_available_promo_code`）
   - 自动生成兑换码（`generate_promo_code`），8-16位随机字符串
   - 验证余额必须 > 0
   - 验证过期日期必须在未来

2. **clean_balance** (第194-225行):
   - 初始余额 = 当前余额，创建时两者相同
   - 币种一旦设置不可更改

3. **post_save_action** (第228-252行):
   - 触发 `gift_card_issued_event` 事件（类型：`GiftCardEvents.ISSUED`）
   - 记录创建人和创建应用
   - 可选发送邮件通知客户
   - 触发 webhook: `GIFT_CARD_CREATED`

### 1.3 事件记录

**文件**: `saleor/giftcard/events.py`

礼品卡所有状态变更都通过 `GiftCardEvent` 表记录：
- `ISSUED` - 签发
- `BOUGHT` - 客户购买
- `USED_IN_ORDER` - 订单中使用
- `REFUNDED_IN_ORDER` - 订单中退款
- `ACTIVATED/DEACTIVATED` - 激活/停用
- `BALANCE_RESET` - 余额重置

---

## 第二阶段：库存绑定

### 2.1 产品类型关联

**文件**: `saleor/product/__init__.py:11-18`

```python
class ProductTypeKind:
    NORMAL = "normal"
    GIFT_CARD = "gift_card"  # 礼品卡产品类型
```

**文件**: `saleor/product/models.py:442-443`

```python
@property
def is_gift_card(self) -> bool:
    return self.product.product_type.kind == ProductTypeKind.GIFT_CARD
```

### 2.2 礼品卡作为商品销售

礼品卡可以像普通商品一样被添加到购物车中进行购买：

1. **产品配置**: 创建一个 `ProductTypeKind.GIFT_CARD` 类型的产品
2. **订单行标记**: 订单行通过 `is_gift_card` 属性标识该行是礼品卡购买
3. **自动发货**: 礼品卡商品通常设置为无需物流（non-shippable）

**文件**: `saleor/order/actions.py:401-405`

```python
if order_info.channel.automatically_fulfill_non_shippable_gift_card:
    order_lines = [line.line for line in order_info.lines_data]
    fulfill_non_shippable_gift_cards(
        order, order_lines, site_settings, user, app, manager
    )
```

### 2.3 与产品库存的关系

礼品卡商品**完全在库存体系内**，与普通商品一样受库存管控：

1. **库存分配 (Allocate)**: 结账完成时调用 `allocate_stocks()` 为礼品卡商品分配库存
2. **库存扣减 (Decrease)**: 履约时调用 `decrease_stock()` 实际扣减库存
3. **库存检查失败**: 如果库存不足，会在履约创建阶段抛出 `InsufficientStock` 异常

礼品卡通过 `GiftCard.product` 外键关联到产品，这意味着：
- 可以通过购买"礼品卡商品"来获得礼品卡
- 购买后系统自动生成礼品卡实例并发放给客户
- 礼品卡商品的库存限制了可销售的礼品卡数量

---

## 第三阶段：订单抵扣

礼品卡的抵扣有两条平行路径：**传统路径**（直接关联 Checkout）和**新支付网关路径**（通过 Transaction 机制）。

### 3.1 传统路径：Checkout 直接关联

#### 3.1.1 结账完成前

**文件**: `saleor/checkout/complete_checkout.py:789,854` 和 `:1388,1530`

在 `create_order` / `create_order_from_checkout` 函数中调用：

```python
add_gift_cards_to_order(checkout_info, order, total_price_left, user, app)
```

#### 3.1.2 核心抵扣逻辑

**文件**: `saleor/order/utils.py:443-526`

```python
def add_gift_cards_to_order(
    checkout_info: "CheckoutInfo",
    order: Order,
    total_price_left: Money,
    user: User | None,
    app: Optional["App"],
):
    # 1. 遍历 checkout 关联的所有礼品卡，加行锁防止并发
    for gift_card in checkout_info.checkout.gift_cards.select_for_update():
        if total_price_left > zero_money(...):
            # 2. 更新礼品卡余额
            total_price_left = update_gift_card_balance(
                gift_card, total_price_left, balance_data
            )
            # 3. 设置使用人信息
            set_gift_card_user(gift_card, used_by_user, used_by_email)
            gift_card.last_used_on = timezone.now()
    
    # 4. 关联礼品卡到订单
    order.gift_cards.add(*order_gift_cards)
    
    # 5. 批量更新余额
    GiftCard.objects.bulk_update(gift_cards_to_update, update_fields)
    
    # 6. 记录 USED_IN_ORDER 事件
    gift_card_events.gift_cards_used_in_order_event(balance_data, order, user, app)
    
    # 7. 失效其他 checkout 的价格缓存（防止同一礼品卡被多处使用）
    Checkout.objects.filter(gift_cards__in=gift_cards_to_update).exclude(
        token=checkout_info.checkout.token
    ).update(price_expiration=timezone.now())
```

#### 3.1.3 余额计算

**文件**: `saleor/order/utils.py:498-511`

```python
def update_gift_card_balance(gift_card, total_price_left, balance_data):
    previous_balance = gift_card.current_balance
    if total_price_left < gift_card.current_balance:
        # 礼品卡余额充足，抵扣部分金额
        gift_card.current_balance = gift_card.current_balance - total_price_left
        total_price_left = zero_money(...)
    else:
        # 礼品卡余额不足，全部用完
        total_price_left = total_price_left - gift_card.current_balance
        gift_card.current_balance_amount = 0
    return total_price_left
```

### 3.2 新支付网关路径：Transaction 机制

#### 3.2.1 初始化支付会话

**文件**: `saleor/payment/utils.py:1917-1925`

```python
if payment_gateway_data.app_identifier == GIFT_CARD_PAYMENT_GATEWAY_ID:
    from ..giftcard.gateway import (
        transaction_initialize_session_with_gift_card_payment_method,
    )
    with transaction.atomic():
        result = transaction_initialize_session_with_gift_card_payment_method(
            session_data, source_object
        )
```

#### 3.2.2 礼品卡支付网关处理

**文件**: `saleor/giftcard/gateway.py:55-95`

```python
def transaction_initialize_session_with_gift_card_payment_method(
    transaction_session_data: "TransactionSessionData",
    source_object: Checkout | Order,
) -> "TransactionSessionResult":
    
    attach_app_identifier_to_transaction(transaction_session_data)
    
    try:
        # 1. 验证会话数据和礼品卡有效性
        validate_transaction_session_data(...)
        gift_card = validate_and_get_gift_card(...)  # 这里会加 select_for_update 锁
    except GiftCardPaymentGatewayException as exc:
        return TransactionSessionResult(
            result=AUTHORIZATION_FAILURE,
            message=str(exc)
        )
    else:
        return TransactionSessionResult(
            result=AUTHORIZATION_SUCCESS,
            message=f"Gift card (ending: {gift_card.display_code})."
        )
    finally:
        # 2. 关键：将礼品卡从之前的 checkout 分离（防止重复使用）
        detach_gift_card_from_previous_checkout_transactions(gift_card)
        # 3. 将礼品卡绑定到当前 transaction
        attach_gift_card_to_transaction(transaction_session_data, gift_card)
```

#### 3.2.3 礼品卡有效性校验与锁定

**文件**: `saleor/giftcard/gateway.py:126-151`

```python
def validate_and_get_gift_card(transaction_session_data):
    gift_card = (
        GiftCard.objects.active(date=timezone.now().date())
        .filter(
            code=transaction_session_data.payment_gateway_data.data["code"],
            currency=transaction_session_data.action.currency,
        )
        .select_for_update()  # 行级锁，防止并发扣款
        .get()
    )
    
    # 检查余额是否足够
    if transaction_session_data.action.amount > gift_card.current_balance_amount:
        raise GiftCardPaymentGatewayException("Insufficient amount")
    
    return gift_card
```

#### 3.2.4 防止重复授权

**文件**: `saleor/giftcard/gateway.py:179-224`

```python
def detach_gift_card_from_previous_checkout_transactions(gift_card):
    # 查找该礼品卡关联的其他未完成 checkout 交易
    transactions_to_cancel_qs = TransactionItem.objects.filter(
        Q(app_identifier=GIFT_CARD_PAYMENT_GATEWAY_ID),
        Q(gift_card=gift_card),
        Q(checkout_id__isnull=False),  # 关联到 checkout
        Q(order_id__isnull=True),      # 尚未关联到订单
    )
    
    for transaction_item in transactions_to_cancel_qs:
        # 创建 CANCEL 事件，取消之前的授权
        create_transaction_event_requested(
            transaction_item,
            transaction_item.amount_authorized.amount,
            TransactionAction.CANCEL,
            ...
        )
    
    # 移除礼品卡关联
    transactions_to_cancel_qs.update(gift_card=None)
```

---

## 第四阶段：余额冻结与扣款

### 4.1 订单确认时扣款

**文件**: `saleor/order/actions.py:377-379`

```python
def confirm_order(...):
    # ... 订单确认逻辑 ...
    
    from ..giftcard.gateway import charge_gift_card_transactions
    charge_gift_card_transactions(order, user=user, app=app)
```

### 4.2 核心扣款逻辑

**文件**: `saleor/giftcard/gateway.py:249-345`

```python
def charge_gift_card_transactions(order, user=None, app=None):
    # 找出该订单中所有礼品卡支付交易（尚未扣款的）
    gift_card_transactions = order.payment_transactions.filter(
        ~Exists(  # 确保不重复扣款
            TransactionEvent.objects.filter(
                transaction=OuterRef("pk"), type=TransactionEventType.CHARGE_REQUEST
            )
        ),
        app_identifier=GIFT_CARD_PAYMENT_GATEWAY_ID,
        gift_card__isnull=False,
        authorized_value__gt=Decimal(0),
        charged_value=Decimal(0),
    )
    
    for gift_card_transaction in gift_card_transactions:
        # 创建 CHARGE 请求事件
        request_event = create_transaction_event_requested(...)
        
        try:
            with transaction.atomic():
                # 再次锁定礼品卡行
                gift_card = (
                    GiftCard.objects.filter(id=gift_card_transaction.gift_card_id)
                    .select_for_update()
                    .get()
                )
                
                if gift_card_transaction.authorized_value > gift_card.current_balance_amount:
                    response["message"] = "Insufficient amount"
                else:
                    # 执行扣款
                    previous_balance = gift_card.current_balance_amount
                    charge_gift_card(
                        gift_card,
                        gift_card_transaction.authorized_value,
                        order,
                    )
                    
                    # 记录使用事件
                    gift_cards_used_in_order_event(
                        balance_data=[(gift_card, previous_balance)],
                        order=order, user=user, app=app
                    )
                    
                    response["result"] = CHARGE_SUCCESS
        except GiftCard.DoesNotExist:
            pass
        
        create_transaction_event_from_request_and_webhook_response(...)
```

### 4.3 扣款执行

**文件**: `saleor/giftcard/gateway.py:227-246`

```python
def charge_gift_card(gift_card, authorized_value, order):
    # 1. 扣除余额
    gift_card.current_balance_amount -= authorized_value
    # 2. 设置使用人
    gift_card.used_by = order.user or User.objects.filter(email=order.user_email).first()
    gift_card.used_by_email = order.user_email
    gift_card.last_used_on = timezone.now()
    gift_card.save(update_fields=[
        "current_balance_amount", "used_by", "used_by_email", "last_used_on"
    ])
```

---

## 接力关系总结

```
卡片生成
    ↓
[GiftCardCreate mutation] → GiftCard 实例 + ISSUED 事件
    ↓
库存绑定（可选）
    ↓
[ProductTypeKind.GIFT_CARD] → 礼品卡可作为商品销售
    ↓
订单抵扣（双路径）
    ├─ 传统路径：Checkout.gift_cards → add_gift_cards_to_order() → 直接扣余额
    ↓                           
新支付网关路径
    ↓
[transactionInitialize] → 礼品卡校验 → 绑定 Transaction → 隐式"冻结"（通过事务锁）
    ↓
[orderConfirm] → charge_gift_card_transactions() → 正式扣余额
    ↓
余额更新
    ↓
GiftCard.current_balance_amount 更新 + USED_IN_ORDER 事件
```

### 关键并发控制机制

1. **行级锁 (`select_for_update`)**:
   - 位置：`gateway.py:137`, `gateway.py:303`, `order/utils.py:456`
   - 作用：防止同一礼品卡被并发扣款

2. **事务原子性**:
   - 使用 `transaction.atomic()` 包裹扣款逻辑
   - 确保校验和扣款在同一事务中

3. **单 checkout 绑定**:
   - `detach_gift_card_from_previous_checkout_transactions()` 确保礼品卡同一时间只能用于一个 checkout

### 余额"冻结"的实现方式

Saleor 没有单独的"冻结余额"字段，而是通过以下机制实现：

1. **事务锁**: `select_for_update()` 在验证时锁定行，其他并发请求等待
2. **单 checkout 关联**: 礼品卡通过 `TransactionItem.gift_card` 关联到特定交易，自动与其他 checkout 解绑
3. **即时扣款**: 在订单确认时直接扣除余额，而非先冻结后扣款

---

## 退款流程（补充）

**文件**: `saleor/giftcard/gateway.py:384-448`

```python
def refund_gift_card_transaction(transaction_item, request_event, user=None, app=None):
    try:
        with transaction.atomic():
            gift_card = (
                GiftCard.objects.filter(id=transaction_item.gift_card_id)
                .select_for_update()
                .get()
            )
            previous_balance = gift_card.current_balance_amount
            # 使用 F() 表达式原子增加余额
            gift_card.current_balance_amount = F("current_balance_amount") + amount
            gift_card.save(update_fields=["current_balance_amount"])
            
            gift_card_refunded_in_order_event(
                gift_card=gift_card,
                order=transaction_item.order,
                previous_balance=previous_balance,
                current_balance=previous_balance + amount,
                ...
            )
    except GiftCard.DoesNotExist:
        pass
```

退款时使用 `F()` 表达式确保原子更新，符合 Saleor 并发安全规范。

---

## 补充细节一：礼品卡商品从订单行到签发张数的调用链

### 完整调用链

当客户购买"礼品卡商品"时，系统需要根据订单行的数量生成对应张数的礼品卡。完整调用链如下：

```
[Checkout Complete] 结账完成
    ↓
create_order() / create_order_from_checkout()
    ↓
Order.objects.create() 创建订单
    ↓
allocate_stocks() 分配库存（包括礼品卡商品的库存）
    ↓
add_gift_cards_to_order() 处理礼品卡抵扣（如果使用了礼品卡支付）
    ↓
order.save() 保存订单
    ↓
transaction.on_commit → order_created() 订单创建后事件
    ↓
检查订单支付状态
    ↓
如果已全额支付 → order_charged() → handle_fully_paid_order()
    ↓
[关键触发点] handle_fully_paid_order()
    ├─ 如果 channel.automatically_fulfill_non_shippable_gift_card = True
    └─ 调用 fulfill_non_shippable_gift_cards()
        ↓
        get_non_shippable_gift_card_lines() 筛选无需物流的礼品卡订单行
        ↓
        fulfill_gift_card_lines() 履约礼品卡行
            ↓
            create_fulfillments() 创建履约单
                ↓
                _create_fulfillment_lines() 创建履约行
                    ├─ 检查库存充足性
                    ├─ decrease_stock() 扣减库存
                    └─ 收集 gift_card_lines_info（包含 quantity）
                ↓
                order_fulfilled() 履约完成
                    ↓
                    [核心签发] gift_cards_create(order, gift_card_lines_info, ...)
                        ├─ 遍历每个 gift_card_lines_info
                        ├─ 按 line_data.quantity 循环生成 GiftCard 实例
                        ├─ GiftCard.objects.bulk_create() 批量创建
                        ├─ BOUGHT 事件记录
                        └─ 发送礼品卡邮件给客户
```

### 关键代码位置

**1. 触发点：订单全额支付后**

**文件**: `saleor/order/actions.py:382-405`

```python
def handle_fully_paid_order(...):
    # ...
    if order_info.channel.automatically_fulfill_non_shippable_gift_card:
        order_lines = [line.line for line in order_info.lines_data]
        fulfill_non_shippable_gift_cards(
            order, order_lines, site_settings, user, app, manager
        )
```

**2. 筛选礼品卡订单行**

**文件**: `saleor/giftcard/utils.py:103-113`

```python
def get_non_shippable_gift_card_lines(lines: Iterable[OrderLine]) -> "QuerySet":
    gift_card_lines = get_gift_card_lines(lines)  # is_gift_card = True
    non_shippable_lines = OrderLine.objects.filter(
        id__in=[line.pk for line in gift_card_lines], 
        is_shipping_required=False  # 无需物流
    )
    return non_shippable_lines
```

**3. 履约创建与库存扣减顺序**

**文件**: `saleor/order/actions.py:1094-1219` (`_create_fulfillment_lines`)

```python
def _create_fulfillment_lines(...):
    # 1. 检查库存
    for line in lines_data:
        # ... 检查 stock 是否充足 ...
        if stock is None and not allow_stock_to_be_exceeded:
            raise InsufficientStock(...)
    
    # 2. 扣减库存（在履约创建时执行）
    if lines_info and should_decrease_stock:
        decrease_stock(lines_info, ...)
    
    # 3. 收集礼品卡行信息（包含 quantity）
    for line in lines_data:
        if order_line.is_gift_card:
            gift_card_lines_info.append(
                GiftCardLineData(
                    quantity=fulfillment_line.quantity,  # 本次履约数量 → 本次签发张数
                    order_line=order_line,
                    variant=variant,
                    fulfillment_line=fulfillment_line,
                )
            )
```

**4. 核心签发逻辑**

**文件**: `saleor/giftcard/utils.py:162-216`

```python
@traced_atomic_transaction()
def gift_cards_create(order, gift_card_lines_info, settings, requestor_user, app, manager):
    expiry_date = calculate_expiry_date(settings)
    gift_cards = []
    for line_data in gift_card_lines_info:
        order_line = line_data.order_line
        price = order_line.unit_price_gross  # 礼品卡面值 = 订单行单价
        # 关键：按 quantity 生成对应张数的礼品卡
        line_gift_cards = [
            GiftCard(
                code=generate_promo_code(),
                initial_balance=price,
                current_balance=price,
                created_by=customer_user,
                created_by_email=user_email,
                product=line_data.variant.product if line_data.variant else None,
                fulfillment_line=line_data.fulfillment_line,
                expiry_date=expiry_date,
            )
            for _ in range(line_data.quantity)  # 本次履约数量 → 本次签发张数
        ]
        gift_cards.extend(line_gift_cards)
    
    # 批量创建礼品卡
    gift_cards = GiftCard.objects.bulk_create(gift_cards)
    events.gift_cards_bought_event(gift_cards, order, requestor_user, app)
    # ... 发送邮件 ...
    return gift_cards
```

### 库存扣减、履约、签发的先后关系

| 步骤 | 操作 | 代码位置 | 说明 |
|------|------|---------|------|
| 1 | **库存分配 (Allocate)** | `complete_checkout.py:835` | 结账完成时，调用 `allocate_stocks()` 锁定库存 |
| 2 | **订单创建** | `complete_checkout.py:807` | 创建 Order 和 OrderLine |
| 3 | **订单支付确认** | `order/actions.py:334-345` | 检查 `charge_status`，如果已支付触发 `handle_fully_paid_order()` |
| 4 | **自动履约** | `order/actions.py:401-405` | 调用 `fulfill_non_shippable_gift_cards()` |
| 5 | **库存扣减 (Decrease)** | `order/actions.py:1210-1215` | `decrease_stock()` 实际扣减库存 |
| 6 | **礼品卡签发** | `giftcard/utils.py:162-216` | `gift_cards_create()` 生成礼品卡实例 |

> **重要**: 礼品卡的签发发生在**库存扣减之后**。这意味着：
> 1. 如果库存不足，会在 `_create_fulfillment_lines` 阶段抛出 `InsufficientStock` 异常
> 2. 只有库存扣减成功后，才会生成礼品卡
> 3. **履约行的 `quantity` 字段**决定了本次生成多少张礼品卡（支持分批履约分批签发）

### 库存不足的失败场景

礼品卡商品的库存检查发生在两个关键节点：

**节点 1：结账完成时的库存分配**

**文件**: `saleor/warehouse/management.py:91-190`

```python
def allocate_stocks(...):
    # 对所有需要追踪库存的订单行（包括礼品卡商品）执行库存分配
    order_lines_info = get_order_lines_with_track_inventory(order_lines_info)
    
    # 加行锁查询可用库存
    stocks = list(
        stock_select_for_update_for_existing_qs(stocks)
        .filter(**filter_lookup)
        .values("id", "product_variant", "pk", "quantity", "warehouse_id")
    )
    
    # 计算已分配数量
    quantity_allocation_for_stocks = ...
    
    # 创建分配记录
    insufficient_stock, allocation_items = _create_allocations(
        line_info, stock_allocations, ...
    )
    
    if insufficient_stock:
        raise InsufficientStock(insufficient_stock)  # 库存不足，结账失败
```

**节点 2：履约创建时的库存扣减**

**文件**: `saleor/warehouse/management.py:586-641`

```python
def decrease_stock(...):
    # 先释放分配
    decrease_allocations(order_lines_info, site_settings, requestor)
    
    # 加行锁查询库存
    stocks = (
        stock_qs_select_for_update()
        .filter(product_variant__in=variants)
        .filter(warehouse_id__in=warehouse_pks)
        .select_related("product_variant", "warehouse")
    )
    
    # 检查并扣减库存
    _decrease_stocks_quantity(
        order_lines_info, variant_and_warehouse_to_stock, ...
    )
```

**文件**: `saleor/order/actions.py:888-920`

```python
if stock is None:
    warehouse_pk = None
    if not allow_stock_to_be_exceeded:
        error_data = InsufficientStockData(
            variant=variant,
            order_line=order_line,
            warehouse_pk=warehouse_pk,
            available_quantity=0,
        )
        insufficient_stocks.append(error_data)

if insufficient_stocks:
    raise InsufficientStock(insufficient_stocks)  # 库存不足，履约失败
```

### 分批履约、分批签发

礼品卡签发是按**履约行数量**而非订单行数量执行的，支持分批履约：

**示例场景**：订单行购买了 5 张礼品卡，分两次履约

| 履约批次 | 履约行数量 | 操作 | 结果 |
|---------|-----------|------|------|
| 第一次履约 | 3 | `gift_cards_create(quantity=3)` | 生成 3 张礼品卡，扣减 3 件库存 |
| 第二次履约 | 2 | `gift_cards_create(quantity=2)` | 生成 2 张礼品卡，扣减 2 件库存 |
| **累计** | **5** | | **共生成 5 张礼品卡** |

**关键代码证据**：

**文件**: `saleor/order/actions.py:882-917`

```python
# 履约时遍历本次履约的所有履约行
for fulfillment_line in fulfillment_lines:
    order_line = fulfillment_line.order_line
    variant = fulfillment_line.order_line.variant
    stock = fulfillment_line.stock
    
    # ... 库存检查 ...
    
    if order_line.is_gift_card:
        gift_card_lines_info.append(
            GiftCardLineData(
                quantity=fulfillment_line.quantity,  # 本次履约的数量
                order_line=order_line,
                variant=variant,
                fulfillment_line=fulfillment_line,
            )
        )
```

**文件**: `saleor/giftcard/utils.py:177-192`

```python
for line_data in gift_card_lines_info:
    order_line = line_data.order_line
    price = order_line.unit_price_gross
    line_gift_cards = [
        GiftCard(
            code=generate_promo_code(),
            initial_balance=price,
            current_balance=price,
            fulfillment_line=line_data.fulfillment_line,  # 关联到具体履约行
            ...
        )
        for _ in range(line_data.quantity)  # 按本次履约数量生成
    ]
```

**测试用例验证**：`saleor/giftcard/tests/test_utils.py:372-400`

```python
def test_gift_cards_create_multiple_quantity(...):
    # given
    quantity = 3
    gift_card_non_shippable_order_line.quantity = quantity
    fulfillment_line = fulfillment.lines.create(
        order_line=gift_card_non_shippable_order_line, 
        quantity=quantity,  # 履约行数量 = 3
        stock=stock
    )
    lines_data = [
        GiftCardLineData(
            quantity=quantity,  # 传入履约行数量
            order_line=gift_card_non_shippable_order_line,
            variant=gift_card_non_shippable_order_line.variant,
            fulfillment_line=fulfillment_line,
        )
    ]
    
    # when
    gift_cards = gift_cards_create(order, lines_data, ...)
    
    # then
    assert len(gift_cards) == quantity  # 生成 3 张礼品卡
```

---

## 补充细节二：新支付网关余额冻结的完整时序

### 时序图

```
[transactionInitialize] 初始化支付会话
    ↓
payment/utils.py: handle_transaction_initialize_session()
    ↓
识别到 GIFT_CARD_PAYMENT_GATEWAY_ID
    ↓
giftcard/gateway.py: transaction_initialize_session_with_gift_card_payment_method()
    ├─ try 块:
    │   ├─ validate_transaction_session_data() 验证数据格式
    │   └─ validate_and_get_gift_card()
    │       ├─ GiftCard.objects.active().filter(code=...).select_for_update() 加行锁
    │       └─ 检查余额 >= 请求金额
    │
    ├─ finally 块 (无论成功失败都执行):
    │   ├─ detach_gift_card_from_previous_checkout_transactions()
    │   │   ├─ 查找该礼品卡关联的其他未完成 checkout 交易
    │   │   ├─ 对每个旧交易创建 CANCEL 事件
    │   │   └─ 清除旧交易的 gift_card 关联
    │   └─ attach_gift_card_to_transaction()
    │       ├─ 设置 transaction.gift_card = gift_card
    │       ├─ 设置 payment_method_type = GIFT_CARD
    │       └─ 设置 gift_card_last_chars, gift_card_brand
    │
    └─ 返回 AUTHORIZATION_SUCCESS / AUTHORIZATION_FAILURE
    ↓
[交易已授权] 礼品卡通过 Transaction 与当前 Checkout 绑定
    ↓
[可选操作 1: 取消授权]
    ↓
graphql/payment/mutations/transaction/transaction_request_action.py
    ↓
action = TransactionAction.CANCEL
    ↓
cancel_gift_card_transaction()
    ├─ 检查 checkout 是否存在、CANCEL 是否在 available_actions 中
    ├─ 创建 CANCEL_SUCCESS 事件
    └─ 注意：不修改礼品卡余额（因为从未扣款）
    ↓
[授权已取消] 礼品卡可被其他 checkout 使用

[可选操作 2: 订单确认 → 扣款]
    ↓
order/actions.py: order_confirmed()
    ↓
charge_gift_card_transactions()
    ├─ 筛选未扣款的礼品卡交易
    ├─ 再次 select_for_update() 锁定礼品卡
    ├─ charge_gift_card() 扣减余额
    └─ 记录 USED_IN_ORDER 事件
    ↓
[扣款完成] 余额已实际扣除

[可选操作 3: 退款]
    ↓
action = TransactionAction.REFUND
    ↓
refund_gift_card_transaction()
    ├─ select_for_update() 锁定礼品卡
    ├─ F("current_balance_amount") + amount 原子增加余额
    └─ 记录 REFUNDED_IN_ORDER 事件
    ↓
[退款完成] 余额已恢复
```

### 避免重复占用的三重机制

**机制 1：行级锁 (select_for_update)**

**文件**: `saleor/giftcard/gateway.py:137, 303`

```python
# 授权时锁定
gift_card = (
    GiftCard.objects.active(date=timezone.now().date())
    .filter(code=code, currency=currency)
    .select_for_update()  # 行级锁，其他请求等待
    .get()
)

# 扣款时再次锁定
gift_card = (
    GiftCard.objects.filter(id=gift_card_transaction.gift_card_id)
    .select_for_update()
    .get()
)
```

**机制 2：自动分离旧交易 (detach_gift_card_from_previous_checkout_transactions)**

**文件**: `saleor/giftcard/gateway.py:179-224`

```python
def detach_gift_card_from_previous_checkout_transactions(gift_card):
    # 找出同一礼品卡关联的其他 checkout 交易
    transactions_to_cancel_qs = TransactionItem.objects.filter(
        Q(app_identifier=GIFT_CARD_PAYMENT_GATEWAY_ID),
        Q(gift_card=gift_card),
        Q(checkout_id__isnull=False),  # 关联到 checkout
        Q(order_id__isnull=True),      # 尚未生成订单
    )
    
    for transaction_item in transactions_to_cancel_qs:
        # 创建 CANCEL 事件，通知旧 checkout 授权已失效
        create_transaction_event_requested(
            transaction_item,
            transaction_item.amount_authorized.amount,
            TransactionAction.CANCEL,
            ...
        )
    
    # 移除旧交易的礼品卡关联
    transactions_to_cancel_qs.update(gift_card=None)
```

**机制 3：不重复扣款校验**

**文件**: `saleor/giftcard/gateway.py:306-316`

```python
gift_card_transactions = order.payment_transactions.filter(
    ~Exists(  # 确保没有 CHARGE_REQUEST 事件，即未扣款过
        TransactionEvent.objects.filter(
            transaction=OuterRef("pk"), 
            type=TransactionEventType.CHARGE_REQUEST
        )
    ),
    app_identifier=GIFT_CARD_PAYMENT_GATEWAY_ID,
    gift_card__isnull=False,
    authorized_value__gt=Decimal(0),
    charged_value=Decimal(0),  # 已扣款金额为 0
)
```

### 取消授权 vs 退款的区别

| 操作 | 触发时机 | 余额变化 | 事件类型 |
|------|---------|---------|---------|
| **CANCEL (取消授权)** | 订单确认前，用户取消支付或礼品卡被其他 checkout 占用 | 余额不变（从未扣款） | `CANCEL_SUCCESS` |
| **REFUND (退款)** | 订单确认并扣款后，需要退回资金 | 余额增加（使用 `F()` 表达式） | `REFUNDED_IN_ORDER` |

**取消授权代码**: `saleor/giftcard/gateway.py:348-381`

```python
def cancel_gift_card_transaction(transaction_item, request_event):
    # 不修改礼品卡余额，只记录事件
    response = {
        "result": TransactionEventType.CANCEL_SUCCESS.upper(),
        "pspReference": str(uuid4()),
        "amount": amount,
    }
    create_transaction_event_from_request_and_webhook_response(...)
```

**退款代码**: `saleor/giftcard/gateway.py:384-448`

```python
def refund_gift_card_transaction(transaction_item, request_event, user=None, app=None):
    with transaction.atomic():
        gift_card = (
            GiftCard.objects.filter(id=transaction_item.gift_card_id)
            .select_for_update()
            .get()
        )
        # 使用 F() 表达式原子增加余额
        gift_card.current_balance_amount = F("current_balance_amount") + amount
        gift_card.save(update_fields=["current_balance_amount"])
        # 记录退款事件
        gift_card_refunded_in_order_event(...)
```

### 余额"冻结"的本质

Saleor 没有显式的"冻结余额"字段，所谓的"冻结"是通过以下方式隐式实现的：

1. **绑定关系锁定**: 礼品卡通过 `TransactionItem.gift_card` 外键与特定交易绑定
2. **自动分离机制**: 新授权自动分离旧授权，确保同一时间只有一个 checkout 能使用
3. **事务行锁**: `select_for_update()` 在关键操作时锁定行，防止并发修改
4. **余额校验**: 每次操作前重新校验余额，确保不会超扣

这种设计避免了维护单独的"冻结余额"字段带来的一致性问题，但也意味着：
- 如果 checkout 长期不完成，礼品卡余额不会被真正扣除
- 如果用户在多个浏览器标签页尝试使用同一张礼品卡，后一个会自动取消前一个的授权

---

## 补充细节三：Checkout 授权转 Order 交易的占用关系变化

### 交易关联迁移

当 checkout 完成并生成订单时，所有关联的支付交易（包括礼品卡支付）会从 checkout 迁移到 order：

**文件**: `saleor/checkout/complete_checkout.py:1534`

```python
# 将 checkout 的支付交易迁移到 order
checkout_info.checkout.payment_transactions.update(order=order, checkout_id=None)
```

**迁移前**：
- `TransactionItem.checkout_id = checkout.token`
- `TransactionItem.order_id = None`

**迁移后**：
- `TransactionItem.checkout_id = None`
- `TransactionItem.order_id = order.id`

### 占用关系的变化

| 阶段 | 关联关系 | 占用状态 |
|------|---------|---------|
| **授权阶段** | `TransactionItem.checkout_id = X`, `gift_card = Y` | 礼品卡 Y 被 checkout X 占用 |
| **订单创建后** | `TransactionItem.order_id = Z`, `checkout_id = None` | 礼品卡 Y 被订单 Z 占用 |

### Detach 逻辑的生效边界

`detach_gift_card_from_previous_checkout_transactions()` 函数只会分离**尚未关联到订单**的交易：

**文件**: `saleor/giftcard/gateway.py:190-200`

```python
transactions_to_cancel_qs = TransactionItem.objects.filter(
    Q(app_identifier=GIFT_CARD_PAYMENT_GATEWAY_ID),
    Q(gift_card=gift_card),
    Q(checkout_id__isnull=False),  # 仍关联到 checkout
    Q(order_id__isnull=True),      # 尚未关联到订单
)
```

**这意味着**：

1. **交易已关联到订单后**：`order_id__isnull=True` 条件不满足，detach 逻辑**不会分离已关联订单的交易，但余额校验仍然生效
2. **订单确认扣款前**：礼品卡仍与交易绑定，新的授权请求无法通过 detach 分离，但会在 `validate_and_get_gift_card()` 中进行余额校验
3. **订单确认扣款后**：礼品卡余额已扣除，占用关系结束

### Detach 过滤条件 vs 余额校验的分工

| 机制 | 约束内容 | 代码位置 | 作用域 |
|------|---------|---------|-------|
| **Detach 过滤条件** | `checkout_id__isnull=False` AND `order_id__isnull=True` | `gateway.py:196-199` | 仅分离 checkout 阶段的授权，不影响已生成订单的交易 |
| **余额校验** | `action.amount <= gift_card.current_balance_amount` | `gateway.py:145-149` | 所有授权请求都要检查，无论交易是否关联订单 |

**核心差异**：
- **Detach 是**抢占式分离**：把礼品卡从旧 checkout 抢过来给新 checkout 使用，但只对 checkout 阶段的交易有效
- **余额校验是**最终防线**：即使 detach 没分离成功（比如已关联订单），余额校验仍会拦截超扣请求

### 完整的占用生命周期

```
[礼品卡授权]
    ↓
TransactionItem.gift_card = gift_card
TransactionItem.checkout_id = checkout.token
TransactionItem.order_id = None
    ↓
[Checkout 阶段：其他 checkout 授权会触发 detach，抢占礼品卡]
    ↓
[Checkout 完成 → 生成订单]
    ↓
TransactionItem.order_id = order.id
TransactionItem.checkout_id = None  ← 关键变化
    ↓
[订单阶段：detach 不再分离此交易，但余额校验仍拦截超扣]
    ↓
[订单确认 → 扣款]
    ↓
charge_gift_card_transactions()
    ↓
GiftCard.current_balance_amount -= authorized_value
    ↓
[占用结束，礼品卡余额已实际扣除]
```

### 关键边界条件与统一结论

#### 授权请求的拦截条件

当礼品卡已关联到某个订单但尚未扣款时，新的授权请求是否被拦截取决于**两个独立检查**：

| 检查点 | 触发条件 | 结果 | 代码位置 |
|--------|---------|------|---------|
| **1. Detach 检查** | 旧交易满足 `checkout_id__isnull=False` AND `order_id__isnull=True` | ✅ 旧交易被取消，新授权成功 | `gateway.py:196-199` |
| **2. 余额校验** | `请求金额 > gift_card.current_balance_amount` | ❌ 授权失败，余额不足 | `gateway.py:145-149` |

> **统一结论**：在"结账转订单但未扣款"阶段：
> - ❌ **不会被 detach 拦截**：旧交易的 `checkout_id` 已清空，不满足 detach 条件
> - ✅ **仍会被余额校验拦截**：只要余额足够，新授权可以成功
> - ⚠️ **存在并发风险**：同一张礼品卡可能被多个订单授权，最终只有先扣款的订单能成功

---

**场景 1：同一礼品卡在两个 checkout 中授权（Checkout 阶段）**

| 操作 | Detach 检查 | 余额校验 | 结果 |
|------|------------|---------|------|
| Checkout A 授权 | - | 通过 | 礼品卡绑定 Transaction A |
| Checkout B 授权 | 触发 detach（Transaction A 仍在 checkout 阶段） | 通过 | Transaction A 被取消，礼品卡绑定 Transaction B |
| Checkout A 完成 | - | - | Transaction A 已被取消，无法扣款 |

**代码证据**：`gateway.py:196-199`
```python
transactions_to_cancel_qs = TransactionItem.objects.filter(
    Q(app_identifier=GIFT_CARD_PAYMENT_GATEWAY_ID),
    Q(gift_card=gift_card),
    Q(checkout_id__isnull=False),  # Transaction A 满足
    Q(order_id__isnull=True),       # Transaction A 满足
)  # → Transaction A 被分离
```

---

**场景 2：礼品卡授权后生成订单，但尚未扣款（订单阶段）**

| 操作 | Detach 检查 | 余额校验 | 结果 |
|------|------------|---------|------|
| Checkout A 授权 | - | 通过 | 礼品卡绑定 Transaction A |
| Checkout A 完成 → Order Z | - | - | Transaction A.order_id = Order Z.id, checkout_id = None |
| Checkout B 授权 | ❌ 不触发（Transaction A.order_id 不为空） | ✅ 余额足够时通过 | Transaction B 也能授权成功 |
| Order Z 先扣款 | - | - | 礼品卡余额减少 |
| Checkout B 完成 → Order Y | - | ❌ 余额不足 | Order Y 扣款失败 |

**代码证据 1：Detach 不分离已关联订单的交易**
`gateway.py:196-199`
```python
transactions_to_cancel_qs = TransactionItem.objects.filter(
    Q(app_identifier=GIFT_CARD_PAYMENT_GATEWAY_ID),
    Q(gift_card=gift_card),
    Q(checkout_id__isnull=False),  # Transaction A.checkout_id = None → 不满足
    Q(order_id__isnull=True),       # Transaction A.order_id = Z → 不满足
)  # → Transaction A 不被分离
```

**代码证据 2：余额校验始终生效**
`gateway.py:145-149`
```python
if transaction_session_data.action.amount > gift_card.current_balance_amount:
    raise GiftCardPaymentGatewayException(
        msg=f"Gift card has insufficient amount ..."
    )
```

**代码证据 3：扣款时二次校验**
`gateway.py:307-314`
```python
if gift_card_transaction.authorized_value > gift_card.current_balance_amount:
    response["message"] = (
        f"Gift card has insufficient amount ..."
    )
else:
    charge_gift_card(gift_card, authorized_value, order)
```

---

**场景 3：订单取消但礼品卡未扣款**
- 需要手动调用 CANCEL 操作释放礼品卡占用
- 或通过订单取消流程自动处理（需检查具体实现）
- 注意：即使不手动取消，新的 checkout 授权仍可能成功（只要余额足够）

---

### 授权入口的额外限制

礼品卡授权只能在 Checkout 上发起，不能直接在 Order 上授权：

**文件**: `saleor/giftcard/gateway.py:111-114`

```python
def validate_transaction_session_data(transaction_session_data, source_object):
    if not isinstance(source_object, Checkout):
        raise GiftCardPaymentGatewayException(
            msg=f"Cannot initialize transaction for payment gateway: {GIFT_CARD_PAYMENT_GATEWAY_ID} and object type other than Checkout."
        )
```

这意味着：
- ✅ 可以在新的 Checkout 上授权已被其他 Order 占用的礼品卡（只要余额足够）
- ❌ 不能直接在 Order 上追加礼品卡授权

---

## 补充细节四：礼品卡履约路径的库存失败分支对照

礼品卡商品的库存检查有两个独立的失败分支，分别抛出不同的异常：

### 两种库存异常的触发位置和场景

| 异常类型 | 触发阶段 | 代码位置 | 触发条件 |
|---------|---------|---------|---------|
| **GiftCardNotApplicable** | 自动履约前的库存检查 | `saleor/giftcard/utils.py:140-144` | 订单行无分配记录且渠道下无可用库存 |
| **InsufficientStock** | 履约创建时的库存扣减 | `saleor/order/actions.py:888-920` | 有库存记录但数量不足 |

### GiftCardNotApplicable 异常分析

**触发时机**：自动履约流程开始时，在 `fulfill_gift_card_lines()` 中检查库存配置

**文件**: `saleor/giftcard/utils.py:139-144`

```python
for line in gift_card_lines.prefetch_related("allocations__stock", "variant__stocks"):
    if allocations := line.allocations.all():
        # 有分配记录，正常处理
        for allocation in allocations:
            ...
    else:
        # 无分配记录，检查渠道下是否有库存
        stock = line.variant.stocks.for_channel(channel_slug).first()
        if not stock:
            raise GiftCardNotApplicable(
                message="Lack of gift card stock for checkout channel.",
            )
```

**触发场景**：
1. 订单行在结账时跳过了库存分配（例如库存追踪被禁用）
2. 分配记录被意外删除
3. 商品在该销售渠道下根本没有配置库存

**测试用例验证**：`saleor/giftcard/tests/test_utils.py:705-725`

```python
def test_fulfill_gift_card_lines_lack_of_stock(...):
    # given
    # 删除该礼品卡商品的所有库存
    gift_card_non_shippable_order_line.variant.stocks.all().delete()
    
    lines = OrderLine.objects.filter(...)
    
    # when & then
    with pytest.raises(GiftCardNotApplicable):
        fulfill_gift_card_lines(lines, staff_user, None, order, site_settings, manager)
```

### InsufficientStock 异常分析

**触发时机**：履约创建过程中，在 `_create_fulfillment_lines()` 中执行实际扣减前

**文件**: `saleor/order/actions.py:888-920`

```python
stock = fulfillment_line.stock

if stock is None:
    warehouse_pk = None
    if not allow_stock_to_be_exceeded:
        error_data = InsufficientStockData(
            variant=variant,
            order_line=order_line,
            warehouse_pk=warehouse_pk,
            available_quantity=0,
        )
        insufficient_stocks.append(error_data)
else:
    warehouse_pk = stock.warehouse_id

if insufficient_stocks:
    raise InsufficientStock(insufficient_stocks)
```

**触发场景**：
1. 结账时分配了库存，但在履约前库存被其他订单占用
2. 库存数量在分配和履约之间发生了变化
3. 手动履约时指定了无货的仓库

### 异常发生时序对照

```
[结账完成]
    ↓
allocate_stocks() → 库存分配
    ↓
[订单创建]
    ↓
[订单支付确认 → 触发自动履约]
    ↓
fulfill_gift_card_lines()
    ├─ 检查 allocations
    └─ 无 allocations 时检查渠道库存
        └─ 无库存 → GiftCardNotApplicable ← 分支 1
    ↓
create_fulfillments()
    ↓
_create_fulfillment_lines()
    ├─ 检查 stock 是否存在
    └─ stock 不存在或不足 → InsufficientStock ← 分支 2
    ↓
decrease_stock() → 实际扣减
    ↓
order_fulfilled()
    ↓
gift_cards_create() → 签发礼品卡
```

### 异常处理策略

| 异常 | 处理方式 | 业务含义 |
|------|---------|---------|
| **GiftCardNotApplicable** | 阻止履约，需人工介入 | 商品配置问题，该渠道根本无法销售此礼品卡 |
| **InsufficientStock** | 阻止履约，等待补货或超售 | 临时缺货，补货后可继续履约 |

> **关键区别**：GiftCardNotApplicable 是**配置级错误**（渠道无库存），InsufficientStock 是**库存数量错误**（有库存但不够）。前者需要检查商品配置，后者需要检查库存数量。
