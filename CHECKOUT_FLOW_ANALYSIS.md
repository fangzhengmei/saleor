# Saleor 客户结账全链路分析报告

## 概述

本报告梳理 Saleor 电商系统中客户从购物车到订单生成的完整结账流程，重点分析事务边界、异常处理与回滚机制。

## 核心模块概览

| 模块 | 主要职责 | 关键文件 |
|------|---------|---------|
| Checkout 核心 | 购物车管理、结账流程编排 | `saleor/checkout/complete_checkout.py` |
| Checkout 计算 | 价格、税费、运费、折扣计算 | `saleor/checkout/calculations.py` |
| Order 核心 | 订单创建与管理 | `saleor/order/actions.py` |
| Warehouse | 库存管理与分配 | `saleor/warehouse/management.py` |
| Payment | 支付处理 | `saleor/payment/gateway.py` |
| GraphQL API | 对外接口层 | `saleor/graphql/checkout/mutations/checkout_complete.py` |

## 全链路流程

### 第一阶段：请求入口与初步验证

**入口点**：`CheckoutComplete` GraphQL Mutation (`checkout_complete.py:49`)

**流程步骤**：
1. 验证 checkout ID/token，若 checkout 不存在则尝试通过 token 查找已创建的订单（幂等性处理）
2. 验证 checkout 邮箱
3. 拉取 checkout 商品行信息，检查商品变体是否可用
4. 验证配送地址与账单地址
5. 调用核心函数 `complete_checkout()`

---

### 第二阶段：结账流程分支决策

**核心函数**：`complete_checkout()` (`complete_checkout.py:1699`)

根据支付状态选择不同的结账流程：

| 分支条件 | 流程 |
|---------|------|
| checkout 已完全授权 OR 有交易记录 OR 允许未支付订单 OR 零金额订单 | `complete_checkout_with_transaction()` |
| 需要处理支付 | `complete_checkout_with_payment()` |

---

### 第三阶段 A：交易流程（无需实时支付）

**核心函数**：`complete_checkout_with_transaction()` (`complete_checkout.py:1775`)

```
验证 checkout 准备状态
    ↓
调用 create_order_from_checkout() 创建订单
```

---

### 第三阶段 B：支付流程（需实时支付）

**核心函数**：`complete_checkout_with_payment()` (`complete_checkout.py:1823`)

此流程采用**多事务分段**设计，将库存锁定与支付处理分离：

#### 事务 1：锁定库存与准备数据
```
开启事务 [transaction_with_commit_on_errors]
    ↓
select_for_update 锁定 checkout 行
    ↓
设置 completing_started_at 标记（防止并发处理）
    ↓
拉取最新 checkout 数据
    ↓
调用 complete_checkout_pre_payment_part()
    ├─ 验证支付状态
    ├─ 验证配送信息
    ├─ 调用 fetch_checkout_data() 计算价格
    ├─ 检查库存可用性 check_stock_and_preorder_quantity_bulk
    └─ 准备订单数据 _get_order_data()
    ↓
_reserve_stocks_without_availability_check() 预留库存
    ↓
提交事务
```

> **设计意图**：支付处理可能耗时较长（如 3D 安全验证），将库存锁定放在独立事务中，完成后立即释放锁，避免长时间占用数据库行锁影响其他用户。

#### 支付处理（事务外）
```
调用支付网关处理支付 _process_payment()
    ├─ gateway.process_payment() 或 gateway.confirm()
    └─ 支付失败时调用 _complete_checkout_fail_handler() 回滚
```

#### 事务 2：创建订单
```
开启事务 [transaction_with_commit_on_errors]
    ↓
select_for_update 重新锁定 checkout
    ↓
调用 complete_checkout_post_payment_part()
    ├─ 检查是否需要额外验证（如 3D Secure）
    ├─ 调用 _create_order() 创建订单
    ├─ allocate_stocks() 正式分配库存
    ├─ allocate_preorders() 分配预售库存
    ├─ delete_checkouts() 删除 checkout
    └─ transaction.on_commit 触发后续事件
    ↓
提交事务
```

---

