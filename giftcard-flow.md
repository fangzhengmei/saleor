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

礼品卡本身**不依赖物理库存**，但可以关联到产品（`GiftCard.product` 外键）。这意味着：
- 可以通过购买"礼品卡商品"来获得礼品卡
- 购买后系统自动生成礼品卡实例并发放给客户

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
