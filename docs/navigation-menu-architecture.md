# Saleor 内容导航与菜单结构扩展实现分析

## 系统分层架构总览

```
┌───────────────────────────────────────────────────────────┐
│                     GraphQL API Layer                     │
│  Types / Schema / Resolvers / Mutations / Permissions    │
├───────────────────────────────────────────────────────────┤
│                  Business Logic Layer                     │
│         Mutations Logic / Validations / Events           │
├───────────────────────────────────────────────────────────┤
│                    Plugin / App Layer                     │
│    AppExtension / Manifest / Webhooks / PluginManager    │
├───────────────────────────────────────────────────────────┤
│                   Context & Cache Layer                  │
│   SiteContext / ChannelContext / CacheKeys / Events      │
├───────────────────────────────────────────────────────────┤
│              Permissions & Localization Layer            │
│     Permission Enums / Translations / i18n Support       │
├───────────────────────────────────────────────────────────┤
│                    Data Model Layer                       │
│   Menu / MenuItem / AppExtension / SiteSettings          │
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
    
    class Meta:
        permissions = ((MenuPermissions.MANAGE_MENUS.codename, "Manage navigation."),)
```

- **核心字段**:
  - `name`: 菜单名称，用于后台识别
  - `slug`: 唯一标识符，支持 Unicode
  - 继承 `ModelWithMetadata` 支持自定义元数据

#### MenuItem 模型 (`saleor/menu/models.py:24-61`)
```python
class MenuItem(ModelWithMetadata, MPTTModel, SortableModel):
    menu = models.ForeignKey(Menu, related_name="items", on_delete=models.CASCADE)
    name = models.CharField(max_length=128)
    parent = models.ForeignKey("self", null=True, blank=True, related_name="children", on_delete=models.CASCADE)
    url = models.URLField(max_length=256, blank=True, null=True)
    category = models.ForeignKey(Category, blank=True, null=True, on_delete=models.CASCADE)
    collection = models.ForeignKey(Collection, blank=True, null=True, on_delete=models.CASCADE)
    page = models.ForeignKey(Page, blank=True, null=True, on_delete=models.CASCADE)
```

- **树结构**: 使用 `MPTTModel` 实现嵌套层级结构
- **链接类型**（互斥）:
  1. `url`: 外部或自定义 URL
  2. `category`: 关联商品分类
  3. `collection`: 关联商品集合
  4. `page`: 关联 CMS 页面
- **排序**: 通过 `SortableModel` 支持同级排序

#### MenuItemTranslation 模型 (`saleor/menu/models.py:63-86`)
```python
class MenuItemTranslation(Translation):
    menu_item = models.ForeignKey(MenuItem, related_name="translations", on_delete=models.CASCADE)
    name = models.CharField(max_length=128)
    
    class Meta:
        unique_together = (("language_code", "menu_item"),)
```

- 实现菜单项名称的多语言翻译
- 基于 `language_code` 和 `menu_item` 的联合唯一约束

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
  - `mount`: 扩展挂载点（如 `PRODUCT_OVERVIEW_CREATE`, `NAVIGATION_SIDEBAR`）
  - `target`: 打开方式（`popup`, `new_tab`, `app_page`, `widget`）
  - `permissions`: 控制谁可以看到和使用该扩展
  - `settings`: 扩展的自定义配置

### 1.3 站点设置关联

**文件位置**: `saleor/site/models.py:38-47`

```python
class SiteSettings(ModelWithMetadata):
    site = models.OneToOneField(Site, related_name="settings", on_delete=models.CASCADE)
    top_menu = models.ForeignKey("menu.Menu", on_delete=models.SET_NULL, related_name="+", blank=True, null=True)
    bottom_menu = models.ForeignKey("menu.Menu", on_delete=models.SET_NULL, related_name="+", blank=True, null=True)
```

- 每个站点通过 `top_menu` 和 `bottom_menu` 关联两个导航菜单
- 实现站点级别的菜单隔离

---

## 2. 插件与应用扩展层 (Plugin & App Extension Layer)

### 2.1 插件导航注册机制

#### Manifest 声明式扩展

**文件位置**: `saleor/app/manifest_schema.py:53-62`

