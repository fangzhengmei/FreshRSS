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

> **关键差异伏笔**：`markFavorite()` 没有对应的 `updateCacheFavorites()` 方法，因为 `_feed` 表根本没有 `cache_nbFavorites` 字段。收藏的总数和未读数在每次页面加载时通过 `COUNT(*)` 实时计算。

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

```php
public function bookmarkAction(): void {
    $id = Minz_Request::paramString('id', plaintext: true);
    $is_favourite = Minz_Request::paramTernary('is_favorite') ?? true;
    if ($id != '' && ctype_digit($id)) {
        $entryDAO = FreshRSS_Factory::createEntryDao();
        $entryDAO->markFavorite($id, $is_favourite);
    }
    if (!$this->ajax) {
        Minz_Request::forward(['c' => 'index', 'a' => 'index'], true);
    }
}
```

关键特征：
- **单篇限定**：`paramString` + `ctype_digit($id)` 强校验，确保是单个数字 ID。不接受数组（不像 `readAction` 用 `paramArrayString`）
- **无范围路由**：没有 `get` 参数分支，不支持分类 / Feed / 标签范围批量
- **无过滤透传**：没有 `state` / `search` / `idMax` 等过滤参数
- **响应简洁**：AJAX 模式下渲染 [entry/bookmark.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/entry/bookmark.phtml)，返回 JSON `{ url, icon }` —— 即告诉前端"下一次反向切换"需要的 URL 和图标 HTML

---

## 四、视图渲染层：列表回显

条目状态通过 View 层 **多处重复渲染**，但都使用同一套数据来源，保证一致性。

### 4.1 渲染源头：`FreshRSS_Entry::isRead() / isFavorite()`

所有模板都从 PHP 模型拿值：
- 若未读 → 链接 URL 不含 `is_read`（点击后标记为已读），图标为 `unread`
- 若已读 → 链接 URL 带 `&is_read=0`（点击后标记为未读），图标为 `read`
- 收藏同理，切换 `is_favorite` 参数和 `starred / non-starred` 图标

### 4.2 列表条目容器：class 与状态绑定

列表中每条文章的根 `<div>` 就把状态编码到 class 中，这是 CSS 样式和 JS 交互的共同基础：

