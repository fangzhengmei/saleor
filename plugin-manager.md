# Saleor Plugin Manager 实现分析

## 1. 概述

Saleor 的插件系统采用 **责任链模式** + **延迟加载** 的设计，核心类为 `PluginsManager`（位于 `saleor/plugins/manager.py`）。插件用于扩展 Saleor 的核心功能，包括：

- 税务计算（如 Avalara 插件）
- 支付网关（如 Stripe、Braintree）
- 邮件通知
- Webhook 分发
- 第三方认证（如 OpenID Connect）

## 2. 插件注册配置

### 2.1 配置文件定义 (`saleor/settings.py:892-922`)

插件通过 settings 中的 `PLUGINS` 列表进行注册，分为两类：

```python
# 内置插件列表
BUILTIN_PLUGINS = [
    "saleor.plugins.webhook.plugin.WebhookPlugin",
    "saleor.plugins.avatax.plugin.DeprecatedAvataxPlugin",
    "saleor.payment.gateways.stripe.plugin.StripeGatewayPlugin",
    "saleor.plugins.user_email.plugin.UserEmailPlugin",
    "saleor.plugins.openid_connect.plugin.OpenIDConnectPlugin",
    # ... 更多内置插件
]

# 通过 entry_points 动态发现的外部插件
EXTERNAL_PLUGINS = []
installed_plugins = importlib.metadata.entry_points(group="saleor.plugins")
for entry_point in installed_plugins:
    plugin_path = f"{entry_point.module}.{entry_point.attr}"
    EXTERNAL_PLUGINS.append(plugin_path)

# 最终插件列表（决定调用优先级！）
PLUGINS: list[str] = BUILTIN_PLUGINS + EXTERNAL_PLUGINS
```

**关键点**：`PLUGINS` 列表的顺序直接决定了插件执行的优先级，排在前面的插件先被调用。

### 2.2 插件基类定义 (`saleor/plugins/base_plugin.py:100-150`)

所有插件必须继承 `BasePlugin` 抽象类：

```python
class BasePlugin:
    # 插件唯一标识（用于数据库存储和配置查找）
    PLUGIN_ID = ""
    PLUGIN_NAME = ""
    PLUGIN_DESCRIPTION = ""
    
    # 是否支持按通道配置（True = 每个 Channel 可独立配置）
    CONFIGURATION_PER_CHANNEL = True
    
    # 默认配置结构
    DEFAULT_CONFIGURATION = []
    DEFAULT_ACTIVE = False
    
    # 配置字段类型定义
    CONFIG_STRUCTURE = None
    
    # 是否隐藏（不在管理界面显示）
    HIDDEN = False
```

### 2.3 插件配置模型 (`saleor/plugins/models.py:9-25`)

插件配置存储在 `PluginConfiguration` 表中：

```python
class PluginConfiguration(models.Model):
    identifier = models.CharField(max_length=128)  # 对应 PLUGIN_ID
    name = models.CharField(max_length=128)
    channel = models.ForeignKey(Channel, null=True, on_delete=models.CASCADE)
    description = models.TextField(blank=True)
    active = models.BooleanField(default=False)
    configuration = JSONField(default=dict)  # 实际配置数据
    
    class Meta:
        unique_together = ("identifier", "channel")  # 联合唯一约束
```

## 3. 插件管理器初始化与加载

### 3.1 管理器构造 (`saleor/plugins/manager.py:131-142`)

```python
class PluginsManager(PaymentInterface):
    # 类级缓存
    plugins_per_channel: dict[str, list["BasePlugin"]] = {}
    global_plugins: list["BasePlugin"] = []
    all_plugins: list["BasePlugin"] = []

    def __init__(self, plugins: list[str], requestor_getter=None, allow_replica=True):
        self.plugins = plugins  # PLUGINS 配置列表
        self._allow_replica = allow_replica
        self.all_plugins = []
        self.global_plugins = []
        self.plugins_per_channel = defaultdict(list)
        
        # 延迟加载标记
        self.loaded_all_channels = False
        self.loaded_channels: set[str] = set()
        self.loaded_global = False
        
        self.requestor_getter = requestor_getter  # 获取当前请求用户
```

### 3.2 延迟加载机制 (`saleor/plugins/manager.py:151-199`)

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

**加载顺序说明**：
1. 通道专属插件（按 PLUGINS 顺序）
2. 全局插件（按 PLUGINS 顺序，追加在后面）

### 3.3 单个插件实例化 (`saleor/plugins/manager.py:104-129`)

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

