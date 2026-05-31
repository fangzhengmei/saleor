# Saleor 内容导航与菜单结构扩展实现分析（代码核实版）

## 系统分层架构总览

```
┌───────────────────────────────────────────────────────────┐
│                     GraphQL API Layer                     │
│  Types / Schema / Resolvers / Mutations / Permissions    │
├───────────────────────────────────────────────────────────┤
│                  Business Logic Layer                     │
│    Mutations Logic / Validations / call_event / Move     │
├───────────────────────────────────────────────────────────┤
│                 Plugin & Webhook Layer                    │
│  BasePlugin / PluginManager / WebhookPlugin / AppExtension│
├───────────────────────────────────────────────────────────┤
│              Context & DataLoader Layer                   │
│  ChannelContext / ChannelQsContext / DataLoader / Cache   │
├───────────────────────────────────────────────────────────┤
│           Permissions & Localization Layer                │
│  Permission Enums / Translation Models / TranslationField │
├───────────────────────────────────────────────────────────┤
│                    Data Model Layer                       │
│  Menu / MenuItem / MenuItemTranslation / AppExtension    │
│                SiteSettings / SiteSettingsTranslation     │
└───────────────────────────────────────────────────────────┘
```

---

## 1. 数据模型层 (Data Model Layer)

### 1.1 菜单核心数据模型

**文件位置**: `saleor/menu/models.py`

#### Menu 模型 (`saleor/menu/models.py:12-21`)

```python
class Menu(ModelWithMetadata):
    name = models.CharField(max_length=250)
    slug = models.SlugField(max_length=255, unique=True, allow_unicode=True)

    class Meta(ModelWithMetadata.Meta):
        ordering = ("pk",)
        permissions = ((MenuPermissions.MANAGE_MENUS.codename, "Manage navigation."),)
```

- **核心字段**:
  - `name`: 菜单名称，用于后台识别
  - `slug`: 唯一标识符，支持 Unicode
  - 继承 `ModelWithMetadata` 支持自定义元数据
- **重要特征**: Menu 本身没有 Translation 模型，**Menu 的 name 字段不可翻译**——这是与 MenuItem 的关键差异

#### MenuItem 模型 (`saleor/menu/models.py:24-61`)

```python
class MenuItem(ModelWithMetadata, MPTTModel, SortableModel):
    menu = models.ForeignKey(Menu, related_name="items", on_delete=models.CASCADE)
    name = models.CharField(max_length=128)
    parent = models.ForeignKey(
        "self", null=True, blank=True, related_name="children", on_delete=models.CASCADE
    )
    url = models.URLField(max_length=256, blank=True, null=True)
    category = models.ForeignKey(Category, blank=True, null=True, on_delete=models.CASCADE)
    collection = models.ForeignKey(Collection, blank=True, null=True, on_delete=models.CASCADE)
    page = models.ForeignKey(Page, blank=True, null=True, on_delete=models.CASCADE)
```

- **树结构**: 使用 `MPTTModel` 实现嵌套层级结构
- **链接类型**（互斥，由 `MenuItemCreate.clean_input` 和 `MenuItemUpdate.construct_instance` 强制执行）:
  1. `url`: 外部或自定义 URL
  2. `category`: 关联商品分类
  3. `collection`: 关联商品集合
  4. `page`: 关联 CMS 页面
- **排序**: 通过 `SortableModel` 支持同级排序，`get_ordering_queryset` 根据是否有 parent 决定排序范围
- **linked_object 属性**: `saleor/menu/models.py:58-60` 返回第一个非空的关联对象 (`category or collection or page`)

#### MenuItemTranslation 模型 (`saleor/menu/models.py:63-86`)

```python
class MenuItemTranslation(Translation):
    menu_item = models.ForeignKey(MenuItem, related_name="translations", on_delete=models.CASCADE)
    name = models.CharField(max_length=128)

    class Meta:
        ordering = ("language_code", "menu_item", "pk")
        unique_together = (("language_code", "menu_item"),)
```

- 实现菜单项名称的多语言翻译
- 基于 `language_code` 和 `menu_item` 的联合唯一约束
- 仅翻译 `name` 字段

### 1.2 应用扩展点模型

**文件位置**: `saleor/app/models.py:155-174`

```python
class AppExtension(models.Model):
    app = models.ForeignKey(App, on_delete=models.CASCADE, related_name="extensions")
    label = models.CharField(max_length=256)
    url = models.URLField()
    mount = models.CharField(max_length=256)
    target = models.CharField(max_length=128, default=DEFAULT_APP_TARGET)
    permissions = models.ManyToManyField(Permission, blank=True)
    http_target_method = models.CharField(blank=False, null=True, choices=...)
    settings = models.JSONField(blank=True, default=dict, db_default={})
```

