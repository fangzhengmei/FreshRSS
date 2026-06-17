# Feed 编码失败的显示链路与候选编码逻辑详解

本文档从代码实现角度深入分析两个问题：
1. **编码失败后的显示链路**：feed 刷新失败后怎么记录错误、页面从哪里拿状态
2. **候选编码构建逻辑**：MIME 类型分支如何影响候选编码、XML 声明编码如何加入候选

---

## 第一部分：编码失败后的显示链路

### 1.1 整体链路概览

```
SimplePie::init() 失败
    │  设置 $this->error
    ▼
FreshRSS_Feed::load()
    │  捕获/检查错误，抛出 FreshRSS_Feed_Exception
    ▼
feedController::actualizeFeeds()
    │  catch 异常
    ├─ Minz_Log::warning() 写日志
    ├─ FeedDAO::updateLastError() 更新数据库 error 字段
    └─ $feed->_error(time()) 更新内存对象
           │
           ▼
数据库：_feed.error 字段 (INT，存时间戳)
           │
           ▼
subscriptionController::feedAction()
    │  $feedDAO->searchById() 查出 feed 对象
    │  $this->view->feed = $feed;
    ▼
View: helpers/feed/update.phtml
    ├─ $this->feed->inError()  判断是否有错误
    └─ $this->feed->lastError() 取最后错误时间戳
```

---

### 1.2 SimplePie 层之前：重试限流检查

在调用 SimplePie 之前，`FreshRSS_Feed::load()` 会先检查是否处于限流冷却期：

相关文件：[Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L612-L615)

```php
if (($retryAfter = FreshRSS_http_Util::getRetryAfter($this->url, $this->proxyParam())) > 0) {
    throw new FreshRSS_Feed_Exception('For that domain, will first retry after ' . date('c', $retryAfter) .
        '. ' . $this->url(includeCredentials: false), code: 503);
}
```

这是一个在 SimplePie 之前就可能抛出错误的特殊路径，用于避免短时间内频繁请求被封禁的域名。

---

### 1.3 SimplePie 层：错误产生的源头

编码失败导致的错误发生在 `SimplePie::init()` 方法中，分两种情况：

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1981-L2002)

#### 情况 A：转码成功但 XML 解析失败

```php
if (isset($parser)) {
    $this->error = $this->feed_url;
    $this->error .= sprintf(
        ' is invalid XML, likely due to invalid characters. XML error: %s at line %d, column %d',
        $parser->get_error_string(),
        $parser->get_current_line(),
        $parser->get_current_column()
    );
}
```

**触发条件**：
- 某个候选编码转码成功（`change_encoding()` 返回非空）
- 但 XML 解析失败（`$parser->parse()` 返回 false）
- 并且这是**最后一次**尝试（$parser 变量保留了最后一次解析的错误）

**典型场景**：
- 源文件是 UTF-8 但 XML 语法有错误
- 编码猜错了，转码后产生乱码，导致 XML 解析失败

#### 情况 B：所有候选编码转码都失败

```php
} else {
    $this->error = 'The data could not be converted to UTF-8.';
    if (!extension_loaded('mbstring') && !extension_loaded('iconv') && !class_exists('\UConverter')) {
        $this->error .= ' You MUST have either the iconv, mbstring or intl (PHP 5.5+) extension installed and enabled.';
    } else {
        // ... 提示缺少哪些扩展
    }
}
```

**触发条件**：所有候选编码的 `change_encoding()` 都返回 `false`。

**典型场景**：
- PHP 缺少转码扩展（iconv、mbstring、intl 都没有）
- 非常罕见的编码，所有转码库都不支持

#### 其他可能的错误

除了编码相关的错误，还有一些其他错误也可能在 `init()` 中产生：

1. **空响应**：`"A feed could not be found at ... Empty body."`
2. **不是有效 Feed**：`"This does not appear to be a valid RSS or Atom feed."`

这些错误也会走同样的记录和显示链路。

---

### 1.4 FreshRSS Feed 模型层：异常封装

`FreshRSS_Feed::load()` 方法调用 SimplePie，并检查错误状态：

相关文件：[Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L601-L652)

