# FreshRSS 全文抓取工作流程

本文档梳理 FreshRSS 中 RSS 全文抓取的完整代码流转，包括触发条件、正文清洗与缓存、展示层合并策略，以及页面重定向、附件去重、描述渲染等细节。

---

## 一、整体架构概览

```
触发入口 → Feed刷新 → 解析RSS条目 → 【条件判断】 → 全文抓取 → 内容清洗 → 缓存 → 合并策略 → 入库 → 展示
                              ↓（不需要全文）
                           直接入库
```

**核心组件**：

| 组件 | 文件路径 | 职责 |
|------|----------|------|
| 实际化脚本 | [actualize_script.php](app/actualize_script.php) | CLI 定时刷新入口 |
| Feed 控制器 | [feedController.php](app/Controllers/feedController.php) | 触发 actualize 动作 |
| Feed 模型 | [Feed.php](app/Models/Feed.php) | 订阅源加载、条目解析、缓存管理 |
| Entry 模型 | [Entry.php](app/Models/Entry.php) | 全文抓取 `loadCompleteContent()`、内容解析 `getContentByParsing()` |
| SimplePie 定制 | [SimplePieCustom.php](app/Models/SimplePieCustom.php) | HTML 内容清洗 `sanitizeHTML()` |
| HTTP 工具 | [httpUtil.php](app/Utils/httpUtil.php) | HTTP 请求、缓存层、编码与重定向处理 |
| 视图层 | [normal.phtml](app/views/index/normal.phtml) | 内容展示 `$entry->content(true)` |

---

## 二、触发条件（何时启动全文抓取）

### 2.1 四大触发入口

#### 入口 1：CLI 定时任务（最常用）

