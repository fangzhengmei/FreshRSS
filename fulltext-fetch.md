# FreshRSS 全文抓取工作流程

本文档梳理 FreshRSS 中 RSS 全文抓取的完整代码流转，包括触发条件、正文清洗与缓存、以及展示层合并策略。

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
| 实际化脚本 | [actualize_script.php](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/actualize_script.php) | CLI 定时刷新入口 |
| Feed 控制器 | [feedController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Controllers/feedController.php) | 触发 actualize 动作 |
| Feed 模型 | [Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/Feed.php) | 订阅源加载、条目解析、缓存管理 |
| Entry 模型 | [Entry.php](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/Entry.php) | 全文抓取 `loadCompleteContent()`、内容解析 `getContentByParsing()` |
| SimplePie 定制 | [SimplePieCustom.php](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/SimplePieCustom.php) | HTML 内容清洗 `sanitizeHTML()` |
| HTTP 工具 | [httpUtil.php](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Utils/httpUtil.php) | HTTP 请求、缓存层、编码处理 |
| 视图层 | [normal.phtml](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/views/index/normal.phtml) | 内容展示 `$entry->content(true)` |

---

## 二、触发条件（何时启动全文抓取）

### 2.1 四大触发入口

#### 入口 1：CLI 定时任务（最常用）

