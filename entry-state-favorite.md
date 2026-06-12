# FreshRSS 阅读状态与收藏切换流程追踪

## 一、数据模型层（Domain Model）

### 1.1 状态常量定义

在 [Entry.php](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Entry.php#L7-L15) 中定义了状态位掩码常量：

```php
class FreshRSS_Entry extends Minz_Model {
    public const STATE_READ          = 1;   // 0b00001
    public const STATE_NOT_READ      = 2;   // 0b00010
    public const STATE_ALL           = 3;   // 0b00011
    public const STATE_FAVORITE      = 4;   // 0b00100
    public const STATE_NOT_FAVORITE  = 8;   // 0b01000
    public const STATE_ANDS          = 15;  // 0b01111
    public const STATE_OR_NOT_READ   = 32;  // 0b100000
    public const STATE_OR_FAVORITE   = 64;  // 0b1000000
    public const STATE_ORS           = 96;  // 0b1100000
}
```

这些常量用于列表查询过滤（Context 层），通过位运算 `FreshRSS_Context::isStateEnabled($state)` 判断某状态是否启用，见 [Context.php](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Context.php#L322-L324)。

### 1.2 Entry 实体的双状态字段

[Entry.php](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Entry.php#L33-L34) 中每条 Entry 维护两个独立状态：

```php
private ?bool $is_read;      // 是否已读
private ?bool $is_favorite;  // 是否收藏
```

对应 getter / setter：
- `isRead()` / `_isRead($value)` —— [Entry.php L470-L472](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Entry.php#L470-L472), [Entry.php L618-L620](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Entry.php#L618-L620)
- `isFavorite()` / `_isFavorite($value)` —— [Entry.php L473-L475](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Entry.php#L473-L475), [Entry.php L622-L624](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Entry.php#L622-L624)

数据库表 `_entry` 对应列为 `is_read` (TINYINT) 和 `is_favorite` (TINYINT)，每次状态变更同时更新 `lastUserModified` 时间戳。

---

## 二、数据持久化层（DAO）

所有数据库操作集中在 [EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/EntryDAO.php)，它通过继承为 SQLite / PostgreSQL 提供特定覆写。

### 2.1 单篇 / 多篇切换阅读状态：`markRead()`

位置：[EntryDAO.php L499-L570](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/EntryDAO.php#L499-L570)

**单篇 ID（string）路径**（L542-L569）：
- 使用 `INNER JOIN _feed` 做原子化 UPDATE
- 同时写 `_entry.is_read`、`_entry.lastUserModified`、`_feed.cache_nbUnreads`（±1）
- WHERE 条件带 `e.is_read=:old_is_read` 防止重复写
- 返回 `rowCount()`，非 0 即表示确实发生状态翻转

```sql
UPDATE `_entry` e INNER JOIN `_feed` f ON e.id_feed=f.id
SET e.is_read=:is_read, `lastUserModified`=:last_user_modified,
    f.`cache_nbUnreads`=f.`cache_nbUnreads` :delta
WHERE e.id=:id AND e.is_read=:old_is_read
```

**多篇 ID（array）路径**（L500-L541）：
- 少于 6 篇：循环调用单篇路径（性能启发式）
- 超过 `MAX_VARIABLE_NUMBER`：`array_chunk` 递归拆分
- 否则用 `WHERE id IN (?,?,...)` 批量 UPDATE
- 批量后调用 `updateCacheUnreads(null, null)` 全量重算 feed 缓存（L538-L540）

两种路径都会先调用 `FreshRSS_UserDAO::touch()` 更新用户级 `last_update` 时间戳。

### 2.2 单篇 / 多篇切换收藏：`markFavorite()`

位置：[EntryDAO.php L416-L451](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/EntryDAO.php#L416-L451)

- 统一按数组处理，单篇先包成 `[$ids]`
- 超量同样递归 chunking
- SQL 仅写 `_entry.is_favorite` 和 `_entry.lastUserModified`
- 写成功后触发扩展钩子：`Minz_HookType::EntriesFavorite`
- **注意**：收藏没有类似 `cache_nbUnreads` 的 feed 级缓存，因此不做缓存更新

### 2.3 批量标记已读（按分类 / Feed / 标签 / 优先级）

DAO 提供了一组针对不同维度的批量方法，均接受 `idMax` 作为 fail-safe ID 上限，避免把刚刚新加载的条目误标记：

| 方法 | 作用范围 | 位置 |
|------|---------|------|
| `markReadEntries($idMax, $onlyFavorites, $priorityMin, $priorityMax, $filters, $state, $is_read)` | 全部 / 收藏 / 按优先级过滤 | [EntryDAO.php L585-L636](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/EntryDAO.php#L585-L636) |
| `markReadCat($id, $idMax, $priorityMin, $filters, $state, $is_read)` | 指定分类下所有 feed | [EntryDAO.php L650-L679](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/EntryDAO.php#L650-L679) |
| `markReadFeed($id_feed, $idMax, $filters, $state, $is_read)` | 指定单个 feed | [EntryDAO.php L692-L742](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/EntryDAO.php#L692-L742) |
| `markReadTag($id, $idMax, $filters, $state, $is_read)` | 指定标签（id=0 代表任意标签） | [EntryDAO.php L750-L784](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/EntryDAO.php#L750-L784) |

共同特征：
- 全都支持 `$state`（读/收藏过滤）和 `$filters`（FreshRSS_BooleanSearch）条件，通过 `sqlListEntriesWhere()` 拼接 WHERE
- 写库后调用 `updateCacheUnreads($catId, $feedId)` 做局部缓存刷新
- `markReadFeed()` 额外开启事务，用精确的 `cache_nbUnreads - $affected` 做增量更新（性能更好）

### 2.4 未读数缓存：`updateCacheUnreads()`

位置：[EntryDAO.php L458-L490](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/EntryDAO.php#L458-L490)

```sql
UPDATE `_feed`
SET `cache_nbUnreads`=(
    SELECT COUNT(*) FROM `_entry` e USE INDEX (entry_feed_read_index)
    WHERE e.id_feed=`_feed`.id AND e.is_read=0)
WHERE ...
```

可选按 `catId` 或 `feedId` 限定重算范围。**这是列表页侧栏、标题栏未读数与数据库保持一致的关键机制**——所有写操作最终都会触达它。

---

## 三、控制器层（Controller）

入口统一在 [entryController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Controllers/entryController.php)。

### 3.1 `readAction()` —— 阅读状态

位置：[entryController.php L47-L222](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Controllers/entryController.php#L47-L222)

**参数解析**：
- `id`：单篇或多篇 ID（`paramArrayString` 自动处理数组）
- `get`：目标范围标识——`c_*` 分类、`f_*` feed、`s` 收藏、`a/A/Z/i` 不同优先级范围、`t_*` 标签、`T` 全部标签
- `idMax`：fail-safe ID 上限
- `is_read`：`1/0/null`，由 `paramTernary` 解析（缺省为 true）
- `state` / `search` / `maxPubDate`：透传给 DAO 作为过滤条件

**分支逻辑**：

1. **无 `id` 参数 → 批量操作**（必须 POST）
   - `get=''` → `markReadEntries()` 全部标记
   - 按 `get[0]` 前缀 switch 到对应 `markReadCat / markReadFeed / markReadTag / markReadEntries`
   - 标签场景还会额外计算 `next_get`，用于标记完后跳到下一个有未读的标签

2. **有 `id` 参数 → 单篇或指定多篇**
   - 从 `id[]` 数组或单个字符串解析成 `$ids`
   - 调用 `$entryDAO->markRead($ids, $is_read)`
   - 同时查询 `TagDAO::getTagsForEntries()` 返回给前端，用于更新标签未读数

**响应**：
- 非 AJAX → `Minz_Request::good()` 302 跳回 index
- AJAX → 渲染 [entry/read.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/entry/read.phtml)，返回 JSON：`{ tags: { "t_123": ["1","2",...], ... } }`

### 3.2 `bookmarkAction()` —— 收藏状态

位置：[entryController.php L232-L246](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Controllers/entryController.php#L232-L246)

逻辑相对简单：
- 参数 `id`（必须数字字符串）、`is_favorite`（缺省 true）
- 调用 `$entryDAO->markFavorite($id, $is_favourite)`
- 非 AJAX → forward 到 index
- AJAX → 渲染 [entry/bookmark.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/entry/bookmark.phtml)，返回 JSON：
  ```json
  { "url": ".../?c=entry&a=bookmark&id=X&is_favorite=0", "icon": "<svg>...</svg>" }
  ```
  即告诉前端"反向切换"所需的新 URL 和新图标 HTML。

---

## 四、视图渲染层：列表回显

条目状态通过 View 层 **三处重复渲染**，但都使用同一套数据来源，保证一致性。

### 4.1 渲染源头：`FreshRSS_Entry::isRead() / isFavorite()`

所有模板都从 PHP 模型拿值：
- 若未读 → 链接 URL 不含 `is_read`（点击后标记为已读），图标为 `unread`
- 若已读 → 链接 URL 带 `&is_read=0`（点击后标记为未读），图标为 `read`
- 收藏同理，切换 `is_favorite` 参数和 `starred / non-starred` 图标

### 4.2 列表头部按钮：`entry_header.phtml`

[entry_header.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/helpers/index/normal/entry_header.phtml#L30-L49) 渲染 Normal 视图每条顶部的读/收藏按钮：

```php
$arUrl = ['c' => 'entry', 'a' => 'read', 'params' => ['id' => $this->entry->id()]];
if ($this->entry->isRead()) {
    $arUrl['params']['is_read'] = '0';   // 已读→未读
}
<a class="read" href="<?= Minz_Url::display($arUrl) ?>">
    <?= _i($this->entry->isRead() ? 'read' : 'unread') ?>
</a>
```

收藏按钮完全类似（L40-L48），class 为 `bookmark`。

### 4.3 列表底部按钮：`entry_bottom.phtml`

[entry_bottom.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/helpers/index/normal/entry_bottom.phtml#L16-L35) 代码结构与 header 完全一致，class 同样为 `read` / `bookmark`。

### 4.4 阅读视图：`article.phtml`

[article.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/helpers/index/article.phtml#L15-L34) 在 Reader 视图的顶部、副标题、底部共三处渲染按钮，所有 `<a>` 都标记同一 class `read` / `bookmark`。

### 4.5 批量操作入口：`stream-footer.phtml`

[stream-footer.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/helpers/stream-footer.phtml#L13-L27) 渲染"全部标为已读"大按钮，其 `formaction` URL 将当前 `get / nextGet / idMax / search / state / sort / order` 全部打包提交给 `entry/readAction()`。

### 4.6 前端注入 JS 上下文：`javascript_vars.phtml`

[javascript_vars.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/helpers/javascript_vars.phtml#L8-L101) 输出 JSON：
- `context.auto_mark_article / scroll / focus / site` —— 四种自动标读触发开关
- `context.csrf` —— AJAX 请求 token
- `context.icons.read / unread / spinner` —— 图标资源 URL
- `shortcuts.mark_read / mark_favorite` —— 键盘快捷键

---

## 五、前端交互层：`main.js`

所有前端逻辑位于 [main.js](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js)。

### 5.1 事件绑定：`init_stream()`

位置：[main.js L1354-L1544](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L1354-L1544)

统一在 `stream.onclick` 中委托：

```js
let el = ev.target.closest('.flux a.read');
if (el) { mark_read(el.closest('.flux'), false, false); return false; }

el = ev.target.closest('.flux a.bookmark');
if (el) { mark_favorite(el.closest('.flux')); return false; }
```

这意味着 **header / bottom / reader 视图下所有 `.read` `.bookmark` 链接共享同一处理函数**，天然保持行为一致。

快捷键同样走相同函数（[main.js L1265-L1275](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L1265-L1275)、[main.js L1314-L1318](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L1314-L1318)）。

### 5.2 阅读状态切换：`mark_read()` + `send_mark_read_queue()`

**`mark_read(div, only_not_read, asBatch)`** —— [main.js L324-L351](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L324-L351)

1. `pending_entries[div.id] = true` 防重复提交
2. 图标临时替换为 `spinner.svg`
3. 根据 `div.classList.contains('not_read')` 判断目标方向 `asRead`
4. **`asBatch=true`** → 推入 `mark_read_queue[]`，设置 1 秒 debounce 定时器合并请求（L342-L346）
5. **`asBatch=false`** → 立即 `send_mark_read_queue([id], asRead)`

**`send_mark_read_queue(queue, asRead, callback)`** —— [main.js L224-L306](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L224-L306)

请求：
```
POST .?c=entry&a=read[&is_read=0]
Body: { ajax: true, _csrf, id: [ids...] }
```

服务器成功响应后（L262-L300）：

| 操作 | 效果 |
|------|------|
| 切换 class | `div.classList.remove/addClass('not_read')` |
| 切换 URL | 所有 `.read` 链接 `href` 增删 `&is_read=0` |
| 切换图标 | 所有 `.read > .icon` 替换为 context.icons.read/unread |
| 增量计数 | `incUnreadsFeed(div, feed_id, ±1)` 更新 feed / category / all / important / favorites 侧栏及标题未读数 |
| 自动移除 | `context.auto_remove_article` 开启时，已读文章从 DOM 移除 |
| 标签未读数 | 使用响应的 `tags` 字段调用 `incUnreadsTag()` 批量更新 |
| 其他 | `faviconNbUnread()`、`toggle_bigMarkAsRead_button()`、`onScroll()` |

失败回滚（L246-L254）：所有图标恢复原状，`delete pending_entries[...]`。

### 5.3 收藏状态切换：`mark_favorite()`

位置：[main.js L360-L440](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L360-L440)

流程与 `mark_read` 对称但使用 XMLHttpRequest（未走 fetch queue）：

1. 同样 `pending_entries` 防重 + spinner 临时图标
2. POST 到 `<a.bookmark>.href`（服务端动态生成的 URL 已决定方向）
3. 响应 JSON 包含 `url`（反向切换用的新 URL）和 `icon`（新图标 HTML）
4. 成功时：
   - `div.classList.toggle('favorite')`
   - `div.querySelectorAll('a.bookmark').href = json.url`
   - `div.querySelectorAll('a.bookmark > .icon').outerHTML = json.icon`
   - 侧栏收藏夹计数 `incLabel()` ±1
   - 若文章未读，同步更新 favorites 伪分类的未读数
5. 失败回滚：恢复 `originalIcon.src/alt`

### 5.4 四类"自动标读"触发

`mark_read()` 的 `asBatch=true` 路径被多处调用，全部通过队列合并以减少 HTTP 请求：

| 触发点 | 位置 | 触发条件来自配置 |
|--------|------|-----------------|
| 打开文章 | [main.js L546-L551](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L546-L551) `toggleContent()` | `context.auto_mark_article` (`mark_when.article`) |
| 焦点移动 | [main.js L567-L569](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L567-L569) / [L585-L587](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L585-L587) | `context.auto_mark_focus` (`mark_when.focus`) |
| 滚动出屏 | [main.js L902-L915](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L902-L915) `onScroll()` | `context.auto_mark_scroll` (`mark_when.scroll`) |
| 点击原网站 | [main.js L1320-L1329](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L1320-L1329) 快捷键 | `context.auto_mark_site` (`mark_when.site`) |

### 5.5 批量标读：`mark_previous_read()`

[main.js L353-L358](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L353-L358)：快捷键 Alt+mark_read 触发，从当前文章向前遍历 `previousElementSibling`，逐条 `mark_read(div, true, true)` 全部入队列，由 debounce 合并为一次请求。

---

## 六、一致性保障机制总览

从用户点击 → UI 反馈 → 数据库 → 列表回显，多层机制共同保障单篇、批量、列表显示的一致性：

### 6.1 数据库层面
- **单篇原子 UPDATE**：`markRead(string)` 用 JOIN 同时写 entry 和 feed.cache_nbUnreads，杜绝"状态写了但缓存没更"的中间态
- **批量重算缓存**：`markRead(array/entries/cat/feed/tag)` 后调用 `updateCacheUnreads()`，用 `COUNT(*)` 从真实数据重算，避免算术累计误差
- **条件写防止重复**：`WHERE is_read<>?` 保证幂等，重复请求不会乱改缓存
- **lastUserModified 同步更新**：每次改状态都写此时间戳，供客户端增量同步

### 6.2 后端接口层面
- **单篇 / 批量共用 DAO 方法**：`markRead($ids, $is_read)` 不管传 string 还是 array，最终写库逻辑一致
- **过滤条件透传**：批量标读时把 `state`（读/收藏筛选）和 `search`（布尔搜索）一并传给 DAO，前端"当前筛选下全部标读"与 DB 精确匹配
- **响应携带辅助数据**：read 响应返回标签与条目映射，bookmark 响应返回下一次切换所需的 URL + 图标，前端无需自行推断

### 6.3 前端 UI 层面
- **单事件委托**：所有 `.read` / `.bookmark` 链接（header / bottom / reader 视图）都绑定到同一 `mark_read` / `mark_favorite` 函数，行为完全统一
- **pending_entries 锁**：请求往返期间阻止重复点击
- **先 spinner 后真实切换**：用户操作得到即时视觉反馈，但真正的 class / URL / icon 切换只在服务器成功响应后执行（非乐观更新，避免状态回滚闪烁）
- **失败自动回滚**：HTTP 错误时把 spinner 换回原图标，class 不变
- **队列 + debounce**：自动标读（滚动 / 焦点 / 打开文章）1 秒窗口内合并为单次 HTTP，既减少请求数又避免逐条来回导致的闪烁
- **侧栏 / 标题 / favicon 联动**：`incUnreadsFeed()` 和 `incUnreadsTag()` 统一维护所有显示未读数的 DOM 节点，不依赖页面刷新
- **批量入口复用 URL 参数**：stream-footer 的 Mark all as read 按钮把当前 `get/state/search/idMax` 原样提交，后端使用与列表查询相同的过滤条件

### 6.4 视图渲染层面
- **单一数据源**：三套模板（entry_header / entry_bottom / article）都从 `$this->entry->isRead()` 和 `$this->entry->isFavorite()` 取状态，URL 生成逻辑在各模板中逐字相同，页面刷新时必然一致
- **class 语义化**：`not_read` 控制未读样式、`favorite` 控制收藏高亮，JS 与 CSS 共用相同 class 名，减少状态分叉
- **JS 上下文注入**：`javascript_vars.phtml` 把 PHP 配置（自动标读开关、快捷键、图标 URL）透传前端，前后端行为参数同源

---

## 七、完整调用链汇总

### 7.1 单篇切换已读
```
用户点击 <a.read>
  → main.js: init_stream() onclick 委托
  → main.js: mark_read(div, false, false)
    → pending_entries[flux_XXX]=true, 图标变 spinner
    → send_mark_read_queue([id], true/false)
      → POST .?c=entry&a=read  {ajax, _csrf, id:[id]}
        → entryController: readAction()
          → $entryDAO->markRead([id], $is_read)
            → EntryDAO: markRead(string)  [JOIN 原子更新 entry + feed.cache]
          → view: entry/read.phtml  →  JSON {tags:{...}}
      ← 成功响应
    → div 切换 not_read class, 所有 a.read href, 所有 a.read 图标
    → incUnreadsFeed() 更新侧栏/标题/favicon
    → delete pending_entries[...]
```

### 7.2 单篇切换收藏
```
用户点击 <a.bookmark>
  → main.js: mark_favorite(div)
    → pending_entries[flux_XXX]=true, 图标变 spinner
    → XHR POST a.bookmark.href  {ajax, _csrf}
      → entryController: bookmarkAction()
        → $entryDAO->markFavorite(id, is_favorite)
          → EntryDAO: markFavorite() [UPDATE _entry.is_favorite + lastUserModified]
          → HookType::EntriesFavorite 扩展钩子
        → view: entry/bookmark.phtml  →  JSON {url, icon}
    ← 成功响应
    → div 切换 favorite class, 所有 a.bookmark href, 所有 a.bookmark 图标
    → 侧栏收藏夹计数 ±1
    → delete pending_entries[...]
```

### 7.3 批量（分类/Feed/全部）标为已读
```
用户点击 #bigMarkAsRead 按钮
  → <form id="stream-footer"> 提交
    formaction URL 携带 get/nextGet/idMax/search/state/sort/order/from
  → entryController: readAction()
    → 无 id 参数 → switch($type_get)
      → markReadCat() 或 markReadFeed() 或 markReadEntries() 等
        → EntryDAO: 批量 UPDATE _entry.is_read
        → updateCacheUnreads(catId/feedId/null) 重算 feed 缓存
  → 非 AJAX: Minz_Request::good() 302 跳回 index → 整页刷新后所有状态重绘
```
