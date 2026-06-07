# Saved Views 与列表状态配合机制分析

## 1. 整体架构概览

Paperless-ngx 中的 Saved Views（保存视图）与文档列表状态形成了一个"持久化配置 ↔ 动态状态"的双向协作体系：

- **后端**：负责存储 SavedView 配置（筛选规则、排序、显示模式等），并基于用户权限控制可见范围
- **前端**：通过 `DocumentListViewService` 管理当前列表状态，支持从 SavedView 加载、修改后回写、以及 URL 参数同步

---

## 2. 后端数据模型

### 2.1 SavedView 模型

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/models.py#L517-L576)

```python
class SavedView(ModelWithOwner):
    name = models.CharField(max_length=128)            # 视图名称
    sort_field = models.CharField(...)                 # 排序字段
    sort_reverse = models.BooleanField(default=False)  # 是否倒序
    page_size = models.PositiveIntegerField(...)       # 分页大小
    display_mode = models.CharField(...)               # 显示模式: table/smallCards/largeCards
    display_fields = models.JSONField(...)             # 显示的列字段 (JSON数组)
```

SavedView 继承自 `ModelWithOwner`，该基类提供 `owner` 字段（指向 User），实现用户所有权。

### 2.2 SavedViewFilterRule 模型

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/models.py#L578-L649)

每个 SavedView 关联多条 FilterRule：

```python
class SavedViewFilterRule(models.Model):
    saved_view = models.ForeignKey(SavedView, on_delete=models.CASCADE, related_name="filter_rules")
    rule_type = models.PositiveSmallIntegerField(choices=RULE_TYPES)  # 规则类型ID (0-49)
    value = models.CharField(max_length=255, blank=True, null=True)   # 规则值
```

支持 50 种规则类型，包括：
- 标题/全文搜索 (0, 1, 19, 20)
- 标签过滤 (6, 7, 17, 22)
- 联系人/文档类型/存储路径 (3, 4, 25 等)
- 日期范围 (8-16, 43-46)
- 所有者权限 (32-35)
- 自定义字段 (36, 38-42)
- ASN 编号 (2, 18, 23, 24)
- 收件箱 (5)
- 共享 (37)
- MIME 类型 (47)
- 简单搜索 (48, 49)

### 2.3 UiSettings 模型（用户可见性偏好）

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/models.py#L652-L661)

存储用户个人的 Saved View 可见性偏好：

```python
class UiSettings(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name="ui_settings")
    settings = models.JSONField(null=True)  # JSON格式存储所有UI偏好
```

Saved views 相关设置存储路径：`settings.saved_views.dashboard_views_visible_ids` / `sidebar_views_visible_ids`

---

## 3. 用户 Scope（权限范围）机制

### 3.1 所有权模型

SavedView 通过 `ModelWithOwner` 基类获取 `owner` 字段，建立用户与视图的归属关系。

