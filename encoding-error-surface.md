# Feed 刷新失败的错误状态完整展示链路

本文档按代码路径系统梳理：
1. **HTTP 429/503 的特判逻辑**（请求拦截、Retry-After 存储、冷却期跳过）
2. **未知错误的兜底文案生成**（429/503 特殊文案 → SimplePie error → 兜底 "Unknown error"）
3. **异常状态码的传递链路**（status_code 怎么传、哪里被消费）
4. **错误时间戳的传递链路**（从异常捕获到页面展示的完整流程）
5. **所有错误展示位置**（管理页、订阅列表、全局侧边栏等）

---

## 一、总览：错误状态完整流程图

```
                                          ┌───────────────────────────────────────┐
                                          │     HTTP 请求返回 429 或 503          │
                                          └───────────────────┬───────────────────┘
                                                              │
                                              SimplePieFetch::on_http_response()
                                                              │
                                                              ▼
                                               FreshRSS_http_Util::setRetryAfter()
                                                              │
                                                              ▼
                                              data/Retry-After/{domain}.txt
                                                           (mtime = 可重试时间戳)

┌────────────────────────────────────────────────────────────────────────────────────┐
│                              下次刷新时：                                            │
│  Feed::load()                                                                       │
│    │                                                                                │
│    ├─→ getRetryAfter() > 0? ──是──→ throw FreshRSS_Feed_Exception(code:503)         │
│    │                           ──否──→ SimplePie::init()                            │
│    │                                                                                │
│    │                    SimplePie::init() 内部失败路径：                             │
│    │                      ├─ fetch_data() 抛异常                                    │
│    │                      ├─ HTTP 状态码非 200                                      │
│    │                      ├─ 编码全部失败 XML 解析错误                                │
│    │                      └─ ...                                                    │
│    │                              │                                                 │
│    │                              ▼                                                 │
│    │              status_code = 实际HTTP状态码 / 0                                  │
│    │              error = 具体错误消息                                               │
│    │                              │                                                 │
│    └──────────────────────────────┘                                                 │
│                                   │                                                  │
│                                   ▼                                                  │
│               Feed::load() 三重判断命中 → 抛出 FreshRSS_Feed_Exception              │
│                                   │                                                  │
│                                   ▼                                                  │
│              feedController::actualizeFeeds() catch 异常                             │
│                                   │                                                  │
│              ┌────────────────────┼────────────────────┐                            │
│              │                    │                    │                            │
│              ▼                    ▼                    ▼                            │
│     Minz_Log::warning()   FeedDAO::updateLastError()  $feed->_error(time())         │
│                              (DB: error = time())     (内存: error = time())         │
│                                                                    │                 │
│                              code == 410? ──是──→ FeedDAO::mute() + TTL 取反         │
│                                                                    │                 │
└────────────────────────────────────────────────────────────────────┼─────────────────┘
                                                                     │
                                                              下次页面请求时：
                                                                     │
                                    ┌────────────────────────────────┼───────────────────────┐
                                    │                                │                       │
                                    ▼                                ▼                       ▼
                  subscription/index.phtml              feed/update.phtml         index/global.phtml
                  (订阅列表页面)                         (feed 管理页 slider)      (全局侧边栏)
                    │                                          │                       │
                    ▼                                          ▼                       ▼
          feed名称加 error CSS类                alert-error 红色提示框         feed名称加 error类
          title="⚠ This feed has encountered..."   "Blast! + 错误时间"          title="错误提示"
```

---

## 二、HTTP 429/503 特判逻辑

这是一条**在 SimplePie 之前就被拦截**的特殊路径。

### 阶段 1：收到 429/503 时记录 Retry-After

**触发点**：`FreshRSS_SimplePieFetch::on_http_response()` 回调

