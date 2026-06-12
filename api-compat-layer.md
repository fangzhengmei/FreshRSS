# FreshRSS API 兼容层分析

## 一、多套协议收束到同一组能力的主线

FreshRSS 提供 **五套** 独立的 API 兼容协议入口，它们最终都收束到同一组底层 DAO 操作。从客户端视角，不管使用哪种协议，操作的是同一份条目、订阅和共享查询数据。

```
┌──────────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Google Reader   │  │    Fever     │  │    Query     │  │  WebSub      │  │    Misc      │
│   API (读写)     │  │   API (读写) │  │   API (只读) │  │  pshb (推送) │  │  (扩展钩子)  │
└────────┬─────────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         │                   │                  │                  │                  │
         ▼                   ▼                  ▼                  ▼                  ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                           FreshRSS DAO / Controller 层                                    │
│  EntryDAO.listWhere / listIdsWhere / markRead / markFavorite / markReadEntries/Cat/Feed/Tag│
│  FeedDAO.listFeeds / searchByUrl / updateFeed / deleteFeed                               │
│  CategoryDAO.listCategories / searchByName / addCategory / deleteCategory                 │
│  TagDAO.tagEntry / listTags / getEntryIdsTagNames / addTag / deleteTag / updateTagName     │
│  FreshRSS_feed_Controller::addFeed / deleteFeed / moveFeed / renameFeed / actualizeFeeds  │
│  FreshRSS_index_Controller::listEntriesByContext                                          │
│  UserDAO.touch / mtime                                                                    │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

### 三条能力主线

| 能力主线 | GReader 入口 | Fever 入口 | Query 入口 | WebSub 入口 | Misc 入口 | 收束的 DAO/Controller 方法 |
|---|---|---|---|---|---|---|
| **条目（读）** | `stream/contents`, `stream/items/ids`, `stream/items/contents` | `items`, `unread_item_ids`, `saved_item_ids` | `greader`, `json`, `rss`, `atom`, `html` 导出（只读） | — | 扩展自行实现 | `EntryDAO.listWhere`, `listIdsWhere`, `listByIds`, `FreshRSS_index_Controller::listEntriesByContext` |
| **条目（写）** | `edit-tag`, `mark-all-as-read` | `mark=item/feed/group as=read/unread/saved/unsaved` | — | — | 扩展自行实现 | `EntryDAO.markRead`, `markFavorite`, `markReadEntries/Cat/Feed/Tag`, `TagDAO.tagEntry` |
| **订阅（读）** | `subscription/list`, `subscription/export`, `tag/list` | `feeds`, `groups`, `feeds_groups`, `favicons` | `opml` 导出（只读） | — | 扩展自行实现 | `FeedDAO.listFeeds`, `CategoryDAO.listCategories`, `TagDAO.listTags`, `FreshRSS_Export_Service::generateOpml` |
| **订阅（写）** | `subscription/edit`, `subscription/quickadd`, `subscription/import` | 无订阅写入口 | — | 推送触发 `actualizeFeedsAndCommit` | 扩展自行实现 | `FreshRSS_feed_Controller::addFeed/deleteFeed/moveFeed/renameFeed`, `FreshRSS_Import_Service::importOpml` |
| **标签/分类（写）** | `rename-tag`, `disable-tag` | 无标签写入口 | — | — | 扩展自行实现 | `CategoryDAO.updateCategory/deleteCategory/addCategory`, `TagDAO.updateTagName/deleteTag/addTag` |
| **共享查询（读）** | — | — | `atom`, `greader`, `html`, `json`, `opml`, `rss` 全格式 | — | — | `FreshRSS_UserQuery` → `FreshRSS_index_Controller::listEntriesByContext` |

---

## 二、写操作鉴权：逐入口精确覆盖

### 2.1 鉴权层级总览

FreshRSS API 存在 **三层鉴权门控**，不同入口穿过不同层：

| 层级 | 检查内容 | 覆盖范围 |
|---|---|---|
| **L1 系统开关** | `api_enabled` 全局开关 | GReader、Fever、Query、Misc 全部入口（pshb.php 自行检查，misc.php L48-L52 检查） |
| **L2 身份认证** | 验证用户是谁 | GReader（Authorization Header）、Fever（api_key）、Query（user + 共享 token） |
| **L3 写操作令牌** | 防 CSRF 额外令牌 | 仅 GReader 的 4 个写端点 |

### 2.2 Google Reader API：全入口逐行审计

**路由总入口**：[greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1077-L1338) `GReaderAPI::parse()`

L2 认证入口判断（[greader.php L1118-L1119](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1118-L1119)）：
```php
if ($pathInfos[1] !== 'accounts') {
    self::authorizationToUser();
}
```
即除 `/accounts/ClientLogin` 外，所有端点都需要 L2 Authorization Header。

下面是 parse() switch-case 中 **全部 17 个分支** 的精确枚举：

#### 不需要 L2 认证的入口（1 个）

| 端点 | 行号 | 读/写 | 说明 |
|---|---|---|---|
| `accounts/ClientLogin` | [L1131-L1143](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1131-L1143) | 认证 | 用明文 Email/Passwd 换取 Auth Token，不经过 `authorizationToUser()`，但内部会 `FreshRSS_Context::initUser()` 并用 `password_verify()` 校验 `apiPasswordHash` |

#### 需要 L2 认证但不需要 L3 Token 的读入口（9 个）

| 端点 | 行号 | 能力主线 | 最终调用 |
|---|---|---|---|
| `stream/contents` | [L1181-L1226](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1181-L1226) | 条目（读） | `streamContents()` → `EntryDAO.listWhere()` → `entriesToArray()` 内部 `toGReader('compat', …)` 流式 JSON 输出 |
| `stream/items/ids` | [L1228-L1232](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1228-L1232) | 条目（读） | `streamContentsItemsIds()` → `EntryDAO.listIdsWhere()` |
| `stream/items/contents` | [L1233-L1236](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1233-L1236) | 条目（读） | `streamContentsItems()` → `EntryDAO.listByIds()` |
| `tag/list` | [L1239-L1244](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1239-L1244) | 订阅（读） | `tagList()` → `CategoryDAO.listCategories()`, `TagDAO.listTags()` |
| `subscription/export` | [L1249-L1250](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1249-L1250) | 订阅（读） | `subscriptionExport()` → `FreshRSS_Export_Service::generateOpml()` |
| `subscription/list` | [L1257-L1260](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1257-L1260) | 订阅（读） | `subscriptionList()` → `CategoryDAO.listCategories(prePopulateFeeds: true, details: true)` |
| `unread-count` | [L1288-L1291](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1288-L1291) | 条目（读） | `unreadCount()` → `FeedDAO.listFeedsNewestItemUsec()`, `TagDAO.listTagsNewestItemUsec()`, 遍历分类统计 `nbNotRead()` |
| `user-info` | [L1331-L1333](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1331-L1333) | — | `userInfo()` 输出 userId/userName/userEmail |
| `token` | [L1328-L1330](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1328-L1330) | — | `token()` 生成 57 字符 L3 Token，供写操作使用 |

#### 需要 L2 认证但不需要 L3 Token 的写入口（3 个）

这是最容易理解偏差的地方——**GReader 的订阅管理写端点不要求 L3 Token**：

| 端点 | 行号 | 能力主线 | `ac` 动作 | 最终调用 |
|---|---|---|---|---|
| `subscription/edit` | [L1262-L1278](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1262-L1278) | 订阅（写） | `subscribe` | `FreshRSS_feed_Controller::addFeed($streamUrl, $title, $addCatId, '', $http_auth)` — 不存在的分类名还会触发 `CategoryDAO.addCategory()` |
| 同上 | 同上 | 订阅（写） | `unsubscribe` | `FreshRSS_feed_Controller::deleteFeed($feedId)` — 还会清理该 Feed 相关的 `FreshRSS_UserQuery` |
| 同上 | 同上 | 订阅（写） | `edit` | `FreshRSS_feed_Controller::moveFeed($feedId, $addCatId)` + `FreshRSS_feed_Controller::renameFeed($feedId, $title)` |
| `subscription/quickadd` | [L1280-L1283](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1280-L1283) | 订阅（写） | 无 `ac` 参数 | `FreshRSS_feed_Controller::addFeed($url)` — 简单单参数版本，返回 streamId/streamName 的 JSON |
| `subscription/import` | [L1252-L1255](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1252-L1255) | 订阅（写） | 无 `ac` 参数 | `FreshRSS_Import_Service::importOpml(self::$ORIGINAL_INPUT)` → `FreshRSS_feed_Controller::actualizeFeedsAndCommit()` 触发全量刷新 → `invalidateHttpCache()` |

**原因**：Google Reader 原始 API 规范中，订阅管理（subscription/edit）与条目状态管理（edit-tag）是两套独立的操作面，前者未设计 Token 机制，FreshRSS 兼容层沿用了此设计。

#### 需要 L2 认证且需要 L3 Token 的写入口（4 个）

| 端点 | 行号 | 能力主线 | Token 校验代码 | 最终调用 |
|---|---|---|---|---|
| `edit-tag` | [L1293-L1301](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1293-L1301) | 条目（写）+ 标签（写） | L1294-L1295 `self::checkToken(userConf(), trim($_POST['T']))` | `editTag()`：`a` 参数触发 `EntryDAO.markRead(true)`, `EntryDAO.markFavorite(true)`, `TagDAO.addTag()` + `TagDAO.tagEntry(true)`；`r` 参数触发 `EntryDAO.markRead(false)`, `EntryDAO.markFavorite(false)`, `TagDAO.tagEntry(false)` |
| `rename-tag` | [L1303-L1308](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1303-L1308) | 标签/分类（写） | L1304-L1305 `self::checkToken(userConf(), trim($_POST['T']))` | `renameTag()`：先查分类命中则 `CategoryDAO.updateCategory()`，否则查标签命中则 `TagDAO.updateTagName()` |
| `disable-tag` | [L1310-L1316](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1310-L1316) | 标签/分类（写） | L1311-L1312 `self::checkToken(userConf(), trim($_POST['T']))` | `disableTag()`：分类命中则 `FeedDAO.changeCategory(→默认分类)` + `CategoryDAO.deleteCategory()`；标签命中则 `TagDAO.deleteTag()` |
| `mark-all-as-read` | [L1318-L1326](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1318-L1326) | 条目（写） | L1319-L1320 `self::checkToken(userConf(), trim($_POST['T']))` | `markAllAsRead()`：按 `s` 流 ID 分派到 `EntryDAO.markReadFeed()`, `markReadCat()`, `markReadTag()`, `markReadEntries()` |

#### L3 Token 的客户端兼容豁免

**位置**：[greader.php L253-L257](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L253-L257) `checkToken()`

```php
if ($user !== Minz_User::INTERNAL_USER && (
    $token === '' || // FeedMe 客户端不发送 Token
    $token === 'x')) { // Reeder 客户端发送固定值 'x'
    return true;
}
```

非内部用户时，空 Token 或固定值 `x` 视为通过。

### 2.3 Fever API：单层认证 + 写后回读，无独立写令牌

**总入口**：[fever.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/fever.php#L599-L608)

```php
if (!$handler->isAuthenticatedApiUser()) {
    echo $handler->wrap(FeverAPI::STATUS_ERR, []);
} else {
    echo $handler->wrap(FeverAPI::STATUS_OK, $handler->process());
}
```

`isAuthenticatedApiUser()` 内部调用 `authenticate()`（[fever.php L169-L194](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/fever.php#L169-L194)），每次请求都用 POST `api_key` 重新认证。

**Fever 全入口枚举（由 `process()` 中的 `isset($_REQUEST[...])` 触发，无独立路由）**：

| 入口参数 | 读/写 | 能力主线 | 最终调用 |
|---|---|---|---|
| `groups` | 读 | 订阅（读） | `getGroups()` → `CategoryDAO.listCategories(prePopulateFeeds: false, details: false)` + `getFeedsGroup()` |
| `feeds` | 读 | 订阅（读） | `getFeeds()` → `FeedDAO.listFeeds()` + `getFeedsGroup()` |
| `favicons` | 读 | 订阅（读） | `getFavicons()` → 本地 ICO 文件 base64 |
| `items` | 读 | 条目（读） | `getItems()` → 内部 `FeverDAO.findEntries()` 自定义 SQL（绕过 `EntryDAO.listWhere()`）→ 每条 `toGReader` 逻辑的等价 Fever 格式组装 |
| `links` | 读 | 无 | `getLinks()` → 固定返回 `[]` |
| `unread_item_ids` | 读 | 条目（读） | `getUnreadItemIds()` → `EntryDAO.listIdsWhere('a', 0, STATE_NOT_READ, limit: 0)` |
| `saved_item_ids` | 读 | 条目（读） | `getSavedItemIds()` → `EntryDAO.listIdsWhere('a', 0, STATE_FAVORITE, limit: 0)` |
| `mark=item as=read` | 写 | 条目（写） | `setItemAsRead($id)` → `EntryDAO.markRead($id, true)` |
| `mark=item as=unread` | 写 | 条目（写） | `setItemAsUnread($id)` → `EntryDAO.markRead($id, false)` |
| `mark=item as=saved` | 写 | 条目（写） | `setItemAsSaved($id)` → `EntryDAO.markFavorite($id, true)` |
| `mark=item as=unsaved` | 写 | 条目（写） | `setItemAsUnsaved($id)` → `EntryDAO.markFavorite($id, false)` |
| `mark=feed as=read` | 写 | 条目（写） | `setFeedAsRead($id, $before)` → `EntryDAO.markReadFeed($id, convertBeforeToId($before))` |
| `mark=group as=read` | 写 | 条目（写） | `setGroupAsRead($id, $before)` → `id==0` 走 `EntryDAO.markReadEntries()`，否则走 `EntryDAO.markReadCat($id, …)` |

**Fever 写后自动回读**（[fever.php L287-L297](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/fever.php#L287-L297)）：

```php
switch ($_REQUEST['as']) {
    case 'read':
    case 'unread':
        $response_arr['unread_item_ids'] = $this->getUnreadItemIds();
        break;
    case 'saved':
    case 'unsaved':
        $response_arr['saved_item_ids'] = $this->getSavedItemIds();
        break;
}
```

写操作执行完毕后立即返回最新状态 ID 列表，这是 Fever 协议层面的一致性保证，不需要 L3 Token。

### 2.4 Query API：纯只读，Token 即授权

**入口**：[query.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/query.php)

- 全入口 0 个写操作，只有 GET 渲染
- 认证方式：`t`（共享 Token，来自用户查询配置 `queries[].token`）+ `user`（用户名）+ 格式 `f`
- 不同格式的能力主线对应：
  - `f=rss|atom|html` → 条目（读）+ 订阅（读）渲染
  - `f=greader|json` → 条目（读）→ `helpers/export/articles.phtml`（强制 `entryIdsTagNames = []`，不导出用户标签）
  - `f=opml` → 订阅（读）→ `index/opml.phtml`（且需要 `query->safeForOpml()`）
- 所有格式最终都走 `FreshRSS_index_Controller::listEntriesByContext()` 获取条目迭代器

### 2.5 WebSub (pshb.php)：无用户鉴权的推送入口

**入口**：[pshb.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/pshb.php)

- 认证：Feed Key（`k` 参数）+ `hub.json` 内容交叉校验，不涉及用户账户
- 写操作：`FreshRSS_feed_Controller::actualizeFeedsAndCommit(feed_url: $topic, simplePiePush: $simplePiePush)` — 对订阅该 Feed 的所有用户逐一 actualize
- 特殊：`Minz_Request::_param('auth_type', 'none')` 强制禁用登录要求

### 2.6 Misc API (misc.php)：扩展级鉴权

**入口**：[misc.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/misc.php)

- L1 系统开关检查（[misc.php L48-L52](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/misc.php#L48-L52)）
- 扩展启用检查：`empty(systemConf()->extensions_enabled[$extensionName])` → 404
- 不做用户身份认证，直接 `Minz_ExtensionManager::callHookUnique(Minz_HookType::ApiMisc)`
- 写操作由扩展自行决定是否实现和鉴权

---

## 三、单条标记 vs 批量同步：一致性保证机制

### 3.1 单条 markRead：原子 JOIN 更新 + 实时缓存修正

**位置**：[EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/EntryDAO.php#L542-L566)

```sql
UPDATE `_entry` e INNER JOIN `_feed` f ON e.id_feed=f.id
SET e.is_read=:is_read, e.`lastUserModified`=:last_user_modified,
    f.`cache_nbUnreads`=f.`cache_nbUnreads` - 1  -- 或 +1，取决于从读→未读还是未读→读
