# FreshRSS OPML 与数据导入导出代码分析

## 一、总体架构

### 核心模块

| 模块 | 文件 | 职责 |
|------|------|------|
| 控制器 | [importExportController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Controllers/importExportController.php) | Web 端请求处理、文件类型识别、导入导出调度 |
| 导入服务 | [ImportService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Services/ImportService.php) | OPML 解析、分类合并、订阅创建核心逻辑 |
| 导出服务 | [ExportService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Services/ExportService.php) | OPML 生成、文章导出、ZIP 打包 |
| OPML 解析库 | [LibOpml.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/lib/marienfressinaud/lib_opml/src/LibOpml/LibOpml.php) | OPML XML 与 PHP 数组互相转换 |
| 订阅 DAO | [FeedDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Models/FeedDAO.php) | 订阅增删改查、重复检测 |
| 分类 DAO | [CategoryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Models/CategoryDAO.php) | 分类增删改查、默认分类管理 |

### 入口点

**Web 端导入**：
- 页面：[importExport/index.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/views/importExport/index.phtml)
- 处理：`importExportController::importAction()` → `importFile()` → `ImportService::importOpml()`

**Web 端导出**：
- 处理：`importExportController::exportAction()` → `ExportService::generateOpml()`

**CLI 导入**：[cli/import-for-user.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/cli/import-for-user.php)

**CLI 导出**：[cli/export-opml-for-user.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/cli/export-opml-for-user.php)

---

## 二、导入流程详解

### 2.1 文件类型识别

`importExportController::guessFileType()` 方法基于文件名后缀识别导入文件类型：

| 文件特征 | 识别类型 | 说明 |
|----------|----------|------|
| `.zip` 后缀 | `zip` | 压缩包，需解压后处理内部文件 |
| `.txt` 后缀 | `txt` | 纯文本 URL 列表，需转换为 OPML |
| 文件名含 `opml` | `opml` | OPML 订阅列表 |
| `.json` 后缀 + 含 `starred` | `json_starred` | 星标文章 |
| `.json` 后缀 | `json_feed` | 订阅文章数据 |
| `.xml` + 含 `Tiny`/`tt-rss` | `ttrss_starred` | TT-RSS 星标文章 |
| `.xml` 后缀 | `opml` | 默认按 OPML 处理 |

导入顺序：OPML → 星标文章 → 订阅文章 → TT-RSS 文章，确保分类和订阅先于文章数据导入。

### 2.2 TXT 文件转 OPML

`importExportController::txtToOpml()` 处理纯文本 URL 列表：

- 移除 UTF-8 BOM 头
- 按行分割，跳过：空行、`#` 开头注释、`<` 开头的 HTML 标签行
- 使用 `filter_var($url, FILTER_VALIDATE_URL)` 验证 URL 有效性，无效则记录警告
- 有效 URL 包装为最小化的 OPML 结构，交由标准 OPML 管道处理

### 2.3 OPML 解析

`ImportService::importOpml()` 方法为 OPML 导入主入口：

1. 使用 `marienfressinaud\LibOpml\LibOpml` 库解析，**`strict: false` 模式**，容错性强
2. 检查并确保默认分类存在
3. 预加载现有分类列表，建立 `categories_by_names` 名称映射
4. 加载数量限制配置
5. 调用 `loadFromOutlines()` 递归解析 OPML 结构
6. 逐分类创建 / 合并分类与订阅

#### OPML 结构递归解析

`loadFromOutlines()` + `loadFromOutline()` 递归遍历 outline 树：

- **分类节点**（含 `@outlines` 子节点）：以 `text` 属性为分类名，递归处理子节点
- **订阅节点**（含 `xmlUrl` 属性）：归入父分类名下
- **category 属性**：无父分类且 outline 有 `category` 属性时，用该属性作为分类名
- 同名分类的订阅会被合并到同一数组中

### 2.4 分类合并策略

分类合并在 `ImportService::importOpml()` 中实现，逻辑如下：

