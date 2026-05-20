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

#### User 模型与 Group 的关联 (`saleor/account/models.py:139-141`)

```python
class User(PermissionsMixin, ModelWithMetadata, AbstractBaseUser, ModelWithExternalReference):
    # 通过 Django 的 PermissionsMixin 继承 groups 字段
    # groups = models.ManyToManyField(Group, related_name="user_set")
```

**关键关系：**
- User ↔ Group：多对多关系，通过 `User.groups` 访问用户所属组
- Group ↔ Permission：多对多关系，通过 `Group.permissions` 访问组权限
- Group ↔ Channel：多对多关系，通过 `Group.channels` 访问组可访问渠道

### 1.2 权限计算模型

#### 有效权限计算 (`saleor/account/models.py:247-288`)

```python
@property
def effective_permissions(self) -> models.QuerySet[Permission]:
    if self._effective_permissions is None:
        self._effective_permissions = get_permissions()
        if not self.is_superuser:
            # 1. 直接分配给用户的权限
            UserPermission = User.user_permissions.through
            user_permission_queryset = UserPermission._default_manager.filter(
                user_id=self.pk
            ).values("permission_id")
            
            # 2. 通过用户所属组继承的权限
            UserGroup = User.groups.through
            GroupPermission = Group.permissions.through
            user_group_queryset = UserGroup._default_manager.filter(
                user_id=self.pk
            ).values("group_id")
            group_permission_queryset = GroupPermission.objects.filter(
                Exists(user_group_queryset.filter(group_id=OuterRef("group_id")))
            ).values("permission_id")
            
            # 合并两种权限来源
            self._effective_permissions = self._effective_permissions.filter(
                Q(Exists(user_permission_queryset.filter(permission_id=OuterRef("pk"))))
                | Q(Exists(group_permission_queryset.filter(permission_id=OuterRef("pk"))))
            )
    return self._effective_permissions
```

**权限来源优先级：**
1. 超级用户 (`is_superuser=True`)：拥有所有权限
2. 用户直接分配的权限 (`user_permissions`)
3. 用户所属组继承的权限 (`groups → permissions`)

---

## 二、账号绑定校验逻辑

### 2.1 操作人权限校验（可管理性检查）

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
    
    user_permission_pks = set(get_user_permissions(user).values_list("pk", flat=True))
    
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

**关键原则：操作人只能管理权限范围小于等于自己的组**

### 2.2 员工创建时的组绑定校验 (`StaffCreate.clean_groups`)

**文件位置：** `saleor/graphql/account/mutations/staff/staff_create.py:127-157`

```python
@classmethod
def clean_groups(cls, requestor: models.User, cleaned_input: dict, errors: dict):
    if cleaned_input.get("add_groups"):
        cls.ensure_requestor_can_manage_groups(
            requestor, cleaned_input, "add_groups", errors
        )

@classmethod
def ensure_requestor_can_manage_groups(
    cls, requestor: models.User, cleaned_input: dict, field: str, errors: dict
):
    if requestor.is_superuser:
        return
    groups = cleaned_input[field]
    user_editable_groups = get_groups_which_user_can_manage(requestor)
    out_of_scope_groups = set(groups) - set(user_editable_groups)
    if out_of_scope_groups:
        # 抛出 OUT_OF_SCOPE_GROUP 错误
```

### 2.3 员工更新时的组绑定校验 (`StaffUpdate.clean_groups`)

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

### 2.4 权限组创建时的用户绑定校验 (`PermissionGroupCreate.clean_users`)

**文件位置：** `saleor/graphql/account/mutations/permission_group/permission_group_create.py:171-198`

```python
@classmethod
def clean_users(cls, requestor: User, errors: dict, cleaned_input: dict, group: models.Group):
    user_items = cleaned_input.get("add_users")
    if user_items:
        cls.ensure_users_are_staff(errors, "add_users", cleaned_input)

@classmethod
def ensure_users_are_staff(cls, errors: dict, field: str, cleaned_input: dict):
    users = cleaned_input[field]
    non_staff_users = [user.pk for user in users if not user.is_staff]
    if non_staff_users:
        # 抛出 ASSIGN_NON_STAFF_MEMBER 错误
```

### 2.5 权限组更新时的用户绑定校验 (`PermissionGroupUpdate.clean_users`)

**文件位置：** `saleor/graphql/account/mutations/permission_group/permission_group_update.py:206-219`

