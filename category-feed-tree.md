# FreshRSS 分类与订阅层级管理分析

## 一、数据模型基础

### 1.1 分类 (Category) 模型

**核心文件**: [Category.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php)

分类模型的关键字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | int | 分类唯一标识 |
| `name` | string | 分类名称（HTML编码） |
| `kind` | int | 分类类型：`KIND_NORMAL=0`（普通），`KIND_DYNAMIC_OPML=2`（动态OPML） |
| `nbFeeds` | int | 该分类下的订阅数量（延迟加载） |
| `nbNotRead` | int | 该分类下的未读文章数（延迟加载，仅统计优先级大于 PRIORITY_HIDDEN 的订阅） |
| `feeds` | array | 该分类下的订阅对象数组（键为feed ID） |
| `lastUpdate` | int | 最后更新时间戳 |
| `error` | bool | 是否有错误 |

### 1.2 订阅 (Feed) 模型

**核心文件**: [Feed.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Feed.php)

订阅模型的关键字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | int | 订阅唯一标识 |
| `url` | string | RSS/Atom 源 URL |
| `kind` | int | 订阅类型（RSS、HTML+XPath、JSON等） |
| `categoryId` | int | 所属分类 ID |
| `category` | FreshRSS_Category | 所属分类对象（延迟加载） |
| `name` | string | 订阅名称 |
| `priority` | int | 优先级，影响是否在主视图显示 |
| `nbEntries` | int | 文章总数（缓存值） |
| `nbNotRead` | int | 未读文章数（缓存值） |
| `error` | int | 错误时间戳，大于0表示有错误 |
| `mute` | bool | 是否静音（禁用刷新） |

**优先级常量**：
- `PRIORITY_IMPORTANT = 20` - 重要订阅
- `PRIORITY_MAIN_STREAM = 10` - 主流订阅（默认）
- `PRIORITY_CATEGORY = 0` - 分类级别
- `PRIORITY_FEED = -5` - 普通订阅
- `PRIORITY_HIDDEN = -10` - 隐藏/归档订阅

---

## 二、默认分类机制

### 2.1 默认分类的定义

