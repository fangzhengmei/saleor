# Saleor 员工权限组与查询接口守卫关系详解

## 一、整体架构概览

Saleor 的权限系统采用**四层防护机制**：字段级访问控制、查询接口守卫、核心权限校验、业务逻辑校验。它们之间的关系如下：

```
┌─────────────────────────────────────────────────────────────┐
│                    GraphQL 请求入口                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│         第一层：字段级权限控制 (PermissionsField)            │
│         - 声明式权限声明                                    │
│         - 自动包装 resolver                                 │
│         saleor/graphql/core/fields.py                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│         第二层：权限装饰器校验 (one_of_permissions_required) │
│         - 从 context 中提取 requestor                       │
│         - 检查用户是否拥有任一所需权限                       │
│         saleor/graphql/decorators.py                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│         第三层：核心权限校验逻辑                             │
│         - 权限枚举匹配                                      │
│         - 用户/APP 权限检查                                 │
│         saleor/permission/utils.py                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│         第四层：权限组范围校验 (业务逻辑层)                   │
│         - 检查用户是否可管理目标权限组                       │
│         - 权限范围、渠道范围校验                            │
│         saleor/graphql/account/utils.py                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、角色分组：权限组 (Group) 的数据结构

### 2.1 Group 模型定义

**文件位置：** `saleor/graphql/account/types.py:1032-1106`

```python
class Group(ModelObjectType[models.Group]):
    id = graphene.GlobalID(required=True)
    name = graphene.String(required=True)
    
    # 🔴 字段级权限控制：只有 MANAGE_STAFF 权限才能查看用户列表
    users = PermissionsField(
        NonNullList(User),
        description="List of group users",
        permissions=[AccountPermissions.MANAGE_STAFF],
    )
    
    permissions = NonNullList(Permission, description="List of group permissions")
    
    # 🔴 动态权限判断：当前用户是否可管理此组
    user_can_manage = graphene.Boolean(
        required=True,
        description="True, if the currently authenticated user has rights to manage a group."
    )
    
    accessible_channels = NonNullList(Channel)
    restricted_access_to_channels = graphene.Boolean(required=True)
```

### 2.2 关键字段解析

| 字段 | 说明 | 权限控制方式 |
|------|------|-------------|
| `users` | 组内用户列表 | 静态字段权限 `MANAGE_STAFF` |
| `permissions` | 组拥有的权限列表 | 无直接权限限制，但受查询接口守卫保护 |
| `user_can_manage` | 当前用户是否可管理此组 | 动态业务逻辑判断 |
| `accessible_channels` | 组可访问的渠道 | DataLoader 异步加载 |
| `restricted_access_to_channels` | 是否限制渠道访问 | 无直接权限限制 |

### 2.3 Group 数据库模型

**文件位置：** `saleor/account/models.py:396-434`

```python
class Group(models.Model):
    name = models.CharField("name", max_length=150, unique=True)
    permissions = models.ManyToManyField(Permission, blank=True)
    restricted_access_to_channels = models.BooleanField(default=False)
    channels = models.ManyToManyField("channel.Channel", blank=True)
```

---

## 三、字段访问控制：PermissionsField 工作原理

### 3.1 PermissionsField 类定义

**文件位置：** `saleor/graphql/core/fields.py:43-65`

```python
class PermissionsField(BaseField):
    def __init__(self, *args, **kwargs):
        # 提取权限列表
        self.permissions = kwargs.pop("permissions", [])
        auto_permission_message = kwargs.pop("auto_permission_message", True)
        
        super().__init__(*args, **kwargs)
        
        # 自动在 description 后追加权限说明
        if auto_permission_message and self.permissions:
            permissions_msg = message_one_of_permissions_required(self.permissions)
            self.description = (self.description or "") + permissions_msg

    def get_resolver(self, parent_resolver):
        resolver = self.resolver or parent_resolver
        # 🎯 关键：用权限装饰器包装 resolver
        if self.permissions:
            resolver = one_of_permissions_required(self.permissions)(resolver)
        resolver = super().get_resolver(resolver)
        return resolver
```

### 3.2 执行流程

1. **字段定义时**：在 Schema 中声明 `PermissionsField` 并指定 `permissions` 参数
2. **Schema 构建时**：`get_resolver` 被调用，将权限装饰器应用到 resolver
3. **请求执行时**：resolver 执行前先经过权限校验

### 3.3 使用示例

**文件位置：** `saleor/graphql/account/types.py:387-391, 433-437`

```python
class User(ModelObjectType[models.User]):
    # 示例1：note 字段需要 MANAGE_USERS 或 MANAGE_STAFF 权限
    note = PermissionsField(
        graphene.String,
        description="A note about the customer.",
        permissions=[AccountPermissions.MANAGE_USERS, AccountPermissions.MANAGE_STAFF],
    )
    
    # 示例2：events 字段同样需要权限
    events = PermissionsField(
        NonNullList(CustomerEvent),
        description="List of events associated with the user.",
        permissions=[AccountPermissions.MANAGE_USERS, AccountPermissions.MANAGE_STAFF],
    )
```

---

## 四、拦截判定：权限装饰器与核心校验

### 4.1 one_of_permissions_required 装饰器

**文件位置：** `saleor/graphql/decorators.py:84-89`

```python
def one_of_permissions_required(perms: Iterable[BasePermissionEnum]):
    def check_perms(context):
        if not one_of_permissions_or_auth_filter_required(context, perms):
            raise PermissionDenied(permissions=perms)
    
    return account_passes_test(check_perms)
```

### 4.2 account_passes_test 包装逻辑

**文件位置：** `saleor/graphql/decorators.py:39-51`

```python
def account_passes_test(test_func):
    """Determine if user/app has permission to access to content."""
    def decorator(f):
        @wraps(f)
        @context(f)
        def wrapper(context, *args, **kwargs):
            test_func(context)  # 🔴 权限校验在 resolver 执行前调用
            return f(*args, **kwargs)
        return wrapper
    return decorator
```

### 4.3 核心权限校验函数

**文件位置：** `saleor/permission/utils.py:32-46`

```python
def one_of_permissions_or_auth_filter_required(
    context, permissions: Iterable[BasePermissionEnum]
) -> bool:
    """Determine whether user or app has rights to perform an action."""
    if not permissions:
        return True

    # 检查常规权限（如 MANAGE_STAFF, MANAGE_USERS 等）
    perm_results = _get_result_of_permissions_checks(context, permissions)
    
    # 检查授权过滤器（如 OWNER 等）
    auth_filters_results = _get_result_of_authorization_filters_checks(
        context, permissions
    )
    
    # 🎯 任一权限通过即可：常规权限 OR 授权过滤器
    return any(perm_results) or any(auth_filters_results)
```

### 4.4 权限检查实现细节

**文件位置：** `saleor/permission/utils.py:49-62`

```python
def _get_result_of_permissions_checks(
    context, permissions: Iterable[BasePermissionEnum]
) -> Iterable[bool]:
    # 过滤掉授权过滤器，只保留常规权限
    permissions = [p for p in permissions if not isinstance(p, AuthorizationFilters)]
    
    from saleor.graphql.utils import get_user_or_app_from_context
    requestor = get_user_or_app_from_context(context)
    
    perm_checks_results = []
    if requestor and permissions:
        # 调用 User 或 App 的 has_perm 方法
        perm_checks_results = [requestor.has_perm(perm) for perm in permissions]
    return perm_checks_results
```

### 4.5 requestor 提取逻辑

**文件位置：** `saleor/graphql/utils/__init__.py:204-207`

```python
def get_user_or_app_from_context(context: "SaleorContext") -> App | User | None:
    # order is important
    # app can be None but user if None then is passed as anonymous
    return context.app or context.user
```

---

## 五、查询守卫中的拦截分支详解

### 5.1 三类拦截分支

查询守卫在权限校验过程中会遇到三类请求者情况，每种情况的拦截逻辑不同：

```
one_of_permissions_or_auth_filter_required(context, permissions)
          │
          ├─► 第一步：分离权限类型
          │     ├─► 常规权限：非 AuthorizationFilters 实例
          │     └─► 授权过滤器：AuthorizationFilters 实例
          │
          ├─► 第二步：常规权限检查（_get_result_of_permissions_checks）
          │     │
          │     └─► get_user_or_app_from_context(context)
          │           │
          │           ├─► 情况1：requestor is None（未认证用户）
          │           │       └─► 返回空列表 → any([]) = False → 常规权限不通过
          │           │
          │           ├─► 情况2：requestor 是 User（已认证用户）
          │           │       └─► 调用 user.has_perm(perm) 逐个检查
          │           │
          │           └─► 情况3：requestor 是 App（应用请求）
          │                   ├─► 检查 app.is_active
          │                   ├─► 调用 app.has_perm(perm) 逐个检查
          │                   └─► 特殊限制：某些操作直接拒绝 App
          │
          └─► 第三步：授权过滤器检查（_get_result_of_authorization_filters_checks）
                │
                ├─► AuthorizationFilters.AUTHENTICATED_APP → is_app(context)
                ├─► AuthorizationFilters.AUTHENTICATED_STAFF_USER → is_staff_user(context)
                ├─► AuthorizationFilters.AUTHENTICATED_USER → is_user(context)
                └─► AuthorizationFilters.OWNER → 无内置函数，需业务逻辑层处理
