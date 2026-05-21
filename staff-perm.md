# 员工账号与权限分组协作路径分析

## 一、权限分组建模

### 1.1 核心数据模型

#### Group 模型 (`saleor/account/models.py:396-434`)

```python
class Group(models.Model):
    name = models.CharField("name", max_length=150, unique=True)
    permissions = models.ManyToManyField(Permission, blank=True)
    restricted_access_to_channels = models.BooleanField(default=False)
    channels = models.ManyToManyField("channel.Channel", blank=True)
```

**字段说明：**
- `name`: 权限组名称，全局唯一
- `permissions`: 多对多关联 `Permission` 模型，存储该组拥有的权限集合
- `restricted_access_to_channels`: 是否限制渠道访问
- `channels`: 多对多关联 `Channel` 模型，存储该组可访问的渠道

#### User 模型与 Group 的关联 (`saleor/permission/models.py:90-119`)

```python
class PermissionsMixin(models.Model):
    is_superuser = models.BooleanField(default=False)
    groups = models.ManyToManyField(
        "account.Group",
        blank=True,
        related_name="user_set",
        related_query_name="user",
    )
    user_permissions = models.ManyToManyField(
        Permission,
        blank=True,
        related_name="user_set",
        related_query_name="user",
    )
```

**关键关系：**
- User ↔ Group：多对多关系，通过 `User.groups` 访问用户所属组，反向通过 `Group.user_set` 访问组内用户
- Group ↔ Permission：多对多关系，通过 `Group.permissions` 访问组权限
- Group ↔ Channel：多对多关系，通过 `Group.channels` 访问组可访问渠道
- User ↔ Permission：多对多关系，通过 `User.user_permissions` 直接分配权限（不推荐使用，优先通过组分配）

### 1.2 权限计算模型（修正）

#### 有效权限计算 (`saleor/account/models.py:247-288`)

```python
@property
def effective_permissions(self) -> models.QuerySet[Permission]:
    if self._effective_permissions is None:
        self._effective_permissions = get_permissions()
        if not self.is_superuser:
            # 子查询1: 直接分配给用户的权限关联表
            UserPermission = User.user_permissions.through
            user_permission_queryset = UserPermission._default_manager.filter(
                user_id=self.pk
            ).values("permission_id")
            
            # 子查询2: 用户所属组的关联表
            UserGroup = User.groups.through
            user_group_queryset = UserGroup._default_manager.filter(
                user_id=self.pk
            ).values("group_id")
            
            # 子查询3: 组与权限的关联表
            GroupPermission = Group.permissions.through
            group_permission_queryset = GroupPermission.objects.filter(
                Exists(user_group_queryset.filter(group_id=OuterRef("group_id")))
            ).values("permission_id")
            
            # 通过两个 EXISTS 子查询的 OR 合并权限来源
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

**关键修正：**
1. **不是简单并集**：权限来源通过 `EXISTS (用户直接权限) OR EXISTS (组继承权限)` 的数据库级查询实现，不是 Python 层的集合并集
2. **两条独立路径**：
   - 路径 A：`User.user_permissions` 中间表 → `Permission`
   - 路径 B：`User.groups` 中间表 → `Group.permissions` 中间表 → `Permission`
3. **与 `get_all_permissions()` 的区别**：
   - `effective_permissions`：Saleor 自定义，基于子查询的 `QuerySet`，用于可管理性校验
   - `get_all_permissions()`：Django 标准方法，通过 auth backends 计算，返回 `set[str]`，用于权限可管理性校验中的权限收集
   - `get_not_manageable_permissions_when_deactivate_or_remove_users` 中使用的是 `user.get_all_permissions()`，不是 `effective_permissions`

**权限来源优先级（实际无优先级，是 OR 关系）：**
1. 超级用户 (`is_superuser=True`)：绕过所有校验，拥有所有权限
2. 非超级用户：`用户直接权限` **OR** `所属组继承权限`，两者满足其一即可

---

## 二、App 调用门禁（新增）

### 2.1 禁止 App 调用的 Mutation 列表

**所有 6 个员工与权限组相关的写操作 Mutation 都明确禁止 App 调用：**

| Mutation | 实现位置 | 门禁检查位置 |
|----------|----------|--------------|
| `StaffCreate` | `staff_create.py` | `check_permissions` 方法 (L88-97) |
| `StaffUpdate` | `staff_update.py` | 继承自 `StaffCreate` |
| `StaffDelete` | `staff_delete.py` | 通过 `StaffDeleteMixin.check_permissions` |
| `PermissionGroupCreate` | `permission_group_create.py` | `check_permissions` 方法 (L140-148) |
| `PermissionGroupUpdate` | `permission_group_update.py` | 继承自 `PermissionGroupCreate` |
| `PermissionGroupDelete` | `permission_group_delete.py` | `check_permissions` 方法 (L69-77) |

### 2.2 门禁实现机制

```python
# 以 PermissionGroupCreate.check_permissions 为例
@classmethod
def check_permissions(
    cls, context, permissions=None, require_all_permissions=False, **data
):
    app = get_app_promise(context).get()
    if app:
        raise PermissionDenied(
            message="Apps are not allowed to perform this mutation."
        )
    return super().check_permissions(context, permissions)
