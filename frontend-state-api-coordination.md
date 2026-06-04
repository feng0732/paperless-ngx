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

def _get_active_search_params(self, request):
    return [param for param in _TANTIVY_SEARCH_PARAM_NAMES if param in request.query_params]

def _is_search_request(self):
    return bool(self._get_active_search_params())

def list(self, request, *args, **kwargs):
    if not self._is_search_request():
        # 模式1：普通ORM筛选
        return super().list(request)
    else:
        # 模式2：Tantivy全文搜索
        # 注意：多个搜索参数互斥，len(active) > 1 会抛出 ValidationError
        if "more_like_id" in request.query_params:
            # 子分支A：more_like_this 独立处理
            return run_more_like_this(backend, user, filtered_qs)
        else:
            # 子分支B：text/title_search/query 走 _get_tantivy_query_and_mode
            return run_text_search(backend, user, filtered_qs)
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

| 前端 rule_type | 定义 filtervar | 实际HTTP参数 | 后端接收方式 | 说明 |
|---------------|----------------|-------------|------------|------|
| 3 (FILTER_CORRESPONDENT) | `correspondent__id` | `correspondent__id` | FilterSet `exact` | 通讯员精确匹配 |
| 22 (FILTER_HAS_TAGS_ANY) | `tags__id__in` | `tags__id__in` | `ObjectFilter(in_list=True)` | 包含任一标签 |
| 6 (FILTER_HAS_TAGS_ALL) | `tags__id__all` | `tags__id__all` | `ObjectFilter`（逐个AND） | 包含所有标签 |
| 27 (FILTER_DOES_NOT_HAVE_CORRESPONDENT) | `correspondent__id__none` | `correspondent__id__none` | `ObjectFilter(exclude=True)` | 排除通讯员 |
| 0 (FILTER_TITLE) | `title__icontains` | `title_search` | Tantivy `SearchMode.TITLE` | 旧版规则，参数转换时被重写 |
| 48 (FILTER_SIMPLE_TITLE) | `title_search` | `title_search` | Tantivy `SearchMode.TITLE` | 新版标题搜索 |
| 19 (FILTER_TITLE_CONTENT) | `title_content` | `text` | Tantivy `SearchMode.TEXT` | 旧版规则，参数转换时被重写 |
| 49 (FILTER_SIMPLE_TEXT) | `text` | `text` | Tantivy `SearchMode.TEXT` | 新版全文搜索 |
| 20 (FILTER_FULLTEXT_QUERY) | `query` | `query` | Tantivy `SearchMode.QUERY` | 高级搜索语法 |
| 21 (FILTER_FULLTEXT_MORELIKE) | `more_like_id` | `more_like_id` | `run_more_like_this` 分支 | 相似文档，独立处理路径 |
| 38 (FILTER_HAS_CUSTOM_FIELDS_ALL) | `custom_fields__id__all` | 前端转42 | 后端保留 `ObjectFilter` | 前端自动转为42 |
| 39 (FILTER_HAS_CUSTOM_FIELDS_ANY) | `custom_fields__id__in` | 前端转42 | 后端保留 `ObjectFilter(in_list=True)` | 前端自动转为42 |
| 42 (FILTER_CUSTOM_FIELDS_QUERY) | `custom_field_query` | `custom_field_query` | `CustomFieldQueryFilter` | JSON表达式 |

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

---

## 第八部分：视图分页回写机制详解

本部分深入分析 `reload()` 响应成功后，分页状态如何回写到 URL，以及普通视图与保存视图的行为差异。

### 8.1 reload() 中的 URL 回写分支