```python
class ManifestExtensionSchema(BaseModel):
    label: str
    url: str
    mount: str
    target: str = DEFAULT_APP_TARGET
    permissions: list[str] = []
    options: dict = {}
```

**示例 Manifest** (`saleor/app/app_manifest_sample.json`):
```json
{
  "extensions": [
    {
      "label": "Create with Sample app",
      "mount": "PRODUCT_OVERVIEW_CREATE",
      "target": "POPUP",
      "permissions": ["MANAGE_PRODUCTS"],
      "url": "https://example.com/extension/"
    }
  ]
}
```

#### 挂载点 (Mount Points)

根据代码中的实际使用，常见挂载点包括:
- `PRODUCT_OVERVIEW_CREATE` - 商品创建页面
- `PRODUCT_OVERVIEW_MORE_ACTIONS` - 商品更多操作菜单
- `PRODUCT_DETAILS_WIDGETS` - 商品详情页小部件
- `NAVIGATION_SIDEBAR` - 后台侧边栏导航

#### 应用安装时的扩展注册

**文件位置**: `saleor/app/installation_utils.py:245-277`

```python
for extension_data in manifest_data.get("extensions", []):
    options = extension_data.get("options", {})
    extension = AppExtension.objects.create(
        app=app,
        label=extension_data.get("label"),
        url=extension_data.get("url"),
        mount=extension_data.get("mount"),
        target=extension_data.get("target", DEFAULT_APP_TARGET),
        http_target_method=http_target_method,
        settings=extension_data.get("options", {}),
    )
    extension.permissions.set(extension_data.get("permissions", []))
```

### 2.2 插件事件系统

**文件位置**: `saleor/plugins/base_plugin.py:940,967`

```python
class BasePlugin:
    menu_updated: Callable[["Menu", None], None]
    menu_item_updated: Callable[["MenuItem", None], None]
    menu_item_created: Callable[["MenuItem", None], None]
```

**插件管理器调度** (`saleor/plugins/manager.py:1931,1965`):
```python
def menu_updated(self, menu: "Menu"):
    default_value = None
    return self.__run_method_on_plugins(
        "menu_updated",
        default_value,
        menu,
        channel_slug=None,
    )

def menu_item_updated(self, menu_item: "MenuItem"):
    default_value = None
    return self.__run_method_on_plugins(
        "menu_item_updated",
        default_value,
        menu_item,
        channel_slug=None,
    )
```

---

## 3. 站点上下文与隔离机制 (Site Context & Isolation)

### 3.1 多站点隔离

#### 上下文处理器

**文件位置**: `saleor/site/context_processors.py:11-16`

```python
def site(request: "HttpRequest") -> dict:
    site = get_current_site(request)
    if isinstance(site, Site):
        prefetch_related_objects([site], "settings__translations")
    return {"site": site}
```

- 根据请求自动识别当前站点
- 预加载站点设置和翻译数据

#### 导航分配机制

**文件位置**: `saleor/graphql/menu/mutations/assign_navigation.py:14-48`

```python
class AssignNavigation(BaseMutation):
    class Arguments:
        menu = graphene.ID(description="ID of the menu.")
        navigation_type = NavigationType(required=True)

    @classmethod
    def perform_mutation(cls, _root, info: ResolveInfo, /, *, menu=None, navigation_type):
        site = get_site_promise(info.context).get()
        if menu is not None:
            menu = cls.get_node_or_error(info, menu, field="menu", only_type=Menu)

        if navigation_type == NavigationType.MAIN:
            site.settings.top_menu = menu
            site.settings.save(update_fields=["top_menu"])
        elif navigation_type == NavigationType.SECONDARY:
            site.settings.bottom_menu = menu
            site.settings.save(update_fields=["bottom_menu"])
```

**导航类型枚举** (`saleor/graphql/menu/enums.py:4-14`):
```python
class NavigationType(graphene.Enum):
    MAIN = "main"      # 主导航（顶部菜单）
    SECONDARY = "secondary"  # 次导航（底部菜单）
```

### 3.2 渠道上下文封装

**文件位置**: `saleor/graphql/menu/types.py:35-61`

```python
class Menu(ChannelContextType[models.Menu]):
    @staticmethod
    def resolve_items(root: ChannelContext[models.Menu], info: ResolveInfo):
        menu_items = MenuItemsByParentMenuLoader(info.context).load(root.node.id)
        return menu_items.then(
            lambda menu_items: [
                ChannelContext(node=menu_item, channel_slug=root.channel_slug)
                for menu_item in menu_items
            ]
        )
```