```

**关键细节：**
1. **检查时机**：在 `BaseMutation.mutate` (L516-531) 中调用，早于 `perform_mutation`
2. **实现方式**：通过 `get_app_promise(context).get()` 判断当前请求方是否为 App
3. **StaffDeleteMixin 额外门禁** (`base.py:436-447`)：
   ```python
   @classmethod
   def check_permissions(cls, context, permissions=None, ...):
       if get_app_promise(context).get():
           raise PermissionDenied(
               message="Apps are not allowed to perform this mutation."
           )
       return super().check_permissions(context, permissions)
   ```

**设计意图**：员工与权限组的管理属于最高级别的系统权限，仅允许人类管理员操作，不允许自动化 App 进行变更。

---

## 三、账号绑定校验逻辑

### 3.1 操作人权限校验（可管理性检查）

#### 核心校验函数

| 函数 | 位置 | 作用 |
|------|------|------|
| `get_groups_which_user_can_manage` | `saleor/graphql/account/utils.py:116-140` | 获取用户可管理的权限组列表 |
| `can_user_manage_group_permissions` | `saleor/graphql/account/utils.py:84-88` | 检查用户是否可管理组的权限 |
| `can_user_manage_group_channels` | `saleor/graphql/account/utils.py:90-98` | 检查用户是否可管理组的渠道 |
| `get_out_of_scope_users` | `saleor/graphql/account/utils.py:67-74` | 获取权限范围超出操作人的用户 |

#### 可管理组的判定逻辑 (`get_groups_which_user_can_manage`)

```python
def get_groups_which_user_can_manage(user: "User") -> list[Group]:
    if not user.is_staff:
        return []
    
    # 使用 effective_permissions 获取操作人的权限主键集合
    user_permission_pks = set(get_user_permissions(user).values_list("pk", flat=True))
    
    # 注解所有组的权限主键列表
    groups = Group.objects.all().annotate(group_perms=ArrayAgg("permissions"))
    
    editable_groups: list[Group] = []
    for group in groups.iterator(chunk_size=1000):
        # 组的权限必须是操作人权限的子集
        out_of_scope_permissions = set(group.group_perms) - user_permission_pks
        out_of_scope_permissions.discard(None)
        if not out_of_scope_permissions:
            editable_groups.append(group)
    
    return editable_groups
```

**关键原则：操作人只能管理权限范围 ≤ 自己的组**（子集关系）

### 3.2 员工创建时的组绑定校验 (`StaffCreate.clean_groups`)

**文件位置：** `saleor/graphql/account/mutations/staff/staff_create.py:127-157`

```python
@classmethod
def clean_groups(cls, requestor: models.User, cleaned_input: dict, errors: dict):
    if cleaned_input.get("add_groups"):
        cls.ensure_requestor_can_manage_groups(
            requestor, cleaned_input, "add_groups", errors
        )
```

### 3.3 员工更新时的组绑定校验 (`StaffUpdate.clean_groups`)

**文件位置：** `saleor/graphql/account/mutations/staff/staff_update.py:89-98`

```python
@classmethod
def clean_groups(cls, requestor: models.User, cleaned_input: dict, errors: dict):
    if cleaned_input.get("add_groups"):
        cls.ensure_requestor_can_manage_groups(
            requestor, cleaned_input, "add_groups", errors
        )
    if cleaned_input.get("remove_groups"):
        cls.ensure_requestor_can_manage_groups(
            requestor, cleaned_input, "remove_groups", errors
        )
```

**额外校验：**
- `check_for_duplicates(data, "add_groups", "remove_groups", "groups")`：防止同一组同时出现在添加和移除列表中
- 操作人不能管理自己范围外的用户 (`get_out_of_scope_users`)

### 3.4 add_users 只校验 staff 不校验可管理范围的设计原因（新增）

#### 权限组创建场景 (`PermissionGroupCreate.clean_users`)

**文件位置：** `saleor/graphql/account/mutations/permission_group/permission_group_create.py:171-198`

```python
@classmethod
def clean_users(cls, requestor: User, errors: dict, cleaned_input: dict, group: models.Group):
    user_items = cleaned_input.get("add_users")
    if user_items:
        cls.ensure_users_are_staff(errors, "add_users", cleaned_input)
```

#### 权限组更新场景 (`PermissionGroupUpdate.clean_users`)

**文件位置：** `saleor/graphql/account/mutations/permission_group/permission_group_update.py:206-219`

```python
@classmethod
def clean_users(cls, requestor: User, errors: dict, cleaned_input: dict, group: models.Group):
    super().clean_users(requestor, errors, cleaned_input, group)  # 只校验 staff
    remove_users = cleaned_input.get("remove_users")
    if remove_users:
        cls.ensure_can_manage_users(requestor, errors, "remove_users", cleaned_input)
        cls.clean_remove_users(requestor, errors, cleaned_input, group)