```python
@classmethod
def clean_users(cls, requestor: User, errors: dict, cleaned_input: dict, group: models.Group):
    super().clean_users(requestor, errors, cleaned_input, group)
    remove_users = cleaned_input.get("remove_users")
    if remove_users:
        cls.ensure_can_manage_users(requestor, errors, "remove_users", cleaned_input)
        cls.clean_remove_users(requestor, errors, cleaned_input, group)
```

**移除用户时的额外校验：**
1. `ensure_can_manage_users`：操作人必须能管理被移除的用户
2. `check_if_removing_user_last_group`：不能将自己从最后一个组中移除
3. `check_if_users_can_be_removed`：移除后不能导致某些权限无人可管理

---

## 三、变更生效的协作路径

### 3.1 变更原子性保障

所有多对多关系变更都使用 `traced_atomic_transaction` 包装：

```python
@classmethod
def _save_m2m(cls, info: ResolveInfo, instance, cleaned_data):
    with traced_atomic_transaction():
        super()._save_m2m(info, instance, cleaned_data)
        # 执行多对多关系变更
```

### 3.2 员工创建流程 (`StaffCreate.perform_mutation`)

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

6. save()
   ├─ 更新搜索向量
   ├─ user.save() → 保存用户基本信息
   └─ 发送密码设置通知（如果需要）

7. _save_m2m()  [关键：组绑定生效点]
   └─ traced_atomic_transaction
       ├─ super()._save_m2m() → 保存其他多对多关系
       └─ instance.groups.add(*groups) → 【组绑定生效】

8. post_save_action()
   └─ call_event(manager.staff_created, instance) → 触发 webhook
```

### 3.3 员工更新流程 (`StaffUpdate.perform_mutation`)

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

3. _save_m2m()  [关键：组变更生效点]
   └─ traced_atomic_transaction
       ├─ super()._save_m2m()
       ├─ instance.groups.add(*add_groups) → 【添加组生效】
       └─ instance.groups.remove(*remove_groups) → 【移除组生效】

4. perform_mutation 后续处理
   ├─ 邮箱变更 → 重新关联礼品卡和订单
   └─ 邮箱/姓名变更 → 标记礼品卡搜索索引为脏
```

### 3.4 权限组创建流程 (`PermissionGroupCreate.perform_mutation`)

**文件位置：** `saleor/graphql/account/mutations/permission_group/permission_group_create.py`

```
1. clean_input()
   ├─ clean_channels() → 校验渠道访问权限
   ├─ clean_permissions() → 校验操作人拥有待分配的权限
   └─ clean_users() → 校验用户都是员工

2. _save_m2m()  [关键：组关系生效点]
   └─ traced_atomic_transaction
       ├─ instance.permissions.add(*add_permissions) → 【权限绑定生效】
       ├─ instance.user_set.add(*users) → 【用户绑定生效】
       ├─ 无渠道限制时清空 channels
       └─ instance.channels.add(*channels) → 【渠道绑定生效】

3. post_save_action()
   └─ call_event(manager.permission_group_created, instance)
```

### 3.5 权限组更新流程 (`PermissionGroupUpdate.perform_mutation`)

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

2. _save_m2m()  [关键：组变更生效点]
   └─ traced_atomic_transaction
       ├─ super()._save_m2m() → 处理 add_*
       ├─ instance.user_set.remove(*remove_users) → 【移除用户生效】
       ├─ instance.permissions.remove(*remove_permissions) → 【移除权限生效】
       └─ instance.channels.remove(*remove_channels) → 【移除渠道生效】

3. 失效 DataLoader 缓存
   └─ AccessibleChannelsByGroupIdLoader.clear(instance.id)
```

### 3.6 权限组删除流程 (`PermissionGroupDelete.perform_mutation`)

**文件位置：** `saleor/graphql/account/mutations/permission_group/permission_group_delete.py`

```
1. clean_instance()
   ├─ 操作人可管理该组的权限和渠道
   ├─ check_if_group_can_be_removed()
   │   ├─ ensure_deleting_not_left_not_manageable_permissions()
   │   └─ ensure_not_removing_requestor_last_group()
   └─ 不能删除自己的最后一个组

2. instance.delete() → 【组删除生效】
   └─ 级联删除所有多对多关系（Django ORM 自动处理）

3. post_save_action()
   └─ call_event(manager.permission_group_deleted, instance)
```

### 3.7 员工删除流程 (`StaffDelete.perform_mutation`)

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
   └─ 级联删除组关联关系

3. post_save_action()
   └─ call_event(manager.staff_deleted, instance)
```