```

### 5.1.1 AuthorizationFilters 枚举与映射

**文件位置：** `saleor/permission/auth_filters.py:17-41`

```python
class AuthorizationFilters(BasePermissionEnum):
    # 任何已认证的 App 都可以访问
    AUTHENTICATED_APP = "authorization_filters.authenticated_app"
    # 任何已认证的 staff 用户都可以访问
    AUTHENTICATED_STAFF_USER = "authorization_filters.authenticated_staff_user"
    # 任何已认证的用户都可以访问
    AUTHENTICATED_USER = "authorization_filters.authenticated_user"
    # 资源所有者可以访问，需业务逻辑层自行判断
    OWNER = "authorization_filters.owner"

# 🔑 授权过滤器到检查函数的映射
AUTHORIZATION_FILTER_MAP = {
    AuthorizationFilters.AUTHENTICATED_APP: is_app,
    AuthorizationFilters.AUTHENTICATED_USER: is_user,
    AuthorizationFilters.AUTHENTICATED_STAFF_USER: is_staff_user,
}

def is_app(context):
    return bool(context.app)

def is_user(context):
    user = context.user
    return user and user.is_active

def is_staff_user(context):
    return is_user(context) and context.user.is_staff

def resolve_authorization_filter_fn(perm):
    return AUTHORIZATION_FILTER_MAP.get(perm)
```

### 5.1.2 两种核心校验函数的组合逻辑

#### 组合逻辑 1：one_of_permissions_or_auth_filter_required（OR 逻辑）

**文件位置：** `saleor/permission/utils.py:32-46`

```python
def one_of_permissions_or_auth_filter_required(
    context, permissions: Iterable[BasePermissionEnum]
) -> bool:
    if not permissions:
        return True

    # 1. 分离并检查常规权限
    perm_results = _get_result_of_permissions_checks(context, permissions)
    
    # 2. 分离并检查授权过滤器
    auth_filters_results = _get_result_of_authorization_filters_checks(
        context, permissions
    )
    
    # 🎯 OR 逻辑：常规权限任一通过 OR 授权过滤器任一通过
    return any(perm_results) or any(auth_filters_results)
```

**执行流程示例（permissions = [MANAGE_USERS, OWNER]）：**
```
1. 分离权限：
   perm_results = [user.has_perm(MANAGE_USERS)]
   auth_filters_results = []  # OWNER 无映射函数，跳过
   
2. 结果计算：
   any([True]) or any([]) = True  → 常规权限通过
   any([False]) or any([]) = False → 都不通过，拦截
```

#### 组合逻辑 2：all_permissions_required（AND + OR 混合逻辑）

**文件位置：** `saleor/permission/utils.py:12-29`

```python
def all_permissions_required(context, permissions: Iterable[BasePermissionEnum]):
    if not permissions:
        return True

    perm_results = _get_result_of_permissions_checks(context, permissions)
    auth_filters_results = _get_result_of_authorization_filters_checks(
        context, permissions
    )
    
    # 🎯 混合逻辑：(所有常规权限都通过) AND (任一授权过滤器通过)
    if auth_filters_results:
        return all(perm_results) and any(auth_filters_results)
    return all(perm_results)
```

**执行流程示例（permissions = [MANAGE_APPS, AUTHENTICATED_APP]）：**
```
1. 分离权限：
   perm_results = [app.has_perm(MANAGE_APPS)]
   auth_filters_results = [is_app(context)]
   
2. 结果计算：
   all([True]) and any([True]) = True  → 全部通过
   all([True]) and any([False]) = False → 授权过滤器不通过，拦截
   all([False]) and any([True]) = False → 常规权限不通过，拦截
```

### 5.1.3 _get_result_of_authorization_filters_checks 实现

**文件位置：** `saleor/permission/utils.py:65-79`

```python
def _get_result_of_authorization_filters_checks(
    context, permissions: Iterable[BasePermissionEnum]
) -> Iterable[bool]:
    # 🔑 只提取 AuthorizationFilters 类型的权限
    authorization_filters = [
        p for p in permissions if isinstance(p, AuthorizationFilters)
    ]
    auth_filters_results = []
    if authorization_filters:
        for p in authorization_filters:
            # 查找对应的检查函数
            perm_fn = resolve_authorization_filter_fn(p)
            if perm_fn:
                # 执行检查函数
                res = perm_fn(context)
                auth_filters_results.append(bool(res))
            # ⚠️  OWNER 没有映射函数，所以不会被加入结果列表！

    return auth_filters_results
```

**关键注意点：**
- `AuthorizationFilters.OWNER` 在 `AUTHORIZATION_FILTER_MAP` 中没有对应函数
- 因此 `resolve_authorization_filter_fn(OWNER)` 返回 `None`
- 不会被加入 `auth_filters_results`，即 `any(auth_filters_results)` 对 OWNER 永远是 `False`
- OWNER 权限必须在**业务逻辑层**通过 `is_owner_or_has_one_of_perms()` 显式检查

### 5.2 分支 1：未认证用户拦截

**触发条件：** `context.app is None` 且 `context.user is None` 或为 AnonymousUser

**拦截路径：**
```python
# _get_result_of_permissions_checks()
requestor = get_user_or_app_from_context(context)  # 返回 None
perm_checks_results = []  # 空列表
return []

# one_of_permissions_or_auth_filter_required()
any([]) = False
any([]) = False  # auth_filters_results 也为空
return False

# one_of_permissions_required()
raise PermissionDenied(permissions=perms)
```

**返回结果：**
```json
{
  "errors": [
    {
      "message": "To access this path, you need one of the following permissions: MANAGE_STAFF",
      "extensions": {
        "exception": {
          "code": "PermissionDenied"
        }
      }
    }
  ],
  "data": null
}
```

### 5.3 分支 2：无权限用户拦截

**触发条件：** requestor 是 User，但 `user.has_perm(perm)` 对所有权限都返回 False

**User.has_perm() 实现：** `saleor/account/models.py:303-310`

```python
def has_perm(self, perm: BasePermissionEnum | str, obj=None) -> bool:
    # 转换为字符串格式，如 "account.manage_staff"
    perm = perm.value if isinstance(perm, BasePermissionEnum) else perm

    # 超级用户直接通过（除非被 effective_permissions 覆盖）
    if self.is_active and self.is_superuser and not self._effective_permissions:
        return True
    
    # 调用认证后端链检查
    return _user_has_perm(self, perm, obj)
```

**_user_has_perm 后端链检查：** `saleor/permission/models.py:8-18`

```python
def _user_has_perm(user, perm, obj):
    """Backend can raise `PermissionDenied` to short-circuit permission checking."""
    for backend in auth.get_backends():
        if not hasattr(backend, "has_perm"):
            continue
        try:
            if backend.has_perm(user, perm, obj):
                return True
        except PermissionDenied:
            return False
    return False
```

**JSONWebTokenBackend.has_perm()：** `saleor/core/auth_backend.py:103-104`

```python
def has_perm(self, user_obj, perm, obj=None):
    return user_obj.is_active and super().has_perm(user_obj, perm, obj=obj)
```

**BaseBackend.has_perm()：** `saleor/core/auth_backend.py:45-46`

```python
def has_perm(self, user_obj, perm, obj=None):
    return perm in self.get_all_permissions(user_obj, obj=obj)
```

### 5.4 分支 3：App 请求拦截

**App.has_perm() 实现：** `saleor/app/models.py:122-128`

```python
def has_perm(self, perm: BasePermissionEnum | str) -> bool:
    """Return True if the app has the specified permission."""
    if not self.is_active:
        return False

    perm_value = perm.value if isinstance(perm, BasePermissionEnum) else perm
    return perm_value in self.get_permissions()
```

**App.get_permissions() 实现：** `saleor/app/models.py:98-107`

```python
def get_permissions(self) -> set[str]:
    """Return the permissions of the app."""
    if not self.is_active:
        return set()
    perm_cache_name = "_app_perm_cache"
    if not hasattr(self, perm_cache_name):
        perms = self.permissions.all()
        perms = perms.values_list("content_type__app_label", "codename").order_by()
        setattr(self, perm_cache_name, {f"{ct}.{name}" for ct, name in perms})
    return getattr(self, perm_cache_name)
