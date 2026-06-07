# Export/Download 打包流程代码脉络分析

## 一、整体架构概览

Paperless-ngx 的导出/下载功能分为 **三条主要链路**：

| 链路 | 入口 | 场景 | 核心文件 |
|------|------|------|----------|
| **单文档下载** | `GET /api/documents/{id}/download/` | 用户下载单个文档 | [views.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/views.py) |
| **批量打包下载** | `POST /api/documents/bulk_download/` | 用户多选文档打包成 ZIP | [bulk_download.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/bulk_download.py), [views.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/views.py#L3714-L3771) |
| **CLI 全量导出** | `document_exporter` 管理命令 | 管理员数据备份迁移 | [document_exporter.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/management/commands/document_exporter.py) |
| **分享链接打包** | Celery 任务 `build_share_link_bundle` | 生成分享链接 ZIP | [tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/tasks.py#L657-L745) |

路由注册在 [paperless/urls.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/paperless/urls.py#L186-L189)。

---

## 二、文件选择逻辑

### 2.1 批量下载的文件选择

批量下载通过 `DocumentSelectionMixin` 和 `DocumentSelectionSerializer` 协同完成文件选择。

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
def _resolve_document_ids(self, *, user, validated_data, permission_codename="view_document"):
    if not validated_data.get("all", False):
        return validated_data["documents"]                    # 直接透传用户选择的ID

    filters = validated_data.get("filters") or {}
    permitted_documents = get_objects_for_user_owner_aware(  # 1) 先做权限过滤
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

**选择流程**：
1. **权限预过滤** → `get_objects_for_user_owner_aware`（见第五节）
2. **ORM 条件过滤** → `DocumentFilterSet`（标签、类型、日期、通信者等）
3. **全文搜索过滤** → Tantivy 搜索后端（`text`/`title_search`/`query`/`more_like_id`）

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
def post(self, request, format=None):
    # 1) 参数校验
    serializer = self.get_serializer(data=request.data)
    serializer.is_valid(raise_exception=True)

    # 2) 解析文档ID
    ids = self._resolve_document_ids(user=request.user, validated_data=serializer.validated_data)
    documents = Document.objects.filter(pk__in=ids)

    # 3) 权限逐文档校验（change_document 权限）
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

    with zipfile.ZipFile(temp_path, "w", compression) as zipf:
        strategy = strategy_class(zipf, follow_formatting=follow_filename_format)
        for document in documents:
            strategy.add_document(document)

    # 6) 返回 FileResponse（文件句柄 open，磁盘文件 unlink）
    f = temp_path.open("rb")
    temp_path.unlink()
    return FileResponse(f, as_attachment=True, filename="documents.zip", content_type="application/zip")
```

**关键细节**：
- 临时文件创建于 `settings.SCRATCH_DIR`，返回响应前立即 `unlink`（由操作系统在句柄关闭时回收）
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
        return Path(folder) / doc.archive_filename   # 直接使用存储路径（已保证唯一）
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
        self.zipf.write(doc.source_path, self.make_unique_filename(doc))  # 回退
```

**OriginalAndArchiveStrategy** — [bulk_download.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/bulk_download.py#L94-L107)
```python
def add_document(self, doc):
    if doc.has_archive_version:
        self.zipf.write(doc.archive_path, self.make_unique_filename(doc, archive=True, folder="archive/"))
    self.zipf.write(doc.source_path, self.make_unique_filename(doc, folder="originals/"))
```

### 4.2 分享链接打包 — build_share_link_bundle

位置：[tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/tasks.py#L657-L745)

这是 **Celery 异步任务**，复用了 `bulk_download.py` 的 Strategy：

```python
def build_share_link_bundle(bundle_id):
    bundle = ShareLinkBundle.objects.get(pk=bundle_id)
    bundle.status = PROCESSING; bundle.save()

    documents = list(bundle.documents.all().order_by("pk"))
    _, temp_zip_path = mkstemp(suffix=".zip", dir=settings.SCRATCH_DIR)

    strategy_class = ArchiveOnlyStrategy if bundle.file_version == ARCHIVE else OriginalsOnlyStrategy
    with zipfile.ZipFile(temp_zip_path, "w", zipfile.ZIP_DEFLATED) as zipf:
        strategy = strategy_class(zipf)
        for document in documents:
            strategy.add_document(document)

    shutil.move(temp_zip_path, settings.SHARE_LINK_BUNDLE_DIR / f"{bundle.slug}.zip")
    bundle.status = READY; bundle.save()
```

### 4.3 CLI 全量导出 — document_exporter

位置：[document_exporter.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/management/commands/document_exporter.py)

**导出流程**：
1. **锁定媒体目录** — `FileLock(settings.MEDIA_LOCK)` 防止导出期间文件变化
2. **快照现有文件** — 记录目标目录中已有文件（用于 `--delete` 清理多余文件）
3. **写入 manifest.json** — 序列化所有模型数据（通信者、标签、文档、权限等）
4. **逐文档导出**：
   - `generate_base_name()` 生成基础名（遵循 `FILENAME_FORMAT` 设置）
   - `generate_document_targets()` 确定 originals/thumbnails/archive 目标路径
   - `copy_document_files()` 复制文件（含校验和比对增量导出、解密处理）
5. **可选 ZIP 打包** — `shutil.make_archive()` 将临时目录打包

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
    """返回用户拥有、无归属、或被显式授权的对象集合"""
    objects_owned = manager.filter(owner=user)          # 自己的
    objects_unowned = manager.filter(owner__isnull=True) # 无主的
    objects_with_perms = get_objects_for_user(           # django-guardian 显式授权
        user=user, perms=perms, klass=manager.all(), accept_global_perms=False
    )
    return objects_owned | objects_unowned | objects_with_perms


def has_perms_owner_aware(user, perms, obj):
    """判断用户对单对象是否有权限"""
    checker = ObjectPermissionChecker(user)
    return (obj.owner is None            # 无主对象人人可访问
            or obj.owner == user          # 所有者
            or checker.has_perm(perms, obj))  # django-guardian 显式授权
```

**权限三元组**：无主公开 + 所有者 + 显式授权（基于 django-guardian）

### 5.2 各链路中的权限校验点

| 链路 | 权限校验位置 | 所需权限 |
|------|-------------|----------|
| **单文档下载** | `_resolve_request_and_root_doc` 内部（调用 `has_perms_owner_aware`） | `view_document` |
| **批量下载** | [views.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/views.py#L3732-L3734) 逐文档循环检查 | `change_document`（⚠️ 比单下载更严格） |
| **分享链接打包** | 创建 ShareLinkBundle 时校验，任务执行时不重复校验 | 创建时的权限 |
| **文件选择阶段** | [views.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/views.py#L2625-L2629) `_resolve_document_ids` 中 `get_objects_for_user_owner_aware` | `view_document` |
| **CLI 导出** | 管理命令本身需要 `is_staff`，导出全量数据 | 管理员权限 |

**测试用例验证** [test_api_bulk_download.py](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/tests/test_api_bulk_download.py#L327-L341)：
```python
def test_download_insufficient_permissions(self):
    user = User.objects.create_user(username="temp_user")  # 普通用户
    self.doc2.owner = self.user  # 文档归属 admin
    response = self.client.post(ENDPOINT, {"documents": [self.doc2.id, self.doc3.id]})
    self.assertEqual(response.status_code, 403)
    self.assertEqual(response.content, b"Insufficient permissions")
```

只要列表中**任意一个文档**权限不足，整个请求返回 403。

### 5.3 序列化器参数

[BulkDownloadSerializer](file:///d:/fz/0601/solo-dogfeeding/code/65-paperless-ngx/src/documents/serialisers.py#L2297-L2320) 定义了请求参数：

```python
class BulkDownloadSerializer(DocumentSelectionSerializer):
    content = serializers.ChoiceField(choices=["archive", "originals", "both"], default="archive")
    compression = serializers.ChoiceField(choices=["none", "deflated", "bzip2", "lzma"], default="none")
    follow_formatting = serializers.BooleanField(default=False)
```

---

## 六、完整调用时序

```
前端 (bulk-editor.component.ts)
  │  downloadSelected()
  │    ├─ 判断 content: 'archive' | 'originals' | 'both'
  │    └─ documentService.bulkDownload({documents, all, filters}, content, follow_formatting)
  ▼
后端 (views.py: BulkDownloadView.post)
  │  1. serializer = BulkDownloadSerializer(data=request.data)
  │  2. ids = DocumentSelectionMixin._resolve_document_ids()
  │  │     ├─ all=false → 透传 documents ID 列表
  │  │     └─ all=true  → get_objects_for_user_owner_aware()
  │  │                     + DocumentFilterSet ORM 过滤
  │  │                     + Tantivy 全文搜索过滤
  │  3. for doc in documents: has_perms_owner_aware("change_document")
  │  4. 根据 content 选择 Strategy:
  │  │     OriginalsOnly / ArchiveOnly / OriginalAndArchive
  │  5. tempfile.mkstemp() → 创建临时 ZIP 文件
  │  6. strategy.add_document(doc) 逐个写入
  │  │     └─ make_unique_filename() → _filename_only / _formatted_filepath
  │  7. FileResponse(f, filename="documents.zip", as_attachment=True)
  ▼
浏览器: saveAs(blob, "documents.zip")
```
