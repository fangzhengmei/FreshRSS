# FreshRSS API 兼容层分析

## 一、多套协议收束到同一组能力的主线

FreshRSS 提供四套独立的 API 兼容协议入口，它们最终都收束到同一组底层 DAO 操作。从客户端视角，不管使用哪种协议，操作的是同一份条目、订阅和共享查询数据。

```
┌─────────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Google Reader   │  │    Fever     │  │    Query     │  │   WebSub     │
│   API (读写)     │  │   API (读写) │  │   API (只读) │  │  pshb (推送) │
└────────┬────────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         │                  │                  │                  │
         ▼                  ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        FreshRSS DAO 层                                  │
│  EntryDAO.markRead / markFavorite / markReadEntries / markReadCat …    │
│  FeedDAO.listFeeds / searchByUrl …                                     │
│  CategoryDAO.listCategories / searchByName …                           │
│  TagDAO.tagEntry / listTags / getEntryIdsTagNames …                    │
│  UserDAO.touch / mtime                                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

**三条能力主线**：

| 能力主线       | GReader 入口                                  | Fever 入口                        | Query 入口               | 收束的 DAO 方法                                              |
|----------------|-----------------------------------------------|-----------------------------------|--------------------------|--------------------------------------------------------------|
| **条目**       | stream/contents, stream/items/ids, edit-tag, mark-all-as-read | items, unread_item_ids, saved_item_ids, mark=item/feed/group | greader/json 导出（只读） | `EntryDAO.listWhere`, `listIdsWhere`, `markRead`, `markFavorite`, `markReadEntries/Cat/Feed/Tag` |
| **订阅**       | subscription/list, subscription/edit, subscription/quickadd, subscription/export/import | feeds, groups, feeds_groups (只读) | opml 导出（只读）         | `FeedDAO.listFeeds`, `FreshRSS_feed_Controller::addFeed/deleteFeed/moveFeed/renameFeed` |
| **共享查询**   | 无                                            | 无                                | atom/json/rss/html/opml 全格式 | `FreshRSS_UserQuery` → `FreshRSS_index_Controller::listEntriesByContext` |

**关键发现**：Fever 对订阅管理是纯只读的（获取 feeds/groups），而 GReader 覆盖了完整的读写。WebSub (pshb.php) 则是唯一的推送入口，直接触发 Feed 刷新，不走用户 API 鉴权。

---

## 二、写操作鉴权：逐入口精确覆盖

### 2.1 鉴权层级总览

FreshRSS API 存在 **三层鉴权门控**，不同入口穿过不同层：

| 层级           | 检查内容                   | 覆盖范围                                         |
|----------------|----------------------------|--------------------------------------------------|
| **L1 系统开关**| `api_enabled` 全局开关     | GReader、Fever、Query、Misc 全部入口             |
| **L2 身份认证**| 验证用户是谁              | GReader（Authorization Header）、Fever（api_key）、Query（user+t 共享 token） |
| **L3 写操作令牌**| 防跨站写操作 CSRF         | 仅 GReader 的部分写端点                          |

### 2.2 Google Reader API：唯一需要三层鉴权的入口

#### L2 身份认证——Authorization Header

**位置**：[greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1118-L1119) `parse()` 中

```php
if ($pathInfos[1] !== 'accounts') {
    self::authorizationToUser();
}
```

**覆盖范围**：除了 `/accounts/ClientLogin`（登录本身）之外，所有 GReader 端点都必须通过 Authorization Header 认证。包括读操作和写操作。

**认证算法**（[greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L178-L207) `authorizationToUser()`）：
- 从 HTTP Authorization 头提取 `GoogleLogin auth` 值
- 按 `/` 分割为 `{username}/{token}`
- 校验 `token === sha1(salt + username + apiPasswordHash)`
- 同时校验用户存在且 `enabled`

**登录端点**（[greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L209-L232) `clientLogin()`）：
- 路径：`/accounts/ClientLogin`
- 用明文 API 密码通过 `password_verify()` 验证 `apiPasswordHash`
- 返回 `SID=... LSID=null Auth=...` 供后续请求携带

#### L3 写操作令牌——Token 校验

**位置**：[greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L247-L263) `checkToken()`

**精确覆盖的写端点**（4 个）：

| 端点                | 行号      | 操作类型               | POST 参数 `T` |
|---------------------|-----------|------------------------|---------------|
| `edit-tag`          | [L1293-L1301](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1293-L1301) | 单条/批量标记已读、收藏、标签 | ✅ checkToken |
| `rename-tag`        | [L1303-L1308](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1303-L1308) | 重命名分类或标签        | ✅ checkToken |
| `disable-tag`       | [L1310-L1316](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1310-L1316) | 删除分类或标签          | ✅ checkToken |
| `mark-all-as-read`  | [L1318-L1326](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1318-L1326) | 范围标记全部已读        | ✅ checkToken |

**不需要 Token 的写端点**（2 个）：

| 端点                  | 行号      | 操作类型             | 原因                     |
|-----------------------|-----------|----------------------|--------------------------|
| `subscription/edit`   | [L1262-L1278](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1262-L1278) | 订阅/退订/编辑 Feed    | Google Reader 原始协议未要求 Token |
| `subscription/quickadd` | [L1280-L1283](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1280-L1283) | 快速添加 Feed         | 同上                     |
| `subscription/import` | [L1252-L1255](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1252-L1255) | OPML 导入             | 同上                     |

**Token 算法**：`str_pad(sha1(salt + user + apiPasswordHash), 57, 'Z')`

**客户端兼容豁免**（[greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L253-L257)）：
- 空 token → 通过（FeedMe 客户端不发送 Token）
- token = `x` → 通过（Reeder 客户端发送固定 `x`）

**不需要 Token 的读端点**（7 个）：stream/contents, stream/items/ids, stream/items/contents, tag/list, subscription/list, subscription/export, unread-count, user-info, token（获取 Token 本身）

### 2.3 Fever API：单层认证，无写操作令牌

**位置**：[fever.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/fever.php#L169-L194) `authenticate()`

- **认证方式**：每次请求都通过 POST `api_key` 认证，读和写都在同一个 `process()` 方法里
- **算法**：`api_key = md5(username:apiPassword)`，小写十六进制
- **Key 反查机制**（[feverUtil.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Utils/feverUtil.php#L41-L56)）：文件映射 `data/fever/.key-{sha1(salt)}-{feverKey}.txt` → 纯文本用户名
- **无双层令牌**：Fever 协议本身没有"写操作令牌"概念，每次请求都重新校验 api_key
- **写操作自动回传最新状态**：写操作执行后立即返回最新的 `unread_item_ids` 或 `saved_item_ids`（[fever.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/fever.php#L287-L297)），这是 Fever 协议的"写后读"一致性保证

### 2.4 Query API：纯只读，Token 即授权

**位置**：[query.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/query.php)

- **无写操作**：仅支持导出/展示，不支持修改任何状态
- **认证**：`t`（共享 token）+ `user`（用户名），Token 来自用户自定义查询配置
- **权限隔离**：不导出用户标签（`entryIdsTagNames = []`），保护隐私

### 2.5 WebSub (pshb.php)：无用户鉴权

**位置**：[pshb.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/pshb.php)

- **认证方式**：通过 Feed Key（`k` 参数）+ hub.json 交叉校验，不涉及用户身份
- **操作**：接收 Hub 推送，触发 Feed 刷新，遍历订阅用户逐一 actualize
- **特殊**：强制 `auth_type = 'none'` 避免登录要求

### 2.6 Misc API (misc.php)：系统级扩展鉴权

**位置**：[misc.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/misc.php)

- **认证**：不需要用户认证，但要求扩展在系统配置中启用
- **调用**：`Minz_ExtensionManager::callHookUnique(Minz_HookType::ApiMisc)`
- **无写操作入口**：扩展自行决定是否实现写操作

---

## 三、单条标记 vs 批量同步：一致性保证机制

### 3.1 单条标记：原子更新 + 实时缓存修正

**代码位置**：[EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/EntryDAO.php#L542-L566) `markRead()` 的单条分支

当传入单个 ID（非数组）时：

```sql
UPDATE `_entry` e INNER JOIN `_feed` f ON e.id_feed=f.id
SET e.is_read=:is_read, `lastUserModified`=:last_user_modified,
    f.`cache_nbUnreads`=f.`cache_nbUnreads` -1   -- 或 +1
