# FreshRSS Feed 刷新与 Entry 入库链路分析

## 一、触发入口矩阵

FreshRSS 通过以下 6 种独立入口触发 Feed 刷新：

### 1.1 Web UI 手动刷新
**控制器**: [feedController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Controllers/feedController.php) 的 `actualizeAction()` (L954-L1033)

通过路由 `?c=feed&a=actualize` 访问，支持参数：

| 参数 | 类型 | 说明 |
|------|------|------|
| `id` | int | 单个 Feed ID；`0`=全部；`-1`=仅提交+刷新缓存 |
| `url` | string | 按 URL 代替 ID 定位单个 Feed |
| `maxFeeds` | int | 全量刷新时最多处理的 Feed 数 (默认 10) |
| `noCommit` | 0/1 | 1=只写入 `_entrytmp`，不迁移到主表 |
| `ajax` | bool | 静默 AJAX 模式，不渲染布局 |

权限：登录用户；若 `allow_anonymous_refresh` 开启，则允许匿名访问 `actualize` 动作。

### 1.2 添加 Feed 时首次刷新
**入口**: [feedController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Controllers/feedController.php) 的 `addFeed()` 静态方法 (L39-L117)

在 `addAction()` 完成数据库插入后，立即调用：
```php
self::actualizeFeedsAndCommit($id, $url);
```
是 **同步阻塞** 的，会等待新 Feed 的第一批条目入库完成后才返回给用户。

### 1.3 系统级 CLI 全量刷新 (Cron)
**脚本**: [actualize_script.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/actualize_script.php)

典型 cron 用法：
```bash
php app/actualize_script.php
```

关键特性：
- 路由模拟：通过 `$_GET['c']='feed'&'a'='actualize'` 复用到 Web 的 `actualizeAction()`
- **全局互斥锁**：`TMP_PATH/actualize.freshrss.lock`，TTL 900 秒，防止并发刷新
- 遍历所有用户，跳过：禁用用户、长期不活跃用户 (`max_inactivity` 配置)
- `FeedBeforeActualize` hook 中每次 `touch($mutexFile)` 续期锁
- 每个用户独立初始化上下文、扩展、翻译

### 1.4 单用户 CLI 刷新
**脚本**: [actualize-user.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/cli/actualize-user.php)

用法：
```bash
./cli/actualize-user.php --user <username>
```

流程比系统级脚本更轻：
1. `minorDbMaintenance()` 数据库小维护
2. `commitNewEntries()` + `updateCachedValues()` (先刷新遗留缓存)
3. `refreshDynamicOpmls()` 动态 OPML 刷新
4. `actualizeFeedsAndCommit()` 直接调用核心函数
5. 不使用全局互斥锁，由调用方自行排程

### 1.5 WebSub (PubSubHubbub) 实时推送
**入口**: [pshb.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/p/api/pshb.php)

被动接收上游 Hub 推送的 feed 内容 (POST body 为 RSS/Atom XML)：

1. 通过 URL 参数 `?k=<key>` 反查到 canonical feed URL
2. 找到所有订阅该 feed 的用户文件 (`PSHB_PATH/feeds/<hash>/*.txt`)
3. 为每个用户初始化上下文后调用：
```php
actualizeFeedsAndCommit(feed_url: $canonical, simplePiePush: $simplePie, selfUrl: $self);
```
传入已解析好的 `SimplePieCustom` 对象，**跳过 HTTP 拉取**，直接走去重/入库链路。

### 1.6 JavaScript 后台自动刷新
**入口**: `actualize.phtml` 视图 + `javascriptController`

由浏览器前端 JS 定时轮询，参数组合与 1.1 的 AJAX 模式相同。

---

## 二、核心刷新流程 (actualizeFeeds)

**核心函数**: [feedController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Controllers/feedController.php) 的 `actualizeFeeds()` (L427-L849)

### 2.1 筛选与前置检查

