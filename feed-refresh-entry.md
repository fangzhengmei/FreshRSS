# FreshRSS Feed 刷新与 Entry 入库链路分析

## 一、触发入口矩阵

FreshRSS 通过以下 **8 种独立入口** 触发 Feed 刷新 / 条目入库：

---

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

---

### 1.2 添加 Feed 时首次刷新
**入口**: [feedController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Controllers/feedController.php) 的 `addFeed()` 静态方法 (L39-L117)

在 `addAction()` 完成数据库插入后，立即调用：
```php
self::actualizeFeedsAndCommit($id, $url);
```
是 **同步阻塞** 的，会等待新 Feed 的第一批条目入库完成后才返回给用户。

---

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

---

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

---

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

---

### 1.6 JavaScript 后台自动刷新
**入口**: `actualize.phtml` 视图 + `javascriptController`

由浏览器前端 JS 定时轮询，参数组合与 1.1 的 AJAX 模式相同。

---

### 1.7 导入订阅时批量入库
**入口**: [importExportController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Controllers/importExportController.php) + CLI [import-for-user.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/cli/import-for-user.php)

支持两种调用方式：
- **Web**: `?c=importExport&a=import` (POST 上传文件)
- **CLI**: `./cli/import-for-user.php --user xxx --filename xxx.zip`

支持的文件格式：

| 格式 | 说明 |
|------|------|
| OPML (`.opml` / `.xml`) | 只导入 Feed 列表（分类 + URL），不导入条目 |
| JSON starred | Google Reader 格式的收藏/标注条目 |
| JSON feed | 包含完整文章内容的 JSON 文件 |
| TXT | 纯文本每行一个 URL，内部转成 OPML 再导入 |
| TT-RSS XML | TinyTiny RSS 导出的 starred 条目，内部转 JSON 再导入 |
| ZIP | 以上格式打包，按文件名自动识别类型 |

导入顺序 (L111-L113)：**OPML 先** → 收藏/标签中 → 其他文章最后，确保 feed 存在后再导入条目。

#### JSON 条目导入的核心链路
[importExportController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Controllers/importExportController.php) L331-L592 `importJson()`:

```
① 遍历 items，按 origin.feedUrl 查找/创建 Feed
② 按 feed 分组收集所有 guid
③ 批量 listHashForFeedGuids() 查已存在条目
④ beginTransaction  ← 第一事务开始
⑤ 逐条 Entry 判定：
   • 新 GUID → addEntry(tmp=true) 写 _entrytmp
   • 已存在 → updateEntry() 直接更 _entry
⑥ commit  ← 第一事务边界（条目落库）

⑦ beginTransaction  ← 第二事务开始
⑧ commitNewEntries()  ← tmp → 主表迁移
⑨ updateCachedValues()
⑩ commit  ← 第二事务边界

⑪ beginTransaction  ← 第三事务开始
⑫ 为每条新条目查 ID → tagEntry() 打标签
⑬ commit  ← 第三事务边界（标签关联）
```

**与常规刷新的关键区别**：
- 不经过 `actualizeFeeds()` 和 `decideEntryGuid()`，**直接使用源文件中的 guid/id**
- 不执行全文抓取、lastSeen 刷新、归档清理等后处理
- 事务粒度更细（3 段式事务），避免大事务锁定过长
- `is_read`/`is_favorite` 状态从导入文件中读取，不应用自动标记规则

---

### 1.8 重新抓取 (Reload Articles)
**入口**: [feedController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Controllers/feedController.php) 的 `reloadAction()` (L1208-L1264)

路由：`?c=feed&a=reload` (POST)，参数：
- `id`: Feed ID (必填)
- `reload_limit`: 重新抓取的文章数量，默认 10

分为 **两个独立阶段**：

**阶段一：强制刷新 Feed**
```php
$feedDAO->updateFeed($feed->id(), ['lastUpdate' => 0]);  // 归零，绕过 TTL
self::actualizeFeedsAndCommit($feed_id);                  // 走完整刷新链路
```
把 `lastUpdate` 设为 0，强制绕过 TTL 检查，等同于把 Feed 当成新订阅重新拉一次。

**阶段二：全文内容重抓**
```php
$entries = $entryDAO->listWhere('f', $feed_id, STATE_ALL, order: 'DESC', limit: $limit);
foreach ($entries as $entry) {
    if ($entry->loadCompleteContent(true)) {    // force=true 强制重抓
        $entry->_lastModified(time());
        if (内容有变化) {
            $entryDAO2->updateEntry($entry->toArray());
        }
    }
}
```