```

#### 设计原因分析

**为什么 add_users 只校验 is_staff，不校验可管理范围？**

| 角度 | 分析 |
|------|------|
| **权限传播方向** | 组是权限的"容器"，用户加入组只能获得组的权限，不会获得超出组范围的权限。由于组本身已通过 `ensure_requestor_can_manage_group()` 校验（组的权限 ≤ 操作人权限），因此向组内添加任何员工都不会导致权限泄露。 |
| **移除 vs 添加的区别** | 移除用户可能导致"权限不可管理"问题（某些权限无人可管理），需要校验；添加用户只会增强权限覆盖，不会引入不可管理风险。 |
| **设计意图** | 采用"组作为权限边界"的设计：只要操作人能管理这个组，就可以自由地向组内添加/移除员工（移除时额外校验不可管理风险）。 |
| **实际约束** | `ensure_requestor_can_manage_group()` 是第一道也是最关键的闸门，确保组本身在操作人的权限范围内。组内的用户操作被视为组内部的管理行为。 |

**校验矩阵对比：**

| 操作 | 校验 staff | 校验可管理范围 | 校验权限可管理性 |
|------|------------|----------------|------------------|
| add_users（创建） | ✅ | ❌ | ❌ |
| add_users（更新） | ✅ | ❌ | ❌ |
| remove_users（更新） | - | ✅ | ✅ |

### 3.5 渠道访问约束在创建与更新场景的分流逻辑（新增）

#### 创建场景 (`PermissionGroupCreate.clean_channels`, L201-234)

```python
@classmethod
def clean_channels(
    cls, info, group, user_accessible_channels, errors, cleaned_input
):
    user = info.context.user
    if cleaned_input.get("restricted_access_to_channels") is False:
        # 分支1: 不限制渠道访问
        if not user.is_superuser:
            # 非超级用户必须能访问系统所有渠道
            channel_ids = set(Channel.objects.values_list("id", flat=True))
            accessible_channel_ids = {c.id for c in user_accessible_channels}
            not_accessible_channels = set(channel_ids - accessible_channel_ids)
            if not_accessible_channels:
                raise ValidationError(
                    {"restricted_access_to_channels": ValidationError(
                        "You can't manage group with channels out of your scope.",
                        code=OUT_OF_SCOPE_CHANNEL,
                    )}
                )
        # 清空 add_channels，无限制时不绑定具体渠道
        cleaned_input["add_channels"] = []
    elif add_channels := cleaned_input.get("add_channels"):
        # 分支2: 限制渠道访问，校验每个渠道都在操作人可访问范围内
        cls.ensure_can_manage_channels(
            user, user_accessible_channels, errors, add_channels
        )
```

#### 更新场景 (`PermissionGroupUpdate.clean_channels`, L138-163)

```python
@classmethod
def clean_channels(
    cls, info, group, user_accessible_channels, errors, cleaned_input
):
    # 先调用父类处理 add_channels 的逻辑
    super().clean_channels(info, group, user_accessible_channels, errors, cleaned_input)
    
    # 更新场景新增: 校验 remove_channels 也在操作人可访问范围内
    if remove_channels := cleaned_input.get("remove_channels"):
        user = info.context.user
        cls.ensure_can_manage_channels(
            user, user_accessible_channels, errors, remove_channels
        )
    
    # 更新场景新增: 处理 restricted_access_to_channels 的状态切换
    restricted_access = cleaned_input.get("restricted_access_to_channels")
    if restricted_access is False or (
        restricted_access is None and group.restricted_access_to_channels is False
    ):
        # 状态为"不限制"或保持"不限制"时，清空所有渠道操作
        cleaned_input["add_channels"] = []
        cleaned_input["remove_channels"] = []
```

**创建 vs 更新 分流对比：**

| 维度 | 创建场景 | 更新场景 |
|------|----------|----------|
| `remove_channels` | 不存在此字段 | 需要校验操作人可管理被移除的渠道 |
| `restricted_access_to_channels` | 只处理显式 `False` | 处理显式 `False` **或** 原值为 `False` 且未修改 |
| 渠道操作清空范围 | 只清空 `add_channels` | 同时清空 `add_channels` 和 `remove_channels` |
| 组已有状态 | 无（新组） | 需要考虑组当前的 `restricted_access_to_channels` 值 |

**设计意图**：更新场景需要处理"保持原有不限制状态"的情况，此时即使用户传入了渠道操作，也应该被忽略。

---

## 四、变更生效的协作路径

### 4.1 save 与 _save_m2m 分段落库对一致性边界的影响（新增+修正）

#### 基类执行顺序 (`DeprecatedModelMutation.perform_mutation`, L814-861)

```python
@classmethod
def perform_mutation(cls, _root, info: ResolveInfo, /, **data):
    instance = cls.get_instance(info, **data)
    cleaned_input = cls.clean_input(info, instance, data.get("input"))
    # ... 元数据处理 ...
    instance = cls.construct_instance(instance, cleaned_input)
    cls.clean_instance(info, instance)
    
    # ============ 关键：两次独立落库 ============
    cls.save(info, instance, cleaned_input)           # 第1次落库 - 实例本身
    cls._save_m2m(info, instance, cleaned_input)      # 第2次落库 - 多对多关系
    # ============================================
    
    cls.post_save_action(info, instance, cleaned_input)
    return cls.success_response(instance)
