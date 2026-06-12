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

> **关键差异伏笔**：`markFavorite()` 没有对应的 `updateCacheFavorites()` 方法，因为 `_feed` 表根本没有 `cache_nbFavorites` 字段。侧栏显示的收藏总数和未读数在每次页面加载时通过 `COUNT(*)` 实时计算，**且仅统计 `priority > -10` 的非隐藏来源**（详见第十节）。

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

---

## 十、侧栏统计范围深度剖析：隐藏来源是否被包含？

侧栏各个计数器的统计范围**并不统一**，取决于优先级阈值和数据来源。优先级常量定义于 [Feed.php L38-L44](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Feed.php#L38-L44)：

```php
public const PRIORITY_IMPORTANT   = 20;
public const PRIORITY_MAIN_STREAM = 10;
public const PRIORITY_CATEGORY    = 0;
public const PRIORITY_FEED        = -5;
public const PRIORITY_HIDDEN      = -10;
```

### 10.1 收藏总数与未读数：排除隐藏来源

**`countUnreadReadFavorites()`** — [EntryDAO.php L2019-L2034](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/EntryDAO.php#L2019-L2034)

```sql
SELECT
    COUNT(*) AS total,
    COUNT(CASE WHEN e.is_read = 0 THEN 1 END) AS unread
FROM `_entry` e
JOIN `_feed` f ON e.id_feed = f.id
WHERE e.is_favorite = 1 AND f.priority > :priority
```
`:priority = PRIORITY_HIDDEN (-10)`，所以条件是 `f.priority > -10`。

**结论**：隐藏来源（priority=-10）**不**包含在侧栏收藏统计中。只有 `priority > -10`（即 PRIORITY_FEED 及以上）的 feed 的收藏文章会被计入。

这个结果在 [Context.php L243](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Context.php#L243) 被赋值给 `FreshRSS_Context::$total_starred`，用于：
- 侧栏"收藏夹"标题数字 `'favorites (123)'` — [aside_feed.phtml L61](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/layout/aside_feed.phtml#L61)
- 侧栏收藏夹的 `data-unread` 属性 — [aside_feed.phtml L58-L60](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/layout/aside_feed.phtml#L58-L60)
- 进入收藏视图时的 `$get_unread` — [Context.php L496](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Context.php#L496)

### 10.2 全局未读数：只统计主流来源

**`FreshRSS_Category::countUnread()`** — [Category.php L339-L345](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Category.php#L339-L345)

```php
self::$total_unread = FreshRSS_Category::countUnread(
    self::categories(),
    FreshRSS_Feed::PRIORITY_MAIN_STREAM  // minPriority = 10
);
```

内部遍历每个 category，调用 `$category->nbNotRead($minPriority)` — [Category.php L95-L114](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Category.php#L95-L114)：

```php
foreach ($this->feeds as $feed) {
    if ($feed->priority() >= $minPriority) {  // >= 10
        $nb += $feed->nbNotRead();
    }
}
```

**结论**：全局未读数只统计 `priority >= 10`（PRIORITY_MAIN_STREAM 及以上）的 feed，隐藏来源（-10）和分类级（0）、feed 级（-5）都不包含。

同理，`self::$total_important_unread` 使用 `PRIORITY_IMPORTANT (20)`，只统计重要 feed。

### 10.3 单个 Feed 未读数：不做优先级过滤

**`Feed::nbNotRead()`** — [Feed.php L413-L420](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Feed.php#L413-L420)

```php
public function nbNotRead(): int {
    if ($this->nbNotRead < 0) {
        $feedDAO = FreshRSS_Factory::createFeedDao();
        $this->nbNotRead = $feedDAO->countNotRead($this->id());
    }
    return $this->nbNotRead;
}
```

`countNotRead($id)` 直接按 feed_id 统计，**没有** feed.priority 过滤。

**结论**：当用户点击某个隐藏 feed 查看时，其未读数仍会正确显示。但该 feed 的未读数不会被累加到全局或分类未读数中。

### 10.4 标签未读数：包含隐藏来源

**`TagDAO::countNotRead()`** — [TagDAO.php L286-L300](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/TagDAO.php#L286-L300)

```sql
SELECT COUNT(*) AS count FROM `_entrytag` et
INNER JOIN `_entry` e ON et.id_entry=e.id
WHERE e.is_read=0
[AND et.id_tag=:id_tag]
```

SQL 只 JOIN `_entry`，不 JOIN `_feed`，因此**没有** feed.priority 过滤条件。

**结论**：标签未读数统计包含所有来源，包括隐藏来源的文章。如果用户给隐藏来源的文章打了标签，其未读数会出现在标签统计中。

### 10.5 前端 JS 未读数映射：排除隐藏来源

**`nbUnreadsPerFeed.phtml`** — [nbUnreadsPerFeed.phtml L11-L12](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/javascript/nbUnreadsPerFeed.phtml#L11-L12)

```php
if ($feed->priority() > FreshRSS_Feed::PRIORITY_HIDDEN) {
    $result['feeds'][$feed->id()] = $feed->nbNotRead();
}
```

**结论**：前端 JS 拿到的 feed 未读数映射表（供 `incUnreadsFeed()` 使用）也排除了隐藏来源，与侧栏显示保持一致。

### 10.6 各计数器统计范围汇总表

| 计数器 | 统计方法 / 来源 | 优先级过滤条件 | 包含隐藏来源？ | 与对应列表查询是否一致 |
|--------|----------------|---------------|---------------|----------------------|
| 收藏总数 `total_starred['all']` | `countUnreadReadFavorites()` → `JOIN _feed WHERE f.priority > -10` | `> PRIORITY_HIDDEN (-10)` | ❌ 不包含 | ✅ 一致（`sqlListWhere('s')` 也用 `> -10`） |
| 收藏未读数 `total_starred['unread']` | 同上 | 同上 | ❌ 不包含 | ✅ 一致 |
| 全局未读数 `total_unread` | `Category::countUnread(categories, PRIORITY_MAIN_STREAM)` | `>= PRIORITY_MAIN_STREAM (10)` | ❌ 不包含 | ⚠️ 部分一致（主视图 `a` 一致，但 `A`/`Z` 视图列表更宽，`get_unread` 仍用此值） |
| 重要未读数 `total_important_unread` | `Category::countUnread(categories, PRIORITY_IMPORTANT)` | `>= PRIORITY_IMPORTANT (20)` | ❌ 不包含 | ✅ 一致（`sqlListWhere('i')` 也用 `>= 20`） |
| 分类未读数 `cat.nbNotRead()` | 默认 `minPriority = PRIORITY_FEED (-5)`，有两条路径：<br>1. feeds 未加载 → `CategoryDAO::countNotRead(id, PRIORITY_FEED)` SQL 查询<br>2. feeds 已加载 → PHP 循环 `>= -5` 累加 | `>= PRIORITY_FEED (-5)`（等效于 `> PRIORITY_HIDDEN`） | ❌ 不包含 | ⚠️ **不一致**（分类视图列表 `sqlListWhere('c')` 用 `>= 0`，比未读数统计更严格） |
| Feed 未读数 `feed.nbNotRead()` | `cache_nbUnreads` 字段，按 feed_id 直接统计 | 无（不判断自身优先级） | ✅ 包含 | ✅ 一致（进入该 feed 时直接显示） |
| 单标签未读数 `tag.nbUnread()` | `TagDAO::countNotRead(id)` → `_entrytag JOIN _entry` | 无（不 JOIN feed） | ✅ 包含 | ✅ 一致（`sqlListWhere('t')` 也无优先级过滤） |
| 全标签未读数 | 同上，不带 id 参数 | 无（不 JOIN feed） | ✅ 包含 | ✅ 一致（`sqlListWhere('T')` 也无优先级过滤） |
| 前端 JS feed 映射 | `nbUnreadsPerFeed.phtml` 中 `f.priority > -10` 过滤 | `> PRIORITY_HIDDEN (-10)` | ❌ 不包含 | ✅ 与侧栏显示一致 |
| 标签列表总数 | `TagDAO::countAll()` 或类似 | 无（不 JOIN feed） | ✅ 包含 | ✅ 一致 |

> **关键不对称**：收藏统计始终排除隐藏来源，标签统计始终包含隐藏来源。这是因为收藏与 feed 优先级强绑定（收藏的"可见性"取决于来源优先级），而标签是独立于 feed 的元数据体系。

> ⚠️ **注意 `A`/`Z` 视图的 `get_unread` 偏差**：全局 `total_unread` 只统计主流（>=10），但 `A` 视图列表包含分类级（>=0）、`Z` 视图包含所有来源。因此在 `A`/`Z` 视图下，标题栏和侧栏的未读数可能与列表实际未读条目数不一致。这是已知的设计取舍，`total_unread` 作为"主要"数字只反映主流内容。

---

## 十一、进入收藏/标签视图时的默认状态自动调整

进入不同视图时，Context 会**自动调整 `$state` 位掩码**，导致默认显示的读/收藏状态发生变化。这个逻辑集中在 [Context.php L250-L262](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Context.php#L250-L262)。

### 11.1 状态调整的执行顺序

```php
// Step 1: 先从请求参数取 state，或使用用户默认配置
self::$state = Minz_Request::paramInt('state') 
    ?: FreshRSS_Context::userConf()->default_state;

// Step 2: 判断用户是否显式指定了 state（URL 中有 ?state=...）
$state_forced_by_user = Minz_Request::paramString('state', plaintext: true) !== '';

// Step 3: 只有用户没有显式指定时，才自动调整
if (!$state_forced_by_user) {
    // 自动调整逻辑...
}
```

**关键**：如果用户在 URL 中显式指定了 `?state=...`，自动调整逻辑会被跳过，完全尊重用户选择。

### 11.2 自动调整规则

```php
if (!$state_forced_by_user) {
    if (FreshRSS_Context::userConf()->show_fav_unread 
        && (self::isCurrentGet('s') || self::isCurrentGet('T') || self::isTag())) {
        // Case 1: 收藏视图、所有标签视图、单个标签视图
        // + 配置"始终显示收藏夹全部内容"开启
        self::$state = FreshRSS_Entry::STATE_NOT_READ | FreshRSS_Entry::STATE_READ;
    } elseif (FreshRSS_Context::userConf()->default_view === 'all') {
        // Case 2: 默认视图配置为"显示全部"
        self::$state = STATE_NOT_READ | STATE_READ;
    } elseif (FreshRSS_Context::userConf()->default_view === 'unread_or_favorite') {
        // Case 3: 默认视图配置为"显示未读或收藏"
        self::$state = STATE_OR_NOT_READ | STATE_OR_FAVORITE;
    } elseif (FreshRSS_Context::userConf()->default_view === 'adaptive' && self::$get_unread <= 0) {
        // Case 4: 自适应视图且当前没有未读
        self::$state = STATE_NOT_READ | STATE_READ;
    }
}
```

### 11.3 `show_fav_unread` 配置的影响

`show_fav_unread`（"始终显示收藏夹全部内容"）在 [reading.phtml L152-L156](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/configure/reading.phtml#L152-L156) 中配置，帮助文本明确说明："同样适用于标签"。

**当配置为 true 时**：

| 当前视图 | 自动设置的 state | 显示效果 |
|---------|-----------------|----------|
| 收藏夹 (`?get=s`) | `STATE_NOT_READ \| STATE_READ = 3` | 显示全部（已读+未读）收藏 |
| 所有标签 (`?get=T`) | 同上 | 显示全部（已读+未读）带标签文章 |
| 单个标签 (`?get=t_123`) | 同上 | 显示该标签下全部文章 |

**为什么这样设计？**

收藏和标签的定位是"知识库"而非"收件箱"。用户进入这些视图通常是想查阅所有已保存的内容，而不是仅看未读。因此当 `show_fav_unread=true` 时，进入这些视图会**自动忽略 `default_view` 配置**，强制显示全部。

### 11.4 进入收藏视图时的额外强制位

除了上述 state 调整，进入收藏视图 `_get('s')` 时还有额外处理 — [Context.php L492-L498](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Context.php#L492-L498)：

```php
case 's':
    self::$current_get['starred'] = true;
    self::$get_unread = self::$total_starred['unread'];
    // Update state if favorite is not yet enabled.
    self::$state = self::$state | FreshRSS_Entry::STATE_FAVORITE;
    break;
```

无论用户如何配置，进入收藏视图时都会**强制 `$state |= STATE_FAVORITE (4)`**，确保 SQL 查询的 WHERE 条件包含 `is_favorite=1`。这与 `sqlListWhere('s')` 中的硬编码条件 `AND e.is_favorite=1` 形成双重保险。

### 11.5 状态变化对列表回显的影响链

```
用户点击侧栏"收藏夹"链接
  → URL: .?get=s
  → Context::updateUsingRequest()
    → _get('s') → $state |= STATE_FAVORITE (强制只看收藏)
    → 若 show_fav_unread=true 且用户未指定 ?state=
      → $state = STATE_NOT_READ | STATE_READ (显示全部)
    → 最终 $state = 1 | 2 | 4 = 7 (STATE_FAVORITE | STATE_ALL)
  → EntryDAO::listByType('s', $state)
    → sqlListWhere('s') → WHERE e.is_favorite=1 AND f.priority > -10
    → sqlListEntriesWhere($state=7)
      → STATE_ANDS (7 & 15 = 7) 不为 0
      → STATE_FAVORITE (4) 已设，STATE_NOT_FAVORITE (8) 未设 → AND is_favorite=1
      → STATE_NOT_READ (2) 和 STATE_READ (1) 同时为 true → 不生成 is_read 条件
      → 最终 WHERE: is_favorite=1（显示全部收藏）
  → 列表渲染时
    → flux div 渲染 not_read / favorite class 基于数据库真实值
    → 按钮 href 基于 $this->entry->isRead() 和 isFavorite()
```

---

## 十二、SQL 查询拼接：状态位如何影响最终列表回显

状态位掩码 `$state` 通过两个函数翻译成实际 SQL：**`sqlListWhere()`** 处理视图类型（get 参数），**`sqlListEntriesWhere()`** 处理状态位和搜索过滤。两者串联决定最终返回哪些条目。

### 12.1 Feed 优先级阈值：`sqlListWhere()` 中的可见性控制

[EntryDAO.php L1535-L1590](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/EntryDAO.php#L1535-L1590) 根据视图类型设置 feed 优先级下限：

| `type` | 视图 | 优先级条件 | 说明 |
|--------|------|-----------|------|
| `'a'` | 全部（主流） | `f.priority >= 10` | `> 0` | 只显示主流及以上 |
| `'A'` | 全部（含分类） | `f.priority >= min(CATEGORY(0), needVisibility)` | 显示分类级及以上 |
| `'Z'` | 全部（含隐藏） | `1=1`（无限制） | 显示所有，包括隐藏 |
| `'i'` | 重要 | `f.priority >= min(IMPORTANT(20), needVisibility)` | 只显示重要来源 |
| `'s'` | 收藏 | `f.priority > min(HIDDEN(-10), needVisibility)` <br> **AND** `e.is_favorite=1` | 排除隐藏来源的收藏 |
| `'c_*'` | 分类 | `f.priority >= min(CATEGORY(0), needVisibility)` <br> **AND** `f.category=?` | 分类内分类级及以上 |
| `'f_*'` | Feed | `e.id_feed=?` | 不判断优先级（直接按 id） |
| `'t_*'` | 单个标签 | `et.id_tag=?` | 不判断优先级（直接按标签关联） |
| `'T'` | 所有标签 | `1=1`（无限制） | 不判断优先级 |

`needVisibility` 来自搜索条件 [Search.php L538-L548](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Search.php#L538-L548)：
- 搜索包含 feed ID → `PRIORITY_HIDDEN (-10)`（允许访问隐藏来源）
- 搜索包含 category ID → `PRIORITY_CATEGORY (0)`
- 否则 → `PRIORITY_IMPORTANT (20)`

**关键结论**：当用户通过搜索直接指定某个隐藏 feed 时，优先级限制会被放宽，用户可以看到该隐藏 feed 的内容。这是一个"显式选择即可见"的设计。

### 12.2 状态位转 SQL：`sqlListEntriesWhere()`

[EntryDAO.php L1394-L1519](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/EntryDAO.php#L1394-L1519) 把 `$state` 位掩码翻译成 WHERE 条件。

#### ANDS 位（bits 1-8，值 1-15）处理：

```php
if ($state & STATE_ANDS) {  // 15 = 0b01111
    if ($state & STATE_NOT_READ) {  // bit 2
        if (!($state & STATE_READ)) {  // 只设 NOT_READ，不设 READ
            $search .= 'AND (is_read=0) ';  // 只显示未读
        }
        // 同时设 NOT_READ 和 READ → 不生成条件（显示全部）
    } elseif ($state & STATE_READ) {  // 只设 READ，不设 NOT_READ
        $search .= 'AND (is_read=1) ';  // 只显示已读
    }

    if ($state & STATE_FAVORITE) {  // bit 4
        if (!($state & STATE_NOT_FAVORITE)) {
            $search .= 'AND (is_favorite=1) ';  // 只显示收藏
        }
    } elseif ($state & STATE_NOT_FAVORITE) {  // bit 8
        $search .= 'AND (is_favorite=0) ';  // 只显示未收藏
    }
}
```

#### ORS 位（bits 6-7，值 32-64）处理：

```php
if ($state & STATE_ORS) {  // 96 = 0b1100000
    if ($state & STATE_OR_NOT_READ) {  // bit 6 = 32
        $search = rtrim($search, ') ');
        $search .= ' OR is_read=0) ';   // 加上"或未读"
    }
    if ($state & STATE_OR_FAVORITE) {   // bit 7 = 64
        $search = rtrim($search, ') ');
        $search .= ' OR is_favorite=1) ';  // 加上"或收藏"
    }
}
```

### 12.3 常见状态组合对应的 SQL

| state 值 | 常量组合 | ANDS 条件 | ORS 附加 | 最终效果 |
|----------|---------|-----------|----------|----------|
| 2 | `STATE_NOT_READ` | `is_read=0` | - | 只显示未读 |
| 3 | `STATE_NOT_READ \| STATE_READ` | -（互斥抵消） | - | 显示全部 |
| 6 | `STATE_NOT_READ \| STATE_FAVORITE` | `is_read=0 AND is_favorite=1` | - | 只显示未读收藏 |
| 7 | `STATE_ALL \| STATE_FAVORITE` | `is_favorite=1` | - | 显示全部收藏 |
| 96 | `STATE_ORS`（无 ANDS） | - | `(1=0 OR ...)` 占位 | 无有效条件 |
| 32 | `STATE_OR_NOT_READ` | - | `(1=0 OR is_read=0)` | 只显示未读（同 state=2） |
| 64 | `STATE_OR_FAVORITE` | - | `(1=0 OR is_favorite=1)` | 只显示收藏 |
| 96 | `STATE_OR_NOT_READ \| STATE_OR_FAVORITE` | - | `(1=0 OR is_read=0 OR is_favorite=1)` | 显示未读 **或** 收藏 |

> **注意 state=96 (`STATE_OR_NOT_READ | STATE_OR_FAVORITE`) 的特殊语义**：这是"未读或收藏"视图，用 OR 连接条件。即使一篇文章已读，只要它被收藏，也会显示。这与 state=6（未读 AND 收藏）的交集语义不同。

### 12.4 完整 SQL 拼接流程

以进入收藏视图（`?get=s`）且 `show_fav_unread=true` 为例：

```php
// Context 层
$state = userConf()->default_state;  // 假设用户默认 state=2（只看未读）
$state_forced_by_user = false;  // URL 中无 ?state=
if (show_fav_unread && isCurrentGet('s')) {
    $state = STATE_NOT_READ | STATE_READ;  // = 3，显示全部
}
_get('s') → $state |= STATE_FAVORITE;  // = 3 | 4 = 7

// DAO 层 sqlListWhere('s')
$where = "f.priority > -10 AND e.is_favorite=1 ";

// DAO 层 sqlListEntriesWhere($state=7)
$search = "AND (is_favorite=1) ";  // 来自 STATE_FAVORITE 位
// STATE_NOT_READ 和 STATE_READ 同时为 true，不生成 is_read 条件

// 最终 SQL
SELECT e.id FROM `_entry` e USE INDEX (entry_feed_read_index)
INNER JOIN `_feed` f ON f.id = e.id_feed
WHERE f.priority > -10 AND e.is_favorite=1 AND (is_favorite=1)
ORDER BY e.id DESC
```

注意 `e.is_favorite=1` 条件出现了两次——一次来自 `sqlListWhere('s')` 的视图类型硬编码，一次来自 `sqlListEntriesWhere()` 的 STATE_FAVORITE 位。数据库优化器会自动合并为等价条件，不影响执行计划。

### 12.5 列表回显一致性的闭环

状态条件在整条链路中**同源、同构**，确保了列表显示与用户预期一致：

1. **渲染侧栏链接时**：`_url('index', $view, 'get', 's')` 生成 `?get=s`，不携带 state 参数，让自动调整逻辑生效
2. **Context 层计算 state 时**：基于相同的 `get` 参数和 `show_fav_unread` 配置
3. **DAO 层拼接 SQL 时**：基于同一个 `$state` 值
4. **模板渲染按钮 href 时**：基于 `$this->entry->isRead()` / `isFavorite()`（即数据库当前值）
5. **JS 增量更新时**：`incLabel()` 和 `incUnreadsTag()` 只修改数字文本，不改变列表内容

下次页面刷新时，第 1-4 步重新执行，所有状态从数据库重新读取，完成一致性闭环。

---

## 十三、收藏视图与标签视图在隐藏来源上的关键不对称

### 13.1 收藏视图排除隐藏来源，标签视图包含隐藏来源

这是两个伪分类最重要的差异，贯穿侧栏统计、列表查询、批量操作三层。

**收藏视图 `get=s`** — [EntryDAO.php L1560-L1564](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/EntryDAO.php#L1560-L1564)：

```php
case 's':
    $where .= 'f.priority > ' . min(FreshRSS_Feed::PRIORITY_HIDDEN, ...) . ' ';
    $where .= 'AND e.is_favorite=1 ';
    break;
```

SQL JOIN `_feed`，并强制 `f.priority > -10`。隐藏来源的收藏文章**不会**出现在列表中。

**标签视图 `get=t_*` / `get=T`** — [EntryDAO.php L1578-L1584](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/EntryDAO.php#L1578-L1584)：

```php
case 't':
    $where .= 'et.id_tag=? ';    // 只按标签 ID 过滤
    break;
case 'T':
    $where .= '1=1 ';            // 所有带标签的文章
    break;
```

SQL JOIN `_entrytag`，但**不 JOIN `_feed`**（不需要 feed 信息），也没有 `f.priority` 条件。隐藏来源的带标签文章**会**出现在列表中。

**为什么不同？**

收藏与 feed 是天然绑定的——`_entry.is_favorite` 是 entry 级字段，但收藏的"可见性"取决于该 entry 所属 feed 的优先级。标签则完全独立于 feed 层级——`_entrytag` 关联只涉及 entry 和 tag 两张表，不存在"标签下的文章来自哪个 feed"的语义约束。因此标签视图不做 feed 优先级过滤。

### 13.2 侧栏统计与列表查询的对称性验证

| 视图 | 侧栏统计方法 | 列表查询方法 | 优先级过滤 | 对称？ |
|------|------------|------------|-----------|--------|
| 收藏 `s` | `countUnreadReadFavorites()` → `JOIN _feed WHERE f.priority > -10` | `sqlListWhere('s')` → `f.priority > -10 AND e.is_favorite=1` | 都 `> -10` | ✅ |
| 标签 `t_*` | `TagDAO::countNotRead($id)` → `JOIN _entry WHERE is_read=0` | `sqlListWhere('t')` → `et.id_tag=?` | 都无过滤 | ✅ |
| 全标签 `T` | `TagDAO::countNotRead()` → 同上 | `sqlListWhere('T')` → `1=1` | 都无过滤 | ✅ |

**结论**：虽然收藏和标签对隐藏来源的处理逻辑不同，但**各自的侧栏统计与列表查询是严格对称的**。侧栏的数字永远等于进入对应视图后实际能看到的条目数。

### 13.3 边缘场景：隐藏来源的文章同时被打标签和被收藏

假设一篇隐藏来源的文章同时被打标签和被收藏：

| 操作 | 侧栏收藏数 | 侧栏标签未读数 | 收藏视图 | 标签视图 |
|------|-----------|-------------|---------|---------|
| 该文章被收藏 | ❌ 不变（排除隐藏来源） | — | ❌ 看不到 | — |
| 该文章被打标签 | — | ✅ +1（包含隐藏来源） | — | ✅ 能看到 |
| 在标签视图标为已读 | — | ✅ -1 | — | ✅ 显示为已读 |
| 在 `Z` 视图搜 `is:starred` | — | — | — | —（搜索不走 `s` 视图） |
| 在 `Z` 视图直接看到并取消收藏 | ✅ 无变化（本就不在统计中） | — | ❌ 仍然看不到 | — |

**关键观察**：隐藏来源的收藏文章在 `get=s` 下既看不到也无法操作。用户只能在 `get=Z` 或直接访问文章链接时才能管理这些收藏。这是刻意的设计——隐藏来源的文章不应出现在"正常"视图中，即使被收藏。

---

## 十四、侧栏渲染的隐藏与显示逻辑

侧栏不仅决定**显示什么数字**，还决定**哪些 feed/category 节点可见**、**数字徽章是否显示**。

### 14.1 `hide_read_feeds`：隐藏已读完的 feed 节点

[aside_feed.phtml L6-L9](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/layout/aside_feed.phtml#L6-L9)：

```php
if (FreshRSS_Context::userConf()->hide_read_feeds &&
    (FreshRSS_Context::isStateEnabled(FreshRSS_Entry::STATE_NOT_READ) ||
     FreshRSS_Context::isStateEnabled(FreshRSS_Entry::STATE_OR_NOT_READ)) &&
    !FreshRSS_Context::isStateEnabled(FreshRSS_Entry::STATE_READ)) {
    $class = ' state_unread';
}
```

三个条件同时满足时，`<nav>` 获得额外 class `state_unread`：
1. 用户开启 `hide_read_feeds` 配置
2. 当前 state 过滤**包含**未读（`STATE_NOT_READ` 或 `STATE_OR_NOT_READ` 位被设）
3. 当前 state 过滤**不包含**已读（`STATE_READ` 位未设）

CSS 规则 `.state_unread .feed[data-unread="0"]` 会隐藏 `data-unread=0` 的 feed 节点。

**进入收藏/标签视图时的联动**：当 `show_fav_unread=true` 时，进入收藏/标签视图会自动设置 `state = STATE_NOT_READ | STATE_READ`，此时条件 3 不满足（`STATE_READ` 已设），因此 `state_unread` class **不会被添加**——所有 feed 节点都保持可见。这是合理的：收藏/标签视图显示全部文章，侧栏也应展示所有 feed 供导航。

当 `show_fav_unread=false` 且 `default_view='adaptive'` 时，进入收藏视图的 state 仍然是 `STATE_NOT_READ`（条件 3 满足），但收藏视图本身不按 feed 展示，侧栏 feed 节点的隐藏不影响内容区域。

### 14.2 `show_unread_count`：控制数字徽章的三级显示

[aside_feed.phtml L23-L24](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/layout/aside_feed.phtml#L23-L24)：

```php
$hideSucGlobal = userConf()->show_unread_count !== 'all' ? ' data-unread-hide="1"' : '';
$hideSucImportant = userConf()->show_unread_count !== 'none' ? '' : ' data-unread-hide="1"';
```

| `show_unread_count` 值 | 主流/全部/收藏/标签 数字 | 重要 feed 数字 |
|------------------------|----------------------|--------------|
| `'all'` | ✅ 显示 | ✅ 显示 |
| `'important'` | ❌ 隐藏 | ✅ 显示 |
| `'none'` | ❌ 隐藏 | ❌ 隐藏 |

per-feed 和 per-category 的 `show_unread_count` 覆盖：
- [Feed.php L321-L330](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Feed.php#L321-L330)：`$feed->showUnreadCount()` 检查 feed 自身属性 → category 属性 → 全局配置
- [aside_feed.phtml L109/L148](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/layout/aside_feed.phtml#L109)：`$hideSucCat = $cat->showUnreadCount() ? '' : ' data-unread-hide="1"'`

**收藏夹伪分类的数字**使用 `$hideSucGlobal`（全局规则），不受 per-feed/per-category 覆盖影响——因为它不是真正的 feed/category。

### 14.3 侧栏收藏夹的特殊渲染

[aside_feed.phtml L57-L63](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/layout/aside_feed.phtml#L57-L63)：

```php
<li class="tree-folder category favorites<?= FreshRSS_Context::isCurrentGet('s') ? ' active' : '' ?>">
    <a class="tree-folder-title" data-unread="<?= format_number(FreshRSS_Context::$total_starred['unread']) ?>"<?=
        $hideSucGlobal ?> href="<?= _url('index', $actual_view, 'get', 's') . $state_filter_manual ?>">
        <?= _i('starred') ?><span class="title" data-unread="<?= format_number(FreshRSS_Context::$total_starred['unread']) ?>"<?=
            $hideSucGlobal ?>><?= _t('index.menu.favorites', format_number(FreshRSS_Context::$total_starred['all'])) ?></span>
    </a>
</li>
```

关键细节：
- `data-unread` 使用 `$total_starred['unread']`（收藏中的未读数）
- 标签文本使用 `_t('index.menu.favorites', $total_starred['all'])`（收藏总数）
- `<a>` 和 `<span>` 都有独立的 `data-unread` 属性——JS `incLabel()` 更新 `.favorites .title` 的 `data-unread` 和文本
- 链接 href 不带 `state` 参数（使用 `$state_filter_manual`，不含自动调整后的 state），让 Context 自动调整逻辑在点击时生效
- 收藏夹没有子节点（不像 category 有 feed 列表），是扁平的"虚拟分类"

### 14.4 `Category::feeds()` 初始化时的 `nbNotRead` 过滤

[Category.php L131-L147](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/Models/Category.php#L131-L147)：

```php
public function feeds(): array {
    if ($this->feeds === null) {
        $feedDAO = FreshRSS_Factory::createFeedDao();
        $this->feeds = $feedDAO->listByCategory($this->id());
        $this->nbFeeds = 0;
        $this->nbNotRead = 0;
        foreach ($this->feeds as $feed) {
            $this->nbFeeds++;
            if ($feed->priority() > FreshRSS_Feed::PRIORITY_HIDDEN) {
                $this->nbNotRead += $feed->nbNotRead();
            }
        }
    }
    return $this->feeds ?? [];
}
```

**关键**：`Category::feeds()` 自动加载时，累加 `nbNotRead` 的条件是 `$feed->priority() > PRIORITY_HIDDEN`（即 `> -10`）。这意味着：
- 分类的 `nbNotRead` **默认排除隐藏来源**
- 与侧栏显示的分类未读数一致
- 但 `$this->feeds` 数组**包含**隐藏来源的 feed 对象（只是不计入未读数）

> 💡 **等价性说明**：`> PRIORITY_HIDDEN (-10)` 与 `>= PRIORITY_FEED (-5)` **效果完全相同**。因为 priority 只能取 5 个离散值（20, 10, 0, -5, -10），不存在介于 -10 和 -5 之间的值。`> -10` 命中的值恰好就是 `>= -5` 命中的值。

`Category::nbNotRead($minPriority)` 无参数默认使用 `PRIORITY_FEED (-5)`，即 `>= -5`。由于上述等价性，`feeds()` 中 `> -10` 计算出的 `$this->nbNotRead` 缓存值与 `nbNotRead()` 使用默认参数时的返回值**完全一致**。代码中的缓存命中判断 `if ($this->nbNotRead > 0 && $minPriority === PRIORITY_FEED)` 正是依赖这个等价关系——只有当请求的 `$minPriority` 等于默认值时才使用缓存，其他情况（如传入 `PRIORITY_CATEGORY (0)` 或 `PRIORITY_MAIN_STREAM (10)`）则走 SQL 重新查询。

### 14.5 侧栏 feed 节点的优先级过滤

[aside_feed.phtml L129-L131](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/layout/aside_feed.phtml#L129-L131)：

```php
if (!$f_active && $feed->priority() < FreshRSS_Feed::PRIORITY_FEED) {
    continue;
}
```

**非当前激活的 feed**，如果 `priority < -5`（即 `PRIORITY_HIDDEN = -10`），侧栏直接**跳过不渲染**。只有当前激活的隐藏 feed 才会在侧栏显示。

这解释了为什么隐藏来源的 feed 在正常情况下不可见——侧栏 HTML 中根本不存在这些节点，JS 也无法操作它们。只有通过直接 URL 访问或搜索才能触达。

---

## 十五、`show_fav_unread` 配置的完整影响链

这个配置项是连接"侧栏统计"和"列表回显"的关键桥梁，其影响贯穿整个请求生命周期。

### 15.1 配置定义

[config-user.default.php L35](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/config-user.default.php#L35)：

```php
'show_fav_unread' => false,  // 默认关闭
```

帮助文本（中文）：_"同样适用于标签"_ — 说明此配置同时影响收藏视图和标签视图。

### 15.2 影响链路图

```
用户配置 show_fav_unread = true
│
├─→ Context::updateUsingRequest() [L253]
│   ├─ 当前 get='s'/'T'/t_* 且用户未指定 ?state=
│   └─ $state = STATE_NOT_READ | STATE_READ (=3, 显示全部)
│
├─→ sqlListEntriesWhere($state=3)
│   └─ STATE_NOT_READ 和 STATE_READ 同时为 true → 不生成 is_read 条件
│   └─ 如果 get='s': $state |= STATE_FAVORITE → 最终 state=7 → AND is_favorite=1
│   └─ 如果 get='t_*': $state=3 → 不生成 is_read 条件（显示标签下全部）
│
├─→ aside_feed.phtml [L6-L9]
│   └─ state=3 包含 STATE_READ → 不添加 state_unread class
│   └─ 侧栏所有 feed 节点保持可见（不受 hide_read_feeds 影响）
│
└─→ 用户体验
    ├─ 进入收藏视图 → 看到所有收藏（含已读）→ 符合"知识库"预期
    ├─ 进入标签视图 → 看到所有标签文章（含已读）→ 符合"知识库"预期
    └─ 进入主视图 → state 不被 show_fav_unread 影响 → 仍按 default_view 规则
```

### 15.3 `show_fav_unread = false` 时的行为

当配置关闭时，进入收藏/标签视图**不会**自动切换到"显示全部"。此时 state 仍按 `default_view` 规则处理：

| `default_view` | state 值 | 收藏视图显示 | 标签视图显示 |
|----------------|---------|------------|------------|
| `adaptive`（有未读） | `STATE_NOT_READ` (2) | 仅未读收藏 | 仅未读标签文章 |
| `adaptive`（无未读） | `STATE_NOT_READ \| STATE_READ` (3) | 全部收藏 | 全部标签文章 |
| `all` | 3 | 全部收藏 | 全部标签文章 |
| `unread_or_favorite` | 96 | 未读或收藏 | 未读或收藏 |

**典型困惑场景**：用户 `default_view='adaptive'`、`show_fav_unread=false`，进入收藏视图时如果收藏中有未读条目，只看到未读的收藏文章。用户以为收藏丢了，但其实已读的收藏只是被过滤掉了。开启 `show_fav_unread` 即可解决。

### 15.4 与侧栏收藏未读数的联动

侧栏收藏夹的 `data-unread` 始终显示 `$total_starred['unread']`（收藏中的未读数），与 `show_fav_unread` 无关。

但 `show_fav_unread` 间接影响"全部标已读"按钮的可见性：

[stream-footer.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/23-FreshRSS/app/views/helpers/stream-footer.phtml) 中 `toggle_bigMarkAsRead_button()` JS 函数根据当前 `get_unread` 值决定按钮显示/隐藏。进入收藏视图时 `get_unread = total_starred['unread']`。当所有收藏都已读时，`get_unread = 0`，按钮自动隐藏——无论 `show_fav_unread` 如何配置。

---

## 十六、侧栏计数与列表回显一致性的完整保障矩阵

综合以上所有分析，一致性由以下六条规则保障：

| 规则 | 机制 | 影响范围 |
|------|------|---------|
| **统计-查询对称** | 侧栏 COUNT 的 WHERE 条件与 `sqlListWhere()` 的 WHERE 条件使用相同的优先级阈值 | 收藏、标签、分类、feed、全局 |
| **渲染同源** | 模板从同一个 `FreshRSS_Context` 静态属性和 `Entry` 模型取值 | 所有视图 |
| **整页刷新兜底** | 批量操作（全部标已读）走 302 重定向，所有状态从数据库重新读取 | 批量标读 |
| **AJAX 全量更新** | `querySelectorAll('a.read/a.bookmark')` 更新同一 flux 内所有按钮 | 单篇切换 |
| **非乐观更新** | 服务器确认后才切换 DOM class / href / icon，失败则回滚 | 所有 AJAX 操作 |
| **增量计数收敛** | JS 增量更新（±1）与 PHP 全量计算（COUNT）在下次刷新时自然对齐 | 侧栏计数 |

唯一的"设计性不一致"：**搜索场景**。`needVisibility()` 会动态下调优先级下限，导致搜索结果可能包含侧栏统计中不存在的隐藏来源条目。这不是 bug，而是搜索语义决定的——用户明确搜索某 feed 时，理应看到该 feed 的内容。