**文件**: [actualize_script.php#L13-L16](app/actualize_script.php#L13-L16)

```php
$_GET['c'] = 'feed';
$_GET['a'] = 'actualize';   // 路由到 feedController::actualizeAction
$_GET['ajax'] = 1;
$_GET['maxFeeds'] = PHP_INT_MAX;
```

- 通过系统 cron 定时调用
- 遍历所有用户，逐用户执行刷新
- 有互斥锁（`actualize.freshrss.lock`）防止并发

#### 入口 2：Web 界面手动刷新

**文件**: [feedController.php#L954-L1033](app/Controllers/feedController.php#L954-L1033)

```php
public function actualizeAction(): int {
    // 支持参数：id（单个Feed）、maxFeeds、noCommit 等
    [$nbUpdatedFeeds, $feed, $nbNewArticles, $feedsCacheToRefresh] = 
        self::actualizeFeeds($id, $url, $maxFeeds);
}
```

- 用户点击界面"刷新"按钮
- 支持单 Feed 刷新或批量刷新

#### 入口 3：添加新 Feed 时

**文件**: [feedController.php#L115](app/Controllers/feedController.php#L115)

```php
// Ok, feed has been added in database. Now we have to refresh entries.
self::actualizeFeedsAndCommit($id, $url);
```

- 订阅新源时立即触发一次完整抓取

#### 入口 4：WebSub（PubSubHubbub）实时推送

**文件**: [p/api/pshb.php](p/api/pshb.php)

- 支持 WebSub 的源直接推送更新
- 调用 `actualizeFeedsAndCommit()`，传入 `$simplePiePush`

---

### 2.2 actualizeFeeds 核心流程

**文件**: [feedController.php#L427-L849](app/Controllers/feedController.php#L427-L849)

#### Step 1：筛选需要刷新的 Feed

```php
// TTL 检查：是否到了刷新时间
$ttl = $feed->ttl() === FreshRSS_Feed::TTL_DEFAULT 
    ? FreshRSS_Context::userConf()->ttl_default 
    : $feed->ttl();

if (time() <= $feed->lastUpdate() + $ttl) {
    continue;  // 未到刷新时间，跳过
}
```

**跳过条件**：
- Feed 被静音（mute=true）且非手动指定刷新
- PubSubHubbub 启用且 24h 内活跃的源（减少拉取频率）
- 未到 TTL 刷新周期，且其他用户没有更新缓存
- 无法获取锁（已有进程在刷新该 Feed）

#### Step 2：加载 Feed 内容

**文件**: [Feed.php#L601-L693](app/Models/Feed.php#L601-L693)

```php
public function load(bool $loadDetails = false, bool $noCache = false): ?FreshRSS_SimplePieCustom {
    $simplePie = new FreshRSS_SimplePieCustom($this->attributes(), $this->curlOptions());
    $simplePie->set_feed_url($url);
    $simplePieResult = $simplePie->init();  // SimplePie 内部有缓存
    
    // 比较 SimplePieHash 判断 Feed 是否有更新
    if ($simplePie->get_hash() !== $this->attributeString('SimplePieHash')) {
        $this->_attribute('SimplePieHash', $simplePie->get_hash());
        return $simplePie;  // 有新内容
    }
    return null;  // 无变化，使用缓存
}
```

根据 Feed 类型使用不同加载方式：
| kind 类型 | 方法 | 说明 |
|-----------|------|------|
| KIND_RSS (0) | `$feed->load()` | 标准 RSS/Atom，SimplePie 解析 |
| KIND_HTML_XPATH (10) | `$feed->loadHtmlXpath()` | HTML 页面 + XPath 抓取 |
| KIND_XML_XPATH (15) | `$feed->loadHtmlXpath()` | XML + XPath |
| KIND_JSON_DOTNOTATION (30) | `$feed->loadJson()` | JSON + 点号语法 |
| KIND_JSONFEED (25) | `$feed->loadJson()` | 标准 JSON Feed 格式 |

#### Step 3：解析并生成 Entry 对象

**文件**: [Feed.php#L807-L942](app/Models/Feed.php#L807-L942)

```php
public function loadEntries(FreshRSS_SimplePieCustom $simplePie): Traversable {
    $items = $simplePie->get_items();
    for ($i = count($items) - 1; $i >= 0; $i--) {
        $item = $items[$i];
        $content = html_only_entity_decode($item->get_content());
        
        $entry = new FreshRSS_Entry(
            $this->id(), $guid, $title, $authorNames,
            $content, $link, $date
        );
        $entry->hash();  // 计算内容哈希
        $entry->loadCompleteContent();  // ⭐ 【关键】尝试加载全文！
        
        yield $entry;
    }
}
```

---

## 三、全文抓取核心流程

### 3.1 `loadCompleteContent()` — 决策与调度中心

**文件**: [Entry.php#L1063-L1145](app/Models/Entry.php#L1063-L1145)

```php
public function loadCompleteContent(bool $force = false): bool {
    $feed = $this->feed();
    
    // ⚠️ 触发全文抓取的必要条件：pathEntries 非空
    if (trim($feed->pathEntries()) != '') {
        // 1. 若文章已在DB中，直接复用已有内容（避免重复HTTP请求）
        $entry = $force ? null : $entryDAO->searchByGuid($this->feedId, $this->guid);
        if ($entry !== null) {
            $this->content = $entry->content(false);
            return false;
        }
        
        // 2. 新文章，执行全文抓取
        $fullContent = $this->getContentByParsing();
        if ('' !== $fullContent) {
            // 3. 包裹标记，便于区分全文和原始内容
            $fullContent = "<!-- FULLCONTENT start //-->{$fullContent}<!-- FULLCONTENT end //-->";
            $originalContent = $this->originalContent();
            
            // 4. 根据 content_action 策略合并
            switch ($feed->attributeString('content_action')) {
                case 'prepend':  // 全文在前，摘要在后
                    $this->_attribute('original_content');
                    $this->content = $fullContent . $originalContent;
                    break;
                case 'append':   // 摘要在前，全文在后
                    $this->_attribute('original_content');
                    $this->content = $originalContent . $fullContent;
                    break;
                case 'replace':  // 用全文替换摘要（默认）
                default:
                    $this->_attribute('original_content', $originalContent);
                    $this->content = $fullContent;
                    break;
            }
            return true;
        }
    } 
    // 【分支B】仅使用 path_entries_filter 过滤已有内容
    elseif (trim($feed->attributeString('path_entries_filter') ?? '') !== '') {
        // 对 RSS 原内容执行 CSS 选择器过滤（不发HTTP请求）
    }
    // 【分支C】不做任何全文处理，保持 RSS 摘要
    return false;
}
```

### 3.1.1 强制重抓（`$force=true`）与多层缓存的交互

`loadCompleteContent(bool $force = false)` 中的 `$force` 参数是单篇强制重抓的开关，但它**仅穿透 DB 层**，与其它缓存层的关系需要明确区分。

#### 三种"缓存跳过/清理"机制的差异

FreshRSS 中有三种容易被混淆的缓存操作，它们作用于不同层级，效果完全不同：

| 机制 | 作用层级 | 实际效果 | 代码位置 |
|------|----------|----------|----------|
| **单篇强制重抓** `loadCompleteContent(true)` | Layer 4 DB | 跳过 `searchByGuid()` DB 查询，但 HTTP 文件缓存、Retry-After 仍生效 | [Entry.php#L1072](app/Models/Entry.php#L1072) |
| **订阅源缓存跳过** `Feed::load($details, $noCache=true)` | Layer 1 SimplePie | `$noCache=true` 时即使 SimplePieHash 未变也返回 SimplePie 对象，但**不删除 `.spc` 缓存文件** | [Feed.php#L684](app/Models/Feed.php#L684) |
| **`clear_cache` 属性** `Feed::load()` 中检查 | Layer 1 SimplePie | 调用 `$this->clearCache()` **删除 `.spc` 文件** + 重建 favicon，但**不删除全文 HTML 缓存** | [Feed.php#L626-L628](app/Models/Feed.php#L626-L628) |

> ⚠️ **关键校正**：`clearCache()` [Feed.php#L1327-L1330](app/Models/Feed.php#L1327-L1330) **仅删除 Feed 级缓存文件**，**不删除全文抓取的 HTML 缓存文件（`.html`）**。
>
> 原因：`clearCache()` 调用 `@unlink($this->cacheFilename())`，而 `cacheFilename()` **无参数**时根据 Feed 类型返回不同扩展名的文件（`.spc` / `.html` / `.json` / `.xml`）[Feed.php#L1301-L1316](app/Models/Feed.php#L1301-L1316)。全文 HTML 缓存使用 `cacheFilename($url . '#' . $pathEntries)` 生成 `.html` 文件 [Entry.php#L950](app/Models/Entry.php#L950)，两者的 URL 参数不同（一个带 `#pathEntries`，一个不带），所以文件名不同，互不影响。

#### `clearCache()` 的实际行为

```php
// Feed.php#L1327-L1330
public function clearCache(): bool {
    $this->faviconRebuild();              // 重建 favicon
    return @unlink($this->cacheFilename()); // 仅删除 Feed 级缓存文件
}

// cacheFilename() 无参数时：根据 Feed kind 返回不同扩展名
// cacheFilename($url) 有参数时：始终返回 .html（全文抓取用）
public function cacheFilename(string $url = ''): string {
    $simplePie = new FreshRSS_SimplePieCustom(...);
    if ($url !== '') {
        $filename = $simplePie->get_cache_filename($url);  // URL 含 #pathEntries
        return CACHE_PATH . '/' . $filename . '.html';      // 全文 HTML 缓存
    }
    $url = htmlspecialchars_decode($this->url);              // Feed 原始 URL
    $filename = $simplePie->get_cache_filename($url);
    switch ($this->kind) {
        case KIND_HTML_XPATH:  return CACHE_PATH . '/' . $filename . '.html';
        case KIND_XML_XPATH:   return CACHE_PATH . '/' . $filename . '.xml';
        case KIND_JSON_*:      return CACHE_PATH . '/' . $filename . '.json';
        case KIND_RSS:
        default:               return CACHE_PATH . '/' . $filename . '.spc';
    }
}
```

> **核心区别**：Feed 级缓存和全文 HTML 缓存使用**不同的 URL** 作为 `get_cache_filename()` 的输入：
> - Feed 级：`$this->url`（Feed 订阅地址）
> - 全文 HTML：`$url . '#' . $pathEntries`（文章 URL + CSS 选择器哈希）
>
> 即使 Feed 类型是 `KIND_HTML_XPATH`（返回 `.html`），两者的文件名因 URL 不同而不会冲突。`clearCache()` 只删除 Feed 级缓存，全文 HTML 缓存不受影响。

#### 全文 HTML 缓存（`.html`）的清理方式

全文抓取产生的 `.html` 缓存文件**没有单独的删除方法**，只能通过以下方式清理：

| 方式 | 触发条件 | 代码位置 |
|------|----------|----------|
| 全局过期清理 | `cleanCache(CLEANCACHE_HOURS)` 随机触发（1/31 概率），默认清理 720 小时（30 天）以上的文件 | [lib_rss.php#L294-L306](lib/lib_rss.php#L294-L306) |
| 自然过期 | `httpGet()` 中 `cacheMtime > time() - cache_duration` 检查失败时重新请求 | [httpUtil.php#L275-L283](app/Utils/httpUtil.php#L275-L283) |

```php
// lib_rss.php#L294-L306 - 全局缓存清理
function cleanCache(int $hours = 720): void {
    $files = glob(CACHE_PATH . '/*.*', GLOB_NOSORT) ?: [];
    foreach ($files as $file) {
        if (str_ends_with($file, 'index.html')) continue;
        $cacheMtime = @filemtime($file);
        if ($cacheMtime !== false && $cacheMtime < time() - (3600 * $hours)) {
            unlink($file);  // 删除所有过期的 .spc / .html / .json 文件
        }
    }
}
```

#### `$force=true` 的触发场景（3 处调用）

| 场景 | 文件位置 | 说明 |
|------|----------|------|
| 文章内容已更新（hash 变化） | [feedController.php#L682](app/Controllers/feedController.php#L682) | `if (strcasecmp($existingHash, $entry->hash()) !== 0)` → `$entry->loadCompleteContent(true);` |
| 手动点击"重新加载"按钮 | [feedController.php#L1248](app/Controllers/feedController.php#L1248) | `reloadAction` 遍历 DB 中已有文章，逐篇强制重抓 |
| 新 Feed 添加 `$feedIsNew=true` | [feedController.php#L567](app/Controllers/feedController.php#L567) | `$simplePie = $feed->load(false, $feedIsNew);` 强制 Feed 级缓存失效 |

#### 四层缓存对 `$force` 的不同响应

```
┌──────────────────────────────────────────────────────────────────────┐
│                    $force=true 对缓存的穿透能力                       │
├──────────────┬──────────────────────────────┬─────────────────────────┤
│ 缓存层级     │ $force=true 时是否穿透       │ 代码位置                 │
├──────────────┼──────────────────────────────┼─────────────────────────┤
│ Layer 4 DB   │ ✅ 完全穿透（不复用DB内容）  │ Entry.php#L1072         │
│              │  $force ? null : searchByGuid()                        │
├──────────────┼──────────────────────────────┼─────────────────────────┤
│ Layer 3      │ ❌ 不穿透（仍检查限流）      │ httpUtil.php#L307       │
│ Retry-After  │  getRetryAfter() 检查生效                              │
├──────────────┼──────────────────────────────┼─────────────────────────┤
│ Layer 2      │ ❌ 不穿透（仍读文件缓存）    │ httpUtil.php#L275       │
│ HTTP HTML    │  cacheMtime 检查依然生效                               │
│ (.html)      │  ⚠️ clearCache() 不删除此文件                           │
├──────────────┼──────────────────────────────┼─────────────────────────┤
│ Layer 1      │ ⚠️  部分穿透（Feed 级）      │ Feed.php#L684           │
│ SimplePie    │  $noCache || SimplePieHash 变化则返回非 null             │
│ (.spc)       │  clear_cache 属性可删除 .spc 文件                       │
└──────────────┴──────────────────────────────┴─────────────────────────┘
```

**关键洞察**：
- `$force=true` **仅跳过 Layer 4 数据库内容复用**，HTTP 文件缓存（Layer 2 `.html`）和 Retry-After 限流（Layer 3）依然生效
- `clearCache()` **仅删除 Layer 1 的 `.spc` 文件**，**不删除** Layer 2 的 `.html` 全文缓存文件
- Feed 级 `$noCache` [Feed.php#L601, L684](app/Models/Feed.php#L601-L684) 仅影响 SimplePieHash 比较逻辑，不删除 SimplePie 的 `.spc` 缓存文件（避免多用户场景下缓存失效）
- 要强制重新抓取全文 HTML，只能等待 `limits.cache_duration` 过期（默认 3600s），或由全局 `cleanCache(720h)` 清理

### 3.2 触发全文抓取的**必要条件**

| 条件 | 位置 | 说明 |
|------|------|------|
| `pathEntries` 非空 | Feed 属性 `$_pathEntries` | **必需条件**：CSS 选择器，指定提取正文的 DOM 节点 |
| `path_entries_conditions` | Feed attributes | **可选**：布尔搜索条件，满足条件的文章才抓取全文 |
| 文章不在 DB 中 | `searchByGuid()` 返回 null | 避免重复抓取已入库文章（除非 `$force=true`） |
| 文章有可用的 link | `$this->link()` | 作为抓取目标 URL |

---

### 3.3 `getContentByParsing()` — 实际抓取与解析

**文件**: [Entry.php#L924-L1058](app/Models/Entry.php#L924-L1058)

```
文章链接 → [HTTP请求(含重定向)+缓存] → HTML内容 → [DOM解析] → [CSS选择器提取] → [过滤] → [HTML清洗] → 纯净全文
```

#### Step 1：条件检查

```php
// 检查 path_entries_conditions（可选）
$conditions = $feed->attributeArray('path_entries_conditions');
if (count($conditions) > 0) {
    $found = false;
    foreach ($conditions as $condition) {
        $booleanSearch = new FreshRSS_BooleanSearch($condition);
        if ($this->matches($booleanSearch)) {  // 文章需匹配搜索条件
            $found = true;
            break;
        }
    }
    if (!$found) return '';  // 不匹配条件则跳过
}
```

#### Step 2：HTTP 请求与文件缓存

```php
$cachePath = $feed->cacheFilename($url . '#' . $feed->pathEntries());
$response = FreshRSS_http_Util::httpGet(
    $url,           // 文章链接
    $cachePath,     // 缓存文件路径
    'html',         // 期望类型
    $feed->attributes(),
    $feed->curlOptions()
);
$html = $response['body'];
```

#### Step 3：页面重定向处理（HTTP 301/302 + HTML meta refresh）

这是本文档重点补充的部分。全文抓取会遇到两类重定向，代码分两层处理，并通过 `$maxRedirs` 参数精细控制总跳转次数。

##### `$maxRedirs` 的完整传递与递减机制

函数签名定义默认总预算为 **4 次**：

```php
// Entry.php#L924
public function getContentByParsing(string $url = '', int $maxRedirs = 4): string {
```

**调用链与递减过程**：

```
初始调用（无显式 maxRedirs）
    ↓
getContentByParsing()              $maxRedirs = 4
    │
    ├─ httpGet() → cURL 自动跟随 N 次 301/302
    │                  → 返回 redirect_count = N
    │
    ├─ $maxRedirs -= N              $maxRedirs = 4 - N
    │
    ├─ 若 $maxRedirs > 0，扫描 <meta refresh>
    │   找到跳转 URL
    │        ↓
    └─ 递归 getContentByParsing(newUrl, $maxRedirs - 1)
                                     $maxRedirs = (4 - N) - 1
```

**命中 HTTP 文件缓存时的特殊处理** [httpUtil.php#L281](app/Utils/httpUtil.php#L281)：
```php
// 若 HTTP 缓存命中（status=-200），redirect_count 固定返回 0
return ['body' => $body, 'effective_url' => $url, 'redirect_count' => 0, ...];
```
> 注意：缓存命中时没有实际的 HTTP 请求，`effective_url` 就是原始 URL，`redirect_count` 为 0，意味着缓存后的 meta refresh 检查有完整的 4 次预算。

##### Layer A：HTTP 协议级重定向（301/302）

由 cURL 自动处理，配置在 [httpUtil.php#L336-L386](app/Utils/httpUtil.php#L336-L386)：

```php
curl_setopt_array($ch, [
    CURLOPT_URL => $url,
    CURLOPT_MAXREDIRS => 4,           // cURL 层面最多 4 次 HTTP 重定向
    CURLOPT_FOLLOWLOCATION => true,   // 自动跟随 301/302
    CURLOPT_ACCEPT_ENCODING => '',    // 启用 gzip/deflate/br
]);

// 安全限制：仅允许 http/https 协议的重定向
if (defined('CURLOPT_PROTOCOLS_STR')) {
    $curl_options[CURLOPT_PROTOCOLS_STR] = 'http,https';
    $curl_options[CURLOPT_REDIR_PROTOCOLS_STR] = 'http,https';
}
```

**用户自定义 CURLOPT_MAXREDIRS**：
- Feed 级别的 `curl_params` 可覆盖默认值 4
- OPML 导入/导出支持 `frss:CURLOPT_MAXREDIRS` 属性 [opml.phtml#L122](app/views/helpers/export/opml.phtml#L122)
- 表单编辑位于 [update.phtml#L812](app/views/helpers/feed/update.phtml#L812)

cURL 请求完成后，获取最终落地 URL 与跳转次数：

```php
// httpUtil.php#L390-L391
$c_effective_url  = curl_getinfo($ch, CURLINFO_EFFECTIVE_URL);   // 最终 URL
$c_redirect_count = curl_getinfo($ch, CURLINFO_REDIRECT_COUNT);  // 实际跳转了几次

// 返回给调用者
return [
    'body' => $body,
    'effective_url' => $c_effective_url,
    'redirect_count' => $c_redirect_count,  // 传递已用次数
    // ...
];
```

##### Layer B：HTML meta refresh 重定向（JS 跳转的替代方案）

很多站点会在 HTML 中嵌入 `<meta http-equiv="refresh">` 进行二次跳转，cURL 无法处理。
FreshRSS 在拿到 HTML 后，通过 DOM 扫描手动跟随。见 [Entry.php#L958-L973](app/Models/Entry.php#L958-L973)：

```php
// 先用 HTTP 重定向次数抵扣总预算
$url = $response['effective_url'] ?: $url;
$maxRedirs -= $response['redirect_count'];   // 总预算 4 - 已用 HTTP 重定向次数

if ($maxRedirs > 0) {
    // 扫描所有 <meta content="...">
    $metas = $xpath->query('//meta[@content]') ?: [];
    foreach ($metas as $meta) {
        if ($meta instanceof DOMElement
            && strtolower(trim($meta->getAttribute('http-equiv'))) === 'refresh') {

            // 解析 content="0; url=http://..."
            // 正则去掉前面的秒数、分号、空格，以及可选的 "url=" 前缀
            $refresh = preg_replace('/^[0-9.; ]*\s*(url\s*=)?\s*/i', '',
                trim($meta->getAttribute('content')));

            // 相对 URL 转绝对
            $refresh = is_string($refresh)
                ? \SimplePie\Misc::absolutize_url($refresh, $url)
                : false;

            if ($refresh != false && $refresh !== $url) {
                // 递归调用自身，重定向预算 -1（meta 自身也算一次跳转）
                return $this->getContentByParsing($refresh, $maxRedirs - 1);
            }
        }
    }
}
```

**重定向处理的完整预算分配**：

> ⚠️ **重大校正**：`$maxRedirs` 与 `CURLOPT_MAXREDIRS` 是**两套独立的限制**，之前的"总跳转次数 ≤ 4"描述有误。

| 限制机制 | 作用范围 | 默认值 | 代码位置 | 可否覆盖 |
|----------|----------|--------|----------|----------|
| `CURLOPT_MAXREDIRS` | **单次** `httpGet()` 的 HTTP 301/302 跳转 | 4 | [httpUtil.php#L346](app/Utils/httpUtil.php#L346) | ✅ `curl_options` / `curl_params` 覆盖 |
| `$maxRedirs` 参数 | 跨递归层的**总预算**，同时抵扣 HTTP + 控制 meta 递归 | 4 | [Entry.php#L924](app/Models/Entry.php#L924) | ❌ 调用方固定传 4 |

**关键洞察 — 两套限制的独立性**：

```php
// httpUtil.php#L346 - 每次调用 httpGet() 都重新设置
CURLOPT_MAXREDIRS => 4,   // 硬编码，但可被 curl_options 覆盖

// Entry.php#L960-L961 - $maxRedirs 是跨递归层的总预算
$maxRedirs -= $response['redirect_count'];  // HTTP 抵扣
if ($maxRedirs > 0) {                        // 仅控制 meta 递归条件
    return $this->getContentByParsing($refresh, $maxRedirs - 1);  // meta 递归 -1
}
```

递归调用 `getContentByParsing($refresh, $maxRedirs - 1)` 时，内部**再次调用** `httpGet()`，`CURLOPT_MAXREDIRS` **重新设为 4**。因此每层递归都有独立的 4 次 HTTP 重定向预算，`$maxRedirs` 只控制 meta refresh 递归的条件（`> 0`）。

**实际最大总跳转次数可达 8 次**（非 4 次）：

```
最极端场景：
  第1层: 0次HTTP重定向 → $maxRedirs = 4-0 = 4 > 0 → meta递归(传入3)
  第2层: 0次HTTP重定向 → $maxRedirs = 3-0 = 3 > 0 → meta递归(传入2)
  第3层: 0次HTTP重定向 → $maxRedirs = 2-0 = 2 > 0 → meta递归(传入1)
  第4层: 0次HTTP重定向 → $maxRedirs = 1-0 = 1 > 0 → meta递归(传入0)
  第5层: 4次HTTP重定向 → $maxRedirs = 0-4 = -4 ≤ 0 → 不再递归
  ─────────────────────────────────────────────────────────
  总跳转 = 4(meta) + 4(HTTP) = 8次
```

> 注：第 5 层的 4 次 HTTP 重定向不受 `$maxRedirs` 限制，只受 `CURLOPT_MAXREDIRS = 4` 限制。`$maxRedirs` 抵扣后变为负数，仅阻止后续 meta 递归，不阻止已发生的 HTTP 重定向。

**缓存命中时的特殊处理**：

[httpUtil.php#L281](app/Utils/httpUtil.php#L281) 命中文件缓存时返回 `redirect_count = 0`：
```php
return ['body' => $body, 'effective_url' => $url, 'redirect_count' => 0, ...];
```
此时 `$maxRedirs` 不被抵扣（`$maxRedirs -= 0`），meta refresh 有完整的初始预算。

##### 重定向后 base href 的计算

无论何种重定向，最终需要确定 HTML 内相对 URL 的基准。见 [Entry.php#L975-L981](app/Models/Entry.php#L975-L981)：

```php
// 优先使用 HTML 中 <base href="...">（如果存在）
$base = $xpath->evaluate('normalize-space(//base/@href)');
if ($base == false || !is_string($base)) {
    $base = $url;   // 回退到最终落地 URL（含 HTTP + meta 重定向之后）
} elseif (str_starts_with($base, '//')) {
    // 协议相对 URL "//www.example.net" 补全 scheme
    $base = (parse_url($url, PHP_URL_SCHEME) ?? 'https') . ':' . $base;
}
```

#### Step 4：CSS 选择器提取正文

```php
$doc = new DOMDocument();
$doc->loadHTML($html, LIBXML_NONET | LIBXML_NOERROR | LIBXML_NOWARNING);
$xpath = new DOMXPath($doc);

// CSS 选择器转 XPath（使用 phpgt/cssxpath 库）
$cssSelector = htmlspecialchars_decode($feed->pathEntries(), ENT_QUOTES);
$translator = new Gt\CssXPath\Translator($cssSelector, '//');
$nodes = $xpath->query($translator->asXPath());

// 【第一次过滤】path_entries_filter：提取前移除不需要的节点
$filter_xpath = (new Gt\CssXPath\Translator($path_entries_filter, 'descendant-or-self::'))->asXPath();
foreach ($filterednodes as $filterednode) {
    $filterednode->remove();  // 删除广告、导航等
}

// 提取匹配的节点 HTML
foreach ($nodes as $node) {
    $html .= $doc->saveHTML($node) . "\n";
}
```

#### Step 5：SimplePie 内容清洗

**文件**: [SimplePieCustom.php#L285-L310](app/Models/SimplePieCustom.php#L285-L310)

```php
$html = FreshRSS_SimplePieCustom::sanitizeHTML($html, $base);
```

清洗规则定义在 `SimplePieCustom` 构造函数中，包括：

| 清洗类别 | 配置位置 | 说明 |
|----------|----------|------|
| 允许的 HTML 属性 | [L68-L80](app/Models/SimplePieCustom.php#L68-L80) | `dir`, `lang`, `title`, `role` 等 |
| 允许的 HTML 元素及属性 | [L81-L226](app/Models/SimplePieCustom.php#L81-L226) | 大段白名单：`a`, `img`, `video`, `table`, MathML 等 |
| 强制剥离的属性 | [L227-L232](app/Models/SimplePieCustom.php#L227-L232) | `data-original` 等不安全属性 |
| 强制添加的属性 | [L233-L241](app/Models/SimplePieCustom.php#L233-L241) | `<audio controls>`, `<iframe sandbox>` 等安全加固 |
| URL 重写规则 | [L242-L267](app/Models/SimplePieCustom.php#L242-L267) | 相对 URL 转绝对 URL |
| HTTPS 强制域名 | [L268-L282](app/Models/SimplePieCustom.php#L268-L282) | `force-https.txt` 列表中的域名强制 https |
| 禁止的 URI Scheme | [L67](app/Models/SimplePieCustom.php#L67) | `javascript:` 被禁用 |

#### Step 6：二次过滤

```php
// 【第二次过滤】sanitize 后再次过滤（因为清洗可能产生新的可匹配节点）
$filterednodes = $xpath->query((new Gt\CssXPath\Translator($path_entries_filter, '//'))->asXPath());
foreach ($filterednodes as $filterednode) {
    $filterednode->remove();
}
```

---

## 四、缓存机制详解

### 4.1 多层缓存架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         缓存层次                                 │
├─────────────────────────────────────────────────────────────────┤
│  Layer 1: SimplePie Feed 缓存（RSS/Atom XML）                    │
│  路径：CACHE_PATH/{sha1(url)}.spc                                │
│  有效期：limits.cache_duration ~ cache_duration_max              │
│  触发：$simplePie->init() 内部                                   │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2: 全文抓取 HTTP 缓存（HTML页面）                         │
│  路径：CACHE_PATH/{sha1(url#selector)}.html                      │
│  有效期：limits.cache_duration（默认小时级）                     │
│  触发：FreshRSS_http_Util::httpGet()                             │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3: Retry-After 限流缓存                                   │
│  路径：DATA_PATH/Retry-After/{domain}.txt                        │
│  触发：HTTP 429/503 响应，存储 Retry-After 值                    │
├─────────────────────────────────────────────────────────────────┤
│  Layer 4: 数据库内容缓存（最终持久化）                            │
│  表：_entry.content + _entry.attributes['original_content']      │
│  避免重复抓取：searchByGuid() 检查已有文章                        │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 HTTP 缓存逻辑

**文件**: [httpUtil.php#L271-L445](app/Utils/httpUtil.php#L271-L445)

```php
public static function httpGet(string $url, ?string $cachePath = null, ...): array {
    // ① 检查本地文件缓存是否有效
    if ($cachePath !== null) {
        $cacheMtime = @filemtime($cachePath);
        if ($cacheMtime !== false && 
            $cacheMtime > time() - intval($limits['cache_duration'])) {
            $body = @file_get_contents($cachePath);
            if ($body != false) {
                return ['status' => -200, /* 表示命中缓存 */ ...];
            }
        }
    }
    
    // ② 检查 Retry-After 限流（防止被目标站封禁）
    if (($retryAfter = self::getRetryAfter($url, $proxy)) > 0) {
        return ['status' => -429, 'fail' => true];
    }
    
    // ③ 实际发起 cURL 请求
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => $url,
        CURLOPT_CONNECTTIMEOUT => $limits['timeout'],
        CURLOPT_MAXREDIRS => 4,
        CURLOPT_FOLLOWLOCATION => true,
        CURLOPT_ACCEPT_ENCODING => '',  // 启用 gzip
    ]);
    $body = curl_exec($ch);
    
    // ④ 编码与 base URL 处理
    $body = self::enforceHttpEncoding($body, $c_content_type);  // 处理字符集
    $body = self::enforceHtmlBase($body, $c_effective_url);     // 注入 <base href>
    
    // ⑤ 写入缓存文件
    if ($cachePath !== null) {
        file_put_contents($cachePath, $body);
    }
    
    // ⑥ 处理 429/503，写入 Retry-After
    if (in_array($c_status, [429, 503], true)) {
        self::setRetryAfter($url, $proxy, $headers['retry-after'] ?? '');
    }
}
```

### 4.3 缓存文件命名规则

**文件**: [Feed.php#L1295-L1317](app/Models/Feed.php#L1295-L1317)

```php
public function cacheFilename(string $url = ''): string {
    $simplePie = new FreshRSS_SimplePieCustom($this->attributes(), $this->curlOptions());
    $filename = $simplePie->get_cache_filename($url);  // sha1
    
    // 按类型加扩展名
    switch ($this->kind) {
        case KIND_RSS:         return CACHE_PATH . '/' . $filename . '.spc';
        case KIND_HTML_XPATH:  return CACHE_PATH . '/' . $filename . '.html';
        case KIND_JSON_DOTNOTATION: return CACHE_PATH . '/' . $filename . '.json';
        // ...
    }
}
```

**全文抓取专用缓存**（`getContentByParsing` 中）：
```php
$cachePath = $feed->cacheFilename($url . '#' . $feed->pathEntries());
// 在 URL 后拼接 CSS 选择器，保证不同 selector 对应不同缓存
```

---

## 五、内容合并与存储

### 5.1 三种合并策略

由 Feed 的 `content_action` 属性控制：

| 策略 | 属性值 | 效果 |
|------|--------|------|
| **替换**（默认） | `replace` | RSS 摘要 → attributes.original_content，content = 全文 |
| **前置** | `prepend` | content = 全文 + 摘要 |
| **追加** | `append` | content = 摘要 + 全文 |

代码实现参考 [Entry.php#L1083-L1097](app/Models/Entry.php#L1083-L1097)。

### 5.2 `original_content` 属性的完整生命周期

`original_content` 是存储 RSS 原始摘要的关键属性，贯穿全文抓取的写入、读取、序列化、恢复等完整生命周期。

#### 写入时机（5 处调用 `_attribute('original_content')`）

所有写入操作位于 [Entry.php#L1085-L1133](app/Models/Entry.php#L1085-L1133)：

| 场景 | 代码 | 说明 |
|------|------|------|
| `content_action = 'prepend'` | `$this->_attribute('original_content');` | **清除**该属性（仅保留 content 拼接结果） |
| `content_action = 'append'` | `$this->_attribute('original_content');` | **清除**该属性（仅保留 content 拼接结果） |
| `content_action = 'replace'` | `$this->_attribute('original_content', $originalContent);` | **保存** RSS 原始摘要到 attributes |
| `path_entries_filter` 过滤时 | `$this->_attribute('original_content');` | 先清除旧值 |
| 过滤后内容变化时 | `$this->_attribute('original_content', $originalContent);` | **保存**过滤前的内容 |

```php
// 核心代码片段
switch ($feed->attributeString('content_action')) {
    case 'prepend':
    case 'append':
        $this->_attribute('original_content');  // 无参数 = 清除
        $this->content = $fullContent . $originalContent;
        break;
    case 'replace':
    default:
        $this->_attribute('original_content', $originalContent);  // 保存
        $this->content = $fullContent;
        break;
}
```

#### 读取与恢复

`originalContent()` 方法 [Entry.php#L210-L213](app/Models/Entry.php#L210-L213) 实现**双重保险**的恢复机制：

```php
public function originalContent(): string {
    return $this->attributeString('original_content') ??      // ① 优先从 attributes 读取
        preg_replace('#<!-- FULLCONTENT start //-->.*<!-- FULLCONTENT end //-->#s', 
            '', $this->content) ?? '';                        // ② 失败则正则移除 FULLCONTENT 注释块
}
```

> 设计意图：即使 `original_content` 属性意外丢失（如数据库迁移），仍可通过正则解析 `content` 字段恢复原始摘要，保证数据可逆。

#### 哈希计算中的角色

`hash()` 方法 [Entry.php#L513-L520](app/Models/Entry.php#L513-L520) 将 `originalContent()` 纳入 MD5 计算：

```php
$this->hash = md5(
    $this->link . $this->title . $this->authors(true) . 
    $this->originalContent() . $this->tags(true) . $attributes
);
```

这意味着：
- RSS 原始摘要变化 → hash 变化 → 触发 `loadCompleteContent(true)` 强制重抓全文
- 即使全文抓取失败或被禁用，RSS 摘要的更新依然能被正确检测

#### 数据库序列化

`toArray()` 方法 [Entry.php#L1152-L1171](app/Models/Entry.php#L1152-L1171) 将整个 `attributes` 数组（含 `original_content`）序列化到 DB：

```php
public function toArray(): array {
    return [
        // ... 其他字段
        'content' => $this->content(false),  // 不含 enclosures 的合并后内容
        'attributes' => $this->attributes(), // 含 original_content 的 JSON
    ];
}
```

最终存储于 `_entry.attributes` 字段（JSON 格式），`original_content` 作为其中的一个键。

#### 特殊分支：取消全文抓取后的恢复

当用户之前为 Feed 配置了全文抓取，之后取消（`pathEntries` 变空），代码会自动恢复原始内容 [Entry.php#L1137-L1141](app/Models/Entry.php#L1137-L1141)：

```php
} else {
    // pathEntries 和 path_entries_filter 都为空
    $originalContent = $this->originalContent();
    if ($originalContent !== $this->content) {
        $this->content = $originalContent;  // 恢复为 RSS 原始摘要
        return true;
    }
}
```

#### FULLCONTENT 注释标记

全文内容在存入 `content` 前会用 HTML 注释包裹，便于正则恢复：

```php
$fullContent = "<!-- FULLCONTENT start //-->{$fullContent}<!-- FULLCONTENT end //-->";
```

三种合并策略下 `content` 字段的最终内容：

| 策略 | content 字段内容 |
|------|-----------------|
| `replace`（默认） | `<!-- FULLCONTENT start //-->全文<!-- FULLCONTENT end //-->` |
| `prepend` | `<!-- FULLCONTENT start //-->全文<!-- FULLCONTENT end //-->RSS摘要` |
| `append` | `RSS摘要<!-- FULLCONTENT start //-->全文<!-- FULLCONTENT end //-->` |

### 5.3 入数据库

**文件**: [EntryDAO.php](app/Models/EntryDAO.php)

```php
// 新文章
$entryDAO->addEntry($entry->toArray(), true);  // 写入 _entrytmp 临时表

// 已有文章更新
$entryDAO->updateEntry($entry->toArray());

// commit 阶段：从 _entrytmp 移到 _entry 主表
$entryDAO->commitNewEntries();
```

### 5.4 哈希与去重

**文件**: [Entry.php#L513-L520](app/Models/Entry.php#L513-L520)

```php
public function hash(): string {
    if ($this->hash === '') {
        // 哈希计算不包含 date，但包含 originalContent 和 enclosures
        $this->hash = md5(
            $this->link . $this->title . $this->authors(true) . 
            $this->originalContent() . $this->tags(true) . $attributes
        );
    }
    return $this->hash;
}
```

**比较逻辑**：
```php
// actualizeFeeds 中判断文章是否有更新
$existingHashForGuids = $entryDAO->listHashForFeedGuids($feed->id(), $newGuids);
if (isset($existingHashForGuids[$entry->guid()])) {
    $existingHash = $existingHashForGuids[$entry->guid()];
    if (strcasecmp($existingHash, $entry->hash()) !== 0) {
        // 哈希不同 → 文章有更新 → 重新抓取全文
        $entry->loadCompleteContent(true);  // $force=true 跳过缓存
        $entryDAO->updateEntry($entry->toArray());
    }
}
```

---

## 六、展示层合并与附件去重

### 6.1 视图渲染入口

**文件**: [normal.phtml#L130-L132](app/views/index/normal.phtml#L130-L132)

```php
<div class="text"><?=
    FreshRSS_Context::userConf()->lazyload && !FreshRSS_Context::userConf()->display_posts 
        ? lazyimg($this->entry->content(true))   // 懒加载：图片延迟加载
        : $this->entry->content(true)            // 正常渲染
?></div>
```

Helper 模板中也使用同样方式：[article.phtml#L102](app/views/helpers/index/article.phtml#L102)

### 6.2 `content()` 方法 — 最终合并 + 附件去重

**文件**: [Entry.php#L192-L313](app/Models/Entry.php#L192-L313)

这是展示层的核心函数，除了合并正文外，还包含了完整的附件去重逻辑。

#### 附件去重判定方法

**方法 1：`containsLink()` — 检查 URL 是否已存在于 HTML 中**

[Entry.php#L192-L194](app/Models/Entry.php#L192-L194)

```php
private static function containsLink(string $html, string $link): bool {
    // 用正则匹配被引号包裹的 URL（不覆盖所有边界情况，简单字符串比较）
    return preg_match('/(?P<delim>[\'"])' . preg_quote($link, '/') . '(?P=delim)/', $html) == 1;
}
```

> 注：该方法仅做简单的引号内字符串匹配，不处理 unquoted 属性值、HTML 注释变体等场景，适合做"明显重复"的快速过滤。

**方法 2：`enclosureIsImage()` — 判断 enclosure 是否为图片类型**

[Entry.php#L197-L205](app/Models/Entry.php#L197-L205)

```php
private static function enclosureIsImage(array $enclosure): bool {
    $elink  = $enclosure['url']    ?? '';
    $length = $enclosure['length'] ?? 0;
    $medium = $enclosure['medium'] ?? '';
    $mime   = $enclosure['type']   ?? '';

    return ($elink != '' && $medium === 'image')           // ① media:content medium="image"
        || str_starts_with($mime, 'image')                  // ② MIME 以 image/ 开头
        || ($mime == '' && $length == 0                     // ③ 兜底：无 MIME 无大小
            && preg_match('/[.](avif|gif|jpe?g|png|svg|webp)([?#]|$)/i', $elink));  // 根据扩展名猜
}
```

#### 完整渲染流程（含去重 + length 输出）

```php
public function content(bool $withEnclosures = true, bool $allowDuplicateEnclosures = false): string {
    if (!$withEnclosures) {
        return $this->content;  // 仅返回已合并的正文（不含任何附件）
    }

    $content = $this->content;

    // ─────────────────────────────────────────────────
    // ① 追加 thumbnail 缩略图 enclosure
    // ─────────────────────────────────────────────────
    $thumbnailAttribute = $this->attributeArray('thumbnail') ?? [];
    if (!empty($thumbnailAttribute['url'])) {
        $elink = $thumbnailAttribute['url'];
        // 去重：只有当正文 HTML 中不包含该 URL 时才追加
        if (is_string($elink) && ($allowDuplicateEnclosures || !self::containsLink($content, $elink))) {
            $content .= <<<HTML
                <figure class="enclosure">
                    <p class="enclosure-content">
                        <img class="enclosure-thumbnail" src="{$elink}" alt="" />
                    </p>
                </figure>
                HTML;
        }
    }

    // ─────────────────────────────────────────────────
    // ② 追加 enclosures（图片 / 音频 / 视频 / 通用附件）
    // ─────────────────────────────────────────────────
    $attributeEnclosures = $this->attributeArray('enclosures');
    if (empty($attributeEnclosures)) {
        return $content;
    }

    foreach ($attributeEnclosures as $enclosure) {
        if (!is_array($enclosure)) continue;
        $elink = $enclosure['url'] ?? '';
        if ($elink == '' || !is_string($elink)) continue;

        // ⭐ 去重检查（与 thumbnail 使用同一判定函数）
        if (!$allowDuplicateEnclosures && self::containsLink($content, $elink)) {
            continue;   // 正文里已有相同 URL（例如 <img src="同一个链接">），不再重复追加
        }

        // 读取 enclosure 附加信息
        $credits    = $enclosure['credit'] ?? '';
        $description = is_string($enclosure['description'] ?? null)
            ? nl2br($enclosure['description'], true)    // 换行符转 <br />
            : '';
        $length     = is_numeric($enclosure['length'] ?? null) ? (int)$enclosure['length'] : 0;
        $medium     = is_string($enclosure['medium'] ?? null) ? $enclosure['medium'] : '';
        $mime       = is_string($enclosure['type'] ?? null) ? $enclosure['type'] : '';
        $thumbnails = is_array($enclosure['thumbnails'] ?? null) ? $enclosure['thumbnails'] : [];
        $etitle     = is_string($enclosure['title'] ?? null) ? $enclosure['title'] : '';

        $content .= "\n" . '<figure class="enclosure">';

        // 2a. 先渲染 enclosure 的缩略图（media:thumbnail）
        foreach ($thumbnails as $thumbnail) {
            if (is_string($thumbnail)) {
                $content .= '<p><img class="enclosure-thumbnail" src="' . $thumbnail
                    . '" alt="" title="' . $etitle . '" /></p>';
            }
        }

        // 2b. 根据类型渲染主 enclosure（4 类）
        if (self::enclosureIsImage(['url' => $elink, 'length' => $length, 'medium' => $medium, 'type' => $mime])) {
            // 图片：直接 <img> 嵌入（无 data-length）
            $content .= '<p class="enclosure-content"><img src="' . $elink
                . '" alt="" title="' . $etitle . '" /></p>';
        } elseif ($medium === 'audio' || str_starts_with($mime, 'audio')) {
            // 音频：<audio controls> + 下载图标，带 data-length 属性
            $content .= '<p class="enclosure-content"><audio preload="none" src="' . $elink
                . ($length === null ? '' : '" data-length="' . $length)
                . ($mime == '' ? '' : '" data-type="' . htmlspecialchars($mime, ENT_COMPAT, 'UTF-8'))
                . '" controls="controls" title="' . $etitle . '"></audio> <a download="" href="' . $elink . '">💾</a></p>';
        } elseif ($medium === 'video' || str_starts_with($mime, 'video')) {
            // 视频：<video controls> + 下载图标，带 data-length 属性
            $content .= '<p class="enclosure-content"><video preload="none" src="' . $elink
                . ($length === null ? '' : '" data-length="' . $length)
                . ($mime == '' ? '' : '" data-type="' . htmlspecialchars($mime, ENT_COMPAT, 'UTF-8'))
                . '" controls="controls" title="' . $etitle . '"></video> <a download="" href="' . $elink . '">💾</a></p>';
        } else {
            // 其它：application/pdf、text/plain 等通用附件，仅显示下载图标
            $content .= '<p class="enclosure-content"><a download="" href="' . $elink
                . ($mime == '' ? '' : '" data-type="' . htmlspecialchars($mime, ENT_COMPAT, 'UTF-8'))
                . ($medium == '' ? '' : '" data-medium="' . htmlspecialchars($medium, ENT_COMPAT, 'UTF-8'))
                . '" title="' . $etitle . '">💾</a></p>';
        }

        // 2c. 渲染版权信息（© credit）
        if ($credits != '') {
            $credits = is_array($credits) ? $credits : [$credits];
            foreach ($credits as $credit) {
                if (is_string($credit)) {
                    $content .= '<p class="enclosure-credits">© ' . $credit . '</p>';
                }
            }
        }
        // 2d. 渲染 enclosure description（使用 HTML5 <figcaption>）
        if ($description != '') {
            $content .= '<figcaption class="enclosure-description">' . $description . '</figcaption>';
        }

        $content .= '</figure>';
    }

    return $content;
}
```

**展示层的最终内容** = `[RSS摘要 + 全文]（按策略合并） + [缩略图 enclosure（去重）] + [附件 enclosures（去重 + 按类型渲染 + length/type/description/credit）]`

#### 展示层 `length` 信息输出（`data-length` 属性）

`enclosure['length']` 来自 RSS `<media:content length="...">`，在 FreshRSS 中有完整的读写闭环。

##### 数据流向

```
SimplePie 解析 <media:content length="1234567">
    ↓ 存入
_entry.attributes.enclosures[n].length = 1234567
    ↓ 读取（Entry::content()）
$length = is_numeric(...) ? (int)$enclosure['length'] : 0  →  1234567（或 0）
    ↓ 输出到 HTML（仅 audio/video）
<audio data-length="1234567" ...> 或 <video data-length="0" ...>
    ↓ 反向解析（Entry::enclosures()）
$enclosure['length'] = (int)$element->getAttribute('data-length')
```

##### ⚠️ 输出条件校正：`data-length` 始终输出（`=== null` 为死代码）

**关键发现**：`$length` 在赋值时已被强制转换为 `int`，永远不会是 `null`，因此 `=== null` 判断是**不可达的死代码**，`data-length` 对 audio/video **始终输出**（即使 length=0）。

```php
// Entry.php#L259 - $length 被强制转为 int
$length = is_numeric($enclosure['length'] ?? null) ? (int)$enclosure['length'] : 0;
//         ↑ 非数字时为 0，数字时为 (int)，结果永远是 int，永远不是 null

// Entry.php#L914 - 音频渲染（$length === null 永远为 false，data-length 始终输出）
$content .= '...<audio preload="none" src="' . $elink
    . ($length === null ? '' : '" data-length="' . $length)   // ← 死代码分支
    . ...

// Entry.php#L920 - 视频渲染（同上）
$content .= '...<video preload="none" src="' . $elink
    . ($length === null ? '' : '" data-length="' . $length)   // ← 死代码分支
    . ...
```

##### `is_numeric()` 宽松性对 `$length` 取值的影响

`is_numeric()` [Entry.php#L259](app/Models/Entry.php#L259) 是**宽松判断**，接受多种格式的数字字符串：

| 输入值 | `is_numeric()` | `(int)` 转换 | `$length` 最终值 |
|--------|---------------|-------------|-----------------|
| `1234567` (int) | ✅ true | 1234567 | `1234567` |
| `"1234567"` (string) | ✅ true | 1234567 | `1234567` |
| `"1e5"` (科学计数法) | ✅ true | 1 | `1`（⚠️ `(int)"1e5" = 1`，非 100000） |
| `"0"` | ✅ true | 0 | `0` |
| `null` | ❌ false | — | `0` |
| `""` | ❌ false | — | `0` |
| `"unknown"` | ❌ false | — | `0` |

> ⚠️ 注意 `"1e5"` 这类科学计数法字符串：`is_numeric()` 返回 true，但 `(int)"1e5"` 在 PHP 中结果为 `1`（而非 100000），导致 `data-length` 输出错误值。

##### `==` 宽松比较 vs `===` 严格比较在 length 处理中的差异

FreshRSS 在不同位置对 `$length` 使用了不同的比较运算符，行为有细微差异：

| 位置 | 比较方式 | 代码 | 影响 |
|------|----------|------|------|
| `content()` data-length 输出 | `=== null`（严格） | [Entry.php#L914](app/Models/Entry.php#L914) | `$length` 已是 int，永远不为 null → **始终输出** data-length |
| `enclosureIsImage()` 图片判定 | `== 0`（宽松） | [Entry.php#L204](app/Models/Entry.php#L204) | `$length` 为 int 时 `== 0` 与 `=== 0` 效果相同 |
| `containsLink()` 去重 | `== 1`（宽松） | [Entry.php#L193](app/Models/Entry.php#L193) | `preg_match` 返回 int 0/1/false，`== 1` 兼容 false |

```php
// enclosureIsImage() Entry.php#L199 - 注意：这里没有 (int) 转换！
$length = $enclosure['length'] ?? 0;   // 原始值，可能是 string/int/null
// ...
($mime == '' && $length == 0           // 宽松比较 ==
    && preg_match('/[.](avif|gif|jpe?g|png|svg|webp)([?#]|$)/i', $elink))

// 但 content() Entry.php#L277 传入 enclosureIsImage() 时用的是已转换的 $length
if (self::enclosureIsImage(['url' => $elink, 'length' => $length, ...]))
//                                              ↑ 这是 content() 中 (int) 转换后的 $length
```

> **关键**：`enclosureIsImage()` 的函数签名 [Entry.php#L196](app/Models/Entry.php#L196) 标注 `length?:int`，但内部 `$enclosure['length'] ?? 0` **不做类型转换**。实际调用时 `content()` 传入的是已 `(int)` 转换的值 [Entry.php#L277](app/Models/Entry.php#L277)，所以 `$length == 0` 在此场景等价于 `=== 0`。但如果直接调用 `enclosureIsImage()` 传入原始字符串值（如 `"0"`），宽松比较 `"0" == 0` 仍为 true。

**校正后的输出规则**：

| 附件类型 | data-length 输出？ | length=0 时 | 代码位置 |
|----------|-------------------|-------------|----------|
| 图片（image） | ❌ 无 | 不适用 | [Entry.php#L907-L910](app/Models/Entry.php#L907-L910) |
| 音频（audio） | ✅ **始终输出** | 输出 `data-length="0"` | [Entry.php#L912-L916](app/Models/Entry.php#L912-L916) |
| 视频（video） | ✅ **始终输出** | 输出 `data-length="0"` | [Entry.php#L918-L922](app/Models/Entry.php#L918-L922) |
| 通用附件（其它） | ❌ 无 | 不适用 | [Entry.php#L924-L928](app/Models/Entry.php#L924-L928) |

> **对比通用附件**：通用附件（else 分支）不输出 `data-length`，但输出 `data-type` 和 `data-medium`，条件同样使用 `$mime == ''` 和 `$medium == ''` 判断（空字符串检查，非 null 检查）。

> **用途**：`data-length` 是 HTML5 自定义数据属性，供前端 JavaScript 读取（如显示文件大小、计算播放进度条百分比等）。FreshRSS 自身不使用该值，仅作为元数据透传给前端。

##### 反向解析（从 HTML 提取 enclosure）

`Entry::enclosures()` 方法 [Entry.php#L340-L344](app/Models/Entry.php#L340-L344) 可以从已渲染的 HTML 中反向解析 enclosure，包括从 `data-length` 恢复 length：

```php
$result = [
    'url'    => $enclosure->getAttribute('src'),
    'type'   => $enclosure->getAttribute('data-type'),
    'medium' => $enclosure->getAttribute('data-medium'),
    'length' => (int)($enclosure->getAttribute('data-length')),  // ← 反向提取
];
```

##### `enclosureIsImage()` 中的 length 角色

`length` 还参与图片类型判定（兜底策略）：
```php
// Entry.php#L203-L204
($mime == '' && $length == 0           // 无 MIME + 无大小
    && preg_match('/[.](avif|gif|jpe?g|png|svg|webp)([?#]|$)/i', $elink))
```
> 当 enclosure 既没有 MIME 类型也没有文件大小时，才尝试通过 URL 扩展名猜测是否为图片。

### 6.3 RSS 输出模板中的附件去重

FreshRSS 自身也能以 RSS 格式输出内容（供其它 RSS 阅读器消费），在 [rss.phtml#L73-L78](app/views/index/rss.phtml#L73-L78) 中实现了更轻量的 URL 级去重：

```php
$urls = [];
foreach ($enclosures as $enclosure) {
    if (empty($enclosure['url']) || isset($urls[$enclosure['url']])) {
        continue;   // 同一 URL 的 enclosure 只输出一个 <media:content>
    }
    $urls[$enclosure['url']] = true;
    // ... 输出 <media:content ...>
}
```

> 注意这里的差别：RSS 输出模板用的是**哈希表去重**（精确 URL 相等），而 HTML 展示层 `content()` 用的是**正则扫描正文**（URL 只要出现在正文 HTML 中就算重复）。

---

## 七、描述（description）渲染全链路

FreshRSS 中有多种 "description" 概念，分别来自 Feed、Enclosure、页面 Context，用途完全不同。本节按数据流向梳理。

### 7.1 Feed description（订阅源描述）

#### 数据来源（3 条写入路径）

**路径 A：RSS/Atom 源解析时自动获取**

[Feed.php#L671-L672](app/Models/Feed.php#L671-L672)（仅当用户没有手动填写时覆盖）：
```php
if ($this->description() === '') {
    $this->_description(html_only_entity_decode($simplePie->get_description()));
}
```

同样逻辑在实际化时也会执行：[feedController.php#L811-L815](app/Controllers/feedController.php#L811-L815)
```php
if (trim($feed->description()) === '') {
    $description = html_only_entity_decode($simplePie->get_description());
    // ... 更新到 feedProperties
}
```

**路径 B：从 OPML 文件导入**

[ImportService.php#L148](app/Services/ImportService.php#L148)：
```php
$description = Minz_Helper::htmlspecialchars_utf8($feed_elt['description'] ?? '');
```

**路径 C：用户在 Web UI 中手动编辑**

编辑表单位于 [update.phtml#L103](app/views/helpers/feed/update.phtml#L103)，存储于 `Feed.description` 字段。

#### 存储与读取

存储在 `Feed` 模型的 `$description` 私有属性：[Feed.php#L60](app/Models/Feed.php#L60)，通过 `description()` / `_description()` 读写：[Feed.php#L311-L312, L548-L549](app/Models/Feed.php#L311-L312)

#### 渲染位置（3 处使用场景）

| 场景 | 文件 | 渲染方式 |
|------|------|----------|
| 添加 Feed 预览页 | [add.phtml#L25-L32](app/views/feed/add.phtml#L25-L32) | `htmlspecialchars($desc, ENT_NOQUOTES, 'UTF-8')` 转义后纯文本展示 |
| 页面 HTML `<meta description>` + RSS `<channel><description>` | [Context.php#L469-L511](app/Models/Context.php#L469-L511) | 单 Feed 视图（`type='f'`）时赋值给 `Context::$description`，供布局模板或 RSS 模板使用 |
| OPML 导出 | [opml.phtml#L20](app/views/helpers/export/opml.phtml#L20) | `htmlspecialchars_decode()` 反转义后写入 `<outline description="...">` |

**RSS 输出中 Feed description 的渲染** [rss.phtml#L11](app/views/index/rss.phtml#L11)：
```xml
<description><?= $this->description ?: _t('index.feed.rss_of', $this->rss_title) ?></description>
```
若 Feed 无 description，回退到翻译字符串 "RSS feed of {Feed 名}"。

非 Feed 视图（全部/重要/星标/分类等）使用系统级 `meta_description` [Context.php#L472-L495](app/Models/Context.php#L472-L495)。

### 7.2 Enclosure description（附件描述）

#### 数据来源

RSS 源 `<media:content>` 下的 `<media:description>` 由 SimplePie 解析后提供 `$enclosure->get_description()`。
在 Feed 解析阶段写入 Entry attributes：[Feed.php#L857, L876-L877](app/Models/Feed.php#L857-L877)

```php
$description = $enclosure->get_description() ?? '';
// ...
if ($description != '') {
    $attributeEnclosure['description'] = $description;
}
```

最终存储于 `_entry.attributes['enclosures'][n]['description']`。

#### 渲染方式

在 Entry::content() 展示层渲染 enclosure 时一并输出 [Entry.php#L258, L306-L308](app/Models/Entry.php#L258-L308)：

```php
// 读取时先做 nl2br（换行转 <br />，XHTML 安全）
$description = is_string($enclosure['description'] ?? null)
    ? nl2br($enclosure['description'], true)
    : '';

// 用 HTML5 <figcaption> 语义化标签包裹
if ($description != '') {
    $content .= '<figcaption class="enclosure-description">' . $description . '</figcaption>';
}
```

### 7.3 API / RSS 输出中的文章内容 description

当 FreshRSS 作为被消费的 RSS 源输出文章时，在 `<item>` 级别使用 `<description>` 标签承载全文内容：[rss.phtml#L102-L104](app/views/index/rss.phtml#L102-L104)

```xml
<description><![CDATA[<?php
echo $item->content(false);   // false = 不追加 enclosure，只输出正文（因为 enclosure 已单独以 <media:content> 输出）
?>]]></description>
```

与 HTML 展示层 `$item->content(true)` 不同，RSS 输出使用 `content(false)` —— 因为 enclosure 已由 `<media:content>` / `<media:thumbnail>` 节点独立承载，避免内容重复。

---

## 八、完整时序图

```
用户/Cron
  │
  ▼
actualize_script.php
  │  设置路由 c=feed, a=actualize
  ▼
feedController::actualizeAction()
  │
  ▼
feedController::actualizeFeeds()
  │
  ├─ 遍历 Feed 列表（按 TTL、锁、PubSub 过滤）
  │
  ├─ 对每个 Feed：
  │    │
  │    ├─ Feed::load() → SimplePie 解析 RSS
  │    │     │
  │    │     ├─ 提取 $simplePie->get_description()（仅当 Feed.description 为空时）
  │    │     └─ 比较 SimplePieHash 判断 Feed 是否变化
  │    │
  │    ├─ Feed::loadEntries() → 遍历 SimplePie items
  │    │     │
  │    │     └─ 对每个 item：
  │    │          │
  │    │          ├─ 创建 FreshRSS_Entry（含 RSS 摘要 content）
  │    │          │
  │    │          ├─ 提取 enclosures → 存入 attributes.enclosures[].{url,type,medium,length,description,credit,thumbnails}
  │    │          │
  │    │          ├─ Entry::hash() → 计算内容哈希（含 originalContent + attributes）
  │    │          │
  │    │          └─ Entry::loadCompleteContent() ⭐
  │    │               │
  │    │               ├─ 检查 pathEntries 是否为空？
  │    │               │    ├─ 空 → 跳过，保留原内容
  │    │               │    └─ 非空 → 继续
  │    │               │
  │    │               ├─ DB 中是否已有该 GUID？
  │    │               │    ├─ 有 → 直接复用 DB content
  │    │               │    └─ 无 → 继续
  │    │               │
  │    │               └─ Entry::getContentByParsing()
  │    │                    │
  │    │                    ├─ 检查 path_entries_conditions？
  │    │                    │    ├─ 不匹配 → 返回空
  │    │                    │    └─ 匹配 → 继续
  │    │                    │
  │    │                    ├─ FreshRSS_http_Util::httpGet(文章URL)
  │    │                    │    │
  │    │                    │    ├─ 检查文件缓存（CACHE_PATH/*.html）
  │    │                    │    │    ├─ 命中 → 返回缓存 status=-200
  │    │                    │    │    └─ 未命中 → cURL 请求
  │    │                    │    │         │
  │    │                    │    │         ├─ cURL 自动跟随 HTTP 301/302（CURLOPT_MAXREDIRS=4，每次 httpGet 独立）
  │    │                    │    │         │
  │    │                    │    │         ├─ HTTP 200？
  │    │                    │    │         │    ├─ 否 → 检查 429/503 → 写 Retry-After
  │    │                    │    │         │    └─ 是 → 编码处理、base href 注入
  │    │                    │    │         │
  │    │                    │    │         └─ 写缓存文件
  │    │                    │
  │    │                    ├─ DOMDocument::loadHTML
  │    │                    │
  │    │                    ├─ 处理 HTML <meta http-equiv="refresh"> 重定向（递归，总预算 $maxRedirs=4，抵扣 HTTP 已用）
  │    │                    │    ├─ 提取 content 属性中的 URL
  │    │                    │    ├─ 相对 URL 转绝对
  │    │                    │    └─ URL 变化则递归调用 getContentByParsing(newUrl, maxRedirs-1)
  │    │                    │    └─ 递归后 httpGet 重置 CURLOPT_MAXREDIRS=4（最大总跳转可达 8 次）
  │    │                    │
  │    │                    ├─ 计算 base href（<base> 标签优先，否则用最终落地 URL）
  │    │                    │
  │    │                    ├─ pathEntries CSS 选择器 → XPath 查询 → 提取节点
  │    │                    │
  │    │                    ├─ 【第一次过滤】path_entries_filter（提取前）
  │    │                    │
  │    │                    ├─ SimplePieCustom::sanitizeHTML() ⭐⭐
  │    │                    │    │
  │    │                    │    ├─ 元素白名单（a/img/video/table...）
  │    │                    │    ├─ 属性白名单（href/src/alt...）
  │    │                    │    ├─ 剥离危险属性（onclick/data-original...）
  │    │                    │    ├─ 相对 URL 转绝对（a/img/iframe...）
  │    │                    │    └─ HTTPS 域名强制重写
  │    │                    │
  │    │                    └─ 【第二次过滤】path_entries_filter（清洗后）
  │    │
  │    │               ├─ 包裹 FULLCONTENT 注释标记
  │    │               │
  │    │               └─ 按 content_action 策略合并
  │    │                    ├─ replace: RSS→original_content, 全文→content
  │    │                    ├─ prepend: content = 全文 + RSS
  │    │                    └─ append:  content = RSS + 全文
  │    │
  │    ├─ 文章去重（对比 DB 中 GUID 的 hash）
  │    │    ├─ 新文章 → addEntry()
  │    │    └─ 已更新 → updateEntry() + loadCompleteContent(true) 重抓
  │    │
  │    └─ commitNewEntries() → _entrytmp → _entry
  │
  ▼
用户访问 normal.phtml
  │
  └─ $entry->content(true)
       │
       ├─ 读取 DB 中已合并的 content（含 original_content 属性）
       │
       ├─ 【thumbnail 去重】containsLink(content, thumb_url)? 是→跳过
       │
       └─ 遍历 enclosures
            ├─ 【enclosure URL 去重】containsLink(content, enclosure_url)? 是→跳过
            ├─ 读取 enclosure.description → nl2br
            ├─ 读取 enclosure.credit → © 前缀
            ├─ 读取 enclosure.length → 仅 audio/video 输出为 data-length
            ├─ enclosureIsImage()? → <img>
            ├─ 是 audio? → <audio controls data-length="N"> + 💾
            ├─ 是 video? → <video controls data-length="N"> + 💾
            └─ 其它? → 💾 下载图标
            └─ (credit + description) → <figure> 内 <p> + <figcaption>
```

---

## 九、关键配置项与可调参数

| 配置项 | 所在文件 | 默认值 | 影响 |
|--------|----------|--------|------|
| `limits.timeout` | [config.default.php](config.default.php) | 15s | HTTP 请求超时（Feed抓取 + 全文抓取） |
| `limits.cache_duration` | config.default.php | 3600s | SimplePie + HTTP 缓存默认有效期 |
| `limits.cache_duration_min/max` | config.default.php | 1800~86400s | HTTP 缓存上下限 |
| `limits.retry_after_max` | config.default.php | - | HTTP 429/503 最大等待 |
| `ttl_default` | 用户配置 | - | Feed 默认刷新周期 |
| `pubsubhubbub_enabled` | 系统配置 | true | 是否启用 WebSub 实时推送 |
| `curl_options` | 系统配置 | [] | 全局代理、SSL 配置等 |
| `CURLOPT_MAXREDIRS` | httpUtil.php 硬编码 | 4 | 单次 `httpGet()` 的 HTTP 301/302 最大重定向次数；可被 Feed `curl_params` 覆盖；与 `$maxRedirs` 独立，递归后每次 `httpGet` 重置 |
| `content_action` | Feed attributes | `replace` | 全文合并策略 |
| `pathEntries` | Feed 属性 | `''` | **全文抓取开关 + CSS 选择器** |
| `path_entries_filter` | Feed attributes | `''` | 需要移除的节点 CSS 选择器 |
| `path_entries_conditions` | Feed attributes | `[]` | 触发全文抓取的搜索条件数组 |
| `original_content` | Entry attributes | 动态 | 保存 RSS 原始摘要，支持恢复与 hash 计算 |
| `content_width` | 用户显示配置 | - | 展示层内容宽度样式 |
| `lazyload` | 用户配置 | true | 图片懒加载 |

---

## 十、特殊场景与边界条件

### 场景 1：RSS 内容本身就是全文
- `pathEntries` 留空即可，跳过全文抓取
- 或配合 `path_entries_filter` 仅过滤掉广告节点

### 场景 2：Feed 有更新，单篇文章无变化
- SimplePieHash 变化 → Feed::load 返回非 null
- 但每篇文章的 hash 与 DB 相同 → 不触发重抓

### 场景 3：文章被上游修改
- GUID 相同但 hash 不同 → `$entry->_isUpdated(true)`
- 调用 `loadCompleteContent(true)` 强制重新抓取

### 场景 4：多用户共享同一 Feed
- `cacheModifiedTime()` 检查其他用户是否刚刷新过缓存
- 利用共享缓存减少重复 HTTP 请求

### 场景 5：目标站限流 429
- 读取 Retry-After 响应头
- 写入 `DATA_PATH/Retry-After/{domain}.txt`，mtime=重试时间戳
- 后续请求直接跳过，直到 mtime < 当前时间

### 场景 6：文章页面存在 HTML meta refresh 跳转
- HTTP 301/302 由 cURL 自动跟随（每次 `httpGet()` 最多 4 次，限 http/https）
- `<meta http-equiv="refresh">` 由 Entry.php 递归解析（总预算 `$maxRedirs` 初值 4，每次 meta 递归 -1）
- `CURLOPT_MAXREDIRS` 与 `$maxRedirs` 是**两套独立限制**，递归后每次 `httpGet()` 重置 `CURLOPT_MAXREDIRS = 4`
- 实际最大总跳转可达 **8 次**（4 次 meta refresh + 最后一层 4 次 HTTP 301/302）
- base href 以重定向后最终 URL 为准
- 命中 HTTP 文件缓存时 redirect_count=0，$maxRedirs 不被抵扣

### 场景 7：RSS 正文中已嵌有 enclosure 的图片
- `containsLink()` 正则扫描正文 HTML 中是否已有相同 URL
- 已有则 enclosure 不重复追加，避免图片重复显示
- RSS 输出模板使用哈希表对 enclosure URL 去重

### 场景 8：强制重抓 `$force=true` 但 HTTP 缓存仍生效
- `loadCompleteContent(true)` 仅跳过 Layer 4 DB 内容复用
- Layer 2 HTTP 文件缓存（`.html`）和 Layer 3 Retry-After 仍检查
- ⚠️ `clearCache()` **仅删除 `.spc` 文件**，不删除 `.html` 全文缓存文件
- 要强制重新请求网页正文，只能等待 `limits.cache_duration` 过期（默认 3600s），或删除整个 `CACHE_PATH` 目录

### 场景 9：取消全文抓取配置后自动恢复
- 用户删除 `pathEntries` 配置后，下次刷新触发恢复分支
- 代码自动读取 `originalContent()` 恢复为 RSS 原始摘要
- 双重保险机制：优先读 attributes.original_content，失败则正则移除 FULLCONTENT 注释

### 场景 10：HTTP 重定向次数用尽但仍有 meta refresh
- `$maxRedirs` 被 HTTP `redirect_count` 抵扣后若 ≤ 0，不再做 meta refresh 递归
- 但单次 `httpGet()` 的 HTTP 重定向次数受 `CURLOPT_MAXREDIRS = 4` 独立控制，不受 `$maxRedirs` 限制
- `$maxRedirs` 抵扣为负数后仅阻止后续 meta 递归，不阻止已发生的 HTTP 重定向
- 递归后每层 `httpGet()` 重新获得独立的 4 次 HTTP 重定向预算

### 场景 11：三种"重抓"机制的区别（易混淆）
| 操作 | 作用范围 | 删除的缓存 | 不删除的缓存 |
|------|----------|-----------|-------------|
| `loadCompleteContent(true)` | 单篇文章 | 无（仅跳过 DB 查询） | `.html` / `.spc` / Retry-After |
| `Feed::load(false, true)` | 整个 Feed | 无（仅跳过 hash 比较） | `.html` / `.spc` / Retry-After |
| `$feed->clearCache()` | Feed 级 | Feed 级缓存（按 kind: `.spc`/`.html`/`.json`/`.xml`）+ favicon | 全文 HTML 缓存 / Retry-After |
| `cleanCache(720h)` | 全局所有缓存 | 所有过期文件（30 天+） | 未过期文件 |

> **关键**：`clearCache()` 删除的是 `cacheFilename()` **无参数**时的 Feed 级文件。全文 HTML 缓存使用 `cacheFilename($url . '#' . $pathEntries)` 生成不同文件名，不受影响。

---

## 十一、手动重抓与调试

### 控制器动作

**重新加载（清除 lastUpdate）**：
[feedController.php#L1208-L1264](app/Controllers/feedController.php#L1208-L1264)
```php
public function reloadAction(): void {
    $feedDAO->updateFeed($feed->id(), ['lastUpdate' => 0]);
    self::actualizeFeedsAndCommit($feed_id);
    
    // 对数据库中最近 N 篇文章强制重抓全文
    foreach ($entries as $entry) {
        $entry->loadCompleteContent(true);  // 强制，忽略 DB 缓存
        $entryDAO2->updateEntry($entry->toArray());
    }
}
```

**选择器预览**：
[feedController.php#L1274-L1338](app/Controllers/feedController.php#L1274-L1338)
```php
public function contentSelectorPreviewAction(): void {
    // 传入 feed_id + CSS selector，实时测试提取效果
    $feed->_pathEntries($content_selector);
    $fullContent = $entry->getContentByParsing();
    // 返回预览 HTML 给前端
}
```

**清除 Feed 缓存**：
[Feed.php#L1327-L1330](app/Models/Feed.php#L1327-L1330)
```php
public function clearCache(): bool {
    $this->faviconRebuild();
    return @unlink($this->cacheFilename());
}
```