### 第四阶段：订单创建核心

**核心函数**：`create_order_from_checkout()` (`complete_checkout.py:1561`)

```
开启事务 [atomic]
    ↓
select_for_update 锁定 checkout
    ↓
如果有凭证券，增加凭证使用量（独立子事务）
    ↓
重新拉取 checkout 数据（确保一致性）
    ↓
调用 _create_order_from_checkout()
    ├─ 创建 Order 记录
    ├─ _create_order_discount() 创建订单折扣
    ├─ _create_order_lines_from_checkout_lines() 创建订单行
    ├─ _handle_allocations_of_order_lines() 分配库存
    ├─ add_gift_cards_to_order() 处理礼品卡
    ├─ 关联支付记录
    ├─ 更新订单搜索向量
    └─ _post_create_order_actions() 后置操作
    ↓
delete_checkouts() 删除 checkout
    ↓
提交事务
```

---

### 第五阶段：库存分配

**核心函数**：`allocate_stocks()` (`warehouse/management.py:91`)

```
@traced_atomic_transaction()
    ↓
过滤需要追踪库存的商品行
    ↓
stock_select_for_update_for_existing_qs() 锁定相关库存行
    ↓
计算现有分配量和预留量
    ↓
按分配策略排序仓库（优先高库存 / 优先仓库排序）
    ↓
_create_allocations() 创建分配记录
    ↓
Stock.objects.bulk_update() 更新 quantity_allocated（使用 F() 表达式原子更新）
    ↓
检查库存是否售罄，触发 out_of_stock 事件
```

**预售库存分配**：`allocate_preorders()` (`warehouse/management.py:811`)
- 锁定 ProductVariantChannelListing 行
- 检查渠道阈值和全局阈值
- 创建 PreorderAllocation 记录

---

## 价格计算链路

### 核心计算流程

**入口函数**：`fetch_checkout_data()` (`checkout/calculations.py`)

```
计算基础价格（base_calculations）
    ├─ base_checkout_delivery_price() 基础运费
    ├─ calculate_base_line_unit_price() 基础单价
    └─ calculate_undiscounted_base_line_unit_price() 未折扣单价
    ↓
应用促销折扣
    └─ create_or_update_discount_objects_from_promotion_for_checkout()
    ↓
计算税费
    ├─ update_checkout_prices_with_flat_rates() 固定税率
    └─ 调用 Tax App 计算（如已配置）
    ↓
计算最终价格
    ├─ checkout_shipping_price() 运费（含税）
    ├─ checkout_subtotal() 小计（含税）
    └─ calculate_checkout_total() 总计
```

### 价格计算的事务边界

价格计算通常**不在数据库事务内**执行，原因：
1. 可能调用外部 Tax App，网络延迟不可控
2. 计算过程只读，不修改数据
3. 多次调用可接受最终一致性

---

## 事务边界分析

### 关键事务装饰器

| 装饰器 | 用途 | 位置 |
|--------|------|------|
| `@traced_atomic_transaction()` | 带追踪的数据库事务，自动回滚 | 库存操作、订单创建 |
| `transaction_with_commit_on_errors()` | 即使发生异常也先提交事务，再抛出 | 支付流程分段 |
| `transaction.atomic()` | 标准 Django 事务 | 凭证使用量更新 |

### 主要事务边界

