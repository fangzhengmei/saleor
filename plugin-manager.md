# Saleor Plugin Manager 实现分析

## 1. 概述

Saleor 的插件系统采用 **责任链模式** + **延迟加载** 的设计，核心类为 `PluginsManager`（位于 `saleor/plugins/manager.py`）。插件用于扩展 Saleor 的核心功能，包括税务计算、支付网关、邮件通知、Webhook 分发、第三方认证等。

本文档重点分析三个容易产生理解偏差的核心问题：
1. 外部插件注入应用列表的完整注册流程
2. 无 channel 情况下插件集合的组成及调用顺序
3. Manager 实例的生命周期与读写库路径的差异

---

## 2. 外部插件注入应用列表的注册流程

### 2.1 完整注册流程详解 (`saleor/settings.py:898-922`)

外部插件通过 Python `entry_points` 机制实现自动发现和注入，完整流程如下：

```python
# saleor/settings.py:898-922

# 1. 内置插件列表（硬编码在代码中）
BUILTIN_PLUGINS = [
    "saleor.plugins.avatax.plugin.DeprecatedAvataxPlugin",
    "saleor.plugins.webhook.plugin.WebhookPlugin",
    "saleor.payment.gateways.stripe.plugin.StripeGatewayPlugin",
    "saleor.plugins.user_email.plugin.UserEmailPlugin",
    "saleor.plugins.openid_connect.plugin.OpenIDConnectPlugin",
    # ... 更多内置插件
]

# 2. 通过 entry_points 动态发现外部插件
EXTERNAL_PLUGINS = []
installed_plugins = importlib.metadata.entry_points(group="saleor.plugins")

for entry_point in installed_plugins:
    # 2.1 构建插件类的完整路径
    plugin_path = f"{entry_point.module}.{entry_point.attr}"
    
    # 2.2 去重检查：避免重复注册
    if plugin_path not in BUILTIN_PLUGINS and plugin_path not in EXTERNAL_PLUGINS:
        
        # 2.3 将插件所属的 Django App 注入 INSTALLED_APPS
        # entry_point.name 对应插件包的 Django App 名称
        if entry_point.name not in INSTALLED_APPS:
            INSTALLED_APPS.append(entry_point.name)
        
        # 2.4 将插件类路径添加到外部插件列表
        EXTERNAL_PLUGINS.append(plugin_path)

# 3. 最终插件列表（顺序决定调用优先级！）
PLUGINS: list[str] = BUILTIN_PLUGINS + EXTERNAL_PLUGINS
```

### 2.2 Entry Points 配置示例与属性详解

外部插件包需要在 `pyproject.toml` 或 `setup.py` 中声明 entry points：

```toml
# pyproject.toml 示例
[project.entry-points."saleor.plugins"]
my_custom_plugin = "my_custom_plugin.plugin:MyCustomPlugin"
```

#### Entry Point 各属性的含义

当 Python 解析上述 entry point 配置时，会生成一个 `EntryPoint` 对象，其属性如下：

| 属性 | 值 | 来源 | 用途 |
|------|----|------|------|
| `entry_point.name` | `"my_custom_plugin"` | entry point 左侧的键名 | 注入到 `INSTALLED_APPS`，作为 Django App 名称 |
| `entry_point.value` | `"my_custom_plugin.plugin:MyCustomPlugin"` | entry point 右侧的完整值 | 原始配置字符串 |
| `entry_point.module` | `"my_custom_plugin.plugin"` | 冒号 `:` 左侧部分 | 插件类所在的模块路径 |
| `entry_point.attr` | `"MyCustomPlugin"` | 冒号 `:` 右侧部分 | 插件类名 |
| **`plugin_path` (代码构建)** | **`"my_custom_plugin.plugin.MyCustomPlugin"`** | **`module + "." + attr`** | **传给 `import_string` 用于导入插件类** |

#### 关键拼接规则（代码实现）

```python
# saleor/settings.py:916
# 注意：使用点号 "." 连接，而非冒号 ":"
plugin_path = f"{entry_point.module}.{entry_point.attr}"
```

**为什么不用 `entry_point.value` 直接作为 plugin_path？**

因为 `import_string`（Django 的工具函数）只接受 **点号分隔** 的导入路径（如 `"module.submodule.ClassName"`），而 entry point value 使用冒号分隔（如 `"module.submodule:ClassName"`）。因此必须通过 `module` 和 `attr` 重新拼接。