相关文件：[SimplePieFetch.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/SimplePieFetch.php#L31-L58)

```php
#[\Override]
protected function on_http_response($response, array $curl_options = []): void
{
    if (in_array($this->get_status_code(), [429, 503], true)) {
        // 解析响应头获取 Retry-After
        $parser = new \SimplePie\HTTP\Parser(...);
        $headers = $parser->parse() ? $parser->headers : [];

        $proxy = is_string($curl_options[CURLOPT_PROXY] ?? null) ? ...;
        $retryAfter = FreshRSS_http_Util::setRetryAfter(
            $this->get_final_requested_uri(),
            $proxy,
            $headers['retry-after'] ?? ''   // ← 可能是空字符串
        );
        // ...
    }
}
```

**触发条件**：只有当 HTTP 状态码是 **429** 或 **503** 时才会走这条路径。

### 阶段 2：setRetryAfter() 的存储策略

相关文件：[httpUtil.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Utils/httpUtil.php#L67-L88)

```php
public static function setRetryAfter(string $url, string $proxy, string $retryAfter): int
{
    $limits = FreshRSS_Context::systemConf()->limits;

    // Retry-After 头可能是秒数，也可能是 HTTP 日期
    if (ctype_digit($retryAfter)) {
        $retryAfter = time() + (int)$retryAfter;
    } else {
        $retryAfter = \SimplePie\Misc::parse_date($retryAfter) ?:
            (time() + max(600, $limits['retry_after_default'] ?? 0));  // 默认至少 600s
    }

    // 上限：最多限制 3600s（1小时）
    $retryAfter = min($retryAfter, time() + max(3600, $limits['retry_after_max'] ?? 0));

    // 存储方式：文件的 mtime 就是可重试的时间戳
    if (!touch($txt, $retryAfter)) { ... }
    return $retryAfter;
}
```

**三种可能的 Retry-After 值来源**：

| 情况 | 计算方式 | 示例 |
|-----|---------|------|
| 响应头是纯数字（秒数） | `time() + (int)$retryAfter` | `"300"` → 当前时间 +5 分钟 |
| 响应头是 HTTP 日期 | `\SimplePie\Misc::parse_date()` 解析 | `"Wed, 21 Oct 2015 07:28:00 GMT"` |
| 响应头缺失或解析失败 | `time() + 系统配置的默认值` | 配置 `retry_after_default = 1500` → 1500s |

**配置项参考**（[config.default.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/config.default.php#L120-L123)）：
```php
'retry_after_default' => 1500,   // Retry-After 缺失时用 25 分钟
'retry_after_max' => 172800,     // 最多限制 48 小时
```

**文件名规则**：[httpUtil.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Utils/httpUtil.php#L8-L21)
```
data/Retry-After/
  ├── urlencode({domain})_{sha256(url)}_{proxy}.txt   ← 非公共域名（含内网）
  └── urlencode({domain}).txt                          ← 公共域名（域级共享限制）
```

### 阶段 3：下次刷新时的冷却期拦截

**触发点**：`FreshRSS_Feed::load()` 方法最开头

相关文件：[Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L612-L615)

```php
if (($retryAfter = FreshRSS_http_Util::getRetryAfter($this->url, $this->proxyParam())) > 0) {
    throw new FreshRSS_Feed_Exception(
        'For that domain, will first retry after ' . date('c', $retryAfter) .
        '. ' . $this->url(includeCredentials: false),
        code: 503   // ← 异常 code 固定为 503
    );
}
```

**这一步的重要特点**：
- ✅ **完全不进入 SimplePie**，所以不会触发编码检测
- ✅ 异常 `code` 固定为 **503**，与实际 HTTP 状态码可能不同
- ✅ 错误消息包含可重试的具体时间（ISO 8601 格式）
- ✅ 每 30 次调用会自动清理过期的 Retry-After 文件

---

## 三、未知错误的兜底文案生成链路

文案生成在 `FreshRSS_Feed::load()` 中，是一个**多级兜底**的结构。

相关文件：[Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L634-L651)

```php
if ($simplePieResult === false
    || $simplePie->get_hash() === ''
    || !empty($simplePie->error())) {

    if ($simplePie->status_code() === 429) {
        $errorMessage = 'HTTP 429 Too Many Requests!';           // ← 第 1 级：429 特判
    } elseif ($simplePie->status_code() === 503) {
        $errorMessage = 'HTTP 503 Service Unavailable!';         // ← 第 2 级：503 特判
    } else {
        $errorMessage = $simplePie->error();                     // ← 第 3 级：SimplePie 原始错误
        if (empty($errorMessage)) {
            $errorMessage = '';                                   // ← 第 4 级：空字符串
        } elseif (is_array($errorMessage)) {
            $errorMessage = json_encode(...);                     // ← 数组转 JSON
        }
    }

    throw new FreshRSS_Feed_Exception(
        ($errorMessage == '' ? 'Unknown error for feed' : $errorMessage) .  // ← 第 5 级：最终兜底
            ' [' . $this->url(includeCredentials: false) . ']',
        $simplePie->status_code()
    );
}
```

### 文案优先级表

| 优先级 | 条件 | 生成的文案 | 示例 |
|-------|------|----------|------|
| 1 | `status_code === 429` | 固定特判文案 | `HTTP 429 Too Many Requests! [https://...]` |
| 2 | `status_code === 503` | 固定特判文案 | `HTTP 503 Service Unavailable! [https://...]` |
| 3 | SimplePie.error 为非空字符串 | SimplePie 原始错误 + URL | `cURL error 28: Timeout after 10001ms [https://...]` |
| 4 | SimplePie.error 为数组 | JSON 编码后 + URL | `{"type":"warning","message":"..."} [https://...]` |
| 5 | SimplePie.error 为空 | 兜底文案 + URL | `Unknown error for feed [https://...]` |

### SimplePie.error 的可能来源

SimplePie 层可能设置的错误消息包括：

| 场景 | 示例错误消息 | 代码位置 |
|-----|------------|---------|
| HTTP 请求异常 | `cURL error 28: Timeout...` / `Connection refused` | [SimplePie.php L2182](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L2182) |
| HTTP 状态码不支持 | `Retrieved unsupported status code "403"` | [SimplePie.php L2192](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L2192) |
| XML 解析错误 | `XML error: not well-formed (invalid token) at line 42` | XML Parser 回调 |
| 编码转码失败 | （通常 XML 解析错误会体现为具体解析错误） | Parser 层 |

> 💡 注意：**冷却期拦截（429/503 Retry-After）不走这条文案链路**，因为它在 `load()` 最开头就直接抛出异常，文案是单独写的：`"For that domain, will first retry after ..."`。

---

## 四、异常状态码的传递链路

### 完整传递路径

```
SimplePie.status_code (int)
        │
        ▼
FreshRSS_Feed::load()
  ├─→ Retry-After 冷却期:    throw Exception(code: 503)  ← 固定 503
  └─→ 其他错误:              throw Exception(code: status_code)
        │
        ▼
feedController::actualizeFeeds() catch ($e)
  ├─→ $e->getCode() === 410?  → 自动 mute feed
  └─→ 其他 code?              → 只记录日志 + 更新 error 时间戳
        │
        ▼
数据库 _feed.error 字段
  └─→ 只存时间戳，不存状态码！
```

### 状态码的来源

**SimplePie.status_code 的可能值**：

| 值 | 含义 | 设置位置 |
|----|------|---------|
| 0 | 请求异常（抛出 ClientException，没有 HTTP 响应） | [SimplePie.php L2089](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L2089) |
| 200 | 正常成功 | HTTP 响应本身 |
| 304 | Not Modified（缓存有效） | HTTP 响应本身 |
| 403, 404, 410, 429, 500, 503... | 各种 HTTP 错误状态码 | HTTP 响应本身 |

### 状态码在哪里被消费？

**唯一消费点：HTTP 410 自动 mute**

相关文件：[feedController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Controllers/feedController.php#L587-L592)

```php
} catch (FreshRSS_Feed_Exception $e) {
    Minz_Log::warning($e->getMessage());
    $feedDAO->updateLastError($feed->id());
    $feed->_error(time());
    if ($e->getCode() === 410) {                   // ← 只有 410 有特殊处理
        // HTTP 410 Gone
        Minz_Log::warning('Muting gone feed: ' . $feed->url(false));
        $feedDAO->mute($feed->id(), true);
        $feed->_ttl(-abs($feed->ttl()));
    }
    // ...
}
```

**状态码不会显示在页面上**。数据库只存时间戳，不存错误码。用户只能通过：
- 日志文件查看具体错误消息（包含 429/503 文案）
- 错误时间戳推断刷新失败的时间

---

## 五、错误时间戳的传递链路

这是错误状态到达用户面前的核心链路。

### 完整流程

```
步骤 1: catch 异常 → 同时更新两处
  ├─ 数据库: FeedDAO::updateLastError($feed->id())  → UPDATE _feed SET error = time()
  └─ 内存:   $feed->_error(time())                   → $this->error = time()

步骤 2: 下次页面请求
  └─ subscriptionController / feedController
       └─→ FeedDAO::searchById() 等 → 从 DB 读取 error 字段
            └─→ 赋值给 Feed 对象的 $error 属性

步骤 3: View 层消费
  ├─ $feed->inError()      → return $this->error > 0
  └─ $feed->lastError()    → return $this->error
```

### 步骤 1 详解：错误发生时

相关文件：[feedController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Controllers/feedController.php#L583-L595)

```php
} catch (FreshRSS_Feed_Exception $e) {
    Minz_Log::warning($e->getMessage());              // 记录日志（含完整错误消息）
    $feedDAO->updateLastError($feed->id());            // DB: error = NOW()
    $feed->_error(time());                             // 内存: error = NOW()
    if ($e->getCode() === 410) { ... }                 // 410 特殊处理
    $feed->unlock();
    continue;                                           // 跳过后续，处理下一个 feed
}
```

**updateLastError 的 SQL**：[FeedDAO.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/FeedDAO.php#L247-L254)

```sql
UPDATE `_feed` SET error=:last_update WHERE id=:id
-- :last_update = time()，即当前 Unix 时间戳
```

### 步骤 2 详解：错误清零时

两种清零场景（详细参考 `encoding-failure-flow.md`）：

| 场景 | 数据库 | 内存对象 |
|-----|--------|---------|
| 正常刷新成功 → `updateLastUpdate()` | `UPDATE SET error=0` | ❌ 不同步（但下次从 DB 重选就是 0） |
| WebSub 推送成功 → `updateLastError(id, 0)` | `UPDATE SET error=0` | ✅ `$feed->_error(0)` 同步 |

### 步骤 3 详解：View 层消费

Feed 对象的两个核心方法：[Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L373-L382)

```php
public function lastError(): int {
    return $this->error;       // ← 直接返回时间戳（int）
}

public function inError(): bool {
    return $this->error > 0;   // ← 0 = 无错，>0 = 有错（时间戳）
}
```

**`lastError() > 1` 的含义**：
- `error == 0`：无错误
- `error == 1`：Legacy 遗留值（旧版本可能用 1 表示"有错误但没记录时间"）
- `error > 1`：真实的 Unix 时间戳（从 1970 年算起，肯定大于 1）

所以 View 层只有在 `error > 1` 时才会格式化显示具体时间。

---

## 六、所有错误展示位置和样式

FreshRSS 在 **5 个位置** 展示 feed 错误状态。

### 位置 1：Feed 管理页（最详细）

**文件**：[feed/update.phtml](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/views/helpers/feed/update.phtml#L17-L34)

```php
<?php if ($this->feed->inError()): ?>
    <p class="alert alert-error">                       <!-- 红色背景 -->
        <span class="alert-head"><?= _t('gen.short.damn') ?></span>  <!-- "Blast!" -->
        <?= _t('sub.feed.error') ?><br />               <!-- "This feed has encountered a problem..." -->
        <?php if ($this->feed->lastError() > 1) { ?>
            <?= _t('sub.feed.last-error-date', ...) ?><br />  <!-- "Last erroneous update 2小时前" -->
        <?php } ?>
<?php else: ?>
    <p class="alert alert-success">                     <!-- 绿色背景（无错时） -->
<?php endif; ?>
        <?= _t('sub.feed.last-update', ...) ?>           <!-- 最后更新时间（始终显示） -->
    </p>
```

**展示效果**：

有错误时：
```
┌───────────────────────────────────────────────────────────┐
│ 🔴 Blast!                                                 │  ← alert-head 粗体
│ This feed has encountered a problem. If this situation... │
│ Last erroneous update 2026-06-17T10:30:00+0800 (2小时前). │
│ Last update: 2026-06-17T08:00:00+0800 (5小时前)           │  ← 旧的最后成功更新时间
└───────────────────────────────────────────────────────────┘
   (红色边框 + 浅红背景 = alert-error 样式)
```

无错误时：
```
┌───────────────────────────────────────────────────────────┐
│ Last update: 2026-06-17T10:00:00+0800 (30分钟前)          │
└───────────────────────────────────────────────────────────┘
   (绿色边框 + 浅绿背景 = alert-success 样式)
```

### 位置 2：订阅列表页 — feed 名称样式

**文件**：[subscription/index.phtml](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/views/subscription/index.phtml#L46-L70)

```php
$error_class = '';
$error_title = '';
if ($feed->inError()) {
    $error_class = ' error';                    // CSS 类：文字变红
    $error_title = _t('sub.feed.error');        // 鼠标悬停提示文案
}
// ...
$title = $error_title !== '' ? '⚠ ' . $error_title . '&#13;' : '';
// ...
<li class="item feed<?= $error_class ?>" title="<?= $title ?>">
    <span class="item-title"><?= $feed->name() ?></span>
</li>
```

**视觉效果**：
- ✅ **CSS**：feed 名称文字变成红色（`.feed.error` 样式）
- ✅ **Tooltip**：鼠标悬停时显示 `⚠ This feed has encountered a problem...`
- ✅ 配合 `onlyFeedsWithError` 参数可以只显示有错误的 feed

### 位置 3：订阅列表页 — 分类标题样式

**文件**：[subscription/index.phtml](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/views/subscription/index.phtml#L34-L36)

```php
<h2><span class="title<?= $cat->inError() ? ' error' : '' ?>"
    <?= $cat->inError() ? ' title="' . _t('sub.category.error') . '"' : '' ?>>
    <?=$cat->name() ?>
</span></h2>
```

**逻辑**：分类下只要有 **任意一个** feed 处于错误状态，分类标题就会显示红色。

### 位置 4：全局侧边栏

**文件**：[index/global.phtml](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/views/index/global.phtml#L75-L78)

```php
if ($feed->inError() && !$feed->mute()) {
    $error_class = ' error';
    $error_title = _t('sub.feed.error');
}
```

**逻辑**：与订阅列表页相同，但 **mute 的 feed 即使有错也不显示红色**（因为 mute 表示用户主动忽略，不需要视觉警告）。

### 位置 5：添加 feed 时的即时反馈

**文件**：[feedController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Controllers/feedController.php#L319-L323)

```php
} catch (FreshRSS_Feed_Exception $e) {
    Minz_Log::warning($e->getMessage());
    Minz_Request::bad(
        _t('feedback.sub.feed.internal_problem', _url('index', 'logs')),  // ← 带日志链接
        $url_redirect
    );
}
```

**文案**（翻译自英文）：
> "The newsfeed could not be added. [Check FreshRSS logs] for details. You can try force adding by appending #force_feed to the URL."

这是**唯一**在用户操作过程中立即弹出的错误反馈（页面顶部的 flash message）。

---

## 七、错误文案翻译参考

**翻译文件位置**：[app/i18n/en/](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/i18n/en/)

| 翻译 key | 英文原文 | 出现位置 |
|---------|---------|---------|
| `gen.short.damn` | "Blast!" | feed 管理页 alert-head |
| `sub.feed.error` | "This feed has encountered a problem. If this situation persists, please verify that it is still reachable." | feed 管理页 + 列表 tooltip |
| `sub.feed.last-error-date` | "Last erroneous update `<time>`%1$s`</time>` (%2$s)." | feed 管理页 |
| `sub.category.error` | "This dynamic OPML category has encountered a problem..." | 分类标题 tooltip |
| `feedback.sub.feed.internal_problem` | "The newsfeed could not be added. [Check FreshRSS logs] for details..." | 添加 feed 失败时 |
| `feedback.sub.feed.invalid_url` | "Given URL was not a valid feed URL:" | URL 无效时 |

---

## 八、关键代码定位表

| 功能 | 文件 | 行号 |
|-----|------|------|
| 429/503 响应拦截 | [SimplePieFetch.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/SimplePieFetch.php) | L37-L57 |
| Retry-After 存储 | [httpUtil.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Utils/httpUtil.php) | L67-L88 |
| Retry-After 冷却期拦截 | [Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php) | L612-L615 |
| 错误文案多级兜底 | [Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php) | L634-L651 |
| catch 异常 + 记录错误 | [feedController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Controllers/feedController.php) | L583-L595 |
| 410 Gone 自动 mute | [feedController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Controllers/feedController.php) | L587-L592 |
| DB: updateLastError | [FeedDAO.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/FeedDAO.php) | L247-L254 |
| DB: updateLastUpdate 清零 | [FeedDAO.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/FeedDAO.php) | L230-L233 |
| Feed: inError/lastError | [Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php) | L373-L382 |
| View: feed 管理页 | [feed/update.phtml](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/views/helpers/feed/update.phtml) | L17-L34 |
| View: 订阅列表 | [subscription/index.phtml](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/views/subscription/index.phtml) | L34-L70 |
| View: 全局侧边栏 | [index/global.phtml](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/views/index/global.phtml) | L75-L78 |