定义于 [models.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/models.py#L32-L44)

```python
class ModelWithOwner(models.Model):
    owner = models.ForeignKey(User, blank=True, null=True, default=None, on_delete=models.SET_NULL)
```

### 3.2 权限过滤器

`ObjectOwnedOrGrantedPermissionsFilter` 控制用户可访问的 SavedView 范围。

定义于 [filters.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/filters.py#L950-L962)

```python
class ObjectOwnedOrGrantedPermissionsFilter(ObjectPermissionsFilter):
    def filter_queryset(self, request, queryset, view):
        objects_with_perms = super().filter_queryset(request, queryset, view)  # 通过 guardian 授权的对象
        objects_owned = queryset.filter(owner=request.user)                     # 用户自己拥有的
        objects_unowned = queryset.filter(owner__isnull=True)                   # 无所有者的（向后兼容）
        return objects_with_perms | objects_owned | objects_unowned
```

用户可见的 SavedView 集合 = {自有的} ∪ {被授权的} ∪ {无所有者的}

### 3.3 视图级权限控制

在 [views.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/views.py#L2548-L2562) 的 `SavedViewViewSet` 中：

```python
class SavedViewViewSet(BulkPermissionMixin, PassUserMixin, ModelViewSet[SavedView]):
    queryset = SavedView.objects.select_related("owner").prefetch_related("filter_rules")
    permission_classes = (IsAuthenticated, PaperlessObjectPermissions)
    filter_backends = (OrderingFilter, ObjectOwnedOrGrantedPermissionsFilter)
```

- `IsAuthenticated`：必须登录
- `PaperlessObjectPermissions`：基于 django-guardian 的对象级权限
- `ObjectOwnedOrGrantedPermissionsFilter`：过滤查询结果集

### 3.4 用户可见性偏好（Dashboard/Sidebar）

每个用户可独立配置 Saved View 在侧边栏和仪表盘的显示状态。该偏好保存在用户自己的 `UiSettings` 中，不影响 SavedView 对象本身。

后端处理见 [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/serialisers.py#L1350-L1408) 的 `_update_legacy_visibility_preferences` 方法，前端调用见 [settings.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/services/settings.service.ts#L728-L739) 的 `updateSavedViewsVisibility`。

---

## 4. 筛选条件保存与加载

### 4.1 保存流程（前端 → 后端）

#### 4.1.1 前端触发

在 [document-list.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/components/document-list/document-list.component.ts#L398-L428) 中：

```typescript
saveViewConfig() {
  let savedView: SavedView = {
    id: this.list.activeSavedViewId,
    filter_rules: this.list.filterRules,        // 当前筛选规则
    sort_field: this.list.sortField,            // 当前排序字段
    sort_reverse: this.list.sortReverse,        // 当前排序方向
    display_mode: this.list.displayMode,        // 当前显示模式
    display_fields: this.activeDisplayFields,   // 当前显示列
  }
  this.savedViewService.patch(savedView).subscribe(...)
}
```

新建视图使用 `saveViewConfigAs()`，通过 `SaveViewConfigDialogComponent` 对话框输入名称和权限。

#### 4.1.2 序列化与存储

在 [serialisers.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/serialisers.py#L1480-L1532) 的 `SavedViewSerializer` 中：

```python
def update(self, instance, validated_data):
    rules_data = validated_data.pop("filter_rules")
    instance = super().update(instance, validated_data)
    if rules_data is not None:
        # 先删除旧规则，再批量创建新规则
        SavedViewFilterRule.objects.filter(saved_view=instance).delete()
        for rule_data in rules_data:
            SavedViewFilterRule.objects.create(saved_view=instance, **rule_data)
    return instance
```

**注意**：filter_rules 的更新是"全量替换"策略，不是增量更新。

### 4.2 加载流程（后端 → 前端）

#### 4.2.1 路由匹配

前端通过两条路由加载 SavedView：

**路由 1**：`/view/:id` — 专用视图页

在 [document-list.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/components/document-list/document-list.component.ts#L274-L303)：

```typescript
this.route.paramMap.pipe(
  filter((params) => params.has('id')),
  switchMap((params) => this.savedViewService.getCached(+params.get('id')))
).subscribe(({ view }) => {
  this.activeSavedView = view
  this.unmodifiedSavedView = view  // 保存原始副本用于修改检测
  this.list.activateSavedViewWithQueryParams(view, queryParams)
  this.list.reload(() => {
    this.savedViewService.setDocumentCount(view, this.list.collectionSize)
  })
})
```

**路由 2**：`/documents?view=:id` — 普通文档列表页通过 query 参数加载

见 [document-list.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/components/document-list/document-list.component.ts#L305-L321) 中的 `loadViewConfig` 方法。

#### 4.2.2 状态激活

在 [document-list-view.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/services/document-list-view.service.ts#L238-L274) 中：

```typescript
activateSavedView(view: SavedView) {
  this._activeSavedViewId = view.id
  this.loadSavedView(view)
}

loadSavedView(view: SavedView) {
  // 将 SavedView 的配置同步到当前列表状态
  this.activeListViewState.filterRules = cloneFilterRules(view.filter_rules)
  this.activeListViewState.sortField = view.sort_field
  this.activeListViewState.sortReverse = view.sort_reverse
  this.activeListViewState.title = view.name
  this.activeListViewState.displayMode = view.display_mode
  this.activeListViewState.pageSize = view.page_size
  this.activeListViewState.displayFields = view.display_fields
  // 跳转到 /view/:id 路由
  this.router.navigate(['view', view.id])
}
```

---

## 5. 前端列表状态管理

### 5.1 ListViewState 数据结构

定义于 [document-list-view.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/services/document-list-view.service.ts#L45-L102)

```typescript
export interface ListViewState {
  title?: string              // 标题（SavedView 名称或 "Documents"）
  documents?: Document[]      // 当前页文档
  currentPage: number         // 当前页码
  collectionSize?: number     // 匹配的文档总数
  sortField: string           // 排序字段
  sortReverse: boolean        // 是否倒序
  filterRules: FilterRule[]   // 筛选规则数组
  selected?: Set<number>      // 选中的文档ID
  allSelected?: boolean       // 是否全选
  pageSize?: number           // 每页条数
  displayMode?: DisplayMode   // 显示模式
  displayFields?: DisplayField[]  // 显示列
}
```

### 5.2 多视图状态隔离

`DocumentListViewService` 使用 `Map<number, ListViewState>` 维护多个独立的视图状态：

```typescript
private listViewStates: Map<number, ListViewState> = new Map()
private _activeSavedViewId: number = null  // null 表示默认/临时视图

private get activeListViewState() {
  if (!this.listViewStates.has(this._activeSavedViewId)) {
    this.listViewStates.set(this._activeSavedViewId, this.defaultListViewState())
  }
  return this.listViewStates.get(this._activeSavedViewId)
}
```

- key 为 `null`：默认文档列表（无 SavedView 上下文）
- key 为 SavedView ID：对应已保存视图的独立状态

这种设计使用户在切换不同 SavedView 时，各自的页码、选中状态等得以保留。

### 5.3 默认视图的本地持久化

非 SavedView 上下文（`_activeSavedViewId == null`）的列表状态会保存到 `localStorage`：

```typescript
private saveDocumentListView() {
  if (this._activeSavedViewId == null) {
    let savedState = {
      collectionSize, currentPage, filterRules, sortField,
      sortReverse, displayMode, displayFields
    }
    localStorage.setItem(DOCUMENT_LIST_SERVICE.CURRENT_VIEW_CONFIG, JSON.stringify(savedState))
  }
}
```

服务构造时自动从 localStorage 恢复。

### 5.4 修改检测（Dirty Tracking）

在 [document-list.component.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/components/document-list/document-list.component.ts#L158-L190) 中，通过对比 `unmodifiedSavedView`（加载时的快照）与当前列表状态来判断是否已修改：

```typescript
get savedViewIsModified(): boolean {
  return (
    this.unmodifiedSavedView.sort_field !== this.list.sortField ||
    this.unmodifiedSavedView.sort_reverse !== this.list.sortReverse ||
    this.unmodifiedSavedView.page_size !== this.list.pageSize ||
    this.unmodifiedSavedView.display_mode !== this.list.displayMode ||
    // display_fields 对比
    // filter_rules 对比（使用 filterRulesDiffer 工具函数）
    filterRulesDiffer(this.unmodifiedSavedView.filter_rules, this.list.filterRules)
  )
}
```

修改后标题会显示 `*` 后缀，提示用户保存。

---

## 6. 筛选规则与 URL 参数的双向映射

### 6.1 FilterRule → Query Params

在 [query-params.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/utils/query-params.ts#L151-L188)：

```typescript
export function queryParamsFromFilterRules(filterRules: FilterRule[]): Params {
  let params = {}
  for (let rule of filterRules) {
    let ruleType = FILTER_RULE_TYPES.find((t) => t.id == rule.rule_type)
    // 根据 ruleType 的 filtervar 字段映射到 URL 参数名
    // 例：rule_type=6 (has tag) → filtervar='tags__id__all'
    params[ruleType.filtervar] = rule.value
  }
  return params
}
```

每个 `FilterRuleType` 定义了 `filtervar`（后端查询参数名）、`multi`（是否多值，逗号分隔）、`datatype` 等属性。

### 6.2 Query Params → FilterRule

反向转换见 [query-params.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/utils/query-params.ts#L103-L149) 的 `filterRulesFromQueryParams`。

### 6.3 列表状态 → URL 同步

在 `DocumentListViewService.reload()` 中：

- **非 SavedView 模式**：导航到 `/documents`，将完整列表状态（筛选、排序、分页）写入 query params
- **SavedView 模式**：仅将分页参数 merge 到当前 URL（筛选/排序由 SavedView 定义，不写入 URL 以保持 URL 简洁）

---

## 7. 前端列表刷新机制

### 7.1 核心 reload 方法

在 [document-list-view.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src-ui/src/app/services/document-list-view.service.ts#L304-L387)：

```typescript
reload(onFinish?, updateQueryParams: boolean = true) {
  this.documentService.listFiltered(
    activeListViewState.currentPage,
    activeListViewState.pageSize ?? this.pageSize,
    activeListViewState.sortField,
    activeListViewState.sortReverse,
    activeListViewState.filterRules,
    { truncate_content: true, include_selection_data: true }
  ).subscribe({
    next: (result) => {
      activeListViewState.collectionSize = result.count
      activeListViewState.documents = result.results
      this.selectionData = result.selection_data
      // 同步 URL 参数...
    },
    error: (error) => {
      // 404: 当前页不存在 → 回到第1页重试
      // 自定义字段已删除 → 重置排序字段为 'created'
    }
  })
}
```

### 7.2 触发刷新的场景

| 操作 | 触发位置 |
|------|---------|
| 切换 SavedView | `activateSavedView` → `reload` |
| 修改筛选规则 | `setFilterRules` → `reload` |
| 修改排序 | `setSort` / `sortField` setter → `reload` |
| 切换页码 | `currentPage` setter → `reload` |
| 修改每页条数 | `pageSize` setter → `reload` |
| 文档消费完成（WebSocket） | `onDocumentConsumptionFinished` → `reload` |
| 文档删除（WebSocket） | `onDocumentDeleted` → `reload` |

### 7.3 include_selection_data 的作用

当请求参数 `include_selection_data=true` 时，后端 `DocumentViewSet.list` 方法（[views.py](file:///d:/fz/0601/solo-dogfeeding/code/59-paperless-ngx/src/documents/views.py#L1185-L1202)）会额外计算当前筛选结果下的聚合数据：

```python
def list(self, request, *args, **kwargs):
    queryset = self.filter_queryset(self.get_queryset())
    selection_data = self._get_selection_data_for_queryset(queryset)
    # 返回各分类（tags/correspondents/document_types/storage_paths/custom_fields）
    # 在当前筛选结果中的文档计数
```

这些数据用于筛选器 UI 中显示每个选项旁边的匹配数量。

---

## 8. 列表状态与 Saved Views 的完整交互流程

```
用户访问 /view/42
       │
       ▼
route.paramMap 触发 ──► savedViewService.getCached(42)
       │
       ▼
activateSavedViewWithQueryParams(view, queryParams)
       │
       ├─► _activeSavedViewId = 42
       ├─► 从 listViewStates Map 获取该视图的状态（或创建默认）
       ├─► 将 view.filter_rules/sort_field/sort_reverse/display_mode 等写入状态
       └─► currentPage = queryParams 中的 page（如存在）
       │
       ▼
reload()
       │
       ├─► documentService.listFiltered(page, pageSize, sortField, sortReverse, filterRules)
       │     │
       │     └─► 将 FilterRule[] 转为 URL query params
       │         （filterRules → queryParamsFromFilterRules）
       │
       ├─► 后端 DocumentFilterSet 应用所有过滤条件
       │     │
       │     ├─► ObjectOwnedOrGrantedPermissionsFilter 确保只返回用户可见文档
       │     └─► DocumentsOrderingFilter 处理排序（含自定义字段）
       │
       ├─► 返回分页结果 + selection_data
       │
       ├─► 更新 activeListViewState.documents / collectionSize
       │
       └─► URL 同步（SavedView 模式下仅同步 page 参数）
       │
       ▼
用户修改筛选条件（FilterEditorComponent）
       │
       ▼
onFilterRulesChange → list.setFilterRules(newRules)
       │
       ├─► 更新 activeListViewState.filterRules
       ├─► 若当前是全文搜索且排序字段为 'score'，重置为 'created'
       ├─► reload()  （带新筛选条件请求后端）
       ├─► reduceSelectionToFilter()  （清除不再匹配的选中项）
       └─► saveDocumentListView()  （非 SavedView 时写入 localStorage）
       │
       ▼
savedViewIsModified 计算变为 true（标题显示 *）
       │
       ▼
用户点击"保存视图"
       │
       ▼
saveViewConfig()
       │
       ├─► 构建 SavedView 对象（当前 filterRules, sortField, displayMode 等）
       ├─► savedViewService.patch(savedView)
       │     │
       │     └─► 后端 SavedViewSerializer.update
       │           ├─► 更新 SavedView 基本字段
       │           └─► 删除旧 SavedViewFilterRule，创建新规则
       │
       └─► 更新 unmodifiedSavedView（重置 dirty 状态）
```

---

## 9. 关键设计点总结

1. **状态隔离**：每个 SavedView 拥有独立的 `ListViewState`，切换时互不干扰。默认视图（null key）的状态持久化到 localStorage。

2. **Dirty Tracking**：通过保存加载时的 `unmodifiedSavedView` 快照，实现修改检测，引导用户保存。

3. **全量替换 FilterRule**：后端保存 filter_rules 时采用"先删后建"的全量替换策略，简化了前端逻辑。

4. **权限分层**：SavedView 对象的可见性由 `owner` + django-guardian 对象权限控制；而 Dashboard/Sidebar 的显示偏好则存储在用户个人的 `UiSettings` 中，两者完全解耦。

5. **URL 同步策略差异**：非 SavedView 模式完整同步到 URL；SavedView 模式仅同步分页参数，保持 URL 简洁，筛选/排序由 SavedView ID 隐含。

6. **WebSocket 驱动刷新**：文档消费完成或删除时，通过 WebSocket 实时推送触发列表 reload，保证数据新鲜度。