**文件**：[document-list-view.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/services/document-list-view.service.ts#L329-L340)

```typescript
// 响应成功后的URL回写逻辑
if (updateQueryParams && !this._activeSavedViewId) {
  // 分支A：普通视图 → 回写全部参数到 /documents
  let base = ['/documents']
  this.router.navigate(base, {
    queryParams: paramsFromViewState(activeListViewState),
    replaceUrl: !this.router.routerState.snapshot.url.includes('?'),
  })
} else if (this._activeSavedViewId) {
  // 分支B：保存视图 → 仅回写 page 参数到 /view/:id
  this.router.navigate([], {
    queryParams: paramsFromViewState(activeListViewState, true), // pageOnly=true
    queryParamsHandling: 'merge',
  })
}
```

### 8.2 两种回写模式对比

| 维度 | 普通视图（/documents） | 保存视图（/view/:id） |
|------|----------------------|---------------------|
| **回写范围** | 全部参数（筛选+排序+分页） | 仅 page 参数 |
| **路由目标** | `/documents` | 当前路由（`[]`表示不变） |
| **合并策略** | 覆盖所有 queryParams | `merge`（仅更新page） |
| **replaceUrl** | 首次无参数时replace，后续push | 无（默认push） |
| **筛选状态来源** | URL参数 | SavedView 后端数据 |

### 8.3 pageOnly 参数的影响

**文件**：[query-params.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/utils/query-params.ts#L27-L40)

```typescript
export function paramsFromViewState(viewState: ListViewState, pageOnly: boolean = false): Params {
  let params = queryParamsFromFilterRules(viewState.filterRules)
  params['sort'] = viewState.sortField
  params['reverse'] = viewState.sortReverse ? 1 : undefined

  if (pageOnly) params = {}          // ← pageOnly=true 时清空筛选+排序参数

  params['page'] = isNaN(viewState.currentPage) ? 1 : viewState.currentPage
  if (pageOnly && viewState.currentPage == 1) params['page'] = undefined  // ← 第1页时不输出page
  return params
}
```

**关键细节**：
- `pageOnly=true` 时，先按正常逻辑构建全部参数，然后**清空**，只保留 `page`
- 当 `currentPage == 1` 且 `pageOnly=true` 时，`page` 也设为 `undefined`，即**第1页时URL上不出现page参数**
- 配合 `queryParamsHandling: 'merge'`，保存视图翻到第1页时URL中的 `page` 参数会被移除

### 8.4 页码越界自动回退

**文件**：[document-list-view.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/services/document-list-view.service.ts#L347-L354)

```typescript
error: (error) => {
  this.isReloading = false
  if (activeListViewState.currentPage != 1 && error.status == 404) {
    // 筛选后结果减少，当前页已不存在，自动回退到第1页
    activeListViewState.currentPage = 1
    this.reload()  // 递归重载，不传updateQueryParams（默认true）
  }
  // ...
}
```

**回退场景**：用户在第5页时新增筛选条件，导致总结果不足5页 → 后端返回404 → 自动跳回第1页重新请求。

### 8.5 保存视图的完整加载流程

**文件**：[document-list-view.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/services/document-list-view.service.ts#L238-L274)

```
用户点击保存视图
    ↓
activateSavedView(view)
    ├─ 设置 _activeSavedViewId = view.id
    └─ loadSavedView(view)
        ├─ filterRules ← view.filter_rules (从后端数据)
        ├─ sortField ← view.sort_field
        ├─ sortReverse ← view.sort_reverse
        ├─ title ← view.name
        ├─ displayMode ← view.display_mode
        ├─ pageSize ← view.page_size
        ├─ displayFields ← view.display_fields
        ├─ reduceSelectionToFilter()
        └─ router.navigate(['view', view.id])  ← 路由跳转
```

**保存视图加载时**，筛选规则**不从URL读取**，而是直接使用 `SavedView.filter_rules`。URL中的 `page` 参数在跳转后由 `activateSavedViewWithQueryParams` 单独处理：

```typescript
activateSavedViewWithQueryParams(view: SavedView, queryParams: ParamMap) {
  const viewState = paramsToViewState(queryParams) // 从URL只取page
  this.activateSavedView(view)
  this.currentPage = viewState.currentPage  // 只覆盖页码
}
```

---

## 第九部分：地址反向同步入口详解

本部分详细追踪从 URL 变化到状态更新的完整入口链路。

### 9.1 两个路由订阅源

**文件**：[document-list.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/components/document-list/document-list.component.ts#L274-L321)

DocumentListComponent 的 `ngOnInit` 中订阅了两个路由事件流：

```typescript
// 订阅源1：路径参数（保存视图）—— /view/:id
this.route.paramMap.pipe(
  filter((params) => params.has('id')),  // 仅在 /view/id 路由触发
  switchMap((params) => {
    return this.savedViewService.getCached(+params.get('id'))
      .pipe(map((view) => ({ view })))
  })
).subscribe(({ view }) => {
  this.activeSavedView = view
  this.list.activateSavedViewWithQueryParams(
    view,
    convertToParamMap(this.route.snapshot.queryParams)  // 从当前URL取page
  )
  this.list.reload()  // 发起API请求
})

// 订阅源2：查询参数（普通视图）—— /documents?sort=created&...
this.route.queryParamMap.pipe(
  filter(() => !this.route.snapshot.paramMap.has('id')),  // 排除保存视图
).subscribe((queryParams) => {
  if (queryParams.has('view')) {
    // URL中带 view=123 → 加载对应保存视图
    this.loadViewConfig(parseInt(queryParams.get('view')))
  } else {
    this.activeSavedView = null
    this.list.activateSavedView(null)
    this.list.loadFromQueryParams(queryParams)  // 从URL恢复状态
  }
})
```

### 9.2 反向同步入口流程图

```
URL变化（用户输入/浏览器前进后退/代码导航）
    ↓
Angular Router 解析
    ├─ 路径含 /view/:id?
    │   ├─ 是 → route.paramMap 订阅触发
    │   │   ├─ getCached(viewId) → 获取SavedView对象
    │   │   ├─ activateSavedViewWithQueryParams(view, queryParams)
    │   │   │   ├─ loadSavedView(view) → filterRules/sort/pageSize 从后端数据填充
    │   │   │   └─ currentPage = 从URL的page参数读取
    │   │   └─ reload() → 发起API请求
    │   │
    │   └─ 否 → route.queryParamMap 订阅触发
    │       ├─ URL带 view=xxx? → loadViewConfig(id)
    │       │   └─ 加载保存视图配置
    │       └─ 无view参数 → loadFromQueryParams(queryParams)
    │           ├─ paramsToViewState(queryParams) → 解析全部状态
    │           │   ├─ filterRulesFromQueryParams()
    │           │   │   ├─ 遍历 FILTER_RULE_TYPES 的 filtervar/isnull_filtervar
    │           │   │   │   └─ 匹配 URL 中存在的参数 → 反查 rule_type.id
    │           │   │   ├─ 多值参数按逗号拆分 → 生成多条 FilterRule
    │           │   │   ├─ 布尔值 "1"→"true" / "0"→"false"
    │           │   │   └─ isnull 参数: "1"→value=null / "0"→value="-1"
    │           │   ├─ sortField ← URL的sort参数
    │           │   ├─ sortReverse ← URL有reverse || 无sort也无reverse(默认true)
    │           │   └─ currentPage ← URL的page参数
    │           └─ 与当前状态比较（filterRulesDiffer）
    │               ├─ 有变化 → 更新状态 + reload()
    │               └─ 无变化 → 跳过（避免循环）
```

### 9.3 URL → FilterRule 的反查逻辑

**文件**：[query-params.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/utils/query-params.ts#L103-L149)

```typescript
export function filterRulesFromQueryParams(queryParams: ParamMap): FilterRule[] {
  // 1. 收集所有可能的筛选参数名（filtervar + isnull_filtervar）
  const allFilterRuleQueryParams: string[] = FILTER_RULE_TYPES
    .map((rt) => rt.filtervar)
    .concat(FILTER_RULE_TYPES.map((rt) => rt.isnull_filtervar))
    .filter((rt) => rt !== undefined)

  // 2. 遍历URL中存在的筛选参数
  allFilterRuleQueryParams
    .filter((frqp) => queryParams.has(frqp))
    .forEach((filterQueryParamName) => {
      // 3. 反查：参数名 → rule_type
      const rule_type = FILTER_RULE_TYPES.find((rt) =>
        rt.filtervar == filterQueryParamName ||
        rt.isnull_filtervar == filterQueryParamName
      )

      // 4. 判断是否是 isnull 类参数
      const isNullRuleType = rule_type.isnull_filtervar == filterQueryParamName
      const nullRuleValue = queryParams.get(filterQueryParamName) == '1'
        ? null
        : NEGATIVE_NULL_FILTER_VALUE.toString()

      // 5. 多值参数按逗号拆分
      const filterQueryParamValues = rule_type.multi
        ? queryParams.get(filterQueryParamName).split(',')
        : [queryParams.get(filterQueryParamName)]

      // 6. 每个值生成一条 FilterRule
      filterRulesFromQueryParams = filterRulesFromQueryParams.concat(
        filterQueryParamValues.map((val) => {
          if (rule_type.datatype == 'boolean')
            val = val.replace('1', 'true').replace('0', 'false')
          return {
            rule_type: rule_type.id,
            value: isNullRuleType ? nullRuleValue : val,
          }
        })
      )
    })

  // 7. 旧版自定义字段规则转换
  return transformLegacyFilterRules(filterRulesFromQueryParams)
}
```

### 9.4 反向同步中的循环防护

**文件**：[document-list-view.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/services/document-list-view.service.ts#L276-L301)

```typescript
loadFromQueryParams(queryParams: ParamMap) {
  const paramsEmpty = queryParams.keys.length == 0
  let newState = this.listViewStates.get(this._activeSavedViewId)
  if (!paramsEmpty) newState = paramsToViewState(queryParams)
  if (newState == undefined) newState = this.defaultListViewState()

  // 仅在状态确实变化时才 reload，避免：
  // reload → URL回写 → queryParamMap触发 → loadFromQueryParams → 无限循环
  if (
    !this.initialized ||
    paramsEmpty ||
    this.activeListViewState.sortField !== newState.sortField ||
    this.activeListViewState.sortReverse !== newState.sortReverse ||
    this.activeListViewState.currentPage !== newState.currentPage ||
    filterRulesDiffer(this.activeListViewState.filterRules, newState.filterRules)
  ) {
    this.activeListViewState.filterRules = newState.filterRules
    this.activeListViewState.sortField = newState.sortField
    this.activeListViewState.sortReverse = newState.sortReverse
    this.activeListViewState.currentPage = newState.currentPage
    this.reload(null, paramsEmpty)
  }
}
```

**循环防护机制**：
1. `reload()` 成功后回写 URL → 触发 `queryParamMap` 订阅
2. `loadFromQueryParams()` 比较新URL参数与当前状态 → **完全相同则跳过reload**
3. 额外保护：`initialized` 标志确保首次加载无条件执行

### 9.5 replaceUrl 的首次导航判断

```typescript
replaceUrl: !this.router.routerState.snapshot.url.includes('?')
```

- 当前URL**不含 `?`** → 说明是从无参数状态首次进入 → `replaceUrl: true`（替换历史记录）
- 当前URL**含 `?`** → 说明是从已有筛选状态变更 → `replaceUrl: false`（新增历史记录，允许浏览器后退）

---

## 第十部分：筛选规则 ↔ 后端参数完整对应表

### 10.1 全部50种筛选规则映射

| ID | 常量 | filtervar | isnull_filtervar | datatype | multi | 后端处理方式 |
|----|------|-----------|-----------------|----------|-------|------------|
| 0 | FILTER_TITLE | `title__icontains` | — | string | ✗ | 参数转换时被重写为 `title_search` → Tantivy TITLE（已废弃，保留兼容旧保存视图） |
| 1 | FILTER_CONTENT | `content__icontains` | — | string | ✗ | ORM `icontains` |
| 2 | FILTER_ASN | `archive_serial_number` | — | number | ✗ | ORM `exact` |
| 3 | FILTER_CORRESPONDENT | `correspondent__id` | `correspondent__isnull` | Correspondent | ✗ | ORM `exact` / `isnull` |
| 4 | FILTER_DOCUMENT_TYPE | `document_type__id` | `document_type__isnull` | DocumentType | ✗ | ORM `exact` / `isnull` |
| 5 | FILTER_IS_IN_INBOX | `is_in_inbox` | — | boolean | ✗ | 自定义 `InboxFilter` |
| 6 | FILTER_HAS_TAGS_ALL | `tags__id__all` | — | Tag | ✓ | 自定义 `ObjectFilter`（AND） |
| 7 | FILTER_HAS_ANY_TAG | `is_tagged` | — | boolean | ✗ | `BooleanFilter(exclude=True)` |
| 8 | FILTER_CREATED_BEFORE | `created__date__lt` | — | date | ✗ | ORM `lt`（已废弃） |
| 9 | FILTER_CREATED_AFTER | `created__date__gt` | — | date | ✗ | ORM `gt`（已废弃） |
| 10 | FILTER_CREATED_YEAR | `created__year` | — | number | ✗ | ORM `year` |
| 11 | FILTER_CREATED_MONTH | `created__month` | — | number | ✗ | ORM `month` |
| 12 | FILTER_CREATED_DAY | `created__day` | — | number | ✗ | ORM `day` |
| 13 | FILTER_ADDED_BEFORE | `added__date__lt` | — | date | ✗ | ORM `lt`（已废弃） |
| 14 | FILTER_ADDED_AFTER | `added__date__gt` | — | date | ✗ | ORM `gt`（已废弃） |
| 15 | FILTER_MODIFIED_BEFORE | `modified__date__lt` | — | date | ✗ | ORM `lt` |
| 16 | FILTER_MODIFIED_AFTER | `modified__date__gt` | — | date | ✗ | ORM `gt` |
| 17 | FILTER_DOES_NOT_HAVE_TAG | `tags__id__none` | — | Tag | ✓ | 自定义 `ObjectFilter(exclude=True)` |
| 18 | FILTER_ASN_ISNULL | `archive_serial_number__isnull` | — | boolean | ✗ | ORM `isnull` |
| 19 | FILTER_TITLE_CONTENT | `title_content` | — | string | ✗ | 参数转换时被重写为 `text` → Tantivy TEXT（已废弃，保留兼容旧保存视图） |
| 20 | FILTER_FULLTEXT_QUERY | `query` | — | string | ✗ | → Tantivy `SearchMode.QUERY`（高级搜索语法） |
| 21 | FILTER_FULLTEXT_MORELIKE | `more_like_id` | — | number | ✗ | → Tantivy `more_like_this_ids()`（单独分支，不走 `_get_tantivy_query_and_mode`） |
| 22 | FILTER_HAS_TAGS_ANY | `tags__id__in` | — | Tag | ✓ | 自定义 `ObjectFilter(in_list=True)` |
| 23 | FILTER_ASN_GT | `archive_serial_number__gt` | — | number | ✗ | ORM `gt` |
| 24 | FILTER_ASN_LT | `archive_serial_number__lt` | — | number | ✗ | ORM `lt` |
| 25 | FILTER_STORAGE_PATH | `storage_path__id` | `storage_path__isnull` | StoragePath | ✗ | ORM `exact` / `isnull` |
| 26 | FILTER_HAS_CORRESPONDENT_ANY | `correspondent__id__in` | — | Correspondent | ✓ | ORM `in` |
| 27 | FILTER_DOES_NOT_HAVE_CORRESPONDENT | `correspondent__id__none` | — | Correspondent | ✓ | 自定义 `ObjectFilter(exclude=True)` |
| 28 | FILTER_HAS_DOCUMENT_TYPE_ANY | `document_type__id__in` | — | DocumentType | ✓ | ORM `in` |
| 29 | FILTER_DOES_NOT_HAVE_DOCUMENT_TYPE | `document_type__id__none` | — | DocumentType | ✓ | 自定义 `ObjectFilter(exclude=True)` |
| 30 | FILTER_HAS_STORAGE_PATH_ANY | `storage_path__id__in` | — | StoragePath | ✓ | ORM `in` |
| 31 | FILTER_DOES_NOT_HAVE_STORAGE_PATH | `storage_path__id__none` | — | StoragePath | ✓ | 自定义 `ObjectFilter(exclude=True)` |
| 32 | FILTER_OWNER | `owner__id` | — | number | ✗ | ORM `exact` |
| 33 | FILTER_OWNER_ANY | `owner__id__in` | — | number | ✓ | ORM `in` |
| 34 | FILTER_OWNER_ISNULL | `owner__isnull` | — | boolean | ✗ | ORM `isnull` |
| 35 | FILTER_OWNER_DOES_NOT_INCLUDE | `owner__id__none` | — | number | ✓ | 自定义 `ObjectFilter(exclude=True)` |
| 36 | FILTER_CUSTOM_FIELDS_TEXT | `custom_fields__icontains` | — | string | ✗ | 自定义 `CustomFieldsFilter`（已废弃，日志警告） |
| 37 | FILTER_SHARED_BY_USER | `shared_by__id` | — | number | ✓ | 自定义 `SharedByUser` |
| 38 | FILTER_HAS_CUSTOM_FIELDS_ALL | `custom_fields__id__all` | — | number | ✓ | `ObjectFilter(field_name="custom_fields__field")` 逐个AND（旧版，前端自动转42） |
| 39 | FILTER_HAS_CUSTOM_FIELDS_ANY | `custom_fields__id__in` | — | number | ✓ | `ObjectFilter(field_name="custom_fields__field", in_list=True)`（旧版，前端自动转42） |
| 40 | FILTER_DOES_NOT_HAVE_CUSTOM_FIELDS | `custom_fields__id__none` | — | number | ✓ | `ObjectFilter(field_name="custom_fields__field", exclude=True)` 逐个排除 |
| 41 | FILTER_HAS_ANY_CUSTOM_FIELDS | `has_custom_fields` | — | boolean | ✗ | `BooleanFilter(field_name="custom_fields", lookup_expr="isnull", exclude=True)` 即"有自定义字段" |
| 42 | FILTER_CUSTOM_FIELDS_QUERY | `custom_field_query` | — | string | ✗ | 自定义 `CustomFieldQueryFilter`（JSON表达式解析） |
| 43 | FILTER_CREATED_TO | `created__date__lte` | — | date | ✗ | ORM `lte` |
| 44 | FILTER_CREATED_FROM | `created__date__gte` | — | date | ✗ | ORM `gte` |
| 45 | FILTER_ADDED_TO | `added__date__lte` | — | date | ✗ | ORM `lte` |
| 46 | FILTER_ADDED_FROM | `added__date__gte` | — | date | ✗ | ORM `gte` |
| 47 | FILTER_MIME_TYPE | `mime_type` | — | string | ✗ | ORM `exact` |
| 48 | FILTER_SIMPLE_TITLE | `title_search` | — | string | ✗ | → Tantivy标题搜索 |
| 49 | FILTER_SIMPLE_TEXT | `text` | — | string | ✗ | → Tantivy全文搜索 |

### 10.2 参数转换中的特殊路由

某些 `rule_type` 的 `filtervar` 与最终HTTP参数名不同，在 [queryParamsFromFilterRules](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/utils/query-params.ts#L151-L188) 中被特殊分支拦截并重写：

| rule_type | 定义中的 filtervar | 最终HTTP参数名 | 转换分支 |
|-----------|-------------------|---------------|---------|
| 19 (FILTER_TITLE_CONTENT) | `title_content` | `text` | 第157-160行：if-else 先于 filtervar 匹配 |
| 49 (FILTER_SIMPLE_TEXT) | `text` | `text` | 第157-160行：同上分支，filtervar 恰好一致 |
| 0 (FILTER_TITLE) | `title__icontains` | `title_search` | 第161-164行：if-else 先于 filtervar 匹配 |
| 48 (FILTER_SIMPLE_TITLE) | `title_search` | `title_search` | 第161-164行：同上分支，filtervar 恰好一致 |

**关键代码逻辑**（[query-params.ts#L156-L165](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/utils/query-params.ts#L156-L165)）：

```typescript
// 这两个 if-else 分支在普通 filtervar 匹配之前执行，拦截了4种规则
if (rule.rule_type === FILTER_TITLE_CONTENT || rule.rule_type === FILTER_SIMPLE_TEXT) {
  params[SIMPLE_TEXT_PARAMETER] = rule.value     // → 'text'
} else if (rule.rule_type === FILTER_TITLE || rule.rule_type === FILTER_SIMPLE_TITLE) {
  params[SIMPLE_TITLE_PARAMETER] = rule.value    // → 'title_search'
}
```

**注意**：FILTER_TITLE(0) 的 filtervar 虽然定义为 `title__icontains`，但**永远不会作为 `title__icontains` 发送到后端**，因为特殊分支会将其重写为 `title_search`。同样，FILTER_TITLE_CONTENT(19) 的 filtervar `title_content` 也不会被使用，而是被重写为 `text`。这两个旧规则仅在保存视图的数据库记录中保留原始 rule_type ID，参数转换时统一走新路径。

### 10.3 isnull 双参数规则

以下规则同时拥有 `filtervar` 和 `isnull_filtervar`，形成双通道筛选：

| rule_type | filtervar（有值时） | isnull_filtervar（空值时） | 示例 |
|-----------|-------------------|--------------------------|------|
| 3 (FILTER_CORRESPONDENT) | `correspondent__id=5` | `correspondent__isnull=1` | 有通讯员 / 无通讯员 |
| 4 (FILTER_DOCUMENT_TYPE) | `document_type__id=3` | `document_type__isnull=1` | 有文档类型 / 无文档类型 |
| 25 (FILTER_STORAGE_PATH) | `storage_path__id=2` | `storage_path__isnull=1` | 有存储路径 / 无存储路径 |

**URL中两个参数互斥**：一个规则只会生成 `filtervar` 或 `isnull_filtervar` 之一，取决于 `value` 是否为 null。

### 10.4 后端Tantivy触发参数

后端在 [UnifiedSearchViewSet.list()](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src/documents/views.py#L2250-L2451) 中根据 `_TANTIVY_SEARCH_PARAM_NAMES = ("text", "title_search", "query", "more_like_id")` 判断是否进入搜索模式。但4个参数在后端的处理路径不同：

| HTTP参数 | 对应前端规则 | 判断方式 | 后端处理路径 |
|---------|------------|---------|------------|
| `text` | FILTER_SIMPLE_TEXT(49) / FILTER_TITLE_CONTENT(19) | `_get_tantivy_query_and_mode` | `SearchMode.TEXT` → `run_text_search` |
| `title_search` | FILTER_SIMPLE_TITLE(48) / FILTER_TITLE(0) | `_get_tantivy_query_and_mode` | `SearchMode.TITLE` → `run_text_search` |
| `query` | FILTER_FULLTEXT_QUERY(20) | `_get_tantivy_query_and_mode` | `SearchMode.QUERY` → `run_text_search` |
| `more_like_id` | FILTER_FULLTEXT_MORELIKE(21) | 单独 `if "more_like_id" in request.query_params` | `_get_more_like_id()` → `run_more_like_this` |

**重要细节**：
1. `more_like_id` **不经过** `_get_tantivy_query_and_mode()`，而是在 `list()` 中通过 `if "more_like_id" in request.query_params` 单独判断，调用 `run_more_like_this()` 分支
2. 四个参数**互斥**：后端在 `parse_search_params()` 中检查 `if len(active) > 1` 则抛出 `ValidationError`，提示"Specify only one of text, title_search, query, or more_like_id"
3. `_get_tantivy_query_and_mode` 只处理前3个参数（text/title_search/query），优先级为 `text` > `title_search` > `query`

**混合筛选行为**：当请求同时包含 Tantivy 参数和 ORM 参数时：
1. 后端先调用 `self.filter_queryset(self.get_queryset())` 应用所有 ORM 筛选器，得到 `filtered_qs`
2. Tantivy 执行全文搜索，返回匹配文档ID列表
3. `intersect_and_order()` 将 Tantivy 结果与 `filtered_qs` 取交集 → 最终结果
4. 排序由 Tantivy 或 ORM 决定（取决于 `use_tantivy_sort` 判断）

### 10.5 旧版规则自动升级

**文件**：[query-params.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/utils/query-params.ts#L60-L101)

`transformLegacyFilterRules()` 会将旧版自定义字段规则自动转换为新的查询语法。此转换**仅在前端参数转换阶段执行**，后端仍保留了原始过滤器的处理能力：

```
旧规则: 两条 FILTER_HAS_CUSTOM_FIELDS_ALL(38)，value 分别为 "5" 和 "8"
    ↓ 前端自动转换（transformLegacyFilterRules）
新规则: FILTER_CUSTOM_FIELDS_QUERY(42) + value='["and",[["5","exists",true],["8","exists",true]]]'

旧规则: 两条 FILTER_HAS_CUSTOM_FIELDS_ANY(39)，value 分别为 "5" 和 "8"
    ↓ 前端自动转换（transformLegacyFilterRules）
新规则: FILTER_CUSTOM_FIELDS_QUERY(42) + value='["or",[["5","exists",true],["8","exists",true]]]'
```

**后端兼容性**：虽然前端会自动将38/39转为42，但后端 [DocumentFilterSet](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src/documents/filters.py#L811-L821) 仍保留了 `custom_fields__id__all` 和 `custom_fields__id__in` 过滤器，可以独立工作。这意味着：
- 如果直接构造 API 请求（绕过前端），`custom_fields__id__all=5,8` 仍然有效
- 前端路径下，38/39 规则被转换后会被从 FilterRule 数组中移除（[query-params.ts#L98-L100](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/utils/query-params.ts#L98-L100)）

**注意**：`FILTER_DOES_NOT_HAVE_CUSTOM_FIELDS`(40) 和 `FILTER_HAS_ANY_CUSTOM_FIELDS`(41) **不参与**旧版转换（代码中有 TODO 注释），它们仍然使用原始的后端过滤器路径。

---

## 第十一部分：FilterEditor 组件的筛选规则双向转换

FilterEditor 是连接用户交互与 FilterRule 体系的桥梁，它需要将 FilterRule 转换为可交互的UI控件状态，再将用户操作转回 FilterRule。

### 11.1 FilterRule → UI控件（setter）

**文件**：[filter-editor.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/components/document-list/filter-editor/filter-editor.component.ts#L428-L771)

当 `filterRules` 被设置时（例如从URL反向同步或保存视图加载），组件遍历每条规则，将值分配到对应的UI模型：

| rule_type | UI模型 | 转换逻辑 |
|-----------|--------|---------|
| FILTER_SIMPLE_TEXT(49) / FILTER_TITLE_CONTENT(19) | `_textFilter` + `textFilterTarget='title-content'` | 直接赋值 |
| FILTER_SIMPLE_TITLE(48) / FILTER_TITLE(0) | `_textFilter` + `textFilterTarget='title'` | 直接赋值 |
| FILTER_ASN(2) | `_textFilter` + `textFilterTarget='asn'` | 直接赋值 |
| FILTER_MIME_TYPE(47) | `_textFilter` + `textFilterTarget='mime-type'` | 直接赋值 |
| FILTER_FULLTEXT_QUERY(20) | `_textFilter` + `textFilterTarget='fulltext-query'` | 拆分相对日期关键词 |
| FILTER_FULLTEXT_MORELIKE(21) | `_moreLikeId` + `textFilterTarget='fulltext-morelike'` | 请求文档详情填充标题 |
| FILTER_HAS_TAGS_ALL(6) | `tagSelectionModel` + `logicalOperator=And` | 设为Selected |
| FILTER_HAS_TAGS_ANY(22) | `tagSelectionModel` + `logicalOperator=Or` | 设为Selected |
| FILTER_DOES_NOT_HAVE_TAG(17) | `tagSelectionModel` | 设为Excluded |
| FILTER_HAS_ANY_TAG(7) | `tagSelectionModel` | null设为Selected |
| FILTER_CORRESPONDENT(3) | `correspondentSelectionModel` | null→Selected / "-1"→Excluded |
| FILTER_HAS_CORRESPONDENT_ANY(26) | `correspondentSelectionModel` + `logicalOperator=Or` | 设为Selected |
| FILTER_DOES_NOT_HAVE_CORRESPONDENT(27) | `correspondentSelectionModel` + `intersection=Exclude` | 设为Excluded |
| FILTER_DOCUMENT_TYPE(4) | `documentTypeSelectionModel` | 同correspondent逻辑 |
| FILTER_HAS_DOCUMENT_TYPE_ANY(28) | `documentTypeSelectionModel` + `logicalOperator=Or` | 设为Selected |
| FILTER_DOES_NOT_HAVE_DOCUMENT_TYPE(29) | `documentTypeSelectionModel` + `intersection=Exclude` | 设为Excluded |
| FILTER_STORAGE_PATH(25) | `storagePathSelectionModel` | 同correspondent逻辑 |
| FILTER_HAS_STORAGE_PATH_ANY(30) | `storagePathSelectionModel` + `logicalOperator=Or` | 设为Selected |
| FILTER_DOES_NOT_HAVE_STORAGE_PATH(31) | `storagePathSelectionModel` + `intersection=Exclude` | 设为Excluded |
| FILTER_CUSTOM_FIELDS_QUERY(42) | `customFieldQueriesModel` | JSON解析为Expression/Atom |
| FILTER_ASN_ISNULL(18) | `textFilterTarget='asn'` + `textFilterModifier` | "true"→null / "false"→notnull |
| FILTER_ASN_GT(23) | `textFilterTarget='asn'` + `modifier='greater'` | 赋值 |
| FILTER_ASN_LT(24) | `textFilterTarget='asn'` + `modifier='less'` | 赋值 |
| FILTER_OWNER(32) | `permissionsSelectionModel.ownerFilter=SELF` | 赋值userID |
| FILTER_OWNER_ANY(33) | `permissionsSelectionModel.ownerFilter=OTHERS` | 加入includeUsers |
| FILTER_OWNER_DOES_NOT_INCLUDE(35) | `permissionsSelectionModel.ownerFilter=NOT_SELF` | 加入excludeUsers |
| FILTER_SHARED_BY_USER(37) | `permissionsSelectionModel.ownerFilter=SHARED_BY_ME` | 赋值userID |
| FILTER_OWNER_ISNULL(34) | `permissionsSelectionModel.ownerFilter=UNOWNED` / `hideUnowned` | "true"→UNOWNED / "false"→hideUnowned |
| FILTER_CREATED_AFTER(9) / FILTER_CREATED_FROM(44) | `dateCreatedFrom` | 旧版日期+1天偏移 / 新版直接赋值 |
| FILTER_CREATED_BEFORE(8) / FILTER_CREATED_TO(43) | `dateCreatedTo` | 旧版日期-1天偏移 / 新版直接赋值 |
| FILTER_ADDED_AFTER(14) / FILTER_ADDED_FROM(46) | `dateAddedFrom` | 旧版日期+1天偏移 / 新版直接赋值 |
| FILTER_ADDED_BEFORE(13) / FILTER_ADDED_TO(45) | `dateAddedTo` | 旧版日期-1天偏移 / 新版直接赋值 |

### 11.2 UI控件 → FilterRule（getter）

**文件**：[filter-editor.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/30-paperless-ngx/src-ui/src/app/components/document-list/filter-editor/filter-editor.component.ts#L773-L1117)

`filterRules` getter 在每次用户操作后重新构建完整的筛选规则数组：

```
textFilter + textFilterTarget 组合:
  'title-content' + text → FILTER_SIMPLE_TEXT(49)
  'title' + text → FILTER_SIMPLE_TITLE(48)
  'asn' + equals + text → FILTER_ASN(2)
  'asn' + null → FILTER_ASN_ISNULL(18)
  'asn' + gt/lt + text → FILTER_ASN_GT(23) / FILTER_ASN_LT(24)
  'mime-type' + text → FILTER_MIME_TYPE(47)
  'fulltext-query' + text → FILTER_FULLTEXT_QUERY(20)
  'fulltext-morelike' + id → FILTER_FULLTEXT_MORELIKE(21)

tagSelectionModel 组合:
  无选中 + Include → FILTER_HAS_ANY_TAG(7) + value='false'
  选中项 + logicalOperator → FILTER_HAS_TAGS_ALL(6) / FILTER_HAS_TAGS_ANY(22)
  排除项 → FILTER_DOES_NOT_HAVE_TAG(17)

correspondentSelectionModel 组合:
  无选中 + Include → FILTER_CORRESPONDENT(3) + value=null
  无选中 + Exclude → FILTER_CORRESPONDENT(3) + value="-1"
  选中项 → FILTER_HAS_CORRESPONDENT_ANY(26)
  排除项 → FILTER_DOES_NOT_HAVE_CORRESPONDENT(27)

（documentType / storagePath 同 correspondent 模式）

customFieldQueriesModel → FILTER_CUSTOM_FIELDS_QUERY(42) + JSON值

dateCreatedFrom / dateCreatedTo → FILTER_CREATED_FROM(44) / FILTER_CREATED_TO(43)
dateAddedFrom / dateAddedTo → FILTER_ADDED_FROM(46) / FILTER_ADDED_TO(45)
dateCreatedRelativeDate / dateAddedRelativeDate → 注入到 FILTER_FULLTEXT_QUERY(20) 中

permissionsSelectionModel 组合:
  SELF → FILTER_OWNER(32)
  NOT_SELF → FILTER_OWNER_DOES_NOT_INCLUDE(35)
  OTHERS → FILTER_OWNER_ANY(33)
  SHARED_BY_ME → FILTER_SHARED_BY_USER(37)
  UNOWNED → FILTER_OWNER_ISNULL(34) + value='true'
  hideUnowned → FILTER_OWNER_ISNULL(34) + value='false'
```

### 11.3 相对日期的特殊处理

相对日期筛选不直接生成独立的 FilterRule，而是**注入到** `FILTER_FULLTEXT_QUERY` 规则中：

```
用户选择 "上周"
    ↓
dateCreatedRelativeDate = RelativeDate.PREVIOUS_WEEK
    ↓
在 filterRules getter 中:
  ├─ 如果已有 FILTER_FULLTEXT_QUERY 规则 → 追加 created:[previous week]
  ├─ 如果已有标题/内容搜索 → 升级为 FILTER_FULLTEXT_QUERY + 追加日期
  └─ 如果没有 → 新建 FILTER_FULLTEXT_QUERY + value="created:[previous week]"
```

这意味着相对日期和全文搜索共享同一条 FilterRule，通过逗号分隔多个查询子句。