- **核心字段说明**:
  - `mount`: 扩展挂载点，自由字符串（migration 0037 已移除枚举约束，改为开放字符串）
  - `target`: 打开方式（`popup`/`new_tab`/`app_page`/`widget`）
  - `permissions`: 控制谁可以看到和使用该扩展（必须是 App permissions 的子集）
  - `settings`: JSON 格式的扩展自定义配置

### 1.3 站点设置关联

**文件位置**: `saleor/site/models.py:38-47`

```python
class SiteSettings(ModelWithMetadata):
    site = models.OneToOneField(Site, related_name="settings", on_delete=models.CASCADE)
    top_menu = models.ForeignKey("menu.Menu", on_delete=models.SET_NULL, related_name="+", blank=True, null=True)
    bottom_menu = models.ForeignKey("menu.Menu", on_delete=models.SET_NULL, related_name="+", blank=True, null=True)
```

- 每个站点通过 `top_menu` 和 `bottom_menu` 关联两个导航菜单
- `related_name="+"` 表示不建立反向关系，因为这两个字段只是配置指针
- 站点自身有翻译模型 `SiteSettingsTranslation`，翻译 `header_text` 和 `description`

---

## 2. 应用扩展挂载点 (Mount Points)——代码核实

### 2.1 挂载点字段演变历史

| Migration | 变更 |
|-----------|------|
| `0005_appextension` | 初始创建，mount 为枚举字段，仅有 `PRODUCT_OVERVIEW_MORE_ACTIONS` |
| `0006_convert_extension_enums_into_one` | 统一枚举，增加 `PRODUCT_OVERVIEW_CREATE`, `PRODUCT_DETAILS_MORE_ACTIONS` |
| `0013_alter_appextension_mount` | 增加 `ORDER_DETAILS_MORE_ACTIONS` |
| `0016_alter_appextension_mount` | 无新增（调整 target） |
| `0031_alter_appextension_mount_alter_appextension_target` | 增加多个 mount 点（见下表） |
| `0032_appextension_http_target_method_and_more` | 增加 `product_details_widgets`, `order_details_widgets` 等 |
| `0034_alter_appextension_mount` | 确认已有列表 |
| **0037_app_extensions_loosen_mount** | **移除枚举约束，mount 改为纯 CharField(max_length=256)** |

**关键结论**: 自 migration 0037 起，`mount` 字段为**开放字符串**，不再受枚举限制。前端 Dashboard 负责解析 mount 值并决定在哪个位置渲染扩展。

### 2.2 已知挂载点完整列表（来自迁移历史和测试代码）

以下为迁移 0032/0034 中记录的最终枚举值，代表 Dashboard 已注册的挂载位置:

| mount 值（小写存储） | GraphQL 返回值（大写） | 含义 |
|----------------------|----------------------|------|
| `product_overview_create` | `PRODUCT_OVERVIEW_CREATE` | 商品列表页「创建」按钮区 |
| `product_overview_more_actions` | `PRODUCT_OVERVIEW_MORE_ACTIONS` | 商品列表页「更多操作」菜单 |
| `product_details_more_actions` | `PRODUCT_DETAILS_MORE_ACTIONS` | 商品详情页「更多操作」菜单 |
| `product_details_widgets` | `PRODUCT_DETAILS_WIDGETS` | 商品详情页小部件区 |
| `order_details_more_actions` | `ORDER_DETAILS_MORE_ACTIONS` | 订单详情页「更多操作」菜单 |
| `order_details_widgets` | `ORDER_DETAILS_WIDGETS` | 订单详情页小部件区 |
| `draft_order_details_more_actions` | `DRAFT_ORDER_DETAILS_MORE_ACTIONS` | 草稿订单详情页「更多操作」 |
| `draft_order_overview_create` | `DRAFT_ORDER_OVERVIEW_CREATE` | 草稿订单列表页「创建」按钮区 |
| `draft_order_overview_more_actions` | `DRAFT_ORDER_OVERVIEW_MORE_ACTIONS` | 草稿订单列表页「更多操作」 |
| `draft_order_details_widgets` | `DRAFT_ORDER_DETAILS_WIDGETS` | 草稿订单详情页小部件区 |

> **澄清**: `NAVIGATION_SIDEBAR` 在之前的分析中被错误列出。代码中不存在此挂载点。App 扩展的挂载点全部面向 Dashboard 后台页面，不直接操作前台导航菜单。

### 2.3 挂载点的大小写处理

**存储层** (`saleor/app/manifest_validations.py:205`): mount 值在写入数据库前被转为小写:
```python
extension["mount"] = extension["mount"].lower()
```

**查询层** (`saleor/graphql/app/filters.py:23-26`): 过滤时将输入值转为小写进行匹配:
```python
def filter_app_extension_mount_name(qs, _, value):
    if value:
        qs = qs.filter(mount__in=[v.lower() for v in value])
    return qs
```

**GraphQL 输出层** (`saleor/graphql/app/types.py:202-203`): 返回时转为大写:
```python
@staticmethod
def resolve_mount_name(root: models.AppExtension, _info: ResolveInfo):
    return root.mount.upper()
```