WHERE e.id=:id AND e.is_read=:old_is_read
```

**一致性保证**：
1. **一条 SQL 内 JOIN 同时更新条目和 Feed 缓存**：原子性由数据库保证
2. **乐观锁**：`WHERE e.is_read=:old_is_read` 只有状态未被并发修改时才生效，返回 affected_rows 反映实际变更数
3. **`lastUserModified = time()`**：记录用户修改时间戳
4. **`FreshRSS_UserDAO::touch()`**：更新用户配置文件 mtime

### 3.2 单条 markFavorite：无缓存依赖，更简单

**位置**：[EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/EntryDAO.php#L468-L491)

```sql
UPDATE `_entry`
SET is_favorite=?, `lastUserModified`=?
WHERE id IN (?)
```

- 无缓存计数需要同步（收藏数不缓存），所以不需要 JOIN
- 单条时 `WHERE id IN (?)` 等价于主键定位
- 同样更新 `lastUserModified` 和 `touch()`

### 3.3 批量 < 6 条：逐条复用单条逻辑

**位置**：[EntryDAO.php L501-L506](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/EntryDAO.php#L501-L506)

```php
if (count($ids) < 6) {
    $affected = 0;
    foreach ($ids as $id) {
        $affected += ($this->markRead($id, $is_read) ?: 0);
    }
    return $affected;
}
```

每条独立执行单条 SQL（带 JOIN 原子缓存修正），整体非事务——部分成功部分失败是可能的。

### 3.4 批量 ≥ 6 条且 ≤ 998 条：单 UPDATE + 全量缓存重算

**位置**：[EntryDAO.php L517-L541](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/EntryDAO.php#L517-L541)

```sql
UPDATE `_entry` SET is_read=?, `lastUserModified`=? WHERE is_read<>? AND id IN (?, ?, …)
```

- 单 SQL 原子更新，乐观锁 `WHERE is_read<>?`
- `touch()` 在 SQL 之前调用（[L521-L524](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/EntryDAO.php#L521-L524)）
- 执行完毕后 `$this->updateCacheUnreads()` → `SELECT COUNT(*) WHERE is_read=0 GROUP BY id_feed` 全量校正缓存

### 3.5 批量 > 998 条：自动分片至 998 条/片

**位置**：[EntryDAO.php L507-L514](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/EntryDAO.php#L507-L514)

```php
} elseif (count($ids) > FreshRSS_DatabaseDAO::MAX_VARIABLE_NUMBER) {
    $idsChunks = array_chunk($ids, FreshRSS_DatabaseDAO::MAX_VARIABLE_NUMBER);
    foreach ($idsChunks as $idsChunk) {
        $affected += ($this->markRead($idsChunk, $is_read) ?: 0);
    }
    return $affected;
}
```

- `MAX_VARIABLE_NUMBER = 998`（[DatabaseDAO.php L17](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/DatabaseDAO.php#L17)）
- 最后一个分片会触发 `updateCacheUnreads()` 做全局缓存校正
- 分片之间无事务

### 3.6 范围标记 markReadFeed：唯一显式事务 + 截止 ID 安全阀

**位置**：[EntryDAO.php L692-L741](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/EntryDAO.php#L692-L741)

```php
$hadTransaction = $this->pdo->inTransaction();
if (!$hadTransaction) { $this->pdo->beginTransaction(); }
// 1. UPDATE _entry SET is_read=?, lastUserModified=?
//    WHERE id_feed=? AND is_read<>? AND id<=?
// 2. UPDATE _feed SET cache_nbUnreads=cache_nbUnreads-{affected_rows} WHERE id=?
if (!$hadTransaction) { $this->pdo->commit(); }
```

**一致性保证**：
1. 显式事务包裹条目 UPDATE + 缓存修正
2. 截止条件 `id <= ?` 防止标记正在流入的新条目
3. 失败时 `rollBack()`

### 3.7 范围标记 markReadCat / markReadTag / markReadEntries：无显式事务

**位置**：[EntryDAO.php L585-L784](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/EntryDAO.php#L585-L784)

- 单条 UPDATE + 截止 ID 安全阀 `id <= ?`
- 完成后 `updateCacheUnreads()`：
  - `markReadCat($catId, …)` → 限定范围 `updateCacheUnreads($catId, null)`
  - `markReadTag($tagId, …)` → 全局 `updateCacheUnreads(null, null)`
  - `markReadEntries(…)` → 全局 `updateCacheUnreads(null, null)`

### 3.8 Fever 的秒级 → 微秒截止 ID 转换

**位置**：[fever.php L569-L571](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/fever.php#L569-L571) `convertBeforeToId()`

```php
return $beforeTimestamp == 0 ? '0' : $beforeTimestamp . '000000';
```

Fever 的 `before` 参数是秒级 Unix 时间戳，追加 `000000` 微秒零填充，与 GReader `ts`（纳秒级条目 ID）殊途同归——都收束到 `id <= ?` 截止条件。

### 3.9 一致性保证对比总结

| 场景 | 显式事务 | 乐观锁 | 缓存修正 | touch() | lastUserModified |
|---|---|---|---|---|---|
| 单条 markRead | 无 | `WHERE is_read=?` | JOIN 内 `cache ±1` | ✅ | ✅ |
| 单条 markFavorite | 无 | 无 | 不需要 | ✅ | ✅ |
| 批量 markRead (<6) | 无 | 每条独立 | 每条独立 `cache ±1` | 每条✅ | 每条✅ |
| 批量 markRead (6–998) | 无 | `WHERE is_read<>?` | 全量 `COUNT(*)` 重算 | ✅ | ✅ |
| 超大批量 (>998) | 无 | 分片内独立 | 最后分片触发全量重算 | ✅ | ✅ |
| markReadFeed | **有** | `WHERE is_read<>?` | 事务内 `cache-{affected}` | ✅ | ✅ |
| markReadCat | 无 | `WHERE is_read<>?` | 分类内全量重算 | ✅ | ✅ |
| markReadEntries | 无 | `WHERE is_read<>?` | 全局重算 | ✅ | ✅ |
| markReadTag | 无 | `WHERE is_read<>?` | 全局重算 | ✅ | ✅ |
| Fever 写后回读 | 无 | 复用 DAO 机制 | 复用 DAO 机制 | ✅（DAO 内部） | ✅（DAO 内部） |

---

## 四、参数裁剪：从协议格式到内部模型的转换

### 4.1 GReader 流 ID → 内部 type/id

**位置**：[greader.php L609-L678](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L609-L678) `streamContentsFilters()` + [L689-L697](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L689-L697) `streamContents()` path dispatch

| 外部流 ID | type | id | 说明 |
|---|---|---|---|
| `user/-/state/com.google/reading-list` | `A` | `''` | 全部（除隐藏） |
| `user/-/state/com.google/starred` | `s` | `''` | 收藏 |
| `user/-/state/org.freshrss/main` | `a` | `''` | 主流 |
| `user/-/state/org.freshrss/important` | `i` | `''` | 重要 |
| `feed/{numeric_id}` | `f` | (int)id | — |
| `feed/{url}` | `f` | `FeedDAO.searchByUrl(url)->id` | URL 出现在 PATH_INFO 时还需从 `REQUEST_URI` 正则提取（[L1193-L1198](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1193-L1198)） |
| `user/-/label/{name}` → 分类命中 | `c` | `CategoryDAO.searchByName(name)->id` | — |
| `user/-/label/{name}` → 标签命中 | `t` | `TagDAO.searchByName(name)->id` | 先查分类，无则查标签 |

### 4.2 条目 ID 双向转换

- 内部：64 位十进制数字字符串
- GReader 对外：`tag:google.com,2005:reader/item/{hex}`（十六进制）
- 输入解析（[greader.php L850-L853](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L850-L853)）：`!ctype_digit || [0]=='0'` → `hex2dec(basename($e_id))`
- 32 位平台使用 GMP 扩展

### 4.3 输出裁剪三种模式

**位置**：[Entry.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/Entry.php#L1219-L1318) `toGReader()`

| 维度 | `compat` 模式（GReader stream 输出） | 默认模式 | `freshrss` 扩展模式 |
|---|---|---|---|
| 内容字段 | `summary.content`，截断 500KB | `content.content` 完整 | `content.content` 完整 |
| 标题/作者 | `escapeToUnicodeAlternative()` 全角转义 | 原样 | 原样 |
| alternate | 移除 `type` | 保留 `type: text/html` | 保留 `type: text/html` |
| origin.title | 全角转义 | 原样 | 原样 |
| origin.feedUrl | 不输出 | 不输出 | 输出 |
| guid | 不输出 | 不输出 | 输出 |
| categories 中 unread | 不输出 | 不输出 | 输出 `user/-/state/com.google/unread` |

`escapeToUnicodeAlternative()`（[lib_rss.php L166-L180](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/lib/lib_rss.php#L166-L180)）把 `&` `<` `>` `'` `"` `^` `?` `\` `/` `,` `;` 等转为 Unicode 全角，因为这些字符在 GReader 流 ID 规范中有特殊含义。

