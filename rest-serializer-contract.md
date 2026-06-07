# REST API Serializer 契约传递脉络梳理

本文档从代码实现角度梳理 Paperless-ngx 中 REST API Serializer 契约在前后端之间的传递链路，覆盖**字段来源**、**权限裁剪**、**通用表单消费**和**SavedView 专项分析**四个部分，所有引用均附带实际代码片段和仓库相对路径位置，便于直接复核。

---

## 1. 整体架构总览

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          后端 (Django REST Framework)                         │
│                                                                              │
│  models.py (Model层)                                                         │
│    │  字段类型、ForeignKey、ManyToMany                                        │
│    ▼                                                                         │
│  serialisers.py (Serializer层)                                               │
│    ├── Meta.fields → Model 字段自动映射                                       │
│    ├── 类体内显式声明字段 (SerializerMethodField / PrimaryKeyRelatedField)     │
│    ├── OwnedObjectSerializer 注入权限字段 (permissions / user_can_change ...) │
│    └── MatchingModelSerializer 注入 document_count / slug                    │
│    │                                                                         │
│    ▼                                                                         │
│  views.py (ViewSet层)                                                        │
│    ├── PassUserMixin: 从 query_params 解析 full_perms，注入 user              │
│    ├── BulkPermissionMixin: 批量预取权限 context，避免 N+1                     │
│    └── PaperlessObjectPermissions: owner / guardian 对象级权限校验            │
│                                                                              │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │ HTTP (JSON)
                                ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                            前端 (Angular)                                     │
