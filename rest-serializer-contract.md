# REST API Serializer 契约传递脉络梳理

本文档从代码实现角度梳理 Paperless-ngx 中 REST API Serializer 契约在前后端之间的传递链路，包括**字段来源**、**权限裁剪**和**表单消费**三个核心环节。

---

## 1. 整体架构总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              后端 (Django REST Framework)                    │
│                                                                             │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────────────┐   │
│  │  Model 层    │───▶│  Serializer 层    │───▶│  ViewSet (PassUserMixin) │   │
│  │  (models.py) │    │ (serialisers.py)  │    │      (views.py)          │   │
│  └──────────────┘    └──────────────────┘    └──────────┬───────────────┘   │
│                          ▲                               │                   │
│                          │ drf-spectacular               │ full_perms 参数   │
│                          │ extend_schema_*               │ user 对象注入     │
│                  ┌───────┴────────┐                      │                   │
│                  │  OpenAPI Schema │◀─────────────────────┘                   │
│                  │   (schema.py)   │                                          │
│                  └────────────────┘                                          │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │ HTTP (JSON)
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            前端 (Angular)                                    │
│                                                                             │
│  ┌────────────────────────────┐    ┌────────────────────────────────────┐   │
│  │  TypeScript 数据接口层      │    │  REST Service 层                    │   │
│  │  (src/app/data/*.ts)       │◀───│  (services/rest/*.service.ts)      │   │
│  └─────────────┬──────────────┘    └────────────────────────────────────┘   │
│                │                                                             │
│                ▼                                                             │
│  ┌────────────────────────────┐    ┌────────────────────────────────────┐   │
│  │  PermissionsService        │    │  Form / Dialog 组件层               │   │
│  │  user_can_change 判定      │◀───│  (edit-dialog / document-detail)   │   │
│  │  permissions 字段解析      │    │  permissions-form 组件              │   │
│  └────────────────────────────┘    └────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 字段来源分析

Serializer 字段来自四个维度：**模型字段自动映射**、**Serializer 显式声明字段**、**Mixin 注入的权限字段**、**运行时动态裁剪字段**。

### 2.1 Serializer 继承体系

```
serializers.ModelSerializer
        │
        ├── DynamicFieldsModelSerializer         # 支持 fields=? 查询参数裁剪
        │       └── DocumentSerializer
        │               └── SearchResultSerializer
        │
        ├── MatchingModelSerializer              # 注入 document_count, slug
        │       ├── CorrespondentSerializer
        │       ├── DocumentTypeSerializer
        │       └── TagSerializer
        │
        ├── OwnedObjectSerializer                # 权限字段核心
        │       (继承 SerializerWithPerms + SetPermissionsMixin)
        │       ├── CorrespondentSerializer
        │       ├── DocumentTypeSerializer
        │       ├── TagSerializer
        │       ├── DocumentSerializer
        │       └── SavedViewSerializer
        │
        └── SerializerWithPerms                  # user/full_perms/all_fields 参数基类
```

相关代码：
- [serialisers.py#L103-L121](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L103-L121) - `DynamicFieldsModelSerializer`
- [serialisers.py#L217-L222](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L217-L222) - `SerializerWithPerms`
- [serialisers.py#L262-L470](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L262-L470) - `OwnedObjectSerializer`

### 2.2 字段来源分解（以 DocumentSerializer 为例）

#### 来源一：Model 字段自动映射（通过 Meta.fields）

在 `DocumentSerializer.Meta.fields` 中声明的字段若与 Django Model 字段名一致，DRF 会自动根据 Model 字段类型生成对应的 Serializer Field。

```python
# serialisers.py#L1226-L1258
class DocumentSerializer(...):
    class Meta:
        model = Document
        fields = (
            "id",                    # Model: AutoField
            "correspondent",         # Model: ForeignKey → PrimaryKeyRelatedField
            "document_type",         # Model: ForeignKey
            "storage_path",          # Model: ForeignKey
            "title",                 # Model: CharField
            "content",               # Model: TextField
            "tags",                  # Model: ManyToManyField → PrimaryKeyRelatedField(many=True)
            "created",               # Model: DateField
            "modified",              # Model: DateTimeField
            "added",                 # Model: DateTimeField
            "deleted_at",            # Model: DateTimeField (SoftDeleteModel)
            "archive_serial_number", # Model: PositiveIntegerField
            "mime_type",             # Model: CharField
            ...
        )
```

对应的 Model 定义见 [models.py#L157-L306](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/models.py#L157-L306)。

#### 来源二：Serializer 显式声明的自定义字段

在 Serializer 类体内直接声明的字段会覆盖或补充 Model 自动映射：

| 字段名 | 类型 | 说明 | 代码位置 |
|--------|------|------|----------|
| `correspondent` | `CorrespondentField` | 自定义 ForeignKey，queryset 全量 | [serialisers.py#L685-L688](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L685-L688) |
| `tags` | `TagsField(many=True)` | 自定义 M2M | [serialisers.py#L690-L693](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L690-L693) |
| `original_file_name` | `SerializerMethodField` | `get_original_file_name()` 返回 `obj.original_filename` | [serialisers.py#L999](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L999)、[#L1081-L1083](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L1081-L1083) |
| `archived_file_name` | `SerializerMethodField` | 仅在有归档版本时返回公开文件名 | [serialisers.py#L1000](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L1000)、[#L1084-L1088](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L1084-L1088) |
| `page_count` | `SerializerMethodField` | 返回 `obj.page_count` | [serialisers.py#L1002](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L1002) |
| `duplicate_documents` | `SerializerMethodField` | 仅在 `retrieve` action 返回基于 checksum 的重复文档列表 | [serialisers.py#L1003](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L1003)、[#L1033-L1041](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L1033-L1041) |
| `notes` | `NotesSerializer(many=True)` | 嵌套序列化 | [serialisers.py#L1005](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L1005) |
| `custom_fields` | `CustomFieldInstanceSerializer(many=True)` | 嵌套，支持 create/update | [serialisers.py#L1011-L1015](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L1011-L1015) |
| `owner` | `PrimaryKeyRelatedField` | 显式声明 allow_null | [serialisers.py#L1017-L1021](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L1017-L1021) |
| `remove_inbox_tags` | `BooleanField` | **write_only=True**，仅写入消费 | [serialisers.py#L1023-L1028](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L1023-L1028) |
| `versions` | `SerializerMethodField` | 返回文档版本链信息 | [serialisers.py#L1009](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L1009)、[#L1043-L1079](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L1043-L1079) |

#### 来源三：Mixin 注入的权限字段（OwnedObjectSerializer）

`OwnedObjectSerializer` 为所有受权限控制的对象注入以下字段：

| 字段名 | 类型 | 读写性 | 说明 |
|--------|------|--------|------|
| `permissions` | `SerializerMethodField` | read_only | 完整权限矩阵：`{view: {users:[...], groups:[...]}, change: {...}}` |
| `user_can_change` | `SerializerMethodField` | read_only | 当前用户是否可修改该对象（布尔值） |
| `is_shared_by_requester` | `SerializerMethodField` | read_only | 当前用户是否为 owner 且对象已共享给他人 |
| `set_permissions` | `SetPermissionsSerializer` (DictField) | write_only | **写入专用**，用于设置权限矩阵 |

关键实现代码：
- [serialisers.py#L405-L414](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L405-L414) - 字段声明
- [serialisers.py#L342-L352](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L342-L352) - `get_permissions()`
- [serialisers.py#L354-L363](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L354-L363) - `get_user_can_change()`
- [serialisers.py#L397-L403](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L397-L403) - `get_is_shared_by_requester()`

#### 来源四：MatchingModelSerializer 注入的匹配字段

`MatchingModelSerializer` 为 Tag/Correspondent/DocumentType 注入：

| 字段名 | 说明 |
|--------|------|
| `document_count` | 关联文档数量（annotate 注入，权限感知） |
| `slug` | 基于 name 的 slugify 结果 |

代码见 [serialisers.py#L124-L167](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L124-L167)。

---

## 3. 权限裁剪机制

权限裁剪是本项目最核心的契约机制，通过**查询参数** + **Serializer 构造参数** + **ListSerializer 预取** 三层协作完成。

### 3.1 控制参数：`full_perms` 与 `all_fields`

#### 参数传递链路

```
HTTP Request Query: ?full_perms=true
         │
         ▼
ViewSet.get_serializer()  [PassUserMixin]
  → 解析 full_perms 参数
  → kwargs["full_perms"] = True/False
  → kwargs["user"] = request.user
         │
         ▼
OwnedObjectSerializer.__init__()
  → 根据 full_perms / all_fields 裁剪字段
```

关键实现：

**PassUserMixin（View 层）**：
```python
# views.py#L360-L382
class PassUserMixin(GenericAPIView):
    def get_serializer(self, *args, **kwargs):
        serializer_class = self.get_serializer_class()
        if issubclass(serializer_class, SerializerWithPerms):
            kwargs.setdefault("user", self.request.user)
            full_perms = get_boolean(
                str(self.request.query_params.get("full_perms", "false"))
            )
            kwargs.setdefault("full_perms", full_perms)
        return super().get_serializer(*args, **kwargs)
```

**OwnedObjectSerializer（Serializer 层）**：
```python
# serialisers.py#L267-L278
def __init__(self, *args, **kwargs) -> None:
    super().__init__(*args, **kwargs)
    if not self.all_fields:
        try:
            if self.full_perms:
                # 列表场景：去掉详细权限字段，保留轻量判定
                self.fields.pop("user_can_change")
                self.fields.pop("is_shared_by_requester")
            else:
                # 列表场景默认：去掉完整权限矩阵
                self.fields.pop("permissions")
        except KeyError:
            pass
```

**DocumentSerializer 自动触发 full_perms**：
```python
# serialisers.py#L1213-L1224
def __init__(self, *args, **kwargs) -> None:
    # PATCH / PUT 自动返回完整权限（编辑表单需要）
    context = kwargs.get("context")
    if context is not None and (
        context.get("request").method == "PATCH"
        or context.get("request").method == "PUT"
    ):
        kwargs["full_perms"] = True
    super().__init__(*args, **kwargs)
```

#### 字段裁剪结果矩阵

| 场景 | `full_perms` | `all_fields` | 返回字段 |
|------|:---:|:---:|---|
| 列表 GET /api/documents/ | false (默认) | false | ❌ `permissions`, ✅ `user_can_change`, ✅ `is_shared_by_requester` |
| 列表 GET /api/documents/?full_perms=true | true | false | ✅ `permissions`, ❌ `user_can_change`, ❌ `is_shared_by_requester` |
| 详情 GET /api/documents/{id}/ | 视前端而定 | 视前端而定 | DocumentService.get() 默认传 `full_perms=true` |
| PATCH/PUT /api/documents/{id}/ | **自动 true** | - | ✅ `permissions`, ❌ `user_can_change`, ❌ `is_shared_by_requester` |
| OpenAPI schema 生成 | - | true | **所有字段均显示** |

### 3.2 列表批量权限预取（BulkPermissionMixin）

为避免 N+1 查询，`BulkPermissionMixin` 在 `get_serializer_context()` 中一次性预取整页对象的所有权限，放入 context 供 Serializer 读取。

```python
# views.py#L385-L479
class BulkPermissionMixin:
    def get_serializer_context(self):
        context = super().get_serializer_context()
        full_perms = get_boolean(str(self.request.query_params.get("full_perms", "false")))
        if not full_perms:
            return context

        # 一次性查出整页所有对象的 user/group 权限
        user_perms = self._get_object_perms(
            objects=queryset,
            perm_codenames=[permission_name_view, permission_name_change],
            actor="users",
        )
        group_perms = self._get_object_perms(...)

        context["users_view_perms"] = {...}
        context["users_change_perms"] = {...}
        context["groups_view_perms"] = {...}
        context["groups_change_perms"] = {...}
        return context
```

Serializer 中的 `_get_perms()` 优先从 context 读取，回退到 django-guardian 单查：
```python
# serialisers.py#L280-L307
def _get_perms(self, obj, codename, target):
    key = f"{target}_{codename}_perms"
    cached = self.context.get(key, {}).get(obj.pk)
    if cached is not None:
        return list(cached)
    # fallback: 从 guardian 单查
    ...
```

### 3.3 对象级权限检查（Permission Classes）

ViewSet 层的 `permission_classes` 控制访问，Serializer 层则在 `update()` 中二次校验：

```python
# permissions.py#L28-L52
class PaperlessObjectPermissions(DjangoObjectPermissions):
    def has_object_permission(self, request, view, obj):
        if hasattr(obj, "owner") and obj.owner is not None:
            if request.user == obj.owner:
                return True  # owner 直接放行
            else:
                return super().has_object_permission(...)  # 检查 django-guardian 显式授权
        else:
            return True  # 无 owner 的对象视为公开
```

Serializer.update() 中二次校验 owner/set_permissions 变更权限：
```python
# serialisers.py#L452-L469
def update(self, instance, validated_data):
    is_superuser = user.is_superuser if user else False
    is_owner = instance.owner == user if user else False
    is_unowned = instance.owner is None

    if (("owner" in validated_data and ...) or "set_permissions" in validated_data) \
       and not (is_superuser or is_owner or is_unowned):
        raise PermissionDenied(_("Insufficient permissions."))
```

### 3.4 文档数量权限感知（document_count）

Tag/Correspondent 等的 `document_count` 并非简单 `Count("documents")`，而是通过 `get_document_count_filter_for_user()` 只统计当前用户可见的文档：

```python
# permissions.py#L207-L220
def get_document_count_filter_for_user(user):
    if getattr(user, "is_superuser", False):
        return Q(documents__deleted_at__isnull=True)
    permitted_ids = _permitted_document_ids(user)  # 构造子查询
    return Q(documents__id__in=permitted_ids)
```

---

## 4. 前端表单消费链路

前端通过**TypeScript 接口约定** → **REST Service 请求** → **Form 组件** → **PermissionsService 判定** 四层消费后端契约。

### 4.1 TypeScript 接口层（静态契约镜像）

与后端 Serializer 一一对应的 TypeScript interface 定义在 `src-ui/src/app/data/` 下：

```typescript
// object-with-permissions.ts
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

```typescript
// document.ts
export interface Document extends ObjectWithPermissions {
  correspondent?: number
  document_type?: number
  tags?: number[]
  title?: string
  content?: string
  created?: string  // ISO string
  notes?: DocumentNote[]
  custom_fields?: CustomFieldInstance[]
  remove_inbox_tags?: boolean  // write-only
  ...
}
```

相关文件：
- [object-with-permissions.ts](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src-ui/src/app/data/object-with-permissions.ts)
- [document.ts](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src-ui/src/app/data/document.ts)

### 4.2 REST Service 层（请求参数控制）

`AbstractPaperlessService` 提供通用 CRUD，子类按需要传递 `full_perms` 参数。

**DocumentService.get() 默认带 full_perms=true**：
```typescript
// document.service.ts#L196-L213
get(id: number, versionID: number = null, fields: string = null): Observable<Document> {
    const params = { full_perms: true, version?, fields? }
    return this.http.get<Document>(this.getResourceUrl(id), { params })
}
```

**AbstractNameFilterService 支持显式传 fullPerms**：
```typescript
// abstract-name-filter-service.ts#L17-L34
listFiltered(..., fullPerms?: boolean, extraParams?) {
    let params = extraParams ?? {}
    if (fullPerms) {
        params['full_perms'] = true
    }
    return this.list(..., params)
}
```

**PATCH 写入时附带 remove_inbox_tags**：
```typescript
// document.service.ts#L305-L313
patch(o: Document, versionID: number = null): Observable<Document> {
    o.remove_inbox_tags = !!this.settingsService.get(
        SETTINGS_KEYS.DOCUMENT_EDITING_REMOVE_INBOX_TAGS
    )
    return this.http.patch<Document>(this.getResourceUrl(o.id), o, ...)
}
```

### 4.3 PermissionsService：权限字段的消费判定

`PermissionsService` 封装了对 `permissions` / `user_can_change` / `owner` 字段的消费逻辑：

```typescript
// permissions.service.ts#L75-L97
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
            object.user_can_change ||  // ← 后端计算好的轻量字段
            object.permissions?.change.users.includes(this.currentUser.id) ||
            ...
        )
    }
}
```

这里体现了 `full_perms` 裁剪的设计意图：
- 列表视图只用 `user_can_change`（布尔值，列表默认返回）判断可否显示编辑按钮
- 详情/编辑视图用完整 `permissions` 矩阵渲染权限编辑表单

### 4.4 编辑表单与权限表单的数据流

#### EditDialogComponent（Tag/Correspondent 等通用编辑对话框）

```
GET /api/tags/5/?full_perms=true
         │
         ▼
object: Tag = { id, name, owner, permissions:{view:{users,groups}, change:{...}}, ... }
         │
         ▼
EditDialogComponent.ngOnInit()
  → object['permissions_form'] = {
       owner: object.owner,
       set_permissions: object.permissions   // ← 后端读的 permissions → 前端写的 set_permissions
     }
  → objectForm.patchValue(object)
         │
         ▼
pngx-permissions-form 组件（FormGroup）
  ├── owner: FormControl
  └── set_permissions: FormGroup
        ├── view: FormGroup (users, groups)
        └── change: FormGroup (users, groups)
         │
         ▼
save()
  → formValues = objectForm.value
  → formValues.owner = permissionsObject.owner
  → formValues.set_permissions = permissionsObject.set_permissions
  → delete formValues.permissions_form
  → service.update(formValues)  → PATCH /api/tags/5/
```

关键代码：
- [edit-dialog.component.ts#L72-L116](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src-ui/src/app/components/common/edit-dialog/edit-dialog.component.ts#L72-L116) - 初始化 permissions_form
- [edit-dialog.component.ts#L159-L196](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src-ui/src/app/components/common/edit-dialog/edit-dialog.component.ts#L159-L196) - 提交时字段转换
- [permissions-form.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src-ui/src/app/components/common/input/permissions/permissions-form/permissions-form.component.ts)

**字段名映射的关键约定**：

| 方向 | 后端字段 | 前端字段 | 说明 |
|------|---------|---------|------|
| 响应 (GET) | `permissions` | `permissions` → `permissions_form.set_permissions` | 只读权限矩阵 |
| 响应 (GET) | `owner` | `owner` → `permissions_form.owner` | 所有者 |
| 请求 (PATCH/POST) | `set_permissions` | `permissions_form.set_permissions` → `set_permissions` | 写入权限矩阵 |
| 请求 (PATCH/POST) | `owner` | `permissions_form.owner` → `owner` | 写入所有者 |

即：**后端读接口用 `permissions`，写接口用 `set_permissions`**，前端在 permissions-form 组件内做桥接。

#### DocumentDetailComponent（文档详情编辑）

文档详情页的 FormGroup 声明了完整对应字段：

```typescript
// document-detail.component.ts#L259-L268
documentForm: FormGroup = new FormGroup({
    title: new FormControl(''),
    content: new FormControl(''),
    created: new FormControl(),
    correspondent: new FormControl(),
    document_type: new FormControl(),
    storage_path: new FormControl(),
    archive_serial_number: new FormControl(),
    tags: new FormControl([]),
    permissions_form: new FormControl(null),  // ← 同 EditDialog
})
```

同样的 permissions_form 转换逻辑也在这里执行（[document-detail.component.ts#L428-L450](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src-ui/src/app/components/document-detail/document-detail.component.ts#L428-L450)）。

### 4.5 权限可见性控制

前端还通过 `shouldSubmitPermissions()` 判断是否允许提交权限改动：

```typescript
// edit-dialog.component.ts#L152-L157
protected shouldSubmitPermissions(): boolean {
    return (
        this.dialogMode === EditDialogMode.CREATE ||
        this.permissionsService.currentUserOwnsObject(this.object)
    )
}
```

即：**新建或 owner 身份才能改权限**，与后端 Serializer.update() 中的校验逻辑呼应。

### 4.6 OpenAPI Schema 生成（drf-spectacular）

后端通过 `drf-spectacular` 生成 OpenAPI Schema，前端**并未直接消费此 Schema 做代码生成**，而是手动维护 TypeScript interface。

但 Schema 生成会用到 `all_fields=True` 来显示完整契约：

```python
# schema.py#L29-L43
def generate_object_with_permissions_schema(serializer_class):
    return {
        operation: extend_schema(
            parameters=[
                OpenApiParameter(name="full_perms", type=OpenApiTypes.BOOL, location=OpenApiParameter.QUERY),
            ],
            responses={
                200: serializer_class(many=operation == "list", all_fields=True),
            },
        )
        for operation in ["list", "retrieve"]
    }
```

应用于 ViewSet：
```python
# views.py#L523
@extend_schema_view(**generate_object_with_permissions_schema(CorrespondentSerializer))
class CorrespondentViewSet(...):
    ...
```

`@extend_schema_field` / `@extend_schema_serializer` 装饰器也在 Serializer 中大量使用，为 OpenAPI 提供精确的类型信息，如：
- [serialisers.py#L225-L257](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L225-L257) - `permissions` 字段的 schema
- [serialisers.py#L309-L341](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L309-L341) - `OwnedObjectSerializer.get_permissions` 的 schema
- [serialisers.py#L986-L988](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py#L986-L988) - `created_date` 废弃字段标注

---

## 5. 完整数据流示例：编辑一个 Tag

### Step 1：前端请求获取编辑数据

```
GET /api/tags/5/?full_perms=true
```

### Step 2：后端处理

1. `PassUserMixin.get_serializer()` 解析 `full_perms=true`，注入 `user=request.user`, `full_perms=True`
2. `BulkPermissionMixin.get_serializer_context()` 预取该对象的权限（不过单对象场景不走批量）
3. `TagSerializer.__init__()` → `OwnedObjectSerializer.__init__()`：
   - `full_perms=True` → 移除 `user_can_change`、`is_shared_by_requester`
   - 保留 `permissions` 字段
4. 序列化时调用 `get_permissions()` 返回完整权限矩阵

### Step 3：响应 JSON

```json
{
  "id": 5,
  "name": "Important",
  "color": "#ff0000",
  "owner": 1,
  "permissions": {
    "view": { "users": [2, 3], "groups": [1] },
    "change": { "users": [2], "groups": [] }
  },
  "document_count": 42,
  "slug": "important"
}
```

### Step 4：前端表单渲染

- `EditDialogComponent` 将 `permissions` → `permissions_form.set_permissions`
- `pngx-permissions-form` 渲染用户/组选择器

### Step 5：用户提交修改

```
PATCH /api/tags/5/
Content-Type: application/json

{
  "name": "Very Important",
  "owner": 1,
  "set_permissions": {
    "view": { "users": [2, 3, 4], "groups": [1] },
    "change": { "users": [2], "groups": [1] }
  }
}
```

### Step 6：后端处理写入

1. `TagSerializer.validate_set_permissions()` 校验 user_id/group_id 存在
2. `OwnedObjectSerializer.update()` 校验当前用户是 owner/superuser
3. `_set_permissions()` 调用 `set_permissions_for_object()` 通过 django-guardian 更新权限表
4. `super().update()` 更新 name 等普通字段
5. PATCH 方法自动触发 `full_perms=True`，返回带完整 `permissions` 的响应

---

## 6. 关键代码索引

### 后端

| 文件 | 核心职责 |
|------|---------|
| [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/serialisers.py) | 所有 Serializer 定义，字段声明、权限裁剪、权限读写 |
| [permissions.py](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/permissions.py) | ViewSet permission classes、权限矩阵读写函数、文档 ID 权限子查询 |
| [views.py](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/views.py) | PassUserMixin（传 user/full_perms）、BulkPermissionMixin（批量预取权限） |
| [schema.py](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/schema.py) | drf-spectacular 扩展，full_perms 参数的 OpenAPI 声明 |
| [models.py](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src/documents/models.py) | Django Model 定义，Serializer 字段的来源根基 |

### 前端

| 文件 | 核心职责 |
|------|---------|
| [data/object-with-permissions.ts](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src-ui/src/app/data/object-with-permissions.ts) | 权限对象 TypeScript 契约 |
| [data/document.ts](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src-ui/src/app/data/document.ts) | Document 接口定义 |
| [services/rest/abstract-paperless-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src-ui/src/app/services/rest/abstract-paperless-service.ts) | 通用 REST CRUD 基类 |
| [services/rest/abstract-name-filter-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src-ui/src/app/services/rest/abstract-name-filter-service.ts) | 支持 fullPerms 参数的 Service 基类 |
| [services/rest/document.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src-ui/src/app/services/rest/document.service.ts) | DocumentService，默认带 full_perms=true、remove_inbox_tags |
| [services/permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src-ui/src/app/services/permissions.service.ts) | 权限字段消费：user_can_change、permissions 矩阵解析 |
| [components/common/edit-dialog/edit-dialog.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src-ui/src/app/components/common/edit-dialog/edit-dialog.component.ts) | 通用编辑对话框，permissions → permissions_form 转换 |
| [components/common/input/permissions/permissions-form/permissions-form.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src-ui/src/app/components/common/input/permissions/permissions-form/permissions-form.component.ts) | 权限编辑表单组件 |
| [components/document-detail/document-detail.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/67-paperless-ngx/src-ui/src/app/components/document-detail/document-detail.component.ts) | 文档详情编辑，同 permissions_form 模式 |