**隐式自动重抓**（在常规刷新中）：
在 [feedController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Controllers/feedController.php) L681-L682，当检测到条目内容变化 (hash 不同) 时，会自动触发：
```php
// If the entry has changed, there is a good chance for the full content to have changed as well.
$entry->loadCompleteContent(true);
```

**技术细节**：
- **MySQL**：用第二个独立 PDO 连接（关闭共享 PDO）做非缓冲查询流式处理
- **SQLite / PostgreSQL**：单连接内存缓冲查询
- 只更新 `content` 字段，**不会触发 hash 变化**（因为 hash 用的是 `originalContent()`，FULLCONTENT 追加段不计入）

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
public function hash(): string {
    if ($this->hash === '') {
        $attributes = empty($this->attributeArray('enclosures'))
            ? ''
            : json_encode($this->attributes(), JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE);
        //Do not include $this->date because it may be automatically generated when lacking
        $this->hash = md5(
            $this->link
          . $this->title
          . $this->authors(true)
          . $this->originalContent()
          . $this->tags(true)
          . $attributes
        );
    }
    return $this->hash;
}
```

#### 各字段的准确含义（修正版）：

| 组成部分 | 方法 / 表达式 | 实际格式 | 说明 |
|---------|------|---------|------|
| `link` | `$this->link` | 原始字符串 | trim 后的链接 |
| `title` | `$this->title` | 原始字符串 | trim 后的标题 |
| `authors` | `$this->authors(true)` | `;作者1; 作者2` | 分号分隔，**以分号开头**；空数组返回空串 |
| `content` | `$this->originalContent()` | 原始 HTML | **不含 FULLCONTENT 扩展追加的全文** |
| `tags` | `$this->tags(true)` | `#tag1 #tag2` | 空格分隔，每个 tag 前缀 `#`；空数组返回空串 |
| `attributes` | 条件表达式 | JSON 串 或 空串 | **不是只有 enclosures**：enclosures 为空则传空串 `""`；否则传 **整个 attributes 数组** 的 JSON |

#### `originalContent()` 的完整逻辑 ([Entry.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Models/Entry.php) L210-L213)：
```php
public function originalContent(): string {
    return $this->attributeString('original_content') ??
        preg_replace('#<!-- FULLCONTENT start //-->.*<!-- FULLCONTENT end //-->#s', '', $this->content) ?? '';
}
```
优先级：
1. 优先使用 `original_content` 属性（CSS/XPath 全文抓取时备份的原始 RSS 摘要）
2. 否则用正则剥离 `<!-- FULLCONTENT start //-->...<!-- FULLCONTENT end //-->` 段
3. 结果为 `null` 则返回空串

> **故意不含 `date` 字段**：因为 RSS 源常缺发布时间，FreshRSS 会用 `time()` 自动填充，会造成"内容没变但 hash 变了"的假更新。

> **FULLCONTENT 排除的意义**：`hash()` 使用 `originalContent()` 而不是 `content()`，因此通过 CSS/XPath 全文抓取扩展追加的内容**不会**改变 hash，避免每次全文重抓都触发假更新。

在 [feedController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Controllers/feedController.php) L631-L691 完成双层判定：

```php
$existingHashForGuids = $entryDAO->listHashForFeedGuids($feed->id(), $newGuids);

foreach ($entries as $entry) {
    if (isset($existingHashForGuids[$entry->guid()])) {
        // GUID 已存在 -> 比较 hash
        if (strcasecmp($existingHash, $entry->hash()) !== 0) {
            // 内容有变化 → updateEntry()
            // 注：变化时会自动调用 loadCompleteContent(true) 重抓全文
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
// SQL 模板（由各数据库的 sqlIgnoreConflict/isCompressed/sqlHexDecode 注入差异）
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

## 五、多数据库落库差异

FreshRSS 支持 MySQL/MariaDB、SQLite、PostgreSQL 三种数据库后端，通过 DAO 继承链实现差异：

```
EntryDAO (基类，MySQL 实现)
    ├── EntryDAOSQLite      // SQLite 覆盖
    └── EntryDAOPGSQL       // PostgreSQL 覆盖（继承 SQLite）