│                                                                              │
│  data/*.ts (TypeScript 接口) → 静态契约镜像                                   │
│    │                                                                         │
│    ▼                                                                         │
│  services/rest/*.service.ts → 带 full_perms 参数的 HTTP 调用                 │
│    │                                                                         │
│    ▼                                                                         │
│  services/permissions.service.ts → permissions / user_can_change 判定        │
│    │                                                                         │
│    ▼                                                                         │
│  组件层 (表单消费)                                                             │
│    ├── edit-dialog.component.ts → permissions_form 桥接读写字段名             │
│    ├── document-detail.component.ts → 同上模式                                │
│    ├── permissions-dialog.component.ts → 独立权限编辑对话框(带 merge 开关)    │
│    ├── saved-views.component.ts → SavedView 管理页，三种表单消费路径          │
│    └── save-view-config-dialog.component.ts → 新建 SavedView 对话框           │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 字段来源分析

Serializer 字段来自四个维度。

### 2.1 Serializer 继承体系

```
serializers.ModelSerializer
    │
    ├── DynamicFieldsModelSerializer         # 支持 ?fields= 参数动态裁剪
    │       └── DocumentSerializer
    │               └── SearchResultSerializer
    │
    ├── MatchingModelSerializer              # document_count, slug
    │       ├── CorrespondentSerializer
    │       ├── DocumentTypeSerializer
    │       └── TagSerializer
    │
    ├── OwnedObjectSerializer                # 权限字段核心
    │       (SerializerWithPerms + SetPermissionsMixin)
    │       ├── CorrespondentSerializer
    │       ├── DocumentTypeSerializer
    │       ├── TagSerializer
    │       ├── DocumentSerializer
    │       └── SavedViewSerializer
    │
    └── SerializerWithPerms                  # user / full_perms / all_fields 参数载体
```

#### DynamicFieldsModelSerializer —— 字段动态裁剪基类

位置：`src/documents/serialisers.py#L103-L121`

```python
class DynamicFieldsModelSerializer(serializers.ModelSerializer[Any]):
    def __init__(self, *args, **kwargs) -> None:
        fields = kwargs.pop("fields", None)
        super().__init__(*args, **kwargs)
        if fields is not None:
            allowed = set(fields)
            existing = set(self.fields)
            for field_name in existing - allowed:
                self.fields.pop(field_name)
```

前端可通过 `?fields=id,title` 裁剪响应字段（典型用例：DocumentService.getVersions 只取 `id,versions`）。

#### SerializerWithPerms —— 权限参数载体

位置：`src/documents/serialisers.py#L217-L222`

```python
class SerializerWithPerms(serializers.Serializer[dict[str, Any]]):
    def __init__(self, *args, **kwargs) -> None:
        self.user = kwargs.pop("user", None)
        self.full_perms = kwargs.pop("full_perms", False)
        self.all_fields = kwargs.pop("all_fields", False)
        super().__init__(*args, **kwargs)
```

### 2.2 字段来源分解（以 DocumentSerializer 为例）

#### 来源一：Meta.fields 声明的 Model 自动映射

位置：`src/documents/serialisers.py#L1226-L1258`

```python
class DocumentSerializer(...):
    class Meta:
        model = Document
        fields = (
            "id",                    # AutoField
            "correspondent",         # ForeignKey → PrimaryKeyRelatedField
            "document_type",         # ForeignKey
            "storage_path",          # ForeignKey
            "title",                 # CharField
            "content",               # TextField
            "tags",                  # ManyToMany
            "created",               # DateField
            "modified",              # DateTimeField
            "added",                 # DateTimeField
            "deleted_at",            # DateTimeField (SoftDeleteModel)
            "archive_serial_number", # PositiveIntegerField
            "mime_type",             # CharField
            "owner",                 # ForeignKey (也显式覆盖声明)
            ...
        )
        list_serializer_class = OwnedObjectListSerializer
```

对应的 Model 字段定义在：`src/documents/models.py#L157-L306`

#### 来源二：Serializer 内显式声明的自定义字段

位置：`src/documents/serialisers.py#L994-L1028`

```python
class DocumentSerializer(...):
    correspondent = CorrespondentField(allow_null=True)      # 自定义 FK
    tags = TagsField(many=True)                              # 自定义 M2M
    document_type = DocumentTypeField(allow_null=True)
    storage_path = StoragePathField(allow_null=True)

    original_file_name = SerializerMethodField()             # get_original_file_name()
    archived_file_name = SerializerMethodField()             # get_archived_file_name()
    created_date = serializers.DateField(required=False)      # 已废弃
    page_count = SerializerMethodField()
    duplicate_documents = SerializerMethodField()

    notes = NotesSerializer(many=True, required=False, read_only=True)
    root_document = serializers.PrimaryKeyRelatedField(read_only=True)
    versions = SerializerMethodField()

    custom_fields = CustomFieldInstanceSerializer(many=True, allow_null=False, required=False)

    owner = serializers.PrimaryKeyRelatedField(queryset=User.objects.all(), required=False, allow_null=True)

    remove_inbox_tags = serializers.BooleanField(default=False, write_only=True, allow_null=True, required=False)
```

其中典型方法实现：

位置：`src/documents/serialisers.py#L1081-L1088` —— 文件名计算字段

```python
def get_original_file_name(self, obj) -> str | None:
    return obj.original_filename

def get_archived_file_name(self, obj) -> str | None:
    if obj.has_archive_version:
        return obj.get_public_filename(archive=True)
    else:
        return None
```

位置：`src/documents/serialisers.py#L1033-L1041` —— 仅在 retrieve 时返回重复文档

```python
@extend_schema_field(DuplicateDocumentSummarySerializer(many=True))
def get_duplicate_documents(self, obj):
    view = self.context.get("view")
    if view and getattr(view, "action", None) != "retrieve":
        return []
    request = self.context.get("request")
    user = request.user if request else None
    duplicates = _get_viewable_duplicates(obj, user)
    return list(duplicates.values("id", "title", "deleted_at"))
```

#### 来源三：OwnedObjectSerializer 注入的权限字段

位置：`src/documents/serialisers.py#L262-L470`

字段声明在：`src/documents/serialisers.py#L405-L414`

```python
permissions = SerializerMethodField(read_only=True, required=False)
user_can_change = SerializerMethodField(read_only=True, required=False)
is_shared_by_requester = SerializerMethodField(read_only=True, required=False)

set_permissions = SetPermissionsSerializer(
    label="Set permissions",
    allow_empty=True,
    required=False,
    write_only=True,
)
```

核心方法实现：

位置：`src/documents/serialisers.py#L342-L363`

```python
@extend_schema_field(field={...})  # OpenAPI 类型定义
def get_permissions(self, obj) -> dict:
    return {
        "view": {
            "users": self._get_perms(obj, "view", "users"),
            "groups": self._get_perms(obj, "view", "groups"),
        },
        "change": {
            "users": self._get_perms(obj, "change", "users"),
            "groups": self._get_perms(obj, "change", "groups"),
        },
    }

def get_user_can_change(self, obj) -> bool:
    checker = ObjectPermissionChecker(self.user) if self.user is not None else None
    return (
        obj.owner is None
        or obj.owner == self.user
        or (
            self.user is not None
            and checker.has_perm(f"change_{obj.__class__.__name__.lower()}", obj)
        )
    )
```

`_get_perms` 优先从 ViewSet 批量预取的 context 读取，回退到 django-guardian 单查：

位置：`src/documents/serialisers.py#L280-L307`

```python
def _get_perms(self, obj, codename: str, target: Literal["users", "groups"]):
    key = f"{target}_{codename}_perms"
    cached = self.context.get(key, {}).get(obj.pk)
    if cached is not None:
        return list(cached)
    # fallback: 从 django-guardian 单查
    if target == "users":
        return list(get_users_with_perms(
            obj,
            only_with_perms_in=[f"{codename}_{obj.__class__.__name__.lower()}"],
            with_group_users=False,
        ).values_list("id", flat=True))
    else:
        return list(get_groups_with_only_permission(
            obj, codename=f"{codename}_{obj.__class__.__name__.lower()}"
        ).values_list("id", flat=True))
```

#### 来源四：MatchingModelSerializer 注入的匹配字段

位置：`src/documents/serialisers.py#L124-L167`

```python
class MatchingModelSerializer(serializers.ModelSerializer[Any]):
    document_count = serializers.IntegerField(read_only=True)

    def get_slug(self, obj) -> str:
        return slugify(obj.name)
    slug = SerializerMethodField()
```

---

## 3. 权限裁剪机制

### 3.1 PassUserMixin：从 HTTP 请求注入 user / full_perms

位置：`src/documents/views.py#L360-L382`

```python
class PassUserMixin(GenericAPIView[Any]):
    def get_serializer(self, *args, **kwargs):
        serializer_class = self.get_serializer_class()
        if isinstance(serializer_class, type) and issubclass(
            serializer_class, SerializerWithPerms,
        ):
            kwargs.setdefault("user", self.request.user)
            try:
                full_perms = get_boolean(
                    str(self.request.query_params.get("full_perms", "false")),
                )
            except ValueError:
                full_perms = False
            kwargs.setdefault("full_perms", full_perms)
        return super().get_serializer(*args, **kwargs)
```

### 3.2 OwnedObjectSerializer.__init__：根据 full_perms / all_fields 裁剪字段

位置：`src/documents/serialisers.py#L267-L278`

```python
def __init__(self, *args, **kwargs) -> None:
    super().__init__(*args, **kwargs)
    if not self.all_fields:
        try:
            if self.full_perms:
                # 列表 full_perms=true：保留完整 permissions 矩阵，去掉轻量字段
                self.fields.pop("user_can_change")
                self.fields.pop("is_shared_by_requester")
            else:
                # 列表默认：保留轻量 user_can_change，去掉大体积 permissions
                self.fields.pop("permissions")
        except KeyError:
            pass
```

### 3.3 DocumentSerializer：PATCH/PUT 自动触发 full_perms=true（仅 Document 独有）

位置：`src/documents/serialisers.py#L1213-L1224`

DocumentSerializer 重写了 `__init__`，检测到 PATCH/PUT 请求时强制 `full_perms=True`：

```python
def __init__(self, *args, **kwargs) -> None:
    self.truncate_content = kwargs.pop("truncate_content", False)
    context = kwargs.get("context")
    if context is not None and (
        context.get("request").method == "PATCH"
        or context.get("request").method == "PUT"
    ):
        kwargs["full_perms"] = True
    super().__init__(*args, **kwargs)
```

> **关键差异**：SavedViewSerializer、TagSerializer、CorrespondentSerializer 等其他继承 OwnedObjectSerializer 的类**都没有重写 `__init__`**，不会根据 HTTP 方法自动改变 full_perms。它们的 full_perms 完全由 `PassUserMixin` 从 query 参数读取（默认 false）。

### 3.4 字段裁剪结果矩阵

| 场景 | Serializer | `full_perms` 来源 | `full_perms` | `all_fields` | `permissions` | `user_can_change` | `is_shared_by_requester` |
|------|------|------|:---:|:---:|:---:|:---:|:---:|
| 列表 GET (默认) | 所有 OwnedObjectSerializer 子类 | PassUserMixin query 默认 | false | false | ❌ | ✅ | ✅ |
| 列表 GET `?full_perms=true` | 所有 OwnedObjectSerializer 子类 | PassUserMixin query 参数 | true | false | ✅ | ❌ | ❌ |
| PATCH/PUT 响应 | DocumentSerializer | DocumentSerializer.__init__ 自动强制 | **true** | - | ✅ | ❌ | ❌ |
| PATCH/PUT 响应 | SavedView / Tag / Correspondent 等 | PassUserMixin query 默认（请求 URL 通常不带 ?full_perms=true） | **false** | - | ❌ | ✅ | ✅ |
| OpenAPI schema | 所有 | all_fields=True 参数 | - | true | ✅ | ✅ | ✅ |

### 3.5 BulkPermissionMixin：列表批量预取权限（防 N+1）

位置：`src/documents/views.py#L385-L479`

```python
class BulkPermissionMixin:
    def get_serializer_context(self):
        context = super().get_serializer_context()
        full_perms = get_boolean(str(self.request.query_params.get("full_perms", "false")))
        if not full_perms:
            return context
        # 确定分页对象
        page = getattr(self, "paginator", None)
        if page and hasattr(page, "page"):
            queryset = page.page.object_list
        ...
        # 一次性查出所有对象的 user/group view/change 权限
        user_perms = self._get_object_perms(
            objects=queryset,
            perm_codenames=[permission_name_view, permission_name_change],
            actor="users",
        )
        group_perms = self._get_object_perms(...)
        context["users_view_perms"] = { pk: user_perms[pk][permission_name_view] ... }
        context["users_change_perms"] = { ... }
        context["groups_view_perms"] = { ... }
        context["groups_change_perms"] = { ... }
        return context
```

### 3.6 对象级权限：ViewSet 层 + Serializer 层双重校验

**ViewSet permission_classes** 位置：`src/documents/permissions.py#L28-L52`

```python
class PaperlessObjectPermissions(DjangoObjectPermissions):
    perms_map = {
        "GET": ["%(app_label)s.view_%(model_name)s"],
        "POST": ["%(app_label)s.add_%(model_name)s"],
        "PUT": ["%(app_label)s.change_%(model_name)s"],
        "PATCH": ["%(app_label)s.change_%(model_name)s"],
        "DELETE": ["%(app_label)s.delete_%(model_name)s"],
    }
    def has_object_permission(self, request, view, obj):
        if hasattr(obj, "owner") and obj.owner is not None:
            if request.user == obj.owner:
                return True
            else:
                return super().has_object_permission(request, view, obj)
        else:
            return True  # 无 owner 视为公开
```

**Serializer.update() 二次校验 owner/权限变更** 位置：`src/documents/serialisers.py#L452-L469`

```python
def update(self, instance, validated_data):
    is_superuser = user.is_superuser if user is not None else False
    is_owner = instance.owner == user if user is not None else False
    is_unowned = instance.owner is None
    if (("owner" in validated_data and validated_data["owner"] != instance.owner)
        or "set_permissions" in validated_data) \
       and not (is_superuser or is_owner or is_unowned):
        raise PermissionDenied(_("Insufficient permissions."))
    ...
```

### 3.7 document_count 权限感知

位置：`src/documents/permissions.py#L207-L220`

```python
def get_document_count_filter_for_user(user):
    if getattr(user, "is_superuser", False):
        return Q(documents__deleted_at__isnull=True)
    permitted_ids = _permitted_document_ids(user)  # owner + 显式授权的子查询
    return Q(documents__id__in=permitted_ids)
```

---

## 4. 前端通用表单消费链路

### 4.1 TypeScript 接口层 —— 静态契约镜像

位置：`src-ui/src/app/data/object-with-permissions.ts`

```typescript
export interface PermissionsObject {
  view: { users: Array<number>; groups: Array<number> }
  change: { users: Array<number>; groups: Array<number> }
}

export interface ObjectWithPermissions extends ObjectWithId {
  owner?: number
  permissions?: PermissionsObject
  user_can_change?: boolean
  is_shared_by_requester?: boolean
}
```

位置：`src-ui/src/app/data/document.ts#L115-L170`

```typescript
export interface Document extends ObjectWithPermissions {
  correspondent?: number
  document_type?: number
  tags?: number[]
  title?: string
  created?: string           // ISO string
  notes?: DocumentNote[]
  custom_fields?: CustomFieldInstance[]
  remove_inbox_tags?: boolean // write-only
  ...
}
```

### 4.2 REST Service 层 —— full_perms 参数控制

**DocumentService.get() 默认带 full_perms=true**

位置：`src-ui/src/app/services/rest/document.service.ts#L196-L213`

```typescript
get(id: number, versionID: number = null, fields: string = null): Observable<Document> {
    const params = { full_perms: true, version?, fields? }
    return this.http.get<Document>(this.getResourceUrl(id), { params })
}
```

**PATCH 写入时附带 write_only 的 remove_inbox_tags**

位置：`src-ui/src/app/services/rest/document.service.ts#L305-L313`

```typescript
patch(o: Document, versionID: number = null): Observable<Document> {
    o.remove_inbox_tags = !!this.settingsService.get(
        SETTINGS_KEYS.DOCUMENT_EDITING_REMOVE_INBOX_TAGS
    )
    this.clearCache()
    return this.http.patch<Document>(this.getResourceUrl(o.id), o, {
        params: versionID ? { version: versionID.toString() } : {},
    })
}
```

### 4.3 PermissionsService —— 权限字段的判定消费

位置：`src-ui/src/app/services/permissions.service.ts#L75-L97`

```typescript
currentUserHasObjectPermissions(action: string, object: ObjectWithPermissions): boolean {
    if (action === PermissionAction.View) {
        return (
            this.currentUserOwnsObject(object) ||
            object.permissions?.view.users.includes(this.currentUser.id) ||
            object.permissions?.view.groups.filter(g =>
                this.currentUser.groups.includes(g)
            ).length > 0
        )
    } else if (action === PermissionAction.Change) {
        return (
            this.currentUserOwnsObject(object) ||
            object.user_can_change ||  // ← 后端返回的轻量布尔值
            object.permissions?.change.users.includes(this.currentUser.id) ||
            object.permissions?.change.groups.filter(g =>
                this.currentUser.groups.includes(g)
            ).length > 0
        )
    }
}
```

### 4.4 EditDialogComponent —— 通用编辑对话框的 permissions_form 桥接

**初始化：GET 响应 → permissions_form**

位置：`src-ui/src/app/components/common/edit-dialog/edit-dialog.component.ts#L72-L116`

```typescript
ngOnInit(): void {
    if (this.object != null && this.dialogMode !== EditDialogMode.CREATE) {
        // 后端返回的 permissions 字段 → 前端内部的 permissions_form.set_permissions
        this.object['permissions_form'] = {
            owner: (this.object as ObjectWithPermissions).owner,
            set_permissions: (this.object as ObjectWithPermissions).permissions,
        }
        this.objectForm.patchValue(this.object)
    } else {
        // 新建模式从 settings 读取默认值
        this.objectForm.patchValue({
            permissions_form: {
                owner: this.settingsService.get(SETTINGS_KEYS.DEFAULT_PERMS_OWNER),
                set_permissions: {
                    view: { users: ..., groups: ... },
                    change: { users: ..., groups: ... },
                },
            },
        })
    }
}
```

**提交：permissions_form → 请求体的 owner / set_permissions**

位置：`src-ui/src/app/components/common/edit-dialog/edit-dialog.component.ts#L159-L196`

```typescript
save() {
    const formValues = this.getFormValues()
    const permissionsObject: PermissionsFormObject =
        this.objectForm.get('permissions_form')?.value
    if (permissionsObject && this.shouldSubmitPermissions()) {
        // 前端内部的 permissions_form → 后端契约的 owner / set_permissions
        formValues.owner = permissionsObject.owner
        formValues.set_permissions = permissionsObject.set_permissions
    }
    delete formValues.permissions_form

    if (!this.shouldSubmitPermissions()) {
        delete newObject['set_permissions']
    }
    // PATCH / PUT ...
}
```

字段名映射的关键约定：

| 方向 | 后端字段 | 前端中间层 | 说明 |
|------|---------|-----------|------|
| 响应 (GET) | `permissions` | `permissions_form.set_permissions` | 只读权限矩阵 |
| 响应 (GET) | `owner` | `permissions_form.owner` | 所有者 |
| 请求 (PATCH/POST) | `set_permissions` | `permissions_form.set_permissions` | 写入权限矩阵 |
| 请求 (PATCH/POST) | `owner` | `permissions_form.owner` | 写入所有者 |

### 4.5 PermissionsFormComponent —— 权限编辑表单 UI

位置：`src-ui/src/app/components/common/input/permissions/permissions-form/permissions-form.component.ts#L17-L97`

```typescript
export interface PermissionsFormObject {
  owner?: number
  set_permissions?: {
    view?: { users?: number[]; groups?: number[] }
    change?: { users?: number[]; groups?: number[] }
  }
}

export class PermissionsFormComponent ... {
  form = new FormGroup({
    owner: new FormControl(null),
    set_permissions: new FormGroup({
      view: new FormGroup({
        users: new FormControl([]),
        groups: new FormControl([]),
      }),
      change: new FormGroup({
        users: new FormControl([]),
        groups: new FormControl([]),
      }),
    }),
  })
}
```

对应模板 HTML 结构位置：`src-ui/src/app/components/common/input/permissions/permissions-form/permissions-form.component.html`

层级为：
- `owner` → Select 用户选择器
- `set_permissions.view.users` / `set_permissions.view.groups`
- `set_permissions.change.users` / `set_permissions.change.groups`

### 4.6 PermissionsDialogComponent —— 独立权限对话框（带 merge 开关，但并非所有调用方都消费）

位置：`src-ui/src/app/components/common/permissions-dialog/permissions-dialog.component.ts#L26-L104`

PermissionsDialogComponent 自身表单里包含 `merge` 开关（默认 true），confirm 时发射 `{ permissions, merge }`。**但 merge 参数只在「批量权限编辑」链路中被真正消费**，单对象权限编辑的调用方会忽略它。

```typescript
export class PermissionsDialogComponent {
  @Input()
  set object(o: ObjectWithPermissions) {
    this.o = o
    this.form.patchValue({
      merge: true,
      permissions_form: {
        owner: o.owner,
        set_permissions: o.permissions,  // GET 的 permissions → 写入的 set_permissions
      },
    })
  }

  public form = new FormGroup({
    permissions_form: new FormControl(),
    merge: new FormControl(true),  // ← 是否合并而非覆盖（仅批量编辑生效）
  })

  get permissions() {
    return {
      owner: this.form.get('permissions_form').value?.owner ?? null,
      set_permissions: this.form.get('permissions_form').value?.set_permissions ?? {
        view: { users: [], groups: [] },
        change: { users: [], groups: [] },
      },
    }
  }

  confirm() {
    this.confirmClicked.emit({
      permissions: this.permissions,
      merge: this.form.get('merge').value,  // ← 发射 merge，但调用方未必使用
    })
  }
}
```

**merge 参数消费链路全景：**

| 调用方 | 是否消费 merge | 原因 |
|--------|:---:|------|
| SavedViewsComponent.editPermissions() | ❌ 忽略 | 单对象 PATCH，`({ permissions })` 解构时丢弃 merge，后端单对象 PATCH 接口也无 merge 参数 |
| MailComponent.editPermissions() | ❌ 忽略 | 虽然解构了 `({ permissions, merge })`，但后续代码完全未使用 merge 变量，仍走单对象 patch() |
| ManagementListComponent.setPermissions() | ✅ 消费 | 调用 `service.bulk_edit_objects(..., permissions, merge, ...)`，merge 传入批量接口 |
| SaveViewConfigDialog（新建 SavedView） | — 不涉及 | 内嵌 permissions_form，无 merge 开关 |
| EditDialogComponent（通用编辑） | — 不涉及 | 内嵌 permissions_form，无 merge 开关 |

后端 `merge` 参数的消费位置：`src/documents/permissions.py#L93-L163` 中的 `set_permissions_for_object(permissions, object, *, merge=False)`，该函数仅被 `BulkEditObjectsView` 和批量编辑链路调用：

```python
def set_permissions_for_object(permissions, object, *, merge: bool = False) -> None:
    for action, entry in permissions.items():
        permission = f"{action}_{object.__class__.__name__.lower()}"
        if "users" in entry:
            users_to_remove = get_users_with_perms(...) if not merge else User.objects.none()
            ...
            for user in users_to_add:
                assign_perm(permission, user, object)
                if action == "change":
                    assign_perm(f"view_{...}", user, object)  # change 隐含 view
```

### 4.7 OpenAPI Schema (drf-spectacular)

位置：`src/documents/schema.py#L29-L43`

```python
def generate_object_with_permissions_schema(serializer_class):
    return {
        operation: extend_schema(
            parameters=[
                OpenApiParameter(name="full_perms", type=OpenApiTypes.BOOL,
                                 location=OpenApiParameter.QUERY),
            ],
            responses={
                200: serializer_class(many=operation == "list", all_fields=True),
            },
        )
        for operation in ["list", "retrieve"]
    }
```

应用于 ViewSet，例如位置：`src/documents/views.py#L523`

```python
@extend_schema_view(**generate_object_with_permissions_schema(CorrespondentSerializer))
class CorrespondentViewSet(...):
```

> **注意**：前端未使用 OpenAPI 生成代码，而是手动维护 TypeScript interface；Schema 主要用于 API 文档和 Swagger UI。

---

## 5. SavedView 保存视图的权限表单消费链路（专项分析）

SavedView 与 Tag/Correspondent 的区别在于它有**三种不同的权限表单消费路径**，且后端 Serializer 有额外的 API 版本兼容逻辑。

### 5.1 后端 SavedViewSerializer 完整实现

位置：`src/documents/serialisers.py#L1318-L1533`

**字段声明与 Meta：**

```python
class SavedViewFilterRuleSerializer(serializers.ModelSerializer[SavedViewFilterRule]):
    class Meta:
        model = SavedViewFilterRule
        fields = ["rule_type", "value"]

class SavedViewSerializer(OwnedObjectSerializer):
    filter_rules = SavedViewFilterRuleSerializer(many=True)  # 嵌套序列化

    class Meta:
        model = SavedView
        fields = [
            "id",
            "name",
            "sort_field",
            "sort_reverse",
            "filter_rules",
            "page_size",
            "display_mode",
            "display_fields",
            "owner",
            "permissions",            # ← 继承自 OwnedObjectSerializer
            "user_can_change",        # ← 继承自 OwnedObjectSerializer
            "set_permissions",        # ← 继承自 OwnedObjectSerializer (write_only)
        ]
```

> **full_perms 关键差异**：SavedViewSerializer **没有重写 `__init__`**，完全继承 OwnedObjectSerializer 的字段裁剪逻辑。它不会像 DocumentSerializer 那样在 PATCH/PUT 时自动强制 full_perms=true。SavedView 的 full_perms 完全由请求 URL 的 query 参数决定（通过 PassUserMixin 解析）。因此：
> - `GET /api/saved_views/?full_perms=true` → 响应含完整 `permissions` 矩阵，不含 `user_can_change`
> - `GET /api/saved_views/`（默认）→ 响应含 `user_can_change`，不含完整 `permissions`
> - `PATCH /api/saved_views/{id}/`（默认）→ 响应含 `user_can_change`，**不含完整 `permissions`**，前端必须重新 GET 才能拿到更新后的权限矩阵

**filter_rules 嵌套写入：** `update()` 中先删后重建

位置：`src/documents/serialisers.py#L1480-L1512`

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

**display_fields 校验：**

位置：`src/documents/serialisers.py#L1462-L1478`

```python
def validate(self, attrs):
    attrs = super().validate(attrs)
    if "display_fields" in attrs and attrs["display_fields"] is not None:
        for field in attrs["display_fields"]:
            if SavedView.DisplayFields.CUSTOM_FIELD[:-2] in field:  # 'custom_field_'
                field_id = int(re.search(r"\d+", field)[0])
                if not CustomField.objects.filter(id=field_id).exists():
                    raise serializers.ValidationError(f"Invalid field: {field}")
            elif field not in SavedView.DisplayFields.values:
                raise serializers.ValidationError(f"Invalid field: {field}")
    return attrs
```

**Legacy 兼容性（API v9 → v10）：** `show_on_dashboard` / `show_in_sidebar` 字段不再存在于 Model，而是迁移到 UiSettings.settings JSON 中。Serializer 在 `to_representation`（旧版本响应）和 `to_internal_value`（旧版本请求）做了桥接：

位置：`src/documents/serialisers.py#L1409-L1460`

```python
def to_representation(self, instance):
    # API v9 及以下：响应里附带 show_on_dashboard / show_in_sidebar
    if api_version < 10:
        ret["show_on_dashboard"] = instance.id in dashboard_ids
        ret["show_in_sidebar"] = instance.id in sidebar_ids
    return ret

def to_internal_value(self, data):
    # API v9 及以下：从请求中剥离 show_on_dashboard / show_in_sidebar，留到 update() 处理
    if api_version >= 10:
        return super().to_internal_value(data)
    ...
    ret.update(legacy_visibility_fields)
    return ret
```

### 5.2 后端 SavedViewViewSet

位置：`src/documents/views.py#L2547-L2562`

```python
@extend_schema_view(**generate_object_with_permissions_schema(SavedViewSerializer))
class SavedViewViewSet(BulkPermissionMixin, PassUserMixin, ModelViewSet[SavedView]):
    model = SavedView
    queryset = SavedView.objects.select_related("owner").prefetch_related("filter_rules")
    serializer_class = SavedViewSerializer
    pagination_class = StandardPagination
    permission_classes = (IsAuthenticated, PaperlessObjectPermissions)
    filter_backends = (OrderingFilter, ObjectOwnedOrGrantedPermissionsFilter)
    ordering_fields = ("name",)
```

### 5.3 前端 SavedView TypeScript 接口

位置：`src-ui/src/app/data/saved-view.ts`

```typescript
export interface SavedView extends ObjectWithPermissions {
  name?: string
  show_on_dashboard?: boolean   // 前端扩展字段（从 UiSettings 合并）
  show_in_sidebar?: boolean     // 同上
  sort_field: string
  sort_reverse: boolean
  filter_rules: FilterRule[]
  page_size?: number
  display_mode?: DisplayMode
  display_fields?: DisplayField[]
}
```

### 5.4 前端 SavedViewService

位置：`src-ui/src/app/services/rest/saved-view.service.ts`

关键点：**Service 的 `list()` 本身不传 full_perms，由调用方决定**；返回时还会从 UiSettings 合并 `show_on_dashboard` / `show_in_sidebar`。

```typescript
export class SavedViewService extends AbstractPaperlessService<SavedView> {
  constructor() {
    super()
    this.resourceName = 'saved_views'
  }

  list(page?, pageSize?, sortField?, sortReverse?, extraParams?): Observable<Results<SavedView>> {
    return super.list(page, pageSize, sortField, sortReverse, extraParams).pipe(
      tap({ next: (r) => {
          const views = r.results.map((view) => this.withUserVisibility(view))
          this.savedViews = views
          r.results = views
      }})
    )
  }

  private withUserVisibility(view: SavedView): SavedView {
    return {
      ...view,
      show_on_dashboard: this.isDashboardVisible(view),
      show_in_sidebar: this.isSidebarVisible(view),
    }
  }

  patch(o: SavedView, reload = false): Observable<SavedView> {
    if (o.display_fields?.length === 0) {
      o.display_fields = null
    }
    return super.patch(o).pipe(...)
  }

  patchMany(objects: SavedView[]): Observable<SavedView[]> {
    return combineLatest(objects.map((o) => this.patch(o, false))).pipe(...)
  }
}
```

### 5.5 SavedView 的三种权限表单消费路径

#### 路径 A：SavedViewsComponent 管理页 —— 独立 PermissionsDialog

位置：`src-ui/src/app/components/manage/saved-views/saved-views.component.ts#L81-L89`

**加载列表时显式传 full_perms=true：**

```typescript
private reloadViews(): void {
    this.loading = true
    this.savedViewService
        .list(null, null, null, false, { full_perms: true })
        .subscribe((r) => {
            this.savedViews = r.results
            this.initialize()
        })
}
```

**权限编辑按钮触发 PermissionsDialog：**

位置：`src-ui/src/app/components/manage/saved-views/saved-views.component.ts#L251-L280`

```typescript
public editPermissions(savedView: SavedView): void {
    const modal = this.modalService.open(PermissionsDialogComponent, {
        backdrop: 'static',
    })
    const dialog = modal.componentInstance as PermissionsDialogComponent
    dialog.object = savedView   // ← object setter 里做 permissions → set_permissions 桥接

    // 关键：只解构 permissions，完全忽略 merge 参数
    // PermissionsDialog 发射 { permissions, merge }，但 SavedView 单对象 PATCH 不支持 merge
    modal.componentInstance.confirmClicked.subscribe(({ permissions }) => {
        modal.componentInstance.buttonsEnabled = false
        const view = {
            id: savedView.id,
            owner: permissions.owner,
        }
        view['set_permissions'] = permissions.set_permissions  // 总是全量覆盖，不支持合并
        this.savedViewService.patch(view as SavedView).subscribe({
            next: () => { this.toastService.showInfo(...); modal.close(); this.reloadViews() },
            ...
        })
    })
}
```

> **merge 对 SavedView 无效**：SavedView 管理页的权限编辑走单对象 PATCH 接口，而后端 `SavedViewSerializer.update()` → `OwnedObjectSerializer.update()` → `_set_permissions()` 调用 `set_permissions_for_object()` 时**不传递 merge 参数**（默认 merge=False），因此 SavedView 的权限编辑总是**全量覆盖**，对话框中的 merge 开关对 SavedView 没有任何实际效果。merge 参数仅对 ManagementListComponent（Tag/Correspondent/DocumentType/StoragePath）的批量权限编辑链路有效。

**权限可见性控制：**

位置：`src-ui/src/app/components/manage/saved-views/saved-views.component.ts#L240-L249`

```typescript
public canEditSavedView(view: SavedView): boolean {
    return this.permissionsService.currentUserHasObjectPermissions(
        PermissionAction.Change, view
    )
}

public canDeleteSavedView(view: SavedView): boolean {
    return this.permissionsService.currentUserOwnsObject(view)
}
```

`initialize()` 里根据 `canEditSavedView()` 动态禁用 FormControl：

位置：`src-ui/src/app/components/manage/saved-views/saved-views.component.ts#L96-L142`

```typescript
for (let view of this.savedViews) {
    const canEdit = this.canEditSavedView(view)
    this.savedViewsGroup.addControl(view.id.toString(), new FormGroup({
        id: new FormControl({ value: null, disabled: !canEdit }),
        name: new FormControl({ value: null, disabled: !canEdit }),
        show_on_dashboard: new FormControl({ value: null, disabled: false }),  // 所有人都能改可见性
        show_in_sidebar: new FormControl({ value: null, disabled: false }),
        page_size: new FormControl({ value: null, disabled: !canEdit }),
        display_mode: new FormControl({ value: null, disabled: !canEdit }),
        display_fields: new FormControl({ value: [], disabled: !canEdit }),
    }))
}
```

#### 路径 B：SaveViewConfigDialogComponent —— 新建保存视图对话框

位置：`src-ui/src/app/components/document-list/save-view-config-dialog/save-view-config-dialog.component.ts`

**FormGroup 结构（含 permissions_form）：**

```typescript
saveViewConfigForm = new FormGroup({
    name: new FormControl(''),
    showInSideBar: new FormControl(false),
    showOnDashboard: new FormControl(false),
    permissions_form: new FormControl(null),  // ← 嵌入权限表单
})
```

模板中直接嵌入权限编辑组件，使用 `accordion=true` 折叠模式：

位置：`src-ui/src/app/components/document-list/save-view-config-dialog/save-view-config-dialog.component.html#L11`

```html
<pngx-permissions-form accordion="true" formControlName="permissions_form"></pngx-permissions-form>
```

**提交时的字段转换**（由 DocumentListComponent.saveViewConfigAs() 处理）：

位置：`src-ui/src/app/components/document-list/document-list.component.ts#L448-L510`

```typescript
saveViewConfigAs() {
    modal.componentInstance.saveClicked.pipe(first()).subscribe((formValue) => {
        let savedView: SavedView = {
            name: formValue.name,
            filter_rules: this.list.filterRules,
            sort_reverse: this.list.sortReverse,
            sort_field: this.list.sortField,
            display_mode: this.list.displayMode,
            display_fields: this.activeDisplayFields,
        }
        const permissions = formValue.permissions_form
        if (permissions) {
            if (permissions.owner !== null && permissions.owner !== undefined) {
                savedView.owner = permissions.owner
            }
            if (permissions.set_permissions) {
                savedView['set_permissions'] = permissions.set_permissions  // ← 关键转换
            }
        }
        this.savedViewService.create(savedView).subscribe(...)
    })
}
```

#### 路径 C：DocumentListComponent.saveViewConfig() —— 覆盖保存（不含权限）

位置：`src-ui/src/app/components/document-list/document-list.component.ts#L398-L428`

这是最简单的路径：用户点击"保存当前视图"按钮，只更新 filter_rules / sort_field 等字段，**完全不涉及权限**。

```typescript
saveViewConfig() {
    if (this.list.activeSavedViewId != null && this.activeSavedViewCanChange) {
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
}
```

权限可编辑性判定使用：
```typescript
this.activeSavedViewCanChange = this.permissionsService.currentUserHasObjectPermissions(
    PermissionAction.Change, this.activeSavedView
)
```

### 5.6 SavedView 三种消费路径对比表

| 路径 | 场景 | 是否带权限 | full_perms | 字段转换方式 |
|------|------|:---:|:---:|---|
| A. SavedViewsComponent.editPermissions | 管理页单独修改权限 | ✅ | 请求列表时 `full_perms=true` | PermissionsDialog.object setter → `permissions` → `set_permissions`，emit 时手动组装 patch body |
| B. SaveViewConfigDialog | 新建保存视图 | ✅ | 新建不涉及 GET，直接 POST | DocumentListComponent.saveViewConfigAs() 里 `permissions_form` → `owner` + `['set_permissions']` |
| C. saveViewConfig() | 覆盖保存已有视图 | ❌ | 不涉及权限字段 | 直接 PATCH，不含任何权限相关字段 |

### 5.7 批量权限编辑：bulk_edit_objects（merge 参数唯一有效的链路）

SavedView **目前未纳入批量编辑**（`object_type` 只支持 tags/correspondents/document_types/storage_paths），因此 PermissionsDialogComponent 中的 `merge` 开关对 SavedView 完全无效。merge 参数仅在以下批量编辑链路中被真正消费：

```
ManagementListComponent.setPermissions() （Tag/Correspondent/DocumentType/StoragePath 管理页）
    ↓ PermissionsDialog confirmClicked.emit({ permissions, merge })
    ↓ ManagementList 解构并消费 merge
    ↓ AbstractNameFilterService.bulk_edit_objects(..., permissions, merge, ...)
    ↓ POST /api/{resource}/bulk_edit_objects/
    ↓ BulkEditObjectsView
    ↓ set_permissions_for_object(permissions=..., object=obj, merge=merge)
```

而 SavedView 管理页的 editPermissions() 是单对象 PATCH 链路，merge 参数在回调解构时就被丢弃：

```
SavedViewsComponent.editPermissions()
    ↓ PermissionsDialog confirmClicked.emit({ permissions, merge })
    ↓ subscribe(({ permissions }) => { ... })  // ← merge 在此处被解构丢弃
    ↓ SavedViewService.patch(view)  // 单对象 PATCH，无 merge 参数
    ↓ SavedViewSerializer.update() → OwnedObjectSerializer.update()
    ↓ _set_permissions() → set_permissions_for_object(permissions, object)  // 默认 merge=False
```

**后端 BulkEditObjectsSerializer：**

位置：`src/documents/serialisers.py#L2783-L2896`

```python
class BulkEditObjectsSerializer(SerializerWithPerms, SetPermissionsMixin):
    object_type = serializers.ChoiceField(
        choices=["tags", "correspondents", "document_types", "storage_paths"],
        write_only=True,
    )
    operation = serializers.ChoiceField(choices=["set_permissions", "delete"], write_only=True)
    owner = serializers.PrimaryKeyRelatedField(queryset=User.objects.all(), required=False, allow_null=True)
    permissions = serializers.DictField(required=False, write_only=True)
    merge = serializers.BooleanField(default=False, write_only=True, required=False)
    ...
```

**后端 BulkEditObjectsView：**

位置：`src/documents/views.py#L4543-L4639`

```python
class BulkEditObjectsView(PassUserMixin):
    def post(self, request, *args, **kwargs):
        ...
        if operation == "set_permissions":
            permissions = serializer.validated_data.get("permissions")
            owner = serializer.validated_data.get("owner")
            merge = serializer.validated_data.get("merge")
            if "owner" in serializer.validated_data and (not merge or (merge and owner is not None)):
                qs_owner_update = qs.filter(owner__isnull=True) if merge else qs
                qs_owner_update.update(owner=owner)
            if "permissions" in serializer.validated_data:
                for obj in qs:
                    set_permissions_for_object(permissions=permissions, object=obj, merge=merge)
```

**前端 AbstractNameFilterService.bulk_edit_objects：**

位置：`src-ui/src/app/services/rest/abstract-name-filter-service.ts#L36-L62`

```typescript
bulk_edit_objects(objects, operation, permissions = null, merge = null, all = false, filters = null) {
    const params: any = { object_type: this.resourceName, operation }
    if (operation === BulkEditObjectOperation.SetPermissions) {
        params['owner'] = permissions?.owner
        params['permissions'] = permissions?.set_permissions
        params['merge'] = merge
    }
    return this.http.post<string>(`${this.baseUrl}bulk_edit_objects/`, params)
}
```

---

## 6. 完整数据流示例：SavedView 权限编辑

### Step 1：SavedViewsComponent 加载列表（前端显式传 full_perms=true）

```
GET /api/saved_views/?full_perms=true
```

后端：
1. 前端 SavedViewsComponent.reloadViews() 在调用 list() 时显式传入 `{ full_perms: true }` 作为 extraParams
2. `PassUserMixin.get_serializer()` 从 query_params 解析到 `full_perms=true`，连同 `user` 一起注入 Serializer
3. `BulkPermissionMixin.get_serializer_context()` 检测到 full_perms=true，预取所有 SavedView 的权限到 context 缓存
4. `SavedViewSerializer.__init__()` → 继承 `OwnedObjectSerializer.__init__()`：full_perms=true → 移除 `user_can_change` 和 `is_shared_by_requester`，保留完整 `permissions` 矩阵
5. `get_permissions()` 从 context 批量缓存读取，返回完整权限矩阵

### Step 2：响应 JSON

```json
{
  "count": 2,
  "results": [
    {
      "id": 5,
      "name": "Invoices",
      "sort_field": "created",
      "sort_reverse": true,
      "filter_rules": [...],
      "owner": 1,
      "permissions": {
        "view": { "users": [2, 3], "groups": [1] },
        "change": { "users": [2], "groups": [] }
      }
    }
  ]
}
```

注意：响应中**没有** `user_can_change` 和 `is_shared_by_requester`（因为 full_perms=true）。

### Step 3：前端渲染 + 权限编辑

1. `SavedViewsComponent.reloadViews()` 收到结果，`SavedViewService.withUserVisibility()` 附加 `show_on_dashboard/show_in_sidebar`
2. 用户点击"编辑权限" → 打开 `PermissionsDialogComponent`
3. `dialog.object = savedView` 触发 setter，把 `savedView.permissions` → `form.permissions_form.set_permissions`

### Step 4：用户提交 PATCH（merge 参数在此被忽略）

```typescript
// SavedViewsComponent.editPermissions() 的回调：只解构 permissions，完全丢弃 merge
modal.componentInstance.confirmClicked.subscribe(({ permissions }) => {
    const view = {
        id: savedView.id,
        owner: permissions.owner,            // 可能为 null
    }
    view['set_permissions'] = permissions.set_permissions  // e.g. {view:..., change:...}
    this.savedViewService.patch(view as SavedView)
})
```

> **merge 被丢弃的位置**：PermissionsDialogComponent.confirm() 实际发射 `{ permissions, merge }`，但 SavedViewsComponent 的 subscribe 回调只解构了 `{ permissions }`，merge 对象在此处即被丢弃，根本不会传到后端。

发出请求：
```
PATCH /api/saved_views/5/
Content-Type: application/json

{
  "id": 5,
  "owner": 1,
  "set_permissions": {
    "view": { "users": [2, 3, 4], "groups": [1] },
    "change": { "users": [2], "groups": [1] }
  }
}
```

注意：请求 body 中**没有** `merge` 字段，因为 SavedViewService.patch() 走单对象 PATCH 接口，而 `merge` 字段只在 `bulk_edit_objects/` 批量接口的请求体中定义。

### Step 5：后端处理写入 + 前端 reload 获取完整权限

后端处理写入：
1. `PassUserMixin.get_serializer()`：请求 URL `PATCH /api/saved_views/5/` **不带** `?full_perms=true` query 参数，因此解析为 `full_perms=false`，注入 Serializer
2. `SavedViewSerializer.__init__()` → 继承 `OwnedObjectSerializer.__init__()`：full_perms=false → 移除 `permissions` 字段，只保留 `user_can_change` 和 `is_shared_by_requester`（注意：SavedViewSerializer **没有** DocumentSerializer 那种 PATCH 自动 full_perms=true 的逻辑）
3. `validate_set_permissions()` 校验所有 user_id/group_id 存在
4. `OwnedObjectSerializer.update()` 校验当前用户是 owner / superuser（否则 PermissionDenied）
5. `_set_permissions()` → `set_permissions_for_object()` 通过 django-guardian 更新权限表
   - **注意**：`_set_permissions()` 调用时**不传 merge 参数**，使用函数默认值 `merge=False`，因此 SavedView 单对象权限编辑总是**全量覆盖**而非合并
   - change 权限会自动附带 view 权限
6. **PATCH 响应**：因为 full_perms=false，响应中**不包含完整 `permissions` 矩阵**，只有 `user_can_change`、`is_shared_by_requester` 等轻量字段

前端 patch 成功后的动作（关键！SavedView 前端不靠 PATCH 响应拿完整权限）：

```typescript
this.savedViewService.patch(view as SavedView).subscribe({
    next: () => {
        this.toastService.showInfo($localize`Permissions updated`)
        modal.close()
        this.reloadViews()  // ← 重新 GET /api/saved_views/?full_perms=true 获取完整权限
    },
})
```

> **Document vs SavedView 对比**：
> - Document PATCH：后端 DocumentSerializer.__init__ 自动强制 full_perms=true，PATCH 响应直接带完整 permissions，前端无需额外 reload
> - SavedView PATCH：后端无自动强制，PATCH 响应只有 user_can_change，前端必须显式 reloadViews() 重新 GET 才能拿到更新后的完整权限矩阵

---

## 7. 关键代码索引

### 后端

| 路径（仓库根目录相对位置） | 职责 |
|------|------|
| `src/documents/serialisers.py` | 所有 Serializer：字段声明、权限裁剪、权限读写、SavedView filter_rules 重建、BulkEditObjectsSerializer |
| `src/documents/permissions.py` | `PaperlessObjectPermissions`、`set_permissions_for_object()`（含 merge）、权限感知 document_count |
| `src/documents/views.py` | `PassUserMixin`（传 user/full_perms）、`BulkPermissionMixin`（批量预取）、`SavedViewViewSet`、`BulkEditObjectsView` |
| `src/documents/schema.py` | `generate_object_with_permissions_schema()` —— drf-spectacular full_perms 参数 schema |
| `src/documents/models.py` | Django Model 定义，Serializer 字段来源根基 |

### 前端

| 路径（仓库根目录相对位置） | 职责 |
|------|------|
| `src-ui/src/app/data/object-with-permissions.ts` | `PermissionsObject` / `ObjectWithPermissions` TS 接口 |
| `src-ui/src/app/data/document.ts` | `Document` TS 接口 |
| `src-ui/src/app/data/saved-view.ts` | `SavedView` TS 接口 |
| `src-ui/src/app/services/rest/abstract-paperless-service.ts` | 通用 REST CRUD 基类 |
| `src-ui/src/app/services/rest/abstract-name-filter-service.ts` | 支持 `fullPerms` 和 `bulk_edit_objects` 的基类 |
| `src-ui/src/app/services/rest/document.service.ts` | `DocumentService`：GET 默认 full_perms=true、PATCH 附 remove_inbox_tags |
| `src-ui/src/app/services/rest/saved-view.service.ts` | `SavedViewService`：列表合并 show_on_dashboard、patchMany |
| `src-ui/src/app/services/permissions.service.ts` | `currentUserHasObjectPermissions()` —— user_can_change / permissions 矩阵判定 |
| `src-ui/src/app/components/common/edit-dialog/edit-dialog.component.ts` | 通用编辑对话框，permissions ↔ permissions_form 桥接 |
| `src-ui/src/app/components/common/permissions-dialog/permissions-dialog.component.ts` | 独立权限对话框（含 merge 开关），SavedView 管理页使用 |
| `src-ui/src/app/components/common/input/permissions/permissions-form/` | 权限编辑表单 UI 组件（owner + view/change users + groups） |
| `src-ui/src/app/components/manage/saved-views/saved-views.component.ts` | SavedView 管理页：full_perms=true 加载列表、PermissionsDialog 改权限 |
| `src-ui/src/app/components/document-list/save-view-config-dialog/` | 新建 SavedView 对话框：内嵌折叠的权限表单 |
| `src-ui/src/app/components/document-list/document-list.component.ts` | saveViewConfigAs() 新建（含权限）、saveViewConfig() 覆盖保存（不含权限） |
