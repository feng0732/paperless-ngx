# Saved Views 与列表状态配合机制分析

## 1. 整体架构概览

Paperless-ngx 中的 Saved Views（保存视图）与文档列表状态形成了一个"持久化配置 ↔ 动态状态"的双向协作体系：

- **后端**：存储 SavedView 配置（筛选规则、排序、显示模式等），基于 owner + django-guardian 对象权限控制可见范围
- **前端**：通过 `DocumentListViewService` 管理当前列表状态，支持从 SavedView 加载、修改后回写、URL 参数同步
- **用户 Scope**：由三层核心（对象级权限、全局权限、UI 偏好）+ 两层派生（序列化标记、API 兼容层）共同决定 SavedView 的可见性、可编辑性、删除权限和侧边栏/仪表盘入口

---

## 2. 后端数据模型

### 2.1 SavedView 模型

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/models.py#L517-L576)

```python
class SavedView(ModelWithOwner):
    name = models.CharField(max_length=128)
    sort_field = models.CharField(...)
    sort_reverse = models.BooleanField(default=False)
    page_size = models.PositiveIntegerField(...)
    display_mode = models.CharField(...)
    display_fields = models.JSONField(...)
```

**重要**：SavedView 模型本身**没有** `show_on_dashboard` / `show_in_sidebar` 字段。这些字段仅存在于旧 API（v9 及以下）的兼容层中，实际存储在用户个人的 `UiSettings.settings["saved_views"]` 下。

SavedView 继承自 `ModelWithOwner`，该基类提供 `owner` 字段（指向 User），实现用户所有权。

### 2.2 SavedViewFilterRule 模型

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/models.py#L578-L649)

每个 SavedView 关联多条 FilterRule：

```python
class SavedViewFilterRule(models.Model):
    saved_view = models.ForeignKey(SavedView, on_delete=models.CASCADE, related_name="filter_rules")
    rule_type = models.PositiveSmallIntegerField(choices=RULE_TYPES)
    value = models.CharField(max_length=255, blank=True, null=True)
```

支持 50 种规则类型，包括标题/全文搜索、标签过滤、联系人/文档类型/存储路径、日期范围、所有者权限、自定义字段、ASN 编号、收件箱、共享、MIME 类型等。

### 2.3 UiSettings 模型（用户个人偏好）

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/models.py#L652-L661)

```python
class UiSettings(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name="ui_settings")
    settings = models.JSONField(null=True)
```

与 SavedView 相关的设置路径：

```
settings.saved_views.dashboard_views_visible_ids  # 仪表盘可见的 SavedView ID 数组
settings.saved_views.sidebar_views_visible_ids    # 侧边栏可见的 SavedView ID 数组
settings.saved_views.dashboard_views_sort_order   # 仪表盘排序
settings.saved_views.sidebar_views_sort_order     # 侧边栏排序
settings.saved_views.sidebar_views_show_count     # 是否显示侧边栏文档计数
settings.saved_views.warn_on_unsaved_change       # 是否警告未保存修改
```

---

## 3. 用户 Scope 机制：对象权限、全局权限、UI 偏好、派生标记、兼容层

SavedView 的"用户 Scope"由以下**三层核心 + 两层派生**共同决定：

**核心层（真正决定行为）**：

| 层级 | 机制 | 存储位置 | 影响范围 |
|------|------|---------|---------|
| 1 | 对象级权限（owner + django-guardian） | SavedView.owner + guardian UserObjectPermission/GroupObjectPermission | 是否能看到对象、是否能编辑、是否能删除具体 SavedView 对象 |
| 2 | 全局 model-level 权限（Django 全局权限） | Django auth_permissions 表 | 是否能对该**类型**执行 add/view/change/delete（与具体对象无关）|
| 3 | Dashboard/Sidebar 可见性设置 | 当前用户的 UiSettings.settings["saved_views"] | 是否出现在侧边栏/仪表盘入口 |

**派生层（只读输出或兼容实现）**：

| 层级 | 机制 | 存储位置 | 影响范围 |
|------|------|---------|---------|
| A | user_can_change / permissions 字段 | 序列化时动态计算（只读派生标记） | 前端快速判断是否可编辑 |
| B | 旧版兼容字段（show_on_dashboard/show_in_sidebar） | 序列化层（API v9）模拟 | 兼容旧 API 客户端，实际读写 UiSettings |

### 3.1 核心层一：对象级权限（决定能否看到、编辑、删除具体 SavedView 对象）

#### 3.1.1 所有权模型

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/models.py#L32-L44)

```python
class ModelWithOwner(models.Model):
    owner = models.ForeignKey(User, blank=True, null=True, default=None, on_delete=models.SET_NULL)
```

- `owner=None`：无所有者（向后兼容旧数据），所有人均可查看和修改
- `owner=某用户`：该用户拥有完全控制权

#### 3.1.2 查询级过滤：用户能看到哪些 SavedView

