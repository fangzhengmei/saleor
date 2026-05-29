# Saleor 员工权限组与查询接口守卫关系详解

## 一、整体架构概览

Saleor 的权限系统采用**三层防护机制**：字段级访问控制、查询接口守卫、业务逻辑校验。它们之间的关系如下：

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

---

## 五、查询接口守卫：Query 与 Mutation 的权限声明

### 5.1 Query 接口守卫

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

### 5.2 Mutation 接口守卫

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

### 5.3 Mutation 权限检查时机

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

## 六、权限组管理的四层校验

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

## 七、权限枚举定义

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

## 八、完整调用链路示例

### 示例 1：查询 permission_groups 接口

```
客户端请求
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
    ├─► get_user_or_app_from_context(context)
    ├─► 对每个权限调用 requestor.has_perm(perm)
    └─► 任一返回 True 则通过
    │
    ▼
校验通过 → 执行实际 resolver
校验失败 → 抛出 PermissionDenied
```

### 示例 2：执行 PermissionGroupUpdate mutation

```
mutation 调用
    │
    ▼
BaseMutation.mutate()
    │
    ├─► check_permissions() 检查 MANAGE_STAFF
    └─► clean_input()
          │
          ├─► ensure_requestor_can_manage_group()
          │    ├─► 检查权限范围
          │    └─► 检查渠道范围
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

## 九、关键设计模式总结

### 9.1 声明式 vs 程序化

| 方式 | 适用场景 | 示例 |
|------|---------|------|
| **声明式** | 简单的权限要求 | `PermissionsField(permissions=[...])` |
| **程序化** | 复杂业务逻辑 | `can_user_manage_group()` |

### 9.2 OR 逻辑 vs AND 逻辑

| 校验层次 | 逻辑 | 说明 |
|---------|------|------|
| 字段/接口级权限 | **OR** | 拥有任一权限即可访问 |
| 权限组管理权限范围 | **AND** | 必须拥有目标组的**所有**权限才能管理 |

### 9.3 关键安全原则

1. **最小权限原则**：用户不能授予自己没有的权限
2. **范围封闭原则**：不能管理超出自己权限范围的组
3. **不可自毁原则**：不能把自己从最后一个组中移除
4. **权限连续性原则**：操作后必须仍有人能管理这些权限

---

## 十、常见问题解答

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