```
  ┌─────────────────────────────────────────────┐
  │  确定 Feed 列表                              │
  │   • 单 Feed: searchById / searchByUrl        │
  │   • 全量:  listFeedsOrderUpdate(-1)           │
  └──────────────────┬──────────────────────────┘
                     ▼
  ┌─────────────────────────────────────────────┐
  │  跳过条件 (任一命中即 skip)                   │
  │   • PubSubHubbub 启用且 lastUpdate<24h       │
  │   • feed 静音 (ttl<0) 且非显式刷新            │
  │   • TTL 未到 且 缓存未被其他用户更新            │
  │   • 无法获取 lock 文件 (1h 过期自动清理)       │
  └─────────────────────────────────────────────┘
```

### 2.2 Feed 内容获取 (按 Kind 分派)

| KIND 常量 | 值 | 加载方法 | 说明 |
|-----------|----|---------|------|
| KIND_RSS | 0 | `$feed->load()` | SimplePie 标准 RSS/Atom，HTTP + 缓存 |
| KIND_HTML_XPATH | 10 | `$feed->loadHtmlXpath()` | DOMDocument + XPath 爬取 HTML |
| KIND_XML_XPATH | 15 | `$feed->loadHtmlXpath()` | XML + XPath |
| KIND_JSONFEED | 25 | `$feed->loadJson()` | JSON Feed 标准 |
| KIND_JSON_DOTNOTATION | 30 | `$feed->loadJson()` | JSON + 点路径，转 RSS 后再 SimplePie |
| KIND_HTML_XPATH_JSON_DOTNOTATION | 35 | `$feed->loadJson()` | 先用 XPath 抽取 JSON，再点路径 |

SimplePie 缓存命中判定：
```php
$simplePie->get_hash() === $feed->attributeString('SimplePieHash')
```
相等则 `$feedIsUnchanged=true`，跳过解析，只执行 `updateLastSeenUnchanged` 刷新所有条目 lastSeen。

---

## 三、Entry 去重依据 (双层去重)

### 3.1 第一层：GUID 去重

**代码位置**: [Feed.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Models/Feed.php) 的 `decideEntryGuid()` (L701-L745) 与 `loadGuids()` (L753-L804)

#### 3.1.1 GUID 策略矩阵

每个 Feed 可配置 `unicityCriteria` 属性决定 GUID 生成算法：

| 策略值 | 公式 | 适用场景 |
|--------|------|---------|
| `null` (默认) | `$item->get_id()` = RSS `<guid>` / Atom `<id>` | 规范提供方 |
| `link` | `get_permalink()` | 源站 guid 缺失或重复时 |
| `sha1:link_published` | sha1(链接 + 发布时间戳) | 博客/新闻通用 |
| `sha1:link_published_title` | sha1(链接 + 时间 + 标题) | 链接不稳定场景 |
| `sha1:link_published_title_content` | sha1(链接 + 时间 + 标题 + 内容) | 最严格，避免同名覆盖 |
| `sha1:title` | sha1(标题) | 完全无链接的源 |
| `sha1:title_published` | sha1(标题 + 时间) | 同上但有时间 |
| `sha1:title_published_content` | sha1(标题 + 时间 + 内容) | |
| `sha1:content` | sha1(正文) | |
| `sha1:content_published` | sha1(正文 + 时间) | |
| `sha1:published` | sha1(发布时间) | 极端 fallback |

#### 3.1.2 自动降级机制

`loadGuids()` 在生成所有 GUID 后检查 **无效/重复率**：
- 阈值：超过 `round(0.05 * 条目数)` (约 5%) 的 GUID 为空或重复
- 且未设置 `unicityCriteriaForced=true`

则执行降级并重算 (递归一次)：
```
null / id ──┐
            ├──▶ sha1:link_published ──▶ sha1:link_published_title ──▶ 停止
   link ────┘
```
兼容 Legacy：若 `hasBadGuids=true`，强制按 `link` 策略。

#### 3.1.3 单条目 Fallback