- 使用 `ChannelContext` 封装菜单和菜单项
- 确保在多渠道环境下正确的数据隔离

---

## 4. 菜单变更缓存刷新机制 (Cache Invalidation)

### 4.1 事件驱动的缓存刷新

菜单变更通过 Webhook 事件通知机制实现缓存刷新，主要在 Mutation 的 `post_save_action` 中触发:

#### 菜单更新事件

**文件位置**: `saleor/graphql/menu/mutations/menu_update.py:41-44`

```python
@classmethod
def post_save_action(cls, info: ResolveInfo, instance, cleaned_input):
    manager = get_plugin_manager_promise(info.context).get()
    cls.call_event(manager.menu_updated, instance)
```

#### 菜单项更新事件

**文件位置**: `saleor/graphql/menu/mutations/menu_item_update.py:48-51`

```python
@classmethod
def post_save_action(cls, info: ResolveInfo, instance, cleaned_input):
    manager = get_plugin_manager_promise(info.context).get()
    cls.call_event(manager.menu_item_updated, instance)
```

#### 菜单项创建事件

**文件位置**: `saleor/graphql/menu/mutations/menu_item_create.py:67-70`

```python
@classmethod
def post_save_action(cls, info: ResolveInfo, instance, cleaned_input):
    manager = get_plugin_manager_promise(info.context).get()
    cls.call_event(manager.menu_item_created, instance)
```

### 4.2 Webhook 事件类型

**文件位置**: `saleor/graphql/menu/mutations/menu_update.py:34-39`

```python
webhook_events_info = [
    WebhookEventInfo(
        type=WebhookEventAsyncType.MENU_UPDATED,
        description="A menu was updated.",
    ),
]
```

相关 Webhook 事件:
- `MENU_UPDATED` - 菜单更新
- `MENU_ITEM_UPDATED` - 菜单项更新
- `MENU_ITEM_CREATED` - 菜单项创建
- `MENU_ITEM_DELETED` - 菜单项删除

### 4.3 内存缓存辅助

**文件位置**: `saleor/core/utils/cache.py:4-20`

```python
class CacheDict(collections.OrderedDict):
    def __init__(self, capacity: int):
        self.capacity = capacity
        super().__init__()

    def __getitem__(self, key):
        value = super().__getitem__(key)
        super().move_to_end(key)
        return value

    def __setitem__(self, key, value):
        super().__setitem__(key, value)
        super().move_to_end(key)
        while len(self) > self.capacity:
            surplus = next(iter(self))
            super().__delitem__(surplus)
```

- LRU 缓存实现，用于数据加载器等场景
- 基于容量的自动淘汰机制

---

## 5. 权限系统关联 (Permission Integration)

### 5.1 菜单管理权限

**文件位置**: `saleor/permission/enums.py:43-44`

```python
class MenuPermissions(BasePermissionEnum):
    MANAGE_MENUS = "menu.manage_menus"
```

**站点设置权限** (`saleor/permission/enums.py:83-85`):
```python
class SitePermissions(BasePermissionEnum):
    MANAGE_SETTINGS = "site.manage_settings"
    MANAGE_TRANSLATIONS = "site.manage_translations"
```

### 5.2 Mutation 权限控制

**文件位置**: `saleor/graphql/menu/mutations/assign_navigation.py:27`

```python
class Meta:
    permissions = (MenuPermissions.MANAGE_MENUS, SitePermissions.MANAGE_SETTINGS)
```

**菜单更新权限** (`saleor/graphql/menu/mutations/menu_update.py:31`):
```python
class Meta:
    permissions = (MenuPermissions.MANAGE_MENUS,)
```

### 5.3 菜单项解析时的权限检查

**文件位置**: `saleor/graphql/menu/types.py:142-198`

