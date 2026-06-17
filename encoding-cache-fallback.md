# HTTP 拉取失败缓存回退 & 手动原始数据场景全解

本文档深入分析两条容易混淆的路径：
1. **HTTP 拉取失败但旧缓存还在**（容易被误归为普通缓存命中）
2. **FreshRSS 中手动原始数据 (`set_raw_data`) 的完整使用场景**

---

## 第一部分：HTTP 拉取失败但旧缓存还在

这条路径非常容易被误认为是"普通缓存命中"，但它们的错误行为完全不同。

### 一图流对比

```
普通缓存命中                      HTTP失败 + 旧缓存回退
─────────────────                  ──────────────────────
fetch_data() 返回 true             fetch_data() 返回 true
  ↓                                   ↓
init() 直接 return true             init() 直接 return true
  ↓                                   ↓
SimplePie.error = (空)             SimplePie.error = 失败原因✓
  ↓                                   ↓
FreshRSS 认为成功 ✓                FreshRSS 认为失败 ✗
  ↓                                   ↓
(不进入编码检测)                    (不进入编码检测)
```

> 💡 两条路径都**不进入候选编码构建**，但错误状态截然不同！

---

### 完整代码路径分析

#### 步骤 1：`fetch_data()` 中触发 HTTP 失败

在 `fetch_data()` 中有三种 HTTP 失败场景会走这条路径：

##### 场景 A：首次请求异常（带缓存校验头）

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L2087-L2101)

```php
try {
    $file = $this->get_http_client()->request(Client::METHOD_GET, $this->feed_url, $headers);
    $this->status_code = $file->get_status_code();
} catch (ClientException $th) {
    $this->check_modified = false;
    $this->status_code = 0;

    if ($this->force_cache_fallback) {   // FreshRSS 默认 false，见下文
        // ... 特殊的强制回退逻辑（FreshRSS 不会走这里）
        return true;
    }

    $failedFileReason = $th->getMessage();  // ← 记录失败原因
}
```

##### 场景 B：首次或重试请求异常

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L2178-L2185)

```php
try {
    $file = $this->get_http_client()->request(Client::METHOD_GET, $this->feed_url, $headers);
} catch (ClientException $th) {
    $this->error = $th->getMessage();       // ← 直接设置 error
    return !empty($this->data);             // ← 有缓存返回 true，无缓存返回 false
}
```

##### 场景 C：HTTP 状态码不支持

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L2191-L2194)

```php
if (!(... status code 200 or 2xx ...)) {
    $this->error = 'Retrieved unsupported status code "' . $this->status_code . '"';
    return !empty($this->data);             // ← 同样的模式
}
```

##### 场景 D：首次请求失败后不再重试

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L2168-L2174)

```php
} elseif (isset($failedFileReason)) {
    $this->error = $failedFileReason;       // ← 设置 error
    return !empty($this->data);             // ← 有缓存返回 true
}
```

#### 步骤 2：`init()` 接收返回值

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1891-L1897)

```php
if (($fetched = $this->fetch_data($cache)) === true) {
    return true;                            // ← 直接返回，跳过编码检测
} elseif ($fetched === false) {
    return false;
}
```

**关键点**：
- `fetch_data()` 返回 `true`（因为 `!empty($this->data)` 为 true）
- `init()` 直接 `return true`
- **完全跳过候选编码构建**（L1908-L1945 的编码逻辑全部不执行）
- 但 `$this->error` 已经被设置了！

#### 步骤 3：FreshRSS 层的判断

相关文件：[Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L633-L639)

```php
if ($simplePieResult === false
    || $simplePie->get_hash() === ''
    || !empty($simplePie->error())) {      // ← 这个条件会命中！
    $error = 'Error during the last parse of the feed! ' . $this->url(includeCredentials: false)
        . ' : ' . ($simplePie->error() ?: 'No error message provided.');
    throw new FreshRSS_Feed_Exception($error);
}
```

**三重判断的分析**：

| 条件 | 值 | 说明 |
|-----|---|------|
| `$simplePieResult === false` | ❌ false | `init()` 返回了 true |
| `$simplePie->get_hash() === ''` | ❌ false | 缓存中有旧的 hash |
| `!empty($simplePie->error())` | ✅ true | **命中！** error 已被设置为失败原因 |

**结果**：FreshRSS 认为刷新失败，抛出 `FreshRSS_Feed_Exception`。

#### 步骤 4：最终效果

