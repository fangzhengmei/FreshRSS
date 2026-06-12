# FreshRSS OPML 与数据导入导出代码分析

## 一、总体架构

### 核心模块

| 模块 | 文件 | 职责 |
|------|------|------|
| 控制器 | [importExportController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Controllers/importExportController.php) | Web 端请求处理、文件类型识别、导入导出调度、JSON文章导入 |
| 导入服务 | [ImportService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Services/ImportService.php) | OPML 解析、分类合并、订阅创建核心逻辑 |
| 导出服务 | [ExportService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Services/ExportService.php) | OPML 生成、文章导出、ZIP 打包 |
| OPML 解析库 | [LibOpml.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/lib/marienfressinaud/lib_opml/src/LibOpml/LibOpml.php) | OPML XML 与 PHP 数组互相转换 |
| 订阅 DAO | [FeedDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Models/FeedDAO.php) | 订阅增删改查、重复检测 |
| 分类 DAO | [CategoryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Models/CategoryDAO.php) | 分类增删改查、默认分类管理 |
| 文章 DAO | [EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Models/EntryDAO.php) | 文章增删改查、按 GUID 查重 |
| 标签 DAO | [TagDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Models/TagDAO.php) | 标签管理、文章标签关联 |

### 入口点

**Web 端导入**：
- 页面：[importExport/index.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/views/importExport/index.phtml)
- 处理：`importExportController::importAction()` → `importFile()` → 分发到各类导入

**Web 端导出**：
- 处理：`importExportController::exportAction()` → `ExportService` 生成各类文件

**CLI 导入**：[cli/import-for-user.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/cli/import-for-user.php)

**CLI 导出 OPML**：[cli/export-opml-for-user.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/cli/export-opml-for-user.php)

---

## 二、OPML 订阅导入流程

### 2.1 文件类型识别与导入顺序

`importExportController::guessFileType()` 基于文件名识别类型：

| 文件特征 | 识别类型 | 说明 |
|----------|----------|------|
| `.zip` 后缀 | `zip` | 压缩包，解压后递归识别内部文件 |
| `.txt` 后缀 | `txt` | 纯文本 URL 列表，转 OPML 后走标准管道 |
| 文件名含 `opml` | `opml` | OPML 订阅列表 |
| `.json` + 含 `starred` | `json_starred` | 星标文章 JSON |
| `.json` 后缀 | `json_feed` | 订阅文章数据 JSON |
| `.xml` + 含 `Tiny`/`tt-rss` | `ttrss_starred` | TT-RSS 星标文章 |
| `.xml` 后缀 | `opml` | 默认按 OPML 处理 |

**导入顺序**：OPML → 星标文章 → 订阅文章 → TT-RSS 文章。OPML 先导入确保分类和订阅存在，后续文章数据才能正确关联。

### 2.2 TXT 纯文本转 OPML

`importExportController::txtToOpml()` 处理 URL 列表：

- 移除 UTF-8 BOM 头
- 跳过空行、`#` 注释行、`<` 开头的 HTML 行
- `filter_var()` 验证 URL 有效性，无效则警告跳过
- 有效 URL 包装为最小 OPML 结构，复用标准导入管道

### 2.3 OPML 解析与结构遍历

`ImportService::importOpml()` 是 OPML 导入主入口：

1. 使用 `LibOpml(strict: false)` 非严格模式解析，容错性强
2. `checkDefault()` 确保默认分类存在
3. 预加载现有分类，建立 `categories_by_names` 名称索引
4. 加载 `limits` 数量限制配置
5. `loadFromOutlines()` 递归遍历 OPML outline 树
6. 逐分类执行：分类合并 → 订阅创建/更新

#### 递归解析逻辑
`loadFromOutline()` 对每个 outline 节点的处理：

- **分类节点**（有 `@outlines` 子节点）：以 `text` 为分类名，递归处理子 outline
- **订阅节点**（有 `xmlUrl`）：归入当前父分类名下
- **category 属性**：无父分类且有 `category` 属性时，用该属性作分类名
- 同名分类的订阅会被数组合并

### 2.4 分类合并策略