即使策略选了某种方式，若结果为空串，`fallback=true` 会逐级尝试：
1. 原生 `item id`
2. `sha1(permalink + date)`
3. `sha1(permalink + date + title)`
4. `sha1(permalink + date + title + content)`
仍空则等于空串 sha1，后续必被判定无效。

---

### 3.2 第二层：内容 Hash 更新检测

**代码位置**: [Entry.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Models/Entry.php) 的 `hash()` (L513-L520)

```php
md5(
    $this->link
  . $this->title
  . $this->authors(true)      // 用 ; 拼接的作者字符串
  . $this->originalContent()   // 不含 FULLCONTENT 附加段
  . $this->tags(true)          // #tag1 #tag2 格式
  . json_encode(enclosures)    // 附件元数据
)
```

> **故意不含 `date` 字段**，因为 RSS 源常缺发布时间，FreshRSS 会用 `time()` 自动填充，会造成"内容没变但 hash 变了"的假更新。

在 [feedController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Controllers/feedController.php) L631-L691 完成双层判定：

```php
$existingHashForGuids = $entryDAO->listHashForFeedGuids($feed->id(), $newGuids);

foreach ($entries as $entry) {
    if (isset($existingHashForGuids[$entry->guid()])) {
        // GUID 已存在 -> 比较 hash
        if (strcasecmp($existingHash, $entry->hash()) !== 0) {
            // 内容有变化 → updateEntry()
        } else {
            // 完全相同 → 跳过，只更新 lastSeen
        }
    } else {
        // 新 GUID → addEntry() 入库
    }
}
```

批量查询 `listHashForFeedGuids()` 避免 N+1 SQL。

---

### 3.3 PHP 层额外去重

- **同 Feed 内 GUID 重复** ([feedController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Controllers/feedController.php) L638-L641)：
  `$newGuids[$entry->guid()]` 已存在则 `continue`，取 SimplePie 中最早出现的。
- **DB 唯一约束兜底**：`_entry(id_feed, guid)` 与 `_entrytmp(id_feed, guid)` 有 UNIQUE INDEX，`INSERT IGNORE` 最后防线。

---

## 四、入库链路与落库边界

### 4.1 两阶段写入流程

```
  ┌──────────────────────────────┐
  │    actualizeFeeds()           │
  │  循环每个 Feed：              │
  │    • 对新/更新条目调用：       │
  │       addEntry(tmp=true)      │
  │       updateEntry()           │
  │    • updateLastSeen()         │
  └──────────────┬───────────────┘
                 ▼
  ┌──────────────────────────────┐
  │  actualizeFeedsAndCommit()    │
  │  所有 Feed 处理完后：          │
  │    beginTransaction()         │
  │    commitNewEntries()         │◀── 临时表 → 主表迁移点
  │    applyLabelActions()        │
  │    keepMaxUnreads()           │
  │    updateCachedValues()       │
  │    commit()                   │◀── 最终原子落库边界
  └──────────────────────────────┘
```

### 4.2 第一阶段：写入 `_entrytmp` 临时表

**方法**: [EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Models/EntryDAO.php) 的 `addEntry()` (L192-L263)

```php
INSERT IGNORE INTO `_entrytmp` (
    id, guid, title, author, content_bin, link, date, lastSeen,
    hash, is_read, is_favorite, id_feed, tags, attributes
) VALUES (:id, ...);
```

字段预处理 (落库前的最后边界)：

| 字段 | 处理 | 边界值 |
|------|------|--------|
| `guid` | `safe_ascii()` + substr | 最长 767 字节 |
| `title` | `safe_utf8()` + `mb_strcut` | 最长 8192 UTF-8 字符 |
| `author` | 同上 | 1024 UTF-8 字符 |
| `content` | `safe_utf8()` | 无显式长度限制；超长会触发 MEDIUMBLOB 升级 |
| `link` | `safe_ascii()` + substr | 16383 字节 |
| `tags` | `safe_utf8()` + `mb_strcut` | 2048 UTF-8 字符 |
| `attributes` | `json_encode(JSON_UNESCAPED_SLASHES \| UNICODE)` | NULL 转 `[]` |
| `hash` | `UNHEX(:hash)` 或 `hex2bin` | MySQL 原生 hex，其他用参数绑定 |
| `content` | MySQL 用 `COMPRESS(:content)` → `content_bin` | |