相关文件：[feedController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Controllers/feedController.php#L583-L595)

```php
} catch (FreshRSS_Feed_Exception $e) {
    Minz_Log::warning($e->getMessage());
    $feedDAO->updateLastError($feed->id());  // ← 数据库 error 更新为当前时间戳
    $feed->_error(time());                   // ← 内存对象 error 更新
    // ...
}
```

**用户看到的效果**：
- 页面显示 feed 有错误（红色提示框）
- 旧内容仍然能显示（因为数据库里有之前解析的条目）
- 错误时间戳被更新

---

### 与普通缓存命中的对比

| 对比项 | 普通缓存命中 | HTTP失败+旧缓存回退 |
|-------|------------|------------------|
| `fetch_data()` 返回值 | `true` | `true` |
| `init()` 返回值 | `true` | `true` |
| `SimplePie.error` | 空 | 有值（失败原因） |
| 是否进入编码检测 | ❌ 不进入 | ❌ 不进入 |
| FreshRSS 判断结果 | 成功 ✅ | 失败 ✗ |
| 数据库 error 字段 | 不变 | 更新为当前时间戳 |
| 页面显示 | 正常 | 红色错误提示 |
| 内容是否可用 | 可用 | 可用（旧内容） |

---

### 关于 `force_cache_fallback`

`force_cache_fallback` 是 SimplePie 的一个已废弃（deprecated）特性：

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1034-L1037)

```php
public function force_cache_fallback(bool $enable = false)
{
    $this->force_cache_fallback = $enable;
}
```

**FreshRSS 没有设置这个值**（`FreshRSS_SimplePieCustom` 构造函数中没有调用 `force_cache_fallback(true)`），所以默认是 `false`，不会走特殊的强制回退逻辑。

即使默认是 `false`，`return !empty($this->data)` 这个模式本身就已经实现了"有缓存就返回 true"的效果。`force_cache_fallback` 只是在请求异常时多做了一件事：更新缓存过期时间，避免下次刷新又去请求服务器。

---

## 第二部分：FreshRSS 使用手动原始数据的完整场景

之前文档提到了 3 种场景，实际上 FreshRSS 中有 **6 种场景** 使用 `set_raw_data()`。

### 总览

```
set_raw_data() 调用位置
├── Feed.php::simplePieFromContent()   ← 内部工具方法，被 5 种 feed 类型调用
│   ├── KIND_HTML_XPATH                  HTML + XPath 网页抓取
│   ├── KIND_XML_XPATH                   XML + XPath 解析
│   ├── KIND_JSON_DOTNOTATION            JSON + 点号路径
│   ├── KIND_JSONFEED                    标准 JSON Feed
│   └── KIND_HTML_XPATH_JSON_DOTNOTATION HTML + XPath 提取 JSON + 点号路径
│
└── p/api/pshb.php                      WebSub 实时推送接口
```

---

### 场景 1-5：通过 `simplePieFromContent()` 调用

相关文件：[Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L949-L955)

```php
private function simplePieFromContent(string $feedContent): FreshRSS_SimplePieCustom {
    $simplePie = new FreshRSS_SimplePieCustom();
    $simplePie->enable_cache(false);       // ← 关闭 SimplePie 缓存
    $simplePie->set_raw_data($feedContent); // ← 设置原始数据
    $simplePie->init();                    // ← 开始解析
    return $simplePie;
}
```

**特点**：
- 禁用 SimplePie 缓存（FreshRSS 有自己的缓存机制）
- 不设置 `feed_url`，直接跳过 `fetch_data()`
- 编码检测只有兜底部分（详见 `encoding-entry-paths.md`）

#### 各 feed 类型的数据流

##### 1. KIND_HTML_XPATH — HTML + XPath 网页抓取

相关文件：[Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L1057-L1209)

```
HTTP GET → HTML 页面
    ↓
DOMDocument + XPath 提取内容（标题、链接、正文等）
    ↓
渲染 index/rss.phtml 模板 → 生成 RSS XML
    ↓
simplePieFromContent() → set_raw_data()
```

##### 2. KIND_XML_XPATH — XML + XPath 解析

与 HTML XPath 相同，只是 HTTP Accept 头不同：
- `httpAccept = 'xml'` 而不是 `'html'`
- 同样通过 XPath 提取，渲染 RSS，走 `simplePieFromContent()`

##### 3. KIND_JSON_DOTNOTATION — JSON + 点号路径

相关文件：[Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L1018-L1054)

```
HTTP GET → JSON 数据
    ↓
json_decode() 解析
    ↓
FreshRSS_dotNotation_Util::convertJsonToRss() → 生成 RSS XML
    ↓
simplePieFromContent() → set_raw_data()
```

##### 4. KIND_JSONFEED — 标准 JSON Feed

与上一条相同，只是使用内置的点号路径配置：

相关文件：[Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L1048-L1048)

```php
$dotnotations = $this->kind() === FreshRSS_Feed::KIND_JSONFEED
    ? $this->dotNotationForStandardJsonFeed()   // ← 内置 JSON Feed 标准映射
    : $json_dotnotation;
```

##### 5. KIND_HTML_XPATH_JSON_DOTNOTATION — HTML + XPath + JSON

两步提取：
```
HTTP GET → HTML 页面
    ↓
extractJsonFromHtml() → XPath 提取 JSON 字符串
    ↓
json_decode() + convertJsonToRss() → 生成 RSS XML
    ↓
simplePieFromContent() → set_raw_data()
```