**协议**: DB 存小写 → 查询用小写匹配 → GraphQL 返回大写。

### 2.4 target 字段值

| target 值 | 含义 |
|-----------|------|
| `popup` | 在弹出窗口中打开（默认值，`DEFAULT_APP_TARGET`） |
| `app_page` | 在应用页面内导航（相对 URL 需要 `appUrl`） |
| `new_tab` | 在新标签页打开（已弃用，迁移到 settings 新模型） |
| `widget` | 嵌入式小部件（已弃用，迁移到 settings 新模型） |

---

## 3. 站点上下文与隔离机制——边界核实

### 3.1 隔离层级分析

Saleor 的菜单隔离分为**两个独立的维度**:

#### 维度一: 站点 (Site) 隔离

**隔离粒度**: 菜单**分配**按站点隔离，但菜单**数据**本身全局共享。

```
Site A (SiteSettings)
  ├── top_menu → Menu#1
  └── bottom_menu → Menu#2

Site B (SiteSettings)
  ├── top_menu → Menu#3
  └── bottom_menu → Menu#1   ← 可以与 Site A 共享同一个 Menu
```

- **SiteSettings** 通过 `OneToOneField` 与 Django `Site` 绑定 (`saleor/site/models.py:39`)
- `AssignNavigation` mutation 操作的是**当前请求站点**的设置 (`saleor/graphql/menu/mutations/assign_navigation.py:35`)
- **菜单和菜单项本身没有 site 外键**——它们是全局数据
- 站点隔离仅发生在「哪个菜单被分配给哪个位置」这一层

#### 维度二: 渠道 (Channel) 隔离

**隔离粒度**: 菜单结构不按渠道隔离，但菜单项关联的**内容实体**（collection、category、page）按渠道做可见性过滤。

`ChannelContext` 结构 (`saleor/graphql/core/context.py:88-89`):
```python
@dataclass
class ChannelContext(BaseContext[N]):
    channel_slug: str | None
```

**菜单查询时 channel 的作用** (`saleor/graphql/menu/schema.py:74-82`):
```python
@staticmethod
def resolve_menu(_root, info: ResolveInfo, *, channel=None, **data):
    if channel is None:
        channel = get_default_channel_slug_or_graphql_error(
            allow_replica=info.context.allow_replica
        )
    return resolve_menu(info, channel, data.get("id"), data.get("name"), data.get("slug"))
```

**关键发现**: Menu/MenuItem 查询中 `channel` 参数主要用于:
1. 作为 `ChannelContext` 的 `channel_slug` 传递给子字段解析器
2. 在 `MenuItem.resolve_collection` 中检查 collection 在该渠道是否可见
3. 在 `MenuItem.resolve_page` 中不使用渠道，仅检查页面可见性和权限
4. `MenuItem.resolve_category` 不做渠道过滤（category 无渠道列表概念）

**菜单查询本身不按渠道过滤**——`resolve_menu` 和 `resolve_menus` 返回全局数据，`channel_slug` 只是作为上下文信息向下游传递。

### 3.2 数据加载器的隔离特征

**文件位置**: `saleor/graphql/menu/dataloaders.py`

```python
class MenuByIdLoader(DataLoader[int, Menu]):
    context_key = "menu_by_id"

    def batch_load(self, keys):
        menus = Menu.objects.using(self.database_connection_name).in_bulk(keys)
        return [menus.get(menu_id) for menu_id in keys]
```

- `MenuItemsByParentMenuLoader` 仅按 `menu_id` 和 `level=0` 过滤，无渠道或站点条件
- DataLoader 的隔离**仅体现在数据库连接选择**（主库/副本），不体现业务过滤

### 3.3 隔离边界总结

| 层面 | 是否隔离 | 实现方式 |
|------|---------|---------|
| 菜单分配（哪个菜单在哪个位置） | **按站点隔离** | `SiteSettings.top_menu / bottom_menu` |
| 菜单数据本身 | **全局共享** | 无 site 外键 |
| 菜单项结构 | **全局共享** | 跟随 Menu |
| 菜单项关联的 Collection | **按渠道可见性过滤** | `CollectionChannelListingByCollectionIdAndChannelSlugLoader` |
| 菜单项关联的 Page | **按 is_visible 过滤** | 不涉及渠道，仅检查发布状态 |
| 菜单项关联的 Category | **无过滤** | 直接返回关联对象 |
| AppExtension 挂载 | **全局共享** | 无站点/渠道隔离 |

---

## 4. 菜单变更事件广播链路——完整核实

### 4.1 call_event 机制

**文件位置**: `saleor/core/utils/events.py:19-28`

```python
def call_event(func_obj, *func_args, **func_kwargs):
    connection = transaction.get_connection()
    if connection.in_atomic_block:
        transaction.on_commit(lambda: func_obj(*func_args, **func_kwargs))
    else:
        func_obj(*func_args, **func_kwargs)
```