```

**App 特殊限制拦截示例：** `saleor/graphql/account/mutations/permission_group/permission_group_create.py:140-148`

```python
@classmethod
def check_permissions(
    cls, context, permissions=None, require_all_permissions=False, **data
):
    app = get_app_promise(context).get()
    if app:
        # 🔴 直接拦截：App 不允许创建权限组
        raise PermissionDenied(message="Apps are not allowed to perform this mutation.")
    return super().check_permissions(context, permissions)
```

### 5.5 OWNER 权限的特殊处理方式

#### 5.5.1 OWNER 权限为何不在查询守卫层处理

如 5.1.3 所述，`AuthorizationFilters.OWNER` 没有内置的检查函数，这是因为：
- 所有权判断依赖于具体业务对象（如订单所有者、用户资料所有者）
- 查询守卫层无法知道当前 resolver 正在访问哪个对象的所有者
- 必须由业务逻辑层显式判断 `requestor == owner`

#### 5.5.2 is_owner_or_has_one_of_perms 实现

**文件位置：** `saleor/graphql/account/utils.py:364-376`

```python
def is_owner_or_has_one_of_perms(
    requestor: Union["User", "App", None], owner: Union["User", "App"] | None, *perms
) -> bool:
    """Check if requestor can access data.
    
    :param requestor: Requestor user or app.
    :param owner: Data owner.
    :param perms:
        Permissions which can give the access to the data.
        Requestor needs to have at least one of given permissions
        to get access to protected resource.
    """
    # 🎯 OR 逻辑：是所有者 OR 拥有任一权限
    return requestor == owner or has_one_of_permissions(requestor, perms)
```

#### 5.5.3 check_is_owner_or_has_one_of_perms 拒绝触发

**文件位置：** `saleor/graphql/account/utils.py:379-394`

```python
def check_is_owner_or_has_one_of_perms(
    requestor: Union["User", "App", None], owner: Optional["User"], *perms
) -> None:
    """Confirm that requestor can access data, raise `PermissionDenied` otherwise."""
    if not is_owner_or_has_one_of_perms(requestor, owner, *perms):
        # 🔴 拒绝时，权限列表包含常规权限 + OWNER
        raise PermissionDenied(permissions=list(perms) + [AuthorizationFilters.OWNER])
```

#### 5.5.4 OWNER 权限使用示例

**文件位置：** `saleor/graphql/app/types.py:84-89`

```python
def has_required_permission(app: models.App, context: SaleorContext):
    requester = get_user_or_app_from_context(context)
    if not is_owner_or_has_one_of_perms(requester, app, AppPermission.MANAGE_APPS):
        raise PermissionDenied(
            # 🔑 权限列表同时包含常规权限和 OWNER
            permissions=[AppPermission.MANAGE_APPS, AuthorizationFilters.OWNER]
        )
```

### 5.6 PermissionDenied 异常类与拒绝返回

**文件位置：** `saleor/core/exceptions.py:72-84`

```python
class PermissionDenied(Exception):
    def __init__(self, message=None, *, permissions: Iterable[Enum] | None = None):
        if not message:
            if permissions:
                # 🔑 枚举的 .name 属性会被用于构建错误消息
                permission_list = ", ".join(p.name for p in permissions)
                message = (
                    "To access this path, you need one of the "
                    f"following permissions: {permission_list}"
                )
            else:
                message = "You do not have permission to perform this action"
        super().__init__(message)
        self.permissions = permissions
```

### 5.7 各类拒绝场景的返回结果汇总

#### 场景 1：仅常规权限检查失败

**触发条件：** `permissions = [MANAGE_STAFF]`，用户无此权限

**返回结果：**
```json
{
  "errors": [
    {
      "message": "To access this path, you need one of the following permissions: MANAGE_STAFF",
      "locations": [{"line": 2, "column": 3}],
      "path": ["permissionGroups"],
      "extensions": {
        "exception": {
          "code": "PermissionDenied"
        }
      }
    }
  ],
  "data": {
    "permissionGroups": null
  }
}
```

#### 场景 2：常规权限 + 授权过滤器检查失败

**触发条件：** `permissions = [MANAGE_APPS, AUTHENTICATED_APP]`，请求来自未认证的 App

**内部逻辑：**
```python
perm_results = [False]  # App 无 MANAGE_APPS 权限
auth_filters_results = [False]  # is_app(context) 返回 False
any([False]) or any([False]) = False
```

**返回结果：**
```json
{
  "errors": [
    {
      "message": "To access this path, you need one of the following permissions: MANAGE_APPS, AUTHENTICATED_APP",
      "path": ["app"],
      "extensions": {
        "exception": {
          "code": "PermissionDenied"
        }
      }
    }
  ],
  "data": {
    "app": null
  }
}
```

#### 场景 3：常规权限 + OWNER 检查失败（业务逻辑层）

**触发条件：** `check_is_owner_or_has_one_of_perms(requestor, owner, MANAGE_APPS)`，请求者既不是所有者也没有 MANAGE_APPS 权限

**拒绝触发点：** `saleor/graphql/account/utils.py:394`

**返回结果：**
```json
{
  "errors": [
    {
      "message": "To access this path, you need one of the following permissions: MANAGE_APPS, OWNER",
      "path": ["app", "privateMeta"],
      "extensions": {
        "exception": {
          "code": "PermissionDenied"
        }
      }
    }
  ],
  "data": {
    "app": {
      "privateMeta": null
    }
  }
}
```

#### 场景 4：all_permissions_required 检查失败

**触发条件：** `permissions = [MANAGE_APPS, AUTHENTICATED_APP]`，App 有 MANAGE_APPS 权限但请求来自用户

**内部逻辑：**
```python
perm_results = [True]  # 用户有 MANAGE_APPS 权限
auth_filters_results = [False]  # is_app(context) 返回 False
all([True]) and any([False]) = False  # AND 逻辑，授权过滤器必须通过
```

**返回结果：**
```json
{
  "errors": [
    {
      "message": "To access this path, you need one of the following permissions: MANAGE_APPS, AUTHENTICATED_APP",
      "extensions": {
        "exception": {
          "code": "PermissionDenied"
        }
      }
    }
  ],
  "data": null
}
```

#### 场景 5：App 特殊限制拦截

**触发条件：** App 调用 `PermissionGroupCreate` mutation

**拒绝触发点：** `saleor/graphql/account/mutations/permission_group/permission_group_create.py:145-147`

**返回结果：**
```json
{
  "errors": [
    {
      "message": "Apps are not allowed to perform this mutation.",
      "extensions": {
        "exception": {
          "code": "PermissionDenied"
        }
      }
    }
  ],
  "data": {
    "permissionGroupCreate": null
  }
}
```

---

## 六、用户权限汇总链路：从 Group 到 has_perm

### 6.1 完整链路图

```
用户发起请求
    │
    ▼
JWT 认证 → 加载 User 对象
    │
    ▼
权限检查调用 user.has_perm("account.manage_staff")
    │
    ├─► 1. User.has_perm()
    │     ├─► 检查 is_superuser
    │     └─► 调用 _user_has_perm()
    │
    ├─► 2. _user_has_perm()
    │     └─► 遍历认证后端，调用 backend.has_perm()
    │
    ├─► 3. JSONWebTokenBackend.has_perm()
    │     ├─► 检查 user.is_active
    │     └─► 调用 get_all_permissions()
    │
    ├─► 4. get_all_permissions()
    │     ├─► get_user_permissions() → _get_permissions("user")
    │     └─► get_group_permissions() → _get_permissions("group")
    │
    └─► 5. _get_permissions()
          ├─► 检查缓存 _effective_permissions_cache
          └─► 调用 user.effective_permissions
                │
                ├─► 5.1 检查缓存 _effective_permissions
                ├─► 5.2 构建权限 QuerySet
                │     ├─► 用户直接权限 (user_permissions)
                │     └─► 用户所属组权限 (groups__permissions)
                └─► 5.3 返回合并后的权限 QuerySet
```

### 6.2 effective_permissions 核心实现

**文件位置：** `saleor/account/models.py:247-288`

```python
@property
def effective_permissions(self) -> models.QuerySet[Permission]:
    if self._effective_permissions is None:
        # 1. 获取所有可用权限的基础 QuerySet
        self._effective_permissions = get_permissions()
        
        if not self.is_superuser:
            # 2. 关联表：用户-权限 中间表
            UserPermission = User.user_permissions.through
            user_permission_queryset = UserPermission._default_manager.filter(
                user_id=self.pk
            ).values("permission_id")

            # 3. 关联表：用户-组 和 组-权限 中间表
            UserGroup = User.groups.through
            GroupPermission = Group.permissions.through
            
            # 🔑 关键：找出用户所属的所有组
            user_group_queryset = UserGroup._default_manager.filter(
                user_id=self.pk
            ).values("group_id")
            
            # 🔑 关键：找出这些组拥有的所有权限
            group_permission_queryset = GroupPermission.objects.filter(
                Exists(user_group_queryset.filter(group_id=OuterRef("group_id")))
            ).values("permission_id")

            # 4. 合并权限：用户直接权限 OR 组权限
            self._effective_permissions = self._effective_permissions.filter(
                Q(
                    Exists(
                        user_permission_queryset.filter(
                            permission_id=OuterRef("pk")
                        )
                    )
                )
                | Q(
                    Exists(
                        group_permission_queryset.filter(
                            permission_id=OuterRef("pk")
                        )
                    )
                )
            )
    return self._effective_permissions