错误分类 (L254-L260)：
- **SQLSTATE Class 23 约束违反** (重复键)：静默 `return false`，不打日志 (属于正常去重)
- 其他错误：`Minz_Log::error`，并触发 `autoUpdateDb` 尝试加列后重试

---

### 4.3 第二阶段：`_entrytmp` → `_entry` 主表迁移

**方法**: [EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Models/EntryDAO.php) 的 `commitNewEntries()` (L265-L288)

```sql
SET @rank = (SELECT MAX(id) - COUNT(*) FROM `_entrytmp`);

INSERT IGNORE INTO `_entry` (...)
SELECT @rank := @rank + 1 AS id,
       guid, title, author, content_bin, link, date, lastSeen,
       hash, is_read, is_favorite, id_feed, tags, attributes
FROM `_entrytmp` etmp
ORDER BY etmp.date, etmp.id;

DELETE FROM `_entrytmp` WHERE id <= @rank;
```

关键设计：
- **ID 重分配**：`_entrytmp` 的 ID 是临时的，迁移时按 `(date, tmp_id)` 排序后连续递增生成，保证主表 ID 反映时间序
- **幂等**：`INSERT IGNORE` 保证并发安全，已被其他进程写入的不会重复
- **事务**：整个块必须在外部事务中，与后续缓存更新原子提交

### 4.4 updateEntry 直接落主表 (不经过 tmp)

**方法**: [EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Models/EntryDAO.php) 的 `updateEntry()` (L297-L396)

```sql
UPDATE `_entry`
SET title=?, author=?, content_bin=COMPRESS(?), ... ,
    lastModified=COALESCE(:last_modified, lastModified),
    is_read=COALESCE(:is_read, is_read),        -- NULL=保留原值
    is_favorite=COALESCE(:is_favorite, is_favorite)
WHERE id_feed=:id_feed AND guid=:guid
```

更新时不改动 `guid` 和 `id_feed` (主键组成)。收藏状态永不自动覆盖；`is_read` 按 `mark_updated_article_unread` 配置决定。

---

### 4.5 何时触发 commitNewEntries

**方法**: [feedController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Controllers/feedController.php) 的 `actualizeFeedsAndCommit()` (L921-L942)

触发条件只有一个：`$nbNewArticles > 0`（即存在新 GUID 的条目被写入 tmp 表）。

完整的事务包裹：

```
① entryDAO->beginTransaction()
    ② commitNewEntries()           // tmp → _entry
    ③ applyLabelActions(N)         // 为新条目批量打标签 (最多 N 条)
    ④ keepMaxUnreads(feeds)        // 超过 keep_max_n_unread 则标旧的为已读
    ⑤ updateCachedValues(feedIds)  // 更新 _feed.cache_nbEntries/cache_nbUnreads
⑥ entryDAO->commit()              // 以上所有变更原子提交
  └─ 失败则整体回滚
```

> **注意**：`updateEntry()` 产生的 UPDATE 不在 `$nbNewArticles` 判断内，**不需要 commitNewEntries**，但仍在外层事务中一并提交。

### 4.6 独立的 lastSeen 刷新边界

即使条目没有任何内容变更，**本次刷新出现的所有 GUID** 都会单独刷新 `lastSeen`：

- [feedController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Controllers/feedController.php) L739: `updateLastSeen($feedId, array_keys($newGuids), $mtime)`
- 若缓存未变 (`$feedIsUnchanged=true`)，L742 用 `updateLastSeenUnchanged($feedId, $mtime)` 把该 feed 所有条目 lastSeen 刷到当前时间
- 这是 `cleanOldEntries` 按 `lastSeen` 清理的依据

---

### 4.7 后处理边界 (post-commit)

