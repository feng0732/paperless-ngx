# Paperless-ngx 存储路径与文件命名规则分析

## 一、整体目录结构

所有文件存储在 `MEDIA_ROOT/documents/` 下，分为三个子目录：

| 目录 | 用途 | 设置项 |
|------|------|--------|
| `originals/` | 原始文档文件 | `ORIGINALS_DIR` |
| `archive/` | 归档 PDF 副本 | `ARCHIVE_DIR` |
| `thumbnails/` | 文档缩略图（WebP 格式） | `THUMBNAIL_DIR` |

定义位置：[settings/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/paperless/settings/__init__.py#L69-L73)

```
MEDIA_ROOT/
└── documents/
    ├── originals/      # 原始上传文件
    ├── archive/        # OCR 后的 PDF 归档副本
    └── thumbnails/     # 缩略图文件 (PK.webp)
```

---

## 二、Document 模型中的路径属性

Document 模型提供了三个关键路径属性，均为只读 property：

### 2.1 source_path（原始文件路径）

定义：[models.py#L430-L434](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/models.py#L430-L434)

```python
@property
def source_path(self) -> Path:
    fname = str(self.filename) if self.filename else f"{self.pk:07}{self.file_type}"
    return (settings.ORIGINALS_DIR / Path(fname)).resolve()
```

- 以 `filename` 字段为准，若为空则使用 `0000001.pdf` 格式（7位零填充ID + 文件扩展名）
- 拼接在 `ORIGINALS_DIR` 下

### 2.2 archive_path（归档副本路径）

定义：[models.py#L444-L449](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/models.py#L444-L449)

```python
@property
def archive_path(self) -> Path | None:
    if self.has_archive_version:
        return (settings.ARCHIVE_DIR / Path(str(self.archive_filename))).resolve()
    else:
        return None
```

- 仅当 `archive_filename` 不为空时才存在
- 拼接在 `ARCHIVE_DIR` 下

### 2.3 thumbnail_path（缩略图路径）

定义：[models.py#L478-L484](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/models.py#L478-L484)

```python
@property
def thumbnail_path(self) -> Path:
    webp_file_name = f"{self.pk:07}.webp"
    webp_file_path = settings.THUMBNAIL_DIR / Path(webp_file_name)
    return webp_file_path.resolve()
```

- **不使用模板**，直接使用文档主键（7位零填充）+ `.webp` 格式
- 例如：`0000001.webp`
- 始终存储在 `THUMBNAIL_DIR` 根目录下，无子目录结构

---

## 三、路径模板（Path Template）系统

### 3.1 StoragePath 模型

定义：[models.py#L147-L154](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/models.py#L147-L154)

```python
class StoragePath(MatchingModel):
    path = models.TextField(_("path"))
```

- `StoragePath` 继承自 `MatchingModel`，支持自动匹配（与 Correspondent、Tag、DocumentType 类似）
- 核心字段 `path` 存储 Jinja2 模板字符串
- Document 通过外键 `storage_path` 关联一个 StoragePath

### 3.2 模板格式来源优先级

在 [file_handling.py#L141-L153](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/file_handling.py#L141-L153) 的 `generate_filename()` 中定义：

```
优先级从高到低：
1. Document.storage_path.path          （关联的存储路径模板）
2. settings.FILENAME_FORMAT            （全局设置，会自动转换为新格式）
3. 无模板 → 使用 {pk:07}{file_type}    （7位零填充ID + 扩展名）
```

旧格式（Python `{var}`）通过 `convert_format_str_to_template_format()` 自动转换为 Jinja2 的 `{{ var }}` 格式：
[utils.py#L4-L24](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/templating/utils.py#L4-L24)

### 3.3 模板渲染上下文

模板在 [filepath.py#L345-L412](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/templating/filepath.py#L345-L412) 的 `validate_filepath_template_and_render()` 中渲染，可用变量：

| 类别 | 变量 | 说明 |
|------|------|------|
| **基础元数据** | `title` | 文档标题（已清理非法字符） |
| | `correspondent` | 往来单位名称，无则为 `-none-` |
| | `document_type` | 文档类型名称，无则为 `-none-` |
| | `asn` | 归档序列号 |
| | `owner_username` | 所有者用户名 |
| | `original_name` | 原始文件名（不含扩展名） |
| | `doc_pk` | 文档ID（7位零填充） |
| **创建日期** | `created`, `created_year`, `created_year_short` | |
| | `created_month`, `created_month_name`, `created_month_name_short` | |
| | `created_day` | |
| **入库日期** | `added`, `added_year`, `added_year_short` | |
| | `added_month`, `added_month_name`, `added_month_name_short` | |
| | `added_day` | |
| **标签** | `tag_list` | 逗号分隔的已排序标签名 |
| | `tag_name_list` | 标签名列表，可循环 |
| **自定义字段** | `custom_fields` | 字典，键为字段名，值含 `type` 和 `value` |
| **完整文档对象** | `document` | 安全文档上下文（id, title, content 等） |

特殊值占位符：当关联对象为空时，使用 `PlaceholderString("-none-")`，其特性：
- 渲染为字符串 `-none-`
- 布尔值为 `False`，可在 `{% if correspondent %}` 判断
- 与 `"none"` 或 `"-none-"` 比较均相等

### 3.4 可用 Jinja2 过滤器

定义位置：[filepath.py#L99-L107](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/templating/filepath.py#L99-L107)

| 过滤器 | 用途 |
|--------|------|
| `datetime` | 格式化日期时间 |
| `slugify` | URL/文件名友好的 slug 转换 |
| `localize_date` | 本地化日期 |
| `get_cf_value` | 获取自定义字段值 |

### 3.5 路径安全校验

`FilePathTemplate` 类继承自 Jinja2 `Template`，渲染后自动清理：
- 移除换行符 `\n`、`\r`
- 去除 `/` 两侧的多余空格
- 去除首尾路径分隔符（始终为相对路径）

`_is_safe_relative_path()` 额外校验：
- 不能是绝对路径
- 不能包含盘符（Windows）
- 不能包含 `..` 路径遍历

### 3.6 模板渲染示例

模板：`{{ correspondent }}/{{ created_year }}/{{ title }}`

若：
- correspondent = "Acme Corp"
- created_year = "2024"
- title = "Invoice January"

渲染结果：`Acme Corp/2024/Invoice January`

最终生成完整文件名（原始文件）：
`Acme Corp/2024/Invoice January.pdf`

---

## 四、文件名生成函数

### 4.1 generate_filename() — 基础生成

定义：[file_handling.py#L125-L185](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/file_handling.py#L125-L185)

核心逻辑：

```
1. 确定 context_doc：
   - 版本文档使用 root_document 进行模板渲染
   - 但会追加 _v{version_index} 后缀

2. 获取模板字符串（按优先级）

3. 渲染模板 → base_path

4. 组装最终文件名：
   {base_filename}{version_suffix}{counter_str}{filetype_str}

   其中：
   - base_filename: 模板渲染结果的 name 部分
   - version_suffix: _v1, _v2...（仅版本文档）
   - counter_str: _01, _02...（冲突时使用）
   - filetype_str: .pdf（归档）或 doc.file_type（原始）

5. 如无模板：
   {doc_pk:07}{version_suffix}{counter_str}{filetype_str}
```

**版本文档注意**：版本文档使用根文档的元数据渲染路径，仅文件名追加版本号后缀。

### 4.2 generate_unique_filename() — 带冲突检测

定义：[file_handling.py#L44-L99](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/file_handling.py#L44-L99)

逻辑：
1. 若为归档文件名，优先尝试基于原始文件名的简单 PDF 版本（保留目录结构）
2. 否则从 `counter=0` 开始调用 `generate_filename()`
3. 检查目标路径是否已存在
4. 存在则 `counter++` 循环，直到找到可用文件名
5. 若与旧文件名相同，直接返回（不重复移动）

**归档文件命名优化**（L67-L82）：
- 优先使用 `原始文件名.stem + .pdf`，保留模板生成的目录结构
- 例如原始文件为 `Corp/2024/receipt_001.jpg`，则归档尝试 `Corp/2024/receipt_001.pdf`

### 4.3 format_filename() — 模板渲染 + 清理

定义：[file_handling.py#L102-L122](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/file_handling.py#L102-L122)

- 调用 `validate_filepath_template_and_render()` 渲染模板
- 若 `FILENAME_FORMAT_REMOVE_NONE` 为 True，清理路径中的 `/-none-/`、` -none-` 等
- 向后兼容：将 `-none-` 替换为 `none`

---

## 五、文件写入流程（Consumer）

在 [consumer.py#L668-L729](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/consumer.py#L668-L729) 的 `try_consume()` 中：

```
1. parser.parse() 完成后获得：
   - thumbnail: 临时缩略图路径
   - archive_path: 临时归档 PDF 路径（可选）

2. 获取 FileLock(MEDIA_LOCK) 串行化写入

3. 对原始文件：
   generate_unique_filename(doc) → document.filename
   create_source_path_directory()  创建父目录
   _write(working_copy, source_path)

4. 对缩略图：
   _write(thumbnail, thumbnail_path)   （路径固定为 PK.webp）

5. 对归档文件（如存在）：
   generate_unique_filename(doc, archive_filename=True) → archive_filename
   create_source_path_directory(archive_path)
   _write(archive_path, document.archive_path)
   compute_checksum(archive_path) → archive_checksum

6. document.save() → 触发 update_filename_and_move_files 信号
   （此时 filename 已非空，信号会进一步调整最终位置）
```

**文件名长度超限保护**：若生成的路径超过 `Document.MAX_STORED_FILENAME_LENGTH`（1024），回退到无模板的默认命名（`{pk:07}.ext`）。

---

## 六、文件自动重命名与移动（信号）

核心信号处理器：[signals/handlers.py#L434-L668](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/signals/handlers.py#L434-L668)

### 6.1 触发时机

监听三个信号：
- `Document.post_save` — 文档保存时
- `Document.tags.through.m2m_changed` — 标签变更时
- `CustomFieldInstance.post_save` — 自定义字段实例变更时（仅当模板引用了 `custom_fields`）

### 6.2 处理流程

```
1. 跳过条件：
   - instance.filename 为空（Consumer 初次创建尚未写入文件）
   - CustomFieldInstance 变更但模板未使用 custom_fields

2. 获取 FileLock(MEDIA_LOCK)
3. refresh_from_db() — 防止锁等待期间数据已更新

4. 生成候选文件名：
   candidate_filename = generate_filename(doc)
   candidate_source_path = ORIGINALS_DIR / candidate_filename

5. 判断逻辑：
   - 候选 == 旧文件名 → 无需移动
   - 候选路径已存在且内容匹配（checksum）→ 视为已移动
   - 候选路径已存在且冲突 → 使用 generate_unique_filename() 加后缀
   - 否则使用候选文件名

6. 同样逻辑处理 archive_filename

7. 实际移动文件：
   validate_move() 校验安全性（不能移出 root，旧文件必须存在，新文件不能存在）
   create_source_path_directory() 创建目标目录
   shutil.move(old_path, new_path)

8. Document.objects.filter(pk=...).update(...) 直接更新数据库
   （不调用 save() 防止无限递归）

9. 清理空目录：
   delete_empty_directories(old_source_path.parent, root=ORIGINALS_DIR)
   delete_empty_directories(old_archive_path.parent, root=ARCHIVE_DIR)

10. 若为根文档，同步处理所有版本文档
```

### 6.3 异常回滚

发生 OSError / DatabaseError / CannotMoveFilesException 时：
- 尝试将文件移回原位置
- 恢复 instance 的旧 filename / archive_filename
- 不抛出异常，记录日志（文件一致性由 sanity_checker 保证）

---

## 七、归档副本（Archive Copy）管理

### 7.1 何时生成归档

归档是经过 OCR 处理的 PDF/A 格式文件，用于长期保存和全文检索。

由 `should_produce_archive()` 判断是否需要生成（基于 MIME 类型和 parser 能力）。

### 7.2 归档文件生命周期

**创建**（Consumer 阶段）：
- Parser 的 `parse()` 方法生成临时 PDF 文件
- `compute_checksum()` 计算 SHA256 → `archive_checksum`
- 写入 `ARCHIVE_DIR` 下的最终位置

**更新**（任务重处理）：
- `update_document_content()` 任务（[tasks.py#L280-L376](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/tasks.py#L280-L376)）
- 重新 OCR → 生成新 archive_path → 替换旧文件
- 使用 `generate_unique_filename(doc, archive_filename=True)` 避免冲突

**删除**：
- 跟随 Document 软删除/硬删除
- 在 `delete_files_on_document_delete()` 信号处理器中删除实际文件

---

## 八、缩略图管理

### 8.1 缩略图命名规则

**不使用模板**，始终为 `{pk:07}.webp`，直接放在 `THUMBNAIL_DIR` 根目录。

### 8.2 缩略图生成

核心函数：[parsers.py#L174-L198](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/parsers.py#L174-L198) 的 `make_thumbnail_from_pdf()`

```
主流程：
1. 使用 ImageMagick `convert` 将 PDF 首页转为 500px 宽的 WebP
   参数：density=300, scale="500x5000>", alpha=remove, strip, auto_orient, use_cropbox

降级流程（convert 失败时）：
1. Ghostscript `gs` 将 PDF 首页渲染为 PNG
2. ImageMagick `convert` 将 PNG 转为 WebP

兜底方案：
1. 复制默认缩略图 resources/document.webp
```

每个 Parser 子类实现自己的 `get_thumbnail(document_path, mime_type)`，大多数最终调用 `make_thumbnail_from_pdf()`。

### 8.3 缩略图生成时机

- **Consumer 消费时**：parser.parse() 后立即生成，和文档一起写入
- **管理命令手动重建**：`document_thumbnails` 命令
  （[management/commands/document_thumbnails.py](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/management/commands/document_thumbnails.py)）
  - 支持 `--document ID` 重建单个文档
  - 支持多进程并行处理
  - 流程：查找 parser → `get_thumbnail()` → `shutil.move()` 覆盖 `thumbnail_path`
- **重处理任务时**：`update_document_content()` 任务也会重新生成缩略图

### 8.4 默认缩略图

`documents/resources/document.webp` 作为兜底默认缩略图，当所有生成方式失败时使用。

---

## 九、空目录清理

工具函数：[file_handling.py#L15-L41](file:///d:/fz/0601/solo-dogfeeding/code/58-paperless-ngx/src/documents/file_handling.py#L15-L41) 的 `delete_empty_directories()`

```
逻辑：
1. 从 directory 开始，逐级向上
2. 若目录为空 → rmdir() 删除
3. 遇到非空目录或到达 root 时停止
4. 若 directory 不在 root 下则直接跳过（安全校验）
```

调用时机：
- 删除文件后：清理原文件的父目录
- 移动文件后：清理旧位置的父目录

---

## 十、并发与锁

所有文件系统操作通过 `FileLock(settings.MEDIA_LOCK)` 串行化，锁文件位置：

```
MEDIA_ROOT/media.lock
```

涉及锁的场景：
- Consumer 写入文件
- Signal 处理器移动/重命名文件
- 归档更新任务

---

## 十一、完整调用链汇总

```
文档入库流程：
consumer.try_consume()
  ├─ parser.parse() → 生成 text, archive_path(临时), thumbnail(临时)
  ├─ document.save() → 写入 DB 元数据
  ├─ FileLock 获得
  │   ├─ generate_unique_filename(doc) → doc.filename
  │   ├─ create_source_path_directory()
  │   ├─ _write(original, doc.source_path)
  │   ├─ _write(thumbnail, doc.thumbnail_path)
  │   ├─ 若有归档：
  │   │   ├─ generate_unique_filename(doc, archive=True) → doc.archive_filename
  │   │   ├─ _write(archive, doc.archive_path)
  │   │   └─ compute_checksum(archive) → doc.archive_checksum
  │   └─ FileLock 释放
  └─ document.save()
      └─ signal: update_filename_and_move_files()
          ├─ FileLock 获得
          ├─ generate_filename() 重新计算（此时元数据更完整）
          ├─ shutil.move() 调整最终位置
          ├─ delete_empty_directories() 清理旧目录
          └─ 同步处理所有版本文档

元数据变更（标签/自定义字段/保存）：
Document.post_save / tags.m2m_changed / CustomFieldInstance.post_save
  └─ update_filename_and_move_files()
      └─ 同上逻辑，重新计算路径并移动文件

缩略图重建：
manage.py document_thumbnails
  └─ _process_document(doc_id)
      ├─ parser.get_thumbnail(source_path, mime_type)
      └─ shutil.move(temp_thumb, doc.thumbnail_path)
```