---

## 四、多 Mutation 协作与交叉校验

### 4.1 "权限可管理性"全局约束

**核心函数：** `get_not_manageable_permissions_*` 系列函数

**约束原则：** 任何变更后，每个权限至少需要一个同时满足以下条件的活跃员工：
1. 拥有 `MANAGE_STAFF` 权限
2. 拥有该权限本身

**校验场景分布：**

| 场景 | 调用位置 | 校验函数 |
|------|----------|----------|
| 停用员工 | `StaffUpdate.clean_is_active` | `get_not_manageable_permissions_when_deactivate_or_remove_users` |
| 删除员工 | `StaffDeleteMixin.clean_instance` | `get_not_manageable_permissions_when_deactivate_or_remove_users` |
| 从组移除权限 | `PermissionGroupUpdate.clean_permissions` | `get_not_manageable_permissions_after_removing_perms_from_group` |
| 从组移除用户 | `PermissionGroupUpdate.clean_remove_users` | `get_not_manageable_permissions_after_removing_users_from_group` |
| 删除权限组 | `PermissionGroupDelete.clean_instance` | `get_not_manageable_permissions_after_group_deleting` |

### 4.2 校验算法详解 (`get_not_manageable_permissions`)

**文件位置：** `saleor/graphql/account/utils.py:245-271`

```
算法流程：
1. 构建 groups_data: {group_pk: {"permissions": set(), "users": set()}}
   - 只包含活跃用户
   - 权限格式为 "app_label.codename"

2. get_users_and_look_for_permissions_in_groups_with_manage_staff()
   ├─ 遍历所有拥有 MANAGE_STAFF 权限且有用户的组
   ├─ 在这些组中查找待校验的权限
   ├─ 找到则从 not_manageable_permissions 中移除
   └─ 收集这些组的所有用户

3. 如果 not_manageable_permissions 已空，返回空集

4. 如果没有 MANAGE_STAFF 用户，返回全部未找到的权限

5. look_for_permission_in_users_with_manage_staff()
   ├─ 遍历所有组
   ├─ 检查组中是否有 MANAGE_STAFF 用户
   ├─ 有则在该组权限中查找剩余未管理的权限
   └─ 找到则从 not_manageable_permissions 中移除

6. 返回剩余的 not_manageable_permissions
```

### 4.3 数据流与状态一致性

```
权限变更请求
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
│  _save_m2m() - 变更生效阶段              │
│  └─ traced_atomic_transaction           │
│     ├─ 所有数据库变更原子执行           │
│     └─ 多对多关系 add/remove            │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  缓存失效阶段                            │
│  └─ AccessibleChannelsByGroupIdLoader   │
│     .clear(instance.id)                 │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  post_save_action() - 事件通知阶段       │
│  └─ call_event(manager.*_created/updated│
│              /deleted, instance)        │
└─────────────────────────────────────────┘
    ↓
Webhook 事件触发 → 外部系统同步
```

---

## 五、关键代码位置索引

### 5.1 模型层
| 内容 | 文件位置 |
|------|----------|
| User 模型 | `saleor/account/models.py:139-327` |
| Group 模型 | `saleor/account/models.py:396-434` |
| 有效权限计算 | `saleor/account/models.py:247-288` |

### 5.2 工具函数
| 内容 | 文件位置 |
|------|----------|
| 可管理组获取 | `saleor/graphql/account/utils.py:116-140` |
| 权限可管理性校验 | `saleor/graphql/account/utils.py:143-271` |
| 组数据映射构建 | `saleor/graphql/account/utils.py:274-309` |

### 5.3 Mutation 层
| 内容 | 文件位置 |
|------|----------|
| StaffCreate | `saleor/graphql/account/mutations/staff/staff_create.py` |
| StaffUpdate | `saleor/graphql/account/mutations/staff/staff_update.py` |
| StaffDelete | `saleor/graphql/account/mutations/staff/staff_delete.py` |
| StaffDeleteMixin | `saleor/graphql/account/mutations/base.py:431-522` |
| PermissionGroupCreate | `saleor/graphql/account/mutations/permission_group/permission_group_create.py` |
| PermissionGroupUpdate | `saleor/graphql/account/mutations/permission_group/permission_group_update.py` |
| PermissionGroupDelete | `saleor/graphql/account/mutations/permission_group/permission_group_delete.py` |

### 5.4 错误码
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