**文件**: [actualize_script.php#L13-L16](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/actualize_script.php#L13-L16)

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

**文件**: [feedController.php#L954-L1033](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Controllers/feedController.php#L954-L1033)

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

**文件**: [feedController.php#L115](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Controllers/feedController.php#L115)

```php
// Ok, feed has been added in database. Now we have to refresh entries.
self::actualizeFeedsAndCommit($id, $url);
```

- 订阅新源时立即触发一次完整抓取

#### 入口 4：WebSub（PubSubHubbub）实时推送

**文件**: [p/api/pshb.php](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/p/api/pshb.php)

- 支持 WebSub 的源直接推送更新
- 调用 `actualizeFeedsAndCommit()`，传入 `$simplePiePush`

---

### 2.2 actualizeFeeds 核心流程

**文件**: [feedController.php#L427-L849](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Controllers/feedController.php#L427-L849)

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

**文件**: [Feed.php#L601-L693](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/Feed.php#L601-L693)

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

**文件**: [Feed.php#L807-L942](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/Feed.php#L807-L942)

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

**文件**: [Entry.php#L1063-L1145](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/Entry.php#L1063-L1145)

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
                    $this->content = $fullContent . $originalContent;
                    break;
                case 'append':   // 摘要在前，全文在后
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
    elseif (trim($feed->attributeString('path_entries_filter')) !== '') {
        // 对 RSS 原内容执行 CSS 选择器过滤（不发HTTP请求）
    }
    // 【分支C】不做任何全文处理，保持 RSS 摘要
    return false;
}
```

### 3.2 触发全文抓取的**必要条件**

| 条件 | 位置 | 说明 |
|------|------|------|
| `pathEntries` 非空 | Feed 属性 `$_pathEntries` | **必需条件**：CSS 选择器，指定提取正文的 DOM 节点 |
| `path_entries_conditions` | Feed attributes | **可选**：布尔搜索条件，满足条件的文章才抓取全文 |
| 文章不在 DB 中 | `searchByGuid()` 返回 null | 避免重复抓取已入库文章（除非 `$force=true`） |
| 文章有可用的 link | `$this->link()` | 作为抓取目标 URL |

---

### 3.3 `getContentByParsing()` — 实际抓取与解析

**文件**: [Entry.php#L924-L1058](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/Entry.php#L924-L1058)

```
文章链接 → [HTTP请求+缓存] → HTML内容 → [DOM解析] → [CSS选择器提取] → [过滤] → [HTML清洗] → 纯净全文
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

#### Step 3：CSS 选择器提取正文

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

#### Step 4：SimplePie 内容清洗

**文件**: [SimplePieCustom.php#L285-L310](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/SimplePieCustom.php#L285-L310)

```php
$html = FreshRSS_SimplePieCustom::sanitizeHTML($html, $base);
```

清洗规则定义在 `SimplePieCustom` 构造函数中，包括：

| 清洗类别 | 配置位置 | 说明 |
|----------|----------|------|
| 允许的 HTML 属性 | [L68-L80](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/SimplePieCustom.php#L68-L80) | `dir`, `lang`, `title`, `role` 等 |
| 允许的 HTML 元素及属性 | [L81-L226](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/SimplePieCustom.php#L81-L226) | 大段白名单：`a`, `img`, `video`, `table`, MathML 等 |
| 强制剥离的属性 | [L227-L232](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/SimplePieCustom.php#L227-L232) | `data-original` 等不安全属性 |
| 强制添加的属性 | [L233-L241](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/SimplePieCustom.php#L233-L241) | `<audio controls>`, `<iframe sandbox>` 等安全加固 |
| URL 重写规则 | [L242-L267](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/SimplePieCustom.php#L242-L267) | 相对 URL 转绝对 URL |
| HTTPS 强制域名 | [L268-L282](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/SimplePieCustom.php#L268-L282) | `force-https.txt` 列表中的域名强制 https |
| 禁止的 URI Scheme | [L67](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/SimplePieCustom.php#L67) | `javascript:` 被禁用 |

#### Step 5：二次过滤

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

**文件**: [httpUtil.php#L271-L445](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Utils/httpUtil.php#L271-L445)

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

**文件**: [Feed.php#L1295-L1317](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/Feed.php#L1295-L1317)

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

代码实现参考 [Entry.php#L1083-L1097](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/Entry.php#L1083-L1097)。

### 5.2 内容标记与恢复

**标记全文内容**：
```php
$fullContent = "<!-- FULLCONTENT start //-->{$fullContent}<!-- FULLCONTENT end //-->";
```

**恢复原始内容**（如需）：
```php
public function originalContent(): string {
    return $this->attributeString('original_content') ??
        preg_replace('#<!-- FULLCONTENT start //-->.*<!-- FULLCONTENT end //-->#s', 
            '', $this->content) ?? '';
}
```

### 5.3 入数据库

**文件**: [EntryDAO.php](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/EntryDAO.php)

```php
// 新文章
$entryDAO->addEntry($entry->toArray(), true);  // 写入 _entrytmp 临时表

// 已有文章更新
$entryDAO->updateEntry($entry->toArray());

// commit 阶段：从 _entrytmp 移到 _entry 主表
$entryDAO->commitNewEntries();
```

### 5.4 哈希与去重

**文件**: [Entry.php#L513-L520](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/Entry.php#L513-L520)

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

## 六、展示层合并

### 6.1 视图渲染入口

**文件**: [normal.phtml#L130-L132](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/views/index/normal.phtml#L130-L132)

```php
<div class="text"><?=
    FreshRSS_Context::userConf()->lazyload && !FreshRSS_Context::userConf()->display_posts 
        ? lazyimg($this->entry->content(true))   // 懒加载：图片延迟加载
        : $this->entry->content(true)            // 正常渲染
?></div>
```

Helper 模板中也使用同样方式：[article.phtml#L102](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/views/helpers/index/article.phtml#L102)

### 6.2 `content()` 方法 — 最终合并

**文件**: [Entry.php#L220-L313](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/Entry.php#L220-L313)

```php
public function content(bool $withEnclosures = true, bool $allowDuplicateEnclosures = false): string {
    if (!$withEnclosures) {
        return $this->content;  // 仅返回已合并的正文
    }
    
    $content = $this->content;
    
    // ① 追加缩略图 enclosure
    $thumbnailAttribute = $this->attributeArray('thumbnail') ?? [];
    if (!empty($thumbnailAttribute['url'])) {
        $content .= '<figure class="enclosure">
            <p class="enclosure-content">
                <img class="enclosure-thumbnail" src="{$elink}" alt="" />
            </p>
        </figure>';
    }
    
    // ② 追加附件 enclosures（图片、音频、视频、文件）
    $attributeEnclosures = $this->attributeArray('enclosures');
    foreach ($attributeEnclosures as $enclosure) {
        $mime = $enclosure['type'] ?? '';
        if (self::enclosureIsImage($enclosure)) {
            $content .= '<p class="enclosure-content"><img src="'.$elink.'" /></p>';
        } elseif (str_starts_with($mime, 'audio')) {
            $content .= '<p class="enclosure-content"><audio controls src="'.$elink.'"></audio></p>';
        } elseif (str_starts_with($mime, 'video')) {
            $content .= '<p class="enclosure-content"><video controls src="'.$elink.'"></video></p>';
        } else {
            $content .= '<p class="enclosure-content"><a download href="'.$elink.'">💾</a></p>';
        }
    }
    
    return $content;
}
```

**展示层的最终内容** = `[RSS摘要 + 全文]（按策略合并） + [缩略图] + [附件 enclosures]`

---

## 七、完整时序图

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
  │    │     └─ 比较 SimplePieHash 判断 Feed 是否变化
  │    │
  │    ├─ Feed::loadEntries() → 遍历 SimplePie items
  │    │     │
  │    │     └─ 对每个 item：
  │    │          │
  │    │          ├─ 创建 FreshRSS_Entry（含 RSS 摘要 content）
  │    │          │
  │    │          ├─ Entry::hash() → 计算内容哈希
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
  │    │                    │    │         ├─ HTTP 200？
  │    │                    │    │         │    ├─ 否 → 检查 429/503 → 写 Retry-After
  │    │                    │    │         │    └─ 是 → 编码处理、base href 注入
  │    │                    │    │         │
  │    │                    │    │         └─ 写缓存文件
  │    │                    │
  │    │                    ├─ DOMDocument::loadHTML
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
       ├─ 读取 DB 中已合并的 content
       └─ 追加 thumbnail + enclosures（图片/音频/视频附件渲染）
```

---

## 八、关键配置项与可调参数

| 配置项 | 所在文件 | 默认值 | 影响 |
|--------|----------|--------|------|
| `limits.timeout` | [config.default.php](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/config.default.php) | 15s | HTTP 请求超时（Feed抓取 + 全文抓取） |
| `limits.cache_duration` | config.default.php | 3600s | SimplePie + HTTP 缓存默认有效期 |
| `limits.cache_duration_min/max` | config.default.php | 1800~86400s | HTTP 缓存上下限 |
| `limits.retry_after_max` | config.default.php | - | HTTP 429/503 最大等待 |
| `ttl_default` | 用户配置 | - | Feed 默认刷新周期 |
| `pubsubhubbub_enabled` | 系统配置 | true | 是否启用 WebSub 实时推送 |
| `curl_options` | 系统配置 | [] | 全局代理、SSL 配置等 |
| `content_action` | Feed attributes | `replace` | 全文合并策略 |
| `pathEntries` | Feed 属性 | `''` | **全文抓取开关 + CSS 选择器** |
| `path_entries_filter` | Feed attributes | `''` | 需要移除的节点 CSS 选择器 |
| `path_entries_conditions` | Feed attributes | `[]` | 触发全文抓取的搜索条件数组 |
| `content_width` | 用户显示配置 | - | 展示层内容宽度样式 |
| `lazyload` | 用户配置 | true | 图片懒加载 |

---

## 九、特殊场景与边界条件

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

---

## 十、手动重抓与调试

### 控制器动作

**重新加载（清除 lastUpdate）**：
[feedController.php#L1208-L1264](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Controllers/feedController.php#L1208-L1264)
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
[feedController.php#L1274-L1338](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Controllers/feedController.php#L1274-L1338)
```php
public function contentSelectorPreviewAction(): void {
    // 传入 feed_id + CSS selector，实时测试提取效果
    $feed->_pathEntries($content_selector);
    $fullContent = $entry->getContentByParsing();
    // 返回预览 HTML 给前端
}
```

**清除 Feed 缓存**：
[Feed.php#L1327-L1330](file:///d:/fz/0601-2/solo-dogfeeding/code/28-FreshRSS/app/Models/Feed.php#L1327-L1330)
```php
public function clearCache(): bool {
    $this->faviconRebuild();
    return @unlink($this->cacheFilename());
}
```