```python
@staticmethod
def resolve_collection(root: ChannelContext[models.MenuItem], info: ResolveInfo):
    requestor = get_user_or_app_from_context(info.context)
    has_required_permission = has_one_of_permissions(
        requestor, ALL_PRODUCTS_PERMISSIONS
    )
    if has_required_permission:
        return CollectionByIdLoader(info.context).load(root.node.collection_id)
    
    # 无权限时检查渠道可用性和集合可见性
    channel_slug = str(root.channel_slug)
    channel = ChannelBySlugLoader(info.context).load(channel_slug)
    # ... 后续的可见性检查逻辑
```

**页面权限检查** (`saleor/graphql/menu/types.py:219-238`):
```python
@staticmethod
def resolve_page(root: ChannelContext[models.MenuItem], info: ResolveInfo):
    if root.node.page_id:
        requestor = get_user_or_app_from_context(info.context)
        requestor_has_access_to_all = (
            requestor
            and requestor.is_active
            and requestor.has_perm(PagePermissions.MANAGE_PAGES)
        )

        def resolve_page_with_channel(page):
            if requestor_has_access_to_all or page.is_visible:
                return ChannelContext(node=page, channel_slug=root.channel_slug)
            return None
```

### 5.4 AppExtension 权限控制

**文件位置**: `saleor/graphql/app/types.py:233-238`

```python
@staticmethod
def resolve_permissions(root: models.AppExtension, _info: ResolveInfo):
    permissions = root.permissions.prefetch_related("content_type").order_by(
        "codename"
    )
    return format_permissions_for_display(permissions)
```

- 每个 AppExtension 可以定义自己的权限要求
- 只有拥有相应权限的用户才能看到和使用该扩展

---

## 6. 本地化与多语言支持 (Localization)

### 6.1 菜单项翻译 Mutation

**文件位置**: `saleor/graphql/translations/mutations/menu_item_translate.py:13-42`

```python
class MenuItemTranslate(BaseTranslateMutation):
    class Arguments:
        id = graphene.ID(required=True)
        language_code = graphene.Argument(LanguageCodeEnum, required=True)
        input = NameTranslationInput(required=True)

    class Meta:
        permissions = (SitePermissions.MANAGE_TRANSLATIONS,)

    @classmethod
    def perform_mutation(cls, root, info: ResolveInfo, /, *, id, input, language_code):
        response = super().perform_mutation(
            root, info, id=id, input=input, language_code=language_code
        )
        instance = ChannelContext(node=response.menuItem, channel_slug=None)
        return cls(**{cls._meta.return_field_name: instance})
```

### 6.2 GraphQL 翻译字段

**文件位置**: `saleor/graphql/menu/types.py:110-114`

```python
translation = TranslationField(
    MenuItemTranslation,
    type_name="menu item",
    resolver=ChannelContextType.resolve_translation,
)
```

### 6.3 站点设置翻译

**文件位置**: `saleor/site/models.py:169-190`

```python
class SiteSettingsTranslation(Translation):
    site_settings = models.ForeignKey(SiteSettings, related_name="translations", on_delete=models.CASCADE)
    header_text = models.CharField(max_length=200, blank=True)
    description = models.CharField(max_length=500, blank=True)
```

---

## 7. GraphQL API 层 (GraphQL API Layer)

### 7.1 菜单查询解析器

**文件位置**: `saleor/graphql/menu/resolvers.py:12-41`

```python
def resolve_menu(info, channel, menu_id=None, name=None, slug=None):
    validate_one_of_args_is_in_query("id", menu_id, "name", name, "slug", slug)
    menu = None
    if menu_id:
        _, id = from_global_id_or_error(menu_id, Menu)
        menu = models.Menu.objects.using(
            get_database_connection_name(info.context)
        ).filter(id=id).first()
    # ... name 和 slug 查询类似
    return ChannelContext(node=menu, channel_slug=channel) if menu else None

def resolve_menus(info, channel):
    return ChannelQsContext(
        qs=models.Menu.objects.using(
            get_database_connection_name(info.context)
        ).all(),
        channel_slug=channel,
    )
```

### 7.2 菜单项数据加载器

**文件位置**: `saleor/graphql/menu/dataloaders.py`

- `MenuByIdLoader`: 按 ID 加载菜单
- `MenuItemByIdLoader`: 按 ID 加载菜单项
- `MenuItemChildrenLoader`: 加载菜单项的子项
- `MenuItemsByParentMenuLoader`: 加载菜单的所有菜单项

---

## 8. 关键交互流程

### 8.1 插件注册导航项完整流程