### 2.3 注入到 INSTALLED_APPS 的意义

将 `entry_point.name` 添加到 Django 的 `INSTALLED_APPS` 有以下作用：

1. **Django App 初始化**：触发插件包的 `apps.py` 中的 `AppConfig.ready()` 方法
2. **模型发现**：Django 能够发现插件包中定义的模型
3. **管理命令发现**：插件包中定义的 `management/commands` 可被发现
4. **迁移发现**：插件包的 `migrations` 目录可被 Django 迁移系统识别

**注意**：`entry_point.name` 必须与插件包的 Django App 名称一致。插件包的 `apps.py` 通常如下定义：

```python
# my_custom_plugin/apps.py
from django.apps import AppConfig

class MyCustomPluginConfig(AppConfig):
    name = "my_custom_plugin"  # 必须与 entry_point.name 一致
    verbose_name = "My Custom Plugin"
```

### 2.4 注册流程时序图

```
应用启动
    ↓
加载 settings.py
    ↓
INSTALLED_APPS 初始化（内置 Django App）
    ↓
执行 PLUGINS 注册逻辑
    ↓
遍历 entry_points(group="saleor.plugins")
    ├─> 构建 plugin_path = module.attr
    ├─> 去重检查
    ├─> 注入 entry_point.name 到 INSTALLED_APPS
    └─> 添加 plugin_path 到 EXTERNAL_PLUGINS
    ↓
PLUGINS = BUILTIN_PLUGINS + EXTERNAL_PLUGINS
    ↓
Django 完成 App 初始化（调用各 App 的 ready()）
```

---

## 3. 无 channel 情况下插件集合的组成及调用顺序

这是最容易产生理解偏差的部分。**有 channel 和无 channel 场景下插件集合的组成逻辑完全不同。**

### 3.1 有 channel 场景（明确指定 channel_slug）

```python
# saleor/plugins/manager.py:2488-2508
def get_plugins(self, channel_slug: str | None = None, ...):
    if channel_slug is not None:
        # 触发该 channel 的插件加载
        self._ensure_channel_plugins_loaded(channel_slug)
        plugins = self.plugins_per_channel[channel_slug]
    ...
```

`plugins_per_channel[channel_slug]` 的组成顺序（`saleor/plugins/manager.py:171-199`）：

```
1. 遍历 PLUGINS 配置
   └─> 对 CONFIGURATION_PER_CHANNEL=True 的插件
       └─> 创建该 channel 专属实例 → 按 PLUGINS 顺序加入列表

2. 追加全局插件（CONFIGURATION_PER_CHANNEL=False 的插件）
   └─> 按 PLUGINS 顺序追加在列表末尾

最终顺序：[通道插件A, 通道插件B, ..., 全局插件X, 全局插件Y, ...]
```

### 3.2 无 channel 场景（channel_slug=None）

这是理解的关键！无 channel 时返回的是 `all_plugins` 列表：

```python
# saleor/plugins/manager.py:2488-2508
def get_plugins(self, channel_slug: str | None = None, ...):
    if channel_slug is not None:
        ...
    else:
        # 只加载全局插件，不加载任何通道插件！
        self._ensure_channel_plugins_loaded(None)
        plugins = self.all_plugins
```

**关键点**：`_ensure_channel_plugins_loaded(None)` **只加载全局插件**，不加载任何通道插件。

让我们看 `_ensure_channel_plugins_loaded(None)` 的实现（`saleor/plugins/manager.py:154-169`）：

```python
def _ensure_channel_plugins_loaded(self, channel_slug: str | None, ...):
    # 当 channel_slug is None 时，只执行这段代码：
    if channel_slug is None and not self.loaded_global:
        global_db_config = self._get_db_plugin_configs(None)
        
        for plugin_path in self.plugins:  # 按 PLUGINS 顺序遍历
            PluginClass = import_string(plugin_path)
            # 只加载 CONFIGURATION_PER_CHANNEL=False 的插件（全局插件）
            if not getattr(PluginClass, "CONFIGURATION_PER_CHANNEL", False):
                plugin = self._load_plugin(PluginClass, global_db_config, ...)
                self.global_plugins.append(plugin)
                self.all_plugins.append(plugin)  # 按 PLUGINS 顺序添加
        self.loaded_global = True
    
    # 注意：channel_slug is None 时，下面这段代码不会执行！
    if channel_slug is not None and channel_slug not in self.loaded_channels:
        ...  # 加载通道插件的逻辑
```