**核心文件**: [CategoryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/CategoryDAO.php#L6-L7)

```php
public const DEFAULTCATEGORYID = 1;
public const DEFAULT_CATEGORY_NAME = 'Uncategorized';
```

- 默认分类的 **ID 固定为 1**
- 默认名称为 `'Uncategorized'`，但在 UI 上会根据当前语言翻译为"默认分类"

### 2.2 默认分类的初始化

**核心方法**: `FreshRSS_CategoryDAO::checkDefault()`

[CategoryDAO.php#L403-L428](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/CategoryDAO.php#L403-L428)

```php
public function checkDefault(): int|bool {
    $def_cat = $this->searchById(self::DEFAULTCATEGORYID);

    if ($def_cat == null) {
        $cat = new FreshRSS_Category(_t('gen.short.default_category'), self::DEFAULTCATEGORYID);
        // 插入数据库...
    }
    return true;
}
```

该方法在多个控制器的 `firstAction()` 中被调用，确保每次请求时默认分类都存在：

- [categoryController.php#L21-L23](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/categoryController.php#L21-L23)
- [subscriptionController.php#L19-L21](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/subscriptionController.php#L19-L21)

### 2.3 默认分类的判断

**核心方法**: `FreshRSS_Category::isDefault()`

[Category.php#L79-L81](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php#L79-L81)

```php
public function isDefault(): bool {
    return $this->id == FreshRSS_CategoryDAO::DEFAULTCATEGORYID;
}
```

### 2.4 默认分类的名称处理

当分类 ID 为默认分类 ID 时，设置名称会被忽略，使用国际化翻译的名称：

[Category.php#L153-L158](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php#L153-L158)

```php
public function _id(int $id): void {
    $this->id = $id;
    if ($id === FreshRSS_CategoryDAO::DEFAULTCATEGORYID) {
        $this->name = _t('gen.short.default_category');
    }
}
```

设置名称时也会跳过默认分类：

[Category.php#L164-L168](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php#L164-L168)

```php
public function _name(string $value): void {
    if ($this->id !== FreshRSS_CategoryDAO::DEFAULTCATEGORYID) {
        $this->name = mb_strcut(trim($value), 0, FreshRSS_DatabaseDAO::LENGTH_INDEX_UNICODE, 'UTF-8');
    }
}
```

### 2.5 默认分类作为"安全网"

默认分类在以下场景中作为兜底：

1. **删除分类时**：被删除分类下的所有订阅会被移到默认分类
2. **移动订阅时**：如果目标分类不存在，会使用默认分类
3. **添加订阅时**：如果未指定分类，会使用默认分类

---

## 三、订阅移动机制

### 3.1 单个订阅移动

**核心方法**: `FreshRSS_feed_Controller::moveFeed()`

[feedController.php#L1048-L1069](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/feedController.php#L1048-L1069)

```php
public static function moveFeed(int $feed_id, int $cat_id, string $new_cat_name = ''): bool {
    if ($feed_id <= 0 || ($cat_id <= 0 && $new_cat_name === '')) {
        return false;
    }
    FreshRSS_UserDAO::touch();

    $catDAO = FreshRSS_Factory::createCategoryDao();
    if ($cat_id > 0) {
        $cat = $catDAO->searchById($cat_id);
        $cat_id = $cat === null ? 0 : $cat->id();
    }
    if ($cat_id <= 1 && $new_cat_name != '') {
        $cat_id = $catDAO->addCategory(['name' => $new_cat_name]);
    }
    if ($cat_id <= 1) {
        $catDAO->checkDefault();
        $cat_id = FreshRSS_CategoryDAO::DEFAULTCATEGORYID;
    }

    $feedDAO = FreshRSS_Factory::createFeedDao();
    return $feedDAO->updateFeed($feed_id, ['category' => $cat_id]);
}
```

移动逻辑的关键点：

1. **参数校验**：feed_id 必须有效，且必须有目标分类 ID 或新分类名称
2. **目标分类验证**：如果指定了 cat_id，验证该分类是否存在，不存在则置为 0
3. **创建新分类**：如果 cat_id 无效但指定了 new_cat_name，则创建新分类
4. **兜底到默认分类**：如果以上都失败，使用默认分类（ID=1）
5. **执行移动**：通过 `updateFeed()` 更新 feed 的 `category` 字段

### 3.2 控制器动作

**动作方法**: `FreshRSS_feed_Controller::moveAction()`

[feedController.php#L1083-L1099](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/feedController.php#L1083-L1099)

接收 POST 参数：
- `f_id` - 要移动的订阅 ID
- `c_id` - 目标分类 ID

### 3.3 批量移动（删除分类时）

**核心方法**: `FreshRSS_FeedDAO::changeCategory()`

[FeedDAO.php#L282-L306](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/FeedDAO.php#L282-L306)

```php
public function changeCategory(int $idOldCat, int $idNewCat): int|false {
    $catDAO = FreshRSS_Factory::createCategoryDao();
    $newCat = $catDAO->searchById($idNewCat);
    if ($newCat === null) {
        $newCat = $catDAO->getDefault();
    }
    if ($newCat === null) {
        return false;
    }

    $sql = 'UPDATE `_feed` SET category=:new_category WHERE category=:old_category';
    // 执行更新...
}
```

删除分类时调用此方法将所有订阅移到默认分类：

[categoryController.php#L231-L254](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/categoryController.php#L231-L254)

```php
public function deleteAction(): void {
    // ...
    if ($id === FreshRSS_CategoryDAO::DEFAULTCATEGORYID) {
        // 默认分类不能删除
        Minz_Request::bad(_t('feedback.sub.category.not_delete_default'), $url_redirect);
    }

    if ($feedDAO->changeCategory($id, FreshRSS_CategoryDAO::DEFAULTCATEGORYID) === false) {
        Minz_Request::bad(_t('feedback.sub.category.error'), $url_redirect);
    }

    if ($catDAO->deleteCategory($id) === false) {
        Minz_Request::bad(_t('feedback.sub.category.error'), $url_redirect);
    }
    // ...
}
```

### 3.4 订阅管理页面移动

在订阅配置页面也可以修改订阅所属分类：

[subscriptionController.php#L356-L368](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/subscriptionController.php#L356-L368)

```php
$values = [
    // ...
    'category' => Minz_Request::paramInt('category'),
    // ...
];

// ...
if ($values['url'] != '' && $feedDAO->updateFeed($id, $values) !== false) {
    $feed->_categoryId($values['category']);
    // ...
}
```

---

## 四、侧边栏统计机制

### 4.1 统计数据结构

**核心文件**: [Context.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Context.php)

全局统计变量：

| 变量 | 类型 | 说明 |
|------|------|------|
| `$total_unread` | int | 主视图未读总数（仅统计 PRIORITY_MAIN_STREAM 及以上） |
| `$total_important_unread` | int | 重要订阅未读数（统计 PRIORITY_IMPORTANT 及以上） |
| `$total_starred` | array | 收藏文章统计（all/read/unread） |
| `$get_unread` | int | 当前视图的未读数 |

### 4.2 统计数据初始化

**核心方法**: `FreshRSS_Context::updateUsingRequest()`

[Context.php#L239-L246](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Context.php#L239-L246)

```php
public static function updateUsingRequest(bool $computeStatistics): void {
    if ($computeStatistics && self::$total_unread === 0) {
        // Update number of read / unread variables.
        $entryDAO = FreshRSS_Factory::createEntryDao();
        self::$total_starred = $entryDAO->countUnreadReadFavorites();
        self::$total_unread = FreshRSS_Category::countUnread(self::categories(), FreshRSS_Feed::PRIORITY_MAIN_STREAM);
        self::$total_important_unread = FreshRSS_Category::countUnread(self::categories(), FreshRSS_Feed::PRIORITY_IMPORTANT);
    }
    // ...
}
```

### 4.3 分类级别的未读统计

**核心方法**: `FreshRSS_Category::countUnread()`

[Category.php#L339-L345](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php#L339-L345)

```php
public static function countUnread(array $categories, int $minPriority = FreshRSS_Feed::PRIORITY_FEED): int {
    $n = 0;
    foreach ($categories as $category) {
        $n += $category->nbNotRead($minPriority);
    }
    return $n;
}
```

### 4.4 单个分类的未读统计

**核心方法**: `FreshRSS_Category::nbNotRead()`

[Category.php#L95-L114](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php#L95-L114)

```php
public function nbNotRead(int $minPriority = FreshRSS_Feed::PRIORITY_FEED): int {
    if ($this->nbNotRead > 0 && $minPriority === FreshRSS_Feed::PRIORITY_FEED) {
        return $this->nbNotRead;
    }
    if ($this->feeds === null) {
        $catDAO = FreshRSS_Factory::createCategoryDao();
        $nb = $catDAO->countNotRead($this->id(), $minPriority);
        if ($minPriority === FreshRSS_Feed::PRIORITY_FEED) {
            $this->nbNotRead = $nb;
        }
        return $nb;
    }
    $nb = 0;
    foreach ($this->feeds as $feed) {
        if ($feed->priority() >= $minPriority) {
            $nb += $feed->nbNotRead();
        }
    }
    return $nb;
}
```

统计方式：
1. **有缓存且使用默认优先级**：直接返回缓存值
2. **未预加载 feeds**：通过 DAO 直接查询数据库
3. **已预加载 feeds**：遍历该分类下所有订阅，累加满足优先级要求的订阅的未读数

### 4.5 单个订阅的未读统计

**核心方法**: `FreshRSS_Feed::nbNotRead()`

[Feed.php#L413-L420](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Feed.php#L413-L420)

```php
public function nbNotRead(): int {
    if ($this->nbNotRead < 0) {
        $feedDAO = FreshRSS_Factory::createFeedDao();
        $this->nbNotRead = $feedDAO->countNotRead($this->id());
    }
    return $this->nbNotRead;
}
```

### 4.6 数据库级别的缓存

Feed 表中有缓存字段用于存储未读数和文章总数：

- `cache_nbEntries` - 文章总数缓存
- `cache_nbUnreads` - 未读数缓存

**更新缓存方法**: `FreshRSS_FeedDAO::updateCachedValues()`

[FeedDAO.php#L577-L608](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/FeedDAO.php#L577-L608)

```php
public function updateCachedValues(int ...$feedIds): int|false {
    $sql = <<<SQL
        UPDATE `_feed`
        LEFT JOIN (
            SELECT
                id_feed,
                COUNT(*) AS total_entries,
                SUM(CASE WHEN is_read = 0 THEN 1 ELSE 0 END) AS unread_entries
            FROM `_entry`
            WHERE $whereEntryIdFeeds
            GROUP BY id_feed
        ) AS entry_counts ON entry_counts.id_feed = `_feed`.id
        SET `cache_nbEntries` = COALESCE(entry_counts.total_entries, 0),
            `cache_nbUnreads` = COALESCE(entry_counts.unread_entries, 0)
        WHERE $whereFeedIds
    SQL;
    // ...
}
```

该方法在以下时机被调用：
- 刷新订阅后
- 提交新文章后
- 标记已读后

### 4.7 侧边栏视图渲染

**核心文件**: [aside_feed.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml)

侧边栏结构：

1. **全局导航项**（第41-63行）：
   - 全部文章（主视图）：显示 `$total_unread`
   - 重要订阅：显示 `$total_important_unread`
   - 收藏文章：显示 `$total_starred['unread']`

2. **标签列表**（第69-94行）：
   - 显示标签分类及其未读数

3. **分类与订阅树**（第96-176行）：
   - 遍历 `$this->categories`
   - 每个分类显示其未读数：`$cat->nbNotRead()`
   - 每个分类下遍历其订阅：`$cat->feeds()`
   - 每个订阅显示其未读数：`$feed->nbNotRead()`
   - 优先级低于 `PRIORITY_FEED` 的订阅在非当前选中时不显示

数据通过 `FreshRSS_Context::categories()` 获取：

[Context.php#L202-L209](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Context.php#L202-L209)

```php
public static function categories(): array {
    if (empty(self::$categories)) {
        $catDAO = FreshRSS_Factory::createCategoryDao();
        self::$categories = $catDAO->listSortedCategories(prePopulateFeeds: true, details: false);
    }
    return self::$categories;
}
```

### 4.8 分类列表预加载

**核心方法**: `FreshRSS_CategoryDAO::listCategories()`

[CategoryDAO.php#L326-L358](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/CategoryDAO.php#L326-L358)

当 `prePopulateFeeds = true` 时，通过 LEFT JOIN 一次性查询所有分类及其订阅，避免 N+1 查询问题：

```php
$sql = <<<SQL
    SELECT c.id AS c_id, c.name AS c_name, ..., {$feedFields}
    FROM `_category` c
    LEFT OUTER JOIN `_feed` f ON f.category=c.id
    GROUP BY f.id, c_id
    ORDER BY c.name, f.name
SQL;
```

### 4.9 未读数显示控制

可以配置是否显示未读数，有三级控制：

1. **全局配置**：`FreshRSS_Context::userConf()->show_unread_count`
   - `'all'` - 全部显示
   - `'important'` - 仅重要订阅显示
   - `'none'` - 都不显示

2. **分类级别**：`$cat->showUnreadCount()`
   - 读取分类的 `show_unread_count` 属性
   - 未设置时继承全局配置

3. **订阅级别**：`$feed->showUnreadCount()`
   - 读取订阅的 `show_unread_count` 属性
   - 未设置时继承分类配置
   - 重要订阅在全局非 'none' 时总是显示

---

## 五、三者之间的关系

### 5.1 层级结构关系

```
侧边栏 (Sidebar)
  ├── 全部文章 (total_unread)
  ├── 重要订阅 (total_important_unread)
  ├── 收藏文章
  ├── 标签
  │     └── 标签1 (nbUnread)
  └── 分类1 (Category) ── nbNotRead()
        ├── 订阅1 (Feed) ── nbNotRead()
        ├── 订阅2 (Feed) ── nbNotRead()
        └── ...
  └── 默认分类 (DEFAULTCATEGORYID=1) ── nbNotRead()
        ├── 订阅3 (Feed) ── nbNotRead()
        └── ...
```

### 5.2 数据流关系

```
数据库表: _entry (文章)
    │
    ├── 每个文章有 is_read 字段和 id_feed 外键
    ▼
数据库表: _feed (订阅)
    │
    ├── category 外键关联到 _category.id
    ├── cache_nbUnreads 缓存未读数
    ├── priority 优先级字段
    │
    ▼
数据库表: _category (分类)
    │
    └── id=1 为默认分类
```

### 5.3 统计聚合关系

未读数统计是**自底向上**聚合的：

1. **文章级**：`_entry.is_read = 0` 表示未读
2. **订阅级**：`_feed.cache_nbUnreads` 缓存该订阅的未读文章数
3. **分类级**：遍历分类下所有订阅，累加满足优先级条件的未读数
4. **全局级**：遍历所有分类，累加满足优先级条件的未读数

**优先级过滤**：
- 主流视图 (`total_unread`)：只统计 `priority >= PRIORITY_MAIN_STREAM (10)` 的订阅
- 重要视图 (`total_important_unread`)：只统计 `priority >= PRIORITY_IMPORTANT (20)` 的订阅
- 分类视图 (`nbNotRead()`)：默认统计 `priority >= PRIORITY_FEED (-5)` 的订阅

### 5.4 移动订阅对统计的影响

移动订阅不会改变文章的已读/未读状态，但会影响分类级别的统计：

```
移动前:
  分类A: 10 未读 (包含订阅X的 3 未读)
  分类B: 5 未读

移动订阅X从A到B后:
  分类A: 7 未读 (减少 3)
  分类B: 8 未读 (增加 3)
  全局总数: 15 未读 (不变)
```

由于分类的未读数是通过其订阅聚合的，移动订阅后不需要额外操作，下一次查询时会自动反映新的统计结果。

### 5.5 默认分类的特殊角色

默认分类在整个系统中扮演着"回收站"和"安全网"的角色：

1. **新订阅默认归类**：添加订阅时如果未指定分类，放入默认分类
2. **删除分类的归宿**：删除分类时，该分类下所有订阅被移到默认分类
3. **无效分类的兜底**：移动订阅时如果目标分类不存在，使用默认分类
4. **不可删除**：默认分类不能被删除，保证系统至少有一个分类

### 5.6 缓存与性能优化

为了避免频繁的数据库查询，系统使用了多层缓存：

1. **数据库缓存**：`_feed.cache_nbUnreads` 和 `_feed.cache_nbEntries`
2. **对象缓存**：`FreshRSS_Feed::$nbNotRead` 和 `FreshRSS_Category::$nbNotRead`
3. **上下文缓存**：`FreshRSS_Context::$categories` 和 `FreshRSS_Context::$total_unread`

缓存更新时机：
- 刷新订阅后 (`actualizeFeedsAndCommit`)
- 标记文章已读后
- 清理旧文章后
- 新增/删除文章后

---

## 六、关键代码文件索引

| 功能 | 文件 | 关键方法/行号 |
|------|------|--------------|
| 分类模型 | [Category.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php) | `isDefault()`, `nbNotRead()`, `feeds()` |
| 分类DAO | [CategoryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/CategoryDAO.php) | `DEFAULTCATEGORYID`, `checkDefault()`, `listCategories()`, `countNotRead()` |
| 订阅模型 | [Feed.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Feed.php) | `category()`, `nbNotRead()`, `priority()` |
| 订阅DAO | [FeedDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/FeedDAO.php) | `changeCategory()`, `updateFeed()`, `updateCachedValues()` |
| 上下文 | [Context.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Context.php) | `categories()`, `updateUsingRequest()`, `$total_unread` |
| 分类控制器 | [categoryController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/categoryController.php) | `deleteAction()`, `createAction()` |
| 订阅控制器 | [feedController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/feedController.php) | `moveFeed()`, `moveAction()`, `addFeed()` |
| 订阅管理控制器 | [subscriptionController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/subscriptionController.php) | `feedAction()` |
| 侧边栏视图 | [aside_feed.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml) | 分类树渲染 |