```

#### 基类默认实现

```python
# save 默认实现 (L763-771)
@classmethod
def save(cls, _info, instance, _cleaned_input, /, instance_tracker=None):
    instance.save()  # 独立事务

# _save_m2m 默认实现 (L748-755)
@classmethod
def _save_m2m(cls, _info, instance, cleaned_data):
    opts = instance._meta
    for f in chain(opts.many_to_many, opts.private_fields):
        if not hasattr(f, "save_form_data"):
            continue
        if f.name in cleaned_data and cleaned_data[f.name] is not None:
            f.save_form_data(instance, cleaned_data[f.name])  # 又一个独立操作
```

#### 子类的事务增强

**所有子类都在 `_save_m2m` 中添加了 `traced_atomic_transaction`，但只包裹多对多操作：**

```python
# 以 StaffCreate._save_m2m 为例
@classmethod
def _save_m2m(cls, info: ResolveInfo, instance, cleaned_data):
    with traced_atomic_transaction():  # 只包裹多对多操作
        super()._save_m2m(info, instance, cleaned_data)
        groups = cleaned_data.get("add_groups")
        if groups:
            instance.groups.add(*groups)
```

#### 一致性边界分析

**问题：save() 和 _save_m2m() 是两个独立的数据库操作，不在同一个事务中**

**风险场景：**
1. `user.save()` 成功 → 用户已创建
2. `_save_m2m()` 失败 → 组关系未绑定
3. **结果**：系统中存在一个"裸奔"的员工账号，没有所属权限组

**各 Mutation 的不一致风险：**

| Mutation | 不一致状态 | 影响程度 |
|----------|------------|----------|
| `StaffCreate` | 用户已创建但未加入任何组 | ⚠️ 高 - 新员工无权限 |
| `StaffUpdate` | 用户基本信息已更新但组关系未变 | ⚠️ 中 - 权限变更未生效 |
| `PermissionGroupCreate` | 组已创建但未绑定权限/用户/渠道 | ⚠️ 高 - 空权限组 |
| `PermissionGroupUpdate` | 组基本信息已更新但关系未变 | ⚠️ 中 - 权限变更未生效 |
| `PermissionGroupDelete` | N/A - delete() 是原子操作 | ✅ 无 |
| `StaffDelete` | N/A - delete() 是原子操作 | ✅ 无 |

**现有防护措施：**
- 所有校验都在 `clean_input()` 阶段完成，`_save_m2m()` 阶段不做业务校验
- `_save_m2m()` 失败主要源于数据库层面的问题（如死锁、连接中断）

### 4.2 员工创建流程 (`StaffCreate.perform_mutation`)

**文件位置：** `saleor/graphql/account/mutations/staff/staff_create.py:225-251`

```
1. get_instance()
   ├─ 检查是否已存在非员工用户（可升级为员工）
   └─ 返回 (instance, send_notification)

2. clean_input()
   ├─ 校验 redirect_url
   ├─ 设置 is_staff = True
   ├─ clean_groups() → 校验操作人可管理的组
   └─ clean_is_active()

3. construct_instance() → 构建用户实例

4. validate_and_update_metadata() → 处理元数据

5. clean_instance() → 实例级校验

6. save()  [第1次落库 - 事务1]
   ├─ 更新搜索向量
   ├─ user.save() → 保存用户基本信息
   └─ 发送密码设置通知（如果需要）

7. _save_m2m()  [第2次落库 - 事务2, 组绑定生效点]
   └─ traced_atomic_transaction
       ├─ super()._save_m2m() → 保存其他多对多关系
       └─ instance.groups.add(*groups) → 【组绑定生效】

8. post_save_action()
   └─ call_event(manager.staff_created, instance) → 触发 webhook
```

### 4.3 StaffCreate 中通知发送与组关系写入的先后顺序及失败影响（新增）

#### 代码位置 (`staff_create.py:164-201`)

```python
@classmethod
def save(
    cls,
    info: ResolveInfo,
    user,
    cleaned_input,
    send_notification=True,
    redirect_url=None,
):
    if any(field in cleaned_input for field in USER_SEARCH_FIELDS):
        update_user_search_vector(user, attach_addresses_data=False, save=False)
    user.save()  # ✅ 用户保存成功
    redirect_url = cleaned_input.get("redirect_url")
    if redirect_url and send_notification:
        manager = get_plugin_manager_promise(info.context).get()
        send_set_password_notification(  # ⚠️ 通知发送在 save() 内
            redirect_url=redirect_url,
            user=user,
            manager=manager,
            channel_slug=None,
            staff=True,
        )
        token = token_generator.make_token(user)
        params = urlencode({"email": user.email, "token": token})
        cls.call_event(  # ⚠️ 密码重置请求事件也在 save() 内
            manager.staff_set_password_requested,
            user,
            None,
            token,
            prepare_url(params, redirect_url),
        )