```
对 OPML 中的每个分类名：
    如果有 forced_category → 强制使用该分类
    否则如果分类名已存在于 categories_by_names → 复用现有分类
    否则如果分类元素有效且未达上限 → 创建新分类
    否则 → 归入默认分类
```

**分类查重**：`CategoryDAO::searchByName()` 按名称精确匹配查询。

**分类创建**：`CategoryDAO::addCategoryObject()` 先查重，不存在才插入，存在则直接返回已有 ID。

> **关键点**：分类按**名称**匹配合并，不按 ID。同名分类直接复用，订阅追加到现有分类下。

### 2.5 重复订阅处理

`FeedDAO::addFeedObject()` 是订阅去重与合并的核心：

#### 去重规则
- 按 **URL** 精确匹配查重：`searchByUrl($feed->url())`
- URL 相同即视为同一订阅

#### 已存在订阅的合并行为
当订阅 URL 已存在时，执行以下更新而非新建：

| 字段 | 处理方式 |
|------|----------|
| `mute` | 设为 `false`（取消静音） |
| `ttl` | 保留现有值，不覆盖 |
| `attributes` | **递归合并**：`array_replace_recursive(existing, import)`，导入属性覆盖同名属性 |
| `kind` | 用导入值更新 |
| `name` | 用导入值更新 |
| `website` | 用导入值更新 |
| `description` | 用导入值更新 |
| `pathEntries` | 用导入值更新 |

> **关键点**：重复订阅不会产生重复记录，而是**原地更新**，属性采用递归合并策略。导入方属性优先级更高。

### 2.6 数量限制

系统级限制由 `limits` 配置控制：

- `max_categories`：最大分类数
- `max_feeds`：最大订阅数

**Web 端**：受数量限制，达上限时记录警告并停止创建
**CLI 端**：不受数量限制（`FreshRSS_Context::$isCli` 判断）

### 2.7 异常与错误处理

#### OPML 解析异常
- 捕获 `\marienfressinaud\LibOpml\Exception`
- 记录错误日志，设置 `lastStatus = false`，直接返回
- 由于使用 `strict: false` 模式，大部分格式问题会被容忍

#### 订阅创建异常
- 捕获 `FreshRSS_Feed_Exception`
- 记录日志，标记状态失败，跳过当前订阅继续处理下一个

#### ZIP 文件异常
- 无 `zip` 扩展：抛出 `FreshRSS_ZipMissing_Exception`，前端提示安装扩展
- ZIP 打开失败：抛出 `FreshRSS_Zip_Exception`，记录错误码

#### 文件上传异常（Web 端）
- 检查 `$_FILES['file']['error']` 错误码
- 上传失败记录警告并重定向回导入页

#### 默认分类缺失
- `checkDefault()` 确保默认分类存在，不存在则自动创建
- 若获取默认分类失败，整个导入失败

---

## 三、导出流程详解

### 3.1 OPML 导出

`ExportService::generateOpml()` 生成 OPML 文件：

1. 加载所有分类及其下的订阅（`prePopulateFeeds: true, details: true`）
2. 调用视图辅助函数 `feedsToOutlines()` 转换为 OPML outline 数组
3. 使用 `LibOpml::render()` 渲染为 XML 字符串
4. 文件名格式：`feeds_YYYY-MM-DD.opml.xml`

#### OPML 命名空间
FreshRSS 扩展了 OPML 标准，使用 `frss` 命名空间：
- 命名空间 URI：`https://freshrss.org/opml`
- 所有自定义属性以 `frss:` 前缀开头

#### 导出的订阅类型
| type 值 | 对应 kind 常量 | 说明 |
|---------|---------------|------|
| `rss` | `KIND_RSS` | 标准 RSS/ATOM 订阅 |
| `HTML+XPath` | `KIND_HTML_XPATH` | HTML 爬虫订阅 |
| `XML+XPath` | `KIND_XML_XPATH` | XML XPath 订阅 |
| `JSON+DotNotation` | `KIND_JSON_DOTNOTATION` | JSON 点表示法订阅 |
| `JSONFeed` | `KIND_JSONFEED` | JSON Feed 格式 |
| `HTML+XPath+JSON+DotNotation` | `KIND_HTML_XPATH_JSON_DOTNOTATION` | 混合类型 |