定义于 [filters.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/filters.py#L950-L962) 的 `ObjectOwnedOrGrantedPermissionsFilter`：

```python
class ObjectOwnedOrGrantedPermissionsFilter(ObjectPermissionsFilter):
    def filter_queryset(self, request, queryset, view):
        objects_with_perms = super().filter_queryset(request, queryset, view)  # django-guardian 显式授权
        objects_owned = queryset.filter(owner=request.user)                     # 用户自有
        objects_unowned = queryset.filter(owner__isnull=True)                   # 无所有者（兼容）
        return objects_with_perms | objects_owned | objects_unowned
```

**可见集合公式**：

```
用户可见 SavedView 集合
  = {owner == 当前用户}
  ∪ {owner IS NULL}
  ∪ {通过 django-guardian 获得 view_savedview 权限的对象}
```

#### 3.1.3 操作级权限：能否 GET/POST/PUT/PATCH/DELETE

定义于 [permissions.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/permissions.py#L28-L51) 的 `PaperlessObjectPermissions`：

```python
class PaperlessObjectPermissions(DjangoObjectPermissions):
    perms_map = {
        "GET":    ["%(app_label)s.view_%(model_name)s"],
        "POST":   ["%(app_label)s.add_%(model_name)s"],
        "PUT":    ["%(app_label)s.change_%(model_name)s"],
        "PATCH":  ["%(app_label)s.change_%(model_name)s"],
        "DELETE": ["%(app_label)s.delete_%(model_name)s"],
    }

    def has_object_permission(self, request, view, obj):
        if hasattr(obj, "owner") and obj.owner is not None:
            if request.user == obj.owner:
                return True                              # 所有者：全部操作通过
            else:
                return super().has_object_permission(    # 非所有者：交给 django-guardian 判断
                    request, view, obj
                )
        else:
            return True  # 无所有者：全部操作通过
```

**权限判断逻辑**（按优先级）：

1. `obj.owner is None` → **全部操作通过**（向后兼容）
2. `request.user == obj.owner` → **全部操作通过**
3. 否则：通过 django-guardian 判断是否具有对应的 model-level + object-level 权限（如 `documents.change_savedview`）

#### 3.1.4 视图层权限装配

在 [views.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/views.py#L2548-L2562) 的 `SavedViewViewSet` 中：

```python
class SavedViewViewSet(BulkPermissionMixin, PassUserMixin, ModelViewSet[SavedView]):
    queryset = SavedView.objects.select_related("owner").prefetch_related("filter_rules")
    permission_classes = (IsAuthenticated, PaperlessObjectPermissions)
    filter_backends = (OrderingFilter, ObjectOwnedOrGrantedPermissionsFilter)
```

- `IsAuthenticated`：必须登录
- `PaperlessObjectPermissions`：操作级权限（GET/PUT/PATCH/DELETE）
- `ObjectOwnedOrGrantedPermissionsFilter`：查询结果过滤（list 时只返回可见对象）

### 3.2 核心层二：全局 model-level 权限（Django 全局权限）

全局 model-level 权限由 Django 内置的 auth_permissions 表定义（如 `documents.add_savedview`、`documents.view_savedview`、`documents.change_savedview`、`documents.delete_savedview`），与具体 SavedView 对象无关，由用户组或用户直接分配。

#### 3.2.1 前端检查：currentUserCan 与 *pngxIfPermissions

`*pngxIfPermissions` 指令定义于 [if-permissions.directive.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/directives/if-permissions.directive.ts#L28-L36)，调用 `permissionsService.currentUserCan(action, type)`：

```typescript
public currentUserCan(
  action: PermissionAction,
  type: PermissionType
): boolean {
  return (
    this.currentUser?.is_superuser ||
    this.permissions?.includes(this.getPermissionCode(action, type))
  )
}
```

判断逻辑：`currentUser.is_superuser` **OR** 用户的全局权限列表包含对应 permission code。

`getPermissionCode` 将 `PermissionAction.Change + PermissionType.SavedView` 拼接为 `change_savedview`（对应 Django 权限 `documents.change_savedview`）。

这是 SavedViewsComponent 中 Permissions 按钮和 Delete 按钮显示的**第二层**必要条件（AND）。

#### 3.2.2 后端检查：DjangoObjectPermissions.has_permission

`PaperlessObjectPermissions` 继承自 `DjangoObjectPermissions`，其基类的 `has_permission` 方法会检查请求用户是否具有全局的 model-level 权限（perms_map 中定义的 HTTP 方法对应的权限）。

例如 `DELETE /api/saved_views/:id` 请求在到达对象级判断之前，会先通过 `has_permission` 检查用户是否有全局 `documents.delete_savedview` 权限（或用户是 superuser）。

#### 3.2.3 全局权限 vs 对象级权限的关系

| 检查阶段 | 位置 | 决定什么 |
|---------|------|---------|
| `has_permission`（全局） | DjangoObjectPermissions 基类 | 用户是否有权对该类型执行该 HTTP 操作 |
| `has_object_permission`（对象级） | PaperlessObjectPermissions | 用户是否有权对这个**具体对象**执行操作 |

两者都通过，请求才会被允许。

### 3.3 派生层 A：user_can_change / permissions 字段（只读序列化派生标记）

#### 3.3.1 后端计算逻辑

定义于 [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/serialisers.py#L354-L363) 的 `OwnedObjectSerializer.get_user_can_change`：

```python
def get_user_can_change(self, obj) -> bool:
    checker = ObjectPermissionChecker(self.user) if self.user is not None else None
    return (
        obj.owner is None                                   # 无所有者：可改
        or obj.owner == self.user                            # 所有者：可改
        or (
            self.user is not None
            and checker.has_perm(                            # django-guardian 显式授权
                f"change_{obj.__class__.__name__.lower()}",
                obj
            )
        )
    )
```

这个字段与 3.1.3 节中 `PaperlessObjectPermissions.has_object_permission` 对 PUT/PATCH 的判断**完全一致**，区别在于它是序列化时**附加到响应对象**的一个布尔标记，供前端直接读取，无需再做权限判断。

#### 3.3.2 字段返回条件（full_perms 参数）

定义于 [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/serialisers.py#L267-L278)：

```python
def __init__(self, *args, **kwargs) -> None:
    super().__init__(*args, **kwargs)
    if not self.all_fields:
        try:
            if self.full_perms:
                self.fields.pop("user_can_change")      # full_perms=true：去掉简化字段
                self.fields.pop("is_shared_by_requester")
            else:
                self.fields.pop("permissions")          # full_perms=false：去掉完整权限
        except KeyError:
            pass
```

| 请求参数 | 返回 `permissions` | 返回 `user_can_change` | 返回 `is_shared_by_requester` |
|---------|-------------------|----------------------|-----------------------------|
| `?full_perms=true` | ✅ 完整（view/change 的 users/groups） | ❌ | ❌ |
| 默认 | ❌ | ✅（简化布尔值） | ✅ |

其中 `permissions` 字段的完整结构由 [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/serialisers.py#L309-L352) 定义：

```python
def get_permissions(self, obj) -> dict:
    return {
        "view":   {"users": [...], "groups": [...]},
        "change": {"users": [...], "groups": [...]},
    }
```

#### 3.3.3 前端使用 user_can_change

定义于 [permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/services/permissions.service.ts#L75-L97)：

```typescript
public currentUserHasObjectPermissions(
  action: string,
  object: ObjectWithPermissions
): boolean {
  if (action === PermissionAction.View) {
    return (
      this.currentUserOwnsObject(object) ||
      object.permissions?.view.users.includes(this.currentUser.id) ||
      object.permissions?.view.groups.filter((g) =>
        this.currentUser.groups.includes(g)
      ).length > 0
    )
  } else if (action === PermissionAction.Change) {
    return (
      this.currentUserOwnsObject(object) ||           // owner 或无 owner
      object.user_can_change ||                        // 后端计算的简化标记
      object.permissions?.change.users.includes(this.currentUser.id) ||
      object.permissions?.change.groups.filter((g) =>
        this.currentUser.groups.includes(g)
      ).length > 0
    )
  }
}
```

前端判断 Change 权限时，会**同时**使用 `currentUserOwnsObject`、`object.user_can_change` 以及完整的 `permissions.change` 列表，任意一个满足即可。

### 3.4 核心层三：Dashboard/Sidebar 可见性设置（用户个人 UI 偏好）

这一层与 SavedView 对象权限**完全解耦**，是每个用户自己的 UI 偏好，不影响 SavedView 对象本身。

#### 3.4.1 存储位置

存储在请求用户自己的 `UiSettings.settings` JSON 中：

```python
UiSettings.settings["saved_views"]["dashboard_views_visible_ids"] = [1, 5, 9]
UiSettings.settings["saved_views"]["sidebar_views_visible_ids"]   = [1, 3, 9]
```

该设置是**用户私有**的：不同用户对同一个 SavedView 可以有不同的可见性设置。

#### 3.4.2 前端注入：SavedViewService.withUserVisibility

定义于 [saved-view.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/services/rest/saved-view.service.ts#L75-L95)：

```typescript
private withUserVisibility(view: SavedView): SavedView {
  return {
    ...view,
    show_on_dashboard: this.isDashboardVisible(view),
    show_in_sidebar: this.isSidebarVisible(view),
  }
}

private isDashboardVisible(view: SavedView): boolean {
  const visibleIds = this.getVisibleViewIds(
    SETTINGS_KEYS.DASHBOARD_VIEWS_VISIBLE_IDS
  )
  return visibleIds.includes(view.id)
}
```

每次 `SavedViewService.list()` 返回结果时，前端会**根据当前用户的 UiSettings 动态注入** `show_on_dashboard` 和 `show_in_sidebar` 两个布尔字段。这两个字段**不回写到 SavedView 对象**，仅用于前端 UI 展示。

#### 3.4.3 保存可见性设置

定义于 [settings.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/services/settings.service.ts#L728-L739)：

```typescript
updateSavedViewsVisibility(
  dashboardVisibleViewIds: number[],
  sidebarVisibleViewIds: number[]
): Observable<any> {
  this.set(SETTINGS_KEYS.DASHBOARD_VIEWS_VISIBLE_IDS, [
    ...new Set(dashboardVisibleViewIds),
  ])
  this.set(SETTINGS_KEYS.SIDEBAR_VIEWS_VISIBLE_IDS, [
    ...new Set(sidebarVisibleViewIds),
  ])
  return this.storeSettings()  // POST /api/ui_settings/
}
```

在 [saved-views.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/components/manage/saved-views/saved-views.component.ts#L164-L238) 的 `save()` 方法中：

```typescript
public save() {
  const groups = Object.values(this.savedViewsGroup.controls) as FormGroup[]
  const visibilityChanged = groups.some(
    (group) =>
      group.get('show_on_dashboard')?.dirty ||
      group.get('show_in_sidebar')?.dirty
  )
  // ...
  groups.forEach((group) => {
    const value = group.getRawValue()
    if (value.show_on_dashboard) dashboardVisibleIds.push(value.id)
    if (value.show_in_sidebar)   sidebarVisibleIds.push(value.id)
    // 重要：发送到后端前删除这两个字段（SavedView 模型上没有）
    delete value.show_on_dashboard
    delete value.show_in_sidebar
    // ...
  })
  // 先 patch 有修改的 SavedView 对象，再单独保存可见性设置到 UiSettings
  if (changed.length)     saveOperation = saveOperation.pipe(switchMap(() => this.savedViewService.patchMany(changed)))
  if (visibilityChanged)  saveOperation = saveOperation.pipe(switchMap(() =>
    this.settings.updateSavedViewsVisibility(dashboardVisibleIds, sidebarVisibleIds)
  ))
}
```

#### 3.4.4 侧边栏/仪表盘列表入口

定义于 [saved-view.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/services/rest/saved-view.service.ts#L97-L127)：

```typescript
get sidebarViews(): SavedView[] {
  const sidebarViews = this.savedViews.filter((v) => this.isSidebarVisible(v))
  const sorted: number[] = this.settingsService.get(SETTINGS_KEYS.SIDEBAR_VIEWS_SORT_ORDER)
  return sorted?.length > 0
    ? sorted.map((id) => sidebarViews.find((v) => v.id === id))
            .concat(sidebarViews.filter((v) => !sorted.includes(v.id)))
            .filter((v) => v)
    : [...sidebarViews]
}

get dashboardViews(): SavedView[] {
  // 类似逻辑
}
```

注意：这里的 `this.savedViews` 已经经过 3.1.2 节的后端过滤，只包含用户有权查看的 SavedView。`sidebarViews` / `dashboardViews` 在此基础上再应用用户个人的可见性偏好。

### 3.5 派生层 B：旧版 API 兼容（show_on_dashboard/show_in_sidebar 字段模拟）

API v10 将 show_on_dashboard/show_in_sidebar 从 SavedView 模型迁移到了 UiSettings。为保持向后兼容，v9 API 仍在序列化层模拟这两个字段。

#### 3.5.1 序列化时注入（GET）

定义于 [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/serialisers.py#L1409-L1434)：

```python
def to_representation(self, instance):
    ret = super().to_representation(instance)
    api_version = self._get_api_version()
    if api_version < 10:
        # 从当前请求用户的 UiSettings 读取可见 ID，注入到响应中
        dashboard_ids = set(...)
        sidebar_ids = set(...)
        if user is not None and hasattr(user, "ui_settings"):
            saved_views = user.ui_settings.settings.get("saved_views", {})
            dashboard_ids = set(saved_views.get("dashboard_views_visible_ids", []))
            sidebar_ids = set(saved_views.get("sidebar_views_visible_ids", []))
        ret["show_on_dashboard"] = instance.id in dashboard_ids
        ret["show_in_sidebar"] = instance.id in sidebar_ids
    return ret
```

这意味着 v9 API 返回的 `show_on_dashboard` / `show_in_sidebar` 是**请求用户个人**的可见性偏好，而非 SavedView 对象的固有属性。

#### 3.5.2 反序列化时提取（PUT/PATCH/POST）

定义于 [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/serialisers.py#L1436-L1460)：

```python
def to_internal_value(self, data):
    api_version = self._get_api_version()
    if api_version >= 10:
        return super().to_internal_value(data)
    # v9：从请求中提取 show_on_dashboard/show_in_sidebar，放到 validated_data 中
    normalized_data = data.copy()
    legacy_visibility_fields = {}
    for field_name in ("show_on_dashboard", "show_in_sidebar"):
        if field_name in normalized_data:
            legacy_visibility_fields[field_name] = boolean_field.to_internal_value(...)
            del normalized_data[field_name]
    ret = super().to_internal_value(normalized_data)
    ret.update(legacy_visibility_fields)
    return ret
```

#### 3.5.3 持久化到 UiSettings

定义于 [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/serialisers.py#L1350-L1408) 的 `_update_legacy_visibility_preferences`，以及在 `create` / `update` 中调用：

```python
def update(self, instance, validated_data):
    show_on_dashboard = validated_data.pop("show_on_dashboard", None)
    show_in_sidebar = validated_data.pop("show_in_sidebar", None)
    # ... 更新 SavedView 基本字段和 filter_rules ...
    ui_settings = self._update_legacy_visibility_preferences(
        instance.id,
        show_on_dashboard=show_on_dashboard,
        show_in_sidebar=show_in_sidebar,
    )
    return instance

def _update_legacy_visibility_preferences(self, saved_view_id, *, show_on_dashboard, show_in_sidebar):
    # 将请求用户的 UiSettings.settings["saved_views"] 中对应 ID 加入或移除
    dashboard_ids = {...}
    sidebar_ids = {...}
    if show_on_dashboard is not None:
        if show_on_dashboard: dashboard_ids.add(saved_view_id)
        else:                  dashboard_ids.discard(saved_view_id)
    if show_in_sidebar is not None:
        if show_in_sidebar:   sidebar_ids.add(saved_view_id)
        else:                  sidebar_ids.discard(saved_view_id)
    ui_settings.settings["saved_views"] = {
        "dashboard_views_visible_ids": sorted(dashboard_ids),
        "sidebar_views_visible_ids": sorted(sidebar_ids),
    }
    ui_settings.save()
    return ui_settings
```

这说明：即使是 v9 API，`show_on_dashboard` / `show_in_sidebar` 也是**写入请求用户的 UiSettings**，而不是 SavedView 对象本身。

---

## 4. 前端权限与列表入口：完整决策链

### 4.1 SavedView 管理页（SavedViewsComponent）

#### 4.1.1 Actions 区块的总体可见性：canDeleteSavedView

在 [saved-views.component.html](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/components/manage/saved-views/saved-views.component.html#L27-L45) 中：

```html
<div class="col-auto">
  @if (canDeleteSavedView(view)) {
    <!-- Permissions 按钮和 Delete 按钮都在这里 -->
  }
</div>
```

`canDeleteSavedView` 定义于 [saved-views.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/components/manage/saved-views/saved-views.component.ts#L247-L249)：

```typescript
public canDeleteSavedView(view: SavedView): boolean {
  return this.permissionsService.currentUserOwnsObject(view)
}
```

`currentUserOwnsObject` 定义于 [permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/services/permissions.service.ts#L66-L73)：

```typescript
public currentUserOwnsObject(object: ObjectWithPermissions): boolean {
  return (
    !object ||                                // object 本身为 falsy（undefined/null）
    !object.owner ||                          // owner 为 falsy（undefined/null/0 等，即无主）
    this.currentUser.is_superuser ||          // 当前用户是 Django superuser
    object.owner === this.currentUser.id      // owner 就是当前用户
  )
}
```

`ObjectWithPermissions` 接口定义于 [object-with-permissions.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/data/object-with-permissions.ts#L14-L22)：

```typescript
export interface ObjectWithPermissions extends ObjectWithId {
  owner?: number
  permissions?: PermissionsObject
  user_can_change?: boolean
  is_shared_by_requester?: boolean
}
```

**结论**：这个 `@if` 条件控制整个 Actions 列（包括 Permissions + Delete 两个按钮）是否出现。仅 `!object` / owner 为 falsy（无主）/ `currentUser.is_superuser` / owner == 当前用户 时可见。

#### 4.1.2 Permissions 按钮：model-level Change 权限

```html
<button
  (click)="editPermissions(view)"
  *pngxIfPermissions="{ action: PermissionAction.Change, type: PermissionType.SavedView }"
>
  Permissions
</button>
```

`*pngxIfPermissions` 指令定义于 [if-permissions.directive.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/directives/if-permissions.directive.ts#L28-L36)，调用 `permissionsService.currentUserCan(action, type)` —— 这是**全局 model-level 权限**检查（即用户是否有 `documents.change_savedview` 这个 Django 全局权限，与具体 SavedView 对象无关）。

**Permissions 按钮显示条件（AND）**：
1. `canDeleteSavedView(view)` = true（`!object` / owner 为 falsy 无主 / `currentUser.is_superuser` / owner == 当前用户）—— 外层 `@if`
2. `*pngxIfPermissions(Change, SavedView)` = true（全局有 `documents.change_savedview` 权限）—— 指令级检查

#### 4.1.3 Delete 按钮：双重权限判断

```html
<pngx-confirm-button
  (confirm)="deleteSavedView(view)"
  *pngxIfPermissions="{ action: PermissionAction.Delete, type: PermissionType.SavedView }"
  iconName="trash"
>
</pngx-confirm-button>
```

Delete 按钮同样有两层判断：

**Delete 按钮显示条件（AND）**：
1. `canDeleteSavedView(view)` = true（`!object` / owner 为 falsy 无主 / `currentUser.is_superuser` / owner == 当前用户）—— 外层 `@if`
2. `*pngxIfPermissions(Delete, SavedView)` = true（全局有 `documents.delete_savedview` 权限）—— 指令级检查

删除动作本身还会经过后端 `PaperlessObjectPermissions.has_object_permission` 的**对象级**校验（见 4.3 节）。

#### 4.1.4 表单字段可编辑性：canEditSavedView

```typescript
public canEditSavedView(view: SavedView): boolean {
  return this.permissionsService.currentUserHasObjectPermissions(
    PermissionAction.Change,
    view
  )
}
```

`currentUserHasObjectPermissions(Change, view)` 定义于 [permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/services/permissions.service.ts#L75-L97)：

```typescript
return (
  this.currentUserOwnsObject(object) ||           // !object / owner 为 falsy / currentUser.is_superuser / owner == 当前用户
  object.user_can_change ||                        // 后端计算的简化标记
  object.permissions?.change.users.includes(this.currentUser.id) ||
  object.permissions?.change.groups.filter((g) =>
    this.currentUser.groups.includes(g)
  ).length > 0
)
```

这控制 `name / page_size / display_mode / display_fields` 表单字段是否 `[disabled]="!canEditSavedView(view)"`。

注意：**`show_on_dashboard` / `show_in_sidebar` 的复选框始终 `disabled: false`**（所有人都能切换），因为它们只写入用户自己的 UiSettings，不修改 SavedView 对象。

#### 4.1.5 canEditSavedView vs canDeleteSavedView：本质差异对比

两个方法名称相似但**底层检查逻辑和控制范围完全不同**，这是权限边界的关键细节：

| 维度 | `canEditSavedView(view)` | `canDeleteSavedView(view)` |
|------|-------------------------|---------------------------|
| **底层调用** | `currentUserHasObjectPermissions(Change, view)` | `currentUserOwnsObject(view)` |
| **判断逻辑**（OR 任一成立） | ① `currentUserOwnsObject(view)`  ② `view.user_can_change == true`  ③ `view.permissions.change.users` 包含当前用户  ④ `view.permissions.change.groups` 与用户组有交集 | ① `!object`（falsy） ② `!object.owner`（owner 为 falsy 无主） ③ `currentUser.is_superuser`  ④ `object.owner === currentUser.id` |
| **范围比较** | **更广**：包含所有 owner/superuser/无主，**加上**被 django-guardian 显式授权 change 对象级权限的非 owner 用户 | **更窄**：仅 owner/superuser/无主，**不包括**仅被授权 change 权限的非 owner |
| **控制什么** | FormGroup 中 `name / page_size / display_mode / display_fields / id` 字段的 `[disabled]` 属性（表单字段是否可编辑） | HTML 模板中整个 Actions 列 `<div>` 的 `@if` 条件（Permissions 按钮 + Delete 按钮整体是否渲染） |
| **额外层级** | 无额外限制 | Delete 按钮在 `@if` 内还需 `*pngxIfPermissions(Delete, SavedView)`（全局 model-level）；Permissions 按钮在 `@if` 内还需 `*pngxIfPermissions(Change, SavedView)`（全局 model-level） |

**关键场景差异**：一个非 owner 用户被 owner 通过 Permissions 对话框授予了 Change 对象级权限（出现在 `permissions.change.users` 中），此时：
- ✅ `canEditSavedView` = true → 可以编辑 name/page_size/display_mode/display_fields
- ❌ `canDeleteSavedView` = false → Actions 列完全不渲染 → **看不到 Permissions 按钮也看不到 Delete 按钮**

### 4.2 文档列表页保存按钮

在文档列表组件中，保存当前视图修改的按钮可见性由 `canEditSavedView(activeSavedView)` 控制（即 Change 对象级权限）。

### 4.3 后端 DELETE 权限映射与实际执行

#### 4.3.1 perms_map：HTTP 方法 → Django 权限名

定义于 [permissions.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/permissions.py#L28-L42)：

```python
class PaperlessObjectPermissions(DjangoObjectPermissions):
    perms_map = {
        "GET":    ["%(app_label)s.view_%(model_name)s"],
        "POST":   ["%(app_label)s.add_%(model_name)s"],
        "PUT":    ["%(app_label)s.change_%(model_name)s"],
        "PATCH":  ["%(app_label)s.change_%(model_name)s"],
        "DELETE": ["%(app_label)s.delete_%(model_name)s"],
    }
```

即 `DELETE /api/saved_views/:id` → 需要 `documents.delete_savedview` 权限。

#### 4.3.2 对象级判断

定义于 [permissions.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/permissions.py#L44-L51)：

```python
def has_object_permission(self, request, view, obj):
    if hasattr(obj, "owner") and obj.owner is not None:
        if request.user == obj.owner:
            return True                                 # 所有者：全部操作通过
        else:
            return super().has_object_permission(       # 非所有者：django-guardian 判断
                request, view, obj
            )
    else:
        return True   # 无所有者：全部操作通过
```

**DELETE 的实际执行需要同时满足**：
1. **全局 model-level**：用户有 `documents.delete_savedview` 权限（由 `DjangoObjectPermissions` 基类的 `has_permission` 检查，或 `currentUser.is_superuser`）
2. **对象级**：`request.user == obj.owner` 或 `obj.owner is None`（无主），或 django-guardian 中该用户有 `delete_savedview` 对象级权限

**结合前端限制的最终结论**：
- 前端删除按钮还需额外通过 `canDeleteSavedView(view)`（`currentUserOwnsObject`）的 `@if` 判断
- 系统**没有**任何 UI 或 API 入口可以授予 `delete_savedview` 对象级权限（见 4.5 节 `set_permissions` 只支持 view/change，[PermissionsFormComponent](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/components/common/input/permissions/permissions-form/permissions-form.component.ts#L62-L74) 的 FormGroup 也只有 view/change 两组）
- 因此实际能看到并执行 DELETE 的用户范围是：`currentUser.is_superuser` **OR**（用户有全局 `documents.delete_savedview` 权限 **AND**（`request.user == obj.owner` **OR** `obj.owner is None`（无主）））

### 4.4 新建 SavedView 时的 owner 归属

定义于 [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/serialisers.py#L435-L450) 的 `OwnedObjectSerializer.create`：

```python
def create(self, validated_data):
    request = self.context.get("request")
    if (
        "owner" not in validated_data
        or (request is not None and "owner" not in request.data)
    ) and self.user:
        validated_data["owner"] = self.user  # 默认归属当前用户
    if "set_permissions" in validated_data:
        self._set_permissions(validated_data["set_permissions"], instance)
    return instance
```

### 4.5 set_permissions：只含 view/change，change 自动赋 view

#### 4.5.1 仅支持 view / change 两个 action

定义于 [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/serialisers.py#L191-L211) 的 `SetPermissionsMixin.validate_set_permissions`：

```python
def validate_set_permissions(self, set_permissions=None):
    permissions_dict = {
        "view": {},
        "change": {},
    }
    if set_permissions is not None:
        for action in ["view", "change"]:          # 只循环 view 和 change
            if action in set_permissions:
                if "users" in set_permissions[action]:
                    users = set_permissions[action]["users"]
                    permissions_dict[action]["users"] = self._validate_user_ids(users)
                if "groups" in set_permissions[action]:
                    groups = set_permissions[action]["groups"]
                    permissions_dict[action]["groups"] = self._validate_group_ids(groups)
            else:
                del permissions_dict[action]
    return permissions_dict
```

传入的 `set_permissions` 如果包含 `delete` 或其他 action 会被**完全忽略**，只有 `view` 和 `change` 会被处理。

#### 4.5.2 change 权限自动附带 view

定义于 [permissions.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/permissions.py#L93-L164) 的 `set_permissions_for_object`：

```python
for action, entry in permissions.items():
    permission = f"{action}_{object.__class__.__name__.lower()}"
    if "users" in entry:
        users_to_add = User.objects.filter(id__in=entry["users"])
        for user in users_to_add:
            assign_perm(permission, user, object)
            if action == "change":
                # change gives view too
                assign_perm(
                    f"view_{object.__class__.__name__.lower()}",
                    user,
                    object,
                )
    if "groups" in entry:
        # 对 groups 同样处理
        for group in groups_to_add:
            assign_perm(permission, group, object)
            if action == "change":
                assign_perm(f"view_{object.__class__.__name__.lower()}", group, object)
```

**结论**：
- 授予 `change` 会**自动同时**授予 `view`，无需单独设置
- `delete` 权限没有任何对象级授权入口，不能通过 `set_permissions` 分发
- 前端 [PermissionsDialogComponent](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/components/common/permissions-dialog/permissions-dialog.component.ts) 也只显示 View / Change 两个权限选项

### 4.6 owner / set_permissions 变更限制：PermissionDenied（非 ValidationError）

定义于 [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/serialisers.py#L452-L469)：

```python
def update(self, instance, validated_data):
    user = getattr(self, "user", None)
    is_superuser = user.is_superuser if user is not None else False
    is_owner = instance.owner == user if user is not None else False
    is_unowned = instance.owner is None

    if (
        ("owner" in validated_data and validated_data["owner"] != instance.owner)
        or "set_permissions" in validated_data
    ) and not (is_superuser or is_owner or is_unowned):
        raise PermissionDenied(           # ← 注意：是 PermissionDenied，不是 ValidationError
            _("Insufficient permissions."),  # ← 错误消息是这个，不是 "Only superusers..."
        )

    if "set_permissions" in validated_data:
        self._set_permissions(validated_data["set_permissions"], instance)
    return super().update(instance, validated_data)
```

**关键差异**：

| 项目 | 实际 | 常见误解 |
|------|------|---------|
| 异常类型 | `rest_framework.exceptions.PermissionDenied` → HTTP **403** | `serializers.ValidationError` → HTTP **400** |
| 错误消息 | `_("Insufficient permissions.")`（翻译） | `"Only superusers, owners or unowned objects can change permissions"` |
| 触发条件 | 试图改 owner 或发 set_permissions，且非 superuser/owner/无主 | 同上 |

只有 superuser、对象 owner、或对象无主时，才能修改 `owner` 字段或设置 `set_permissions`。

---

## 5. 筛选条件保存与加载

### 5.1 保存流程（前端 → 后端）

在 [document-list.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/components/document-list/document-list.component.ts#L398-L428)：

```typescript
saveViewConfig() {
  let savedView: SavedView = {
    id: this.list.activeSavedViewId,
    filter_rules: this.list.filterRules,
    sort_field: this.list.sortField,
    sort_reverse: this.list.sortReverse,
    display_mode: this.list.displayMode,
    display_fields: this.activeDisplayFields,
  }
  this.savedViewService.patch(savedView).subscribe(...)
}
```

后端 [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/serialisers.py#L1480-L1512)：

```python
def update(self, instance, validated_data):
    if "filter_rules" in validated_data:
        rules_data = validated_data.pop("filter_rules")
    else:
        rules_data = None
    instance = super().update(instance, validated_data)
    if rules_data is not None:
        SavedViewFilterRule.objects.filter(saved_view=instance).delete()
        for rule_data in rules_data:
            SavedViewFilterRule.objects.create(saved_view=instance, **rule_data)
    return instance
```

**策略**：filter_rules 采用"先删后建"的全量替换。

### 5.2 加载流程（后端 → 前端）

**路由 1**：`/view/:id` — 专用视图页

在 [document-list.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/components/document-list/document-list.component.ts#L274-L303)：

```typescript
this.route.paramMap.pipe(...).subscribe(({ view }) => {
  this.activeSavedView = view
  this.unmodifiedSavedView = view  // 快照，用于 dirty tracking
  this.list.activateSavedViewWithQueryParams(view, queryParams)
  this.list.reload(...)
})
```

**路由 2**：`/documents?view=:id` — 普通文档列表页

见同文件的 `loadViewConfig` 方法。

状态激活在 [document-list-view.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/services/document-list-view.service.ts#L238-L274)：

```typescript
activateSavedView(view: SavedView) {
  this._activeSavedViewId = view.id
  this.loadSavedView(view)  // 写入 filterRules/sortField/sortReverse/displayMode 等
  this.router.navigate(['view', view.id])
}
```

---

## 6. 前端列表状态管理

### 6.1 ListViewState 数据结构

定义于 [document-list-view.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/services/document-list-view.service.ts#L45-L102)：

```typescript
export interface ListViewState {
  title?: string
  documents?: Document[]
  currentPage: number
  collectionSize?: number
  sortField: string
  sortReverse: boolean
  filterRules: FilterRule[]
  selected?: Set<number>
  allSelected?: boolean
  pageSize?: number
  displayMode?: DisplayMode
  displayFields?: DisplayField[]
}
```

### 6.2 多视图状态隔离

```typescript
private listViewStates: Map<number, ListViewState> = new Map()
private _activeSavedViewId: number = null  // null = 默认/临时视图
```

- key = `null`：默认文档列表，持久化到 `localStorage`
- key = SavedView ID：对应 SavedView 的独立状态（页码、选中项等），切换时保留

### 6.3 Dirty Tracking

在 [document-list.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/components/document-list/document-list.component.ts#L158-L190)：

```typescript
get savedViewIsModified(): boolean {
  return (
    this.unmodifiedSavedView.sort_field !== this.list.sortField ||
    this.unmodifiedSavedView.sort_reverse !== this.list.sortReverse ||
    this.unmodifiedSavedView.page_size !== this.list.pageSize ||
    this.unmodifiedSavedView.display_mode !== this.list.displayMode ||
    // display_fields 对比
    filterRulesDiffer(this.unmodifiedSavedView.filter_rules, this.list.filterRules)
  )
}
```

修改后标题显示 `*` 后缀。

---

## 7. 筛选规则与 URL 参数的双向映射

### 7.1 FilterRule → Query Params

在 [query-params.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/utils/query-params.ts#L151-L188)：

```typescript
export function queryParamsFromFilterRules(filterRules: FilterRule[]): Params {
  let params = {}
  for (let rule of filterRules) {
    let ruleType = FILTER_RULE_TYPES.find((t) => t.id == rule.rule_type)
    params[ruleType.filtervar] = rule.value
  }
  return params
}
```

### 7.2 URL 同步策略

在 `DocumentListViewService.reload()` 中：
- **非 SavedView 模式**（`_activeSavedViewId == null`）：导航到 `/documents`，完整同步筛选/排序/分页到 query params
- **SavedView 模式**：仅同步分页参数，筛选/排序由 SavedView ID 隐含

---

## 8. 前端列表刷新机制

### 8.1 核心 reload 方法

在 [document-list-view.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/services/document-list-view.service.ts#L304-L387)：

```typescript
reload(onFinish?, updateQueryParams: boolean = true) {
  this.documentService.listFiltered(
    activeListViewState.currentPage,
    activeListViewState.pageSize ?? this.pageSize,
    activeListViewState.sortField,
    activeListViewState.sortReverse,
    activeListViewState.filterRules,
    { truncate_content: true, include_selection_data: true }
  ).subscribe(...)
}
```

### 8.2 触发场景

| 操作 | 触发位置 |
|------|---------|
| 切换 SavedView | `activateSavedView` → `reload` |
| 修改筛选规则 | `setFilterRules` → `reload` |
| 修改排序 | `setSort` / `sortField` setter → `reload` |
| 切换页码 | `currentPage` setter → `reload` |
| 修改每页条数 | `pageSize` setter → `reload` |
| 文档消费完成（WebSocket） | `onDocumentConsumptionFinished` → `reload` |
| 文档删除（WebSocket） | `onDocumentDeleted` → `reload` |

### 8.3 include_selection_data

当 `include_selection_data=true` 时，后端 `DocumentViewSet.list`（[views.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/views.py#L1185-L1202)）额外计算当前筛选结果下 tags/correspondents/document_types/storage_paths/custom_fields 的匹配文档计数，供筛选器 UI 显示。

---

## 9. 完整交互流程

```
用户访问 /view/42
       │
       ▼
route.paramMap 触发 ──► savedViewService.getCached(42)
       │
       ├─► 后端：ObjectOwnedOrGrantedPermissionsFilter 过滤
       │     （若用户无权看该视图 → 404）
       │
       ▼
activateSavedViewWithQueryParams(view, queryParams)
       │
       ├─► _activeSavedViewId = 42
       ├─► 从 listViewStates Map 获取状态（或创建默认）
       ├─► 将 view.filter_rules/sort_field/sort_reverse/display_mode 写入
       └─► currentPage = queryParams.page（如存在）
       │
       ▼
reload()
       │
       ├─► documentService.listFiltered(...)
       │     └─► FilterRule[] → queryParamsFromFilterRules → HTTP 请求
       │
       ├─► 后端 DocumentFilterSet + DocumentsOrderingFilter 处理
       │     └─► ObjectOwnedOrGrantedPermissionsFilter 过滤文档
       │
       ├─► 返回分页结果 + selection_data
       ├─► 更新 activeListViewState.documents / collectionSize
       └─► URL 同步（SavedView 模式仅 page）
       │
       ▼
用户修改筛选条件（FilterEditorComponent）
       │
       ▼
onFilterRulesChange → list.setFilterRules(newRules)
       │
       ├─► 更新 activeListViewState.filterRules
       ├─► reload()
       ├─► reduceSelectionToFilter()
       └─► 非 SavedView 时 saveDocumentListView()（localStorage）
       │
       ▼
savedViewIsModified = true（标题显示 *）
       │
       ▼
用户点击"保存视图"
       │
       ├─► 前置检查：canEditSavedView(view)
       │     （需 currentUserHasObjectPermissions(Change, view)）
       │
       ▼
saveViewConfig()
       │
       ├─► 构建 SavedView 对象
       ├─► savedViewService.patch(savedView)
       │     │
       │     └─► SavedViewSerializer.update
       │           ├─► PaperlessObjectPermissions 校验 PUT 权限
       │           ├─► 更新 SavedView 基本字段
       │           └─► 全量替换 SavedViewFilterRule
       │
       └─► 更新 unmodifiedSavedView（重置 dirty）
```

---

## 10. 关键设计点总结

1. **用户 Scope 的分层结构（三层核心 + 两层派生）**：
   - **核心层 1**：对象级权限（owner + django-guardian view/change 对象级权限）— 决定能否看到、编辑、删除具体 SavedView 对象
   - **核心层 2**：全局 model-level 权限（Django 全局 add/view/change/delete）— `*pngxIfPermissions` 指令检查的就是这个，与具体对象无关
   - **核心层 3**：用户个人 UI 偏好（UiSettings 中 dashboard/sidebar 可见 ID）— 决定侧边栏/仪表盘入口
   - **派生层 A**：序列化派生标记（user_can_change、is_shared_by_requester、permissions 字段）— 只读输出，供前端快速判断
   - **派生层 B**：API v9 兼容层（show_on_dashboard/show_in_sidebar 模拟）— 实际读写 UiSettings

2. **full_perms 请求参数二选一**：`full_perms=true` 返回完整权限列表（view/change users/groups），默认返回简化的 `user_can_change` + `is_shared_by_requester`。

3. **currentUserOwnsObject 的精确实现**（[permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/services/permissions.service.ts#L66-L73)）：
   ```typescript
   !object || !object.owner || this.currentUser.is_superuser || object.owner === this.currentUser.id
   ```
   接口名是 `ObjectWithPermissions`，没有 `currentUserIsSuperuser` 这个属性，直接用 `this.currentUser.is_superuser`。

4. **canEditSavedView vs canDeleteSavedView：名称相似但本质不同**：
   - `canEditSavedView` → `currentUserHasObjectPermissions(Change, view)`：范围**更广**，owner/superuser/无主 **OR** 被显式授权 change 权限的非 owner 都能编辑；控制表单字段 disabled
   - `canDeleteSavedView` → `currentUserOwnsObject(view)`：范围**更窄**，仅 owner/superuser/无主，**不包括**仅授权 change 权限的非 owner；控制整个 Actions 列（Permissions + Delete 按钮）是否渲染
   - 关键场景：被授予 change 对象级权限的非 owner 用户 → 可以编辑字段，但看不到 Permissions/Delete 按钮

5. **删除按钮的双重权限判断（AND）**：
   - 外层 `@if (canDeleteSavedView(view))` → `currentUserOwnsObject(view)`（`!object` / owner 为 falsy 无主 / `currentUser.is_superuser` / owner == 当前用户）
   - 内层 `*pngxIfPermissions(Delete, SavedView)` → 全局 `documents.delete_savedview` model-level 权限
   - 后端执行 DELETE 还需再经 `PaperlessObjectPermissions.has_permission`（全局）+ `has_object_permission`（对象级）双重通过
   - 系统**没有**任何 UI 或 API 入口能分发 `delete_savedview` 对象级权限（`set_permissions` 只支持 view/change，[PermissionsFormComponent](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/components/common/input/permissions/permissions-form/permissions-form.component.ts#L62-L74) 的 FormGroup 也只有 view/change 两组）

6. **set_permissions 只支持 view/change，change 自动赋 view**：
   - `validate_set_permissions` 只循环 `["view", "change"]`，其他 action（如 delete）被完全忽略
   - `set_permissions_for_object` 中 `action == "change"` 时会自动 `assign_perm("view_...")`

7. **owner/set_permissions 变更限制：PermissionDenied（403），非 ValidationError（400）**：
   - 触发条件：试图变更 owner 字段或提交 set_permissions，且非 `is_superuser` / `is_owner` / `is_unowned`
   - 异常类型：`rest_framework.exceptions.PermissionDenied` → HTTP 403
   - 错误消息：`_("Insufficient permissions.")`（翻译），不是 "Only superusers, owners or unowned objects can change permissions"

8. **前端全局权限 vs 对象级权限分离**：
   - `*pngxIfPermissions` 指令 → `currentUserCan` → **全局 model-level** Django 权限
   - `canEditSavedView` → **对象级**（currentUserOwnsObject **OR** user_can_change **OR** permissions.change.users/groups）
   - `canDeleteSavedView` → **对象级**（仅 currentUserOwnsObject）
   - Permissions 按钮和 Delete 按钮需要同时满足外层 `@if`（对象级）+ 内层 `*pngxIfPermissions`（全局）

9. **状态隔离**：每个 SavedView 拥有独立的 `ListViewState`（Map key 为 SavedView ID），默认视图（null key）持久化到 localStorage。

10. **Dirty Tracking**：通过 `unmodifiedSavedView` 快照对比检测修改。

11. **全量替换 FilterRule**："先删后建"简化前后端逻辑。

12. **URL 同步策略差异**：非 SavedView 模式完整同步，SavedView 模式仅同步分页。

13. **WebSocket 驱动刷新**：文档消费完成或删除时实时 reload。