**缓存生效条件**: 如果 mutation 在原子事务中执行（大多数 mutation 如此），事件处理器将在事务提交后才触发。这意味着:
- 事务回滚时事件不会触发
- 事件触发时数据已持久化
- DataLoader 缓存可能仍持有旧数据直到请求结束

### 4.2 完整 Mutation-Event-Webhook 映射

| Mutation | 文件 | post_save_action 调用 | Webhook 事件类型 | Payload 内容 |
|----------|------|----------------------|-----------------|-------------|
| `MenuCreate` | `menu_create.py:111-113` | `manager.menu_created` | `MENU_CREATED` | `{id, slug, meta}` |
| `MenuUpdate` | `menu_update.py:41-44` | `manager.menu_updated` | `MENU_UPDATED` | `{id, slug, meta}` |
| `MenuDelete` | `menu_delete.py:34-36` | `manager.menu_deleted` | `MENU_DELETED` | `{id, slug, meta}` |
| `MenuBulkDelete` | `menu_bulk_delete.py:36-42` | `manager.menu_deleted`（逐个） | `MENU_DELETED` | `{id, slug, meta}` |
| `MenuItemCreate` | `menu_item_create.py:67-70` | `manager.menu_item_created` | `MENU_ITEM_CREATED` | `{id, name, menu:{id}, meta}` |
| `MenuItemUpdate` | `menu_item_update.py:48-51` | `manager.menu_item_updated` | `MENU_ITEM_UPDATED` | `{id, name, menu:{id}, meta}` |
| `MenuItemDelete` | `menu_item_delete.py:34-36` | `manager.menu_item_deleted` | `MENU_ITEM_DELETED` | `{id, name, menu:{id}, meta}` |
| `MenuItemBulkDelete` | `menu_item_bulk_delete.py:36-42` | `manager.menu_item_deleted`（逐个） | `MENU_ITEM_DELETED` | `{id, name, menu:{id}, meta}` |
| `MenuItemMove` | `menu_item_move.py:196-197` | `manager.menu_item_updated`（条件触发） | `MENU_ITEM_UPDATED` | `{id, name, menu:{id}, meta}` |
| `AssignNavigation` | `assign_navigation.py:32-48` | **无 post_save_action** | **无 Webhook 事件** | — |
| `MenuItemTranslate` | `menu_item_translate.py:13-42` | **通过 BaseTranslateMutation** | `TRANSLATION_CREATED` / `TRANSLATION_UPDATED` | 翻译对象列表 |

### 4.3 MenuItemMove 的条件触发

**文件位置**: `saleor/graphql/menu/mutations/menu_item_move.py:196-200`

```python
if operation.sort_order or operation.parent_changed:
    cls.call_event(manager.menu_item_updated, menu_item)

menu = qs.get(pk=menu.pk)
MenuItemsByParentMenuLoader(info.context).clear(menu.id)
```

**关键发现**:
1. `menu_item_updated` 事件仅在 `sort_order` 或 `parent_changed` 时触发
2. 这是**唯一一个手动清除 DataLoader 缓存**的 mutation（`MenuItemsByParentMenuLoader(info.context).clear(menu.id)`）
3. 其他 mutation 不清除 DataLoader 缓存——依赖请求级别的 DataLoader 生命周期自动过期

### 4.4 AssignNavigation 无事件广播

**文件位置**: `saleor/graphql/menu/mutations/assign_navigation.py:14-48`

`AssignNavigation` mutation 直接修改 `site.settings.top_menu` / `bottom_menu` 并 `save()`，但:
- 没有 `post_save_action`
- 没有声明 `webhook_events_info`
- 没有触发任何菜单相关事件

**影响**: 当导航分配变更时，订阅了 `MENU_UPDATED` 的 Webhook 不会收到通知。前端/CDN 缓存只能通过其他机制（如轮询或站点设置变更事件）感知此变更。

### 4.5 WebhookPlugin 事件广播实现

**文件位置**: `saleor/plugins/webhook/plugin.py:807-884`

Menu 事件 payload (`_trigger_menu_event`):
```python
payload = {
    "id": graphene.Node.to_global_id("Menu", menu.id),
    "slug": menu.slug,
    "meta": self._generate_meta(),
}
```

MenuItem 事件 payload (`__trigger_menu_item_event`):
```python
payload = {
    "id": graphene.Node.to_global_id("MenuItem", menu_item.id),
    "name": menu_item.name,
    "menu": {
        "id": graphene.Node.to_global_id("Menu", menu_item.menu_id)
    },
    "meta": self._generate_meta(),
}
```