### 3.4 数据库配置读取 (`saleor/plugins/manager.py:201-209`)

```python
def _get_db_plugin_configs(self, channel: Channel | None):
    plugin_manager_configs = PluginConfiguration.objects.using(self.database)\
        .filter(channel=channel)
    configs = {}
    for db_plugin_config in plugin_manager_configs.iterator(chunk_size=1000):
        configs[db_plugin_config.identifier] = db_plugin_config
    return configs
```

## 4. 运行时钩子分发机制

### 4.1 核心调用流程 (`saleor/plugins/manager.py:211-255`)

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

### 4.2 单个插件调用 (`saleor/plugins/manager.py:233-255`)

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

### 4.3 获取插件列表 (`saleor/plugins/manager.py:2488-2508`)

```python
def get_plugins(self, channel_slug: str | None = None, active_only=False,
                plugin_ids: list[str] | None = None) -> list["BasePlugin"]:
    # 触发延迟加载
    if channel_slug is not None:
        self._ensure_channel_plugins_loaded(channel_slug)
        plugins = self.plugins_per_channel[channel_slug]
    else:
        self._ensure_channel_plugins_loaded(None)
        plugins = self.all_plugins

    # 过滤激活状态
    if active_only:
        plugins = [plugin for plugin in plugins if plugin.active]

    # 按 plugin_ids 过滤
    if plugin_ids:
        plugins = [plugin for plugin in plugins if plugin.PLUGIN_ID in plugin_ids]

    return plugins
```

## 5. 特殊调用模式

### 5.1 "直到第一个成功"模式 (`saleor/plugins/manager.py:2582-2599`)

适用于只需一个插件处理的场景（如获取税率类型列表）：

```python
def __run_plugin_method_until_first_success(self, method_name: str, *args,
                                            channel_slug: str | None, 
                                            plugins: list["BasePlugin"] | None = None,
                                            **kwargs):
    if plugins is None:
        plugins = self.get_plugins(channel_slug=channel_slug, active_only=True)
    
    for plugin in plugins:
        result = self.__run_method_on_single_plugin(
            plugin, method_name, None, *args, **kwargs
        )
        if result is not None:  # 第一个返回非 None 结果的插件获胜
            return result
    return None
```

### 5.2 税务方法特殊模式 (`saleor/plugins/manager.py:632-657`)

税务插件的错误处理逻辑更复杂：

```python
def __run_tax_method_until_first_success(self, method_name: str, *args,
                                         channel_slug: str | None, **kwargs
                                         ) -> TaxData | None:
    plugins = self.get_plugins(channel_slug=channel_slug, active_only=True)
    error = None
    
    for plugin in plugins:
        try:
            tax_data = self.__run_method_on_single_plugin(
                plugin, method_name, default_value, *args, **kwargs
            )
        except TaxDataError as e:
            error = e  # 记录错误，继续尝试下一个
            continue
        
        if tax_data is not None:
            return tax_data  # 成功则返回
    
    if error:
        raise error  # 全部失败且有错误则抛出最后一个
    return default_value
```

### 5.3 支付网关模式 (`saleor/plugins/manager.py:2556-2580`)

支付方法指定具体网关插件调用：

```python
def __run_payment_method(self, gateway: str, method_name: str,
                         payment_information: "PaymentData", 
                         channel_slug: str, **kwargs) -> "GatewayResponse":
    # 查找指定的支付插件
    plugin = self.get_plugin(gateway, channel_slug)
    if plugin is not None:
        resp = self.__run_method_on_single_plugin(
            plugin, method_name, previous_value=None,
            payment_information=payment_information, **kwargs
        )
        if resp is not None:
            return resp
    
    raise Exception(f"Payment plugin {gateway} for {method_name} is inaccessible!")
```

## 6. 插件方法实现示例

以 Avalara 税务插件的 `calculate_checkout_total` 为例：

```python
# saleor/plugins/avatax/plugin.py:211-254
class DeprecatedAvataxPlugin(BasePlugin):
    PLUGIN_ID = "mirumee.taxes.avalara"
    
    def calculate_checkout_total(self, checkout_info: "CheckoutInfo",
                                 lines: list["CheckoutLineInfo"],
                                 address: Optional["Address"],
                                 previous_value: TaxedMoney) -> TaxedMoney:
        # 1. 检查是否应该跳过此插件
        if self._skip_plugin(previous_value):
            return previous_value  # 返回前一个插件的结果
        
        # 2. 调用 Avalara API 获取税务数据
        response = self._get_checkout_tax_data(checkout_info, lines, previous_value)
        if response is None:
            return previous_value
        
        # 3. 应用税务计算
        currency = checkout_info.checkout.currency
        taxed_total = zero_taxed_money(currency)
        for line in lines:
            taxed_line_total_data = self._calculate_checkout_line_total_price(...)
            taxed_total += taxed_line_total_data
        
        # 4. 返回计算结果（传给下一个插件作为 previous_value）
        return max(taxed_total, zero_taxed_money(taxed_total.currency))
```

