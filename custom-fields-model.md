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

### 3.5 序列化器保存流程（API 写入入口的真实行为）

> **关键事实核对**：系统中**没有独立的 CustomFieldInstance API 端点**。
> URL 路由 [paperless/urls.py L88](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/paperless/urls.py#L88) 仅注册了 `custom_fields`（CustomField 定义的 CRUD），不存在 `custom_field_instances` 路由。
>
> CustomFieldInstance 只能通过 **DocumentSerializer 的嵌套字段 `custom_fields`** 进行写入。

#### 序列化器链路

核心序列化器：[serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/serialisers.py#L989-L1258)

```python
class DocumentSerializer(OwnedObjectSerializer, NestedUpdateMixin, DynamicFieldsModelSerializer):
    custom_fields = CustomFieldInstanceSerializer(many=True, allow_null=False, required=False)
```

`NestedUpdateMixin` 来自第三方库 `drf_writable_nested`，它会对嵌套的 `custom_fields` 数组中的每个元素自动调用 `CustomFieldInstanceSerializer.create()` 或 `.update()`。

#### CustomFieldInstanceSerializer.create() 的工作流程

[serialisers.py L819-L844](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/serialisers.py#L819-L844)

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

#### DocumentSerializer.update() 中的额外处理

[serialisers.py L1130-L1210](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/serialisers.py#L1130-L1210)

1. **DocumentLink 字段移除检测**：如果新传入的 custom_fields 中不再包含某个旧的 DOCUMENTLINK 字段，对该字段的每个目标文档调用 `bulk_edit.remove_doclink()` 清理反向链接。

2. **硬删除已软删除的实例**：更新完成后执行
   ```python
   CustomFieldInstance.deleted_objects.filter(document=instance).delete()
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

#### 阶段一：PostDocumentSerializer 校验（上传入口）

[serialisers.py L2088-L2255](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/serialisers.py#L2088-L2255)

`custom_fields` 字段定义为 `serializers.JSONField`，支持两种输入格式，由 `validate_custom_fields()` 方法校验。

**格式一：dict `{field_id: value, ...}`** — 每个字段指定具体值

```python
# 校验流程：
for field_id, value in custom_fields.items():
    1. field_id 转为 int，失败抛错
    2. CustomField.objects.get(id=field_id_int)，不存在抛错
    3. ★ 调用 CustomFieldInstanceSerializer.validate({field, value})
    4. 存入 normalized {field_id_int: value}
```

> **⚠️ 关键边界：DocumentLink 在上传阶段不校验用户权限**
>
> `validate_custom_fields()` 中调用序列化器验证的方式是：
> ```python
> custom_field_serializer = CustomFieldInstanceSerializer()  # 未传 context！
> custom_field_serializer.validate({"field": field, "value": value})
> ```
>
> 由于 `context` 为空，在 `CustomFieldInstanceSerializer.validate()` 中：
> ```python
> request = self.context.get("request")  # 返回 None
> validate_documentlink_targets(
>     getattr(request, "user", None) if request is not None else None,  # user = None
>     doc_ids,
> )
> ```
>
> `validate_documentlink_targets` 函数逻辑：
> ```python
> if Document.objects.filter(id__in=doc_ids).count() != len(doc_ids):
>     raise ValidationError("Some documents ... don't exist or were specified twice.")
> if user is None:
>     return  # ← 直接返回，跳过权限校验！
> # 以下权限校验（has_perms_owner_aware）在上传阶段不会执行
> ```
>
> **结论**：上传阶段 DocumentLink 只校验**目标文档存在且不重复**，不校验当前用户对目标文档的 `change_document` 权限。
> 这与 API 文档更新（PUT/PATCH）路径不同——PUT/PATCH 通过 DocumentSerializer 嵌套字段传入，context 被正确传递，会执行完整权限校验。

**格式二：list `[field_id, ...]`** — 字段值全为 None

```python
# 校验流程：
1. 全部元素转为 int，失败抛错
2. CustomField.objects.filter(id__in=ids).count() != len(set(ids)) → 抛错
3. 直接返回 ids 列表
```

> **⚠️ 关键边界：list 格式重复字段检测条件有缺陷，重复 id 不会被检测到**
>
> 判断条件为：
> ```python
> if CustomField.objects.filter(id__in=ids).count() != len(set(ids)):
>     raise ValidationError("Some custom fields don't exist or were specified twice.")
> ```
>
> `set(ids)` 在比较之前就已经去除了重复。举例验证：
>
> | 输入 ids | len(ids) | len(set(ids)) | DB count | 条件结果 | 实际行为 |
> |----------|----------|---------------|----------|----------|----------|
> | `[1, 2, 3]`（正常） | 3 | 3 | 3 | `3 != 3` → False | ✅ 通过 |
> | `[1, 1, 2]`（id=1 重复） | 3 | **2**（set 去重） | 2 | `2 != 2` → False | ✅ **通过（漏检！）** |
> | `[1, 2, 999]`（id=999 不存在） | 3 | 3 | 2 | `2 != 3` → True | ❌ 抛错 |
>
> 后续在 [views.py L3134-L3135](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/views.py#L3134-L3135)：
> ```python
> elif isinstance(cf, list) and cf:
>     custom_fields = dict.fromkeys(cf, None)  # dict key 自动去重
> ```
>
> **结论**：list 格式中**重复的自定义字段 id 不会触发校验错误**，会被 `dict.fromkeys()` 静默合并为单个键（值为 None）。
> 错误消息中写的 "were specified twice" 实际上无法检测到，只有不存在的 id 会被检测到。

> **事实核对**：API 上传阶段就已经通过 `CustomFieldInstanceSerializer.validate()` 做了类型校验（dict 格式），
> 但存在两个边界：① DocumentLink 不校验用户权限；② list 格式重复 id 无法检测。ConsumerPlugin 最终落库时不会再次校验。

#### 阶段二：views.py 组装 overrides 并投递任务

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

#### WorkflowTriggerPlugin：消费阶段的工作流注入（可能引入未校验的值）

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

> **事实核对**：
> 1. 这里传入的是一个**全新的空 overrides**，消费阶段工作流的赋值通过 `update()` 与 API 传入的值合并
> 2. ⚠️ **工作流注入的 custom_fields 值完全不经过校验**。`run_workflows → apply_assignment_to_overrides` 直接把工作流配置中的值写入 overrides dict
> 3. 如果工作流配置了非法值（如无效日期、超出 int4 范围的整数），会直接通过后续的落库逻辑写入数据库，可能触发数据库层异常或产生脏数据

#### ConsumerPlugin.apply_overrides()：最终落库（信任上游数据）

[consumer.py L877-L938](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/consumer.py#L877-L938)

Document 对象创建后，调用 `apply_overrides()` 将 `metadata.custom_fields` 写入数据库：

```python
# 先设置 document 自身的字段（correspondent、document_type、tags、storage_path、asn、权限等）
...
# 最后处理 custom_fields
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
- 使用 `CustomFieldInstance.objects.create()` 直接插入，**完全不经过序列化器验证**
- 不处理 DocumentLink 的对称链接（不调用 `reflect_doclinks`）
- 由于文档刚创建，UniqueConstraint 理论上不会触发；若同一 custom_fields 中存在重复 key，由 `filter().distinct()` 去重
- 写入发生在 Document.save() **之前**（`apply_overrides` 修改完 document 后，外部才调用 `document.save()`）

**API 上传 → 落库的完整校验链路：**

```
API 上传
  ↓ PostDocumentSerializer.validate_custom_fields()
  ↓ ✅ CustomFieldInstanceSerializer.validate() 完整校验（仅 dict 格式的值）
  ↓ 封装为 DocumentMetadataOverrides
  ↓ Celery consume_file()
  ↓ WorkflowTriggerPlugin（消费阶段工作流）
  ↓ ⚠️ 工作流注入的值不校验，直接 update() 合并
  ↓ ConsumerPlugin.apply_overrides()
  ↓ ❌ 不再校验，直接 CustomFieldInstance.objects.create()
  ↓ document.save()
  ↓ document_consumption_finished 信号
```

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

#### 导入：document_importer.py

完整执行流程在 `_run_import()` 方法：[document_importer.py L451-L509](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/management/commands/document_importer.py#L451-L509)

使用 `bulk_create(..., update_conflicts=True)` 按 PK 批量 upsert（`load_data_to_database` L354-L430）。

**关键特征**：

1. **禁用的信号列表**（`_run_import` 中通过 `disable_signal()` context manager 精确禁用）：
   - `post_save → update_filename_and_move_files`（sender=Document）
   - `m2m_changed → update_filename_and_move_files`（sender=Document.tags.through）
   - `post_save → update_filename_and_move_files`（sender=CustomFieldInstance）
   - `post_save → check_paths_and_prune_custom_fields`（sender=CustomField）
   - auditlog：unregister Document、Correspondent、Tag、DocumentType、Note、CustomField、CustomFieldInstance

2. **不禁用的信号**：
   - 自定义信号 `document_consumption_finished` 和 `document_updated`（导入过程中也不会发送）
   - CustomFieldInstance 的其他 post_save receiver（如果有）

3. **禁用约束检查**：`connection.constraint_checks_disabled()` 允许乱序导入

4. **排除 GeneratedField**：`value_monetary_amount` 由数据库自动生成，不手动写入

5. **不做任何验证**：直接反序列化写入，信任导出文件的完整性

6. **导入结束自动重建索引**：
   ```python
   # 在 with disable_signal(...) 上下文之外执行
   self.stdout.write("Updating search index...")
   call_command("document_index", "reindex", no_progress_bar=self.no_progress_bar)
   ```
   > **事实核对**：document_importer **在禁用信号的上下文退出后**，自动调用 `document_index reindex` 重建搜索索引。
   > 这与之前描述的"需手动调用"不同，实际是自动完成的。
   > 之所以在信号禁用上下文之外调用，是因为 reindex 命令本身可能需要发送信号。

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

1. **批量清理失效选项**：已移除的 option id 对应的实例值被置为 None
   ```python
   select_options = {opt["id"]: opt["label"] for opt in custom_field.extra_data.get("select_options", [])}
   custom_field.fields.exclude(value_select__in=select_options.keys()).update(value_select=None)
   ```
   > **事实核对**：此处使用的是 Django ORM 的 `.update()` 方法，它直接执行 SQL `UPDATE`，**不触发 post_save 信号**。
   > 这意味着：
   > - 不会触发 `update_filename_and_move_files`（被清理为 None 的实例不会触发文件名更新）
   > - **不会触发搜索索引更新**（搜索索引也依赖信号链）

2. **逐个触发文件名更新**：遍历所有有该字段的文档，手动调用 `update_filename_and_move_files`
   ```python
   for cf_instance in custom_field.fields.select_related("document").iterator():
       update_filename_and_move_files(CustomFieldInstance, cf_instance)
   ```
   > **事实核对**：这个 `for` 循环遍历**所有**有该字段的实例（包括值未被清理的），手动传入 `sender=CustomFieldInstance`。
   > `update_filename_and_move_files` 内部会检查模板是否使用了 `custom_fields`，如未使用则直接 return。
   > 这个循环**不会触发搜索索引更新**，因为它只处理文件名逻辑，不会发送 `document_updated` 信号。

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

| 保存路径 | 是否经过 CustomFieldInstanceSerializer 验证 | 是否校验 DocumentLink 目标权限 | 是否处理 DocumentLink 对称链接 | 值为 None 时行为 |
|----------|------------------------------------------|---------------------------|-----------------------------|------------------|
| **API 文档更新**（PUT/PATCH，DocumentSerializer.custom_fields） | ✅ 内部调用，完整验证 | ✅ 校验当前用户 `change_document` 权限 | ✅ `reflect_doclinks` + `remove_doclink` | 清空该字段值 |
| **API 上传阶段一**（POST /post_document，PostDocumentSerializer） | ✅ dict 格式完整验证；list 格式仅校验 id 存在 | ⚠️ **不校验权限**（context 为空，user=None），仅校验文档存在且不重复 | N/A（还未落库） | N/A（还未落库） |
| **API 上传阶段二**（Celery 消费，ConsumerPlugin.apply_overrides） | ❌ 不再验证（信任上游） | ❌ | ❌ | 创建值为 None 的实例 |
| **批量编辑**（bulk_edit.modify_custom_fields） | ❌ 无验证 | ❌ | ✅ `reflect_doclinks`（仅 add） | 清值或创建空实例 |
| **工作流 apply_assignment_to_document** | ❌ 无验证 | ❌ | ❌ | 不更新已有实例（静默跳过） |
| **工作流 apply_removal_to_document** | N/A（删除） | N/A | ❌ | hard_delete |
| **document_importer 导入** | ❌ 无验证，信任导出数据 | ❌ | ❌ | 按导出值原样写入 |
| **SELECT 定义变更→process_cf_select_update** | ❌ 只清理失效选项 id | N/A | N/A | 失效选项置为 None |

> **事实核对**：不存在独立的 "CustomFieldInstanceViewSet"。API 路由 [paperless/urls.py L88](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/paperless/urls.py#L88) 仅注册 `custom_fields`（CustomField 定义 CRUD）。
> CustomFieldInstance 只能通过 DocumentSerializer 的嵌套 `custom_fields` 字段写入，由第三方库 `drf_writable_nested.NestedUpdateMixin` 调度。

#### 验证缺失的影响与风险

1. **DocumentLink 不对称**：通过消费者/工作流写入的文档链接不会自动创建反向链接。只有 API 文档更新和批量编辑路径会调用 `reflect_doclinks()`。
   - 修复方式：在这些路径手动调用，或接受「非 API 路径产生的链接是单向的」。

2. **DocumentLink 权限边界（上传 vs 更新）**：
   - 上传阶段（POST `/post_document/`）不校验用户对目标文档的权限，可能让用户创建指向无权限文档的链接
   - 文档更新阶段（PUT/PATCH）会校验，是因为 DocumentSerializer 嵌套序列化器自动传递了 `context`（含 request.user）
   - 代码根因：PostDocumentSerializer 中手动实例化 `CustomFieldInstanceSerializer()` 时**未传 context**，导致 `self.context.get("request")` 返回 None
   - 代码位置：[serialisers.py L2212-L2234](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/serialisers.py#L2212-L2234)

3. **list 格式重复字段漏检**：
   - 重复的自定义字段 id 不会触发校验错误（`set()` 在比较前已去重），会被 `dict.fromkeys()` 静默合并
   - 错误消息 "were specified twice" 具有误导性，实际该条件只能检测到不存在的 id
   - 代码位置：[serialisers.py L2246-L2249](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/serialisers.py#L2246-L2249)

4. **非法值可落库**：消费者和工作流路径没有类型检查。如果工作流配置了非法的日期字符串或超出 int4 范围的整数，会触发数据库层 IntegrityError 或静默产生脏数据。
   - 设计考量：工作流的字段值是管理员在后台配置的，假设其可信；消费者的 metadata 来源也是受控的（API 或工作流）。

5. **SELECT 无效值**：除了 API 路径会校验 option id 是否存在，其他路径都不校验。但 `process_cf_select_update` 会在字段定义变更时做一次兜底清理。

### 9.2 保存与搜索索引的关系

搜索索引的更新**不完全依赖信号**，部分路径通过手动调用后端 API 完成。

#### 信号连接的真实情况

[apps.py L10-L37](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/apps.py#L10-L37)

```python
document_consumption_finished.connect(add_to_index)       # ✅ 首次消费连接
document_consumption_finished.connect(run_workflows_added)
document_updated.connect(run_workflows_updated)
document_updated.connect(send_websocket_document_updated)
# ⚠️ 注意：document_updated 没有连接 add_to_index！
```

> **事实核对**：只有 `document_consumption_finished` 信号连接了 `add_to_index`。
> `document_updated` 信号**不触发搜索索引更新**，它只触发工作流和 Websocket。
> 非首次消费的场景必须**手动调用**搜索后端 API。

#### 各路径的索引触发真实情况

| 保存路径 | 触发搜索重索引的方式 |
|----------|-------------------|
| **API 文档更新**（PUT/PATCH） | views.py DocumentViewSet.update() **手动调用** `get_backend().add_or_update(refreshed_doc)`（L1174-L1176） |
| **批量编辑**（bulk_edit API） | tasks.py `bulk_update_documents` **手动批量调用** `batch.add_or_update(doc)`（L267-L269） |
| **API 上传→首次消费** | `document_consumption_finished` 信号 → `add_to_index` |
| **消费者添加新版本**（已有 root_document） | consumer.py L732 发送 `document_updated` 信号，但该信号不连接 add_to_index；**无手动调用**，可能导致新版本下 custom_fields 变更未反映到索引 |
| **工作流（DOCUMENT_ADDED）** | 首次消费路径，由 `document_consumption_finished` → `add_to_index` 覆盖 |
| **工作流（DOCUMENT_UPDATED）** | `document_updated` 信号 → 不连接 add_to_index；工作流 mutation 直接 save CustomFieldInstance，**无索引更新** |
| **工作流（CONSUMPTION）** | 最终由 ConsumerPlugin 触发 `document_consumption_finished` 覆盖 |
| **document_importer 导入** | 导入过程中禁用信号，但 **导入结束后自动调用 `document_index reindex`** 重建索引（在信号禁用上下文之外执行） |
| **SELECT 定义变更→process_cf_select_update** | `.update(value_select=None)` 是批量 SQL，不触发任何信号；**完全无索引更新**。for 循环只调用文件名更新函数，不更新搜索索引 |

搜索后端遍历 `document.custom_fields.all()`，使用 `value_for_search` 属性将值转为字符串后写入 Tantivy 索引（见 5.1 节）。

#### 索引与数据库不一致的场景

以下场景可能导致搜索索引残留旧的 Custom Fields 值：

1. **SELECT 字段 option 被删除**：`process_cf_select_update` 批量更新数据库，但不更新索引
2. **工作流修改 Custom Fields（DOCUMENT_UPDATED 触发）**：实例被 save，但索引不更新
3. **消费者添加新版本**：如果新版本路径改变了 custom_fields（理论上不会，因为版本是同一文档的不同文件），索引不会同步

修复方式：调用 `document_index` 管理命令重建索引，或手动触发一次 API 文档更新。

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

   > **事实核对**：Websocket `document_updated` 消息的 payload **完全不包含 custom_fields 值**。
   > 它只是一个轻量通知，告知前端"该文档有更新，请重新拉取"。

   Websocket payload 定义：[plugins/helpers.py L163-L181](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/plugins/helpers.py#L163-L181)

   ```python
   payload = {
       "type": "document_updated",
       "data": {
           "document_id": document_id,
           "modified": modified,
           "owner_id": owner_id,             # 权限相关
           "users_can_view": [...],          # 权限相关
           "groups_can_view": [...],         # 权限相关
       },
   }
   ```

   `DocumentMetadataOverrides.from_document(document)` 在 handler 中确实提取了 `custom_fields`，但该值**未被发送到 Websocket**，仅 `owner_id`、`view_users`、`view_groups` 被使用。

   Websocket 通知的触发点：
   - API 文档更新：views.py `DocumentViewSet.update()` 发送 `document_updated` 信号
   - 批量编辑：tasks.py `bulk_update_documents` 对每个文档发送信号
   - 消费者添加新版本：consumer.py L732 对 root_document 发送信号
   - 删除版本 / 更新版本标签等：views.py 其他位置
   - **注意**：首次消费完成（`document_consumption_finished`）不触发 `document_updated`，而是通过 `_send_progress(SUCCESS, document_id=...)` 通知前端

### 9.4 全景关系图（经代码事实核对）

```
                    ┌──────────────────────────────────────────────┐
                    │           保存入口（四条路径）                  │
                    └──────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼────────────────┐
                    ▼               ▼                ▼
            ┌───────────┐   ┌──────────────┐   ┌────────────┐
            │序列化器验证│   │  直接ORM     │   │ bulk导入   │
            │(仅API文档) │   │(工作流/消费者)│   │ (无验证)   │
            └─────┬─────┘   └──────┬───────┘   └─────┬──────┘
                  │                │                  │
                  └────────────────┼──────────────────┘
                                   ▼
                    ┌────────────────────────────────┐
                    │     CustomFieldInstance 落库      │
                    └────────────────┬───────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    ▼                ▼                ▼
           ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
           │CustomField   │  │ 搜索索引更新  │  │ 前端展示     │
           │post_save信号 │  │ (手动调用/   │  │ (读时处理)    │
           │→ 文件重命名   │  │  仅首次消费走 │  │ SELECT id    │
           │              │  │   信号)      │  │  → label     │
           └──────────────┘  └──────┬───────┘  └──────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
            document_        document_          API 更新时
            consumption_      updated 信号       手动调用
            finished信号     → 工作流 + Websocket  get_backend()
            → add_to_index   (不触发索引更新)    .add_or_update()
```

**关键事实**：
1. 搜索索引更新与信号是松耦合的：只有首次消费走信号链，其余路径需手动调用后端 API
2. `document_updated` 信号只触发工作流（DOCUMENT_UPDATED）和 Websocket 通知，不触发索引
3. Websocket payload 不含 custom_fields 数据，仅通知前端重新拉取
4. `process_cf_select_update` 的 `.update()` 批量操作不触发任何信号

### 9.5 API 上传校验、落库行为与导入索引重建的关系总结

#### 关系一：API 上传校验与最终落库的"一次校验"原则

API 上传的 custom_fields 值在 **PostDocumentSerializer.validate_custom_fields()** 阶段完成校验（dict 格式），之后的链路：

```
PostDocumentSerializer.validate_custom_fields()  ✅ 校验（仅一次，有两处边界）
  ├─ dict 格式：CustomFieldInstanceSerializer.validate() 完整类型校验
  │   └─ DocumentLink：只校验目标存在，不校验用户权限（context 为空）
  └─ list 格式：仅校验 id 存在性，重复 id 无法检测（set() 先去重）
  ↓ 值封装进 DocumentMetadataOverrides
  ↓ Celery 异步任务 consume_file()
  ↓ WorkflowTriggerPlugin 可能追加工作流值  ⚠️ 完全不校验
  ↓ ConsumerPlugin.apply_overrides()  ❌ 不再校验，直接 create()
  ↓ Document.save()
  ↓ document_consumption_finished → add_to_index ✅ 写入搜索索引
```

**设计意图**：校验只在 API 入口做一次，后续各环节（Celery、插件链、落库）信任上游数据，追求性能。

**风险点汇总（三处校验不完整）**：

| 风险 | 位置 | 说明 |
|------|------|------|
| DocumentLink 未校验权限 | PostDocumentSerializer（dict 格式） | 手动实例化 `CustomFieldInstanceSerializer()` 未传 context，`request` 为 None → `validate_documentlink_targets` 中 `user is None` 直接 return，跳过 `has_perms_owner_aware` 权限检查。PUT/PATCH 更新路径无此问题（嵌套序列化器自动传递 context） |
| list 格式重复 id 漏检 | PostDocumentSerializer（list 格式） | 判断条件 `DB.count() != len(set(ids))` 中 `set()` 已去重，重复 id 无法触发错误，后续由 `dict.fromkeys()` 静默合并 |
| 工作流注入值不校验 | WorkflowTriggerPlugin → apply_assignment_to_overrides | 管理员配置的工作流值直接写入 overrides dict，无任何类型校验 |

#### 关系二：落库与搜索索引的时序

CustomFieldInstance 的写入发生在 `document.save()` **之前**（`apply_overrides` → CustomFieldInstance.create() → document.save()）。

搜索索引的写入发生在 `document_consumption_finished` 信号触发时，此时：
- Document 和 CustomFieldInstance 都已写入数据库
- 事务已提交
- 搜索后端遍历 `document.custom_fields.all()`，此时能读到刚写入的实例

因此正常流程下不存在"索引写入时实例还没创建"的竞态问题。

#### 关系三：document_importer 信号禁用与索引重建的配合

document_importer 禁用信号的目的是避免导入大量数据时反复触发文件名重算、SELECT 清理等副作用，造成巨大性能开销。

导入结束后立即调用 `document_index reindex` 重建索引，弥补了信号禁用导致的索引缺失。这是一个"先批量写数据库，最后统一建索引"的高效模式：

```
with disable_signal(...):           # 禁用 4 个 post_save/m2m_changed + auditlog
    load_data_to_database()         # 批量 upsert，无任何信号触发
    _import_files_from_manifest()   # 复制文件

# 上下文退出，信号恢复
call_command("document_index", "reindex")  # 统一重建搜索索引
```

**与 Custom Fields 的关系**：
- 导入过程中 CustomFieldInstance 的批量写入不会触发 `update_filename_and_move_files`
- 导入结束后 `document_index reindex` 会遍历所有文档，将 custom_fields 写入 Tantivy 索引
- 如果文件名模板使用了 custom_fields，导入后的文件名可能与模板不一致（因为信号被禁用，文件名重算未执行），需手动触发

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
| Celery 任务调度（consume_file / bulk_update_documents） | [src/documents/tasks.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/tasks.py) |
| 工作流分配/移除 mutations | [src/documents/workflows/mutations.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/workflows/mutations.py) |
| 工作流执行上下文/actions | [src/documents/workflows/actions.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/workflows/actions.py) |
| 工作流工具（Prefetch/annotate） | [src/documents/workflows/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/workflows/utils.py) |
| 信号处理（工作流、索引、文件名） | [src/documents/signals/handlers.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/signals/handlers.py) |
| **信号连接配置**（apps.ready） | [src/documents/apps.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/apps.py) |
| **Websocket StatusManager** | [src/documents/plugins/helpers.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/documents/plugins/helpers.py) |
| **API URL 路由注册** | [src/paperless/urls.py](file:///d:/fz/0601/solo-dogfeeding/code/60-paperless-ngx/src/paperless/urls.py) |
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
