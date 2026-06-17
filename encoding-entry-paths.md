# Feed 编码检测的三条入口路径对比分析

本文档从代码实现角度分析 SimplePie / FreshRSS 的三条数据入口路径：
1. **缓存命中**（Cache Hit）
2. **手动原始数据**（`set_raw_data()`）
3. **HTTP 拉取**（HTTP Fetch）

重点对比它们是否会进入候选编码构建，以及候选编码的顺序差异。

---

## 一图流总结

```
                    ┌─────────────────────┐
                    │   SimplePie::init()  │
                    └─────────┬───────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
    feed_url 不为空?    raw_data 已设置?   都没有 → 返回 false
            │                 │
            ▼                 ▼
    ┌──────────────┐   ┌───────────────┐
    │ fetch_data() │   │  直接进入编码  │
    └──────┬───────┘   │  检测阶段      │
           │           └───────┬───────┘
           │                   │
     返回 true?          $sniffed = 未定义
     (缓存命中)         $headers = 未定义
           │
           ▼
    return true
  (不进入编码检测)
```

---

## 路径 1：缓存命中（Cache Hit）

### 入口条件

在 `fetch_data()` 方法中，当检测到有效缓存时，**直接返回 `true`**，完全跳过编码检测阶段。

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L2036-L2156)

### 缓存命中的三种情况

| 情况 | 触发条件 | 说明 |
|-----|---------|------|
| 缓存未过期 | `cache_expiration_time >= time()` | 最常见的情况 |
| HTTP 304 | 服务器返回 304 Not Modified | 缓存仍然有效 |
| 哈希未变 | 内容哈希与缓存相同 | 服务器没返回 304 但内容没变 |

**代码证据**（缓存未过期的情况）：

```php
// SimplePie.php L2152-L2156
else {
    $this->raw_data = false;
    return true;
}
```

### 是否进入候选编码构建？

**❌ 完全不进入。**

- `$this->data` 直接从缓存加载（已经是解析好的结构化数据）
- `$this->raw_data` 被设为 `false`（表示"缓存仍有效"的标记）
- 直接 `return true`，完全跳过编码检测和 XML 解析阶段
- **不会触发任何转码操作**

### 缓存中存了什么？

缓存中存储的是**解析后的结构化数据**，不是原始 XML：

```php
// SimplePie.php L2273-L2281
$this->data = [
    'url' => $this->feed_url,
    'feed_url' => $file->get_final_requested_uri(),
    'build' => Misc::get_build(),
    'cache_expiration_time' => ...,
    'cache_version' => self::CACHE_VERSION,
    'hash' => $this->clean_hash($file->get_body_content()),
    // ... 还有解析后的条目数据
];
```

---

## 路径 2：手动原始数据（set_raw_data）

### 入口条件

通过 `set_raw_data($data)` 直接设置原始数据，且 **没有设置 `feed_url`**。

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L871-L874)

```php
public function set_raw_data(string $data)
{
    $this->raw_data = $data;
}
```

### 代码中的判断

在 `init()` 方法中：

```php
// SimplePie.php L1883-L1899
if ($this->feed_url !== null) {
    // ... 走 HTTP 拉取路径 ...
    // 只有这里会定义 $sniffed 和 $headers
    [$headers, $sniffed] = $fetched;
}
// 如果 feed_url 为 null 且 raw_data 不为 null，直接跳到编码检测阶段
```

### 是否进入候选编码构建？

**✅ 进入，但只有兜底编码部分。**

关键变量状态：
- `$sniffed` = **未定义**（没有走 `fetch_data()`）
- `$headers` = **未定义**
- `$this->raw_data` = 用户设置的数据

### 候选编码顺序

```
候选编码 = [
  ① input_encoding（如果设置了）
  ② xml_encoding() 结果      ← 兜底部分开始
  ③ UTF-8
  ④ ISO-8859-1
]
```

**没有的部分**：
- ❌ 没有 HTTP Content-Type 的 charset
- ❌ 没有 MIME 类型分支（分支 A/B/C/D 都跳过）
- ❌ 没有 RFC 3023 默认编码

### 为什么走不了 MIME 分支？

MIME 分支的入口条件是 `isset($sniffed)`，而 `$sniffed` 是 `fetch_data()` 通过 Sniffer 嗅探得到的。手动设置 raw_data 不走 `fetch_data()`，所以 `$sniffed` 未定义，所有 MIME 分支都跳过。