## 7. 调用优先级与数据流

### 7.1 优先级排序规则

插件调用顺序由以下因素决定（按优先级从高到低）：

1. **`PLUGINS` 配置列表顺序** - 最关键，排在前面的插件先被调用
2. **通道专属插件优先** - `CONFIGURATION_PER_CHANNEL=True` 的插件排在全局插件前面
3. **全局插件追加在后** - `CONFIGURATION_PER_CHANNEL=False` 的插件统一追加在列表末尾

### 7.2 数据流示意

```
调用 manager.calculate_checkout_total(...)
          ↓
default_value = base_calculations.checkout_total(...)  # 基础计算值
          ↓
插件1.calculate_checkout_total(..., previous_value=default_value) → value1
          ↓
插件2.calculate_checkout_total(..., previous_value=value1) → value2
          ↓
插件3.calculate_checkout_total(..., previous_value=value2) → value3
          ↓
...（所有插件依次处理）
          ↓
返回最终结果
```

## 8. 关键设计要点

### 8.1 previous_value 约定

- 每个插件方法接收 `previous_value` 参数（前一个插件的输出）
- 插件可选择：
  - 基于 `previous_value` 进行修改后返回
  - 完全忽略 `previous_value` 重新计算
  - 返回 `NotImplemented` 表示跳过（透传 `previous_value`）
  - 检查 `previous_value` 是否已被处理（如 `net != gross` 表示已含税）

### 8.2 线程安全与缓存

- `PluginsManager` 每个请求实例化一次（通过 `get_plugins_manager` context）
- 插件实例存储在实例变量中，非全局共享
- 数据库配置使用 `iterator(chunk_size=1000)` 分批加载

### 8.3 配置持久化

插件配置保存方法 (`saleor/plugins/manager.py:2636-2667`)：

```python
def save_plugin_configuration(self, plugin_id, channel_slug: str | None, 
                              cleaned_data: dict) -> None | PluginConfiguration:
    # 1. 创建或获取 PluginConfiguration 记录
    plugin_configuration, _ = PluginConfiguration.objects.get_or_create(
        identifier=plugin_id,
        channel=channel,
        defaults={"configuration": plugin.configuration},
    )
    # 2. 调用插件的配置验证和保存逻辑
    configuration = plugin.save_plugin_configuration(
        plugin_configuration, cleaned_data
    )
    # 3. 更新内存中插件实例的状态
    plugin.active = configuration.active
    plugin.configuration = configuration.configuration
    return configuration
```

## 9. 常见钩子方法分类

### 9.1 计算类钩子（链式调用）
- `calculate_checkout_total` / `calculate_order_total`
- `calculate_checkout_shipping` / `calculate_order_shipping`
- `calculate_checkout_line_total` / `calculate_order_line_total`
- `get_checkout_line_tax_rate` / `get_order_line_tax_rate`

### 9.2 事件通知类钩子（广播调用）
- `order_created` / `order_updated` / `order_cancelled`
- `product_created` / `product_updated` / `product_deleted`
- `customer_created` / `customer_updated`
- `checkout_created` / `checkout_updated`

### 9.3 支付类钩子（指定调用）
- `authorize_payment` / `capture_payment` / `refund_payment`
- `process_payment` / `initialize_payment`
- `list_payment_sources`

### 9.4 认证类钩子
- `authenticate_user`
- `external_obtain_access_tokens`
- `external_verify`

## 10. 总结

Saleor Plugin Manager 的核心设计模式：

| 模式 | 用途 |
|------|------|
| **责任链模式** | 插件按顺序处理，前一个的输出作为后一个的输入 |
| **延迟加载** | 首次访问通道时才加载插件，节省启动时间 |
| **模板方法** | `__run_method_on_plugins` 定义调用流程，插件实现具体逻辑 |
| **策略模式** | 通过 `plugin_ids` 参数可选择特定插件子集执行 |

关键理解点：
1. `PLUGINS` 配置顺序 = 调用优先级
2. `previous_value` 是插件间数据传递的核心
3. 通道插件在前，全局插件在后
4. 返回 `NotImplemented` = 跳过当前插件，透传值