WHERE e.id=:id AND e.is_read=:old_is_read
```

**一致性保证**：
1. **一条 SQL 同时更新条目状态和 Feed 缓存计数**：`INNER JOIN _feed` 确保条目更新与缓存递减/递增在同一语句中完成
2. **乐观锁**：`WHERE e.is_read=:old_is_read` 确保状态未变才更新，防止重复标记
3. **`lastUserModified` 更新**：记录用户修改时间戳，用于增量同步判断
4. **`FreshRSS_UserDAO::touch()`**：更新用户配置文件修改时间，客户端可通过此时间判断数据是否变化

**markFavorite 单条**：没有 Feed 缓存计数需要修正（收藏数不缓存），使用更简单的 `WHERE id IN (?)` 更新，同样更新 `lastUserModified` 和调用 `touch()`。

### 3.2 批量标记（少量 <6 条）：逐条调用单条逻辑

**代码位置**：[EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/EntryDAO.php#L501-L506)

```php
if (count($ids) < 6) {	//Speed heuristics
    $affected = 0;
    foreach ($ids as $id) {
        $affected += ($this->markRead($id, $is_read) ?: 0);
    }
    return $affected;
}
```

**设计原因**：少量条目时，逐条执行单条 SQL（带 JOIN 缓存修正）比批量 SQL + 全量缓存重算更高效。每条 SQL 自带乐观锁和缓存原子修正。

**一致性保证**：每条独立保证原子性，但整个批量**不是事务性的**——部分成功部分失败是可能的。

### 3.3 批量标记（≥6 条）：单条 SQL + 全量缓存重算

**代码位置**：[EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/EntryDAO.php#L517-L541)

```sql
UPDATE `_entry`
SET is_read=?, `lastUserModified`=?
WHERE is_read<>? AND id IN (?, ?, ?, ...)
```

**一致性保证**：
1. **单条 SQL 原子更新**：所有 ID 在一条 UPDATE 中处理
2. **乐观锁**：`WHERE is_read<>?` 跳过已是目标状态的条目
3. **全量缓存重算**：执行后调用 `updateCacheUnreads()`，重新 `SELECT COUNT(*) WHERE is_read=0` 对 `_feed` 表的 `cache_nbUnreads` 做全量校正
4. **`touch()` 在 SQL 之前调用**：确保用户修改时间戳被更新

**缓存重算 vs 递减/递增的差异**：
- 单条标记：用 `cache_nbUnreads -1` / `+1` 精确修正（O(1)）
- 批量标记：用 `SELECT COUNT(*)` 全量重算（O(n) 但绝对准确）
- 范围标记（markReadFeed）：在事务中先 UPDATE 条目再 `cache_nbUnreads - {affected}` 修正

### 3.4 超大批量：自动分片

**代码位置**：[EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/EntryDAO.php#L507-L514)

```php
} elseif (count($ids) > FreshRSS_DatabaseDAO::MAX_VARIABLE_NUMBER) {
    $idsChunks = array_chunk($ids, FreshRSS_DatabaseDAO::MAX_VARIABLE_NUMBER);
    foreach ($idsChunks as $idsChunk) {
        $affected += ($this->markRead($idsChunk, $is_read) ?: 0);
    }
    return $affected;
}
```

- `MAX_VARIABLE_NUMBER = 998`（基于 SQLite 限制，[DatabaseDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/DatabaseDAO.php#L17)）
- 每个分片独立执行，最后一个分片执行后触发 `updateCacheUnreads()`
- **注意**：分片之间没有事务包裹，并发请求可能导致部分分片成功部分失败

### 3.5 范围标记（mark-all-as-read）：事务 + 截止 ID 安全阀

#### markReadFeed（事务性最强）

**代码位置**：[EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/EntryDAO.php#L692-L741)

```php
$hadTransaction = $this->pdo->inTransaction();
if (!$hadTransaction) {
    $this->pdo->beginTransaction();
}
// 1. UPDATE _entry SET is_read=? WHERE id_feed=? AND is_read<>? AND id<=?
// 2. UPDATE _feed SET cache_nbUnreads=cache_nbUnreads-{affected} WHERE id=?
if (!$hadTransaction) {
    $this->pdo->commit();
}
```

**一致性保证**：
1. **显式事务**：条目更新 + 缓存修正包裹在 `beginTransaction/commit` 中
2. **截止 ID 安全阀**：`id <= ?`（来自 GReader 的 `ts` 纳秒时间戳，或 Fever 的 `before` 秒级时间戳追加 `000000` 变为微秒 ID），防止标记正在流入的新条目
3. **失败回滚**：SQL 执行失败时调用 `$this->pdo->rollBack()`

#### markReadCat / markReadEntries / markReadTag

**代码位置**：[EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/EntryDAO.php#L585-L784)

- **无显式事务**：单条 UPDATE + `updateCacheUnreads()` 全量重算
- **截止 ID 安全阀**：统一使用 `id <= ?` 条件
- `markReadCat` 的缓存重算范围限定为该分类（`updateCacheUnreads($id, null)`）
- `markReadEntries` 和 `markReadTag` 的缓存重算是全局的（`updateCacheUnreads(null, null)`）

#### Fever API 的时间戳转换

**位置**：[fever.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/fever.php#L569-L571) `convertBeforeToId()`

```php
private function convertBeforeToId(int $beforeTimestamp): string {
    return $beforeTimestamp == 0 ? '0' : $beforeTimestamp . '000000';
}
```

Fever 的 `before` 参数是秒级 Unix 时间戳，追加 `000000` 转为微秒级条目 ID 格式，与 GReader 的 `ts`（纳秒级）殊途同归——都收束到 `id <= ?` 的截止条件。

### 3.6 一致性保证对比总结

| 场景              | 事务       | 乐观锁                | 缓存修正方式           | touch() | lastUserModified |
|-------------------|------------|-----------------------|------------------------|---------|------------------|
| 单条 markRead     | 无显式事务 | `WHERE is_read<>?`    | JOIN 内 `cache ±1`     | ✅      | ✅               |
| 单条 markFavorite | 无显式事务 | 无（收藏无缓存依赖）  | 不需要                 | ✅      | ✅               |
| 批量 markRead (<6)| 无显式事务 | 每条独立乐观锁        | 每条独立 `cache ±1`    | 每条✅  | 每条✅            |
| 批量 markRead (≥6)| 无显式事务 | `WHERE is_read<>?`    | 全量 `COUNT(*)` 重算   | ✅      | ✅               |
| 超大批量 (≥998)   | 无显式事务 | 分片内独立乐观锁      | 最后分片触发全量重算   | ✅      | ✅               |
| markReadFeed      | **显式事务** | `WHERE is_read<>?`    | 事务内 `cache-{affected}` | ✅  | ✅               |
| markReadCat       | 无显式事务 | `WHERE is_read<>?`    | 限定分类的全量重算     | ✅      | ✅               |
| markReadEntries   | 无显式事务 | `WHERE is_read<>?`    | 全量重算               | ✅      | ✅               |
| markReadTag       | 无显式事务 | `WHERE is_read<>?`    | 全量重算               | ✅      | ✅               |
| Fever 写后回读    | 无显式事务 | 复用 DAO 同一机制     | 复用 DAO 同一机制      | ✅      | ✅               |

---

## 四、参数裁剪：从协议格式到内部模型的转换

### 4.1 流 ID（Stream ID）解析——GReader 的核心转换

**位置**：[greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L609-L678) `streamContentsFilters()`

GReader 协议使用字符串形式的流 ID，FreshRSS 需将其转为内部数字 ID 和类型标记：

| 外部流 ID 格式                                 | 解析后 type | 解析后 ID        | 说明                     |
|------------------------------------------------|-------------|------------------|--------------------------|
| `user/-/state/com.google/reading-list`         | `A`         | `''`（空）       | 全部条目                 |
| `user/-/state/com.google/starred`              | `s`         | `''`             | 收藏条目                 |
| `user/-/state/org.freshrss/main`               | `a`         | `''`             | 主流条目（FreshRSS 扩展）|
| `user/-/state/org.freshrss/important`          | `i`         | `''`             | 重要条目（FreshRSS 扩展）|
| `feed/{numeric_id}`                            | `f`         | 数字 Feed ID     | 按 Feed 过滤             |
| `feed/{url}`                                   | `f`         | 查询得到的 Feed ID | URL 查库转为数字 ID    |
| `user/-/label/{name}`                          | `c`         | 查询得到的分类 ID | 分类名查库              |
| `user/-/label/{name}`（标签）                   | `t`         | 查询得到的标签 ID | 先查分类，无则查标签    |

**URL 型 Feed ID 的特殊处理**：Feed URL 出现在 PATH_INFO 中时，需要从 `REQUEST_URI` 正则提取（[greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L1193-L1198)），因为 PATH_INFO 经 Web 服务器处理后可能丢失 URL 参数。

### 4.2 条目 ID 的双向转换

**位置**：[Entry.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/Entry.php#L1200-L1208) `dec2hex()` + [greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L35-L51) `hex2dec()`

- **内部存储**：64 位十进制数字字符串（如 `1234567890123456789`）
- **GReader 对外格式**：`tag:google.com,2005:reader/item/{hex}`（十六进制）
- **输入解析**：若条目 ID 以 `tag:google.com,2005:reader/item/` 为前缀或为十六进制，则 `hex2dec()` 转回十进制
- **32 位平台特殊处理**：使用 GMP 扩展进行大数运算

### 4.3 输出裁剪的三种模式

**位置**：[Entry.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/Entry.php#L1219-L1318) `toGReader()`

| 维度           | `compat` 模式                  | 默认模式                   | `freshrss` 模式            |
|----------------|--------------------------------|----------------------------|-----------------------------|
| **触发场景**   | GReader API 的 stream 输出     | 内部导出/非兼容场景        | FreshRSS 自有扩展协议       |
| **内容字段**   | `summary.content`，截断 500KB  | `content.content`，完整    | `content.content`，完整     |
| **标题/作者**  | `escapeToUnicodeAlternative()` | 原样输出                   | 原样输出                    |
| **alternate**  | 移除 `type` 字段               | 保留 `type: text/html`     | 保留 `type: text/html`     |
| **origin**     | 标题转义                       | 标题原样 + 无 feedUrl      | 标题原样 + 输出 feedUrl    |
| **guid**       | 不输出                         | 不输出                     | 输出                        |
| **unread 标签**| 不输出                         | 不输出                     | `user/-/state/com.google/unread` |

**`escapeToUnicodeAlternative()`**（[lib_rss.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/lib/lib_rss.php#L166-L180)）：将 `&` `<` `>` `'` `"` `^` `?` `\` `/` `,` `;` 等字符替换为 Unicode 全角形式，因为 GReader 流 ID 规范中这些字符有特殊含义，不能原样出现在标签名等字段中。