### 4.4 Fever 参数裁剪差异

- 分页：`since_id`（正向）/ `max_id`（反向），无 continuation token
- 固定上限 50 条（[fever.php L128-L131](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/fever.php#L128-L131) `FeverDAO.findEntries()` 内 `LIMIT 50`）
- 不支持：`is_spark=0`，`links=[]`
- 状态：`is_read` / `is_saved` 为 0/1 整数，而非 GReader 的 categories 标签

---

## 五、增量同步：多协议的收束方式

### 5.1 GReader 增量同步三件套

1. **时间范围过滤**：`ot`/`nt` → `streamContentsFilters()` 同时对 `date`（发布时间）和 `lastUserModified`（用户修改时间）构建 OR 关系的 `FreshRSS_BooleanSearch`
2. **Continuation 游标**：条目 ID 作为分页 token，`count+1` 多取 1 条，`LimitIterator($items, offset: 1)` 跳过重复，满 `count` 条时输出 `continuation: lastEntryId`
3. **未读计数信号**：`unread-count` 按 Feed/分类/标签/阅读列表分维度统计，附带 `newestItemTimestampUsec`

### 5.2 Fever 增量同步二件套

1. **全量 ID 列表 diff**：`unread_item_ids` 和 `saved_item_ids` 逗号分隔，客户端本地计算差异
2. **写后回读**：写操作响应直接附最新 ID 列表，无需额外请求

### 5.3 用户级变更信号

**`FreshRSS_UserDAO::touch()`**（[UserDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/UserDAO.php#L59-L66)）

```php
return touch(USERS_PATH . '/' . $username . '/config.php');
```

- 所有写操作都会触发（GReader `subscription/edit`, `edit-tag`, `subscription/import`；Fever 所有 mark；Web 界面操作）
- Query API 的 `httpConditional()` 也用此 mtime 做 HTTP 缓存校验

### 5.4 条目级变更信号

- `lastUserModified` 每次 markRead/markFavorite 都更新为 `time()`
- `ot` 在 GReader `streamContentsFilters()` 中同时对 `date` 和 `lastUserModified` 取 OR，确保用户修改过的旧条目也会在增量同步中被捕获

---

## 六、密码与凭证管理

**位置**：[apiController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Controllers/apiController.php#L13-L33) `updatePassword()`

一次 API 密码设置同时生成两套凭证：

| 凭证 | 算法 | 存储位置 | 用途 |
|---|---|---|---|
| `apiPasswordHash` | `FreshRSS_password_Util::hash()`（bcrypt 系） | 用户配置 `data/users/{name}/config.php` | GReader Authorization Header 和 ClientLogin |
| `feverKey` | `md5(strtolower($username) . ':' . $api_password)` | 用户配置 + 文件 `data/fever/.key-{sha1(salt)}-{feverKey}.txt`（纯文本用户名） | Fever `api_key` 参数 |

API 密码与 Web 登录密码 (`passwordHash`) 完全独立，互不影响。