```

### 6.3 组权限汇总 SQL 逻辑

```sql
-- 伪代码：查询用户的所有有效权限
SELECT p.*
FROM permission p
WHERE 
  -- 用户直接拥有该权限
  EXISTS (
    SELECT 1 
    FROM user_user_permissions uup 
    WHERE uup.user_id = ? AND uup.permission_id = p.id
  )
  OR
  -- 用户所属的某个组拥有该权限
  EXISTS (
    SELECT 1 
    FROM group_permissions gp
    WHERE gp.permission_id = p.id
    AND EXISTS (
      SELECT 1 
      FROM user_groups ug 
      WHERE ug.user_id = ? AND ug.group_id = gp.group_id
    )
  )
```

### 6.4 后端权限缓存机制

**文件位置：** `saleor/core/auth_backend.py:67-82`

```python
def _get_permissions(self, user_obj, obj, from_name):
    """Return the permissions of `user_obj` from `from_name`."""
    if not user_obj.is_active or user_obj.is_anonymous or obj is not None:
        return set()

    perm_cache_name = "_effective_permissions_cache"
    if not getattr(user_obj, perm_cache_name, None):
        # 从 effective_permissions QuerySet 转换为字符串集合
        perms = getattr(self, f"_get_{from_name}_permissions")(user_obj)
        perms = perms.using(settings.DATABASE_CONNECTION_REPLICA_NAME)
        perms = perms.values_list("content_type__app_label", "codename").order_by()
        # 缓存格式：{"account.manage_staff", "order.manage_orders", ...}
        setattr(user_obj, perm_cache_name, {f"{ct}.{name}" for ct, name in perms})
    return getattr(user_obj, perm_cache_name)
```

### 6.5 has_perms 多权限检查

**User.has_perms()：** `saleor/account/models.py:312-320`

```python
def has_perms(
    self, perm_list: Iterable[BasePermissionEnum | str], obj=None
) -> bool:
    # 转换为字符串列表
    perm_list = [
        perm.value if isinstance(perm, BasePermissionEnum) else perm
        for perm in perm_list
    ]
    # 🔑 调用 Django 父类方法，逐个调用 has_perm
    return super().has_perms(perm_list, obj)
```

**PermissionsMixin.has_perms()：** `saleor/permission/models.py:158-163`

```python
def has_perms(self, perm_list, obj=None):
    """Return True if the user has each of the specified permissions."""
    # 🔑 AND 逻辑：必须拥有所有权限
    return all(self.has_perm(perm, obj) for perm in perm_list)
```

---

## 七、拒绝路径触发点和返回结果汇总

### 7.1 拒绝触发点一览

| 层级 | 触发点 | 代码位置 | 异常类型 | 涉及权限类型 |
|------|--------|---------|---------|-------------|
| 字段级 | `one_of_permissions_required` 装饰器 | `saleor/graphql/decorators.py:84-89` | `PermissionDenied` | 常规权限 + AuthorizationFilters |
| Mutation 级 | `BaseMutation.check_permissions` | `saleor/graphql/core/mutations.py` | `PermissionDenied` | 常规权限 + AuthorizationFilters |
| 授权过滤器 | `_get_result_of_authorization_filters_checks` | `saleor/permission/utils.py:74-77` | 间接（返回 False） | `AUTHENTICATED_APP` / `AUTHENTICATED_USER` / `AUTHENTICATED_STAFF_USER` |
| 业务逻辑 | `check_is_owner_or_has_one_of_perms` | `saleor/graphql/account/utils.py:389-394` | `PermissionDenied` | 常规权限 + `OWNER` |
| 业务逻辑 | `has_required_permission` | `saleor/graphql/app/types.py:86-89` | `PermissionDenied` | 常规权限 + `OWNER` |
| App 限制 | `PermissionGroupCreate.check_permissions` | `saleor/graphql/account/mutations/permission_group/permission_group_create.py:140-148` | `PermissionDenied` | 自定义消息 |
| 业务逻辑 | `ensure_requestor_can_manage_group` | `saleor/graphql/account/mutations/permission_group/permission_group_update.py:120-136` | `ValidationError` | 权限范围校验 |
| 业务逻辑 | `ensure_can_manage_permissions` | `saleor/graphql/account/mutations/permission_group/permission_group_create.py:150-168` | `ValidationError` | 权限范围校验 |
| 字段级 | `CustomerEvent.resolve_user` | `saleor/graphql/account/types.py:240-256` | `PermissionDenied` | 常规权限 |

### 7.2 AuthorizationFilters 拒绝触发路径详解

#### 7.2.1 AUTHENTICATED_APP 拒绝路径

**触发条件：** 请求来自未认证的 App 或用户，`permissions = [AUTHENTICATED_APP]`

```python
# 1. 装饰器调用
one_of_permissions_required([AUTHENTICATED_APP])
    ↓
# 2. 核心校验
one_of_permissions_or_auth_filter_required(context, [AUTHENTICATED_APP])
    ├─► perm_results = []  # 过滤后无常规权限
    └─► auth_filters_results
          └─► _get_result_of_authorization_filters_checks
                ├─► resolve_authorization_filter_fn(AUTHENTICATED_APP) → is_app
                └─► is_app(context) → False
    ↓
# 3. 结果计算
any([]) or any([False]) = False
    ↓
# 4. 抛出异常
raise PermissionDenied(permissions=[AUTHENTICATED_APP])
```

**返回消息：** `"To access this path, you need one of the following permissions: AUTHENTICATED_APP"`

#### 7.2.2 OWNER 拒绝路径

**触发条件：** 请求者既不是资源所有者，也没有所需的常规权限

```python
# 业务逻辑层显式调用
check_is_owner_or_has_one_of_perms(requestor, owner, MANAGE_APPS)
    ↓
# 1. 检查所有权
requestor == owner → False
    ↓
# 2. 检查常规权限
has_one_of_permissions(requestor, (MANAGE_APPS,)) → False
    ↓
# 3. 抛出异常（注意权限列表包含 OWNER）
raise PermissionDenied(permissions=[MANAGE_APPS, OWNER])
```

**返回消息：** `"To access this path, you need one of the following permissions: MANAGE_APPS, OWNER"`

### 7.3 PermissionDenied 返回格式

```json
{
  "errors": [
    {
      "message": "To access this path, you need one of the following permissions: MANAGE_STAFF",
      "locations": [{"line": 2, "column": 3}],
      "path": ["permissionGroups"],
      "extensions": {
        "exception": {
          "code": "PermissionDenied"
        }
      }
    }
  ],
  "data": {
    "permissionGroups": null
  }
}
```

### 7.4 ValidationError（业务逻辑校验失败）返回格式

```json
{
  "errors": [],
  "data": {
    "permissionGroupUpdate": {
      "errors": [
        {
          "field": "addPermissions",
          "message": "You can't add permission that you don't have.",
          "code": "OUT_OF_SCOPE_PERMISSION",
          "permissions": ["MANAGE_APPS"]
        }
      ],
      "group": null
    }
  }
}
```

---

## 八、两条校验链并排对比：查询守卫链 vs 业务逻辑链

### 8.1 核心架构对比图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        查询守卫链（声明式）                              │
├─────────────────────────────────────────────────────────────────────────┤
│  触发点：PermissionsField / BaseMutation.check_permissions              │
│  核心函数：one_of_permissions_or_auth_filter_required                  │
│  权限检查：直接调用 requestor.has_perm(perm)                            │
│  授权过滤器：is_app / is_user / is_staff_user 内置函数                  │
│  OWNER 处理：跳过（无内置函数）                                          │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        业务逻辑链（程序化）                              │
├─────────────────────────────────────────────────────────────────────────┤
│  触发点：resolver / clean_input 内部显式调用                            │
│  核心函数：has_one_of_permissions / check_is_owner_or_has_one_of_perms  │
│  权限检查：通过 permission_required 间接调用 has_perm                   │
│  OWNER 处理：显式判断 requestor == owner                                │
│  MANAGE_STAFF：App 被特殊拦截！                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 查询守卫链：one_of_permissions_or_auth_filter_required

**完整调用链：**

```
BaseMutation.mutate()
    ↓
check_permissions()  [saleor/graphql/core/mutations.py:498-514]
    ↓
