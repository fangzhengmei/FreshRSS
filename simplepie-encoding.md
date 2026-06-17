# FreshRSS Feed 编码识别与转码流程详解

## 概述

FreshRSS 使用 SimplePie 库作为 Feed 解析引擎，并在其基础上进行了自定义扩展。编码识别和转码是 Feed 处理中最复杂的部分之一，整个流程可以分为四个阶段：

1. **HTTP 数据获取** — 下载 Feed 内容
2. **编码探测** — 多维度识别源数据编码
3. **转码与解析** — 尝试将数据转为 UTF-8 并解析 XML
4. **兜底与错误处理** — 转码/解析失败时的降级策略

---

## 一、HTTP 数据获取阶段

### 1.1 FreshRSS 自定义 Fetch 类

FreshRSS 通过继承 SimplePie 的 `File` 类，实现了自定义的 HTTP 获取逻辑。

相关文件：
- [SimplePieCustom.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/SimplePieCustom.php) — 注册自定义 File 类
- [SimplePieFetch.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/app/Models/SimplePieFetch.php) — 自定义 HTTP 获取

核心代码（SimplePieCustom 构造函数中注册）：

```php
$this->get_registry()->register(\SimplePie\File::class, FreshRSS_SimplePieFetch::class);
```

### 1.2 File 类的预处理

在 `File` 类构造函数的最后，对 body 做了一次前导空白清理：

相关文件：[File.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/File.php#L303-L309)

```php
if ($this->success) {
    // Leading whitespace may cause XML parsing errors
    $this->body = preg_replace('/^[ \n\r\t\v]+</', '<', $this->body);
}
```

**注意**：这里只 trim 前导空白字符（空格、换行、回车、制表符、垂直制表符），**不包括** `\x00`（null 字节），因为 UTF-16/UTF-32 编码中 null 字节是正常字符的一部分。

---

## 二、编码探测阶段

编码探测发生在 `SimplePie::init()` 方法中，核心逻辑是构建一个**候选编码列表**，然后按优先级逐个尝试转码。

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1908-L1945)

### 2.1 候选编码列表的构建顺序

候选编码按以下优先级添加到列表中（优先级从高到低）：