```
App Manifest 声明
       ↓
1. 定义 extensions 数组，指定 mount/label/url/permissions
       ↓
App Installation 安装
       ↓
2. 调用 install_app() 解析 manifest
3. 验证 mount 点和 target 格式
4. 验证 extension URL 格式（相对路径需 appUrl）
5. 验证 extension 权限在 App 权限范围内
6. 创建 AppExtension 记录并关联权限
       ↓
Dashboard 前端渲染
       ↓
7. 查询 appExtensions 按 mount 点筛选
8. 检查当前用户是否拥有 extension 所需权限
9. 渲染导航项到指定挂载点
```

### 8.2 菜单变更刷新流程

```
菜单 Mutation 执行
       ↓
1. MenuUpdate / MenuItemUpdate / AssignNavigation
       ↓
post_save_action 触发
       ↓
2. 获取 PluginManager
3. 调用 call_event(manager.menu_updated, instance)
       ↓
PluginManager 广播
       ↓
4. 遍历所有激活插件调用 menu_updated 方法
5. 触发 WebhookEventAsyncType.MENU_UPDATED 事件
       ↓
外部系统处理
       ↓
6. 订阅了菜单事件的 Webhook 接收通知
7. 前端/缓存系统收到通知后刷新本地缓存
```

### 8.3 多站点菜单隔离流程

```
HTTP 请求到达
       ↓
1. get_current_site(request) 识别站点
       ↓
SiteSettings 加载
       ↓
2. 获取站点关联的 top_menu / bottom_menu
       ↓
Context Processor 注入
       ↓
3. site 上下文注入模板/请求
       ↓
菜单查询
       ↓
4. 通过站点设置的菜单 ID 查询具体菜单项
5. 按 channel_slug 过滤可见的关联内容
6. 应用权限检查过滤不可见项
       ↓
返回菜单数据
```

---

## 9. 核心设计模式总结

| 模式 | 应用位置 | 作用 |
|------|---------|------|
| **MPTT 树结构** | `MenuItem` 模型 | 高效存储和查询嵌套菜单层级 |
| **ChannelContext** | 菜单类型封装 | 实现多渠道数据隔离 |
| **DataLoader** | 菜单数据加载器 | 解决 GraphQL N+1 查询问题 |
| **Webhook 事件** | 菜单 mutations | 实现事件驱动的缓存失效 |
| **Manifest 声明式** | App 扩展 | 插件通过声明式配置注册导航 |
| **PermissionField** | GraphQL 字段 | 字段级别的权限控制 |
| **Translation 模型** | 菜单项翻译 | 多语言支持的标准实现 |
| **LRU Cache** | `CacheDict` | 内存缓存，容量控制 |

---

## 10. 关键参考文件汇总

| 层级 | 文件路径 | 核心内容 |
|------|---------|---------|
| 数据模型 | `saleor/menu/models.py` | Menu, MenuItem, MenuItemTranslation |
| 数据模型 | `saleor/app/models.py` | AppExtension 扩展点模型 |
| 数据模型 | `saleor/site/models.py` | SiteSettings 菜单关联 |
| 应用安装 | `saleor/app/installation_utils.py` | 插件安装时的扩展注册 |
| 应用安装 | `saleor/app/manifest_schema.py` | Manifest 数据结构定义 |
| 应用安装 | `saleor/app/manifest_validations.py` | Manifest 验证逻辑 |
| 插件系统 | `saleor/plugins/base_plugin.py` | 插件事件方法声明 |
| 插件系统 | `saleor/plugins/manager.py` | 插件事件广播调度 |
| GraphQL | `saleor/graphql/menu/types.py` | 菜单 GraphQL 类型定义 |
| GraphQL | `saleor/graphql/menu/resolvers.py` | 菜单查询解析器 |
| GraphQL | `saleor/graphql/menu/mutations/*.py` | 菜单 CRUD mutations |
| 站点上下文 | `saleor/site/context_processors.py` | 站点上下文处理器 |
| 权限系统 | `saleor/permission/enums.py` | 权限枚举定义 |
| 翻译系统 | `saleor/graphql/translations/mutations/menu_item_translate.py` | 菜单项翻译 |
| 缓存工具 | `saleor/core/utils/cache.py` | LRU 缓存实现 |