```php
// SimplePie.php L1919-L1937
if (isset($sniffed)) {   // ← 这个条件不满足
    // 分支 A：application/*+xml
    // 分支 B：text/*+xml
    // 分支 C：其他 text/*
    // 分支 D：非 text 类型
}
// 直接跳到兜底部分
```

### FreshRSS 中的使用场景

FreshRSS 在以下情况使用 `set_raw_data()`：

1. **JSON feed 转 RSS**：JSON 数据转换为 RSS XML 后，用 `set_raw_data()` 喂给 SimplePie 解析
2. **HTML XPath 转 RSS**：HTML 页面通过 XPath 提取内容生成 RSS 后，用 `set_raw_data()` 解析
3. **JSON dot notation 转 RSS**：类似 JSON feed，但用自定义点号路径

相关文件：[Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L949-L955)

```php
private function simplePieFromContent(string $feedContent): FreshRSS_SimplePieCustom {
    $simplePie = new FreshRSS_SimplePieCustom();
    $simplePie->enable_cache(false);
    $simplePie->set_raw_data($feedContent);
    $simplePie->init();
    return $simplePie;
}
```

> 💡 由于这些场景下数据是 FreshRSS 自己生成的，编码通常是已知的 UTF-8，所以即使没有 MIME 分支也问题不大。

---

## 路径 3：HTTP 拉取（HTTP Fetch）

### 入口条件

设置了 `feed_url`，且 `fetch_data()` 成功获取了新数据（不是从缓存加载）。

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L2292-L2304)

```php
$this->raw_data = $file->get_body_content();
// ...
$sniffer = $this->registry->create(Sniffer::class, [$fileResponse]);
$sniffed = $sniffer->get_type();

return [$headers, $sniffed];
```

### 是否进入候选编码构建？

**✅ 完整进入，所有分支都可能执行。**

关键变量状态：
- `$sniffed` = **已定义**（Content-Type Sniffer 嗅探结果）
- `$headers` = **已定义**（HTTP 响应头数组）
- `$this->raw_data` = HTTP 响应体

### 候选编码完整顺序

```
候选编码 = [
  ① input_encoding（如果设置了）
  ──────────────────────────────────
  ② HTTP charset（如果有，分支 A/B）
  ③ xml_encoding() 结果（分支 A 才有）
  ④ RFC 3023 默认编码（分支 A → UTF-8，分支 B → US-ASCII）
  ⑤ 其他 text/* 类型 → UTF-8（分支 C）
  ──────────────────────────────────  兜底部分开始
  ⑥ xml_encoding() 结果（再次调用）
  ⑦ UTF-8
  ⑧ ISO-8859-1
]
```

### 不同 MIME 类型下的候选列表示例

#### 示例 A：application/rss+xml + UTF-8 BOM

HTTP 头：`Content-Type: application/rss+xml; charset=ISO-8859-1`
文件开头有 UTF-8 BOM

| 步骤 | 来源 | 编码 |
|-----|------|------|
| 1 | input_encoding（无） | - |
| 2 | HTTP charset（分支 A） | ISO-8859-1 |
| 3 | xml_encoding()（分支 A） | UTF-8 |
| 4 | RFC 3023 默认（分支 A） | UTF-8（去重） |
| 5 | 兜底 xml_encoding() | UTF-8（去重） |
| 6 | 兜底 UTF-8 | UTF-8（去重） |
| 7 | 兜底 ISO-8859-1 | ISO-8859-1（去重） |

**最终顺序**：`ISO-8859-1 → UTF-8`

---

#### 示例 B：text/xml + XML 声明 GB2312

HTTP 头：`Content-Type: text/xml`（无 charset）
文件内容：`<?xml version="1.0" encoding="GB2312"?>`

但注意：FreshRSS 修改了 Sniffer，`text/xml` 会进入 `feed_or_html()` 检测，如果内容是 RSS/Atom，会被重分类为 `application/rss+xml` 或 `application/atom+xml`。

假设内容确实是 RSS，实际 sniffed 类型是 `application/rss+xml`（分支 A）：

| 步骤 | 来源 | 编码 |
|-----|------|------|
| 1 | input_encoding（无） | - |
| 2 | HTTP charset（分支 A，无） | - |
| 3 | xml_encoding()（分支 A） | GB2312 → UTF-8 |
| 4 | RFC 3023 默认（分支 A） | UTF-8（去重） |
| 5 | 兜底 xml_encoding() | GB2312 → UTF-8（去重） |
| 6 | 兜底 UTF-8 | UTF-8（去重） |
| 7 | 兜底 ISO-8859-1 | ISO-8859-1 |