在 `actualizeFeeds()` 内、`commitNewEntries` 前后交错执行的非核心数据操作：

| 操作 | 触发频率 | 作用 |
|------|---------|------|
| `markAsReadUponGone` | 每次刷新 | feed 为 empty 或条目消失时按策略标读 |
| `cleanOldEntries` | 随机 1/30 | 按 `archiving` 配置删除过期条目 |
| `updateLastUpdate` | 每次刷新成功 | 写 `_feed.lastUpdate = mtime`，清 error=0 |
| `feedDAO->updateFeed` (属性) | URL 变/图标变/空名等 | 刷新 feed 自身元数据 |
| `pubSubHubbubSubscribe` | 首次发现 hub | 发起 WebSub 订阅 |
| `minorDbMaintenance` | 非 noCommit 且非单 feed | DB 层小优化 |
| `refreshDynamicOpmls` | id=0 全量刷新 | 动态 OPML 源刷新 |
| `cleanCache` | 随机 1/30 | 删除旧的 SimplePie 缓存文件 |

---

## 五、扩展 Hook 插入点

在关键链路中可通过扩展接管或修改流程：

| Hook 名 | 位置 | 可用操作 |
|---------|------|---------|
| `FeedsListBeforeActualize` | actualizeFeeds 开头 | 修改待刷新 Feed 列表 |
| `FeedBeforeActualize` | 每个 Feed 处理前 | 修改 Feed 对象；返回 null 跳过 |
| `SimplepieBeforeInit` | SimplePie 初始化前后 | 注入自定义解析配置 |
| `SimplepieAfterInit` | | 读取解析结果 |
| `EntryBeforeInsert` | 每条 Entry 首次处理 | 改写 Entry；返回 null 丢弃 |
| `EntryBeforeAdd` | 确定新增后、落库前 | 最后机会改写 |
| `EntryBeforeUpdate` | 确定更新后、落库前 | 改写要更新的 Entry |
| `EntryAutoRead` | applyFilterActions | 触发自动标读 (reception/title/guid) |
| `EntryAutoUnread` | 更新判定为更新时 | 触发自动标未读 |
| `EntriesFavorite` | markFavorite | 收藏变更通知 |
| `CheckUrlBeforeAdd` | addFeed 最早 | 校验/改写订阅 URL |
| `FeedBeforeInsert` | addFeed 入库前 | 最后改写 Feed |
| `FreshrssUserMaintenance` | CLI/批量刷新开始前 | 自定义维护任务 |

---

## 六、完整调用图总览

```
用户 / Cron / WebSub / JS
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│ actualizeAction  │  actualize-user.php  │  pshb.php      │
│ addFeed          │  actualize_script.php│                │
└────────────────────┬─────────────────────────────────────┘
                     ▼
        actualizeFeedsAndCommit()  ──────────────┐
                     │                            │
                     ▼                            │
           ┌── actualizeFeeds() ──┐               │
           │  1. 过滤 Feeds       │               │
           │  2. SimplePie 解析   │               │
           │  3. loadGuids()      │               │
           │     └ decideEntryGuid 降级           │
           │  4. loadEntries()                    │
           │     └ Entry->hash() (md5)            │
           │  5. listHashForFeedGuids (批量 SQL)  │
           │  6. 每条 Entry:                      │
           │     ├ 新 GUID ──▶ addEntry(_entrytmp)│
           │     └ 内容变 ──▶ updateEntry(_entry) │
           │  7. updateLastSeen                   │
           │  8. 概率 cleanOldEntries             │
           └───────────────────┬──────────────────┘
                               │
                               ▼
              ┌───────────────────────────────┐
              │ ① beginTransaction()          │
              │ ② commitNewEntries()          │
              │    └ _entrytmp ─▶ _entry      │
              │ ③ applyLabelActions()         │
              │ ④ keepMaxUnreads()            │
              │ ⑤ updateCachedValues()        │
              │ ⑥ commit()                    │◀── 原子提交边界
              └───────────────────────────────┘
```