```php
$simplePieResult = $simplePie->init();

if ($simplePieResult === false
    || $simplePie->get_hash() === ''
    || !empty($simplePie->error())) {

    // 特殊 HTTP 状态码处理
    if ($simplePie->status_code() === 429) {
        $errorMessage = 'HTTP 429 Too Many Requests!';
    } elseif ($simplePie->status_code() === 503) {
        $errorMessage = 'HTTP 503 Service Unavailable!';
    } else {
        $errorMessage = $simplePie->error();
        // 错误消息可能是字符串或数组
        if (is_array($errorMessage)) {
            $errorMessage = json_encode($errorMessage, ...);
        }
    }

    throw new FreshRSS_Feed_Exception(
        ($errorMessage == '' ? 'Unknown error for feed' : $errorMessage)
            . ' [' . $this->url(includeCredentials: false) . ']',
        $simplePie->status_code()
    );
}
```

**关键点**：
- 三重失败判断：`init() 返回 false` OR `hash 为空` OR `error() 非空`
- 429 和 503 状态码有特殊的错误消息（不使用 SimplePie 的 error 文本）
- 异常消息格式：`"{错误描述} [{feed_url}]"`
- 异常 code 字段是 HTTP 状态码

---

### 1.5 Controller 层：错误记录与数据库更新

`feedController::actualizeFeeds()` 是刷新 feed 的核心入口，捕获 `FreshRSS_Feed_Exception`：

