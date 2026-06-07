# Custom Fields 数据模型代码分析

## 1. 总体架构

Custom Fields 采用 **元数据模式（Metadata Pattern）** 实现，核心由两个模型组成：

| 模型 | 职责 | 对应表 |
|------|------|--------|
| `CustomField` | 定义自定义字段的 **元信息**：名称、数据类型、额外配置 | `documents_customfield` |
| `CustomFieldInstance` | 存储某文档上某自定义字段的 **具体值** | `documents_customfieldinstance` |

这种设计的好处：
- 字段定义与字段值分离，支持运行时动态增加字段
- 一个字段定义可被多个文档复用
- 每种数据类型有独立的存储列，保持类型安全

---

## 2. 字段定义：CustomField 模型

### 2.1 核心字段

定义位置：[models.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/models.py#L1036-L1091)

```python
class CustomField(models.Model):
    created = models.DateTimeField(default=timezone.now, db_index=True, editable=False)
    name = models.CharField(max_length=128)                    # 字段名，全局唯一
    data_type = models.CharField(max_length=50, choices=FieldDataType.choices, editable=False)
    extra_data = models.JSONField(null=True, blank=True)       # 扩展配置（select选项、默认货币等）
```

**约束**：`name` 全局唯一（`UniqueConstraint`）。

### 2.2 数据类型枚举（FieldDataType）

定义位置：[models.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/models.py#L1041-L1052)

| 类型值 | 存储列 | 说明 |
|--------|--------|------|
| `string` | `value_text` | 短文本，max_length=128 |
| `longtext` | `value_long_text` | 长文本，TextField |
| `url` | `value_url` | URL，使用 Django URLField |
| `date` | `value_date` | 日期，DateField |
| `boolean` | `value_bool` | 布尔值，BooleanField |
| `integer` | `value_int` | 整数，IntegerField（PostgreSQL int4 范围） |
| `float` | `value_float` | 浮点数，FloatField |
| `monetary` | `value_monetary` + `value_monetary_amount` | 货币（字符串存储 + 自动生成的数值列） |
| `documentlink` | `value_document_ids` | 文档链接，JSONField 存储 ID 数组 |
| `select` | `value_select` | 下拉选择，存储 option 的 id（max_length=16） |

### 2.3 extra_data 扩展配置

不同数据类型使用 `extra_data` 存储不同的配置信息：

- **SELECT 类型**：
  ```json
  {
    "select_options": [
      {"id": "abc123...", "label": "选项A"},
      {"id": "def456...", "label": "选项B"}
    ]
  }
  ```
  - `id` 为 16 位随机字符串（创建时自动生成，如未提供）
  - `label` 为用户可见的显示文本

- **MONETARY 类型**：
  ```json
  {
    "default_currency": "USD"   // 3 位 ISO 4217 货币代码
  }
  ```

---

## 3. 取值保存：CustomFieldInstance 模型

### 3.1 存储结构

定义位置：[models.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/models.py#L1093-L1227)

```python
class CustomFieldInstance(SoftDeleteModel):
    created = models.DateTimeField(default=timezone.now, db_index=True, editable=False)
    document = models.ForeignKey(Document, on_delete=models.CASCADE, related_name="custom_fields")
    field = models.ForeignKey(CustomField, on_delete=models.CASCADE, related_name="fields")

    # 各类型独立存储列
    value_text = models.CharField(max_length=128, null=True)
    value_bool = models.BooleanField(null=True)
    value_url = models.URLField(null=True)
    value_date = models.DateField(null=True)
    value_int = models.IntegerField(null=True)
    value_float = models.FloatField(null=True)
    value_monetary = models.CharField(null=True, max_length=128)
    value_monetary_amount = models.GeneratedField(...)   # 自动生成的数值列
    value_document_ids = models.JSONField(null=True)
    value_select = models.CharField(null=True, max_length=16)
    value_long_text = models.TextField(null=True)
```

**约束**：`(document, field)` 组合唯一，即一个文档对一个自定义字段只能有一个值。

### 3.2 类型映射：TYPE_TO_DATA_STORE_NAME_MAP

定义位置：[models.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/models.py#L1099-L1110)

这是核心的类型→列名映射字典，所有读写操作都通过它定位到正确的存储列：

```python
TYPE_TO_DATA_STORE_NAME_MAP = {
    STRING: "value_text",
    URL: "value_url",
    DATE: "value_date",
    BOOL: "value_bool",
    INT: "value_int",
    FLOAT: "value_float",
    MONETARY: "value_monetary",
    DOCUMENTLINK: "value_document_ids",
    SELECT: "value_select",
    LONG_TEXT: "value_long_text",
}
```

### 3.3 value 属性：统一读取入口

定义位置：[models.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/models.py#L1200-L1207)

```python
@property
def value(self):
    value_field_name = self.get_value_field_name(self.field.data_type)
    return getattr(self, value_field_name)
```

通过 `field.data_type` 自动定位到正确的列，外部代码只需调用 `instance.value` 即可获得正确类型的值。

### 3.4 Monetary 类型的特殊处理

`value_monetary_amount` 是一个 **数据库生成列（GeneratedField）**，定义位置：[models.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/models.py#L1152-L1171)

- 存储格式：`value_monetary` 存字符串，如 `"USD100.00"` 或 `"100.00"`
- 自动解析规则：
  - 如果以数字开头 → 整串转为 Decimal
  - 如果以非数字开头 → 剥离前 3 字符（ISO 货币代码），剩余部分转为 Decimal
- 用途：用于算术比较和排序（`gt`, `gte`, `lt`, `lte`, `exact`, `range`）

### 3.5 序列化器保存流程

核心序列化器：[serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/serialisers.py#L819-L925)

**CustomFieldInstanceSerializer.create()** 的工作流程：

```
1. 获取 document 和 custom_field 对象
2. 根据 custom_field.data_type 通过 get_value_field_name() 获得存储列名
3. 如果是 DOCUMENTLINK 类型，先调用 reflect_doclinks() 处理对称链接
4. 使用 update_or_create() 执行保存：
   CustomFieldInstance.objects.update_or_create(
       document=document,
       field=custom_field,
       defaults={data_store_name: validated_data["value"]}
   )
```

### 3.6 DocumentLink 的对称链接机制

定义位置：[bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/bulk_edit.py#L967-L1055)

`reflect_doclinks(document, field, target_doc_ids)` 函数确保文档链接是双向的：

- 当文档 A 的链接字段值为 `[B_id, C_id]` 时：
  - 文档 B 的同名字段自动包含 `[A_id]`
  - 文档 C 的同名字段自动包含 `[A_id]`
- 如果 A 移除了对 B 的链接，B 也自动移除对 A 的链接
- 防止自链接（`doc_id in value` 时跳过）

`remove_doclink(document, field, target_doc_id)` 用于删除单向的对称链接。

### 3.7 批量编辑

定义位置：[bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/bulk_edit.py#L287-L349)

```python
def modify_custom_fields(doc_ids, add_custom_fields, remove_custom_fields):
    # add_custom_fields 支持两种格式：
    #   1. dict: {field_id: value, ...}
    #   2. list: [field_id, field_id, ...] （值为 None）
```

同样使用 `update_or_create` 并处理 DOCUMENTLINK 的对称链接。

---

## 4. 验证规则

验证逻辑主要集中在两个序列化器中：

### 4.1 CustomFieldSerializer：字段定义的验证

定义位置：[serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/serialisers.py#L705-L783)

| 验证项 | 规则 |
|--------|------|
| name 唯一性 | 全局唯一，更新时排除自身 |
| SELECT 类型 | `extra_data.select_options` 必须是非空列表，且每个 option 必须有非空 label |
| MONETARY 类型 | `extra_data.default_currency` 如果设置，必须是 3 字符字符串或 None |
| SELECT option id | 若未提供 id，自动生成 16 位随机字符串 |

### 4.2 CustomFieldInstanceSerializer：字段值的验证

定义位置：[serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/serialisers.py#L849-L917)

每种数据类型的验证规则：

| 数据类型 | 验证逻辑 |
|----------|----------|
| `url` | 非空时调用 `uri_validator()`，需有 scheme + netloc/path |
| `integer` | Django `integer_validator` + PostgreSQL int4 范围（±2147483648） |
| `monetary` | 先尝试 DecimalValidator（12位精度2位小数），失败则匹配正则 `^[A-Z]{3}-?\d+(\.\d{1,2})$` |
| `string` | MaxLengthValidator(128) |
| `select` | 值必须是 `extra_data.select_options` 中某个 option 的 id |
| `documentlink` | 必须是列表；所有 doc_id 必须存在；当前用户对所有目标文档有 change 权限 |
| `date` | 通过 `serializers.DateField().to_internal_value()` 解析 |

URL 验证实现：[validators.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/validators.py#L7-L60)

### 4.3 DocumentSerializer 中的额外验证

- 文档更新时，如果传入的 `custom_fields` 移除了某些 DOCUMENTLINK 字段，自动调用 `remove_doclink()` 清理对称链接
- 更新完成后，**硬删除** 所有已软删除的 CustomFieldInstance：
  ```python
  CustomFieldInstance.deleted_objects.filter(document=instance).delete()
  ```

---

## 5. 搜索与查询

### 5.1 全文搜索索引（Tantivy）

索引写入：[_backend.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/search/_backend.py#L423-L436)

```python
for cfi in document.custom_fields.all():
    search_value = cfi.value_for_search   # 统一转为字符串
    if search_value is None:
        continue
    doc.add_json("custom_fields", {
        "name": cfi.field.name,
        "value": search_value,
    })
```

**value_for_search 属性**：[models.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/models.py#L1209-L1226)

- 对于 SELECT 类型：将存储的 option id 解析为用户可读的 label
- 其他类型：`str(self.value)`
- 值为 None 时返回 None，跳过索引

索引 Schema 定义：[_schema.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/search/_schema.py#L82)

`custom_fields` 被定义为 JSON 字段，支持结构化查询如：
- `custom_fields.name:发票编号`
- `custom_fields.value:100.00`

### 5.2 结构化查询：CustomFieldQueryParser

定义位置：[filters.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/filters.py#L367-L748)

这是一个递归下降解析器，将 JSON 格式的查询表达式转换为 Django `Q` 对象。

#### 查询语法

6 条规则（伪代码）：

```
1. [custom_field_id_or_name, "exists", true/false]
   → 是否存在该字段

2. [custom_field_id_or_name, "isnull", true/false]
   → 字段值是否为 null

3. [custom_field_id_or_name, op, value]
   → 字段值匹配条件（op 见下方）

4. ["AND", [q0, q1, ..., qn]]
   → 逻辑与

5. ["OR", [q0, q1, ..., qn]]
   → 逻辑或

6. ["NOT", q]
   → 逻辑非
```

#### 支持的操作符（按数据类型分组）

| 数据类型 | 支持的操作符组 | 操作符 |
|----------|---------------|--------|
| string, url, longtext, monetary | basic, string | exact, in, isnull, exists, icontains |
| date | basic, arithmetic | exact, in, isnull, exists, gt, gte, lt, lte, range, year__*, month__*, day__* 等 |
| boolean | basic | exact, in, isnull, exists |
| integer, float, monetary | basic, arithmetic | exact, in, isnull, exists, gt, gte, lt, lte, range |
| monetary（特殊） | basic, string, arithmetic | 同上 + icontains |
| documentlink | basic, containment | exists, isnull, contains |
| select | basic, subset | exact, in, isnull, exists |

Monetary 的算术比较使用 `value_monetary_amount`（自动生成的 Decimal 列）而非 `value_monetary` 字符串。

#### 安全限制

- `max_query_depth = 10`：最大嵌套深度
- `max_atom_count = 20`：最大原子条件数
- 前端限制（TypeScript）：`CUSTOM_FIELD_QUERY_MAX_DEPTH = 4`，`CUSTOM_FIELD_QUERY_MAX_ATOMS = 5`

#### 实现原理

每个原子条件通过 **Count + annotation** 方式实现，避免多实例 join 问题：

```python
# 1. 为该原子创建一个计数注解
annotation = Count("custom_fields", filter=Q(custom_fields__field=cf) & Q(value__op=value))
# 2. 过滤计数 > 0 的文档
query = Q(**{f"{annotation_name}__gt": 0})
```

DocumentLink 的 `contains` 操作有特殊处理：通过反向查询交集实现。

### 5.3 排序支持

定义位置：[filters.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/filters.py#L978-L1116)

`DocumentsOrderingFilter` 支持 `ordering=custom_field_{id}` 或 `ordering=-custom_field_{id}`。

每种数据类型的排序注解：

| 类型 | 排序字段 |
|------|----------|
| string, longtext | value_text |
| integer | value_int |
| float | value_float |
| date | value_date |
| monetary | value_monetary_amount |
| url | value_url |
| boolean | value_bool |
| documentlink | value_document_ids（JSON） |
| select | 使用 Case/When 将 option id 映射为按 label 排序的索引值 |

排序时：有字段的文档排在无字段的文档前面（`-has_field`）。

### 5.4 过滤查询参数（DocumentFilterSet）

定义位置：[filters.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/filters.py#L776-L868)

| 参数 | 说明 |
|------|------|
| `custom_fields__icontains` | 已弃用，全文搜索所有自定义字段名称和值 |
| `custom_fields__id__all` | 文档必须包含所有指定字段 |
| `custom_fields__id__none` | 文档不能包含任何指定字段 |
| `custom_fields__id__in` | 文档包含任一指定字段 |
| `has_custom_fields` | 布尔值，是否有任何自定义字段 |
| `custom_field_query` | JSON 格式的高级查询（CustomFieldQueryParser） |

### 5.5 前端查询表达式构建

前端 TypeScript 定义：
- [custom-field-query.ts](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src-ui/src/app/data/custom-field-query.ts)
- [custom-field-query-element.ts](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src-ui/src/app/utils/custom-field-query-element.ts)

两个核心类：
- `CustomFieldQueryAtom`：原子条件 `[field_id, operator, value]`
- `CustomFieldQueryExpression`：逻辑表达式 `["AND"|"OR"|"NOT", [...]]`

通过递归 serialize() 方法序列化为后端可解析的 JSON 数组。

---

## 6. 展示与编辑

### 6.1 前端展示组件

[custom-field-display.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src-ui/src/app/components/common/custom-field-display/custom-field-display.component.ts) + [HTML](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src-ui/src/app/components/common/custom-field-display/custom-field-display.component.html)

每种数据类型的渲染方式：

| 类型 | 展示方式 |
|------|----------|
| Monetary | `CurrencyPipe` 格式化，货币代码优先从值中提取，其次用字段默认值，最后用 locale 默认 |
| Date | `CustomDatePipe` 本地化格式化 |
| URL | 超链接，新窗口打开 |
| DocumentLink | 渲染为可点击的文档标题 badge 列表（需额外请求文档标题） |
| Boolean | 只读复选框 |
| Select | 解析 option id → label |
| LongText | 截断到 20 字符 + 省略号 + tooltip |
| 其他（string, integer, float） | 纯文本展示 |

### 6.2 前端编辑组件

[custom-fields-values.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src-ui/src/app/components/common/input/custom-fields-values/custom-fields-values.component.ts)

根据 `CustomField.data_type` 动态渲染对应的输入子组件：

- TextComponent, TextAreaComponent, DateComponent, NumberComponent, UrlComponent
- SelectComponent, MonetaryComponent, CheckComponent, DocumentLinkComponent

### 6.3 后端 Admin 管理

[admin.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/admin.py#L214-L247)

- `CustomFieldsAdmin`：只读 `created` 和 `data_type`，只可编辑 `name`
- `CustomFieldInstancesAdmin`：全部字段只读，用于查看

---

## 8. 保存入口全景：DocumentMetadataOverrides 与四大写入路径

Custom Fields 的保存并非只有 API 序列化器一条路径。系统中有一个核心数据载体 `DocumentMetadataOverrides`，以及围绕它的 **四大保存入口**。理解这些入口才能真正把握字段值是如何流转的。

### 8.1 数据载体：DocumentMetadataOverrides

定义位置：[data_models.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/data_models.py#L12-L147)

这是一个 dataclass，用于在消费/导入/工作流等多个环节之间传递「覆盖值」。与 Custom Fields 相关的字段：

```python
@dataclasses.dataclass
class DocumentMetadataOverrides:
    custom_fields: dict | None = None   # {field_id: value, ...}
```

#### update() 合并逻辑

[data_models.py L94-L97](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/data_models.py#L94-L97)

```python
if self.custom_fields is None:
    self.custom_fields = other.custom_fields
elif other.custom_fields is not None:
    self.custom_fields.update(other.custom_fields)
```

行为：后者覆盖前者的同名字段，保留不同字段。这使得多个工作流的赋值可以叠加。

#### from_document() 反向构造

[data_models.py L127-L130](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/data_models.py#L127-L130)

```python
overrides.custom_fields = {
    custom_field.field.id: custom_field.value
    for custom_field in doc.custom_fields.all()
}
```

从已有文档提取当前值，用于「文档更新」场景下的 websocket 状态同步等。

---

### 8.2 路径一：API / WebUI 上传 → 消费者异步写入

这是最常见的入口：用户在 WebUI 上传文件或通过 REST API POST `/api/documents/post_document/`。

#### 入口代码：views.py DocumentViewSet.post_document()

[views.py L3100-L3160](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/views.py#L3100-L3160)

```python
cf = serializer.validated_data.get("custom_fields")
# 支持两种格式：
#   dict: {field_id: value, ...}  → 直接使用
#   list: [field_id, ...]         → 值全部置为 None
custom_fields = None
if isinstance(cf, dict) and cf:
    custom_fields = cf
elif isinstance(cf, list) and cf:
    custom_fields = dict.fromkeys(cf, None)

input_doc_overrides = DocumentMetadataOverrides(
    ...,
    custom_fields=custom_fields,
)
consume_file.apply_async(kwargs={"input_doc": input_doc, "overrides": input_doc_overrides})
```

#### 异步消费流程：tasks.py consume_file()

[tasks.py L124-L220](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/tasks.py#L124-L220)

Celery 任务按顺序执行以下插件链（每个插件都可以修改 `overrides`）：

```
ConsumerPreflightPlugin
  → AsnCheckPlugin
  → CollatePlugin
  → BarcodePlugin
  → AsnCheckPlugin（条码后重检）
  → WorkflowTriggerPlugin  ← 在这里调用 CONSUMPTION 类型工作流，可能追加 custom_fields
  → ConsumerPlugin          ← 最终落库
```

每个插件的核心约定：
```python
plugin = plugin_class(input_doc, overrides, ...)
plugin.run()
overrides = plugin.metadata   # 覆盖，支持链式修改
```

#### WorkflowTriggerPlugin：消费阶段的工作流注入

[consumer.py L74-L87](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/consumer.py#L74-L87)

```python
class WorkflowTriggerPlugin:
    def run(self):
        overrides, msg = run_workflows(
            trigger_type=WorkflowTrigger.WorkflowTriggerType.CONSUMPTION,
            document=self.input_doc,
            overrides=DocumentMetadataOverrides(),  # 新建空的 overrides
        )
        if overrides:
            self.metadata.update(overrides)  # 与 API 传入的合并
```

注意这里传入的是一个**全新的空 overrides**，消费阶段工作流的赋值通过 `update()` 与 API 传入的值合并。

#### ConsumerPlugin.apply_overrides()：最终落库

[consumer.py L877-L938](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/consumer.py#L877-L938)

Document 对象创建后，调用 `apply_overrides()` 将 `metadata.custom_fields` 写入数据库：

```python
if self.metadata.custom_fields:
    for field in CustomField.objects.filter(
        id__in=self.metadata.custom_fields.keys(),
    ).distinct():
        value_field_name = CustomFieldInstance.get_value_field_name(
            data_type=field.data_type,
        )
        args = {
            "field": field,
            "document": document,
            value_field_name: self.metadata.custom_fields.get(field.id, None),
        }
        CustomFieldInstance.objects.create(**args)
```

**关键特征**：
- 使用 `objects.create()` 直接插入，**不经过序列化器验证**
- 不处理 DocumentLink 的对称链接（`reflect_doclinks`）
- 如果字段已存在（虽理论上不会发生，因为文档刚创建），会触发 UniqueConstraint 异常

---

### 8.3 路径二：文档导入导出（document_importer / document_exporter）

用于备份恢复或实例迁移。

#### 导出：document_exporter.py

[document_exporter.py L396-L397](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/management/commands/document_exporter.py#L396-L397)

```python
"custom_fields": CustomField.objects.all(),
"custom_field_instances": CustomFieldInstance.global_objects.all(),  # 包含软删除
```

使用 Django 标准序列化器，导出**所有字段值（含软删除）**。

#### 导入：document_importer.py load_data_to_database()

[document_importer.py L354-L430](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/management/commands/document_importer.py#L354-L430)

使用 `bulk_create(..., update_conflicts=True)` 按 PK 批量 upsert。**关键特征**：

1. **禁用信号**：导入时禁用以下信号以避免性能损耗和级联副作用
   - `update_filename_and_move_files`（Document + CustomFieldInstance）
   - `check_paths_and_prune_custom_fields`（CustomField）
   - auditlog 全部模型

2. **禁用约束检查**：`connection.constraint_checks_disabled()` 允许乱序导入

3. **排除 GeneratedField**：`value_monetary_amount` 由数据库自动生成，不手动写入

4. **不做任何验证**：直接反序列化写入，信任导出文件的完整性

---

### 8.4 路径三：工作流（Workflow）分配与移除

工作流是自动化的核心，Custom Fields 是其重要的操作对象。工作流在三种触发时机执行：

| 触发类型 | 触发时机 | 操作对象 | 执行模式 |
|----------|----------|----------|----------|
| `CONSUMPTION` | 文档消费过程中 | `DocumentMetadataOverrides` | 修改 overrides（不落库） |
| `DOCUMENT_ADDED` | 文档刚创建完成 | Document 实例 | 直接修改数据库 |
| `DOCUMENT_UPDATED` | 文档字段更新后 | Document 实例 | 直接修改数据库 |
| `SCHEDULED` | 定时调度 | Document 实例 | 直接修改数据库 |

#### WorkflowAction 的 Custom Fields 相关字段

[models.py L1671-L1784](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/models.py#L1671-L1784)

```python
class WorkflowAction:
    assign_custom_fields = models.ManyToManyField(CustomField, ...)
    assign_custom_fields_values = models.JSONField(...)  # {"field_id_str": value, ...}
    remove_custom_fields = models.ManyToManyField(CustomField, ...)
    remove_all_custom_fields = models.BooleanField(default=False)
```

`assign_custom_fields_values` 的 key 是字段 ID 的**字符串形式**，这是因为 JSONField 中 key 必须是字符串。

#### run_workflows()：统一调度器

[signals/handlers.py L854-L960](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/signals/handlers.py#L854-L960)

核心分支逻辑：

```python
if action.type == WorkflowAction.WorkflowActionType.ASSIGNMENT:
    if use_overrides:
        apply_assignment_to_overrides(action, overrides)      # 消费阶段：写到 overrides
    else:
        apply_assignment_to_document(action, document, ...)   # 已存在文档：直接落库

elif action.type == WorkflowAction.WorkflowActionType.REMOVAL:
    if use_overrides:
        apply_removal_to_overrides(action, overrides)
    else:
        apply_removal_to_document(action, document)
```

`use_overrides = (overrides is not None)`，消费阶段传 overrides 就走 overrides 分支，否则走 document 分支。

#### apply_assignment_to_document()：直接落库

[workflows/mutations.py L86-L110](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/workflows/mutations.py#L86-L110)

```python
if action.has_assign_custom_fields:
    for field in action.assign_custom_fields.all():
        value_field_name = CustomFieldInstance.get_value_field_name(
            data_type=field.data_type,
        )
        args = {
            value_field_name: action.assign_custom_fields_values.get(
                str(field.pk), None,
            ),
        }
        # 注释：for some reason update_or_create doesn't work here
        instance = CustomFieldInstance.objects.filter(
            field=field, document=document,
        ).first()
        if instance and args[value_field_name] is not None:
            setattr(instance, value_field_name, args[value_field_name])
            instance.save()
        elif not instance:
            CustomFieldInstance.objects.create(**args, field=field, document=document)
```

**关键特征**：
- 手动实现了 `update_or_create`（注释说 update_or_create 不工作，推测与 SoftDeleteModel 的管理器有关）
- **值为 None 时不更新已有实例**（静默跳过，不会清值）
- 不经过序列化器验证
- 不处理 DocumentLink 对称链接

#### apply_assignment_to_overrides()：写入 overrides

[workflows/mutations.py L180-L191](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/workflows/mutations.py#L180-L191)

```python
if action.has_assign_custom_fields:
    if overrides.custom_fields is None:
        overrides.custom_fields = {}
    overrides.custom_fields.update({
        field.pk: action.assign_custom_fields_values.get(str(field.pk), None)
        for field in action.assign_custom_fields.all()
    })
```

消费阶段不直接落库，而是写入 overrides，最终由 ConsumerPlugin.apply_overrides() 统一落库。

#### apply_removal_to_document()：硬删除实例

[workflows/mutations.py L266-L272](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/workflows/mutations.py#L266-L272)

```python
if action.remove_all_custom_fields:
    CustomFieldInstance.objects.filter(document=document).hard_delete()
elif action.has_remove_custom_fields:
    CustomFieldInstance.objects.filter(
        field__in=action.remove_custom_fields.all(),
        document=document,
    ).hard_delete()
```

使用 `hard_delete()` 直接从数据库删除（绕过软删除）。不会通知 DocumentLink 的对称方。

#### apply_removal_to_overrides()：从 overrides 中移除

[workflows/mutations.py L348-L354](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/workflows/mutations.py#L348-L354)

```python
if action.remove_all_custom_fields:
    overrides.custom_fields = None
elif action.has_remove_custom_fields and overrides.custom_fields:
    for field in action.remove_custom_fields.filter(
        pk__in=overrides.custom_fields.keys(),
    ):
        overrides.custom_fields.pop(field.pk, None)
```

#### SCHEDULED 触发器的特殊用法

[tasks.py L494-L506](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/tasks.py#L494-L506)

调度工作流可以用某个 CustomField 的日期值作为「调度触发时间」：

```python
cf_filter_kwargs = {
    "field": trigger.schedule_date_custom_field,
    "value_date__isnull": False,
    "value_date__lte": threshold,
    "value_date__gte": earliest_date,
}
recent_cf_instances = CustomFieldInstance.objects.filter(**cf_filter_kwargs)
matched_ids = [cfi.document_id for cfi in recent_cf_instances]
```

这是 CustomFieldInstance 直接被业务逻辑查询的典型场景，绕过了搜索索引，使用 ORM 直接过滤。

---

### 8.5 路径四：信号驱动的级联更新

CustomField **定义**的变更会触发已有实例的级联更新。

#### check_paths_and_prune_custom_fields + process_cf_select_update

[signals/handlers.py L671-L710](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/signals/handlers.py#L671-L710)

当 CustomField（且类型为 SELECT）保存时触发：

```python
@receiver(models.signals.post_save, sender=CustomField)
def check_paths_and_prune_custom_fields(sender, instance, **kwargs):
    if instance.data_type == CustomField.FieldDataType.SELECT and instance.fields.count() > 0:
        process_cf_select_update.apply_async(kwargs={"custom_field": instance})
```

异步任务 `process_cf_select_update` 做两件事：

1. **清理失效选项**：已移除的 option id 对应的实例值被置为 None
   ```python
   select_options = {opt["id"]: opt["label"] for opt in custom_field.extra_data.get("select_options", [])}
   custom_field.fields.exclude(value_select__in=select_options.keys()).update(value_select=None)
   ```

2. **触发文件名更新**：如果文件名模板使用了该自定义字段，重新生成文件名
   ```python
   for cf_instance in custom_field.fields.select_related("document").iterator():
       update_filename_and_move_files(CustomFieldInstance, cf_instance)
   ```

#### update_filename_and_move_files：文件名联动

[signals/handlers.py L431-L442](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/signals/handlers.py#L431-L442)

```python
@receiver(models.signals.post_save, sender=CustomFieldInstance, weak=False)
def update_filename_and_move_files(sender, instance, **kwargs):
    if isinstance(instance, CustomFieldInstance):
        if not _filename_template_uses_custom_fields(instance.document):
            return
        instance = instance.document
    # ... 后续执行文件重命名和移动
```

只要文件名模板中包含 `custom_fields` 占位符，任何 CustomFieldInstance 的保存都会触发文件重命名。

---

## 9. 各保存路径与验证、搜索、展示的关系

### 9.1 四大路径的验证差异对比

并非所有路径都经过第4章描述的序列化器验证。以下是完整对比：

| 保存路径 | 是否经过 CustomFieldInstanceSerializer 验证 | 是否处理 DocumentLink 对称链接 | 值为 None 时行为 |
|----------|------------------------------------------|-----------------------------|------------------|
| **API 单实例**（CustomFieldInstanceViewSet） | ✅ 完整验证（URL、整数范围、货币格式、SELECT id 合法性、DocumentLink 权限等） | ✅ `reflect_doclinks` / `remove_doclink` | 清空该字段值 |
| **API 文档更新**（DocumentSerializer.custom_fields） | ✅ 内部调用 CustomFieldInstanceSerializer | ✅ | 清空该字段值 |
| **批量编辑**（bulk_edit.modify_custom_fields） | ❌ 无验证 | ✅ `reflect_doclinks`（仅 add） | 清值或创建空实例 |
| **API 上传→ConsumerPlugin** | ❌ 无验证 | ❌ | 创建值为 None 的实例 |
| **工作流 apply_assignment_to_document** | ❌ 无验证 | ❌ | 不更新已有实例（静默跳过） |
| **工作流 apply_removal_to_document** | N/A（删除） | ❌ | hard_delete |
| **document_importer 导入** | ❌ 无验证，信任导出数据 | ❌ | 按导出值原样写入 |
| **SELECT 定义变更→process_cf_select_update** | ❌ 只清理失效选项 id | N/A | 失效选项置为 None |

#### 验证缺失的影响与风险

1. **DocumentLink 不对称**：通过消费者/工作流写入的文档链接不会自动创建反向链接。只有 API 单实例/文档更新和批量编辑路径会调用 `reflect_doclinks()`。
   - 修复方式：在这些路径手动调用，或接受「非 API 路径产生的链接是单向的」。

2. **非法值可落库**：消费者和工作流路径没有类型检查。如果工作流配置了非法的日期字符串或超出 int4 范围的整数，会触发数据库层 IntegrityError 或静默产生脏数据。
   - 设计考量：工作流的字段值是管理员在后台配置的，假设其可信；消费者的 metadata 来源也是受控的（API 或工作流）。

3. **SELECT 无效值**：除了 API 路径会校验 option id 是否存在，其他路径都不校验。但 `process_cf_select_update` 会在字段定义变更时做一次兜底清理。

### 9.2 保存与搜索索引的关系

搜索索引的更新依赖 Django 信号，不依赖具体保存路径。

#### add_to_index 信号

[signals/handlers.py L794-L800](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/signals/handlers.py#L794-L800)

`document_consumption_finished` 和 `document_updated` 信号触发 `add_to_index()`，调用搜索后端的 `add_or_update()`。

搜索后端遍历 `document.custom_fields.all()`，使用 `value_for_search` 属性将值转为字符串后写入 Tantivy 索引（见 5.1 节）。

#### 各路径的索引触发情况

| 保存路径 | 触发搜索重索引的方式 |
|----------|-------------------|
| API 单实例保存/删除 | Document.post_save → document_updated 信号 |
| API 文档更新 | Document.post_save → document_updated 信号 |
| 批量编辑 | bulk_update_documents 完成后手动发送 document_updated |
| API 上传→ConsumerPlugin | `document_consumption_finished` 信号 |
| 工作流（DOCUMENT_ADDED/UPDATED） | 触发 Document.save() → document_updated 信号 |
| 工作流（CONSUMPTION） | 最终由 ConsumerPlugin 触发 document_consumption_finished |
| document_importer 导入 | ⚠️ **禁用信号**，导入后需手动调用 `document_index` 命令重建索引 |
| SELECT 定义变更 | 触发 CustomFieldInstance 的 post_save → update_filename_and_move_files，但**不会触发 Document 的 document_updated** |

**注意**：`process_cf_select_update` 虽然修改了 CustomFieldInstance 的值（`value_select=None`），但它触发的是 CustomFieldInstance 的 post_save，而搜索索引依赖 Document 级别的信号。因此 SELECT 字段 option 被删除后，已有文档的搜索索引可能还残留旧的 label 值，需要手动重建索引或等待下次文档更新。

### 9.3 保存与前端展示的关系

前端展示依赖 API 返回的完整数据，展示逻辑本身是「读时处理」（SELECT id→label、货币格式化、日期本地化），与保存路径无关。但保存路径影响**数据能否被正确读取**：

1. **SELECT 类型的读写一致性**：
   - 保存时写入的是 option id（16 位字符串）
   - 展示时从 `field.extra_data.select_options` 中反查 label
   - 如果非 API 路径写入了不存在的 option id，展示时会显示空值
   - `process_cf_select_update` 的清理逻辑可以部分缓解此问题

2. **DocumentLink 类型的展示依赖**：
   - 展示组件需要额外请求文档标题（`value_document_ids` 只存 ID）
   - 不对称的链接（通过消费者/工作流写入）在目标文档的展示中不会出现反向链接

3. **文件名模板与保存路径的交互**：
   - 如果文件名模板使用了 `{custom_fields.xxx}`，CustomFieldInstance 的保存会触发文件重命名
   - document_importer 导入时禁用了该信号，导入后文件名可能与模板不匹配，需要手动 `renaming_suggestions` 或重新触发

4. **Websocket 通知**：
   - API 更新文档后，`send_websocket_document_updated` 会使用 `DocumentMetadataOverrides.from_document()` 提取当前 custom_fields 值，推送给前端
   - 工作流修改 Document 后触发的 Document.save() 也会走同样路径
   - 消费者路径在 `document_consumption_finished` 后同样会通知

### 9.4 全景关系图

```
                    ┌────────────────────────────────────────────┐
                    │          保存入口（四条路径）                │
                    └────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
            ┌───────────┐   ┌───────────┐   ┌───────────┐
            │序列化器验证│   │  直接ORM  │   │ bulk导入  │
            │ (仅API)   │   │(工作流/消费)│   │(无验证)   │
            └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
                  │               │               │
                  └───────────────┼───────────────┘
                                  ▼
                    ┌──────────────────────────────┐
                    │   CustomFieldInstance 落库     │
                    └───────────────┬──────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
            ┌───────────┐   ┌───────────┐   ┌───────────┐
            │post_save  │   │搜索索引   │   │前端展示   │
            │信号触发   │──▶│add_to_index│  │(读时处理) │
            │文件重命名 │   │(需信号)   │   │id→label   │
            └───────────┘   └───────────┘   └───────────┘
```

---

## 10. 关键文件索引

| 功能 | 文件路径 |
|------|----------|
| 数据模型定义 | [src/documents/models.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/models.py) |
| MetadataOverrides 数据载体 | [src/documents/data_models.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/data_models.py) |
| API 序列化 + 验证 | [src/documents/serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/serialisers.py) |
| 过滤器 + 查询解析器 | [src/documents/filters.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/filters.py) |
| 批量编辑 + DocLink 对称处理 | [src/documents/bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/bulk_edit.py) |
| 消费者主流程（落库逻辑） | [src/documents/consumer.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/consumer.py) |
| Celery 任务调度（consume_file） | [src/documents/tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/tasks.py) |
| 工作流分配/移除 mutations | [src/documents/workflows/mutations.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/workflows/mutations.py) |
| 工作流执行上下文/actions | [src/documents/workflows/actions.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/workflows/actions.py) |
| 工作流工具（Prefetch/annotate） | [src/documents/workflows/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/workflows/utils.py) |
| 信号处理（工作流、索引、文件名） | [src/documents/signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/signals/handlers.py) |
| 文档导入命令 | [src/documents/management/commands/document_importer.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/management/commands/document_importer.py) |
| 文档导出命令 | [src/documents/management/commands/document_exporter.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/management/commands/document_exporter.py) |
| URL 验证器 | [src/documents/validators.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/validators.py) |
| 搜索索引写入 | [src/documents/search/_backend.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/search/_backend.py) |
| 搜索索引 Schema | [src/documents/search/_schema.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/search/_schema.py) |
| API 视图（上传入口） | [src/documents/views.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/views.py) |
| 前端数据模型 | [src-ui/src/app/data/custom-field.ts](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src-ui/src/app/data/custom-field.ts) |
| 前端展示组件 | [src-ui/src/app/components/common/custom-field-display/](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src-ui/src/app/components/common/custom-field-display/) |
| 前端编辑组件 | [src-ui/src/app/components/common/input/custom-fields-values/](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src-ui/src/app/components/common/input/custom-fields-values/) |
| 前端查询表达式 | [src-ui/src/app/utils/custom-field-query-element.ts](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src-ui/src/app/utils/custom-field-query-element.ts) |