### 3.3 all_plugins 的填充顺序（关键！时序敏感）

`all_plugins` 是一个扁平列表，其填充顺序**完全取决于加载触发的历史时序**。这是最容易产生理解偏差的部分。

**先明确代码执行时序**（`saleor/plugins/manager.py:171-199`）：

```python
if channel_slug is not None and channel_slug not in self.loaded_channels:
    # ⚠️ 第一步：先加载通道插件，先写入 all_plugins
    for plugin_path in self.plugins:
        if getattr(PluginClass, "CONFIGURATION_PER_CHANNEL", False):
            plugin = self._load_plugin(...)
            self.plugins_per_channel[channel_slug].append(plugin)
            self.all_plugins.append(plugin)  # 先写通道插件！
    
    # ⚠️ 第二步：才加载全局插件，后写入 all_plugins
    self._ensure_channel_plugins_loaded(None)
    self.plugins_per_channel[channel_slug].extend(self.global_plugins)
```

**重要对比**：
- `plugins_per_channel[channel_slug]` 的顺序是**确定的**：[通道插件按 PLUGINS 顺序] + [全局插件按 PLUGINS 顺序]
- `all_plugins` 的顺序是**不确定的**：取决于哪个 channel 先被加载

---

#### 基础假设

为了清晰对比，假设 `PLUGINS` 配置如下（按顺序）：

| 序号 | 插件类 | CONFIGURATION_PER_CHANNEL | 类型 |
|------|--------|--------------------------|------|
| 1 | AvataxPlugin | True | 通道插件 |
| 2 | WebhookPlugin | True | 通道插件 |
| 3 | StripePlugin | True | 通道插件 |
| 4 | UserEmailPlugin | False | 全局插件 |
| 5 | AdminEmailPlugin | False | 全局插件 |
| 6 | OpenIDPlugin | False | 全局插件 |

---

#### 场景 A：先触发无 channel 的调用，再触发有 channel 的调用

```
1. 调用 get_plugins(channel_slug=None)
    ↓
_ensure_channel_plugins_loaded(None) 只加载全局插件
    ↓
遍历 PLUGINS，只处理 CONFIGURATION_PER_CHANNEL=False 的
    ↓
all_plugins = [UserEmail, AdminEmail, OpenID]  ✅ 按 PLUGINS 顺序
    ↓
2. 后续调用 get_plugins(channel_slug="usd")
    ↓
_ensure_channel_plugins_loaded("usd")
    ├─> 第一步：加载通道插件（Avatax, Webhook, Stripe）
    │   └─> all_plugins 追加 → [UserEmail, AdminEmail, OpenID, Avatax(usd), Webhook(usd), Stripe(usd)]
    └─> 第二步：调用 _ensure_channel_plugins_loaded(None)，但 loaded_global=True，跳过
```

**all_plugins 最终顺序**：`[全局×3, usd通道×3]`

**无 channel 调用顺序**：全局插件在前，usd 通道插件在后，各自按 PLUGINS 顺序。

---

#### 场景 B：先触发有 channel 的调用，再触发无 channel 的调用

**⚠️ 这是与场景 A 顺序完全相反的情况！**

```
1. 调用 get_plugins(channel_slug="usd")
    ↓
_ensure_channel_plugins_loaded("usd")
    ├─> 第一步：先加载通道插件（Avatax, Webhook, Stripe）
    │   └─> all_plugins = [Avatax(usd), Webhook(usd), Stripe(usd)]  ⚠️ 通道插件在前！
    └─> 第二步：才调用 _ensure_channel_plugins_loaded(None) 加载全局插件
        └─> all_plugins 追加 → [Avatax(usd), Webhook(usd), Stripe(usd), UserEmail, AdminEmail, OpenID]
    ↓
2. 调用 get_plugins(channel_slug=None)
    ↓
返回 all_plugins = [Avatax(usd), Webhook(usd), Stripe(usd), UserEmail, AdminEmail, OpenID]
```

**all_plugins 最终顺序**：`[usd通道×3, 全局×3]`

