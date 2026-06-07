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

## 7. 关键文件索引

| 功能 | 文件路径 |
|------|----------|
| 数据模型定义 | [src/documents/models.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/models.py) |
| API 序列化 + 验证 | [src/documents/serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/serialisers.py) |
| 过滤器 + 查询解析器 | [src/documents/filters.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/filters.py) |
| 批量编辑 + DocLink 对称处理 | [src/documents/bulk_edit.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/bulk_edit.py) |
| URL 验证器 | [src/documents/validators.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/validators.py) |
| 搜索索引写入 | [src/documents/search/_backend.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/search/_backend.py) |
| 搜索索引 Schema | [src/documents/search/_schema.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/search/_schema.py) |
| API 视图 | [src/documents/views.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/views.py) |
| 前端数据模型 | [src-ui/src/app/data/custom-field.ts](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src-ui/src/app/data/custom-field.ts) |
| 前端展示组件 | [src-ui/src/app/components/common/custom-field-display/](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src-ui/src/app/components/common/custom-field-display/) |
| 前端编辑组件 | [src-ui/src/app/components/common/input/custom-fields-values/](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src-ui/src/app/components/common/input/custom-fields-values/) |
| 前端查询表达式 | [src-ui/src/app/utils/custom-field-query-element.ts](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src-ui/src/app/utils/custom-field-query-element.ts) |