相关文件：[feedController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Controllers/feedController.php#L583-L595)

```php
try {
    $simplePie = $feed->load(false, $feedIsNew);
    // ... 成功处理
} catch (FreshRSS_Feed_Exception $e) {
    Minz_Log::warning($e->getMessage());          // 1. 写日志
    $feedDAO->updateLastError($feed->id());      // 2. 更新数据库
    $feed->_error(time());                        // 3. 更新内存对象

    if ($e->getCode() === 410) {
        // HTTP 410 Gone：自动 mute
        Minz_Log::warning('Muting gone feed: ' . $feed->url(false));
        $feedDAO->mute($feed->id(), true);
        $feed->_ttl(-abs($feed->ttl()));
    }

    $feed->unlock();
    continue;  // 跳过当前 feed，继续处理下一个
}
```

#### 数据库更新：FeedDAO::updateLastError()

相关文件：[FeedDAO.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/FeedDAO.php#L247-L262)

```sql
UPDATE `_feed` SET error=:last_update WHERE id=:id
```

**`error` 字段的含义**：
- 类型：`INT`
- 值为 `0` → 无错误
- 值为 `>0` → 最后一次错误的 Unix 时间戳
- **不存储具体错误信息**，只存储时间戳！

> ⚠️ 重要：错误详情只存在于日志中，数据库只记录"有没有错"和"什么时候错的"。

---

### 1.6 View 层：错误状态展示

页面通过 feed 对象的两个方法判断和显示错误：

#### inError() — 是否有错误

相关文件：[Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L380-L382)

```php
public function inError(): bool {
    return $this->error > 0;
}
```

#### lastError() — 最后错误时间

相关文件：[Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L373-L375)

```php
public function lastError(): int {
    return $this->error;
}
```

#### 视图模板中的使用

相关文件：[helpers/feed/update.phtml](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/views/helpers/feed/update.phtml#L17-L34)

```php
<?php if ($this->feed->inError()): ?>
    <p class="alert alert-error">
        <span class="alert-head"><?= _t('gen.short.damn') ?></span>
        <?= _t('sub.feed.error') ?><br />
        <?php if ($this->feed->lastError() > 1) { ?>
            <?= _t('sub.feed.last-error-date',
                timestampToMachineDate($this->feed->lastError()),
                timeago($this->feed->lastError())
            ) ?><br />
        <?php } ?>
<?php else: ?>
    <p class="alert alert-success">
<?php endif; ?>
```

**显示逻辑**：
- 有错误 → 红色 `alert-error` 提示框，显示"Damn" + "Error while loading feed." + 最后错误时间
- 无错误 → 绿色 `alert-success` 提示框，显示最后更新时间
- `lastError() > 1` 才显示具体时间（值为 1 是 legacy 情况，只有错误标记没有时间）

#### 其他显示位置

除了 feed 管理页面，错误状态还会在以下位置显示：

1. **Feed 列表页面**（subscription/index.phtml）：feed 名称会添加 `error` CSS 类（通常显示为红色），鼠标悬停显示"Error while loading feed."提示。

2. **分类错误状态**：如果分类下有 feed 出错，分类标题也会显示错误样式（`$cat->inError()`）。

3. **错误筛选**：支持只显示有错误的 feed（`onlyFeedsWithError` 参数）。

---

### 1.7 错误清零的条件

error 字段被重置为 0 有两种主要场景：

#### 场景 1：正常刷新成功（最常见）

`actualizeFeeds()` 成功路径中会调用 `updateLastUpdate()`，该方法同时更新 `lastUpdate` 和 `error=0`：

相关文件：[FeedDAO.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/FeedDAO.php#L230-L233)

```sql
UPDATE `_feed` SET `lastUpdate`=:last_update, error=0 WHERE id=:id
```

**关键点**：
- 数据库中的 error 被重置为 0
- 但**内存中的 `$feed->error` 属性不会被更新**（只有 `lastUpdate` 被同步到内存对象）
- 由于页面展示时会从数据库重新查询 feed 对象，所以用户看到的状态是正确的

相关文件：[feedController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Controllers/feedController.php#L753-L755)

```php
if ($simplePiePush === null) {    // Not WebSub
    $feedDAO->updateLastUpdate($feed->id(), $mtime);
    $feed->_lastUpdate($mtime);   // 只同步了 lastUpdate，没有同步 error
    // ...
}
```

#### 场景 2：WebSub 推送成功

当收到 WebSub 推送且 feed 当前处于错误状态时，会显式重置 error：

相关文件：[feedController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Controllers/feedController.php#L758-L762)

```php
} elseif ($feed->inError()) {
    // Reset feed error state in case of successful WebSub push
    $feedDAO->updateLastError($feed->id(), 0);
    $feed->_error(0);
}
```

**关键点**：
- WebSub 场景下，数据库和内存对象**都会**被显式更新
- 只有当 feed 当前处于错误状态时才会重置（避免不必要的数据库写入）

---

## 第二部分：候选编码构建逻辑详解

### 2.1 整体结构

候选编码列表的构建发生在 `SimplePie::init()` 中，结构可以概括为：

```
候选编码列表 = []

如果有 $sniffed（即不是从缓存加载）：
    根据 $sniffed 的 MIME 类型走不同分支，追加一批编码

无论如何都追加兜底编码：
    xml_encoding() 的结果 + UTF-8 + ISO-8859-1

最后去重
```

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1908-L1945)

---

### 2.2 $sniffed 什么时候存在？

`$sniffed` 是 Content-Type 嗅探的结果，来自 `fetch_data()` 方法。

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L2300-L2304)

```php
$fileResponse = File::fromResponse($file);
$sniffer = $this->registry->create(Sniffer::class, [$fileResponse]);
$sniffed = $sniffer->get_type();

return [$headers, $sniffed];
```

**但 `fetch_data()` 有三种返回情况**：
- 返回 `true` → 从缓存加载成功，直接走缓存数据，**不会到编码检测阶段**
- 返回 `false` → 获取失败，直接返回错误，**不会到编码检测阶段**
- 返回 `[$headers, $sniffed]` → 成功获取了新数据，才会进行编码检测

所以 `$sniffed` 只有在"成功获取新数据（不是缓存）"的情况下才存在。

---

### 2.3 MIME 类型嗅探的完整分支

`Sniffer::get_type()` 方法负责根据 HTTP 头和内容嗅探 MIME 类型。

相关文件：[Sniffer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/Content/Type/Sniffer.php#L49-L91)

#### 分支逻辑图

```
有 Content-Type 头吗？
    │
    ├─ 是
    │   │
    │   ├─ 是 text/plain (+ 特定 charset) 且 无 content-encoding
    │   │   → 调用 text_or_binary() 嗅探（可能返回 text/plain 或 application/octet-stream）
    │   │
    │   ├─ 提取 type/subtype 部分（去掉 ;charset 等参数）
    │   │
    │   ├─ unknown/unknown 或 application/unknown
    │   │   → 调用 unknown() 深度嗅探
    │   │
    │   ├─ 以 +xml 结尾 → 直接返回该类型
    │   │   （注意：只检查后缀，不区分 application/ 和 text/）
    │   │
    │   ├─ image/* → 调用 image() 验证后返回
    │   │
    │   ├─ text/html、text/xml、application/xml
    │   │   → 调用 feed_or_html() 深度嗅探
    │   │
    │   └─ 其他情况 → 直接返回官方类型
    │
    └─ 否
        → 调用 unknown() 深度嗅探
```

**FreshRSS 的修改点**（与原版 SimplePie 的区别）：

1. **第72行**：增加了 `+xml` 后缀的直接返回（原版没有这个分支）
   ```php
   elseif (substr($official, -4) === '+xml') {
       return $official;
   }
   ```

2. **第81-82行**：增加了 `text/xml` 和 `application/xml` 进入 `feed_or_html()` 判断
   ```php
   elseif ($official === 'text/html'
       || $official === 'text/xml' // FreshRSS
       || $official === 'application/xml' // FreshRSS
   ) {
       return $this->feed_or_html();
   }
   ```

#### feed_or_html() 的返回值

`feed_or_html()` 方法通过扫描文件开头的标签来判断是 RSS/Atom feed 还是 HTML：

相关文件：[Sniffer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/Content/Type/Sniffer.php#L178-L232)

**可能的返回值**：

| 检测到的内容 | 返回类型 | 对应编码分支 |
|------------|---------|------------|
| `<rss` 或 `<rdf:RDF` | `application/rss+xml` | 分支 A（application/*+xml） |
| `<feed` | `application/atom+xml` | 分支 A（application/*+xml） |
| 其他情况（注释、DOCTYPE、HTML 标签等） | `text/html` | 分支 C（其他 text/*） |

> 💡 这意味着：即使 HTTP 头声明的是 `text/xml` 或 `application/xml`，只要内容检测到是 RSS/Atom feed，最终 sniffed 类型就会是 `application/rss+xml` 或 `application/atom+xml`，从而进入**分支 A**（能获得更优的编码检测策略）。

#### text_or_binary() 中的 BOM 检测

对于 `text/plain` 类型（带特定 charset），会通过 `text_or_binary()` 进一步判断是文本还是二进制：

相关文件：[Sniffer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/Content/Type/Sniffer.php#L98-L112)

```php
if (substr($body, 0, 2) === "\xFE\xFF"        // UTF-16BE BOM
    || substr($body, 0, 2) === "\xFF\xFE"     // UTF-16LE BOM
    || substr($body, 0, 4) === "\x00\x00\xFE\xFF"  // UTF-32BE BOM
    || substr($body, 0, 3) === "\xEF\xBB\xBF") {   // UTF-8 BOM
    return 'text/plain';
}
```

如果检测到 BOM，就认为是文本，返回 `text/plain`（进入分支 C，默认 UTF-8）。

---

### 2.4 MIME 类型如何影响候选编码

回到 `SimplePie::init()` 中的编码候选构建，`$sniffed` 决定走哪个分支：

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1916-L1937)

```php
$application_types = ['application/xml', 'application/xml-dtd', 'application/xml-external-parsed-entity'];
$text_types = ['text/xml', 'text/xml-external-parsed-entity'];

if (isset($sniffed)) {
    // 分支 A：application/*+xml 类型
    if (in_array($sniffed, $application_types)
        || (substr($sniffed, 0, 12) === 'application/' && substr($sniffed, -4) === '+xml')) {
        // ...
    }
    // 分支 B：text/*+xml 类型
    elseif (in_array($sniffed, $text_types)
        || (substr($sniffed, 0, 5) === 'text/' && substr($sniffed, -4) === '+xml')) {
        // ...
    }
    // 分支 C：其他 text/* 类型
    elseif (substr($sniffed, 0, 5) === 'text/') {
        // ...
    }
    // 分支 D：非 text 类型 → 不添加任何编码
}
```

#### 分支 A：application/*+xml 类型

**匹配条件**（满足任一）：
- `application/xml`
- `application/xml-dtd`
- `application/xml-external-parsed-entity`
- 以 `application/` 开头且以 `+xml` 结尾（如 `application/rss+xml`、`application/atom+xml`）

**追加的编码**：
1. HTTP Content-Type 头中的 charset（如果有）
2. `xml_encoding()` 返回的整个数组（BOM 检测 + XML 声明解析结果）
3. `'UTF-8'`（RFC 3023 规定 application/xml 默认 UTF-8）

#### 分支 B：text/*+xml 类型

**匹配条件**（满足任一）：
- `text/xml`
- `text/xml-external-parsed-entity`
- 以 `text/` 开头且以 `+xml` 结尾

**追加的编码**：
1. HTTP Content-Type 头中的 charset（如果有）
2. `'US-ASCII'`（RFC 3023 规定 text/xml 默认 US-ASCII）

> ⚠️ 注意：分支 B **不调用** `xml_encoding()`！这是与分支 A 的重要区别。
> 原因：RFC 3023 规定 text/xml 的默认编码是 US-ASCII，且优先相信 HTTP 头。

#### 分支 C：其他 text/* 类型

**匹配条件**：以 `text/` 开头但不是 XML 类型（如 `text/html`、`text/plain`）

**追加的编码**：
1. `'UTF-8'`

> 对于 HTML 等非 XML 文本，默认假设是 UTF-8。

#### 分支 D：非 text 类型

**匹配条件**：既不是 application/xml 系列，也不是 text 系列（如 `application/octet-stream`、图片等）

**追加的编码**：无（什么都不加）

---

### 2.5 兜底编码（无条件追加）

不管 `$sniffed` 是什么、甚至 `$sniffed` 不存在，都会追加这一批编码：

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1939-L1942)

```php
$encodings = array_merge($encodings, $this->registry->call(Misc::class, 'xml_encoding', [$this->raw_data, &$this->registry]));
$encodings[] = 'UTF-8';
$encodings[] = 'ISO-8859-1';
```

追加顺序：
1. `xml_encoding()` 返回的数组（BOM 检测 + XML 声明）
2. `'UTF-8'`
3. `'ISO-8859-1'`

**为什么兜底还要再调一次 xml_encoding()？**
- 如果 `$sniffed` 不存在（从缓存加载？）或者走的是分支 B/C/D，可能之前没有调用过 `xml_encoding()`
- 确保无论 MIME 类型如何，BOM 和 XML 声明的检测结果都会被考虑

---

### 2.6 xml_encoding() 的返回值详解

`Misc::xml_encoding()` 返回一个**数组**，数组元素的顺序就是优先级顺序。

相关文件：[Misc.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/Misc.php#L2011-L2088)

#### 返回值的可能情况

##### 情况 1：有 BOM → 数组只有 1 个元素

```
BOM 字节序列        → 返回数组
──────────────────────────────────
\x00\x00\xFE\xFF    → ['UTF-32BE']
\xFF\xFE\x00\x00    → ['UTF-32LE']
\xFE\xFF            → ['UTF-16BE']
\xFF\xFE            → ['UTF-16LE']
\xEF\xBB\xBF        → ['UTF-8']
```

注意：BOM 检测是 **if-elseif 链**，只匹配第一个命中的。

##### 情况 2：无 BOM 但检测到 `<?xml` 声明 → 数组有 1~2 个元素

对于 ASCII 兼容的 UTF-8/单字节编码（前5字节是 `<?xml`）：

```php
elseif (substr($data, 0, 5) === "\x3C\x3F\x78\x6D\x6C") {  // "<?xml"
    if ($pos = strpos($data, "\x3F\x3E")) {  // "?>"
        $parser = $registry->create(Parser::class, [substr($data, 5, $pos - 5)]);
        if ($parser->parse()) {
            $encoding[] = $parser->encoding;  // ① 解析出的 encoding 属性
        }
    }
    $encoding[] = 'UTF-8';  // ② 默认 UTF-8
}
```

返回数组的两种可能：
- 成功解析出 encoding 属性 → `["GB2312", "UTF-8"]`（解析出的编码在前，优先级更高）
- 解析失败或没有 encoding 属性 → `["UTF-8"]`

对于 UTF-16/UTF-32 的情况也是类似逻辑：
1. 先把声明部分按推断的编码转成 UTF-8
2. 解析 encoding 属性
3. 返回 `[解析出的编码, 推断的基础编码]`

例如 UTF-16BE 无 BOM 但有声明：
```php
$encoding[] = $parser->encoding;  // 可能是 'UTF-16' 或其他
$encoding[] = 'UTF-16BE';         // 推断的基础编码
```

##### 情况 3：什么都没检测到 → 数组只有 1 个元素

```php
else {
    $encoding[] = 'UTF-8';
}
```

返回 `["UTF-8"]` 作为兜底。

---

### 2.7 编码候选列表示例

假设一个 `application/rss+xml` 类型的 feed，HTTP 头声明 `charset=ISO-8859-1`，但文件实际带 UTF-8 BOM 和 `<?xml version="1.0" encoding="GB2312"?>` 声明：

**编码候选列表构建过程**：

| 步骤 | 来源 | 添加的编码 | 列表状态 |
|------|------|-----------|----------|
| 1 | input_encoding (无) | - | [] |
| 2 | HTTP charset (分支A) | ISO-8859-1 | [ISO-8859-1] |
| 3 | xml_encoding() (分支A) | UTF-8 (BOM 检测) | [ISO-8859-1, UTF-8] |
| 4 | RFC 3023 默认 (分支A) | UTF-8 | [ISO-8859-1, UTF-8] (去重) |
| 5 | 兜底 xml_encoding() | UTF-8 (BOM 检测) | [ISO-8859-1, UTF-8] (去重) |
| 6 | 兜底 UTF-8 | UTF-8 | [ISO-8859-1, UTF-8] (去重) |
| 7 | 兜底 ISO-8859-1 | ISO-8859-1 | [ISO-8859-1, UTF-8] (去重) |

**最终候选列表**：`["ISO-8859-1", "UTF-8"]`

> 等等，不对。如果有 BOM 的话，xml_encoding() 应该只返回 ['UTF-8']（情况 1）。
> 但如果有 BOM，为什么还会有 XML 声明里的 GB2312？
> 实际上有 BOM 的情况下，xml_encoding() 直接返回 BOM 对应的编码，**不会去解析 XML 声明**。
> 
> 让我修正这个例子...

再举一个更真实的例子：UTF-8 编码的 RSS feed，没有 BOM，XML 声明写的是 `encoding="UTF-8"`，HTTP Content-Type 是 `application/rss+xml; charset=utf-8`：

| 步骤 | 来源 | 添加的编码 | 列表状态 |
|------|------|-----------|----------|
| 1 | input_encoding (无) | - | [] |
| 2 | HTTP charset (分支A) | UTF-8 | [UTF-8] |
| 3 | xml_encoding() (分支A) | UTF-8 (声明解析) → UTF-8 (默认) | [UTF-8] (去重) |
| 4 | RFC 3023 默认 (分支A) | UTF-8 | [UTF-8] (去重) |
| 5-7 | 兜底编码 | UTF-8 + ISO-8859-1 | [UTF-8, ISO-8859-1] |

**最终候选列表**：`["UTF-8", "ISO-8859-1"]`

再举一个声明编码和实际不一致的例子：文件实际是 GBK 编码，XML 声明写 `encoding="GB2312"`，HTTP Content-Type 是 `text/xml`（无 charset）：

| 步骤 | 来源 | 添加的编码 | 列表状态 |
|------|------|-----------|----------|
| 1 | input_encoding (无) | - | [] |
| 2 | HTTP charset (分支B，无) | - | [] |
| 3 | RFC 3023 默认 (分支B) | US-ASCII | [US-ASCII] |
| 4 | 兜底 xml_encoding() | GB2312 (声明) → UTF-8 (默认) | [US-ASCII, GB2312, UTF-8] |
| 5 | 兜底 UTF-8 | UTF-8 | [US-ASCII, GB2312, UTF-8] (去重) |
| 6 | 兜底 ISO-8859-1 | ISO-8859-1 | [US-ASCII, GB2312, UTF-8, ISO-8859-1] |

**最终候选列表**：`["US-ASCII", "GB2312", "UTF-8", "ISO-8859-1"]`

尝试顺序：US-ASCII → GB2312 → UTF-8 → ISO-8859-1

实际 GBK 编码的数据：
- US-ASCII 会在第一个高位字节处截断，转码成功但内容不完整，XML 解析失败
- GB2312 可能转码成功（GBK 是 GB2312 的超集），XML 可能成功
- 如果 GB2312 不行，继续试 UTF-8、ISO-8859-1

---

### 2.8 去重机制

最后会调用 `array_unique()` 去重：

```php
$encodings = array_unique($encodings);
```

`array_unique()` 会**保留第一个出现的键值**，所以优先级高的编码会保留在前面。

注意：`array_unique()` 是字符串比较，`'UTF-8'` 和 `'utf-8'` 会被认为是不同的。但所有添加的编码都经过了 `strtoupper()` 或 `XML\Declaration\Parser`（也会转大写），所以不会有大小写不一致的问题。

---

## 第三部分：关键问题解答

### Q1：为什么数据库里只存错误时间戳，不存错误详情？

设计取舍：
- 错误详情写在日志里（`Minz_Log::warning`），需要排查时看日志
- 数据库只存状态标记，节省空间且查询快
- UI 层只需要"有没有错/什么时候错的"来决定显示样式

### Q2：转码失败后 error 字段什么时候会被清零？

下一次刷新成功时。`actualizeFeeds()` 成功路径会更新 feed 记录，error 字段被重置为 0。

### Q3：如果所有编码都转码成功但都解析失败，最终错误信息是哪个编码的？

是**最后一个**候选编码的解析错误。因为 `$parser` 变量在循环中每次都会被覆盖，循环结束后保留的是最后一次的结果。

最后一个编码通常是 `ISO-8859-1`（因为它总能转码成功），所以最后那条错误信息往往是"乱码导致的 XML 解析错误"，参考价值有限。

### Q4：text/xml 类型为什么不做 BOM 检测？

RFC 3023 规定 `text/xml` 的默认编码是 US-ASCII，且 HTTP 头的 charset 优先级最高。SimplePie 遵循这个规范，在 text/xml 分支只追加 HTTP charset 和 US-ASCII 默认值。

但**兜底阶段**仍然会调用 `xml_encoding()`，所以 BOM 和 XML 声明实际上还是会被考虑，只是优先级靠后。

### Q5：有 BOM 的情况下还会解析 XML 声明里的 encoding 吗？

不会。`xml_encoding()` 中 BOM 检测放在最前面，是 if-elseif 结构。只要命中了 BOM，就直接返回对应的编码，不会再去解析 XML 声明。

这符合 XML 规范：BOM 的优先级高于 XML 声明中的 encoding 属性。

### Q6：input_encoding 是怎么用的？什么时候设置？

`input_encoding` 是用户强制指定的编码，优先级最高。可以通过 `set_input_encoding()` 设置。

在 FreshRSS 中，feed 的属性中没有直接的 input_encoding 设置。一般用户不需要手动指定，SimplePie 的自动检测通常能搞定。

### Q7：三重失败判断（init返回false / hash为空 / error非空）各是什么含义？

相关文件：[Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L634)

```php
if ($simplePieResult === false
    || $simplePie->get_hash() === ''
    || !empty($simplePie->error())) {
```

三种失败条件的含义：

| 条件 | 含义 | 典型场景 |
|-----|------|---------|
| `init() === false` | 初始化完全失败 | 网络错误、空响应、所有编码都失败 |
| `get_hash() === ''` | 内容哈希为空 | 空内容、或解析后没有生成 hash |
| `!empty(error())` | 有错误消息 | XML 解析错误、不是有效 feed 等 |

三者是 OR 关系，满足任意一个就认为失败。这是一种多重保险的设计，防止某些情况下 init 返回 true 但实际内容有问题。

### Q8：从 HTTP 头的 text/xml 到最终进入编码分支 A，中间发生了什么？

完整链路：

1. HTTP 头返回 `Content-Type: text/xml`
2. Sniffer::get_type() 检测到 `text/xml`，调用 `feed_or_html()`
3. `feed_or_html()` 扫描内容，发现 `<rss` 或 `<feed` 标签
4. 返回 `application/rss+xml` 或 `application/atom+xml`
5. SimplePie::init() 中，sniffed 类型匹配 `application/*+xml`，进入**分支 A**
6. 分支 A 追加：HTTP charset + xml_encoding() 结果 + UTF-8

> 💡 这就是 FreshRSS 修改 Sniffer 的意义：即使服务器错误地返回 `text/xml` 或 `application/xml`，只要内容确实是 RSS/Atom，最终也能进入最优的编码检测分支（分支 A）。

### Q9：为什么分支 A 和兜底都调用 xml_encoding()，不会重复吗？

确实会重复调用，但因为有 `array_unique()` 去重，所以最终候选列表不会有重复。

重复调用的原因：
- **分支 A 中调用**：让 BOM 和 XML 声明检测结果有更高的优先级（排在 RFC 3023 默认编码前面）
- **兜底调用**：确保即使 `$sniffed` 不存在（比如从缓存加载）或者走了其他分支，BOM 和 XML 声明仍然会被考虑

代价是 `xml_encoding()` 被调用了两次（对于分支 A 的情况），但这是一个很轻量的操作（只是字符串比较和简单解析），性能影响可以忽略。