**无 channel 调用顺序**：usd 通道插件在前，全局插件在后，各自按 PLUGINS 顺序。

---

#### 场景 C：通过 get_all_plugins() 触发（按数据库 Channel 顺序加载）

假设数据库中有两个 Channel，按创建顺序：`eur` → `usd`

```
调用 get_all_plugins()
    ↓
遍历 channels：先 eur，后 usd
    ↓
加载 eur:
    ├─> 第一步：all_plugins = [Avatax(eur), Webhook(eur), Stripe(eur)]
    └─> 第二步：all_plugins = [Avatax(eur), Webhook(eur), Stripe(eur), UserEmail, AdminEmail, OpenID]
    ↓
加载 usd:
    ├─> 第一步：all_plugins 追加 → [..., Avatax(usd), Webhook(usd), Stripe(usd)]
    └─> 第二步：loaded_global=True，跳过
```

**all_plugins 最终顺序**：`[eur通道×3, 全局×3, usd通道×3]`

**无 channel 调用顺序**：eur 通道插件 → 全局插件 → usd 通道插件，每类内部按 PLUGINS 顺序。

---

#### 场景 D：先调用 eur，再调用 usd，最后调用 None

```
1. 调用 get_plugins(channel_slug="eur")
   → all_plugins = [Avatax(eur), Webhook(eur), Stripe(eur), UserEmail, AdminEmail, OpenID]

2. 调用 get_plugins(channel_slug="usd")
   → all_plugins = [Avatax(eur), Webhook(eur), Stripe(eur), UserEmail, AdminEmail, OpenID, Avatax(usd), Webhook(usd), Stripe(usd)]

3. 调用 get_plugins(channel_slug=None)
   → 返回上述 all_plugins
```

**all_plugins 最终顺序**：`[eur通道×3, 全局×3, usd通道×3]`

---

### 3.4 无 channel 场景的调用总结对比表

| 触发时序 | all_plugins 顺序 | get_plugins(None) 返回顺序 |
|---------|----------------|--------------------------|
| 先 None 后 usd | `[全局×3, usd通道×3]` | `[全局×3, usd通道×3]` |
| 先 usd 后 None | `[usd通道×3, 全局×3]` | `[usd通道×3, 全局×3]` |
| get_all_plugins() | `[eur通道×3, 全局×3, usd通道×3]` | `[eur通道×3, 全局×3, usd通道×3]` |
| 先 eur 再 usd 再 None | `[eur通道×3, 全局×3, usd通道×3]` | `[eur通道×3, 全局×3, usd通道×3]` |

### 3.5 关键结论

1. **`plugins_per_channel[channel_slug]` 顺序确定**：无论加载历史如何，始终是 `[通道插件按 PLUGINS 顺序] + [全局插件按 PLUGINS 顺序]`

2. **`all_plugins` 顺序不确定**：完全取决于加载触发的历史时序，可能出现：
   - 全局插件在前（先调用无 channel）
   - 通道插件在前（先调用有 channel）
   - 多个通道插件交替出现（按 Channel 加载顺序）

3. **`get_plugins(channel_slug=None)` 返回 `all_plugins`**：因此其调用顺序也是不确定的，取决于加载历史

4. **设计意图**：`all_plugins` 主要用于管理后台查询所有插件配置，而非用于需要确定顺序的业务逻辑。业务逻辑应始终明确指定 `channel_slug`。

### 3.6 无 channel 场景的使用场景

无 channel 的插件调用主要用于：

1. **全局事件**：如 `customer_created`、`product_created` 等不依赖特定 channel 的事件
2. **跨通道操作**：如 `order_bulk_created` 批量订单创建
3. **全局认证**：如 `authenticate_user`、`external_obtain_access_tokens`
4. **管理后台查询**：查询所有插件的配置列表

---

## 4. Manager 实例的生命周期与读写库路径的差异

### 4.1 Manager 实例的创建入口 (`saleor/plugins/manager.py:2818-2826`)

```python
def get_plugins_manager(
    allow_replica: bool,
    requestor_getter: Callable[[], "Requestor"] | None = None,
) -> PluginsManager:
    with tracer.start_as_current_span("get_plugins_manager"):
        if allow_replica:
            return PluginsManager(settings.PLUGINS, requestor_getter, allow_replica)
        with allow_writer():
            return PluginsManager(settings.PLUGINS, requestor_getter, allow_replica)
```