```
┌─────────────────────────────────────────────────────────┐
│ complete_checkout_with_payment                          │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │ 事务 1: 库存锁定与数据准备                        │  │
│  │  - select_for_update(checkout)                   │  │
│  │  - 价格计算（无事务）                            │  │
│  │  - 库存预留                                      │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  支付处理（事务外，可能耗时）                           │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │ 事务 2: 订单创建                                  │  │
│  │  - select_for_update(checkout)                   │  │
│  │  - @traced_atomic_transaction: allocate_stocks   │  │
│  │  - @traced_atomic_transaction: allocate_preorders│  │
│  │  - 删除 checkout                                  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 行级锁定策略

**锁定对象**：
- `Checkout` 表：使用 `select_for_update()` 防止并发处理同一 checkout
- `Stock` 表：使用 `stock_select_for_update_for_existing_qs()` 锁定库存行
- `ProductVariantChannelListing` 表：预售库存锁定
- `Payment` 表：支付处理时锁定支付记录

**锁定顺序**：
1. 始终按主键排序锁定，避免死锁
2. 库存表按 `stock_id` 排序
3. 分配表按 `stock_id` 排序

---

## 异常处理与回滚机制

### 核心失败处理器

**函数**：`_complete_checkout_fail_handler()` (`complete_checkout.py:1996`)

**回滚操作**：
1. **释放 checkout 处理标记**：清空 `completing_started_at`
2. **回滚凭证使用**：调用 `_release_checkout_voucher_usage()`
3. **退款/作废支付**：调用 `gateway.payment_refund_or_void()`

### 异常场景与回滚

| 异常类型 | 触发点 | 回滚操作 |
|---------|--------|---------|
| `InsufficientStock` | 库存验证/分配时 | 释放凭证、退款支付 |
| `GiftCardNotApplicable` | 礼品卡验证时 | 释放凭证、退款支付 |
| `NotApplicable` | 凭证不可用时 | 释放凭证、退款支付 |
| `PaymentError` | 支付处理失败 | 释放凭证、退款支付 |
| `TaxError` | 税费计算失败 | 释放凭证 |
| `ValidationError` | 数据验证失败 | 释放凭证、退款支付 |

### 幂等性设计

1. **Checkout 已删除处理**：如果 checkout 不存在，尝试通过 token 查找订单
2. **订单已创建处理**：`_create_order()` 首先检查是否已存在该 checkout_token 的订单
3. **重复请求防护**：`completing_started_at` 标记防止并发处理

### 部分失败处理

**支付成功但订单创建失败**：
- 支付记录已关联 checkout
- `_complete_checkout_fail_handler()` 会自动退款
- 用户可重新发起结账

**订单创建成功但 webhook 通知失败**：
- 使用 `transaction.on_commit()` 确保事务提交后才发送
- 通知失败不影响订单状态，依赖异步重试机制

---

## 并发控制机制

### 1. 乐观锁：Checkout 处理标记

```python
checkout.completing_started_at = timezone.now()
checkout.save(update_fields=["completing_started_at"])
```

其他请求看到此标记后可拒绝处理。

### 2. 悲观锁：数据库行锁

```python
Checkout.objects.select_for_update().filter(pk=checkout_pk).first()
```

### 3. 原子更新：F() 表达式

```python
Stock.objects.filter(pk=stock_pk).update(
    quantity_allocated=F("quantity_allocated") + quantity
)
```

避免读取-修改-写入的竞态条件。

### 4. 唯一约束：防止重复

- Order 表的 `checkout_token` 唯一索引
- Voucher 使用记录的唯一约束

---

## 关键设计决策

### 1. 支付处理移出事务

**原因**：
- 支付网关调用可能耗时数秒甚至数十秒
- 长时间持有数据库行锁会导致严重的并发问题
- 库存预留后，其他用户看到的是已预留状态

**代价**：
- 需要处理支付成功但订单创建失败的边缘情况
- 增加了回滚逻辑的复杂度

### 2. 多事务分段

**设计模式**：
```
锁定资源 → 释放锁 → 耗时操作 → 重新锁定 → 最终确认
```

适用于包含外部调用的长流程。

### 3. transaction.on_commit 的使用

所有异步事件（订单创建通知、库存预警等）都通过 `transaction.on_commit()` 注册，确保：
- 只有事务真正提交后才触发
- 事务回滚时不会发送错误通知

---

## 总结

Saleor 结账流程采用了**分段事务+悲观锁+补偿机制**的设计模式：

1. **数据一致性**：通过数据库事务和行级锁保证核心数据一致性
2. **并发性能**：将耗时操作（支付）移出事务，减少锁持有时间
3. **故障恢复**：通过 `_complete_checkout_fail_handler()` 实现自动回滚
4. **幂等性**：多重检查确保重复请求不会产生重复订单

该设计在保证数据正确性的同时，最大限度地提升了系统的并发处理能力。