one_of_permissions_or_auth_filter_required(context, permissions)
    ├─► _get_result_of_permissions_checks(context, permissions)
    │     ├─► 过滤掉 AuthorizationFilters
    │     ├─► get_user_or_app_from_context(context) → requestor
    │     └─► [requestor.has_perm(perm) for perm in permissions]  ← 直接调用 has_perm
    │
    └─► _get_result_of_authorization_filters_checks(context, permissions)
          ├─► 只提取 AuthorizationFilters
          ├─► resolve_authorization_filter_fn(perm) → 检查函数
          └─► [perm_fn(context) for perm in auth_filters]
    ↓
any(perm_results) or any(auth_filters_results)  ← OR 逻辑
```

**关键特征：**
- 直接调用 `App.has_perm(perm)` 检查权限
- **不经过** `permission_required` 函数
- **不会**对 MANAGE_STAFF 做特殊拦截

### 8.3 业务逻辑链：has_one_of_permissions

**完整调用链：**

```
check_is_owner_or_has_one_of_perms(requestor, owner, *perms)
    ↓
is_owner_or_has_one_of_perms(requestor, owner, *perms)
    ├─► requestor == owner  ← OWNER 判断
    └─► has_one_of_permissions(requestor, perms)  ← 业务逻辑链入口
          ↓
          for perm in permissions:
              permission_required(requestor, (perm,))  ← 关键！经过此函数
                  ↓
                  if isinstance(requestor, User):
                      return requestor.has_perms(perms)
                  if requestor:  ← App 分支
                      # 🔴 MANAGE_STAFF 特殊拦截
                      if AccountPermissions.MANAGE_STAFF in perms:
                          return False  ← App 永远无法通过！
                      return requestor.has_perms(perms)
                  return False
```

### 8.4 MANAGE_STAFF 语义下 App 请求的差异详解

#### 8.4.1 差异根源：两条链使用不同的检查函数

| 检查路径 | 核心函数 | MANAGE_STAFF 对 App 的行为 |
|---------|---------|--------------------------|
| 查询守卫链 | `one_of_permissions_or_auth_filter_required` → `requestor.has_perm()` | ✅ 正常检查 App 的权限表 |
| 业务逻辑链 | `has_one_of_permissions` → `permission_required` | ❌ 直接返回 `False`，强制拦截 |

#### 8.4.2 permission_required 的特殊拦截代码

**文件位置：** `saleor/permission/utils.py:82-94`

```python
def permission_required(
    requestor: Union["User", "App", None], perms: Iterable[BasePermissionEnum]
) -> bool:
    from ..account.models import User

    if isinstance(requestor, User):
        return requestor.has_perms(perms)
    if requestor:
        # 🔴 关键注释：for now MANAGE_STAFF permission for app is not supported
        if AccountPermissions.MANAGE_STAFF in perms:
            return False  # 直接拦截，不检查 App 的权限表
        return requestor.has_perms(perms)
    return False
```

#### 8.4.3 实际场景示例

**场景 1：App 拥有 MANAGE_STAFF 权限，调用 PermissionGroup Mutation**

```python
# Mutation 层检查（查询守卫链）
BaseMutation.check_permissions()
    → one_of_permissions_or_auth_filter_required(context, [MANAGE_STAFF])
    → app.has_perm(MANAGE_STAFF) → True (App 表中确实有这个权限)
    → ✅ 通过！

# PermissionGroupCreate 额外检查（业务逻辑链）
PermissionGroupCreate.check_permissions()
    → if app:
          raise PermissionDenied(
              message="Apps are not allowed to perform this mutation."
          )  # 🔴 直接拦截，自定义消息
```

**场景 2：App 拥有 MANAGE_STAFF 权限，调用 check_is_owner_or_has_one_of_perms**

```python
# 业务逻辑链
check_is_owner_or_has_one_of_perms(app, owner, MANAGE_STAFF)
    → requestor == owner → False (App 不是所有者)
    → has_one_of_permissions(app, [MANAGE_STAFF])
        → permission_required(app, (MANAGE_STAFF,))
            → MANAGE_STAFF in perms → True
            → return False  # 🔴 强制拦截！
    → raise PermissionDenied(permissions=[MANAGE_STAFF, OWNER])
```

#### 8.4.4 设计意图分析

这种不对称设计的原因：
1. **权限组管理是敏感操作**：涉及员工权限分配，通常只允许人类管理员操作
2. **App 权限模型设计限制**：App 主要用于集成，不应该管理人类员工的权限
3. **两层防护**：
   - 第一层：`permission_required` 中硬编码拦截（通用防护）
   - 第二层：具体 Mutation 中再次检查并自定义错误消息（特定防护）

---

## 九、嵌套字段拒绝时的返回格式详解

### 9.1 嵌套字段权限控制示例

**文件位置：** `saleor/graphql/account/types.py:240-256`

```python
class CustomerEvent(ModelObjectType[models.CustomerEvent]):
    @staticmethod
    def resolve_user(root: models.CustomerEvent, info: ResolveInfo):
        user = info.context.user
        user = cast(User, user)
        if (
            user == root.user  # OWNER 判断
            or user.has_perm(AccountPermissions.MANAGE_USERS)
            or user.has_perm(AccountPermissions.MANAGE_STAFF)
        ):
            return root.user
        # 🔴 嵌套字段拒绝
        raise PermissionDenied(
            permissions=[
                AccountPermissions.MANAGE_STAFF,
                AccountPermissions.MANAGE_USERS,
                AuthorizationFilters.OWNER,
            ]
        )
```

### 9.2 嵌套字段拒绝的返回格式

**GraphQL 查询：**
```graphql
query {
  me {
    events(first: 10) {
      edges {
        node {
          id
          user {  # 🔴 这个嵌套字段会被拒绝
            id
            email
          }
        }
      }
    }
  }
}
```

**返回结果：**
```json
{
  "errors": [
    {
      "message": "To access this path, you need one of the following permissions: MANAGE_STAFF, MANAGE_USERS, OWNER",
      "locations": [{"line": 6, "column": 11}],
      "path": ["me", "events", "edges", 0, "node", "user"],  // 🔑 完整路径！
      "extensions": {
        "exception": {
          "code": "PermissionDenied"
        }
      }
    }
  ],
  "data": {
    "me": {
      "events": {
        "edges": [
          {
            "node": {
              "id": "Q3VzdG9tZXJFdmVudDox",
              "user": null  // 🔑 嵌套字段为 null，外层数据正常返回
            }
          }
        ]
      }
    }
  }
}
```

### 9.3 关键特征说明

| 特征 | 说明 |
|------|------|
| `errors.path` | 包含从根到被拒绝字段的完整路径，数组下标表示列表索引 |
| 局部 `null` | 被拒绝的字段返回 `null`，不影响其他字段和外层数据 |
| 部分成功 | 查询整体成功（HTTP 200），只是部分字段被拒绝 |
| 错误定位 | `locations` 指向 GraphQL 查询文档中具体的字段位置 |

### 9.4 多层嵌套拒绝示例

**查询：**
```graphql
query {
  app(id: "QXBwOjE=") {
    id
    name
    privateMeta {  # 第一层嵌套，可能被拒绝
      key
      value
    }
    extensions {  # 第二层嵌套
      id
      app {  # 第三层嵌套，可能被拒绝
        id
        token
      }
    }
  }
}
```

**返回结果（两个嵌套字段都被拒绝）：**
```json
{
  "errors": [
    {
      "message": "To access this path, you need one of the following permissions: MANAGE_APPS, OWNER",
      "path": ["app", "privateMeta"],
      "extensions": {"code": "PermissionDenied"}
    },
    {
      "message": "To access this path, you need one of the following permissions: OWNER",
      "path": ["app", "extensions", 0, "app"],
      "extensions": {"code": "PermissionDenied"}
    }
  ],
  "data": {
    "app": {
      "id": "QXBwOjE=",
      "name": "My App",
      "privateMeta": null,  // 第一层嵌套被拒绝
      "extensions": [
        {
          "id": "QXBwRXh0ZW5zaW9uOjE=",
          "app": null  // 第三层嵌套被拒绝
        }
      ]
    }
  }
}
```

---

## 十、权限组创建的校验调用顺序与短路点

### 10.1 PermissionGroupCreate 完整调用栈

以 `PermissionGroupCreate` 为例，App 请求的完整校验流程如下：

```
mutation PermissionGroupCreate($input: PermissionGroupCreateInput!) {
  permissionGroupCreate(input: $input) {
    group { id name }
    permissionGroupErrors { field message code }
  }
}
        ↓