### 4.2 读写库路径的选择

`PluginsManager` 通过 `_allow_replica` 参数决定使用哪个数据库连接：

```python
# saleor/plugins/manager.py:96-102
@property
def database(self):
    return (
        settings.DATABASE_CONNECTION_REPLICA_NAME  # "replica"
        if self._allow_replica
        else settings.DATABASE_CONNECTION_DEFAULT_NAME  # "default"
    )
```

数据库配置（`saleor/settings.py:123-149`）：

```python
DATABASE_CONNECTION_DEFAULT_NAME = "default"
DATABASE_CONNECTION_REPLICA_NAME = "replica"

DATABASES = {
    DATABASE_CONNECTION_DEFAULT_NAME: dj_database_url.config(
        env="DATABASE_URL",  # 主库连接
        default="postgres://saleor:saleor@localhost:5432/saleor",
    ),
    DATABASE_CONNECTION_REPLICA_NAME: dj_database_url.config(
        env="DATABASE_URL_REPLICA",  # 从库连接，默认与主库相同
        default="postgres://saleor:saleor@localhost:5432/saleor",
        test_options={"MIRROR": DATABASE_CONNECTION_DEFAULT_NAME},
    ),
}
```

### 4.3 allow_writer() 的作用

当 `allow_replica=False` 时，必须使用 `allow_writer()` 上下文管理器：

```python
# saleor/plugins/manager.py:2825
with allow_writer():
    return PluginsManager(settings.PLUGINS, requestor_getter, allow_replica)
```

`allow_writer()` 是 Saleor 的数据库路由保护机制，确保：
- 写操作只能在主库（"default"）上执行
- 防止在从库（"replica"）上执行写操作导致数据不一致

### 4.4 Manager 实例的生命周期

#### 4.4.1 GraphQL 请求上下文（最常见）

通过 DataLoader 实现同一请求内的 Manager 实例复用（`saleor/graphql/plugins/dataloaders.py:31-60`）：

```python
class PluginManagerByRequestorDataloader(DataLoader[Requestor, PluginsManager]):
    context_key = "plugin_manager_by_requestor"

    def batch_load(self, keys):
        allow_replica = getattr(self.context, "allow_replica", True)
        return [get_plugins_manager(allow_replica, lambda: user) for user in keys]

class AnonymousPluginManagerLoader(DataLoader[None, PluginsManager]):
    context_key = "anonymous_plugin_manager"

    def batch_load(self, keys):
        allow_replica = getattr(self.context, "allow_replica", True)
        return [get_plugins_manager(allow_replica, None) for key in keys]

# 获取 manager 的函数
def get_plugin_manager_promise(context: SaleorContext) -> Promise[PluginsManager]:
    app = get_app_promise(context).get()
    return plugin_manager_promise(context, app)
```

**生命周期**：
- 每个 GraphQL 请求上下文（`SaleorContext`）独立
- 同一请求中，同一 requestor（user 或 app）共享同一个 Manager 实例
- 请求结束后，随上下文一起被垃圾回收

#### 4.4.2 读写库路径在 GraphQL 中的使用

```python
# GraphQL Query（只读）：allow_replica=True（默认）
# saleor/graphql/plugins/dataloaders.py:35
allow_replica = getattr(self.context, "allow_replica", True)

# GraphQL Mutation（写操作）：allow_replica=False
# 示例：saleor/graphql/plugins/mutations.py 中的插件配置更新
manager = get_plugins_manager(allow_replica=False)
```

#### 4.4.3 后台任务（Celery Task）

```python
# 示例：saleor/plugins/user_email/tasks.py:38
with allow_writer():
    manager = get_plugins_manager(allow_replica=False)
    manager.notify(...)
```

**生命周期**：
- 每个任务独立创建 Manager 实例
- 任务执行完毕后销毁
- 必须使用 `allow_replica=False` 和 `allow_writer()` 上下文

#### 4.4.4 测试环境

```python
# 示例：saleor/graphql/tests/fixtures.py:173
"plugins": get_plugins_manager(allow_replica=False),
```

### 4.5 读写库路径对比表