@classmethod
def _save_m2m(cls, info: ResolveInfo, instance, cleaned_data):
    with traced_atomic_transaction():
        super()._save_m2m(info, instance, cleaned_data)
        groups = cleaned_data.get("add_groups")
        if groups:
            instance.groups.add(*groups)  # ❌ 组关系写入在后
```

#### 先后顺序总结

| 阶段 | 操作 | 位置 |
|------|------|------|
| 1 | `user.save()` → 用户落库 | `save()` 方法内 |
| 2 | `send_set_password_notification()` → 发送邮件 | `save()` 方法内（user.save() 之后） |
| 3 | `call_event(staff_set_password_requested)` → 触发事件 | `save()` 方法内 |
| 4 | `instance.groups.add(*groups)` → 组关系写入 | `_save_m2m()` 方法内 |
| 5 | `call_event(staff_created)` → 员工创建事件 | `post_save_action()` 内 |

#### 失败影响分析

**场景 1：通知发送失败**
- 触发点：`send_set_password_notification()` 抛出异常
- 影响范围：
  - ✅ `user.save()` 已成功 → 用户已存在
  - ❌ 通知发送失败 → `save()` 方法整体失败
  - ❌ `_save_m2m()` 不会执行 → 组关系**未写入**
  - ❌ `staff_created` 事件**未触发**
- **最终状态**：系统中存在一个没有权限组的员工账号，用户未收到设置密码邮件

**场景 2：组关系写入失败**
- 触发点：`instance.groups.add(*groups)` 抛出异常
- 影响范围：
  - ✅ `user.save()` 已成功 → 用户已存在
  - ✅ 通知已发送 → 用户收到了设置密码邮件
  - ✅ `staff_set_password_requested` 事件已触发
  - ❌ `_save_m2m()` 失败 → 组关系**未写入**
  - ❌ `staff_created` 事件**未触发**
- **最终状态**：用户收到了邮件，但账号没有权限组，无法登录执行操作

**场景 3：post_save_action 失败（罕见）**
- 触发点：`call_event(staff_created)` 异常
- 影响范围：
  - ✅ 用户已创建
  - ✅ 通知已发送
  - ✅ 组关系已写入
  - ❌ webhook 事件未触发
- **最终状态**：数据一致，但外部系统未收到通知

### 4.4 员工更新流程 (`StaffUpdate.perform_mutation`)

**文件位置：** `saleor/graphql/account/mutations/staff/staff_update.py:181-197`

```
1. get_instance() → 获取待更新用户

2. clean_input()
   ├─ 校验操作人可管理该用户（get_out_of_scope_users）
   ├─ check_for_duplicates(add_groups, remove_groups)
   └─ super().clean_input()
       ├─ clean_groups(add_groups + remove_groups)
       └─ clean_is_active()
           ├─ 不能停用自己
           ├─ 不能停用超级用户
           └─ 停用后不能导致权限无人可管理

3. save()  [第1次落库 - 事务1]
   └─ user.save() → 更新用户基本信息

4. _save_m2m()  [第2次落库 - 事务2, 组变更生效点]
   └─ traced_atomic_transaction
       ├─ super()._save_m2m()
       ├─ instance.groups.add(*add_groups) → 【添加组生效】
       └─ instance.groups.remove(*remove_groups) → 【移除组生效】

5. perform_mutation 后续处理
   ├─ 邮箱变更 → 重新关联礼品卡和订单
   └─ 邮箱/姓名变更 → 标记礼品卡搜索索引为脏
```

### 4.5 权限组创建流程 (`PermissionGroupCreate.perform_mutation`)

**文件位置：** `saleor/graphql/account/mutations/permission_group/permission_group_create.py`

```
1. clean_input()
   ├─ clean_channels() → 校验渠道访问权限
   ├─ clean_permissions() → 校验操作人拥有待分配的权限
   └─ clean_users() → 校验用户都是员工

2. save()  [第1次落库 - 事务1]
   └─ group.save() → 保存组基本信息

3. _save_m2m()  [第2次落库 - 事务2, 组关系生效点]
   └─ traced_atomic_transaction
       ├─ instance.permissions.add(*add_permissions) → 【权限绑定生效】
       ├─ instance.user_set.add(*users) → 【用户绑定生效】
       ├─ 无渠道限制时清空 channels
       └─ instance.channels.add(*channels) → 【渠道绑定生效】

4. post_save_action()
   └─ call_event(manager.permission_group_created, instance)
```

### 4.6 权限组更新流程 (`PermissionGroupUpdate.perform_mutation`)

**文件位置：** `saleor/graphql/account/mutations/permission_group/permission_group_update.py`

```
1. clean_input()
   ├─ ensure_requestor_can_manage_group() → 操作人可管理该组
   ├─ check_duplicates() → 三个维度的重复检查
   │   ├─ add_permissions ↔ remove_permissions
   │   ├─ add_users ↔ remove_users
   │   └─ add_channels ↔ remove_channels
   └─ super().clean_input()
       ├─ clean_channels() → add/remove 都校验
       ├─ clean_permissions() → add/remove 都校验
       │   └─ ensure_permissions_can_be_removed() → 移除后权限仍可管理
       └─ clean_users() → add/remove 都校验
           ├─ ensure_can_manage_users()
           ├─ check_if_removing_user_last_group()
           └─ check_if_users_can_be_removed()

