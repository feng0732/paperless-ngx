# 前端列表状态与后端参数协同机制

本文档详细说明 paperless-ngx 中文档列表的筛选状态、请求参数、响应落地和页面同步的完整数据流。

## 整体架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                        前端 (Angular)                               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │
│  │  用户交互   │───▶│  筛选状态   │───▶│  URL参数    │             │
│  │ (FilterEditor)│   │ (ListViewState)│   │ (queryParams)│         │
│  └─────────────┘    └─────────────┘    └──────┬──────┘             │
│                                                │                    │
│                                                ▼                    │
│                                        ┌─────────────┐             │
│                                        │ API请求构建 │             │
│                                        │ (listFiltered)│            │
│                                        └──────┬──────┘             │
└───────────────────────────────────────────────┼────────────────────┘
                                                │
┌───────────────────────────────────────────────┼────────────────────┐
│                        后端 (Django DRF)                           │
│                                                │                    │
│                                                ▼                    │
│                                        ┌─────────────┐             │
│                                        │  参数解析   │             │
│                                        │ (FilterSet) │             │
│                                        └──────┬──────┘             │
│                                                │                    │
│                                                ▼                    │
│                                        ┌─────────────┐             │
│                                        │  搜索执行   │             │
│                                        │ (Tantivy/ORM)│            │
│                                        └──────┬──────┘             │
│                                                │                    │
│                                                ▼                    │
│                                        ┌─────────────┐             │
│                                        │  响应序列化 │             │
│                                        │ (Serializer)│             │
│                                        └──────┬──────┘             │
└───────────────────────────────────────────────┼────────────────────┘
                                                │
┌───────────────────────────────────────────────┼────────────────────┐
│                        前端 (Angular)                               │
│                                                ▼                    │
│                                        ┌─────────────┐             │
│                                        │  响应落地   │             │
│                                        │ (reload())  │             │
│                                        └──────┬──────┘             │
│                                                │                    │
│                                                ▼                    │
│                                        ┌─────────────┐             │
│                                        │  页面同步   │             │
│                                        │ (ListViewState)│          │
│                                        └─────────────┘             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 第一部分：前端筛选状态管理

### 1.1 核心状态定义：ListViewState

