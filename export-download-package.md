# Export/Download 打包流程代码脉络分析

## 一、整体架构概览

Paperless-ngx 的导出/下载功能分为 **四条主要链路**：

| 链路 | 入口 | 场景 | 核心文件 |
|------|------|------|----------|
| **单文档下载** | `GET /api/documents/{id}/download/` | 用户下载单个文档 | [views.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/views.py) |
| **批量打包下载** | `POST /api/documents/bulk_download/` | 用户多选文档打包成 ZIP | [bulk_download.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/bulk_download.py), [views.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/views.py#L3714-L3771) |
| **分享链接打包** | 先 `POST /api/share_link_bundles/` 创建 → Celery 任务 `build_share_link_bundle` 异步打包 | 生成对外分享 ZIP | [tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/tasks.py#L657-L745), [views.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/views.py#L4284-L4372) |
| **CLI 全量导出** | `python manage.py document_exporter` 管理命令 | 运维人员做数据备份迁移 | [document_exporter.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/management/commands/document_exporter.py) |

路由注册在 [paperless/urls.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/paperless/urls.py#L186-L189)。

---

## 二、文件选择逻辑

### 2.1 批量下载的文件选择

批量下载通过 `DocumentSelectionMixin` 和 `DocumentSelectionSerializer` 协同完成文件选择。这是所有批量操作（下载、编辑、删除、重处理等）共用的基础能力。

#### 2.1.1 请求参数序列化 — DocumentSelectionSerializer

位置：[serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/serialisers.py#L1559-L1591)

```python
class DocumentSelectionSerializer(DocumentListSerializer):
    documents = serializers.ListField(child=serializers.IntegerField())  # 显式文档ID列表
    all = serializers.BooleanField(default=False)                        # 是否选择全部匹配
    filters = serializers.DictField()                                    # 过滤条件
```

**验证规则**：
- 当 `all=true` 时，`documents` 字段可以为空，由后端通过 `filters` 重建文档列表
- 当 `all=false`（默认）时，`documents` 字段必填且必须是有效的整数ID列表

#### 2.1.2 ID 解析 — DocumentSelectionMixin._resolve_document_ids

位置：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/views.py#L2607-L2642)

```python
def _resolve_document_ids(
    self,
    *,
    user: User,
    validated_data: dict[str, Any],
    permission_codename: str = "view_document",   # 注意：默认值是 view_document
) -> list[int]:
    if not validated_data.get("all", False):
        # ⚠️ all=false 时直接透传用户传入的 ID，完全不做权限过滤
        return validated_data["documents"]

    # 只有 all=true 时才走下面这条完整的过滤链路
    filters = validated_data.get("filters") or {}
    orm_filters = {k: v for k, v in filters.items() if k not in _TANTIVY_SEARCH_PARAM_NAMES}

    permitted_documents = get_objects_for_user_owner_aware(  # 1) 权限预过滤
        user, permission_codename, Document
    )
    filtered_documents = DocumentFilterSet(                  # 2) ORM 条件过滤
        data=orm_filters, queryset=permitted_documents
    ).qs.distinct()
    search_filtered_ids = self._get_search_document_ids(     # 3) Tantivy 全文搜索过滤
        user=user, filters=filters
    )
    if search_filtered_ids is not None:
        filtered_documents = filtered_documents.filter(pk__in=search_filtered_ids)
    return list(filtered_documents.values_list("pk", flat=True))
```

**关键分支**：
- **`all=false`（前端显式选择）**：直接 `return validated_data["documents"]` — **不做任何权限过滤**，权限完全依赖后续各操作的逐文档校验
- **`all=true`（全选匹配）**：走完整的三阶段过滤链路
  1. 权限预过滤 → `get_objects_for_user_owner_aware`（默认 `view_document`）
  2. ORM 条件过滤 → `DocumentFilterSet`（标签、类型、日期、通信者等）
  3. 全文搜索过滤 → Tantivy 搜索后端（`text`/`title_search`/`query`/`more_like_id`）

**注意**：`BulkDownloadView` 调用 `_resolve_document_ids` 时未传 `permission_codename`，因此 `all=true` 场景使用的是默认的 `view_document` 权限做预过滤，而不是后续逐文档校验的 `change_document`。

### 2.2 前端选择方式

位置：[bulk-editor.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src-ui/src/app/components/document-list/bulk-editor/bulk-editor.component.ts#L869-L889)

```typescript
downloadSelected() {
  let downloadFileType = ...          // 'both' | 'archive' | 'originals'
  this.documentService.bulkDownload(
    this.getSelectionQuery(),         // { documents, all, filters }
    downloadFileType,
    this.downloadForm.get('downloadUseFormatting').value
  ).subscribe(result => saveAs(result, 'documents.zip'))
}
```

位置：[document.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src-ui/src/app/services/rest/document.service.ts#L417-L431)

```typescript
bulkDownload(selection, content = 'both', useFilenameFormatting = false) {
  return this.http.post(this.getResourceUrl(null, 'bulk_download'), {
    ...selection,
    content,
    follow_formatting: useFilenameFormatting
  }, { responseType: 'blob' })
}
```

---

## 三、响应构造逻辑

### 3.1 单文档下载响应 — serve_file

位置：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/views.py#L4473-L4522)

```python
def serve_file(*, doc, use_archive, disposition, follow_formatting=False):
    # 选择文件版本
    if use_archive:
        file_handle = doc.archive_file
        filename = doc.archive_filename if follow_formatting else doc.get_public_filename(archive=True)
        mime_type = "application/pdf"
    else:
        file_handle = doc.source_file
        filename = doc.filename if follow_formatting else doc.get_public_filename()
        mime_type = doc.mime_type

    response = FileResponse(file_handle, content_type=mime_type)

    # RFC 5987 编码处理非 ASCII 文件名（兼容 Firefox/Chromium）
    filename_normalized = normalize("NFKD", filename.replace(",", "_")).encode("ascii", "ignore").decode("ascii")
    filename_encoded = quote(filename)
    response["Content-Disposition"] = (
        f"{disposition}; "
        f'filename="{filename_normalized}"; '
        f"filename*=utf-8''{filename_encoded}"
    )
    return response
```

**文件名解析链路** `doc.get_public_filename()`：[models.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/models.py#L455-L472)

```python
def get_public_filename(self, *, archive=False, counter=0, suffix=None):
    result = str(self)                    # Document.__str__ → "{created} {title}"
    if counter: result += f"_{counter:02}" # 冲突追加序号
    result += ".pdf" if archive else self.file_type
    return pathvalidate.sanitize_filename(result, replacement_text="-")
```

**文件路径解析**：
- `doc.source_path` → `settings.ORIGINALS_DIR / filename` [models.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/models.py#L431-L434)
- `doc.archive_path` → `settings.ARCHIVE_DIR / archive_filename` [models.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/models.py#L445-L449)

### 3.2 批量下载响应 — BulkDownloadView.post

位置：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/views.py#L3719-L3771)

```python
class BulkDownloadView(DocumentSelectionMixin, GenericAPIView[Any]):
    permission_classes = (IsAuthenticated,)   # 只有登录要求，无对象级 DRF permission_class
    serializer_class = BulkDownloadSerializer

    def post(self, request, format=None):
        # 1) 参数校验
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        # 2) 解析文档ID（此处 _resolve_document_ids 默认用 view_document 做 all=true 预过滤）
        ids = self._resolve_document_ids(user=request.user, validated_data=serializer.validated_data)
        documents = Document.objects.filter(pk__in=ids)

        # 3) 独立的逐文档权限校验（用 change_document，比单文档下载更严格）
        #    ⚠️ BulkDownloadView 不使用 DocumentOperationPermissionMixin，自己手写循环
        for document in documents:
            if not has_perms_owner_aware(request.user, "change_document", document):
                return HttpResponseForbidden("Insufficient permissions")

        # 4) 选择策略类
        if content == "both":        strategy_class = OriginalAndArchiveStrategy
        elif content == "originals": strategy_class = OriginalsOnlyStrategy
        else:                        strategy_class = ArchiveOnlyStrategy

        # 5) 临时文件构建 ZIP
        settings.SCRATCH_DIR.mkdir(parents=True, exist_ok=True)
        fd, temp_name = tempfile.mkstemp(dir=settings.SCRATCH_DIR, suffix="-compressed-archive")
        os.close(fd)
        temp_path = Path(temp_name)

        try:
            with zipfile.ZipFile(temp_path, "w", compression) as zipf:
                strategy = strategy_class(zipf, follow_formatting=follow_filename_format)
                for document in documents:
                    strategy.add_document(document)
            f = temp_path.open("rb")
            temp_path.unlink()
        except Exception:
            temp_path.unlink(missing_ok=True)
            raise

        # 6) 返回 FileResponse（磁盘文件已 unlink，由 OS 在句柄关闭时回收）
        return FileResponse(f, as_attachment=True, filename="documents.zip", content_type="application/zip")
```

**关键细节**：
- 临时文件创建于 `settings.SCRATCH_DIR`，返回响应前立即 `unlink`
- 压缩方式通过 `BulkDownloadSerializer.validate_compression` 映射：`none→ZIP_STORED`, `deflated→ZIP_DEFLATED`, `bzip2→ZIP_BZIP2`, `lzma→ZIP_LZMA` [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/serialisers.py#L2312-L2320)

---

## 四、批量导出逻辑

### 4.1 Strategy 模式 — BulkArchiveStrategy 家族

位置：[bulk_download.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/bulk_download.py)

**类层次结构**：
```
BulkArchiveStrategy (抽象基类)
├── OriginalsOnlyStrategy          — 仅导出原始文件
├── ArchiveOnlyStrategy            — 仅导出归档PDF（无归档则回退原始）
└── OriginalAndArchiveStrategy     — 两者都导出，分 originals/ 和 archive/ 子目录
```

Strategy 模式同时被 **BulkDownloadView** 和 **build_share_link_bundle Celery 任务** 复用。

#### 4.1.1 命名策略 — 两种文件名构造方式

```python
class BulkArchiveStrategy:
    def __init__(self, zipf, *, follow_formatting=False):
        self.zipf = zipf
        if follow_formatting:
            self.make_unique_filename = self._formatted_filepath  # 使用文档存储路径格式
        else:
            self.make_unique_filename = self._filename_only       # 使用公共显示名
```

**方式一：`_filename_only`（默认）** — [bulk_download.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/bulk_download.py#L24-L46)

```python
def _filename_only(self, doc, *, archive=False, folder=""):
    counter = 0
    while True:
        filename = folder + doc.get_public_filename(archive=archive, counter=counter)
        if filename in self.zipf.namelist(): counter += 1  # 冲突检测
        else: return filename
```

**方式二：`_formatted_filepath`（遵循 FILENAME_FORMAT）** — [bulk_download.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/bulk_download.py#L48-L70)

```python
def _formatted_filepath(self, doc, *, archive=False, folder=""):
    if archive and doc.has_archive_version:
        return Path(folder) / doc.archive_filename   # 直接使用存储路径（文档入库时已保证唯一）
    else:
        return Path(folder) / doc.filename
```

#### 4.1.2 三种打包策略

**OriginalsOnlyStrategy** — [bulk_download.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/bulk_download.py#L76-L78)
```python
def add_document(self, doc):
    self.zipf.write(doc.source_path, self.make_unique_filename(doc))
```

**ArchiveOnlyStrategy** — [bulk_download.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/bulk_download.py#L81-L91)
```python
def add_document(self, doc):
    if doc.has_archive_version:
        self.zipf.write(doc.archive_path, self.make_unique_filename(doc, archive=True))
    else:
        self.zipf.write(doc.source_path, self.make_unique_filename(doc))  # 无归档则回退
```

**OriginalAndArchiveStrategy** — [bulk_download.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/bulk_download.py#L94-L107)
```python
def add_document(self, doc):
    if doc.has_archive_version:
        self.zipf.write(doc.archive_path, self.make_unique_filename(doc, archive=True, folder="archive/"))
    self.zipf.write(doc.source_path, self.make_unique_filename(doc, folder="originals/"))
```

### 4.2 分享链接打包 — 两段式流程

分享链接打包分**同步创建校验**和**异步打包执行**两个阶段。

#### 阶段一：创建时同步校验 — ShareLinkBundleViewSet.create

位置：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/views.py#L4308-L4372)

```python
class ShareLinkBundleViewSet(PassUserMixin, ModelViewSet[ShareLinkBundle]):
    permission_classes = (IsAuthenticated, PaperlessObjectPermissions)  # Bundle 对象本身的 CRUD 权限
    filter_backends = (..., ObjectOwnedOrGrantedPermissionsFilter)

    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        document_ids = serializer.validated_data["document_ids"]
        documents_qs = Document.objects.filter(pk__in=document_ids).select_related("owner")
        # ...

        documents = list(documents_qs)
        for document in documents:
            # 逐文档校验 view_document 权限（和单文档下载一致）
            if not has_perms_owner_aware(request.user, "view_document", document):
                raise ValidationError({"document_ids": _("Insufficient permissions to share document %(id)s.")})

        # ... 保存 bundle，owner=request.user
        build_share_link_bundle.apply_async(kwargs={"bundle_id": bundle.pk})  # 提交异步任务
```

#### 阶段二：Celery 异步任务执行 — build_share_link_bundle

位置：[tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/tasks.py#L657-L745)

```python
def build_share_link_bundle(bundle_id):
    try:
        bundle = ShareLinkBundle.objects.filter(pk=bundle_id).prefetch_related("documents").get()
    except ShareLinkBundle.DoesNotExist:
        return

    bundle.status = PROCESSING; bundle.save()

    documents = list(bundle.documents.all().order_by("pk"))
    _, temp_zip_path = mkstemp(suffix=".zip", dir=settings.SCRATCH_DIR)

    # ⚠️ 此处不再做任何文档权限校验，完全信任创建阶段的同步校验结果
    strategy_class = ArchiveOnlyStrategy if bundle.file_version == ARCHIVE else OriginalsOnlyStrategy
    with zipfile.ZipFile(temp_zip_path, "w", zipfile.ZIP_DEFLATED) as zipf:
        strategy = strategy_class(zipf)
        for document in documents:
            strategy.add_document(document)

    shutil.move(temp_zip_path, settings.SHARE_LINK_BUNDLE_DIR / f"{bundle.slug}.zip")
    bundle.status = READY; bundle.save()
```

#### 阶段三：公开下载 — SharedLinkView

位置：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/views.py#L4408-L4470)

```python
class SharedLinkView(View):
    authentication_classes = []   # ⚠️ 无任何认证要求
    permission_classes = []       # ⚠️ 无任何权限要求

    def get(self, request, slug):
        share_link = ShareLink.objects.filter(slug=slug).first()
        if share_link is not None:
            if share_link.expiration is not None and share_link.expiration < timezone.now():
                return HttpResponseRedirect("/accounts/login/?sharelink_expired=1")
            # 直接返回文件，只校验 slug 存在 + expiration
            return serve_file(doc=share_link.document, use_archive=..., disposition="inline")

        bundle = ShareLinkBundle.objects.filter(slug=slug).first()
        if bundle is None: return redirect(...)
        if bundle.expiration ...: return redirect(...)
        if bundle.status in {PENDING, PROCESSING}: return HttpResponse(202)

        # 直接返回 ZIP 文件，不做任何用户认证或文档级权限校验
        response = FileResponse(bundle.absolute_file_path.open("rb"), content_type="application/zip")
        # ...
        return response
```

### 4.3 CLI 全量导出 — document_exporter

位置：[document_exporter.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/management/commands/document_exporter.py)

⚠️ **这是 Django management command，完全运行在命令行环境，不经过 Django REST Framework 的用户认证/权限体系，也没有任何 per-document 权限过滤。** 能否运行该命令取决于操作系统层面谁能访问项目代码和执行 `python manage.py`。

```python
class Command(CryptMixin, PaperlessCommand):
    def handle(self, *args, **options):
        # ... 解析参数、处理 --zip 临时目录 ...
        with FileLock(settings.MEDIA_LOCK):   # 全局媒体锁，防止导出期间有并发写入
            self.dump()

    def dump(self):
        # 1. 快照目标目录现有文件
        # 2. 准备 manifest.json — 序列化全量模型数据
        manifest_key_to_object_query = {
            "documents": Document.global_objects.order_by("id").all(),  # ⚠️ 全量，无权限过滤
            "correspondents": Correspondent.objects.all(),
            "tags": Tag.objects.all(),
            "users": User.objects.exclude(username__in=_excluded_usernames),
            "user_object_permissions": UserObjectPermission.objects.exclude(...),
            # ... 共约 30 种模型
        }

        document_manifest = []
        with StreamingManifestWriter(manifest_path, ...) as writer:
            with transaction.atomic():
                for key, qs in manifest_key_to_object_query.items():
                    if key == "documents":
                        for batch in serialize_queryset_batched(qs, batch_size=self.batch_size):
                            for record in batch:
                                self._encrypt_record_inline(record)  # 可选加密敏感字段
                            document_manifest.extend(batch)
                    # ...

        # 3. 逐文档导出文件
        document_map = {d.pk: d for d in Document.global_objects.order_by("id")}
        for index, document_dict in enumerate(document_manifest):
            document = document_map[document_dict["pk"]]
            base_name = self.generate_base_name(document)
            original_target, thumbnail_target, archive_target = (
                self.generate_document_targets(document, base_name, document_dict)
            )
            if not self.data_only:
                self.copy_document_files(document, original_target, thumbnail_target, archive_target)
                # 含 --compare-checksums 增量导出、解密处理、--use-filename-format 路径格式化
```

**导出流程**：
1. **全局媒体锁** — `FileLock(settings.MEDIA_LOCK)` 防止导出期间文件变化
2. **快照现有文件** — 记录目标目录中已有文件（用于 `--delete` 清理多余文件）
3. **写入 manifest.json** — 分批序列化所有模型数据（通信者、标签、文档、权限、用户等约 30 种模型），可选 BLAKE2b 比对去重写入、可选加密敏感字段
4. **逐文档导出文件**：
   - `generate_base_name()` 生成基础名（遵循 `FILENAME_FORMAT` 设置）
   - `generate_document_targets()` 确定 originals/thumbnails/archive 目标路径
   - `copy_document_files()` 复制文件（含校验和比对增量导出、PGP 解密处理）
5. **可选 ZIP 打包** — `shutil.make_archive()` 将临时目录整体打包

**核心文件复制** [document_exporter.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/management/commands/document_exporter.py#L617-L645)：
```python
def copy_document_files(self, document, original_target, thumbnail_target, archive_target):
    self.check_and_copy(document.source_path, document.checksum, original_target)
    if thumbnail_target: self.check_and_copy(document.thumbnail_path, None, thumbnail_target)
    if archive_target:   self.check_and_copy(document.archive_path, document.archive_checksum, archive_target)
```

---

## 五、权限过滤逻辑

### 5.1 权限判定核心函数

位置：[permissions.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/permissions.py#L258-L288)

```python
def get_objects_for_user_owner_aware(user, perms, Model, *, include_deleted=False):
    """返回用户拥有、无归属、或被显式授权的对象集合（用于 QuerySet 级过滤）"""
    objects_owned = manager.filter(owner=user)              # ① 自己是所有者
    objects_unowned = manager.filter(owner__isnull=True)     # ② 对象无主（公开）
    objects_with_perms = get_objects_for_user(               # ③ django-guardian 显式授权
        user=user, perms=perms, klass=manager.all(), accept_global_perms=False
    )
    return objects_owned | objects_unowned | objects_with_perms


def has_perms_owner_aware(user, perms, obj):
    """判断用户对单个对象是否有权限（逐条校验）"""
    checker = ObjectPermissionChecker(user)
    return (obj.owner is None            # ① 无主对象人人可访问
            or obj.owner == user          # ② 自己是所有者
            or checker.has_perm(perms, obj))  # ③ django-guardian 显式授权
```

**权限三元组**：无主公开 ∨ 所有者 ∨ django-guardian 显式授权。三条满足任意一条即可。

### 5.2 各链路权限校验对比

| 链路 | 校验位置 | 校验权限 | 备注 |
|------|---------|---------|------|
| **单文档下载** | `_resolve_request_and_root_doc` → `has_perms_owner_aware` [views.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/views.py#L1273-L1277) | `view_document` | 基础只读权限 |
| **批量下载 — ID 选择阶段（all=true）** | `_resolve_document_ids` → `get_objects_for_user_owner_aware` [views.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/views.py#L2625-L2629) | `view_document`（默认值） | 只在全选匹配时生效，做预过滤 |
| **批量下载 — ID 选择阶段（all=false）** | 无 | — | 直接透传前端传入的 ID，完全不做权限过滤 |
| **批量下载 — 逐文档最终校验** | `BulkDownloadView.post` 手写 for 循环 [views.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/views.py#L3732-L3734) | `change_document` | ⚠️ 比单下载更严格；独立实现，**不使用** `DocumentOperationPermissionMixin` |
| **分享链接打包 — 创建阶段** | `ShareLinkBundleViewSet.create` 逐文档校验 [views.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/views.py#L4328-L4337) | `view_document` | 和单文档下载一致 |
| **分享链接打包 — Celery 执行阶段** | 无 | — | 完全信任创建阶段的校验结果，不复验 |
| **分享链接打包 — Bundle 对象本身** | `PaperlessObjectPermissions` + `ObjectOwnedOrGrantedPermissionsFilter` | 对象级 CRUD | 控制谁能查看/修改/删除 bundle 记录 |
| **分享链接公开下载（SharedLinkView）** | 无 | — | `authentication_classes=[]`, `permission_classes=[]`；只校验 slug 存在 + expiration 未过期 |
| **CLI 全量导出** | 无 | — | Django management command，不经过用户权限体系，由 OS 控制谁可运行 |

**与其他批量操作的权限实现对比**：

- `BulkEditView` / `DeleteDocumentsView` / `ReprocessDocumentsView` 等继承 `DocumentOperationPermissionMixin`，该 Mixin 的 `_has_document_permissions` [views.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/views.py#L2659-L2731) 除了逐文档校验 `change_document` 之外，对破坏性操作（delete/rotate/edit_pdf/set_permissions 等）还额外要求用户必须是**全部文档的所有者**，并对创建/删除文档额外校验全局 `add_document` / `delete_document` 权限。
- `BulkDownloadView` **不继承** `DocumentOperationPermissionMixin`，而是自己手写了一个简单的 for 循环只校验 `change_document`，没有所有者要求。

### 5.3 批量下载权限测试验证

位置：[test_api_bulk_download.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/tests/test_api_bulk_download.py#L327-L341)

```python
def test_download_insufficient_permissions(self):
    user = User.objects.create_user(username="temp_user")   # 普通用户
    self.client.force_authenticate(user=user)
    self.doc2.owner = self.user   # superuser（不是 temp_user），doc2 对 temp_user 不满足三元组
    self.doc2.save()

    response = self.client.post(
        ENDPOINT,
        json.dumps({"documents": [self.doc2.id, self.doc3.id]}),
        content_type="application/json",
    )

    self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
    self.assertEqual(response.content, b"Insufficient permissions")
```

只要列表中**任意一个文档**权限不足，整个请求返回 403。注意这个测试用的是 `all=false` 模式（显式传 ID），此时 `_resolve_document_ids` 不做预过滤，完全由逐文档的 `change_document` 校验拦下来。

### 5.4 序列化器参数

[BulkDownloadSerializer](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/serialisers.py#L2297-L2320) 定义了请求参数：

```python
class BulkDownloadSerializer(DocumentSelectionSerializer):
    content = serializers.ChoiceField(choices=["archive", "originals", "both"], default="archive")
    compression = serializers.ChoiceField(choices=["none", "deflated", "bzip2", "lzma"], default="none")
    follow_formatting = serializers.BooleanField(default=False)

    def validate_compression(self, compression):
        return {
            "none": zipfile.ZIP_STORED,
            "deflated": zipfile.ZIP_DEFLATED,
            "bzip2": zipfile.ZIP_BZIP2,
            "lzma": zipfile.ZIP_LZMA,
        }[compression]
```

---

## 六、完整调用时序

### 6.1 批量下载（BulkDownloadView）

```
前端 (bulk-editor.component.ts)
  │  downloadSelected()
  │    ├─ 判断 content: 'archive' | 'originals' | 'both'
  │    └─ documentService.bulkDownload({documents, all, filters}, content, follow_formatting)
  ▼
后端 (views.py: BulkDownloadView.post)
  │  permission_classes: (IsAuthenticated,)           — 仅要求登录
  │
  │  1. serializer = BulkDownloadSerializer(data=request.data)
  │
  │  2. ids = DocumentSelectionMixin._resolve_document_ids()
  │  │     ├─ all=false → 直接 return validated_data["documents"]
  │  │     │              ⚠️ 完全不做权限过滤
  │  │     └─ all=true  → get_objects_for_user_owner_aware("view_document")
  │  │                     + DocumentFilterSet ORM 过滤
  │  │                     + Tantivy 全文搜索过滤
  │
  │  3. for doc in documents:                              — 独立逐文档校验
  │  │     if not has_perms_owner_aware(user, "change_document", doc) → 403
  │
  │  4. 根据 content 选择 Strategy:
  │  │     OriginalsOnly / ArchiveOnly / OriginalAndArchive
  │
  │  5. tempfile.mkstemp(SCRATCH_DIR) → 创建临时 ZIP
  │  6. strategy.add_document(doc) 逐个写入
  │  │     └─ make_unique_filename() → _filename_only / _formatted_filepath
  │  7. f = temp_path.open("rb"); temp_path.unlink()
  │  8. FileResponse(f, filename="documents.zip", as_attachment=True)
  ▼
浏览器: saveAs(blob, "documents.zip")
```

### 6.2 分享链接打包（两段式）

```
前端 → POST /api/share_link_bundles/
  ▼
ShareLinkBundleViewSet.create
  │  permission_classes: (IsAuthenticated, PaperlessObjectPermissions)
  │  1. serializer.is_valid()
  │  2. for doc in documents:
  │  │     if not has_perms_owner_aware(user, "view_document", doc) → ValidationError
  │  3. bundle.save(owner=request.user, status=PENDING)
  │  4. build_share_link_bundle.apply_async(bundle_id=bundle.pk)
  ▼
Celery Worker: build_share_link_bundle(bundle_id)
  │  ⚠️ 不做任何权限校验，信任创建阶段结果
  │  1. bundle.status = PROCESSING
  │  2. documents = bundle.documents.all()
  │  3. ZIP 构建（复用 BulkArchiveStrategy）
  │  4. shutil.move 到 SHARE_LINK_BUNDLE_DIR
  │  5. bundle.status = READY
  ▼
外部匿名用户 → GET /share/{slug}/
  ▼
SharedLinkView.get
  │  authentication_classes = []
  │  permission_classes = []
  │  1. 按 slug 查找 ShareLink / ShareLinkBundle
  │  2. 校验 expiration 未过期
  │  3. 直接 FileResponse 返回文件 / ZIP
```