2. save()  [第1次落库 - 事务1]
   └─ group.save() → 更新组基本信息

3. _save_m2m()  [第2次落库 - 事务2, 组变更生效点]
   └─ traced_atomic_transaction
       ├─ super()._save_m2m() → 处理 add_*
       ├─ instance.user_set.remove(*remove_users) → 【移除用户生效】
       ├─ instance.permissions.remove(*remove_permissions) → 【移除权限生效】
       └─ instance.channels.remove(*remove_channels) → 【移除渠道生效】

4. 失效 DataLoader 缓存
   └─ AccessibleChannelsByGroupIdLoader.clear(instance.id)
```

### 4.7 StaffUpdate 与 PermissionGroupUpdate 的事件触发顺序对比（新增）

#### 基类标准顺序 (`core/mutations.py:814-861`)

```
基类 perform_mutation 标准流程：
├─ save()               → 实例基本信息保存
├─ _save_m2m()          → 多对多关系保存
└─ post_save_action()   → 事件触发
```

#### StaffUpdate 的特殊扩展 (`staff_update.py:181-197`)

```python
@classmethod
def perform_mutation(cls, root, info: ResolveInfo, /, **data):
    original_instance, _ = cls.get_instance(info, **data)
    response = super().perform_mutation(root, info, **data)  # 先执行基类完整流程
    user = response.user
    has_new_email = user.email != original_instance.email
    has_new_name = original_instance.get_full_name() != user.get_full_name()

    if has_new_email:
        assign_user_gift_cards(user)      # ⚠️ 在 post_save_action 之后执行
        match_orders_with_new_user(user)   # ⚠️ 在 post_save_action 之后执行

    if has_new_email or has_new_name:
        if gift_cards := get_user_gift_cards(user):
            mark_gift_cards_search_index_as_dirty(gift_cards)  # ⚠️ 在 post_save_action 之后执行

    return response
```

#### PermissionGroupUpdate 的特殊扩展 (`permission_group_update.py:83-91`)

```python
@classmethod
def _save_m2m(cls, info: ResolveInfo, instance, cleaned_data):
    with traced_atomic_transaction():
        super()._save_m2m(info, instance, cleaned_data)
        if remove_users := cleaned_data.get("remove_users"):
            instance.user_set.remove(*remove_users)
        if remove_permissions := cleaned_data.get("remove_permissions"):
            instance.permissions.remove(*remove_permissions)
        if remove_channels := cleaned_data.get("remove_channels"):
            instance.channels.remove(*remove_channels)
    # Invalidate dataloader for group channels
    AccessibleChannelsByGroupIdLoader(info.context).clear(instance.id)  # ⚠️ 在 _save_m2m 内执行
```

#### 详细对比表

| 阶段 | StaffUpdate | PermissionGroupUpdate | 说明 |
|------|-------------|------------------------|------|
| **1. save()** | `user.save()` | `group.save()` | 实例基本信息保存，独立事务 |
| **2. _save_m2m()** | 组关系 add/remove + traced_atomic_transaction | 权限/用户/渠道 add/remove + traced_atomic_transaction + **DataLoader 缓存清理** | PermissionGroupUpdate 在事务结束后立即清理缓存 |
| **3. post_save_action()** | `call_event(staff_updated)` | `call_event(permission_group_updated)` | 标准 webhook 事件触发 |
| **4. 额外 side effects** | ✅ 重新关联礼品卡<br>✅ 匹配订单到新邮箱<br>✅ 标记礼品卡搜索索引为脏 | ❌ 无 | StaffUpdate 在基类 perform_mutation 返回后执行 |

#### 关键差异点

| 差异点 | StaffUpdate | PermissionGroupUpdate |
|--------|-------------|------------------------|
| **DataLoader 缓存清理时机** | 无（不需要） | `_save_m2m()` 内，事务结束后 |
| **Side Effect 位置** | `perform_mutation()` 末尾，**post_save_action 之后** | 无额外 side effect |
| **Side Effect 事务性** | 不在事务中，失败不影响主流程 | N/A |
| **事件触发前的一致性** | 事件触发时，用户组关系已更新 | 事件触发时，组关系已更新，缓存已清理 |

#### StaffUpdate Side Effect 的风险

**问题：** `assign_user_gift_cards`、`match_orders_with_new_user`、`mark_gift_cards_search_index_as_dirty` 都在 `post_save_action` **之后**执行，且不在事务中。

**风险场景：**
1. `staff_updated` 事件已触发 → 外部系统收到用户邮箱变更通知
2. `assign_user_gift_cards` 失败 → 礼品卡未重新关联
3. **结果**：外部系统与内部系统状态不一致

### 4.8 权限组删除流程 (`PermissionGroupDelete.perform_mutation`)

**文件位置：** `saleor/graphql/account/mutations/permission_group/permission_group_delete.py`

```
1. clean_instance()
   ├─ 操作人可管理该组的权限和渠道
   ├─ check_if_group_can_be_removed()
   │   ├─ ensure_deleting_not_left_not_manageable_permissions()
   │   └─ ensure_not_removing_requestor_last_group()
   └─ 不能删除自己的最后一个组