```

核心差异通过以下模板方法注入：
- `sqlIgnoreConflict(string $sql)` - 改写 INSERT 语句的冲突处理
- `isCompressed()` - 是否使用 COMPRESS 压缩内容
- `hasNativeHex()` - 是否支持 SQL 原生 hex 解码
- `sqlHexDecode(string $x)` - hash 字段的解码函数
- `commitNewEntries()` - 临时表→主表迁移算法（三数据库完全不同）

### 5.1 核心差异总表

| 特性 | MySQL / MariaDB<br>([EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Models/EntryDAO.php)) | SQLite<br>([EntryDAOSQLite.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Models/EntryDAOSQLite.php)) | PostgreSQL<br>([EntryDAOPGSQL.php](file:///d:/fz/0601-1/solo-dogfeeding/code/21-FreshRSS/app/Models/EntryDAOPGSQL.php)) |
|------|---------|--------|------------|
| **内容压缩** | `COMPRESS(:content)` → `content_bin` | 不压缩 → `content` TEXT | 不压缩 → `content` TEXT |
| **isCompressed()** | `true` | `false` | `false` (继承 SQLite) |
| **Hash 解码** | `UNHEX(:hash)` (SQL 原生) | PHP `hex2bin()` 后绑定 | `decode(:hash, 'hex')` (SQL 原生) |
| **hasNativeHex()** | `true` | `false` | `true` |
| **冲突忽略语法** | `INSERT IGNORE INTO` | `INSERT OR IGNORE INTO` | `... ON CONFLICT DO NOTHING` |
| **commitNewEntries 算法** | 用户变量 `@rank` | 临时表 + `rowid` | PL/pgSQL `DO $$` 块 + `row_number() OVER()` |
| **随机函数** | `RAND()` | `RANDOM()` | `RANDOM()` |
| **字符串拼接** | `CONCAT(s1, s2)` | `s1 \|\| s2` | 继承 SQLite |
| **正则匹配** | `REGEXP ?` | `REGEXP ?` + PHP 回调注册 | `~ ?` / `~* ?` + `\b→\y` 转义 |
| **LIMIT ALL** | 省略或 `LIMIT 18446744073709551615` | `LIMIT -1` | `LIMIT ALL` |
| **autoUpdateDb 检测** | SQLSTATE 42S22 查错信息 | `PRAGMA table_info('entry')` | 错误码 `UNDEFINED_COLUMN` + 文本匹配 |

### 5.2 addEntry 具体差异

**MySQL** (基类默认实现)：
```php
// sqlIgnoreConflict 替换: INSERT INTO → INSERT IGNORE INTO
// isCompressed()=true: 列名用 content_bin，值用 COMPRESS(:content)
// sqlHexDecode(':hash') = UNHEX(:hash)
// hasNativeHex()=true: hash 直接绑定 hex 字符串
INSERT IGNORE INTO `_entrytmp` (id, guid, ..., content_bin, ..., hash, ...)
VALUES (:id, :guid, ..., COMPRESS(:content), ..., UNHEX(:hash), ...);
```

**SQLite** (调用 `sqlIgnoreConflict` + PHP 侧 hex2bin)：
```php
// sqlIgnoreConflict 替换: INSERT INTO → INSERT OR IGNORE INTO
// isCompressed()=false: 列名用 content，值用 :content
// sqlHexDecode(':hash') = ':hash' (原样输出)
// hasNativeHex()=false: hash 先经 PHP hex2bin() 转二进制再绑定
INSERT OR IGNORE INTO `_entrytmp` (id, guid, ..., content, ..., hash, ...)
VALUES (:id, :guid, ..., :content, ..., :hash_bin, ...);
```

**PostgreSQL** (继承 SQLite 的内容列，但用原生 decode)：
```php
// sqlIgnoreConflict 追加: ... ON CONFLICT DO NOTHING
// isCompressed()=false (继承 SQLite): 列名用 content
// sqlHexDecode(':hash') = decode(:hash, 'hex')
// hasNativeHex()=true: hash 直接绑定 hex 字符串
INSERT INTO `_entrytmp` (...) VALUES (...) ON CONFLICT DO NOTHING;
```

### 5.3 commitNewEntries 具体差异

**MySQL**（用户变量法，单 SQL 事务）：
```sql
SET @rank = (SELECT MAX(id) - COUNT(*) FROM `_entrytmp`);
INSERT IGNORE INTO `_entry` (...)
    SELECT @rank := @rank + 1 AS id, ... FROM `_entrytmp` ORDER BY date, id;