| 优先级 | 编码来源 | 适用条件 | 代码位置 |
|--------|----------|----------|----------|
| 1 | 用户强制指定 `input_encoding` | 调用 `set_input_encoding()` 时 | [SimplePie.php#L1912-L1914](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1912-L1914) |
| 2 | HTTP Content-Type 头中的 charset | `application/*+xml` 类型 | [SimplePie.php#L1922-L1923](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1922-L1923) |
| 3 | BOM + XML 声明探测 | `application/*+xml` 类型 | [SimplePie.php#L1925](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1925) |
| 4 | `UTF-8` | `application/*+xml` 类型默认 | [SimplePie.php#L1926](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1926) |
| 5 | HTTP Content-Type 头中的 charset | `text/*+xml` 类型 | [SimplePie.php#L1928-L1929](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1928-L1929) |
| 6 | `US-ASCII` | `text/*+xml` 类型默认 | [SimplePie.php#L1931](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1931) |
| 7 | `UTF-8` | `text/*` 类型 | [SimplePie.php#L1935](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1935) |
| 8 | BOM + XML 声明探测 (兜底) | 所有情况 | [SimplePie.php#L1940](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1940) |
| 9 | `UTF-8` (最终兜底) | 所有情况 | [SimplePie.php#L1941](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1941) |
| 10 | `ISO-8859-1` (最终兜底) | 所有情况 | [SimplePie.php#L1942](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1942) |

> **设计要点**：RFC 3023 规定了 XML 媒体类型的默认编码：
> - `application/xml` 及 `application/*+xml` → 默认 UTF-8
> - `text/xml` 及 `text/*+xml` → 默认 US-ASCII
>
> 但 SimplePie 还会额外追加 BOM/XML 声明探测结果和 UTF-8/ISO-8859-1 兜底。

### 2.2 BOM 检测与 XML 声明解析

`Misc::xml_encoding()` 方法实现了基于 BOM（Byte Order Mark）和 XML 声明的编码探测，遵循 XML 1.0 Appendix F.1 规范。

相关文件：[Misc.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/Misc.php#L2011-L2088)

#### BOM 检测顺序（按字节序列匹配）

```
BOM 字节序列            → 编码
───────────────────────────────────
\x00\x00\xFE\xFF        → UTF-32BE
\xFF\xFE\x00\x00        → UTF-32LE
\xFE\xFF                → UTF-16BE
\xFF\xFE                → UTF-16LE
\xEF\xBB\xBF            → UTF-8
```

检测逻辑是 **if-elseif 链**，找到第一个匹配的 BOM 就立即返回。

#### 无 BOM 时的启发式探测

如果没有 BOM，则通过查找 `<?xml` 开头的模式来推断编码：

| 模式（前 N 字节） | 推断编码 | 说明 |
|-------------------|----------|------|
| `\x00\x00\x00\x3C\x00\x00\x00\x3F...` (20字节) | UTF-32BE | 每字符4字节，大端 |
| `\x3C\x00\x00\x00\x3F\x00\x00\x00...` (20字节) | UTF-32LE | 每字符4字节，小端 |
| `\x00\x3C\x00\x3F\x00\x78...` (10字节) | UTF-16BE | 每字符2字节，大端 |
| `\x3C\x00\x3F\x00\x78\x00...` (10字节) | UTF-16LE | 每字符2字节，小端 |
| `\x3C\x3F\x78\x6D\x6C` (5字节 = `<?xml`) | UTF-8 (或其超集) | ASCII 兼容编码 |

#### XML 声明中的 encoding 属性

对于有 `<?xml ... ?>` 声明的情况，SimplePie 会：
1. 先按推断的编码将 XML 声明部分转换为 UTF-8
2. 用 `XML\Declaration\Parser` 解析声明中的 `encoding` 属性
3. 将解析出的编码名**追加**到候选列表中（优先级高于该编码的默认值）

相关文件：[XML/Declaration/Parser.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/XML/Declaration/Parser.php)

该解析器是一个**有限状态机**，按顺序解析：
- `version` 属性（必须有）
- `encoding` 属性（可选）
- `standalone` 属性（可选）

#### fallback 兜底

如果以上都没匹配到，返回 `['UTF-8']` 作为最终兜底。

---

## 三、转码与 XML 解析阶段

### 3.1 转码尝试循环

在 `SimplePie::init()` 中，候选编码列表构建完成后，会**按顺序逐个尝试**转码：

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1947-L1979)

```php
foreach ($encodings as $encoding) {
    if ($utf8_data = $this->registry->call(Misc::class, 'change_encoding', [$this->raw_data, $encoding, 'UTF-8'])) {
        $parser = $this->registry->create(Parser::class);
        if ($parser->parse($utf8_data, 'UTF-8', $this->permanent_url ?? '')) {
            // 解析成功，直接返回
            return true;
        }
    }
}
```

**关键点**：
- 转码成功但 XML 解析失败 → 继续尝试下一个编码
- 转码失败 → 直接跳过该编码
- 第一个"转码成功且解析成功"的编码就是最终使用的编码

### 3.2 编码转换实现

`Misc::change_encoding()` 方法按优先级尝试多种转码方式：

相关文件：[Misc.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/Misc.php#L293-L328)

| 优先级 | 转码方式 | 适用条件 | 说明 |
|--------|----------|----------|------|
| 1 | Windows-1252 → UTF-8 查表法 | 输入是 windows-1252 且输出是 UTF-8 | 完全可预测，速度最快 |
| 2 | mbstring (`mb_convert_encoding`) | mbstring 扩展可用 | 行为随 PHP 版本变化 |
| 3 | iconv (`iconv()`) | iconv 扩展可用 | 行为随操作系统变化 |
| 4 | intl (`UConverter::transcode`) | intl 扩展可用 | PHP 5.5+ |

都失败时返回 `false`。

#### US-ASCII 的特殊处理

如果输入编码是 `US-ASCII`，会先截断所有高位字节（0x80-0xFF），因为严格的 ASCII 不包含这些字符。

相关文件：[Misc.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/Misc.php#L299-L307)

### 3.3 XML 解析前的 BOM 处理与声明重写

即使数据已经转成 UTF-8，`Parser::parse()` 还会再做一次 BOM 剥离和 XML 声明重写。

相关文件：[Parser.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/Parser.php#L89-L123)

#### BOM 剥离

```php
// 按 UTF-32BE → UTF-32LE → UTF-16BE → UTF-16LE → UTF-8 顺序检查
// 找到 BOM 就截断对应长度的前缀
```

**为什么转成 UTF-8 后还要剥离 BOM？**
因为转码函数（如 mb_convert_encoding）**可能会保留 BOM**，也可能会把源编码的 BOM 转成目标编码的 BOM。XML 解析器对 BOM 的处理不一致，所以 SimplePie 主动剥离。

#### XML 声明重写

如果数据以 `<?xml ` 开头，SimplePie 会：
1. 解析原始 XML 声明
2. 剥离原始声明
3. 用**实际使用的编码**重写声明

```php
$data = '<?xml version="' . $declaration->version . '" encoding="' . $encoding . '" standalone="'.($declaration->standalone?'yes':'no').'?>' . "\n" .
    self::set_doctype($data);
```

**目的**：确保 XML 解析器使用的编码与声明中的编码一致，避免解析器自行探测出错。

### 3.4 XML 解析器创建

使用 `xml_parser_create_ns()` 创建 XML 解析器，编码参数为转换后的 UTF-8：

相关文件：[Parser.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/Parser.php#L138-L139)

```php
$xml = xml_parser_create_ns($this->encoding, $this->separator);
```

---

## 四、转码失败与兜底显示

### 4.1 SimplePie 层面的错误

当所有候选编码都尝试过且都失败时，分两种情况：

相关文件：[SimplePie.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/simplepie/simplepie/src/SimplePie.php#L1981-L2006)

#### 情况 1：有解析器实例（转码成功但 XML 解析失败）

错误信息：
```
{url} is invalid XML, likely due to invalid characters. XML error: {error} at line {line}, column {column}
```

这种情况通常是：
- 编码猜对了，但 XML 内容本身有语法错误
- 编码猜错了，导致乱码进而 XML 解析失败

#### 情况 2：没有解析器实例（转码全部失败）

错误信息：
```
The data could not be converted to UTF-8. You MUST have either the iconv, mbstring or intl extension installed and enabled.
```

如果缺少转码扩展，会追加提示安装哪个扩展。

### 4.2 FreshRSS 层面的编码安全

FreshRSS 在 `lib_rss.php` 中提供了 `safe_utf8()` 函数，用于确保字符串是有效的 UTF-8：

相关文件：[lib_rss.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/lib_rss.php#L152-L164)

```php
// 优先级：mbstring > iconv > 原样返回
if (function_exists('mb_convert_encoding')) {
    function safe_utf8(?string $text): string {
        return $text === null ? '' : (mb_convert_encoding($text, 'UTF-8', 'UTF-8') ?: '');
    }
} elseif (function_exists('iconv')) {
    function safe_utf8(?string $text): string {
        return $text === null ? '' : (iconv('UTF-8', 'UTF-8//IGNORE', $text) ?: '');
    }
} else {
    function safe_utf8(?string $text): string {
        return $text ?? '';
    }
}
```

**原理**：
- `mb_convert_encoding($text, 'UTF-8', 'UTF-8')` — 看似自相矛盾，实际作用是**验证并修正** UTF-8 序列，遇到无效字节会被替换或丢弃
- `iconv('UTF-8', 'UTF-8//IGNORE', $text)` — `//IGNORE` 标志会静默丢弃无法转换的字符
- 都没有时直接返回原字符串（最宽松的兜底）

### 4.3 Feed 订阅时的错误显示

当 Feed 刷新失败时，错误信息会通过 FreshRSS 的错误处理机制展示给用户。错误标题和详情会被 `htmlspecialchars()` 转义后显示在 HTML 页面中。

相关文件：[lib_rss.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-FreshRSS/lib/lib_rss.php#L389-L419)

---

## 五、整体流程图

```
HTTP 响应获取 (SimplePie_File / FreshRSS_SimplePieFetch)
    │
    ├─ gzip/deflate 解压
    └─ 前导空白 trim（保留 null 字节）
           │
           ▼
Content-Type 嗅探 (Content\Type\Sniffer)
    │
    ├─ 检查 BOM 判断文本/二进制
    ├─ 检查 HTML/Feed 特征标记
    └─ 返回 sniffed 类型
           │
           ▼
构建候选编码列表 (SimplePie::init)
    │
    ├─ 1. 用户指定 input_encoding
    ├─ 2. HTTP Content-Type charset
    ├─ 3. BOM + XML 声明探测 (xml_encoding)
    ├─ 4. RFC 3023 默认编码 (UTF-8 / US-ASCII)
    ├─ 5. 再次 BOM + XML 声明探测（兜底）
    ├─ 6. UTF-8（最终兜底）
    └─ 7. ISO-8859-1（最终兜底）
           │
           ▼
逐编码尝试 (foreach $encodings)
    │
    ├─ change_encoding(数据, 候选编码, UTF-8)
    │   ├─ Windows-1252 查表（最快）
    │   ├─ mbstring
    │   ├─ iconv
    │   └─ UConverter
    │
    └─ 转码成功 → Parser::parse()
        ├─ 再次剥离 BOM
        ├─ 重写 XML 声明 encoding
        └─ xml_parser_create_ns 解析
            │
            ├─ 解析成功 → 返回数据 ✓
            └─ 解析失败 → 继续下一编码 ✗
                  │
                  ▼
所有编码都失败
    │
    ├─ 有解析错误 → XML 错误信息
    └─ 无解析错误 → 转码扩展缺失提示
```

---

## 六、关键设计要点

### 6.1 为什么要做这么多层探测？

1. **HTTP 头不可信** — 很多服务器配置错误，Content-Type 中的 charset 可能不对
2. **BOM 可能缺失** — 很多 UTF-8 文件不带 BOM
3. **XML 声明可能缺失** — 不是所有 XML 都有声明
4. **编码兼容性** — UTF-8 是 ASCII 的超集，很多声明为 US-ASCII 的实际是 UTF-8

### 6.2 为什么转码成功后还要验证 XML 解析？

转码成功只意味着"从 A 编码转到了 UTF-8"，但不代表 A 编码就是正确的源编码。错误的编码也可能转码成功（只是内容乱码），但乱码的 XML 通常解析失败。

### 6.3 为什么 Parser 里还要再剥一次 BOM？

因为不同的转码函数对 BOM 的处理不一致：
- 有的会保留 BOM
- 有的会把源 BOM 转成目标编码的 BOM
- 有的会直接丢弃 BOM

SimplePie 选择主动剥离，确保 XML 解析器行为一致。

### 6.4 为什么最终兜底是 ISO-8859-1？

ISO-8859-1（Latin-1）是单字节编码，**任何字节序列都是合法的 Latin-1**。所以用它转码永远不会失败，是最后的安全网。代价是：如果源编码不是 Latin-1，内容会乱码，但至少不会报错。