2. instance.delete() → 【组删除生效】
   └─ Django ORM 级联删除所有多对多关系（原子操作）

3. post_save_action()
   └─ call_event(manager.permission_group_deleted, instance)
```

### 4.9 员工删除流程 (`StaffDelete.perform_mutation`)

**文件位置：** `saleor/graphql/account/mutations/staff/staff_delete.py` + `base.py:431-522`

```
1. clean_instance()  [StaffDeleteMixin]
   ├─ check_if_users_can_be_deleted()
   │   ├─ 只能删除员工（is_staff=True）
   │   ├─ 不能删除自己
   │   └─ 不能删除超级用户
   ├─ check_if_requestor_can_manage_users() → 操作人可管理该用户
   └─ check_if_removing_left_not_manageable_permissions()

2. instance.delete() → 【用户删除生效】
   └─ Django ORM 级联删除组关联关系（原子操作）

3. post_save_action()
   └─ call_event(manager.staff_deleted, instance)
```

---

## 五、多 Mutation 协作与交叉校验（修正）

### 5.1 "权限可管理性"全局约束（修正）

**核心函数：** `get_not_manageable_permissions_*` 系列函数

**约束原则（修正）：** 任何变更后，每个权限至少需要通过以下两种方式之一可管理：
- **方式 A（组级）**：存在一个组，该组同时拥有 `MANAGE_STAFF` 权限和该权限，且组内至少有一个活跃员工
- **方式 B（用户级）**：存在一个活跃员工，该员工拥有 `MANAGE_STAFF` 权限，且该员工所属的某个组拥有该权限

**关键修正**：不是"一个员工同时拥有 MANAGE_STAFF 和该权限"，而是"MANAGE_STAFF 的持有者（组或用户）能够通过组关系覆盖到该权限"。

**校验场景分布：**

| 场景 | 调用位置 | 校验函数 |
|------|----------|----------|
| 停用员工 | `StaffUpdate.clean_is_active` | `get_not_manageable_permissions_when_deactivate_or_remove_users` |
| 删除员工 | `StaffDeleteMixin.clean_instance` | `get_not_manageable_permissions_when_deactivate_or_remove_users` |
| 从组移除权限 | `PermissionGroupUpdate.clean_permissions` | `get_not_manageable_permissions_after_removing_perms_from_group` |
| 从组移除用户 | `PermissionGroupUpdate.clean_remove_users` | `get_not_manageable_permissions_after_removing_users_from_group` |
| 删除权限组 | `PermissionGroupDelete.clean_instance` | `get_not_manageable_permissions_after_group_deleting` |

### 5.2 校验算法详解 (`get_not_manageable_permissions`)（修正）

**文件位置：** `saleor/graphql/account/utils.py:245-271`

```
算法输入：
- groups_data: 所有组的映射 {group_pk: {"permissions": set(), "users": set()}}
- not_manageable_permissions: 待校验的权限集合

算法流程：
1. 第一阶段扫描（组级扫描）：
   get_users_and_look_for_permissions_in_groups_with_manage_staff()
   ├─ 遍历所有组
   ├─ 筛选条件：组拥有 MANAGE_STAFF 权限 且 组内有活跃用户
   ├─ 对符合条件的组：
   │   ├─ 在该组的 permissions 中查找 not_manageable_permissions
   │   ├─ 找到的权限从 not_manageable_permissions 中移除（组级可管理）
   │   └─ 收集该组的所有用户到 manage_staff_users 集合
   └─ 返回 manage_staff_users（所有在"有 MANAGE_STAFF 组"中的用户）

2. 快速终止检查：
   ├─ 如果 not_manageable_permissions 已空 → 返回空集（全部可管理）
   └─ 如果 manage_staff_users 为空 → 返回剩余权限（无人能管理）

3. 第二阶段扫描（用户级扫描）：
   look_for_permission_in_users_with_manage_staff()
   ├─ 遍历所有组
   ├─ 筛选条件：组内有用户属于 manage_staff_users（即用户在某个有 MANAGE_STAFF 的组中）
   ├─ 对符合条件的组：
   │   └─ 在该组的 permissions 中查找剩余的 not_manageable_permissions
   │   └─ 找到的权限从 not_manageable_permissions 中移除（用户级可管理）
   └─ 无返回值，直接修改 not_manageable_permissions

4. 返回 not_manageable_permissions（剩余的就是不可管理的权限）
```

**算法举例：**
```
假设：
- 组A: 权限 {MANAGE_STAFF, P1}, 用户 {U1}
- 组B: 权限 {P2}, 用户 {U1}
- 待校验权限: {P1, P2, P3}

第一阶段扫描：
- 组A有 MANAGE_STAFF，在组权限中找到 P1，移除
- not_manageable_permissions = {P2, P3}
- manage_staff_users = {U1}

第二阶段扫描：
- 组B的用户 {U1} 在 manage_staff_users 中，在组权限中找到 P2，移除
- not_manageable_permissions = {P3}