BaseMutation.mutate()  [saleor/graphql/core/mutations.py:517-531]
        ├─► 第 522 行：if not cls.check_permissions(info.context, data=data):
        │           raise PermissionDenied(permissions=cls._meta.permissions)
        │
        └─► PermissionGroupCreate.check_permissions()  ← 🔴 override 方法！
                ├─► 第 143 行：app = get_app_promise(context).get()
                │    ├─► if app:  ← 🔴 短路点！不调用父类校验
                │    │       raise PermissionDenied(
                │    │           message="Apps are not allowed to perform this mutation."
                │    │       )
                │    └─► 只有非 App 请求才继续：
                │           return super().check_permissions(context, permissions)
                │                   ↓
                │           BaseMutation.check_permissions()
                │                   ├─► 提取 all_permissions = (MANAGE_STAFF,)
                │                   └─► one_of_permissions_or_auth_filter_required()
                │                           └─► user.has_perm(MANAGE_STAFF)
                │
                └─► clean_input()  ← 校验通过后才执行
                      ├─► clean_channels()
                      ├─► clean_permissions()
                      │     └─► ensure_can_manage_permissions()
                      └─► clean_users()
                            └─► ensure_users_are_staff()
```

### 10.2 App 请求的短路点位置详解

#### 短路点 1：PermissionGroupCreate.check_permissions 第 144-147 行

**文件位置：** `saleor/graphql/account/mutations/permission_group/permission_group_create.py:140-148`

```python
@classmethod
def check_permissions(
    cls, context, permissions=None, require_all_permissions=False, **data
):
    app = get_app_promise(context).get()  # 🔴 第 143 行
    if app:  # 🔴 第 144 行 - 短路条件
        # 🔴 短路：直接抛出异常，不调用 super()
        raise PermissionDenied(
            message="Apps are not allowed to perform this mutation."
        )
    # 只有非 App 请求才会执行到这里
    return super().check_permissions(context, permissions)
```

**关键特征：**
- **短路时机**：在 `super().check_permissions()` 之前
- **触发条件**：`get_app_promise(context).get()` 返回非 None 值
- **短路效果**：完全跳过父类的 MANAGE_STAFF 权限检查
- **相同逻辑**：`PermissionGroupDelete.check_permissions` 也有完全相同的短路逻辑

#### 短路点 2：permission_required 函数（业务逻辑链）

**文件位置：** `saleor/permission/utils.py:82-94`

```python
def permission_required(
    requestor: Union["User", "App", None], perms: Iterable[BasePermissionEnum]
) -> bool:
    if isinstance(requestor, User):
        return requestor.has_perms(perms)
    if requestor:
        # 🔴 MANAGE_STAFF 硬编码拦截
        if AccountPermissions.MANAGE_STAFF in perms:
            return False  # 🔴 短路：不检查 App 的权限表
        return requestor.has_perms(perms)
    return False
```

**短路时机**：在调用 `requestor.has_perms(perms)` 之前

### 10.3 三类校验的触发条件与错误返回对比

| 校验类型 | 触发点 | 触发条件 | 异常类型 | 错误消息示例 |
|---------|--------|---------|---------|-------------|
| **BaseMutation 通用校验** | `BaseMutation.check_permissions` → `one_of_permissions_or_auth_filter_required` | `cls._meta.permissions` 非空，且请求者无对应权限 | `PermissionDenied` | `"To access this path, you need one of the following permissions: MANAGE_STAFF"` |
| **Override check_permissions 短路** | `PermissionGroupCreate.check_permissions` 第 144 行 | `get_app_promise(context).get()` 返回 App | `PermissionDenied` | `"Apps are not allowed to perform this mutation." |
| **OWNER helper 校验** | `check_is_owner_or_has_one_of_perms` | `requestor != owner` 且 `!has_one_of_permissions(requestor, perms)` | `PermissionDenied` | `"To access this path, you need one of the following permissions: MANAGE_APPS, OWNER"` |

### 10.4 三类校验的详细对比

#### 对比 1：BaseMutation 通用校验

**触发时机**：`BaseMutation.mutate()` 第 522 行调用 `cls.check_permissions()`

**适用场景**：所有继承 `BaseMutation` 的类，通过 `Meta.permissions` 声明权限

**核心代码**：
```python
# saleor/graphql/core/mutations.py:518-523
def mutate(cls, root, info: ResolveInfo, **data):
    if not cls.check_permissions(info.context, data=data):
        # 🔴 通用校验失败时，使用 cls._meta.permissions 构造错误
        raise PermissionDenied(permissions=cls._meta.permissions)
```

**错误返回格式**：
```json
{
  "errors": [
    {
      "message": "To access this path, you need one of the following permissions: MANAGE_STAFF",
      "path": ["permissionGroupCreate"],
      "extensions": {
        "exception": {
          "code": "PermissionDenied"
        }
      }
    }
  ],
  "data": {
    "permissionGroupCreate": null
  }
}
```

**关键特征**：
- 错误消息自动生成，包含所有 `Meta.permissions` 的枚举名称
- `permissions` 参数传入的是 `cls._meta.permissions`
- 用于通用的、基于声明的权限检查

---

#### 对比 2：Override check_permissions 短路

**触发时机**：子类重写 `check_permissions` 方法，在调用 `super()` 之前

**适用场景**：需要对特定请求者类型（如 App）做额外限制的 Mutation

**核心代码**：
```python
# saleor/graphql/account/mutations/permission_group/permission_group_create.py:140-148
def check_permissions(cls, context, permissions=None, require_all_permissions=False, **data):
    app = get_app_promise(context).get()
    if app:
        # 🔴 自定义错误消息，不使用自动生成的消息
        raise PermissionDenied(
            message="Apps are not allowed to perform this mutation."
        )
    return super().check_permissions(context, permissions)
```

**错误返回格式**：
```json
{
  "errors": [
    {
      "message": "Apps are not allowed to perform this mutation.",
      "path": ["permissionGroupCreate"],
      "extensions": {
        "exception": {
          "code": "PermissionDenied"
        }
      }
    }
  ],
  "data": {
    "permissionGroupCreate": null
  }
}
```

**关键特征**：
- 自定义错误消息，更具描述性
- 不传入 `permissions` 参数给 `PermissionDenied`
- 短路 `super()` 调用，父类的通用校验完全不执行
- 同类 Mutation：`PermissionGroupDelete.check_permissions` 也有相同逻辑

---

#### 对比 3：OWNER helper 校验

**触发时机**：resolver 或业务逻辑中显式调用 `check_is_owner_or_has_one_of_perms`

**适用场景**：需要判断请求者是否为资源所有者或拥有特定权限的场景

**核心代码**：
```python
# saleor/graphql/account/utils.py:379-394
def check_is_owner_or_has_one_of_perms(
    requestor: Union["User", "App", None], owner: Optional["User"], *perms
) -> None:
    if not is_owner_or_has_one_of_perms(requestor, owner, *perms):
        # 🔴 权限列表 = 常规权限 + OWNER
        raise PermissionDenied(permissions=list(perms) + [AuthorizationFilters.OWNER])
```

**使用示例**：
```python
# saleor/graphql/account/types.py:259-264
def resolve_app(root: models.CustomerEvent, info: ResolveInfo):
    requestor = get_user_or_app_from_context(info.context)
    check_is_owner_or_has_one_of_perms(
        requestor, root.user, AppPermission.MANAGE_APPS
    )
    return AppByIdLoader(info.context).load(root.app_id) if root.app_id else None
```

**错误返回格式**：
```json
{
  "errors": [
    {
      "message": "To access this path, you need one of the following permissions: MANAGE_APPS, OWNER",
      "path": ["customerEvent", "app"],
      "extensions": {
        "exception": {
          "code": "PermissionDenied"
        }
      }
    }
  ],
  "data": {
    "customerEvent": {
      "app": null
    }
  }
}
```

**关键特征**：
- 错误消息自动生成，包含常规权限 + `OWNER`
- `permissions` 参数包含 `AuthorizationFilters.OWNER`
- 用于业务逻辑层的所有权判断
- 经过业务逻辑链，会触发 `permission_required` 中的 MANAGE_STAFF 拦截

### 10.5 三类校验的执行顺序总结

以 App 请求 `PermissionGroupCreate` 为例：

```
时间线：
T1: BaseMutation.mutate() 开始执行
T2: 调用 cls.check_permissions() → 实际调用 PermissionGroupCreate.check_permissions()
T3: ├─► get_app_promise(context).get() → 返回 App 对象
T4: ├─► if app: → True，进入短路分支
T5: └─► raise PermissionDenied(message="Apps are not allowed...")
    ← 在此处终止，后续的 super().check_permissions() 和 clean_input() 都不会执行
```

以无 MANAGE_STAFF 权限的用户请求 `PermissionGroupCreate` 为例：

```
时间线：
T1: BaseMutation.mutate() 开始执行
T2: 调用 cls.check_permissions() → PermissionGroupCreate.check_permissions()
T3: ├─► get_app_promise(context).get() → None
T4: ├─► 不进入短路分支
T5: └─► return super().check_permissions(context, permissions)
            ↓
T6:       BaseMutation.check_permissions()
            ├─► all_permissions = (MANAGE_STAFF,)
            └─► one_of_permissions_or_auth_filter_required()
                    └─► user.has_perm(MANAGE_STAFF) → False
            ↓
T7:       return False
            ↓
T8: BaseMutation.mutate() 第 523 行：
    raise PermissionDenied(permissions=cls._meta.permissions)
    ← 在此处终止，clean_input() 不会执行
```