| 维度 | allow_replica=True（读库） | allow_replica=False（写库） |
|------|--------------------------|---------------------------|
| 数据库连接 | "replica"（从库） | "default"（主库） |
| 适用场景 | GraphQL Query 查询、只读操作 | GraphQL Mutation、写操作、Celery 任务 |
| 性能 | 可分散读压力，提升性能 | 必须保证数据一致性，只能用主库 |
| allow_writer() | 不需要 | 必须使用 |
| 典型调用 | `get_plugin_manager_promise(context)` | `get_plugins_manager(allow_replica=False)` |

### 4.6 数据库查询中的应用

所有数据库查询都通过 `self.database` 属性指定连接：

```python
# saleor/plugins/manager.py:173-177
channel = (
    Channel.objects.using(self.database)
    .filter(slug=channel_slug)
    .first()
)

# saleor/plugins/manager.py:203-205
plugin_manager_configs = PluginConfiguration.objects.using(
    self.database
).filter(channel=channel)
```

---

## 5. 补充：插件加载与调用核心机制

### 5.1 延迟加载机制 (`saleor/plugins/manager.py:151-199`)

插件采用 **按需加载（Lazy Loading）** 策略，首次访问时才实例化：

```python
def _ensure_channel_plugins_loaded(self, channel_slug: str | None, channel: Channel | None = None):
    # 加载全局插件（CONFIGURATION_PER_CHANNEL = False 的插件）
    if channel_slug is None and not self.loaded_global:
        global_db_config = self._get_db_plugin_configs(None)
        
        for plugin_path in self.plugins:  # 按 PLUGINS 顺序遍历
            PluginClass = import_string(plugin_path)
            if not getattr(PluginClass, "CONFIGURATION_PER_CHANNEL", False):
                plugin = self._load_plugin(PluginClass, global_db_config, ...)
                self.global_plugins.append(plugin)
                self.all_plugins.append(plugin)
        self.loaded_global = True

    # 加载特定通道的插件（CONFIGURATION_PER_CHANNEL = True 的插件）
    if channel_slug is not None and channel_slug not in self.loaded_channels:
        channel_db_config = self._get_db_plugin_configs(channel)
        
        for plugin_path in self.plugins:
            PluginClass = import_string(plugin_path)
            if getattr(PluginClass, "CONFIGURATION_PER_CHANNEL", False):
                plugin = self._load_plugin(PluginClass, channel_db_config, channel=channel, ...)
                self.plugins_per_channel[channel_slug].append(plugin)
                self.all_plugins.append(plugin)
        
        # 把全局插件追加到通道插件列表后面
        self._ensure_channel_plugins_loaded(None)
        self.plugins_per_channel[channel_slug].extend(self.global_plugins)
        self.loaded_channels.add(channel_slug)
```

### 5.2 单个插件实例化 (`saleor/plugins/manager.py:104-129`)

```python
def _load_plugin(self, PluginClass: type["BasePlugin"], db_configs_map: dict, 
                 channel: Optional["Channel"] = None, ...) -> "BasePlugin":
    # 从数据库读取配置，若无则使用默认值
    if PluginClass.PLUGIN_ID in db_configs_map:
        db_config = db_configs_map[PluginClass.PLUGIN_ID]
        plugin_config = db_config.configuration
        active = db_config.active
        channel = db_config.channel
    else:
        plugin_config = PluginClass.DEFAULT_CONFIGURATION
        active = PluginClass.get_default_active()

    return PluginClass(
        configuration=plugin_config,
        active=active,
        channel=channel,
        requestor_getter=requestor_getter,
        db_config=db_config,
        allow_replica=allow_replica,
    )
```

### 5.3 运行时钩子分发 (`saleor/plugins/manager.py:211-255`)

插件调用的核心是 **责任链模式**，前一个插件的输出作为后一个插件的输入：

```python
def __run_method_on_plugins(self, method_name: str, default_value: Any, *args,
                            channel_slug: str | None, plugin_ids: list[str] | None = None,
                            **kwargs):
    """按顺序在所有激活的插件上运行指定方法"""
    value = default_value
    
    # 获取当前通道的插件列表（已按优先级排序）
    plugins = self.get_plugins(channel_slug=channel_slug, active_only=True, 
                               plugin_ids=plugin_ids)
    
    for plugin in plugins:
        # 逐个调用插件，previous_value 传递上一个插件的结果
        value = self.__run_method_on_single_plugin(
            plugin, method_name, value, *args, **kwargs
        )
    return value
```