**最终顺序**：`GB2312 → UTF-8 → ISO-8859-1`

如果内容不是 RSS/Atom，sniffed 类型保持 `text/xml`（分支 B）：

| 步骤 | 来源 | 编码 |
|-----|------|------|
| 1 | input_encoding（无） | - |
| 2 | HTTP charset（分支 B，无） | - |
| 3 | RFC 3023 默认（分支 B） | US-ASCII |
| 4 | 兜底 xml_encoding() | GB2312 → UTF-8 |
| 5 | 兜底 UTF-8 | UTF-8（去重） |
| 6 | 兜底 ISO-8859-1 | ISO-8859-1 |

**最终顺序**：`US-ASCII → GB2312 → UTF-8 → ISO-8859-1`

---

#### 示例 C：text/html（非 XML）

HTTP 头：`Content-Type: text/html; charset=UTF-8`
文件是普通 HTML 页面

| 步骤 | 来源 | 编码 |
|-----|------|------|
| 1 | input_encoding（无） | - |
| 2 | 分支 C（text/* 默认） | UTF-8 |
| 3 | 兜底 xml_encoding() | UTF-8（去重，因为没有 `<?xml` 声明，直接返回 UTF-8） |
| 4 | 兜底 UTF-8 | UTF-8（去重） |
| 5 | 兜底 ISO-8859-1 | ISO-8859-1 |

**最终顺序**：`UTF-8 → ISO-8859-1`

---

## 三条路径对比总结

| 对比项 | 缓存命中 | 手动 raw_data | HTTP 拉取 |
|-------|---------|--------------|----------|
| **进入编码检测？** | ❌ 不进入 | ✅ 进入（部分） | ✅ 完整进入 |
| **`$sniffed`** | - | 未定义 | 已定义 |
| **`$headers`** | - | 未定义 | 已定义 |
| **input_encoding** | - | ✅ | ✅ |
| **HTTP charset** | - | ❌ | ✅ |
| **MIME 分支** | - | ❌ | ✅ |
| **xml_encoding()** | - | ✅（兜底） | ✅（分支 A + 兜底） |
| **UTF-8 兜底** | - | ✅ | ✅ |
| **ISO-8859-1 兜底** | - | ✅ | ✅ |
| **FreshRSS 场景** | 正常刷新命中缓存 | JSON/HTML 转 RSS | 新数据拉取 |

---

## 关键代码定位

### 入口判断：`SimplePie::init()`

- L1873-L1875：判断是否有 feed_url 或 raw_data
- L1883-L1899：feed_url 路径 → 调用 fetch_data()
- L1901-L1906：空响应检查
- L1908-L1945：候选编码构建

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1873-L1945)

### 缓存命中返回：`SimplePie::fetch_data()`

- L2036-L2157：缓存检测与命中逻辑
- L2131、L2146、L2155：三处 `return true` 的缓存命中点

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L2036-L2157)

### HTTP 拉取返回：`SimplePie::fetch_data()`

- L2292-L2304：设置 raw_data、headers、sniffed
- 返回 `[$headers, $sniffed]` 数组

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L2292-L2304)

### 手动设置原始数据：`SimplePie::set_raw_data()`

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L871-L874)

### FreshRSS 中的使用：`FreshRSS_Feed::simplePieFromContent()`

相关文件：[Feed.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/Feed.php#L949-L955)

---

## 常见误区澄清

### ❌ 误区 1：缓存数据也要经过编码检测

**真相**：缓存里存的是已经解析好的结构化数据，直接用，完全不经过编码检测和 XML 解析。

### ❌ 误区 2：set_raw_data 也会有 MIME 类型检测

**真相**：set_raw_data 不走 `fetch_data()`，所以 `$sniffed` 未定义，MIME 分支全部跳过。只有兜底编码。

### ❌ 误区 3：HTTP 拉取一定走完整的编码检测

**真相**：如果拉取后发现内容哈希和缓存一样（`hash === hash`），也会直接返回 true，跳过编码检测。见 L2134-L2147。

### ✅ 正确理解：编码检测只发生在"有新的原始数据需要解析"时

无论是 HTTP 拉取到的新数据，还是手动设置的原始数据，只要是第一次解析，都会走编码检测。缓存命中意味着之前已经解析过了，直接用结果。