**所有菜单事件的 Webhook 权限要求** (`saleor/webhook/event_types.py:358-381`):
```python
MENU_CREATED:    {"name": "Menu created",    "permission": MenuPermissions.MANAGE_MENUS}
MENU_UPDATED:    {"name": "Menu updated",    "permission": MenuPermissions.MANAGE_MENUS}
MENU_DELETED:    {"name": "Menu deleted",    "permission": MenuPermissions.MANAGE_MENUS}
MENU_ITEM_CREATED: {"name": "Menu item created", "permission": MenuPermissions.MANAGE_MENUS}
MENU_ITEM_UPDATED: {"name": "Menu item updated", "permission": MenuPermissions.MANAGE_MENUS}
MENU_ITEM_DELETED: {"name": "Menu item deleted", "permission": MenuPermissions.MANAGE_MENUS}
```

只有拥有 `MANAGE_MENUS` 权限的 App 才能接收菜单 Webhook 事件。

### 4.6 DataLoader 缓存生效条件

| DataLoader | context_key | 缓存清除方式 |
|-----------|------------|-------------|
| `MenuByIdLoader` | `menu_by_id` | 请求级生命周期自动过期 |
| `MenuItemByIdLoader` | `menuitem_by_id` | 请求级生命周期自动过期 |
| `MenuItemsByParentMenuLoader` | `menuitems_by_parent_menu` | MenuItemMove 手动 clear；其他 mutation 请求级过期 |
| `MenuItemChildrenLoader` | `menuitem_children` | 请求级生命周期自动过期 |

DataLoader 的缓存绑定在 `SaleorContext.dataloaders` 字典上 (`saleor/graphql/core/context.py:20`)，每个 HTTP 请求创建新的 dataloader 实例，因此缓存仅在单次请求内有效。

---

## 5. 权限系统——一致性核实

### 5.1 权限分配一致性矩阵

| 操作 | 声明权限 (Meta.permissions) | Webhook 接收权限 |
|------|---------------------------|-----------------|
| MenuCreate | `MANAGE_MENUS` | `MANAGE_MENUS` |
| MenuUpdate | `MANAGE_MENUS` | `MANAGE_MENUS` |
| MenuDelete | `MANAGE_MENUS` | `MANAGE_MENUS` |
| MenuBulkDelete | `MANAGE_MENUS` | `MANAGE_MENUS` |
| MenuItemCreate | `MANAGE_MENUS` | `MANAGE_MENUS` |
| MenuItemUpdate | `MANAGE_MENUS` | `MANAGE_MENUS` |
| MenuItemDelete | `MANAGE_MENUS` | `MANAGE_MENUS` |
| MenuItemBulkDelete | `MANAGE_MENUS` | `MANAGE_MENUS` |
| MenuItemMove | `MANAGE_MENUS` | `MANAGE_MENUS` |
| **AssignNavigation** | **`MANAGE_MENUS` + `MANAGE_SETTINGS`** | **无事件** |
| **MenuItemTranslate** | **`MANAGE_TRANSLATIONS`** | **`MANAGE_TRANSLATIONS`**（translations 事件） |

**发现 1**: AssignNavigation 同时需要 `MANAGE_MENUS` 和 `MANAGE_SETTINGS`，但变更后不广播菜单事件。这是一个**权限覆盖但不通知**的缺口——拥有 `MANAGE_SETTINGS` 但没有 `MANAGE_MENUS` 的用户可以改变导航分配，但无法通过 Webhook 被通知。

**发现 2**: MenuItemTranslate 使用 `MANAGE_TRANSLATIONS` 权限而非 `MANAGE_MENUS`，触发的也是 `TRANSLATION_CREATED`/`TRANSLATION_UPDATED` 事件而非菜单事件。翻译变更不会触发菜单内容的缓存失效通知。

### 5.2 菜单项关联内容的读取权限

| 关联字段 | 权限检查 | 无权限时的行为 |
|---------|---------|--------------|
| `category` | 无权限检查 | 始终返回关联 Category |
| `collection` | `ALL_PRODUCTS_PERMISSIONS` | 检查渠道是否激活 + collection 是否在渠道中可见，不可见返回 `None` |
| `page` | `MANAGE_PAGES` | 未发布页面返回 `None`；有权限时返回所有页面 |

**不一致性**: `category` 无任何权限/可见性检查，而 `collection` 和 `page` 均有。这意味着未发布的 Category 可以通过菜单项被暴露。

### 5.3 AppExtension 权限约束

**文件位置**: `saleor/app/manifest_validations.py:173-193`

```python
def _clean_extension_permissions(extension, app_permissions, errors):
    # ...
    if len(extension_permissions) != len(permissions_data):
        errors["extensions"].append(
            ValidationError(
                "Extension permission must be listed in App's permissions.",
                code=AppErrorCode.OUT_OF_SCOPE_PERMISSION.value,
            )
        )
```

Extension 的权限必须是 App 已声明权限的子集。App 不能通过 Extension 获得自身未申请的权限。

---

## 6. 本地化与多语言——一致性核实