以通过 `check_is_owner_or_has_one_of_perms` 校验的场景为例：

```
时间线：
T1: resolver 开始执行
T2: 调用 check_is_owner_or_has_one_of_perms(requestor, owner, MANAGE_APPS)
T3: ├─► requestor == owner → False
T4: └─► has_one_of_permissions(requestor, [MANAGE_APPS])
          └─► permission_required(requestor, (MANAGE_APPS,))
                  ├─► 不是 MANAGE_STAFF，不触发硬编码拦截
                  └─► requestor.has_perms([MANAGE_APPS]) → False
          ↓
T5:    return False
        ↓
T6: raise PermissionDenied(permissions=[MANAGE_APPS, OWNER])
    ← 在此处终止，resolver 后续逻辑不执行
```

---

## 十一、查询接口守卫：Query 与 Mutation 的权限声明

### 11.1 Query 接口守卫

**文件位置：** `saleor/graphql/account/schema.py:163-181`

```python
class AccountQueries(graphene.ObjectType):
    # 🔴 权限组列表：需要 MANAGE_STAFF 权限
    permission_groups = FilterConnectionField(
        GroupCountableConnection,
        filter=PermissionGroupFilterInput(),
        sort_by=PermissionGroupSortingInput(),
        description="List of permission groups.",
        permissions=[AccountPermissions.MANAGE_STAFF],  # 🔑 权限声明
        doc_category=DOC_CATEGORY_USERS,
    )
    
    # 🔴 单个权限组：需要 MANAGE_STAFF 权限
    permission_group = PermissionsField(
        Group,
        id=graphene.ID(required=True),
        description="Look up permission group by ID.",
        permissions=[AccountPermissions.MANAGE_STAFF],  # 🔑 权限声明
        doc_category=DOC_CATEGORY_USERS,
    )
```

### 11.2 Mutation 接口守卫

**文件位置：** `saleor/graphql/core/mutations.py:158-212`

```python
class BaseMutation(graphene.Mutation):
    @classmethod
    def __init_subclass_with_meta__(
        cls,
        permissions: Collection[BasePermissionEnum] | None = None,
        auto_permission_message=True,
        **options,
    ):
        # 验证权限格式
        cls._validate_permissions(permissions)
        
        _meta.permissions = permissions
        
        # 自动在 description 后追加权限说明
        if permissions and auto_permission_message:
            permissions_msg = message_one_of_permissions_required(permissions)
            description = f"{description} {permissions_msg}"
        
        super().__init_subclass_with_meta__(description=description, _meta=_meta, **options)
```

### 11.3 Mutation 权限检查时机

**文件位置：** `saleor/graphql/account/mutations/permission_group/permission_group_create.py:140-148`

```python
class PermissionGroupCreate(DeprecatedModelMutation):
    class Meta:
        description = "Create new permission group."
        permissions = (AccountPermissions.MANAGE_STAFF,)  # 🔑 权限声明
    
    @classmethod
    def check_permissions(
        cls, context, permissions=None, require_all_permissions=False, **data
    ):
        # 🔴 额外校验：App 不允许创建权限组
        app = get_app_promise(context).get()
        if app:
            raise PermissionDenied(message="Apps are not allowed to perform this mutation.")
        return super().check_permissions(context, permissions)
```

---

## 十一、权限组管理的四层校验

以 `PermissionGroupUpdate` 为例，权限校验分为四层：

### 第一层：Mutation 级权限声明

**文件位置：** `saleor/graphql/account/mutations/permission_group/permission_group_update.py:57-78`

```python
class PermissionGroupUpdate(PermissionGroupCreate):
    class Meta:
        description = "Update permission group."
        permissions = (AccountPermissions.MANAGE_STAFF,)  # 🔑 第一层校验
```

### 第二层：请求者能否管理该组

**文件位置：** `saleor/graphql/account/mutations/permission_group/permission_group_update.py:99-136`

```python
@classmethod
def clean_input(cls, info: ResolveInfo, instance, data, **kwargs):
    requestor = info.context.user
    # 🔑 第二层校验：确保请求者有权管理目标组
    cls.ensure_requestor_can_manage_group(info, requestor, instance)
    # ... 其他校验
```

### 第三层：权限范围校验

**文件位置：** `saleor/graphql/account/utils.py:77-98`

```python
def can_user_manage_group(info, user: "User", group: Group) -> bool:
    """User can't manage a group with permission or channel that is out of his scope."""
    return (
        can_user_manage_group_permissions(user, group)  # 权限范围
        and 
        can_user_manage_group_channels(info, user, group)  # 渠道范围
    )

def can_user_manage_group_permissions(user: "User", group: Group) -> bool:
    """用户必须拥有目标组的所有权限才能管理它"""
    permissions = get_group_permission_codes(group)
    return user.has_perms(permissions)  # 注意：这里是 has_perms (全部拥有)
```

### 第四层：操作内容校验

**文件位置：** `saleor/graphql/account/mutations/permission_group/permission_group_create.py:150-168`

```python
@classmethod
def ensure_can_manage_permissions(
    cls, requestor: "User", errors: dict, field: str, permission_items: list[str]
):
    """Check if requestor can manage permissions from input.
    
    Requestor cannot manage permissions witch he doesn't have.
    """
    missing_permissions = get_out_of_scope_permissions(requestor, permission_items)
    if missing_permissions:
        # 🔑 不能添加自己没有的权限
        error_msg = "You can't add permission that you don't have."
        code = PermissionGroupErrorCode.OUT_OF_SCOPE_PERMISSION.value
        params = {"permissions": missing_permissions}
        cls.update_errors(errors, error_msg, field, code, params)
```

---

## 十二、权限枚举定义

**文件位置：** `saleor/permission/enums.py:1-100`

```python
class BasePermissionEnum(Enum):
    @property
    def codename(self):
        return self.value.split(".")[1]

class AccountPermissions(BasePermissionEnum):
    MANAGE_USERS = "account.manage_users"      # 管理客户
    MANAGE_STAFF = "account.manage_staff"      # 管理员工（权限组）
    IMPERSONATE_USER = "account.impersonate_user"

class OrderPermissions(BasePermissionEnum):
    MANAGE_ORDERS = "order.manage_orders"
    MANAGE_ORDERS_IMPORT = "order.manage_orders_import"

# ... 其他权限枚举
```

---

## 十三、完整调用链路示例

### 示例 1：查询 permission_groups 接口（无权限用户）

```
客户端请求（未认证）
    │
    ▼
GraphQL 执行引擎
    │
    ▼
FilterConnectionField (继承 PermissionsField)
    │
    ▼
get_resolver() 被调用
    │
    ├─► 提取 permissions=[AccountPermissions.MANAGE_STAFF]
    │
    ▼
one_of_permissions_required() 装饰器应用
    │
    ▼
请求到达时
    │
    ▼
account_passes_test 包装函数
    │
    ├─► 从 args 中提取 info.context
    │
    ▼
one_of_permissions_or_auth_filter_required()
    │
    ├─► get_user_or_app_from_context(context) → None
    ├─► perm_checks_results = []
    └─► any([]) = False
    │
    ▼
抛出 PermissionDenied(permissions=[MANAGE_STAFF])
    │
    ▼
GraphQL 错误格式化
    │
    ▼
返回错误响应
```

### 示例 2：User.has_perm() 完整调用链

```python
user.has_perm(AccountPermissions.MANAGE_STAFF)
    │
    ├─► perm = "account.manage_staff"
    │
    ├─► 检查：is_active and is_superuser and not _effective_permissions
    │     └─► 如果是超级用户且未被覆盖，直接返回 True
    │
    ▼
_user_has_perm(user, "account.manage_staff", None)
    │
    ├─► 遍历 auth.get_backends()
    │
    ├─► JSONWebTokenBackend.has_perm(user, "account.manage_staff")
    │     │
    │     ├─► 检查 user.is_active
    │     │
    │     └─► BaseBackend.has_perm()
    │           │
    │           └─► "account.manage_staff" in get_all_permissions(user)
    │                 │
    │                 ├─► get_user_permissions(user)
    │                 │     └─► _get_permissions(user, None, "user")
    │                 │           │
    │                 │           ├─► 检查缓存 _effective_permissions_cache
    │                 │           └─► user.effective_permissions
    │                 │                 │
    │                 │                 ├─► 构建 QuerySet：
    │                 │                 │     ├─► 用户直接权限
    │                 │                 │     └─► 组权限（通过 groups 关联）
    │                 │                 │
    │                 │                 └─► 转换为 {"account.manage_staff", ...}
    │                 │
    │                 └─► get_group_permissions(user)
    │                       └─► 同上（也返回 effective_permissions）
    │
    └─► 返回 True/False
```