#### 导出的 frss 扩展属性
- `frss:priority`：订阅优先级（important/main/category/feed/hidden）
- `frss:unicityCriteria`：文章去重规则
- `frss:unicityCriteriaForced`：是否强制去重规则
- `frss:xPath*`：XPath 爬虫配置（item/title/content/uri/author/timestamp 等）
- `frss:json*`：JSON 点表示法配置
- `frss:xPathToJson`：XPath 转 JSON 配置
- `frss:cssFullContent`：全文抓取 CSS 选择器
- `frss:cssFullContentConditions`：全文抓取条件
- `frss:cssContentFilter`：内容过滤
- `frss:filtersActionRead`：自动标读过滤器
- `frss:CURLOPT_*`：各种 cURL 参数（cookie/useragent/proxy/headers 等）

#### 动态 OPML 分类
分类为 `KIND_DYNAMIC_OPML` 类型时，导出 `frss:opmlUrl` 属性。

### 3.2 文章数据导出

**星标/标签文章**：`generateStarredEntries()`
- 参数 `S`=仅星标，`T`=仅标签，`ST`=星标或标签
- 输出为 JSON 格式（Google Reader 兼容格式）
- 星标与标签文章合并在同一个文件中避免内容重复

**指定订阅文章**：`generateFeedEntries($feed_id, $max_number_entries)`
- 每个订阅一个 JSON 文件
- 默认最多 50 条

### 3.3 多文件 ZIP 打包

- 单个文件导出：直接返回原文件
- 多个文件导出：打包为 ZIP 压缩包
- 无 `zip` 扩展时返回错误提示
- ZIP 文件名格式：`freshrss_{username}_{YYYY-MM-DD}_export.zip`

---

## 四、核心代码路径速查

### 导入主路径
```
importAction (Web) / cli/import-for-user.php (CLI)
    ↓
importFile()
    ├─ guessFileType() 识别文件类型
    ├─ ZIP 解压 / TXT 转 OPML
    └─ 按顺序导入各类文件
        └─ OPML 导入 → ImportService::importOpml()
            ├─ LibOpml::parseString() 解析 XML
            ├─ loadFromOutlines() 递归解析结构
            ├─ 分类合并（按名称匹配）
            └─ 订阅创建 / 更新
                └─ FeedDAO::addFeedObject()
                    ├─ searchByUrl() 查重
                    ├─ 不存在 → addFeed() 新建
                    └─ 已存在 → 更新字段 + 合并 attributes
```

### 导出主路径
```
exportAction (Web) / cli/export-opml-for-user.php (CLI)
    ↓
ExportService::generateOpml()
    ├─ listCategories() 加载分类与订阅
    ├─ 视图辅助 feedsToOutlines() 转换格式
    └─ LibOpml::render() 生成 XML
```

---

## 五、关键设计特点

1. **名称匹配的分类合并**：分类按名称精确匹配，同名分类直接复用，保证导入不会产生重复分类
2. **URL 匹配的订阅去重**：订阅按 URL 匹配，重复时执行原地更新而非新建
3. **属性递归合并**：订阅的 `attributes` 字段使用 `array_replace_recursive` 合并，导入方优先级高
4. **容错解析模式**：OPML 解析使用 `strict: false`，最大程度兼容各种非标准 OPML 文件
5. **顺序导入策略**：先导入 OPML（分类+订阅），再导入文章数据，确保外键关联有效
6. **Web/CLI 差异**：CLI 模式下不受数量限制，便于管理员批量操作
7. **FreshRSS 扩展命名空间**：通过 `frss:` 命名空间导出高级配置，与标准 OPML 兼容