### 6.1 翻译覆盖范围

| 模型 | 可翻译字段 | Translation 模型 | Translate Mutation | 翻译查询入口 |
|------|----------|-----------------|-------------------|------------|
| `Menu` | **无** | **不存在** | **不存在** | — |
| `MenuItem` | `name` | `MenuItemTranslation` | `MenuItemTranslate` | `MenuItem.translation` 字段 + `translations` 查询 |
| `SiteSettings` | `header_text`, `description` | `SiteSettingsTranslation` | (通用站点设置更新) | 上下文处理器预加载 |

### 6.2 翻译处理的不一致性

**问题 1: Menu.name 不可翻译**

Menu 模型没有 Translation 子类，菜单名称无法多语言化。当菜单被分配到 `top_menu` / `bottom_menu` 位置时，前端只能显示原始名称。如果需要多语言菜单名，需创建多个 Menu 对象。

**问题 2: MenuItem 翻译通过独立的 Translation 事件广播**

`MenuItemTranslate` 继承 `BaseTranslateMutation` (`saleor/graphql/translations/mutations/utils.py:100-162`)，触发的是 `translations_created` / `translations_updated` 事件，而非菜单相关事件:

```python
manager = get_plugin_manager_promise(info.context).get()
if created:
    cls.call_event(manager.translations_created, [translation])
else:
    cls.call_event(manager.translations_updated, [translation])
```

这意味着翻译变更与菜单内容变更走的是**不同的事件通道**。只订阅 `MENU_ITEM_UPDATED` 的系统不会收到翻译变更通知。

**问题 3: MenuItemTranslatableContent 的双 ID 支持**

`BaseTranslateMutation.clean_node_id` (`saleor/graphql/translations/mutations/utils.py:105-132`) 支持:
1. MenuItem 的 ID → 直接翻译
2. `MenuItemTranslatableContent` 的 ID → 自动转换为 MenuItem ID

GraphQL 类型映射 (`saleor/graphql/translations/schema.py:42`):
```python
TYPES_TRANSLATIONS_MAP = {
    MenuItem: translation_types.MenuItemTranslatableContent,
    ...
}
```

`MenuItemTranslatableContent` (`saleor/graphql/translations/types.py:892-906`) 暴露:
```python
class MenuItemTranslatableContent(ModelObjectType[menu_models.MenuItem]):
    id = graphene.GlobalID(required=True)
    menu_item_id = graphene.ID(required=True)
    name = graphene.String(required=True)
    translation = TranslationField(MenuItemTranslation, type_name="menu item")
    menu_item = graphene.Field("saleor.graphql.menu.types.MenuItem")
```

**翻译查询统一走 `translations` 查询入口** (`saleor/graphql/translations/schema.py:89-98`)，需要 `MANAGE_TRANSLATIONS` 权限，`kind=MENU_ITEM` 过滤。

### 6.3 翻译与权限的交叉

| 操作 | 权限 | 与菜单管理权限的关系 |
|------|------|-------------------|
| 修改菜单项结构（增删改移） | `MANAGE_MENUS` | — |
| 修改菜单项翻译 | `MANAGE_TRANSLATIONS` | 独立权限，不要求 `MANAGE_MENUS` |
| 查看翻译列表 | `MANAGE_TRANSLATIONS` | 独立权限 |
| 修改站点设置翻译 | `MANAGE_TRANSLATIONS` | 独立权限 |

**潜在问题**: 拥有 `MANAGE_TRANSLATIONS` 但没有 `MANAGE_MENUS` 的用户可以修改菜单项名称的翻译，但不能修改菜单结构。这是设计意图（权限分离），但翻译修改不触发菜单缓存失效事件。

---

## 7. GraphQL API 层完整结构

### 7.1 Schema 定义

**文件位置**: `saleor/graphql/menu/schema.py`

**Queries**:
| 字段 | 参数 | 说明 |
|------|------|------|
| `menu` | `channel`, `id`?, `name`?, `slug`? | 按ID/名称/slug查询单个菜单 |
| `menus` | `channel`, `sort_by`?, `filter`? | 列表查询，支持排序和过滤 |
| `menu_item` | `channel`, `id` | 按ID查询单个菜单项 |
| `menu_items` | `channel`, `sort_by`?, `filter`? | 列表查询 |

**Mutations**:
| 字段 | 说明 |
|------|------|
| `assign_navigation` | 分配导航菜单到站点位置 |
| `menu_create` | 创建菜单 |
| `menu_delete` | 删除菜单 |
| `menu_bulk_delete` | 批量删除菜单 |
| `menu_update` | 更新菜单 |
| `menu_item_create` | 创建菜单项 |
| `menu_item_delete` | 删除菜单项 |
| `menu_item_bulk_delete` | 批量删除菜单项 |
| `menu_item_update` | 更新菜单项 |
| `menu_item_translate` | 翻译菜单项 |
| `menu_item_move` | 移动菜单项位置/层级 |