### 示例 3：执行 PermissionGroupUpdate mutation

```
mutation 调用
    │
    ▼
BaseMutation.mutate()
    │
    ├─► check_permissions() 检查 MANAGE_STAFF
    │     ├─► 检查是否为 App → 如果是，直接拒绝
    │     └─► 调用 user.has_perm(MANAGE_STAFF)
    │
    └─► clean_input()
          │
          ├─► ensure_requestor_can_manage_group()
          │    ├─► can_user_manage_group_permissions()
          │    │     └─► user.has_perms(group_permissions)
          │    └─► can_user_manage_group_channels()
          │
          ├─► clean_permissions()
          │    └─► ensure_can_manage_permissions()
          │         └─► 不能添加自己没有的权限
          │
          ├─► clean_users()
          │    ├─► ensure_users_are_staff()
          │    └─► ensure_can_manage_users()
          │
          └─► clean_channels()
               └─► ensure_can_manage_channels()
    │
    ▼
save() 执行
```

---

## 十四、关键设计模式总结

### 14.1 声明式 vs 程序化

| 方式 | 适用场景 | 示例 |
|------|---------|------|
| **声明式** | 简单的权限要求 | `PermissionsField(permissions=[...])` |
| **程序化** | 复杂业务逻辑 | `can_user_manage_group()` |

### 14.2 OR 逻辑 vs AND 逻辑

| 校验层次 | 逻辑 | 说明 |
|---------|------|------|
| 字段/接口级权限 | **OR** | 拥有任一权限即可访问 |
| 权限组管理权限范围 | **AND** | 必须拥有目标组的**所有**权限才能管理 |

### 14.3 关键安全原则

1. **最小权限原则**：用户不能授予自己没有的权限
2. **范围封闭原则**：不能管理超出自己权限范围的组
3. **不可自毁原则**：不能把自己从最后一个组中移除
4. **权限连续性原则**：操作后必须仍有人能管理这些权限

### 14.4 缓存机制

| 缓存位置 | 作用 | 代码位置 |
|---------|------|---------|
| `user._effective_permissions` | 缓存权限 QuerySet | `saleor/account/models.py:240, 285` |
| `user._effective_permissions_cache` | 缓存权限字符串集合 | `saleor/core/auth_backend.py:76-81` |
| `app._app_perm_cache` | 缓存 App 权限字符串集合 | `saleor/app/models.py:102-106` |

---

## 十五、常见问题解答

### Q: 为什么需要两层权限检查（字段级 + 业务逻辑级）？

**A:** 
- 字段级检查是**通用的、粗粒度**的访问控制
- 业务逻辑级检查是**具体的、细粒度**的范围控制
- 例如：`MANAGE_STAFF` 只表示你可以管理权限组，但具体能管理**哪些**组，需要业务逻辑进一步判断

### Q: has_perm() vs has_perms() 有什么区别？

**A:**
- `has_perm(perm)`: 检查是否拥有**单个**权限（用于 OR 逻辑）
- `has_perms(perms)`: 检查是否拥有**所有**权限（用于 AND 逻辑）
- 在 `one_of_permissions_or_auth_filter_required` 中是 OR 逻辑
- 在 `can_user_manage_group_permissions` 中是 AND 逻辑

### Q: App 为什么不能创建权限组？

**A:** 
- 权限组管理属于**超级用户级**操作
- App 的权限范围通常受限
- 代码中显式判断：`PermissionGroupCreate.check_permissions()` 中如果是 App 直接抛出异常

### Q: 未认证用户和无权限用户的返回结果有区别吗？

**A:** 
- 从 GraphQL 响应格式来看，两者都返回 `PermissionDenied` 错误
- 但在内部逻辑中：
  - 未认证用户：`requestor is None` → `perm_checks_results = []` → `any([]) = False`
  - 无权限用户：`requestor.has_perm(perm)` 逐个返回 `False` → `any([False, False]) = False`
- 错误消息相同，都是提示需要哪些权限

### Q: 权限是如何从 Group 汇总到 User 的？

**A:** 
1. 用户与组通过 `user_groups` 中间表关联
2. 组与权限通过 `group_permissions` 中间表关联
3. `effective_permissions` 使用 `EXISTS` 子查询合并用户直接权限和组权限
4. 最终转换为 `{"app_label.codename", ...}` 字符串集合进行快速查找

### Q: AuthorizationFilters.OWNER 为什么在查询守卫层不生效？

**A:**
- `OWNER` 在 `AUTHORIZATION_FILTER_MAP` 中没有对应的检查函数
- 因为所有权判断依赖于具体的业务对象（如订单、用户资料），查询守卫层不知道当前访问的是哪个对象
- 必须在业务逻辑层通过 `check_is_owner_or_has_one_of_perms(requestor, owner, *perms)` 显式调用
- 拒绝时权限列表会同时包含常规权限和 `OWNER`，例如 `[MANAGE_APPS, OWNER]`

### Q: one_of_permissions_or_auth_filter_required 和 all_permissions_required 有什么区别？

**A:**
- `one_of_permissions_or_auth_filter_required`：**OR 逻辑**，常规权限任一通过 OR 授权过滤器任一通过
- `all_permissions_required`：**AND + OR 混合逻辑**，所有常规权限都通过 AND 任一授权过滤器通过
- 前者用于"只要满足一个条件即可"的场景，后者用于"必须满足特定身份 + 所有权限"的场景

### Q: 当 permissions 同时包含常规权限和 AuthorizationFilters 时，错误消息怎么显示？

**A:**
- `PermissionDenied` 会遍历所有传入的权限枚举，调用 `.name` 属性拼接
- 例如 `permissions = [MANAGE_APPS, OWNER]` 会显示：
  `"To access this path, you need one of the following permissions: MANAGE_APPS, OWNER"`
- 枚举的 `.name` 是大写的，如 `OWNER`、`AUTHENTICATED_APP`，而不是枚举值字符串

### Q: 为什么 App 拥有 MANAGE_STAFF 权限却被拒绝？两条校验链有什么差异？

**A:**
这是因为 Saleor 有两条独立的权限校验链，对 MANAGE_STAFF 的处理不同：

1. **查询守卫链**：`one_of_permissions_or_auth_filter_required` → 直接调用 `app.has_perm()`，正常检查 App 的权限表
2. **业务逻辑链**：`has_one_of_permissions` → `permission_required` → 硬编码拦截 MANAGE_STAFF

关键代码在 `saleor/permission/utils.py:90-92`：
```python
if AccountPermissions.MANAGE_STAFF in perms:
    return False  # App 永远无法通过业务逻辑链！
```

**设计意图**：权限组管理是敏感操作，只允许人类管理员操作，App 不应该管理员工权限。

### Q: 嵌套字段被拒绝时，为什么外层数据还能正常返回？

**A:**
这是 GraphQL 的标准错误处理机制：
- 单个 resolver 抛出异常时，只会影响该字段的返回值（设为 null）
- `errors` 数组中会记录具体错误，包含完整的 `path` 路径定位被拒绝的字段
- HTTP 状态码仍为 200，表示查询"部分成功"
- 客户端可以根据 `errors.path` 精确定位哪个字段出了问题

### Q: PermissionGroupCreate 重写 check_permissions 的短路点在哪里？为什么不调用父类？

**A:**
短路点在 `saleor/graphql/account/mutations/permission_group/permission_group_create.py:144`：
```python
app = get_app_promise(context).get()
if app:
    raise PermissionDenied(
        message="Apps are not allowed to perform this mutation."
    )
return super().check_permissions(context, permissions)
```

**为什么不先调用父类：**
- 这是一种优化：如果 App 请求直接被禁止，就不需要再检查 MANAGE_STAFF 权限
- 同时也避免了 App 拥有 MANAGE_STAFF 权限时可能出现的逻辑漏洞
- 设计原则：先做最严格的拦截，再做通用校验

### Q: BaseMutation 通用校验、override 短路、OWNER helper 三种方式的错误返回有什么不同？

**A:**
主要差异体现在三个方面：

| 差异点 | BaseMutation 通用校验 | Override 短路 | OWNER helper |
|--------|---------------------|--------------|-------------|
| **错误消息** | 自动生成，列出所有需要的权限 | 自定义，更具描述性 | 自动生成，包含常规权限 + OWNER |
| **permissions 参数** | 传入 `cls._meta.permissions` | 不传入 | 传入 `list(perms) + [OWNER]` |
| **触发时机** | `super().check_permissions()` 返回 False 时 | `app` 不为 None 时 | `requestor != owner` 且无权限时 |
| **消息示例** | `"To access this path, you need one of the following permissions: MANAGE_STAFF"` | `"Apps are not allowed to perform this mutation."` | `"To access this path, you need one of the following permissions: MANAGE_APPS, OWNER"` |