分类合并逻辑在 `ImportService::importOpml()` [L78-L110](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Services/ImportService.php#L78-L110)：

```
对 OPML 中的每个分类名：
    有 forced_category → 强制使用该分类
    分类名已存在 → 复用现有分类（categories_by_names 查找）
    分类不存在且未达上限 → 创建新分类
    其他情况 → 归入默认分类
```

- **查重方式**：`CategoryDAO::searchByName()` 按名称精确匹配
- **创建方式**：`CategoryDAO::addCategoryObject()` 先查重再插入，返回已有或新 ID
- **关键点**：分类按**名称**匹配，不按 ID。同名分类直接复用，订阅追加到现有分类

### 2.5 重复订阅处理（URL 维度）

`FeedDAO::addFeedObject()` [L104-L161](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Models/FeedDAO.php#L104-L161) 是订阅去重核心：

#### 去重规则
- 按 **URL** 精确匹配：`searchByUrl($feed->url())`
- URL 相同即视为同一订阅

#### 已存在时的字段更新

| 字段 | 处理方式 |
|------|----------|
| `category` (分类) | **不更新**，保留原有分类 |
| `mute` | 强制设为 `false`（取消静音） |
| `ttl` | 保留现有值，不覆盖 |
| `attributes` | **递归合并**：`array_replace_recursive(existing, import)`，导入方覆盖 |
| `kind` / `name` / `website` / `description` | 用导入值覆盖更新 |
| `pathEntries` | 用导入值覆盖更新 |

> **⚠️ 关键结论：重复订阅且分类不同时，最终保留在第一次创建时的分类下，分类不会被移动。**

### 2.6 数量限制

由系统配置 `limits` 控制：
- `max_categories`：最大分类数
- `max_feeds`：最大订阅数

Web 端受限制，达上限后记录警告并停止创建；CLI 模式（`FreshRSS_Context::$isCli`）不受限制。

### 2.7 异常与错误处理

| 异常类型 | 处理方式 |
|----------|----------|
| OPML 解析异常 | 捕获 `LibOpml\Exception`，记录日志，整个 OPML 导入失败 |
| 订阅创建异常 | 捕获 `FreshRSS_Feed_Exception`，跳过当前订阅，继续下一个 |
| ZIP 无扩展 | 抛出 `FreshRSS_ZipMissing_Exception`，前端提示 |
| ZIP 打开失败 | 抛出 `FreshRSS_Zip_Exception`，记录错误码 |
| 文件上传失败 | 检查 `$_FILES['file']['error']`，警告后重定向 |
| 默认分类缺失 | `checkDefault()` 自动创建，仍失败则整体导入失败 |

---

## 三、文章数据导入流程

### 3.1 文章导入入口

`importExportController::importJson()` [L331-L592](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Controllers/importExportController.php#L331-L592) 处理 JSON 格式文章导入（Google Reader 兼容格式）。

导入分三个阶段：
1. **准备阶段**：解析 JSON → 识别文章所属订阅 → 查找/创建对应订阅
2. **导入阶段**：按 GUID 查重 → 新增或更新文章 → 事务提交
3. **标签阶段**：解析文章标签 → 创建缺失标签 → 建立文章-标签关联

### 3.2 来源订阅补充机制

文章通过 `origin` 字段关联到订阅，处理逻辑在 `importJson()` [L356-L412](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Controllers/importExportController.php#L356-L412)：

#### 订阅 URL 获取优先级
1. `origin.feedUrl` → 直接使用
2. `origin.streamId`（`feed/` 前缀）→ 去掉前缀取 URL
3. `origin.htmlUrl` → 用网站 URL 代替
4. 都没有 → 使用占位 URL `http://import.localhost/import.xml` 并标记禁用

#### 订阅查找与创建
- 先用 `searchByUrl()` 在数据库查找
- 不存在则调用 `addFeedJson()` 创建
- 创建时使用 origin 中的 `title`（默认 "Import"）、`htmlUrl`、`category` 等信息
- 分类从 `origin.category` 获取，没有则用默认分类

> **设计意图**：即使没有 OPML 文件，仅导入文章 JSON 也能自动补建对应的订阅记录，确保文章都有归属。

### 3.3 星标信息入库

星标状态从文章的 `categories` 数组中解析 [L453-L469](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Controllers/importExportController.php#L453-L469)：

- 匹配 `user/-/state/com.google/starred` → `is_starred = true`
- 匹配 `user/-/state/com.google/read` → `is_read = true`
- 匹配 `user/-/state/com.google/unread` → `is_read = false`

星标是文章的固有属性，存储在 `_entry` 表的 `is_favorite` 字段中。

### 3.4 标签（Label）信息入库

标签也从 `categories` 数组解析，匹配 `user/-/label/` 前缀 [L459-L461](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Controllers/importExportController.php#L459-L461)：

#### 处理流程
1. 遍历文章时收集 `labelName → [articles]` 映射
2. 文章导入完成后，在独立事务中处理标签关联
3. 标签不存在则调用 `tagDAO->addTag()` 创建
4. 通过 `_entrytag` 关联表建立文章与标签的多对多关系
5. 关联使用 `INSERT IGNORE` 避免重复报错

#### 标签与分类的区别
| 维度 | 分类 (Category) | 标签 (Tag/Label) |
|------|-----------------|------------------|
| 层级 | 一级结构，每个订阅属于一个分类 | 扁平结构，每篇文章可有多个标签 |
| 存储 | `_category` 表 + `_feed.category` 字段 | `_tag` 表 + `_entrytag` 关联表 |
| OPML 中 | 以 outline 嵌套结构表示 | 以 `frss:label` 命名空间属性表示 |

### 3.5 文章重复处理（GUID 维度）

文章按 **GUID** 去重，逻辑在 `importJson()` [L521-L550](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Controllers/importExportController.php#L521-L550)：

1. 先批量查询每个订阅下已存在的 GUID 哈希（`listHashForFeedGuids`）
2. 导入时检查：存在 → 更新文章，不存在 → 新增文章
3. 同一导入批次内也通过 `$newGuids` 数组去重，避免同一文件内重复

### 3.6 事务与性能

文章导入使用多段事务：
1. 第一段事务：批量插入/更新文章
2. `commitNewEntries()`：提交新文章的二级索引
3. `updateCachedValues()`：更新订阅缓存计数
4. 第二段事务：批量建立标签关联

---

## 四、导出流程详解

### 4.1 OPML 订阅导出

`ExportService::generateOpml()` [L46-L56](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Services/ExportService.php#L46-L56)：

1. 加载所有分类及其订阅（`prePopulateFeeds: true, details: true`）
2. 视图辅助函数 `feedsToOutlines()` 转换为 OPML outline 数组
3. `LibOpml::render()` 渲染为 XML
4. 文件名：`feeds_YYYY-MM-DD.opml.xml`

#### 导出的扩展属性
FreshRSS 通过 `frss:` 命名空间导出高级配置（命名空间 URI：`https://freshrss.org/opml`）：
- 订阅类型、优先级、去重规则
- XPath / JSON 爬虫配置
- 全文抓取 CSS 选择器
- 自动标读过滤器
- cURL 参数（cookie、代理、请求头等）
- 动态 OPML 分类的 `frss:opmlUrl`

### 4.2 文章数据导出

#### 星标 / 标签文章
`ExportService::generateStarredEntries()` [L71-L89](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Services/ExportService.php#L71-L89)：
- 参数 `S`=仅星标，`T`=仅标签，`ST`=星标或标签
- 输出 Google Reader 兼容 JSON 格式
- **星标与标签合并在同一个文件**，避免同一篇文章重复出现

#### 指定订阅文章
`ExportService::generateFeedEntries()` [L96-L124](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Services/ExportService.php#L96-L124)：
- 每个订阅一个 JSON 文件
- 默认最多 50 条

### 4.3 OPML 与文章文件的关系

> **重要：OPML 文件与文章文件是分别生成的独立文件，不存在内容合并。**

它们的关系是：
- **结构上独立**：OPML 只含订阅+分类结构；文章 JSON 只含文章数据
- **通过 origin 关联**：文章 JSON 中的 `origin.feedUrl` / `origin.streamId` 对应 OPML 中的订阅 URL
- **打包方式**：多个文件统一打包进 ZIP 压缩包

导出文件清单（ZIP 内）：
```
freshrss_用户_日期_export.zip
├─ feeds_日期.opml.xml          ← 订阅+分类（OPML 格式）
├─ starred_日期.json            ← 星标+标签文章（JSON 格式）
├─ feed_日期_分类ID_订阅ID.json ← 各订阅的文章（每个订阅一个文件）
└─ ...
```

### 4.4 多文件 ZIP 打包

`ExportService::zip()` [L152-L178](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Services/ExportService.php#L152-L178)：

- 单个文件 → 直接返回原文件内容
- 多个文件 → 打包为 ZIP 压缩包
- 无 `zip` 扩展时返回错误提示
- 文件名格式：`freshrss_{username}_{YYYY-MM-DD}_export.zip`

---

## 五、关键数据结构

### 5.1 数据库表关系

```
_category (分类表)
    └── _feed (订阅表)  ←─ category 外键
            └── _entry (文章表)  ←─ id_feed 外键
                    └── _entrytag (文章标签关联表)
                            └── _tag (标签表)
```

### 5.2 导出的 JSON 文章格式（Google Reader 兼容）

每篇文章的关键字段：
```json
{
  "id": "tag:google.com,2005:reader/item/...",
  "guid": "文章唯一标识",
  "title": "标题",
  "published": "时间戳",
  "content": { "content": "HTML内容" },
  "alternate": [{ "href": "文章链接" }],
  "origin": {
    "streamId": "feed/订阅ID",
    "feedUrl": "订阅URL",
    "htmlUrl": "网站URL",
    "title": "订阅名称"
  },
  "categories": [
    "user/-/state/com.google/reading-list",
    "user/-/state/com.google/starred",
    "user/-/label/标签名",
    "user/-/state/org.freshrss/important"
  ]
}
```

---

## 六、核心代码路径速查

### OPML 订阅导入路径
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
                    ├─ searchByUrl() 按 URL 查重
                    ├─ 不存在 → addFeed() 新建（带分类）
                    └─ 已存在 → 更新字段（分类不变）+ 合并 attributes
```

### 文章导入路径
```
importFile()
    └─ JSON 文章导入 → importJson()
        ├─ 解析 JSON 文章列表
        ├─ 遍历文章：从 origin 提取 feedUrl
        │   ├─ searchByUrl() 查找订阅
        │   └─ 不存在则 addFeedJson() 补建订阅
        ├─ 按 GUID 批量查重
        ├─ 事务1：批量插入/更新文章
        ├─ commitNewEntries() + 更新缓存
        └─ 事务2：解析标签 + tagEntry() 建立关联
```

### 导出路径
```
exportAction (Web) / cli/export-opml-for-user.php (CLI)
    ↓
ExportService
    ├─ generateOpml() → OPML 订阅文件
    ├─ generateStarredEntries() → 星标/标签文章 JSON
    ├─ generateFeedEntries() → 各订阅文章 JSON
    └─ zip() → 多文件打包 ZIP
```

---

## 七、关键设计特点总结

1. **名称匹配的分类合并**：分类按名称精确匹配，同名直接复用，保证导入不产生重复分类

2. **URL 匹配的订阅去重**：订阅按 URL 匹配，重复时原地更新但**分类不变**

3. **属性递归合并**：订阅 `attributes` 用 `array_replace_recursive` 合并，导入方优先级高

4. **星标与标签同文件导出**：星标文章和标签文章合并在一个 JSON 文件中，避免内容重复

5. **文章自动补建订阅**：导入文章 JSON 时，若订阅不存在会根据 origin 信息自动补建

6. **顺序导入策略**：先 OPML（分类+订阅），后文章数据，确保外键关联有效

7. **容错解析模式**：OPML 使用 `strict: false` 解析，最大程度兼容非标准文件

8. **Web/CLI 差异**：CLI 不受数量限制，便于管理员批量操作

9. **FreshRSS 扩展命名空间**：`frss:` 命名空间导出高级配置，与标准 OPML 兼容

10. **多段事务设计**：文章导入分多段事务，平衡数据一致性与性能
