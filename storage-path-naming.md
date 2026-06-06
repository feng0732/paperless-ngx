# Paperless-ngx Storage Path 与文件命名规则代码分析

## 一、整体目录结构配置

所有文件存储的根目录配置在 [paperless/settings/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/paperless/settings/__init__.py#L65-L88) 中：

```
MEDIA_ROOT/                    # 媒体根目录 (PAPERLESS_MEDIA_ROOT)
└── documents/
    ├── originals/             # ORIGINALS_DIR - 原始文件目录
    ├── archive/               # ARCHIVE_DIR - 归档PDF/A目录
    └── thumbnails/            # THUMBNAIL_DIR - 缩略图目录
```

相关配置项：
- `ORIGINALS_DIR` = `MEDIA_ROOT / "documents" / "originals"` - 存放用户上传的原始文件
- `ARCHIVE_DIR` = `MEDIA_ROOT / "documents" / "archive"` - 存放转换后的 PDF/A 归档文件
- `THUMBNAIL_DIR` = `MEDIA_ROOT / "documents" / "thumbnails"` - 存放 WebP 格式缩略图
- `MEDIA_LOCK` = `MEDIA_ROOT / "media.lock"` - 文件锁，用于多线程/进程同步
- `FILENAME_FORMAT` - 全局文件名格式模板（环境变量 `PAPERLESS_FILENAME_FORMAT`）
- `FILENAME_FORMAT_REMOVE_NONE` - 是否移除模板中的 `-none-` 占位符

---

## 二、StoragePath 模型

[documents/models.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/models.py#L147-L155) 中定义的 `StoragePath` 模型：

```python
class StoragePath(MatchingModel):
    path = models.TextField(_("path"))
```

- 继承自 `MatchingModel`，支持匹配规则（自动分配给文档）
- `path` 字段存储 Jinja2 模板字符串，用于为每个文档生成相对路径
- `Document` 模型通过外键 `storage_path` 关联到 StoragePath

---

## 三、路径模板（Path Template）实现机制

### 3.1 模板引擎配置

模板系统基于 Jinja2 的沙箱环境，定义在 [documents/templating/environment.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/templating/environment.py)：

```python
class JinjaEnvironment(SandboxedEnvironment):
    def is_safe_callable(self, obj):
        # 阻止访问 .save() / .delete() / .update() 方法
        ...

_template_environment = JinjaEnvironment(
    trim_blocks=True,
    lstrip_blocks=True,
    autoescape=False,
    extensions=["jinja2.ext.loopcontrols"],
)
```

### 3.2 自定义 FilePathTemplate 类

在 [documents/templating/filepath.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/templating/filepath.py#L34-L52) 中定义：

```python
class FilePathTemplate(Template):
    def render(self, *args, **kwargs) -> str:
        def clean_filepath(value: str) -> str:
            # 1. 移除换行符
            # 2. 移除斜杠前后的多余空格
            # 3. 去除首尾分隔符（确保是相对路径）
            ...
        return clean_filepath(original_render)
```

### 3.3 模板可用变量

模板上下文通过多个函数构建，位于 [documents/templating/filepath.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/templating/filepath.py)：

#### 基本元数据上下文 (`get_basic_metadata_context`)

| 变量 | 说明 |
|------|------|
| `title` | 文档标题（已清理非法字符） |
| `correspondent` | 联系人名称，无则为 `-none-` |
| `document_type` | 文档类型名称，无则为 `-none-` |
| `asn` | 归档序列号，无则为 `-none-` |
| `owner_username` | 所有者用户名，无则为 `-none-` |
| `original_name` | 原始文件名（不含扩展名） |
| `doc_pk` | 文档主键（7位补零，如 `0000001`） |

#### 创建日期上下文 (`get_creation_date_context`)

| 变量 | 示例 |
|------|------|
| `created` | ISO 格式完整日期 |
| `created_year` | `2024` |
| `created_year_short` | `24` |
| `created_month` | `06` |
| `created_month_name` | `June` |
| `created_month_name_short` | `Jun` |
| `created_day` | `15` |

#### 添加日期上下文 (`get_added_date_context`)

与创建日期类似，变量前缀为 `added_`。

#### 标签上下文 (`get_tags_context`)

| 变量 | 说明 |
|------|------|
| `tag_list` | 排序后用逗号连接的标签名（已清理） |
| `tag_name_list` | 标签名称列表（可循环） |

#### 自定义字段上下文 (`get_custom_fields_context`)

```
custom_fields.<字段名>.type   - 字段类型
custom_fields.<字段名>.value  - 字段值
```

#### 完整文档对象 (`get_safe_document_context`)

通过 `document` 变量可访问：`id`, `pk`, `title`, `content`, `page_count`, `created`, `added`, `modified`, `archive_serial_number`, `mime_type`, `checksum`, `archive_checksum`, `filename`, `archive_filename`, `original_filename`, `owner`, `tags`, `correspondent`, `document_type`, `storage_path`

### 3.4 可用过滤器

注册在 [documents/templating/filepath.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/templating/filepath.py#L101-L107)：

- `get_cf_value(custom_fields, name, default)` - 获取自定义字段值
- `datetime(format)` - 日期格式化（strftime）
- `slugify` - Django slug 化
- `localize_date(format, locale)` - Babel 本地化日期

### 3.5 模板验证与渲染

核心函数 `validate_filepath_template_and_render` 在 [filepath.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/templating/filepath.py#L345-L412)：

1. 若无真实文档，使用 `create_dummy_document()` 创建假文档进行验证
2. 构建完整上下文字典
3. 使用 `FilePathTemplate` 渲染模板
4. 安全检查：确保渲染结果是**相对路径**且不包含 `..` 遍历

### 3.6 旧格式兼容

[documents/templating/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/templating/utils.py) 中的 `convert_format_str_to_template_format()` 将旧的 Python `{var}` 格式转换为 Jinja2 `{{ var }}` 格式。

---

## 四、文件命名生成流程

### 4.1 核心函数调用链

```
generate_unique_filename()    # file_handling.py - 生成唯一不冲突文件名
  └── generate_filename()     # file_handling.py - 根据模板生成基础文件名
        └── format_filename() # file_handling.py - 应用模板渲染 + 后处理
              └── validate_filepath_template_and_render()  # filepath.py
```

### 4.2 `generate_filename()` 详细逻辑

位于 [documents/file_handling.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/file_handling.py#L125-L185)：

**步骤 1：确定格式字符串来源（优先级从高到低）**

1. `Document.storage_path.path` - 文档关联的 StoragePath 模板
2. `settings.FILENAME_FORMAT` - 全局配置模板（自动转换旧格式）
3. None - 使用文档 ID 默认命名

**步骤 2：渲染模板获取基础路径**

调用 `format_filename()` 渲染模板，返回相对路径（可包含子目录）。

**步骤 3：组装最终文件名**

```
最终文件名 = {基础文件名}{版本后缀}{冲突计数器}{扩展名}
```

- 版本后缀：`_v1`, `_v2` 等（仅版本文档）
- 冲突计数器：`_01`, `_02` 等（仅当发生冲突时）
- 扩展名：归档文件固定 `.pdf`，原始文件使用 `doc.file_type`

**步骤 4：无模板时的默认命名**

```
{doc_pk:07}{版本后缀}{冲突计数器}{扩展名}
例如: 0000001.pdf, 0000001_v1_01.pdf
```

### 4.3 `generate_unique_filename()` 冲突避免

位于 [documents/file_handling.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/file_handling.py#L44-L99)：

1. 先尝试 `generate_filename(counter=0)`
2. 若目标路径已存在且不等于旧路径，counter++ 重试（`_01`, `_02`, ...）
3. 对于归档文件，优先尝试与原始文件同名的 `.pdf` 版本

### 4.4 `format_filename()` 后处理

位于 [documents/file_handling.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/file_handling.py#L102-L122)：

- 若 `FILENAME_FORMAT_REMOVE_NONE=True`：
  - 移除 `/-none-/` 目录段
  - 移除 ` -none-` 后缀
  - 移除孤立的 `-none-`
- 向后兼容：将剩余 `-none-` 替换为 `none`

---

## 五、归档副本（Archive）管理

### 5.1 是否生成归档的判断逻辑

[documents/consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/consumer.py#L124-L189) 中的 `should_produce_archive()`：

| 条件 | 结果 |
|------|------|
| 解析器需要 PDF 渲染（如图片格式） | ✅ 生成 |
| 解析器无法生成归档（如纯文本） | ❌ 不生成 |
| `ARCHIVE_FILE_GENERATION=always` | ✅ 生成 |
| `ARCHIVE_FILE_GENERATION=never` | ❌ 不生成 |
| `auto` + 图片 MIME 类型 | ✅ 生成 |
| `auto` + 原生数字 PDF（有结构标签/文本足够） | ❌ 不生成 |
| `auto` + 扫描 PDF（无文本/文本极少） | ✅ 生成 |

### 5.2 归档文件存储路径

Document 模型属性 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/models.py#L440-L453)：

```python
@property
def has_archive_version(self) -> bool:
    return self.archive_filename is not None

@property
def archive_path(self) -> Path | None:
    if self.has_archive_version:
        return (settings.ARCHIVE_DIR / Path(str(self.archive_filename))).resolve()
```

- `archive_filename` 存储相对路径（可含子目录，由模板生成）
- `archive_path` 返回绝对路径
- 归档文件扩展名固定为 `.pdf`（PDF/A 格式）
- `archive_checksum` 存储归档文件的校验和

---

## 六、缩略图（Thumbnail）管理

### 6.1 缩略图路径规则

Document 模型属性 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/models.py#L478-L488)：

```python
@property
def thumbnail_path(self) -> Path:
    webp_file_name = f"{self.pk:07}.webp"
    webp_file_path = settings.THUMBNAIL_DIR / Path(webp_file_name)
    return webp_file_path.resolve()
```

**命名规则（固定，不使用模板）：**
```
THUMBNAIL_DIR / {doc_pk:07}.webp
例如: .../thumbnails/0000001.webp
```

特点：
- 固定使用 WebP 格式
- 命名仅依赖文档主键，7位数字补零
- **不使用路径模板**，无分级目录结构

### 6.2 缩略图生成

两种生成方式：

**方式 1：消费流程中实时生成**
[consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/consumer.py#L526-L536)
```python
thumbnail = document_parser.get_thumbnail(self.working_copy, mime_type)
# 之后写入 document.thumbnail_path
```

**方式 2：管理命令批量重新生成**
[documents/management/commands/document_thumbnails.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/management/commands/document_thumbnails.py)
- 命令：`document_thumbnails`
- 支持 `--document <id>` 指定单个文档
- 支持多进程并行处理
- 每个解析器类实现 `get_thumbnail()` 方法

---

## 七、文件自动重命名与移动机制

### 7.1 触发信号

位于 [documents/signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/signals/handlers.py#L431-L668) 的 `update_filename_and_move_files()` 监听以下信号：

- `post_save` (Document) - 文档保存后
- `m2m_changed` (Document.tags.through) - 标签变更时
- `post_save` (CustomFieldInstance) - 自定义字段值变更时（仅当模板使用了 custom_fields）

### 7.2 执行流程

```
1. 获取 FileLock(settings.MEDIA_LOCK) 锁
2. 从数据库刷新文档数据
3. 调用 generate_filename() 计算新路径
   - 若路径超长，跳过
   - 若目标已存在且校验和匹配，视为已移动
   - 若目标已存在且非同一文件，调用 generate_unique_filename()
4. 对原始文件和归档文件分别计算
5. validate_move() 安全检查：
   - 新路径必须在 ORIGINALS_DIR / ARCHIVE_DIR 内
   - 旧文件必须存在
   - 新文件不能已存在
6. create_source_path_directory() 创建父目录
7. shutil.move() 执行文件移动
8. Document.objects.filter(pk=...).update() 更新数据库（避免触发 save 递归）
9. delete_empty_directories() 清理空目录
10. 若为根文档，同步处理所有版本文档
```

### 7.3 异常回滚

若移动或保存过程中出现异常：
1. 尝试将文件移回原位置
2. 恢复 instance 的旧 filename / archive_filename 值
3. 记录警告日志

### 7.4 目录清理

`delete_empty_directories()` 在 [file_handling.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/file_handling.py#L15-L41)：
- 从文件所在目录开始向上遍历
- 遇到空目录则删除
- 到达根目录（ORIGINALS_DIR / ARCHIVE_DIR）停止
- 确保不越界删除根目录外的内容

---

## 八、消费流程中的完整文件处理

[documents/consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/consumer.py#L586-L784) 中的文件落盘流程：

```
事务开始
  │
  ├─ 1. _store() 创建 Document 数据库记录（filename 此时为空）
  │
  ├─ 2. document_consumption_finished 信号
  │     └─ 自动分配 correspondent / document_type / tags / storage_path
  │
  ├─ 3. 获取 MEDIA_LOCK
  │     │
  │     ├─ 3a. generate_unique_filename(document) → filename
  │     │    超长则回退到 use_format=False 的默认命名
  │     ├─ 3b. 创建父目录
  │     ├─ 3c. 写入原始文件 → document.source_path
  │     │
  │     ├─ 3d. 写入缩略图 → document.thumbnail_path
  │     │
  │     └─ 3e. 若有归档文件：
  │           generate_unique_filename(archive_filename=True) → archive_filename
  │           创建父目录
  │           写入归档文件 → document.archive_path
  │           计算 archive_checksum
  │
  ├─ 4. 释放锁，调用 document.save()
  │     └─ 触发 post_save → update_filename_and_move_files()
  │        （此时 filename 已有值，会根据最新 storage_path 等重新计算并移动）
  │
  └─ 5. 删除临时文件（原始/工作副本/unmodified_original/shadow file）
事务结束
```

---

## 九、文档删除时的文件清理

[documents/signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/signals/handlers.py#L342-L402) 的 `cleanup_document_deletion()`：

1. 获取 MEDIA_LOCK
2. 若配置了 EMPTY_TRASH_DIR：
   - 将原始文件移动到回收站目录（自动处理同名冲突）
   - 归档文件和缩略图直接删除
3. 否则三个文件全部直接删除
4. 清理原始文件和归档文件所在的空目录

---

## 十、核心文件索引

| 文件 | 主要职责 |
|------|---------|
| [documents/file_handling.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/file_handling.py) | 文件名生成、目录创建、空目录清理 |
| [documents/templating/filepath.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/templating/filepath.py) | 模板上下文构建、渲染、验证 |
| [documents/templating/environment.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/templating/environment.py) | Jinja2 沙箱环境配置 |
| [documents/templating/filters.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/templating/filters.py) | 自定义模板过滤器 |
| [documents/templating/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/templating/utils.py) | 旧格式到新格式的转换 |
| [documents/signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/signals/handlers.py) | 自动重命名/移动、删除清理 |
| [documents/consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/consumer.py) | 消费流程中的文件落盘 |
| [documents/models.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/models.py) | Document / StoragePath 模型定义及 path 属性 |
| [documents/management/commands/document_thumbnails.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/documents/management/commands/document_thumbnails.py) | 缩略图重新生成命令 |
| [paperless/settings/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/57-paperless-ngx/src/paperless/settings/__init__.py#L65-L88) | 目录配置和全局模板配置 |