[normal.phtml L79-L82](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/index/normal.phtml#L79-L82)：
```php
<div class="flux<?= !$this->entry->isRead() ? ' not_read' : ''
    ?><?= $this->entry->isFavorite() ? ' favorite' : ''
    ?>" id="flux_<?= $this->entry->id() ?>">
```

`not_read` class 控制未读高亮，`favorite` class 控制收藏高亮。两者互不干扰，独立切换。

### 4.3 列表头部按钮：`entry_header.phtml`

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

### 4.4 列表底部按钮：`entry_bottom.phtml`

[entry_bottom.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/helpers/index/normal/entry_bottom.phtml#L16-L35) 代码结构与 header 完全一致，class 同样为 `read` / `bookmark`。

### 4.5 阅读视图：`article.phtml`

[article.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/helpers/index/article.phtml#L15-L34) 在 Reader 视图的顶部、副标题、底部共三处渲染按钮，所有 `<a>` 都标记同一 class `read` / `bookmark`。

### 4.6 批量操作入口：`stream-footer.phtml`

[stream-footer.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/helpers/stream-footer.phtml#L13-L27) 渲染"全部标为已读"大按钮，其 `formaction` URL 将当前 `get / nextGet / idMax / search / state / sort / order` 全部打包提交给 `entry/readAction()`。

> **重要差异**：stream-footer 只有"全部标已读"按钮，**没有对应的"全部收藏"或"全部取消收藏"按钮**。收藏在列表底部没有批量入口。

### 4.7 前端注入 JS 上下文：`javascript_vars.phtml`

[javascript_vars.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/helpers/javascript_vars.phtml#L8-L101) 输出 JSON：
- `context.auto_mark_article / scroll / focus / site` —— 四种自动标读触发开关
- `context.csrf` —— AJAX 请求 token
- `context.icons.read / unread / spinner` —— 图标资源 URL
- `shortcuts.mark_read / mark_favorite` —— 键盘快捷键

> 注意：只有 `auto_mark_*` 四种自动标读开关，**没有自动收藏开关**。收藏不参与"自动触发"体系。

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

流程与 `mark_read` 对称但存在关键差异：

1. 同样 `pending_entries` 防重 + spinner 临时图标
2. **使用 XMLHttpRequest 而非 fetch** —— 代码演化遗留，功能等价但风格不统一
3. POST 到 `<a.bookmark>.href`（服务端动态生成的 URL 已决定方向）
4. 响应 JSON 包含 `url`（反向切换用的新 URL）和 `icon`（新图标 HTML）
5. 成功时：
   - `div.classList.toggle('favorite')`
   - `div.querySelectorAll('a.bookmark').forEach(a => a.href = json.url)` —— 全量替换所有收藏链接的 href
   - `div.querySelectorAll('a.bookmark > .icon').forEach(img => img.outerHTML = json.icon)` —— 全量替换所有图标
   - 侧栏收藏夹计数：`favourites.textContent.replace(..., incLabel(p1, inc, false))` —— 对括号内的数字 ±1
   - 若文章未读，同步更新 favorites 伪分类的 `data-unread` 属性
6. 失败回滚：恢复 `originalIcon.src/alt`

### 5.4 四类"自动标读"触发

`mark_read()` 的 `asBatch=true` 路径被多处调用，全部通过队列合并以减少 HTTP 请求：

| 触发点 | 位置 | 触发条件来自配置 |
|--------|------|-----------------|
| 打开文章 | [main.js L546-L551](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L546-L551) `toggleContent()` | `context.auto_mark_article` (`mark_when.article`) |
| 焦点移动 | [main.js L567-L569](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L567-L569) / [L585-L587](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L585-L587) | `context.auto_mark_focus` (`mark_when.focus`) |
| 滚动出屏 | [main.js L902-L915](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L902-L915) `onScroll()` | `context.auto_mark_scroll` (`mark_when.scroll`) |
| 点击原网站 | [main.js L1320-L1329](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L1320-L1329) 快捷键 | `context.auto_mark_site` (`mark_when.site`) |

> 收藏没有任何"自动触发"机制。收藏是纯用户主动行为。

### 5.5 批量标读：`mark_previous_read()`

[main.js L353-L358](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L353-L358)：快捷键 Alt+mark_read 触发，从当前文章向前遍历 `previousElementSibling`，逐条 `mark_read(div, true, true)` 全部入队列，由 debounce 合并为一次请求。

> 收藏没有对应的"批量收藏前面 N 篇"的快捷键或功能。

---

## 六、收藏与阅读状态的批量路径深度对比

这是两类状态切换中**最关键的差异**：底层 DAO 都支持数组批量写入，但上层 Web UI 对收藏的批量入口是**缺失的**。以下从五层逐一拆解。

### 6.1 DAO 层：都支持数组，但优化深度不同

#### `markRead()` 的数组路径 —— 三层策略

[EntryDAO.php L499-L570](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/EntryDAO.php#L499-L570) 根据数量走三种策略：

| 数量范围 | 策略 | 原因 |
|---------|------|------|
| 1~5 篇 | 循环调用单篇路径（`markRead(string)`） | 单篇路径用 JOIN 同时更新 feed.cache_nbUnreads，小批量下逐条增量更新比整表重算更快 |
| 6 ~ MAX_VARIABLE_NUMBER | 单条 `UPDATE ... WHERE id IN (...)` + 事后全量 `updateCacheUnreads(null, null)` | 批量越大，逐条 JOIN 的开销越显著，不如一次 UPDATE + 一次 COUNT 重算 |
| > MAX_VARIABLE_NUMBER | `array_chunk` 递归拆分 | 避免 SQL 参数数量上限 |

关键细节：
- 批量 SQL 带 `WHERE is_read<>?` 条件（L522），幂等防重复
- 批量后调用 `updateCacheUnreads(null, null)` 全量重算所有 feed 的未读缓存（L538-L540），用真实 `COUNT(*)` 兜底，杜绝算术累计误差
- 单篇路径则用 JOIN 原子写，零误差、零额外查询

#### `markFavorite()` 的数组路径 —— 单层策略

[EntryDAO.php L416-L451](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/EntryDAO.php#L416-L451) 只有一条路径：

1. 单篇 string 直接包成 `[$ids]`
2. 超量就 `array_chunk` 递归
3. 一律 `UPDATE ... WHERE id IN (...)`

与 `markRead()` 的关键差异：

| 对比项 | markRead | markFavorite |
|--------|----------|--------------|
| 单篇路径 | ✅ JOIN 原子更新 + feed 缓存增量 | ❌ 无单篇优化，统一走数组路径 |
| 批量幂等条件 | ✅ `WHERE is_read<>?` 防重复写 | ❌ 无条件，状态未变也会执行 UPDATE（rowCount=0） |
| 缓存更新 | ✅ JOIN 增量 / COUNT 重算两种策略 | ❌ 无 feed 级收藏缓存，不更新任何缓存 |
| 扩展钩子 | ❌ 无 | ✅ `Minz_HookType::EntriesFavorite`（批量时传整个 ids 数组） |

### 6.2 控制器层：阅读有范围批量，收藏只有单篇

#### `readAction()`：七类批量入口

[entryController.php L47-L222](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Controllers/entryController.php#L47-L222) 支持的批量维度：

| 类型 | `get` 前缀 | 对应 DAO 方法 | 场景 |
|------|-----------|--------------|------|
| 全部 | `a` / `A` / `Z` / `i` | `markReadEntries()` | 全部文章 / 主视图 / 归档 / 重要 |
| 收藏 | `s` | `markReadEntries(onlyFavorites=true)` | 只标记收藏夹内的 |
| 分类 | `c_*` | `markReadCat()` | 整个分类一键标读 |
| Feed | `f_*` | `markReadFeed()` | 单个订阅源一键标读 |
| 标签 | `t_*` / `T` | `markReadTag()` | 某个标签或所有标签 |
| 搜索结果 | （search 参数） | `markReadEntries(filters=...)` | 当前布尔搜索结果内全部标读 |
| 指定 ID 列表 | `id[]` 数组 | `markRead(array)` | 前端批量队列提交 |

所有批量入口都支持 `idMax` fail-safe 和 `state` 状态过滤，与列表查询使用完全相同的过滤条件。

#### `bookmarkAction()`：仅单篇

[entryController.php L232-L246](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Controllers/entryController.php#L232-L246)：

- `paramString` 而非 `paramArrayString`——**不接受数组**
- `ctype_digit($id)` 强校验——确保是单个数字 ID
- 没有 `get` 参数分支——不支持分类/Feed/标签范围批量
- 没有 `search` / `state` 过滤——无法按条件批量

**结论：Web 控制器层不存在"按分类/Feed/标签批量收藏"的入口。** `bookmarkAction` 的设计意图就是"一条一条切换"。

#### 为什么 Web UI 没有批量收藏入口？

产品逻辑上的原因：
1. **收藏是主动筛选行为**——用户逐条审阅后"加星"，不同于已读标记的"清扫"语义
2. **收藏没有未读缓存**——`_feed` 表没有 `cache_nbFavorites` 列，批量收藏后不需要重算任何缓存
3. **收藏夹是虚拟分类**——它没有独立的 feed/category 结构，无法像 `markReadCat` 那样通过 JOIN `_feed` 做范围标记

### 6.3 视图按钮层：阅读有"全部标读"按钮，收藏没有对应按钮

#### 阅读状态的三个批量入口

1. **底部流尾** —— [stream-footer.phtml L52-L64](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/helpers/stream-footer.phtml#L52-L64) 渲染 `#bigMarkAsRead` 大按钮，用户可配置显示为 big/small/none

2. **侧栏 Feed 下拉菜单** —— [aside_feed.phtml L218-L221](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/layout/aside_feed.phtml#L218-L221) 每个 feed 的配置菜单中有"标记此 feed 已读"按钮

3. **顶部导航** —— 标读按钮和下拉菜单（标当前 / 标一天前 / 标一周前 / 标记为未读）

#### 收藏状态没有对应的批量按钮

在所有视图文件中搜索 `bookmark` / `favorite` / `starred`，不存在任何形如"全部收藏/全部取消收藏"的按钮或表单入口。模板中收藏按钮只出现在单条 entry 内部：

- [entry_header.phtml L40-L48](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/helpers/index/normal/entry_header.phtml#L40-L48)
- [entry_bottom.phtml L28-L35](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/helpers/index/normal/entry_bottom.phtml#L28-L35)
- [article.phtml L33 / L69 / L147](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/helpers/index/article.phtml#L33)

### 6.4 前端 JS 层：阅读有队列合批，收藏纯单篇逐请求

| 特性 | `mark_read` | `mark_favorite` |
|------|------------|-----------------|
| 请求方式 | `fetch()` + async/await | `XMLHttpRequest` 即发即走 |
| 批量队列 | `mark_read_queue[]` + debounce 定时器（1秒） | 无队列 |
| 多篇合并 | `send_mark_read_queue([id1, id2, ...])` | 不支持 |
| 自动触发 | 4 种自动标读（打开/滚动/焦点/点击原网站） | 无自动触发 |
| 向前批量标读 | `mark_previous_read()` Alt+快捷键 | 无对应功能 |
| 响应驱动 UI | 服务器返回 `tags` 映射，前端批量更新 | 服务器返回 `url + icon`，前端逐条更新 |

[main.js L324-L351](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L324-L351) 中 `mark_read(div, only_not_read, asBatch)` 的第三个参数 `asBatch` 控制是否入队。当自动标读触发时 `asBatch=true`，1 秒窗口内多个条目合并为一次 `id[]` 请求发送。

[main.js L360-L440](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/scripts/main.js#L360-L440) 中 `mark_favorite(div)` 没有队列概念，每次点击立即发送 XHR。不存在"多篇文章逐个收藏合并为一次请求"的路径。

### 6.5 API 层：批量收藏能力已完整暴露

虽然 Web UI 没有批量收藏入口，但底层能力是完整的，且在 Google Reader 兼容 API 和 Fever API 中被使用：

#### GReader API —— `edit-tag` 端点

[greader.php L908-L955](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/api/greader.php#L908-L955) 的 `editTag()` 方法：

```php
// 添加 starred 标签（即收藏）
case 'user/-/state/com.google/starred':
    $entryDAO->markFavorite($e_ids, true);
    break;

// 移除 starred 标签（即取消收藏）
case 'user/-/state/com.google/starred':
    $entryDAO->markFavorite($e_ids, false);
    break;
```

其中 `$e_ids` 是从请求参数 `i=` 解析出的**条目 ID 数组**，可一次性传入多篇。这就是为什么 DAO 层 `markFavorite()` 必须支持数组——**为了兼容 Google Reader API 的批量编辑语义**，而非为 Web UI 准备。

#### Fever API —— `mark=item` 端点

[fever.php](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/p/api/fever.php) 的 mark 请求也支持 `with_ids` 参数传入逗号分隔的多个 ID，最终传给 `markFavorite(array|string $id, bool)`。

---

## 七、列表回显一致性机制深度剖析

尽管阅读和收藏在批量能力上存在巨大不对称，列表回显始终保持一致。以下从五个层面解释其原理。

### 7.1 第一层：服务端渲染是唯一真相源

[normal.phtml L79-L82](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/index/normal.phtml#L79-L82) / [reader.phtml L37-L40](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/index/reader.phtml#L37-L40) 中每个 flux `<div>` 的 class 完全由数据库当前值决定：

```php
<div class="flux<?= !$this->entry->isRead() ? ' not_read' : ''
    ?><?= $this->entry->isFavorite() ? ' favorite' : ''
    ?>..." id="flux_<?= $this->entry->id() ?>">
```

无论上一次操作是单篇还是批量、是 AJAX 还是整页刷新，每次页面加载都从数据库重新读取，CSS class 与数据库状态严格对齐。这是**最终一致性**的根本保障。

### 7.2 第二层：AJAX 操作后全量更新同一 flux 内所有按钮

`mark_read` 和 `mark_favorite` 成功后，都使用 `querySelectorAll` 对当前文章的 DOM 做全面同步更新：

- **阅读状态**：`div.classList.remove/addClass('not_read')` + `div.querySelectorAll('a.read')` 批量更新 href 和图标
- **收藏状态**：`div.classList.toggle('favorite')` + `div.querySelectorAll('a.bookmark')` 批量更新 href 和图标

`querySelectorAll` 确保同一 flux 内的 header / bottom / article 多处按钮同时被更新，不会出现"顶部改了底部没改"的情况。

> 💡 收藏的 URL 和图标都**不是前端根据当前状态推算**的，而是**由服务端响应 JSON 直接给出**（`json.url` + `json.icon`）。这样做的好处：
> - 前端不需要知道 URL 构造规则（`is_favorite=0` 的约定）
> - 服务端是唯一的状态仲裁者，避免前后端逻辑不一致
> - 如果以后 URL 规则变了（比如加 csrf 参数），前端无需改动

### 7.3 第三层：批量标读走整页刷新，绕过 DOM 一致性问题

stream-footer 和 nav_menu 的"全部标已读"按钮提交表单后，`readAction()` 对非 AJAX 请求返回 302 跳转，浏览器重新加载整个列表页。此时所有 flux 的 class / href / icon 全部由服务端重新渲染，不存在局部更新的问题。

这是一种"**重而可靠**"的策略——范围批量操作影响条目多、涉及缓存复杂，宁可靠整页刷新保证正确，也不用前端局部更新冒险出错。

### 7.4 第四层：侧栏计数器的双路径收敛

侧栏未读数和收藏数有两条更新路径，最终指向同一结果：

| 计数器 | 页面加载时（真相源） | AJAX 操作后（增量更新） |
|--------|---------------------|----------------------|
| Feed 未读数 | `feed.nbNotRead()` ← `_feed.cache_nbUnreads` | `incUnreadsFeed()` ±1 |
| 分类未读数 | `cat.nbNotRead()` ← 聚合 feed 缓存 | `incUnreadsFeed()` 找到 category 祖先 ±1 |
| 全部未读数 | `FreshRSS_Context::$total_unread` | `incUnreadsFeed()` 中 `.all .title` ±1 |
| 重要未读数 | `FreshRSS_Context::$total_important_unread` | `incUnreadsFeed()` 中 `.important .title` ±1（feed_priority>=20） |
| 收藏总数 | `FreshRSS_Context::$total_starred['all']` | `incLabel()` ±1 修改 `.favorites .title` 文本 |
| 收藏未读数 | `FreshRSS_Context::$total_starred['unread']` | `elem.setAttribute('data-unread', feed_unreads + inc)` |
| 标签未读数 | `tag.nbUnread()` 实时 COUNT | `incUnreadsTag()` ±1 |

增量更新（JS 侧）与全量计算（PHP 侧）在下次页面刷新时自然收敛。由于收藏只走单篇 AJAX，`±1` 增量不会出现批量场景下的算术误差；而批量标读后走整页刷新，计数器由服务端重新计算，也不存在不一致风险。

### 7.5 第五层：非乐观更新 + 失败回滚

两种状态切换都采用**非乐观更新**策略：

1. 点击后先替换为 spinner 图标给用户即时反馈
2. 发起 AJAX 请求
3. **服务器返回 200 且响应有效后**才真正切换 class / URL / 图标 / 计数器
4. 网络错误或服务器错误时，把 spinner 换回原图标，状态保持不变
5. `pending_entries[div.id]` 锁防止重复提交

这种"宁可让用户多等几百毫秒，也不出现先显示成功、失败后又闪回去"的策略，保证了 UI 状态与服务器状态严格一致。

### 7.6 为什么收藏没有批量入口也能保持一致？

因为**收藏的一致性问题天然更简单**：

- 只有单篇操作 → 没有"批量后部分成功部分失败"的复杂场景
- 没有自动触发 → 不会有"滚动时后台悄悄改了状态"与用户操作的冲突
- 没有 auto_remove → 条目始终留在 DOM 中，不存在"条目已移除但按钮未更新"的问题
- 单篇操作 + 非乐观更新 → 每一次操作都是原子的，成功或失败泾渭分明

---

## 八、不对称性总结表

| 维度 | 阅读状态（read） | 收藏状态（favorite） |
|------|-----------------|---------------------|
| DAO 批量能力 | ✅ string + array | ✅ string + array |
| DAO 缓存联动 | ✅ `cache_nbUnreads` 原子更新/重算 | ❌ 无 feed 级收藏缓存 |
| 控制器批量路由 | ✅ `get` 参数按范围路由（分类/Feed/标签/优先级/搜索） | ❌ 仅 `id` 单篇 |
| 控制器过滤透传 | ✅ `state/search/idMax` | ❌ 无 |
| 底部流尾批量按钮 | ✅ `bigMarkAsRead` | ❌ 无 |
| 侧栏 per-feed 标读 | ✅ "标此 feed 已读" | ❌ 无 |
| 前端队列合并 | ✅ debounce 队列（1秒） | ❌ 即发即走 |
| 自动触发 | ✅ 4 种自动标读（打开/滚动/焦点/点击原网站） | ❌ 无 |
| 向前批量快捷键 | ✅ Alt+mark_read → `mark_previous_read()` | ❌ 无 |
| API 批量入口 | ✅ GReader `edit-tag` + Fever `mark` | ✅ GReader `edit-tag` + Fever `mark` |
| 过滤规则自动设置 | ✅ `read` 动作 | ✅ `star` 动作（入库时） |
| AJAX 请求方式 | `fetch()` + async/await | `XMLHttpRequest` |
| AJAX 后 DOM 更新 | `querySelectorAll('a.read')` 全量替换 | `querySelectorAll('a.bookmark')` 全量替换 |
| 侧栏计数器更新 | `incUnreadsFeed()` + `incUnreadsTag()` | `incLabel()` + data-unread 属性 |
| 整页刷新兜底 | ✅ 302 跳转 | ✅ 302 跳转 |
| 失败回滚 | ✅ spinner 换回原图 + 状态不变 | ✅ spinner 换回原图 + 状态不变 |
| DOM 移除机制 | ✅ `auto_remove_article` 可选 | ❌ 无 |

---

## 九、完整调用链汇总

### 9.1 单篇切换已读
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

### 9.2 单篇切换收藏
```
用户点击 <a.bookmark>
  → main.js: init_stream() onclick 委托
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
    → 若未读，同步更新 favorites 伪分类 data-unread
    → delete pending_entries[...]
```

### 9.3 批量（分类/Feed/全部）标为已读
```
用户点击 #bigMarkAsRead 按钮 或 侧栏 feed 标读按钮
  → <form id="stream-footer"> / <form id="mark-read-aside"> 提交
    formaction URL 携带 get/nextGet/idMax/search/state/sort/order/from
  → entryController: readAction()
    → 无 id 参数 → switch($type_get)
      → markReadCat() 或 markReadFeed() 或 markReadEntries() 等
        → EntryDAO: 批量 UPDATE _entry.is_read
        → updateCacheUnreads(catId/feedId/null) 重算 feed 缓存
  → 非 AJAX: Minz_Request::good() 302 跳回 index → 整页刷新后所有状态重绘
```

### 9.4 GReader API 批量收藏（第三方客户端）
```
移动 App 调用 edit-tag
  → POST /api/greader.php/reader/api/0/edit-tag
    Body: i=id1&i=id2&i=id3&a=user/-/state/com.google/starred
  → greader.php: editTag()
    → $entryDAO->markFavorite([id1, id2, id3], true)
      → EntryDAO: markFavorite(array)  [UPDATE ... WHERE id IN (?,?,?)]
      → HookType::EntriesFavorite 扩展钩子（传整个 ids 数组）
  → 返回 OK
```