DELETE FROM `_entrytmp` WHERE id <= @rank;
```
- 利用 MySQL 用户变量 `@rank` 自增生成连续 ID
- 依赖外部事务保证原子性；若外部无事务则方法内部自动开启/提交

**SQLite**（临时表 + rowid 法，4 条 SQL）：
```sql
DROP TABLE IF EXISTS `tmp`;
CREATE TEMP TABLE `tmp` AS SELECT ... FROM `_entrytmp` ORDER BY date, id;
INSERT OR IGNORE INTO `_entry`
    SELECT rowid + (SELECT MAX(id) - COUNT(*) FROM `tmp`) AS id, ...
    FROM `tmp` ORDER BY date, id;
DELETE FROM `_entrytmp` WHERE id <= (SELECT MAX(id) FROM `tmp`);
DROP TABLE IF EXISTS `tmp`;
```
- 用临时表缓存排序结果，利用 SQLite 内置 `rowid` 生成连续 ID
- 方法内部自带事务保护（检测外部无事务则开启）

**PostgreSQL**（PL/pgSQL + 窗口函数法，匿名块）：
```sql
DO $$
DECLARE
    maxrank bigint := (SELECT MAX(id) FROM `_entrytmp`);
    rank bigint := (SELECT maxrank - COUNT(*) FROM `_entrytmp`);
BEGIN
    INSERT INTO `_entry` (...)
    (SELECT rank + row_number() OVER(ORDER BY etmp.date, etmp.id) AS id, ...
     FROM `_entrytmp` AS etmp
     WHERE NOT EXISTS (
         SELECT 1 FROM `_entry` AS ereal
         WHERE (etmp.id = ereal.id)
            OR (etmp.id_feed = ereal.id_feed AND etmp.guid = ereal.guid))
     ORDER BY etmp.date, etmp.id);
    DELETE FROM `_entrytmp` WHERE id <= maxrank;
END $$;
```
- 用 `row_number() OVER()` 窗口函数生成连续 ID
- **去重方式特殊**：用 `WHERE NOT EXISTS` 子查询双重判断 (id 冲突 + guid+feedId 冲突)，而不是 `ON CONFLICT DO NOTHING`（代码注释 TODO：升级到 PG 9.5+ 语法）
- PL/pgSQL 块本身就是原子的，不需要额外事务

### 5.4 其他重要差异

**`markRead` 实现**：
- MySQL：批量 ID 一次 `UPDATE ... WHERE id IN (...)`，性能最优
- SQLite：循环逐个 ID 更新（每个都有独立子事务 + 缓存更新），批量性能较差
- PostgreSQL：继承 SQLite 实现

**正则函数注册**：
- MySQL：`REGEXP` 是内置的，无需额外操作
- SQLite：执行前检测 `REGEXP` 关键字，用 `sqliteCreateFunction` 注册 PHP 回调实现正则匹配
- PostgreSQL：无需注册，用 `~` (区分大小写) / `~*` (不区分) 操作符，且把 PCRE 语法 `\b` → `\y`、`\B` → `\Y` 做转义

**autoUpdateDb 列缺失检测**：
- MySQL：捕获 SQLSTATE `42S22` (ER_BAD_FIELD_ERROR)，解析错误信息中的列名
- SQLite：每次出错都执行 `PRAGMA table_info('entry')` 检查表结构
- PostgreSQL：捕获错误码 `UNDEFINED_COLUMN`，在错误信息第一行匹配列名

---

## 六、扩展 Hook 插入点

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

## 七、完整调用图总览

```
用户 / Cron / WebSub / JS / 导入 / 重抓
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│ actualizeAction  │  actualize-user.php  │  pshb.php      │
│ addFeed          │  actualize_script.php│  importJson    │
│ reloadAction     │                      │                │
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
           │        (自动 loadCompleteContent)    │
           │  7. updateLastSeen                   │
           │  8. 概率 cleanOldEntries             │
           └───────────────────┬──────────────────┘
                               │
                               ▼
              ┌───────────────────────────────┐
              │ ① beginTransaction()          │
              │ ② commitNewEntries()          │
              │    └ _entrytmp ─▶ _entry      │
              │    (MySQL/SQLite/PG 各不同)   │
              │ ③ applyLabelActions()         │
              │ ④ keepMaxUnreads()            │
              │ ⑤ updateCachedValues()        │
              │ ⑥ commit()                    │◀── 原子提交边界
              └───────────────────────────────┘
```