返回 {P3} → P3 不可管理
```

### 5.3 数据流与状态一致性

```
权限变更请求
    ↓
┌─────────────────────────────────────────┐
│  BaseMutation.mutate()                  │
│  └─ check_permissions()                 │
│     ├─ App 门禁检查（6 个 Mutation）    │
│     └─ MANAGE_STAFF 权限检查            │
    ↓
┌─────────────────────────────────────────┐
│  clean_input() - 输入校验阶段            │
│  ├─ 操作人权限范围检查                   │
│  ├─ 目标资源可管理性检查                 │
│  ├─ 重复/冲突输入检查                    │
│  └─ 变更后全局约束检查（权限可管理性）   │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  save() - 第1次落库（独立事务）          │
│  └─ 实例本身保存（User/Group）           │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  _save_m2m() - 第2次落库（独立事务）     │
│  └─ traced_atomic_transaction           │
│     ├─ 多对多关系 add/remove            │
│     └─ 权限组更新时清空 DataLoader 缓存 │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  post_save_action() - 事件通知阶段       │
│  └─ call_event(manager.*_created/updated│
│              /deleted, instance)        │
└─────────────────────────────────────────┘
    ↓
[StaffUpdate 特有] 邮箱/姓名变更 Side Effects
    ↓
Webhook 事件触发 → 外部系统同步
```

**一致性边界说明：**
- ✅ `clean_input()` 阶段的所有校验在内存中完成，原子性有保障
- ⚠️ `save()` 和 `_save_m2m()` 是两个独立事务，存在中间不一致窗口
- ✅ `_save_m2m()` 内部的多对多操作在同一个 `traced_atomic_transaction` 中
- ✅ 删除操作（`instance.delete()`）由 Django ORM 保证原子性
- ⚠️ StaffUpdate 的 Side Effects 在 `post_save_action()` 之后执行，不在事务中

---

## 六、关键代码位置索引

### 6.1 模型层
| 内容 | 文件位置 |
|------|----------|
| User 模型 | `saleor/account/models.py:139-327` |
| Group 模型 | `saleor/account/models.py:396-434` |
| PermissionsMixin（groups/user_permissions 定义） | `saleor/permission/models.py:90-119` |
| effective_permissions 计算 | `saleor/account/models.py:247-288` |

### 6.2 工具函数
| 内容 | 文件位置 |
|------|----------|
| 可管理组获取 | `saleor/graphql/account/utils.py:116-140` |
| 权限可管理性校验算法 | `saleor/graphql/account/utils.py:245-271` |
| 组数据映射构建 | `saleor/graphql/account/utils.py:274-309` |
| 停用/删除用户时的权限校验 | `saleor/graphql/account/utils.py:143-184` |

### 6.3 Mutation 基类
| 内容 | 文件位置 |
|------|----------|
| perform_mutation 执行顺序 | `saleor/graphql/core/mutations.py:814-861` |
| save 默认实现 | `saleor/graphql/core/mutations.py:763-771` |
| _save_m2m 默认实现 | `saleor/graphql/core/mutations.py:748-755` |

### 6.4 Mutation 层
| 内容 | 文件位置 |
|------|----------|
| StaffCreate | `saleor/graphql/account/mutations/staff/staff_create.py` |
| StaffUpdate | `saleor/graphql/account/mutations/staff/staff_update.py` |
| StaffDelete | `saleor/graphql/account/mutations/staff/staff_delete.py` |
| StaffDeleteMixin | `saleor/graphql/account/mutations/base.py:431-522` |
| PermissionGroupCreate | `saleor/graphql/account/mutations/permission_group/permission_group_create.py` |
| PermissionGroupUpdate | `saleor/graphql/account/mutations/permission_group/permission_group_update.py` |
| PermissionGroupDelete | `saleor/graphql/account/mutations/permission_group/permission_group_delete.py` |

### 6.5 错误码
| 错误码 | 场景 |
|--------|------|
| `OUT_OF_SCOPE_GROUP` | 尝试管理权限范围外的组 |
| `OUT_OF_SCOPE_USER` | 尝试管理权限范围外的用户 |
| `OUT_OF_SCOPE_PERMISSION` | 尝试分配自己没有的权限 |
| `OUT_OF_SCOPE_CHANNEL` | 尝试分配自己无访问权的渠道 |
| `ASSIGN_NON_STAFF_MEMBER` | 尝试将非员工加入权限组 |
| `LEFT_NOT_MANAGEABLE_PERMISSION` | 变更将导致某些权限无人可管理 |
| `CANNOT_REMOVE_FROM_LAST_GROUP` | 不能从最后一个组中移除自己 |
| `DUPLICATED_INPUT_ITEM` | 同一项目同时出现在 add/remove 列表 |
| `DEACTIVATE_OWN_ACCOUNT` | 不能停用自己的账号 |
| `DEACTIVATE_SUPERUSER_ACCOUNT` | 不能停用超级用户 |
| `DELETE_OWN_ACCOUNT` | 不能删除自己的账号 |
| `DELETE_SUPERUSER_ACCOUNT` | 不能删除超级用户 |
| `DELETE_NON_STAFF_USER` | 不能删除非员工用户 |