**文件**：[document-list-view.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/services/document-list-view.service.ts#L45-L102)

```typescript
export interface ListViewState {
  title?: string                    // 视图标题
  documents?: Document[]            // 当前页文档列表
  currentPage: number               // 当前页码
  collectionSize?: number           // 总文档数
  sortField: string                 // 排序字段
  sortReverse: boolean              // 是否倒序
  filterRules: FilterRule[]         // 筛选规则数组
  selected?: Set<number>            // 选中的文档ID
  allSelected?: boolean             // 是否全选
  pageSize?: number                 // 每页大小
  displayMode?: DisplayMode         // 显示模式
  displayFields?: DisplayField[]    // 显示字段
}
```

### 1.2 筛选规则数据结构

**文件**：[filter-rule.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/data/filter-rule.ts#L1-L4)

```typescript
export interface FilterRule {
  rule_type: number    // 筛选规则类型ID
  value: string        // 筛选值
}
```

### 1.3 筛选规则类型定义

**文件**：[filter-rule-type.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/data/filter-rule-type.ts#L74-L385)

每个筛选规则类型定义了前端规则到后端参数的映射：

```typescript
export const FILTER_RULE_TYPES: FilterRuleType[] = [
  {
    id: FILTER_TITLE,               // 类型ID: 0
    filtervar: 'title__icontains',  // 后端查询参数名
    datatype: 'string',             // 数据类型
    multi: false,                   // 是否支持多值
    default: '',
  },
  {
    id: FILTER_CORRESPONDENT,       // 类型ID: 3
    filtervar: 'correspondent__id', // 后端查询参数名
    isnull_filtervar: 'correspondent__isnull', // 空值检查参数
    datatype: DataType.Correspondent,
    multi: false,
  },
  {
    id: FILTER_HAS_TAGS_ANY,        // 类型ID: 22
    filtervar: 'tags__id__in',      // 后端查询参数名
    datatype: DataType.Tag,
    multi: true,                    // 支持多值，用逗号分隔
  },
  // ... 约50种筛选规则类型
]
```

### 1.4 状态管理服务：DocumentListViewService

**文件**：[document-list-view.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/services/document-list-view.service.ts#L113-L722)

这是前端列表状态的核心管理服务，使用 `Map<number, ListViewState>` 维护多个视图的状态（普通视图和保存视图）。

#### 状态初始化

```typescript
private defaultListViewState(): ListViewState {
  return {
    title: null,
    documents: [],
    currentPage: 1,
    collectionSize: null,
    sortField: 'created',
    sortReverse: true,
    filterRules: [],
    selected: new Set<number>(),
    allSelected: false,
  }
}
```

#### 状态更新触发点

| 操作 | 方法 | 状态变化 |
|------|------|----------|
| 设置筛选规则 | `setFilterRules(filterRules, resetPage)` | 更新 `filterRules`，可选重置 `currentPage` |
| 设置排序 | `setSort(field, reverse)` | 更新 `sortField` 和 `sortReverse` |
| 切换页码 | `currentPage = page` | 更新 `currentPage` |
| 切换每页大小 | `pageSize = size` | 更新 `pageSize` |

---

## 第二部分：请求参数构建

### 2.1 筛选规则转URL参数

**文件**：[query-params.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/utils/query-params.ts#L151-L188)

核心转换函数 `queryParamsFromFilterRules()`：

```typescript
export function queryParamsFromFilterRules(filterRules: FilterRule[]): Params {
  if (filterRules) {
    let params = {}
    for (let rule of filterRules) {
      let ruleType = FILTER_RULE_TYPES.find((t) => t.id == rule.rule_type)
      
      // 特殊处理：全文搜索简化参数
      if (rule.rule_type === FILTER_TITLE_CONTENT || rule.rule_type === FILTER_SIMPLE_TEXT) {
        params['text'] = rule.value
      } else if (rule.rule_type === FILTER_TITLE || rule.rule_type === FILTER_SIMPLE_TITLE) {
        params['title_search'] = rule.value
      }
      // 空值筛选
      else if (ruleType.isnull_filtervar && rule.value == null) {
        params[ruleType.isnull_filtervar] = 1
      }
      // 空值否定筛选
      else if (ruleType.isnull_filtervar && rule.value == NEGATIVE_NULL_FILTER_VALUE.toString()) {
        params[ruleType.isnull_filtervar] = 0
      }
      // 多值筛选（逗号分隔）
      else if (ruleType.multi) {
        params[ruleType.filtervar] = params[ruleType.filtervar]
          ? params[ruleType.filtervar] + ',' + rule.value
          : rule.value
      }
      // 普通筛选
      else {
        params[ruleType.filtervar] = rule.value
        if (ruleType.datatype == 'boolean')
          params[ruleType.filtervar] = rule.value == 'true' || rule.value == '1' ? 1 : 0
      }
    }
    return params
  }
  return null
}
```

### 2.2 完整视图状态转URL参数

**文件**：[query-params.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/utils/query-params.ts#L27-L40)

```typescript
export function paramsFromViewState(viewState: ListViewState, pageOnly: boolean = false): Params {
  let params = queryParamsFromFilterRules(viewState.filterRules)
  params['sort'] = viewState.sortField
  params['reverse'] = viewState.sortReverse ? 1 : undefined
  if (pageOnly) params = {}
  params['page'] = isNaN(viewState.currentPage) ? 1 : viewState.currentPage
  if (pageOnly && viewState.currentPage == 1) params['page'] = undefined
  return params
}
```

### 2.3 API请求构建

**文件**：[abstract-paperless-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/services/rest/abstract-paperless-service.ts#L44-L77)

```typescript
list(page?: number, pageSize?: number, sortField?: string, sortReverse?: boolean, extraParams?): Observable<Results<T>> {
  let httpParams = new HttpParams()
  
  // 分页参数
  if (page) httpParams = httpParams.set('page', page.toString())
  if (pageSize) httpParams = httpParams.set('page_size', pageSize.toString())
  
  // 排序参数：前缀 "-" 表示倒序
  let ordering = sortField ? (sortReverse ? '-' : '') + sortField : null
  if (ordering) httpParams = httpParams.set('ordering', ordering)
  
  // 筛选参数
  for (let extraParamKey in extraParams) {
    if (extraParams[extraParamKey] != null) {
      httpParams = httpParams.set(extraParamKey, extraParams[extraParamKey])
    }
  }
  
  return this.http.get<Results<T>>(this.getResourceUrl(), { params: httpParams })
}
```

**文件**：[document.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/services/rest/document.service.ts#L173-L188)

```typescript
listFiltered(page?: number, pageSize?: number, sortField?: string, sortReverse?: boolean, 
             filterRules?: FilterRule[], extraParams = {}): Observable<Results<Document>> {
  return this.list(
    page,
    pageSize,
    sortField,
    sortReverse,
    // 合并筛选规则转换的参数和额外参数
    Object.assign(extraParams, queryParamsFromFilterRules(filterRules))
  )
}
```

### 2.4 参数转换示例

| 前端筛选规则 | 转换后的HTTP参数 | 说明 |
|-------------|------------------|------|
| `{rule_type: 3, value: "5"}` | `correspondent__id=5` | 通讯员ID等于5 |
| `{rule_type: 22, value: "1"}, {rule_type: 22, value: "2"}` | `tags__id__in=1,2` | 包含标签1或2 |
| `{rule_type: 3, value: null}` | `correspondent__isnull=1` | 通讯员为空 |
| `{rule_type: 49, value: "invoice"}` | `text=invoice` | 全文搜索"invoice" |
| `{rule_type: 20, value: "tag:urgent"}` | `query=tag:urgent` | 高级搜索查询 |

---

## 第三部分：后端参数接收与处理

### 3.1 后端筛选器定义

**文件**：[filters.py](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src/documents/filters.py#L776-L867)

```python
class DocumentFilterSet(FilterSet):
    # 自定义筛选器
    is_tagged = BooleanFilter(field_name="tags", lookup_expr="isnull", exclude=True)
    tags__id__all = ObjectFilter(field_name="tags")
    tags__id__none = ObjectFilter(field_name="tags", exclude=True)
    tags__id__in = ObjectFilter(field_name="tags", in_list=True)
    is_in_inbox = InboxFilter()
    custom_field_query = CustomFieldQueryFilter("custom_field_query")
    
    class Meta:
        model = Document
        fields = {
            "id": ["in", "exact"],
            "title": ["istartswith", "iendswith", "icontains", "iexact"],
            "archive_serial_number": ["exact", "gt", "gte", "lt", "lte", "isnull"],
            "created": ["year", "month", "day", "gt", "gte", "lt", "lte"],
            "correspondent__id": ["in", "exact"],
            "correspondent__name": ["istartswith", "iendswith", "icontains", "iexact"],
            # ... 更多字段
        }
```

### 3.2 两种查询模式

**文件**：[views.py](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src/documents/views.py#L2233-L2451)

后端根据查询参数判断使用哪种搜索模式：

```python
_TANTIVY_SEARCH_PARAM_NAMES = ("text", "title_search", "query", "more_like_id")

def _is_search_request(self):
    return bool(self._get_active_search_params())

def list(self, request, *args, **kwargs):
    if not self._is_search_request():
        # 模式1：普通ORM筛选
        return super().list(request)
    else:
        # 模式2：Tantivy全文搜索
        return self._search_list(request)
```

#### 模式1：ORM筛选（非搜索请求）

当查询参数中不包含 `text`、`title_search`、`query`、`more_like_id` 时，使用标准Django ORM筛选：

```python
# DocumentViewSet.list()
def list(self, request, *args, **kwargs):
    if not get_boolean(str(request.query_params.get("include_selection_data", "false"))):
        return super().list(request, *args, **kwargs)
    
    # 应用所有筛选器
    queryset = self.filter_queryset(self.get_queryset())
    
    # 计算选中数据统计
    selection_data = self._get_selection_data_for_queryset(queryset)
    
    # 分页
    page = self.paginate_queryset(queryset)
    serializer = self.get_serializer(page, many=True)
    response = self.get_paginated_response(serializer.data)
    response.data["selection_data"] = selection_data
    return response
```

#### 模式2：Tantivy全文搜索

当包含搜索参数时，使用Tantivy搜索引擎：

```python
def run_text_search(backend, user, filtered_qs):
    # 1. 解析查询模式
    query_str, search_mode = _get_tantivy_query_and_mode(request.query_params)
    # search_mode: TEXT / TITLE / QUERY
    
    # 2. Tantivy搜索获取匹配的文档ID
    all_ids = backend.search_ids(
        query_str,
        user=user,
        sort_field=sort_field_name,
        sort_reverse=sort_reverse,
        search_mode=search_mode,
    )
    
    # 3. 与ORM筛选结果取交集（权限过滤等）
    ordered_ids = intersect_and_order(all_ids, filtered_qs, use_tantivy_sort=use_tantivy_sort)
    
    # 4. 分页 + 高亮
    page_offset = (page_num - 1) * page_size
    page_ids = ordered_ids[page_offset : page_offset + page_size]
    page_hits = backend.highlight_hits(query_str, page_ids, search_mode=search_mode)
    
    return SearchResultPage(ordered_ids=ordered_ids, hits=page_hits, page_offset=page_offset)
```

### 3.3 查询预处理

**文件**：[_query.py](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src/documents/search/_query.py#L339-L411)

查询在交给Tantivy之前会经过多轮预处理：

```python
def parse_user_query(index, raw_query, tz):
    # 1. 日期关键词重写
    query_str = rewrite_natural_date_keywords(raw_query, tz)
    # "created:today" → "created:[2024-01-15T00:00:00Z TO 2024-01-16T00:00:00Z]"
    
    # 2. 查询规范化
    query_str = normalize_query(query_str)
    # "tag:foo,bar" → "tag:foo AND tag:bar"
    
    # 3. Tantivy解析 + 字段权重
    exact = index.parse_query(query_str, DEFAULT_SEARCH_FIELDS, field_boosts={"title": 2.0})
    
    # 4. 可选：模糊搜索混合
    if threshold is not None:
        fuzzy = index.parse_query(query_str, DEFAULT_SEARCH_FIELDS, 
                                 fuzzy_fields={f: (True, 1, True) for f in DEFAULT_SEARCH_FIELDS})
        return boolean_query([
            (Should, exact),
            (Should, boost_query(fuzzy, 0.1)),  # 模糊匹配权重更低
        ])
    
    return exact
```

### 3.4 权限过滤

**文件**：[_query.py](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src/documents/search/_query.py#L413-L442)

```python
def build_permission_filter(schema, user):
    # 可见文档条件：
    # 1. 无所有者（公开文档）
    # 2. 当前用户是所有者
    # 3. 当前用户被明确授权
    
    owner_any = Query.exists_query("owner_id")
    no_owner = boolean_query([
        (Must, Query.all_query()),
        (MustNot, owner_any),
    ])
    owned = Query.term_query(schema, "owner_id", user.pk)
    shared = Query.term_query(schema, "viewer_id", user.pk)
    
    return Query.disjunction_max_query([no_owner, owned, shared])
```

---

## 第四部分：响应数据结构

### 4.1 标准响应格式

**文件**：[results.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/data/results.ts#L1-L26)

```typescript
export interface Results<T> {
  count: number              // 总结果数
  display_count?: number     // 显示数量（用于树形结构）
  results: T[]               // 当前页结果
}

export interface DocumentResults extends Results<Document> {
  selection_data?: SelectionData  // 筛选结果统计
}

export interface SelectionData {
  selected_storage_paths: SelectionDataItem[]
  selected_correspondents: SelectionDataItem[]
  selected_tags: SelectionDataItem[]
  selected_document_types: SelectionDataItem[]
  selected_custom_fields: SelectionDataItem[]
}

export interface SelectionDataItem {
  id: number
  document_count: number
}
```

### 4.2 搜索结果额外字段

**文件**：[_backend.py](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src/documents/search/_backend.py#L83-L90)

```python
class SearchHit(TypedDict):
    id: int                    # 文档ID
    score: float               # 匹配分数（0-1，归一化后）
    rank: int                  # 排名（1-based）
    highlights: dict[str, str] # 高亮片段，如 {"content": "...<b>keyword</b>..."}
```

---

## 第五部分：响应落地与页面同步

### 5.1 核心重载方法：reload()

**文件**：[document-list-view.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/services/document-list-view.service.ts#L304-L387)

```typescript
reload(onFinish?, updateQueryParams: boolean = true) {
  this.cancelPending()
  this.isReloading = true
  this.error = null
  
  let activeListViewState = this.activeListViewState
  
  // 1. 发起API请求
  this.documentService
    .listFiltered(
      activeListViewState.currentPage,
      activeListViewState.pageSize ?? this.pageSize,
      activeListViewState.sortField,
      activeListViewState.sortReverse,
      activeListViewState.filterRules,
      { truncate_content: true, include_selection_data: true }
    )
    .pipe(takeUntil(this.unsubscribeNotifier))
    .subscribe({
      next: (result) => {
        const resultWithSelectionData = result as DocumentResults
        
        // 2. 响应落地：更新状态
        this.initialized = true
        this.isReloading = false
        activeListViewState.collectionSize = result.count
        activeListViewState.documents = result.results
        this.selectionData = resultWithSelectionData.selection_data ?? null
        
        // 3. 同步选中状态到当前页
        this.syncSelectedToCurrentPage()
        
        // 4. URL同步
        if (updateQueryParams && !this._activeSavedViewId) {
          // 普通视图：更新完整URL参数
          this.router.navigate(['/documents'], {
            queryParams: paramsFromViewState(activeListViewState),
            replaceUrl: !this.router.routerState.snapshot.url.includes('?'),
          })
        } else if (this._activeSavedViewId) {
          // 保存视图：仅更新page参数
          this.router.navigate([], {
            queryParams: paramsFromViewState(activeListViewState, true),
            queryParamsHandling: 'merge',
          })
        }
        
        if (onFinish) onFinish()
      },
      error: (error) => {
        this.isReloading = false
        // 错误处理：页码越界自动回退到第1页
        if (activeListViewState.currentPage != 1 && error.status == 404) {
          activeListViewState.currentPage = 1
          this.reload()
        }
        // ... 其他错误处理
      },
    })
}
```

### 5.2 选中状态同步

**文件**：[document-list-view.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/services/document-list-view.service.ts#L211-L222)

```typescript
private syncSelectedToCurrentPage() {
  if (!this.allSelected) {
    return
  }
  
  // 全选模式下，同步选中集合为当前页的文档ID
  this.selected.clear()
  this.documents?.forEach((doc) => this.selected.add(doc.id))
  
  if (!this.collectionSize) {
    this.selectNone()
  }
}
```

### 5.3 本地存储持久化

**文件**：[document-list-view.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/services/document-list-view.service.ts#L523-L539)

```typescript
private saveDocumentListView() {
  if (this._activeSavedViewId == null) {
    // 仅保存普通视图的状态到localStorage
    let savedState: ListViewState = {
      collectionSize: this.activeListViewState.collectionSize,
      currentPage: this.activeListViewState.currentPage,
      filterRules: this.activeListViewState.filterRules,
      sortField: this.activeListViewState.sortField,
      sortReverse: this.activeListViewState.sortReverse,
      displayMode: this.activeListViewState.displayMode,
      displayFields: this.activeListViewState.displayFields,
    }
    localStorage.setItem(
      DOCUMENT_LIST_SERVICE.CURRENT_VIEW_CONFIG,
      JSON.stringify(savedState)
    )
  }
}
```

### 5.4 URL参数反向同步到状态

**文件**：[query-params.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/utils/query-params.ts#L42-L58)

```typescript
export function paramsToViewState(queryParams: ParamMap): ListViewState {
  let filterRules = filterRulesFromQueryParams(queryParams)
  let sortField = queryParams.get('sort')
  let sortReverse = queryParams.has('reverse') || 
    (!queryParams.has('sort') && !queryParams.has('reverse'))
  let currentPage = queryParams.has('page') ? parseInt(queryParams.get('page')) : 1
  
  return {
    currentPage,
    filterRules,
    sortField,
    sortReverse,
  }
}
```

### 5.5 完整数据流时序

```
用户操作 (点击筛选按钮)
    ↓
FilterEditorComponent.onFilterRulesChange(rules)
    ↓
DocumentListViewService.setFilterRules(rules)
    ├─ 更新 activeListViewState.filterRules
    ├─ 重置 currentPage = 1 (可选)
    └─ 调用 reload()
        ├─ 取消未完成请求
        ├─ 设置 isReloading = true
        ├─ 调用 documentService.listFiltered()
        │   ├─ queryParamsFromFilterRules(rules) → {tags__id__in: "1,2", ...}
        │   ├─ 构建 HttpParams: page, page_size, ordering, ...
        │   └─ GET /api/documents/?tags__id__in=1,2&ordering=-created&page=1
        │
        └─ 响应返回: {count: 42, results: [...], selection_data: {...}}
            ├─ 更新 collectionSize = 42
            ├─ 更新 documents = results
            ├─ 更新 selection_data
            ├─ syncSelectedToCurrentPage()
            ├─ 同步URL: /documents?tags__id__in=1,2&sort=created&reverse=1&page=1
            └─ 触发 Angular 变更检测 → 页面自动刷新
```

---

## 第六部分：关键对齐点

### 6.1 前端筛选规则 ↔ 后端参数映射表

| 前端 rule_type | 前端 filtervar | 后端 FilterSet 字段 | 说明 |
|---------------|----------------|---------------------|------|
| 3 (FILTER_CORRESPONDENT) | `correspondent__id` | `correspondent__id` | 通讯员精确匹配 |
| 22 (FILTER_HAS_TAGS_ANY) | `tags__id__in` | `tags__id__in` | 包含任一标签 |
| 6 (FILTER_HAS_TAGS_ALL) | `tags__id__all` | `tags__id__all` | 包含所有标签 |
| 27 (FILTER_DOES_NOT_HAVE_CORRESPONDENT) | `correspondent__id__none` | `correspondent__id__none` | 排除通讯员 |
| 49 (FILTER_SIMPLE_TEXT) | `text` | 无（Tantivy） | 简单全文搜索 |
| 20 (FILTER_FULLTEXT_QUERY) | `query` | 无（Tantivy） | 高级搜索查询 |
| 21 (FILTER_FULLTEXT_MORELIKE) | `more_like_id` | 无（Tantivy） | 相似文档 |

### 6.2 排序字段映射

| 前端 sortField | 后端 ordering 参数 | Tantivy 排序字段 |
|----------------|-------------------|------------------|
| `created` | `-created` | `created` |
| `correspondent__name` | `correspondent__name` | `correspondent_sort` |
| `score` | `score` | （无，按相关性） |
| `custom_field_5` | `-custom_field_5` | （ORM子查询注解） |

### 6.3 特殊值约定

| 前端 value | 后端参数值 | 含义 |
|-----------|-----------|------|
| `null` | `field__isnull=1` | 字段为空 |
| `"-1"` (NEGATIVE_NULL_FILTER_VALUE) | `field__isnull=0` | 字段不为空 |
| `"true"` / `"1"` | `1` | 布尔值真 |
| `"false"` / `"0"` | `0` | 布尔值假 |
| 多值规则重复 | `"1,2,3"` | 多值用逗号分隔 |

---

## 第七部分：常见问题排查

### 7.1 筛选不生效？

1. **检查参数转换**：在 `queryParamsFromFilterRules()` 打断点，确认规则正确转换为参数
2. **检查后端接收**：在 `DocumentFilterSet` 中确认参数字段存在
3. **检查搜索模式**：如果是 `text`/`query` 参数，会走Tantivy分支而非ORM

### 7.2 排序不一致？

1. **文本字段排序**：`correspondent__name` 等文本字段在Tantivy和ORM中排序可能不同
2. **自定义字段排序**：通过 `DocumentsOrderingFilter` 动态注解实现，性能较差
3. **相关性排序**：`score` 排序仅在Tantivy搜索时有效

### 7.3 页码跳转异常？

1. **筛选后页码越界**：`reload()` 中有自动回退到第1页的逻辑
2. **保存视图分页**：保存视图仅同步 `page` 参数，其他筛选参数不同步到URL

### 7.4 选中状态丢失？

1. **筛选变更**：`reduceSelectionToFilter()` 会移除不再可见的选中项
2. **全选模式**：`syncSelectedToCurrentPage()` 会将全选同步为当前页选中