### 7.2 过滤器

**文件位置**: `saleor/graphql/menu/filters.py`

- **MenuFilter**: `search`（名称/slug 模糊搜索）, `slug`, `slugs`
- **MenuItemFilter**: `search`（名称模糊搜索）

**注意**: 过滤器不包含站点或渠道过滤——菜单数据本身不做站点/渠道隔离。

### 7.3 AppExtension 过滤器

**文件位置**: `saleor/graphql/app/filters.py:23-58`

- **mountName**: 列表过滤，大小写不敏感
- **targetName**: 单值过滤，大小写不敏感

---

## 8. 关键交互流程（修正版）

### 8.1 插件注册导航项完整流程

```
App Manifest 声明
       ↓
1. 定义 extensions 数组，指定 mount/label/url/target/permissions/options
       ↓
Manifest 验证 (manifest_validations.py / manifest_schema.py)
       ↓
2. Pydantic 验证结构完整性 (ManifestExtensionSchema)
3. mount 值转小写存储
4. target 值转小写存储
5. 验证 extension URL（相对路径需 appUrl 配合）
6. 验证 extension permissions 是 App permissions 的子集
       ↓
App Installation (installation_utils.py)
       ↓
7. 创建 AppExtension 记录，存入 mount/target/url/settings
8. 设置 extension.permissions 关联
       ↓
Dashboard 前端渲染
       ↓
9. 查询 appExtensions，可按 mountName 过滤
10. AppExtension.resolve_app 检查访问权限
    (Owner 或 MANAGE_APPS 或 staff)
11. AppExtension.resolve_permissions 返回所需权限列表
12. 前端根据用户权限决定是否渲染扩展入口
```

### 8.2 菜单变更完整事件链路

```
Mutation 执行
       ↓
BaseMutation.mutate() → check_permissions → perform_mutation
       ↓
post_save_action / bulk_action
       ↓
call_event(manager.menu_*, instance)  ← call_event 检查是否在事务中
       ↓
[如果在事务中] → transaction.on_commit() 延迟触发
[如果不在事务中] → 立即触发
       ↓
PluginManager.__run_method_on_plugins("menu_updated", ...)
       ↓
遍历所有激活插件 → WebhookPlugin.menu_updated()
       ↓
WebhookPlugin._trigger_menu_event():
  1. _get_webhooks_for_event(event_type) → 查找订阅了该事件且拥有 MANAGE_MENUS 权限的 Webhook
  2. 构造 payload {id, slug, meta} 或 {id, name, menu:{id}, meta}
  3. trigger_webhooks_async(payload, event_type, webhooks, ...)
       ↓
异步 Webhook 投递（Celery 任务）
       ↓
外部系统接收通知 → 刷新缓存
```

**例外**:
- `AssignNavigation`: 不触发任何事件
- `MenuItemTranslate`: 触发 `translations_created`/`translations_updated`
- `MenuItemMove`: 仅在 sort_order 或 parent_changed 时触发
- `MenuItemMove`: 唯一清除 DataLoader 缓存的 mutation

### 8.3 多站点菜单隔离流程（修正版）

```
HTTP 请求到达
       ↓
1. Django get_current_site(request) 通过域名识别站点
       ↓
SiteSettings 加载
       ↓
2. context_processors.site() 注入站点上下文
3. 预加载 settings.translations（站点设置翻译）
       ↓
GraphQL 菜单查询
       ↓
4. resolve_menu/resolve_menus: 从 DB 查全局菜单数据
   → 包装为 ChannelContext(node=menu, channel_slug=channel)
   → channel 参数默认使用 get_default_channel_slug_or_graphql_error()
       ↓
菜单项解析
       ↓
5. resolve_items → MenuItemsByParentMenuLoader 批量加载（无渠道过滤）
6. resolve_collection → 检查渠道可见性（ChannelQsContext 传递渠道）
7. resolve_page → 检查 is_visible 和 MANAGE_PAGES 权限
8. resolve_category → 直接返回（无过滤）
       ↓
返回菜单数据
```

---

## 9. 发现的问题与不一致性

| 编号 | 问题 | 影响 | 位置 |
|------|------|------|------|
| P1 | `AssignNavigation` 变更后无 Webhook 事件 | 前端/CDN 无法通过事件感知导航分配变更 | `assign_navigation.py` |
| P2 | `Menu.name` 不可翻译 | 多语言站点需为同一导航创建多个 Menu | `menu/models.py` |
| P3 | `MenuItem.category` 无可见性/权限检查 | 未发布分类可通过菜单暴露 | `menu/types.py:126-129` |
| P4 | MenuItemTranslate 不触发菜单缓存失效事件 | 翻译变更后订阅菜单事件的系统不知道刷新 | `translations/mutations/utils.py:157-160` |
| P5 | MenuItemMove 是唯一清除 DataLoader 缓存的 mutation | 其他 mutation 依赖请求级过期，跨请求场景下可能有短暂不一致 | `menu_item_move.py:200` |
| P6 | AppExtension mount 字段已放开为自由字符串 | 第三方可注册 Dashboard 未知的 mount 点，前端需容错处理 | `app/migrations/0037` |