---

### 场景 6：WebSub (PubSubHubbub) 实时推送

相关文件：[pshb.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/p/api/pshb.php#L101-L104)

```php
$simplePie = new FreshRSS_SimplePieCustom();
$simplePie->enable_cache(false);
$simplePie->set_raw_data($ORIGINAL_INPUT);  // ← POST 请求体（原始 XML）
$simplePie->init();
```

**数据流**：

```
WebSub Hub → HTTP POST 推送 XML
    ↓
php://input → $ORIGINAL_INPUT
    ↓
set_raw_data() → SimplePie 解析
    ↓
actualizeFeedsAndCommit(simplePiePush: $simplePie) → 更新 feed
```

**特点**：
- 数据直接来自 HTTP POST body，不是主动拉取
- 同样禁用 SimplePie 缓存
- 编码检测只有兜底部分

---

### 手动原始数据场景的编码检测特点

所有 6 种场景都有相同的编码检测模式：

```
候选编码 = [
  ① xml_encoding() 结果    ← 兜底
  ② UTF-8                  ← 兜底
  ③ ISO-8859-1             ← 兜底
]
```

**没有的部分**：
- ❌ 没有 `$sniffed`（MIME 分支全部跳过）
- ❌ 没有 HTTP Content-Type charset
- ❌ 没有 RFC 3023 默认编码

**为什么没问题？**

| 场景 | 编码保证 |
|-----|---------|
| KIND_HTML_XPATH / XML_XPATH | `index/rss.phtml` 模板明确输出 `<?xml version="1.0" encoding="UTF-8"?>` |
| JSON 系列 | `convertJsonToRss()` 生成的 RSS 是 UTF-8 |
| WebSub 推送 | WebSub 规范要求 feed 本身应该带正确的 XML 声明，xml_encoding() 能识别 |

> 💡 这些场景下数据是 FreshRSS 自己生成的（UTF-8），或者是规范要求带编码声明的（WebSub），所以即使只有兜底编码也足够。

---

## 关键代码定位

### HTTP 失败回退路径

- L2087-L2101：场景 A，请求异常（带缓存校验头）
- L2168-L2174：场景 D，首次失败后不再重试
- L2178-L2185：场景 B，请求异常（首次或重试）
- L2191-L2194：场景 C，不支持的 HTTP 状态码
- L1891-L1897：`init()` 接收返回值

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php)

### FreshRSS 错误判断

- L633-L639：三重失败判断（`!empty($simplePie->error())` 是关键）

相关文件：[Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php)

### 手动原始数据调用位置

- [Feed.php L949-L955](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L949-L955)：`simplePieFromContent()`
- [Feed.php L1018-L1054](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L1018-L1054)：`loadJson()`（3 种 JSON 场景）
- [Feed.php L1057-L1209](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L1057-L1209)：`loadHtmlXpath()`（2 种 XPath 场景）
- [pshb.php L101-L104](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/p/api/pshb.php#L101-L104)：WebSub 推送

---

## 常见误区澄清

### ❌ 误区 1：`fetch_data()` 返回 `true` 就一定是成功

**真相**：返回 `true` 只表示"有可用数据"（可能是新数据也可能是旧缓存）。具体要看 `SimplePie.error` 是否为空。

| `fetch_data()` 返回 | `error` 为空 | 含义 |
|-------------------|-------------|------|
| `true` | ✅ 是 | 真正的缓存命中（内容未过期、304、或哈希未变） |
| `true` | ❌ 否 | HTTP 失败但有旧缓存（错误路径） |
| `false` | ❌ 否 | HTTP 失败且无缓存（完全失败） |
| `[$headers, $sniffed]` | - | 成功获取新数据，进入编码检测 |

### ❌ 误区 2：`init()` 返回 `true` 就不会抛出异常

**真相**：`FreshRSS_Feed::load()` 有三重判断，即使 `init()` 返回 `true`，只要 `error()` 非空，仍然会抛出异常。

### ❌ 误区 3：FreshRSS 只有 3 种场景用 `set_raw_data()`

**真相**：有 6 种，WebSub 推送接口是独立的一条路径，和 `simplePieFromContent()` 中的 5 种 feed 类型是并列关系。

### ✅ 正确理解：两条"不进入编码检测"的路径

所有不进入候选编码构建的路径汇总：

| 路径 | `init()` 返回 | `error` | FreshRSS 判断 |
|-----|--------------|---------|--------------|
| 普通缓存命中（未过期） | `true` | 空 | 成功 |
| 普通缓存命中（304） | `true` | 空 | 成功 |
| 普通缓存命中（哈希未变） | `true` | 空 | 成功 |
| **HTTP失败 + 旧缓存** | `true` | **有值** | **失败** |

前 3 条是真正的成功，第 4 条是"表面成功实际失败"的特殊路径。
