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
| 文章模型 | [Entry.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Models/Entry.php) | `toGReader()` 导出格式生成、标签/固有 tags 存储 |

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

## 三、文章导出：分类标签与文章标签的写入位置

文章通过 `FreshRSS_Entry::toGReader()` [L1219-L1319](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Models/Entry.php#L1219-L1319) 方法导出为 Google Reader 兼容 JSON 格式。该方法通过 `$mode` 参数区分导出模式，不同模式下写入内容有差异。

### 3.1 `categories` 数组中写入的四类信息

所有"分类/标签/状态"相关信息**全部写入文章的 `categories` 数组**，用前缀区分语义：

| 类别 | 写入格式 | 代码位置 | 说明 |
|------|----------|----------|------|
| **系统状态** | `user/-/state/com.google/reading-list` 等 | L1242, L1305-L1311 | 阅读列表、已读/未读、星标等 |
| **订阅优先级状态** | `user/-/state/org.freshrss/important` 等 | L1274-L1280 | main / important / hidden |
| **订阅所属分类名** | `user/-/label/分类名` | L1262-L1264 | **仅非 `freshrss` 模式写入** |
| **用户给文章打的标签（Label）** | `user/-/label/标签名` | L1312-L1314 | 由 `$labels` 参数传入，来自 `_tag` + `_entrytag` |
| **文章固有 tags** | 直接写入字符串，无前缀 | L1315-L1317 | 来自 `_entry.tags` 字段，`$this->tags()` 返回 |

> **⚠️ 极其重要：订阅所属分类名与用户文章标签（Label）使用相同的 `user/-/label/` 前缀，在 JSON 中无法区分来源。**

### 3.2 `origin` 对象中写入的订阅信息

`origin` 对象存储文章所属订阅的来源信息，不同模式写入不同字段：

| 字段 | 写入条件 | 代码位置 | 内容 |
|------|----------|----------|------|
| `streamId` | 始终写入 | L1245 | `'feed/' + feedId`，但 feedId 是数据库自增 ID，跨实例不可用 |
| `htmlUrl` | feed 不为 null | L1266 | 订阅的网站 URL |
| `title` | feed 不为 null | L1267 | 订阅名称 |
| `feedUrl` | **仅 `mode === 'freshrss'`** | L1271 | **订阅的 RSS/ATOM URL，是重新匹配订阅的关键** |

> **⚠️ 关键差异：`feedUrl` 只在 `freshrss` 模式下才写入 origin。非 `freshrss` 模式（兼容模式）导出时，origin 里没有订阅 URL。**

### 3.3 `origin.category` 的缺失

`importExportController::addFeedJson()` [L619](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Controllers/importExportController.php#L619) 在创建订阅时会尝试读取 `$origin['category']`，但：

- **`toGReader()` 从不把分类名写入 `origin.category` 字段**
- 分类名只出现在 `categories` 数组中（`user/-/label/分类名` 格式）
- 导入时 `categories` 数组中 `user/-/label/` 前缀的内容**全部被当作文章标签（Label）处理**，不会作为订阅分类

> **推论：FreshRSS 自导出的文章 JSON 在重新导入时，如果不配合 OPML 文件，订阅分类信息会丢失，新建的订阅会进入默认分类。**

### 3.4 导出模式对比

| 模式 | 适用场景 | `origin.feedUrl` | 分类名写入 categories | `guid` 字段 |
|------|----------|-------------------|----------------------|-------------|
| `freshrss` | FreshRSS 内部导入导出 | ✅ 写入 | ❌ 不写入 | ✅ 写入 |
| `compat` | 兼容外部阅读器 | ❌ 不写入 | ✅ 写入（作为 label） | ❌ 不写入 |
| 空字符串 | 默认/API 调用 | ❌ 不写入 | ✅ 写入（作为 label） | ❌ 不写入 |

在 `ExportService` 中：
- `generateStarredEntries()` 和 `generateFeedEntries()` 均调用 `toGReader('freshrss')` → 写入 `feedUrl`，**不写入分类名到 categories**

---

## 四、文章导入：如何从来源信息找回原订阅

### 4.1 订阅 URL 获取链路

`importExportController::importJson()` [L372-L383](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Controllers/importExportController.php#L372-L383) 按以下优先级获取订阅 URL：

```
优先级 1：origin.feedUrl            ← 仅 freshrss 模式导出时有
优先级 2：origin.streamId（去 feed/ 前缀） ← Google Reader 兼容格式，通常是数字 ID，跨实例无效
优先级 3：origin.htmlUrl            ← 网站 URL，可能不是 feed URL
兜底   ：http://import.localhost/import.xml（标记禁用）
```

**重新匹配成功的关键**：导出时必须使用 `freshrss` 模式（已写入 `origin.feedUrl`），或者 OPML 文件已先导入创建了订阅。

### 4.2 订阅查找与创建流程

```
遍历每篇文章
    ↓
从 origin 提取 feedUrl（如上优先级）
    ↓
searchByUrl(feedUrl) 在数据库查找订阅
    ↓
    ├─ 找到 → 使用该订阅 ID 关联文章
    └─ 找不到 → 调用 addFeedJson(origin) 补建订阅
                    ↓
                    addFeedJson 内部：
                    ├─ URL 取 origin.feedUrl 或 origin.htmlUrl
                    ├─ 分类名取 origin.category（通常为空，因为导出时没写）
                    │   └─ 为空则使用默认分类
                    ├─ 名称取 origin.title（默认 "Import"）
                    └─ 调用 addFeedObject() 入库（内部再次按 URL 查重）
```

### 4.3 `addFeedJson()` 分类处理的细节

`addFeedJson()` [L618-L623](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Controllers/importExportController.php#L618-L623)：

```php
$cat_id = FreshRSS_CategoryDAO::DEFAULTCATEGORYID;
$cat_name = trim($origin['category'] ?? '');
if ($cat_name !== '') {
    $new_cat = $this->categoryDAO->searchByName($cat_name);
    $cat_id = $new_cat?->id() ?: $this->categoryDAO->addCategory(['name' => $cat_name]) ?: DEFAULTCATEGORYID;
}
```

- 由于 `toGReader()` 从不写入 `origin.category`，`$cat_name` 通常为空字符串
- `$cat_id` 直接取 `DEFAULTCATEGORYID`
- 除非文章 JSON 来自其他外部系统并正确设置了 `origin.category`，否则**自动补建的订阅全部进入默认分类**

### 4.4 导入时 categories 数组的解析

`importJson()` [L443-L464](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Controllers/importExportController.php#L443-L464) 对 `categories` 数组逐项处理：

```
遍历 categories 数组中的每个字符串：
    ↓
匹配 user/xxx/ 前缀？
    ├─ 否 → 保留为文章固有 tags（存入 _entry.tags 字段）
    └─ 是 → 进一步匹配：
            ├─ state/com.google/starred  → is_starred = true
            ├─ state/com.google/read     → is_read = true
            ├─ state/com.google/unread   → is_read = false
            ├─ label/标签名              → 收集为文章 Label（存入 _tag + _entrytag）
            └─ 其他（如 state/org.freshrss/important）→ 直接丢弃
```

**关键点**：
- `user/-/label/` 前缀的全部当作文章 Label，与订阅分类无关
- `user/-/state/org.freshrss/main` 等订阅优先级信息在导入时**被丢弃**，不会恢复到订阅上
- 无前缀的字符串保留为文章固有 tags（如 TT-RSS 的 `tag_cache`）

### 4.5 重新匹配订阅的完整链路

**链路 A：OPML 先导入（推荐，分类信息完整保留）**
```
1. 导入 OPML → 分类创建 / 合并 → 订阅创建（带正确分类）
2. 导入文章 JSON → 从 origin.feedUrl 提取 URL
3. searchByUrl() 命中 OPML 已创建的订阅
4. 文章通过 feedId 关联到已有订阅 → 分类正确保留
```

**链路 B：仅导入文章 JSON（不推荐，分类丢失）**
```
1. 导入文章 JSON → 从 origin.feedUrl 提取 URL
2. searchByUrl() 未命中
3. addFeedJson() 补建订阅 → origin.category 为空 → 进入默认分类
4. 文章关联到新建订阅 → 所有订阅在默认分类下
5. 原分类名以 user/-/label/分类名 形式出现在 categories 数组
   → 被当作文章 Label 处理 → 每篇文章都打上原分类名作为标签
```

---

## 五、重复订阅且分类不同：全链路表现

### 5.1 场景定义

- 已有状态：订阅 URL `http://example.com/feed.xml` 存在，属于**分类 A**
- 导入数据：OPML 中同一 URL 出现在**分类 B** 下，或文章 JSON 的 categories 含 `user/-/label/分类B`

### 5.2 OPML 导入链路的表现

```
OPML 解析 → 分类 B 下的订阅列表包含该 URL
    ↓
循环处理：addFeedObject(url, category=分类B)
    ↓
searchByUrl(url) 命中已存在的订阅（在分类 A）
    ↓
执行 UPDATE，更新字段不包含 category
    ↓
最终结果：
    ✓ 订阅保留在分类 A（原分类不变）
    ✓ 订阅的 name / website / attributes 等被 OPML 中的值更新
    ✗ 分类 B 不会包含该订阅
    ✗ 不会产生重复订阅记录
```

### 5.3 文章 JSON 导入链路的表现

```
文章 categories 含 "user/-/label/分类B"（导出时写入的原分类名）
    ↓
从 origin.feedUrl 获取 URL → searchByUrl() 命中分类 A 下的订阅
    ↓
文章关联到该订阅（feedId 指向分类 A 的订阅）
    ↓
解析 categories：
    "user/-/label/分类B" → 识别为 Label
    → 收集到 labels 映射 → 创建/查找 "分类B" 标签 → 建立文章-标签关联
    ↓
最终结果：
    ✓ 文章归入分类 A 下的订阅（因为订阅在分类 A）
    ✓ 文章额外打上一个叫 "分类B" 的标签（Label）
    ✗ 订阅本身不会被移动到分类 B
    ✗ 分类 B 不会被创建为订阅分类（除非 OPML 中有）
```

### 5.4 数据库层面的最终状态

| 数据 | 存储位置 | 最终值 |
|------|----------|--------|
| 订阅分类 | `_feed.category` | 分类 A 的 ID（不变） |
| 订阅名称等属性 | `_feed` 各字段 | OPML/JSON 中导入的值（覆盖） |
| 文章所属订阅 | `_entry.id_feed` | 分类 A 下订阅的 ID |
| "分类B" 这个名称 | `_tag.name` + `_entrytag` | 作为文章标签存在，每篇文章关联 |
| 分类 B | `_category` 表 | **不会被创建**（除非 OPML 中有其他订阅） |

### 5.5 用户视角的表现

在 FreshRSS UI 中：
- **订阅列表**：该订阅显示在"分类 A"下，与导入前相同
- **分类 B 订阅列表**：空，看不到该订阅
- **文章视图**：每篇文章显示有"分类B"标签（Label），可通过标签过滤器筛选
- **标签列表**：出现一个名为"分类B"的标签，包含所有从该分类导出的文章

---

## 六、导出流程详解

### 6.1 OPML 订阅导出

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

### 6.2 文章数据导出

#### 星标 / 标签文章
`ExportService::generateStarredEntries()` [L71-L89](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Services/ExportService.php#L71-L89)：
- 参数 `S`=仅星标，`T`=仅标签，`ST`=星标或标签
- 调用 `toGReader('freshrss')` → 写入 `feedUrl`，**不写入分类名到 categories**
- **星标与标签合并在同一个文件**，避免同一篇文章重复出现

#### 指定订阅文章
`ExportService::generateFeedEntries()` [L96-L124](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Services/ExportService.php#L96-L124)：
- 每个订阅一个 JSON 文件
- 默认最多 50 条
- 同样调用 `toGReader('freshrss')`

### 6.3 OPML 与文章文件的关系

> **重要：OPML 文件与文章文件是分别生成的独立文件，不存在内容合并。**

它们的关系是：
- **结构上独立**：OPML 只含订阅+分类结构；文章 JSON 只含文章数据
- **通过 URL 关联**：文章 JSON 中的 `origin.feedUrl` 对应 OPML 中的订阅 URL（`xmlUrl` 属性）
- **打包方式**：多个文件统一打包进 ZIP 压缩包

导出文件清单（ZIP 内）：
```
freshrss_用户_日期_export.zip
├─ feeds_日期.opml.xml          ← 订阅+分类（OPML 格式）
├─ starred_日期.json            ← 星标+标签文章（JSON 格式）
├─ feed_日期_分类ID_订阅ID.json ← 各订阅的文章（每个订阅一个文件）
└─ ...
```

### 6.4 多文件 ZIP 打包

`ExportService::zip()` [L152-L178](file:///d:/fz/0601-1/solo-dogfeeding/code/26-FreshRSS/app/Services/ExportService.php#L152-L178)：

- 单个文件 → 直接返回原文件内容
- 多个文件 → 打包为 ZIP 压缩包
- 无 `zip` 扩展时返回错误提示
- 文件名格式：`freshrss_{username}_{YYYY-MM-DD}_export.zip`

---

## 七、关键数据结构

### 7.1 数据库表关系

```
_category (分类表)
    └── _feed (订阅表)  ←─ category 外键（每个订阅属于一个分类）
            └── _entry (文章表)  ←─ id_feed 外键（每篇文章属于一个订阅）
                    ├── is_favorite 字段  ←─ 星标状态
                    ├── tags 字段         ←─ 文章固有 tags（字符串数组）
                    └── _entrytag (关联表) ←─ 多对多
                            └── _tag (标签表)  ←─ 用户打的 Label
```

### 7.2 导出的 JSON 文章格式（Google Reader 兼容）

每篇文章的关键字段：
```json
{
  "id": "tag:google.com,2005:reader/item/...",
  "frss:id": "FreshRSS内部文章ID",
  "guid": "文章唯一标识（仅freshrss模式）",
  "title": "标题",
  "published": "时间戳",
  "content": { "content": "HTML内容" },
  "alternate": [{ "href": "文章链接" }],
  "origin": {
    "streamId": "feed/订阅数据库ID",
    "title": "订阅名称",
    "htmlUrl": "网站URL",
    "feedUrl": "订阅URL（仅freshrss模式，重新匹配的关键）"
  },
  "categories": [
    "user/-/state/com.google/reading-list",
    "user/-/state/com.google/starred",
    "user/-/state/com.google/read",
    "user/-/state/org.freshrss/important",
    "user/-/label/订阅分类名（非freshrss模式）",
    "user/-/label/用户文章标签名",
    "文章固有tags（无前缀）"
  ]
}
```

---

## 八、核心代码路径速查

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
                    └─ 已存在 → 更新字段（category 不变）+ 合并 attributes
```

### 文章导入路径
```
importFile()
    └─ JSON 文章导入 → importJson()
        ├─ 解析 JSON 文章列表
        ├─ 遍历文章：从 origin 提取 feedUrl（优先级：feedUrl > streamId > htmlUrl）
        │   ├─ searchByUrl() 查找订阅
        │   └─ 不存在则 addFeedJson() 补建订阅（origin.category 为空 → 默认分类）
        ├─ 解析 categories 数组：
        │   ├─ state/com.google/* → is_starred / is_read
        │   ├─ label/* → 文章 Label（通过 _tag + _entrytag）
        │   └─ 其他无前缀 → 文章固有 tags（存入 _entry.tags）
        ├─ 按 GUID 批量查重
        ├─ 事务1：批量插入/更新文章
        ├─ commitNewEntries() + 更新缓存
        └─ 事务2：tagEntry() 批量建立文章-标签关联
```

### 文章导出路径
```
ExportService::generateStarredEntries() / generateFeedEntries()
    ↓
FreshRSS_Entry::toGReader('freshrss')
    ├─ origin.feedUrl → 写入（重新匹配的关键）
    ├─ origin.streamId / htmlUrl / title → 写入
    ├─ origin.category → ❌ 不写入
    ├─ categories：
    │   ├─ state/com.google/* → 星标/已读状态
    │   ├─ state/org.freshrss/* → 订阅优先级
    │   ├─ user/-/label/分类名 → ❌ freshrss 模式不写入
    │   ├─ user/-/label/标签名 → 写入（用户 Label）
    │   └─ 无前缀 → 写入（文章固有 tags）
    └─ guid → 写入（freshrss 模式）
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

## 九、关键设计特点与潜在问题总结

### 9.1 设计特点

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

### 9.2 已知的链路特性（非 Bug，是设计选择）

| 特性 | 表现 | 影响场景 |
|------|------|----------|
| **分类名与文章 Label 格式相同** | 都使用 `user/-/label/` 前缀，导入时无法区分 | 仅导入文章 JSON 时，原分类名变成每篇文章的 Label |
| **`origin.category` 从未写入** | `toGReader()` 不输出该字段 | 仅导入文章 JSON 时，自动补建的订阅全部进入默认分类 |
| **重复订阅不迁移分类** | `addFeedObject()` UPDATE 不包含 category 字段 | 同一 URL 在不同分类的 OPML 中出现时，保留首次导入的分类 |
| **订阅优先级导入时丢失** | `categories` 中 `state/org.freshrss/*` 在导入时被丢弃 | 文章导入不会恢复订阅的 main/important/hidden 优先级 |
| **`freshrss` 模式不写分类名** | `mode !== 'freshrss'` 才写入 `user/-/label/分类名` | 自导出文章 JSON 不含分类名信息 |
| **`streamId` 是数据库自增 ID** | `'feed/' + feedId` 跨实例不匹配 | 跨实例迁移时 streamId 无用，必须依赖 `origin.feedUrl` |

### 9.3 正确的导出+导入流程（完整保留所有信息）

```
导出：
    1. 勾选 OPML（订阅+分类结构）
    2. 勾选星标/标签文章
    3. 勾选需要导出的订阅文章
    → 生成 ZIP 包

导入（按内置顺序自动执行）：
    1. OPML 先导入 → 分类创建/合并 + 订阅创建（带正确分类）
    2. 星标文章 JSON → 按 origin.feedUrl 命中已有订阅 → 文章关联正确
    3. 订阅文章 JSON → 同上
```

如果跳过 OPML 仅导入文章 JSON，分类信息会降级为文章 Label，订阅进入默认分类。