---

## 10. 核心设计模式总结

| 模式 | 应用位置 | 作用 |
|------|---------|------|
| **MPTT 树结构** | `MenuItem` 模型 | 高效存储和查询嵌套菜单层级 |
| **ChannelContext** | Menu/MenuItem 类型封装 | 传递渠道上下文给子字段解析器 |
| **ChannelQsContext** | 菜单列表查询 | 携带渠道信息的 QuerySet |
| **DataLoader** | 菜单数据加载器 | 解决 GraphQL N+1 查询问题 |
| **call_event + on_commit** | 菜单 mutations | 确保事务提交后才广播事件 |
| **Manifest 声明式** | App 扩展 | 插件通过声明式配置注册导航扩展 |
| **PermissionsField** | GraphQL 字段 | 字段级别的权限控制 |
| **Translation 模型** | MenuItemTranslation | 多语言支持的标准实现 |
| **CacheDict (LRU)** | DataLoader 内部 | 内存缓存，容量控制 |
| **枚举→自由字符串演进** | AppExtension.mount | 从受限枚举到开放字符串的架构演进 |

---

## 11. 关键参考文件汇总

| 层级 | 文件路径 | 核心内容 |
|------|---------|---------|
| 数据模型 | `saleor/menu/models.py` | Menu, MenuItem, MenuItemTranslation |
| 数据模型 | `saleor/app/models.py` | AppExtension 扩展点模型 |
| 数据模型 | `saleor/site/models.py` | SiteSettings 菜单关联、SiteSettingsTranslation |
| 迁移 | `saleor/app/migrations/0037_app_extensions_loosen_mount.py` | mount 字段从枚举转为自由字符串 |
| 迁移 | `saleor/app/migrations/0032_appextension_http_target_method_and_more.py` | 完整 mount 枚举列表 |
| 应用安装 | `saleor/app/installation_utils.py` | 插件安装时的扩展注册 |
| 应用安装 | `saleor/app/manifest_schema.py` | Manifest Pydantic 数据结构 |
| 应用安装 | `saleor/app/manifest_validations.py` | Manifest 验证（权限、URL、mount 小写化） |
| 过滤 | `saleor/app/filters.py` | AppExtension mountName/targetName 过滤器 |
| 插件系统 | `saleor/plugins/base_plugin.py` | 插件事件方法声明 (menu_created/updated/deleted 等) |
| 插件系统 | `saleor/plugins/manager.py` | 插件事件广播调度 |
| 插件系统 | `saleor/plugins/webhook/plugin.py` | WebhookPlugin 菜单事件实现与 payload 结构 |
| 事件工具 | `saleor/core/utils/events.py` | call_event + on_commit 机制 |
| Webhook | `saleor/webhook/event_types.py` | 6 种菜单事件及其权限映射 |
| GraphQL | `saleor/graphql/menu/schema.py` | 完整菜单 Queries/Mutations 定义 |
| GraphQL | `saleor/graphql/menu/types.py` | Menu/MenuItem 类型、权限检查 resolver |
| GraphQL | `saleor/graphql/menu/resolvers.py` | 菜单查询解析器 |
| GraphQL | `saleor/graphql/menu/mutations/*.py` | 所有菜单 CRUD mutations |
| GraphQL | `saleor/graphql/menu/dataloaders.py` | 4 个菜单 DataLoader |
| GraphQL | `saleor/graphql/menu/enums.py` | NavigationType 枚举 (MAIN/SECONDARY) |
| GraphQL | `saleor/graphql/menu/filters.py` | Menu/MenuItem 过滤器 |
| 上下文 | `saleor/graphql/core/context.py` | ChannelContext, ChannelQsContext, SaleorContext |
| 上下文 | `saleor/graphql/core/types/context.py` | ChannelContextType, resolve_translation |
| 站点 | `saleor/site/context_processors.py` | 站点上下文处理器 |
| 权限 | `saleor/permission/enums.py` | MenuPermissions, SitePermissions 枚举 |
| 翻译 | `saleor/graphql/translations/mutations/menu_item_translate.py` | 菜单项翻译 Mutation |
| 翻译 | `saleor/graphql/translations/mutations/utils.py` | BaseTranslateMutation, 翻译事件广播 |
| 翻译 | `saleor/graphql/translations/schema.py` | TranslatableKinds.MENU_ITEM, TYPES_TRANSLATIONS_MAP |
| 翻译 | `saleor/graphql/translations/types.py` | MenuItemTranslation, MenuItemTranslatableContent |
| 缓存 | `saleor/core/utils/cache.py` | CacheDict (LRU) 实现 |