### 4.4 Fever 的参数裁剪差异

- **分页方式不同**：GReader 用 continuation token（条目 ID 游标），Fever 用 `since_id`（正向）/ `max_id`（反向）
- **固定每页上限**：50 条（硬编码在 [fever.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/fever.php#L129) `FeverDAO.findEntries()` 中）
- **不支持的字段**：`is_spark` 固定为 0，`links` 固定为空数组
- **状态表示**：`is_read` / `is_saved` 为 0/1 整数，而非 GReader 的 categories 标签

---

## 五、增量同步：多协议的收束方式

### 5.1 GReader 的增量同步三件套

1. **时间范围**：`ot`/`nt` → 构建 `FreshRSS_BooleanSearch`，对 `date`（发布时间）和 `lastUserModified`（用户修改时间）取 OR 关系
2. **Continuation 游标**：条目 ID 作为分页 token，多取 1 条跳过首条确保不遗漏不重复
3. **未读计数**：`unread-count` 端点按 Feed/分类/标签分维度统计，附带 `newestItemTimestampUsec`

### 5.2 Fever 的增量同步二件套

1. **全量 ID 列表**：`unread_item_ids` 和 `saved_item_ids` 返回逗号分隔的全量 ID 列表，客户端本地 diff
2. **写后回读**：写操作后立即返回最新 ID 列表，不需要额外请求

### 5.3 用户级变更检测

**`FreshRSS_UserDAO::touch()`**（[UserDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Models/UserDAO.php#L59-L66)）：

```php
public static function touch(string $username = ''): bool {
    return touch(USERS_PATH . '/' . $username . '/config.php');
}
```

- 每次写操作都调用 `touch()`，更新用户配置文件的修改时间
- 客户端可通过 `FreshRSS_UserDAO::mtime()` 检测用户数据是否变化
- 这是跨协议的统一变更信号：GReader 写操作、Fever 写操作、Web 界面操作都会触发

### 5.4 条目级变更检测

- **`lastUserModified` 字段**：每次 `markRead` / `markFavorite` 都更新此字段为 `time()`
- `streamContentsFilters()` 中 `ot` 参数同时对 `date` 和 `lastUserModified` 构建 OR 条件，确保用户修改过的条目在增量同步中被捕获
- 这是 GReader 协议独有的增量能力，Fever 协议没有利用此字段

### 5.5 多协议增量同步收束图

```
GReader 客户端                         Fever 客户端
    │                                      │
    ├─ ot/nt 时间范围                      ├─ since_id/max_id 翻页
    ├─ continuation 游标分页               ├─ unread_item_ids 全量 diff
    ├─ unread-count 未读数变化检测         ├─ saved_item_ids 全量 diff
    │                                      ├─ 写后回读最新 ID 列表
    │                                      │
    ▼                                      ▼
┌──────────────────────────────────────────────────────┐
│              EntryDAO.listWhere / listIdsWhere        │
│              + lastUserModified 时间戳过滤            │
│              + FreshRSS_UserDAO::touch() 变更信号     │
└──────────────────────────────────────────────────────┘
```

---

## 六、密码与凭证管理

**位置**：[apiController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Controllers/apiController.php#L13-L33) `updatePassword()`

一次 API 密码设置同时生成两套凭证：

| 凭证             | 算法                              | 存储位置                              | 用途          |
|------------------|-----------------------------------|---------------------------------------|---------------|
| `apiPasswordHash`| `FreshRSS_password_Util::hash()`  | 用户配置 `data/users/{name}/config.php` | GReader 认证 |
| `feverKey`       | `md5(username:apiPasswordPlain)`  | 用户配置 + 文件 `data/fever/.key-*.txt` | Fever 认证   |

API 密码与 Web 登录密码 (`passwordHash`) 完全独立，互不影响。