### 5.4 单个插件调用 (`saleor/plugins/manager.py:233-255`)

```python
def __run_method_on_single_plugin(self, plugin: Optional["BasePlugin"], 
                                  method_name: str, previous_value: Any, 
                                  *args, **kwargs) -> Any:
    """在单个插件上运行方法，支持 NotImplemented 跳过机制"""
    plugin_method = getattr(plugin, method_name, NotImplemented)
    
    # 1. 插件未实现该方法 → 返回 previous_value
    if plugin_method == NotImplemented:
        return previous_value
    
    # 2. 方法不可调用 → 抛出异常
    if not callable(plugin_method):
        raise ValueError(f"Method {method_name} is not callable")
    
    # 3. 调用插件方法（传入 previous_value）
    returned_value = plugin_method(*args, **kwargs, previous_value=previous_value)
    
    # 4. 插件返回 NotImplemented → 返回 previous_value（跳过）
    if returned_value == NotImplemented:
        return previous_value
    
    return returned_value
```

---

## 6. 常见理解偏差修正

### 偏差 1：认为 PLUGINS 列表顺序只影响有 channel 场景

**修正**：PLUGINS 列表顺序影响所有场景。无论是全局插件还是各通道插件，内部都是按 PLUGINS 顺序遍历的。

### 偏差 2：认为无 channel 时 all_plugins 包含所有通道的插件

**修正**：无 channel 时 `_ensure_channel_plugins_loaded(None)` 只加载全局插件，不加载任何通道插件。只有先触发了某 channel 的调用，all_plugins 才会包含该 channel 的插件。

### 偏差 3：认为 all_plugins 的顺序是确定的（全局插件始终在前）

**修正**：`all_plugins` 的顺序完全取决于加载触发的历史时序。当先调用有 channel 的接口时，通道插件会先写入 `all_plugins`，全局插件后写入，导致顺序变为 `[通道插件, 全局插件]`。只有 `plugins_per_channel[channel_slug]` 的顺序是确定的。

### 偏差 4：认为 Manager 是全局单例

**修正**：Manager 不是全局单例。每个 GraphQL 请求为每个 requestor 创建独立实例，通过 DataLoader 在请求内复用。Celery 任务每次创建新实例。

### 偏差 5：认为 allow_replica 只是性能优化

**修正**：allow_replica 不仅是性能优化，还涉及数据一致性。写操作必须使用 `allow_replica=False`，并配合 `allow_writer()` 确保操作在主库执行。

### 偏差 6：认为 entry_points 只注册插件类

**修正**：entry_points 注册流程还会将插件所属的 Django App 注入 `INSTALLED_APPS`，这对插件的模型、迁移、管理命令等功能至关重要。

---

## 7. 总结

### 7.1 核心设计模式

| 模式 | 用途 | 关键代码 |
|------|------|---------|
| **责任链模式** | 插件按顺序处理，前一个的输出作为后一个的输入 | `__run_method_on_plugins` |
| **延迟加载** | 首次访问通道时才加载插件，节省启动时间 | `_ensure_channel_plugins_loaded` |
| **模板方法** | Manager 定义调用流程，插件实现具体逻辑 | `__run_method_on_single_plugin` |
| **策略模式** | 通过 `plugin_ids` 参数可选择特定插件子集执行 | `get_plugins(plugin_ids=...)` |

### 7.2 关键理解点

1. **外部插件注册**：entry_points 不仅注册插件类，还注入 Django App 到 `INSTALLED_APPS`
2. **无 channel 调用**：只返回已加载的插件，顺序取决于加载触发时机，**时序敏感**
3. **all_plugins 顺序不确定**：取决于加载历史，先调用有 channel 则通道插件在前，先调用无 channel 则全局插件在前
4. **plugins_per_channel 顺序确定**：始终是 `[通道插件按 PLUGINS 顺序] + [全局插件按 PLUGINS 顺序]`
5. **Manager 生命周期**：请求级隔离，DataLoader 实现同一请求内复用
6. **读写库路径**：读操作走 replica，写操作走 default，由 `allow_replica` 参数控制
7. **PLUGINS 顺序**：决定所有场景下的插件调用优先级
