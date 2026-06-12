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
| `attributes` | array | JSON 扩展属性，包含 `position`、`show_unread_count`、`archiving`、`defaultSort`、`defaultOrder` 等 |

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
| `attributes` | array | JSON 扩展属性，包含 `show_unread_count`、`defaultSort`、`defaultOrder` 等 |

**优先级常量**：
- `PRIORITY_IMPORTANT = 20` - 重要订阅
- `PRIORITY_MAIN_STREAM = 10` - 主流订阅（默认）
- `PRIORITY_CATEGORY = 0` - 分类级别
- `PRIORITY_FEED = -5` - 普通订阅
- `PRIORITY_HIDDEN = -10` - 隐藏/归档订阅

---

## 二、分类树的排序规则

### 2.1 分类间排序

**核心方法**: `FreshRSS_CategoryDAO::listSortedCategories()`

[CategoryDAO.php#L306-L323](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/CategoryDAO.php#L306-L323)

```php
public function listSortedCategories(bool $prePopulateFeeds = true, bool $details = false): array {
    $categories = $this->listCategories($prePopulateFeeds, $details);

    uasort($categories, static function (FreshRSS_Category $a, FreshRSS_Category $b) {
        $aPosition = $a->attributeInt('position');
        $bPosition = $b->attributeInt('position');
        if ($aPosition === $bPosition) {
            return strnatcasecmp($a->name(), $b->name());
        } elseif (null === $aPosition) {
            return 1;
        } elseif (null === $bPosition) {
            return -1;
        }
        return ($aPosition < $bPosition) ? -1 : 1;
    });

    return $categories;
}
```

分类排序规则（按优先级递减）：

1. **有 `position` 属性的分类**：按 position 数值升序排列（position 越小越靠前）
2. **没有 `position` 属性的分类**（`attributeInt('position')` 返回 `null`）：排在所有有 position 的分类之后
3. **position 相同或均无 position**：按分类名称自然排序（`strnatcasecmp`，不区分大小写）

`position` 属性存储在分类的 `attributes` JSON 字段中，用户在分类设置页面可以手动指定。

设置 position 的代码位于分类更新控制器：

[categoryController.php#L146-L147](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/categoryController.php#L146-L147)

```php
$position = Minz_Request::paramInt('position') ?: null;
$category->_attribute('position', $position);
```

当 `position` 为 0 或空时，存为 `null`，表示不指定位置，退回到按名称排序。

### 2.2 订阅间排序

分类内订阅的排序规则是**固定的按名称自然排序**，不提供用户自定义排序。

排序发生在两个位置：

**位置一：Category 对象内部排序**

[Category.php#L298-L303](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php#L298-L303)

```php
private function sortFeeds(): void {
    if ($this->feeds === null) {
        return;
    }
    uasort($this->feeds, static fn(FreshRSS_Feed $a, FreshRSS_Feed $b) => strnatcasecmp($a->name(), $b->name()));
}
```

此方法在以下时机被调用：
- `_feeds()` 赋值时（[Category.php#L176](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php#L176)）
- `feeds()` 延迟加载时（[Category.php#L144](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php#L144)）
- `addFeed()` 添加订阅时（[Category.php#L201](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php#L201)）

**位置二：FeedDAO 查询时排序**

[FeedDAO.php#L540](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/FeedDAO.php#L540)

```php
uasort($feeds, static fn(FreshRSS_Feed $a, FreshRSS_Feed $b) => strnatcasecmp($a->name(), $b->name()));
```

在 `listByCategory()` 方法中，查询出订阅后同样按名称排序。

**位置三：SQL 预加载排序**

[CategoryDAO.php#L337](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/CategoryDAO.php#L337)

```sql
ORDER BY c.name, f.name
```

在 `listCategories()` 预加载订阅时，SQL 已经按 feed 名称排序。但这个排序在后续 `listSortedCategories()` 的 `uasort` 后会被覆盖（因为分类间排序会打乱），所以 Category 的 `sortFeeds()` 仍需重新排序。

### 2.3 侧边栏中的分类显示位置

在侧边栏视图 [aside_feed.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml) 中，分类的显示位置由 `listSortedCategories()` 的排序结果决定。

侧边栏结构（从上到下）：

1. **全部文章**（固定顶部）- `total_unread`
2. **重要订阅**（固定）- `total_important_unread`
3. **收藏文章**（固定）- `total_starred['unread']`
4. **标签**（可折叠）- 各标签及未读数
5. **用户分类列表**（按 `position` + 名称排序）- 各分类及其下订阅
6. **底部占位** - `<li class="tree-bottom">`

每个分类 `<li>` 上会记录 `data-position` 属性：

[aside_feed.phtml#L111-L112](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml#L111-L112)

```php
<li id="c_<?= $cat->id() ?>" class="tree-folder category<?= $c_active ? ' active' : '' ?>"<?=
    null === $position ? '' : " data-position='$position'" ?> data-unread="<?= $cat->nbNotRead() ?>"<?= $hideSucCat ?>>
```

该 `data-position` 可供前端 JavaScript 做拖拽排序等交互使用。

分类的展开/折叠由 `display_categories` 用户配置控制：

[aside_feed.phtml#L107-L108](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml#L107-L108)

```php
$c_show = ($c_active && in_array(FreshRSS_Context::userConf()->display_categories, ['active', 'remember'], true))
    || FreshRSS_Context::userConf()->display_categories === 'all';
```

`display_categories` 的可选值（默认 `'active'`）：
- `'all'` - 始终展开所有分类
- `'active'` - 仅展开当前选中的分类
- `'remember'` - 展开当前选中的分类（记忆上次的展开状态）
- `'none'` - 折叠所有分类

---

## 三、隐藏订阅的显隐规则

### 3.1 优先级与显隐的对应关系

FreshRSS 的订阅优先级决定了订阅在不同视图中的可见性：

| 优先级常量 | 值 | 含义 | 侧边栏可见性 | 主视图可见性 |
|------------|-----|------|-------------|-------------|
| `PRIORITY_IMPORTANT` | 20 | 重要 | 始终显示 | 显示 |
| `PRIORITY_MAIN_STREAM` | 10 | 主流（默认） | 显示 | 显示 |
| `PRIORITY_CATEGORY` | 0 | 仅分类 | 显示 | 不显示 |
| `PRIORITY_FEED` | -5 | 普通 | 显示 | 不显示 |
| `PRIORITY_HIDDEN` | -10 | 隐藏 | 条件显示 | 不显示 |

### 3.2 侧边栏中的隐藏订阅显示条件

**核心过滤逻辑**: [aside_feed.phtml#L128-L131](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml#L128-L131)

```php
foreach ($feeds as $feed):
    $f_active = FreshRSS_Context::isCurrentGet('f_' . $feed->id());
    if (!$f_active && $feed->priority() < FreshRSS_Feed::PRIORITY_FEED) {
        continue;
    }
```

这段代码的逻辑是：**如果一个订阅的优先级低于 `PRIORITY_FEED`（即 `PRIORITY_HIDDEN = -10`），且该订阅不是当前正在查看的订阅，则在侧边栏中跳过不显示。**

也就是说，隐藏订阅（`PRIORITY_HIDDEN`）**在以下条件下仍会出现在侧边栏**：

1. **用户正在查看该隐藏订阅**：当 `$f_active === true` 时，即使优先级为 `PRIORITY_HIDDEN`，该订阅仍然显示在侧边栏中。这确保用户能通过侧边栏操作当前正在查看的订阅。
2. **优先级 >= `PRIORITY_FEED`（-5）的订阅**：始终显示，无论是否处于活跃状态。

这种设计的意图是：隐藏订阅通常不应占用侧边栏空间，但如果用户通过 URL 直接访问了该订阅（例如收藏了书签），侧边栏仍需提供导航入口。

### 3.3 "已读隐藏"功能

侧边栏还有一个"隐藏已读订阅"的全局选项：

[aside_feed.phtml#L6-L10](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml#L6-L10)

```php
if (FreshRSS_Context::userConf()->hide_read_feeds &&
    (FreshRSS_Context::isStateEnabled(FreshRSS_Entry::STATE_NOT_READ) || FreshRSS_Context::isStateEnabled(FreshRSS_Entry::STATE_OR_NOT_READ)) &&
    !FreshRSS_Context::isStateEnabled(FreshRSS_Entry::STATE_READ)) {
    $class = ' state_unread';
}
```

当用户开启 `hide_read_feeds`（默认 `true`）且当前筛选条件为"仅未读"时，侧边栏整体添加 `state_unread` CSS 类。前端 CSS 通过此类名隐藏没有未读文章的分类和订阅，这是通过 CSS 而非 PHP 实现的过滤。

### 3.4 未读数显示控制的三级体系

未读数标记 `data-unread-hide="1"` 的添加逻辑：

**全局级别**：

[aside_feed.phtml#L23-L24](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml#L23-L24)

```php
$hideSucGlobal = FreshRSS_Context::userConf()->show_unread_count !== 'all' ? ' data-unread-hide="1"' : '';
$hideSucImportant = FreshRSS_Context::userConf()->show_unread_count !== 'none' ? '' : ' data-unread-hide="1"';
```

**分类级别**：

[Category.php#L121-L123](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php#L121-L123)

```php
public function showUnreadCount(): bool {
    return $this->attributeBoolean('show_unread_count') ??
        (FreshRSS_Context::userConf()->show_unread_count === 'all');
}
```

**订阅级别**：

[Feed.php#L321-L330](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Feed.php#L321-L330)

```php
public function showUnreadCount(): bool {
    $sucGlobal = FreshRSS_Context::userConf()->show_unread_count;
    $isImportant = $this->priority >= self::PRIORITY_IMPORTANT;
    if ($isImportant && $sucGlobal !== 'none') {
        return true;
    }
    return $this->attributeBoolean('show_unread_count') ??
        $this->category()?->attributeBoolean('show_unread_count') ??
        ($sucGlobal === 'all' || ($sucGlobal === 'important' && $isImportant));
}
```

三级显示控制汇总：

| 全局设置 | 分类属性 | 订阅属性 | 最终结果 |
|---------|---------|---------|---------|
| `all` | 未设置 | 未设置 | 显示 |
| `all` | `true` | 未设置 | 显示 |
| `all` | `false` | 未设置 | 隐藏 |
| `all` | `false` | `true` | 显示 |
| `important` | 未设置 | 未设置 | 仅重要订阅显示 |
| `important` | `true` | 未设置 | 显示（分类级覆盖） |
| `important` | `false` | 未设置 | 隐藏 |
| `none` | 未设置 | 未设置 | 都不显示 |
| `none` | 未设置 | 未设置 | 重要订阅仍显示 |
| `none` | `true` | 未设置 | 显示（分类级覆盖） |

**特殊规则**：重要订阅（`priority >= 20`）在全局设置非 `none` 时始终显示未读数，不受分类和订阅级属性覆盖。

### 3.5 隐藏订阅的统计贡献

隐藏订阅的未读数**不计入分类级和全局统计**：

[Category.php#L39-L47](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php#L39-L47)

```php
foreach ($feeds as $feed) {
    $feed->_category($this);
    $this->nbFeeds++;
    if ($feed->priority() > FreshRSS_Feed::PRIORITY_HIDDEN) {
        $this->nbNotRead += $feed->nbNotRead();
        $this->hasFeedsWithError |= ($feed->inError() && !$feed->mute());
    }
}
```

条件是 `priority > PRIORITY_HIDDEN`（即 > -10），意味着优先级为 `PRIORITY_HIDDEN`（-10）的订阅不贡献未读数。但 `PRIORITY_FEED`（-5）及以上的订阅仍然贡献。

---

## 四、订阅移动后分类内顺序为何仍留有待处理

### 4.1 代码中的 TODO 标注

在 `moveAction()` 方法中有一个明确的 TODO：

[feedController.php#L1081](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/feedController.php#L1081)

```php
/**
 * @todo should handle order of the feed inside the category.
 */
public function moveAction(): void {
```

### 4.2 问题分析

移动订阅的操作本身很简单——只更新了 `_feed` 表的 `category` 字段：

[feedController.php#L1068](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/feedController.php#L1068)

```php
return $feedDAO->updateFeed($feed_id, ['category' => $cat_id]);
```

`updateFeed()` 方法是一个通用的字段更新方法，它只会 SET 传入的字段，不会处理排序：

[FeedDAO.php#L167-L213](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/FeedDAO.php#L167-L213)

```php
public function updateFeed(int $id, array $valuesTmp): bool {
    // ... 仅 SET 传入的字段
    $sql = "UPDATE `_feed` SET {$set} WHERE id=?";
    // ...
}
```

### 4.3 当前排序的实际处理方式

移动订阅后，**分类内订阅的排序不是在移动操作时处理的**，而是在下次加载分类树时重新排序：

1. `FreshRSS_Context::categories()` 调用 `listSortedCategories()`
2. `listSortedCategories()` 调用 `listCategories()` 预加载数据
3. 预加载后，每个 Category 对象的 `feeds` 数组通过 `daoToCategoriesPrepopulated()` 构建
4. 在 `Category::_feeds()` 赋值时调用 `sortFeeds()` 按名称排序

[Category.php#L170-L177](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php#L170-L177)

```php
public function _feeds(array|FreshRSS_Feed $values): void {
    if (!is_array($values)) {
        $values = [$values];
    }
    $this->feeds = array_values($values);
    $this->sortFeeds();
}
```

因此，**移动后的订阅在下次页面加载时会自动按名称排序归位**，不需要额外的排序操作。

### 4.4 为何仍有 TODO

问题在于：FreshRSS **没有提供订阅级别的 position 属性**来支持用户自定义订阅顺序。

对比分类排序：分类有 `attributes.position` 字段，用户可以手动指定分类的显示顺序。但订阅没有类似的 position 机制——所有订阅都是固定的按名称排序。

这意味着：

1. **用户无法自定义订阅在分类内的显示顺序**——只能按名称字母序
2. **移动订阅后，如果用户期望订阅出现在分类的特定位置**（如最顶部或最底部），当前机制无法满足
3. TODO 的意图是：为 `moveAction()` 添加类似分类 position 的支持，让移动操作可以指定订阅在目标分类中的位置

目前，订阅的 `attributes` JSON 字段中只有 `defaultSort` 和 `defaultOrder`，它们控制的是**文章列表**的排序，而非订阅自身在侧边栏中的排序。

### 4.5 相关的用户配置

虽然不能自定义订阅位置，但有一个相关的简化配置：

[config-user.default.php#L112](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/config-user.default.php#L112)

```php
'simplify_over_n_feeds' => 1000,
```

当订阅总数超过此阈值时，侧边栏会简化显示（隐藏 favicon、隐藏配置下拉菜单等），以提升性能。

---

## 五、默认分类在中文界面的名称来源

### 5.1 名称的运行时覆盖机制

默认分类的名称有一个特殊机制：**数据库中存储的名称与界面显示的名称是分离的**。

当设置分类 ID 为默认分类 ID 时，名称会被强制覆盖为翻译值：

[Category.php#L153-L158](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php#L153-L158)

```php
public function _id(int $id): void {
    $this->id = $id;
    if ($id === FreshRSS_CategoryDAO::DEFAULTCATEGORYID) {
        $this->name = _t('gen.short.default_category');
    }
}
```

同时，`_name()` 方法会忽略对默认分类的名称修改：

[Category.php#L164-L168](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php#L164-L168)

```php
public function _name(string $value): void {
    if ($this->id !== FreshRSS_CategoryDAO::DEFAULTCATEGORYID) {
        $this->name = mb_strcut(trim($value), 0, FreshRSS_DatabaseDAO::LENGTH_INDEX_UNICODE, 'UTF-8');
    }
}
```

这意味着：
- 无论数据库中存储的名称是什么，运行时都会替换为当前语言的翻译值
- 用户无法修改默认分类的名称
- 语言切换后，默认分类的名称会自动跟随变化

### 5.2 翻译键的解析路径

翻译键 `gen.short.default_category` 的解析过程：

1. `_t()` 函数（[Translate.php#L432](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/lib/Minz/Translate.php#L432)）是 `Minz_Translate::t()` 的别名
2. `t()` 方法调用 `resolveKey()` 解析键名

[Translate.php#L257-L292](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/lib/Minz/Translate.php#L257-L292)

```php
private static function resolveKey(string $key): array|string|null {
    $group = explode('.', $key);
    // 'gen.short.default_category' => $group = ['gen', 'short', 'default_category']
    $top_level = array_shift($group);  // 'gen'
    // 加载 i18n/zh-CN/gen.php 文件
    self::loadKey($top_level);
    // 在翻译数组中逐级查找: $translates['gen']['short']['default_category']
    foreach ($group as $i18n_level) {
        $translationValue = $translationValue[$i18n_level];
    }
    return $translationValue;
}
```

3. 根据当前语言加载对应的翻译文件

[Translate.php#L158-L196](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/lib/Minz/Translate.php#L158-L196)

```php
private static function loadLang(string $path): void {
    $selected_lang_path = $path . '/' . self::$lang_name;
    // 如果当前语言目录不存在，回退到英语
    if (!$uses_selected_language) {
        $lang_path = $path . '/en';
    }
    // 扫描目录下的 .php 文件
    foreach ($list_i18n_files as $i18n_filename) {
        $i18n_key = basename($i18n_filename, '.php');  // 如 'gen'
        self::$lang_files[$i18n_key][] = $lang_path . '/' . $i18n_filename;
    }
}
```

### 5.3 各语言的翻译值

**英语**（默认）：

[en/gen.php#L320](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/i18n/en/gen.php#L320)

```php
'default_category' => 'Uncategorized',
```

**简体中文**：

[zh-CN/gen.php#L314](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/i18n/zh-CN/gen.php#L314)

```php
'default_category' => '未分类',
```

**繁体中文**：

[zh-TW/gen.php#L314](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/i18n/zh-TW/gen.php#L314)

```php
'default_category' => '未分類',
```

翻译文件的结构：每个 i18n 子目录下的 `gen.php` 对应 `gen.*` 翻译键，文件返回一个嵌套 PHP 数组，键名 `short` 对应数组中的 `short` 子键，`default_category` 对应最终的翻译值。

### 5.4 数据库中的名称与显示名称的差异

数据库中默认分类的名称存储有一个历史演变：

- **旧版本**：`resetDefaultCategoryName()` 方法会将数据库中的名称重置为硬编码的 `'Uncategorized'`

[CategoryDAO.php#L13-L20](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/CategoryDAO.php#L13-L20)

```php
public function resetDefaultCategoryName(): bool {
    $stm = $this->pdo->prepare('UPDATE `_category` SET name = :name WHERE id = :id');
    return $stm !== false &&
        $stm->bindValue(':id', self::DEFAULTCATEGORYID, PDO::PARAM_INT) &&
        $stm->bindValue(':name', self::DEFAULT_CATEGORY_NAME) &&  // 'Uncategorized'
        $stm->execute();
}
```

- **初始化时**：`checkDefault()` 创建默认分类时使用翻译值

[CategoryDAO.php#L407](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/CategoryDAO.php#L407)

```php
$cat = new FreshRSS_Category(_t('gen.short.default_category'), self::DEFAULTCATEGORYID);
```

这意味着数据库中存储的名称可能是英文 `'Uncategorized'`，也可能是创建时的语言翻译值，但**无论数据库中存储什么，运行时都会被 `_id()` 方法覆盖为当前语言的翻译值**。

### 5.5 名称覆盖的完整流程图

```
数据库 _category 表
  id=1, name='Uncategorized'  (或创建时的翻译值)
        │
        │  查询
        ▼
  CategoryDAO::searchById() / listCategories()
        │
        │  构建 Category 对象
        │  调用 _id(1)
        ▼
  Category::_id(1)
        │
        │  id === DEFAULTCATEGORYID → 覆盖 name
        ▼
  $this->name = _t('gen.short.default_category')
        │
        │  翻译键解析
        ▼
  当前语言为 zh-CN → '未分类'
  当前语言为 en    → 'Uncategorized'
  当前语言为 zh-TW → '未分類'
        │
        │  Category::_name() 被跳过
        ▼
  最终显示: 当前语言的翻译值
```

---

## 六、侧边栏统计机制

### 6.1 统计数据结构

**核心文件**: [Context.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Context.php)

全局统计变量：

| 变量 | 类型 | 说明 |
|------|------|------|
| `$total_unread` | int | 主视图未读总数（仅统计 PRIORITY_MAIN_STREAM 及以上） |
| `$total_important_unread` | int | 重要订阅未读数（统计 PRIORITY_IMPORTANT 及以上） |
| `$total_starred` | array | 收藏文章统计（all/read/unread） |
| `$get_unread` | int | 当前视图的未读数 |

### 6.2 统计数据初始化

**核心方法**: `FreshRSS_Context::updateUsingRequest()`

[Context.php#L239-L246](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Context.php#L239-L246)

```php
public static function updateUsingRequest(bool $computeStatistics): void {
    if ($computeStatistics && self::$total_unread === 0) {
        $entryDAO = FreshRSS_Factory::createEntryDao();
        self::$total_starred = $entryDAO->countUnreadReadFavorites();
        self::$total_unread = FreshRSS_Category::countUnread(self::categories(), FreshRSS_Feed::PRIORITY_MAIN_STREAM);
        self::$total_important_unread = FreshRSS_Category::countUnread(self::categories(), FreshRSS_Feed::PRIORITY_IMPORTANT);
    }
}
```

### 6.3 分类级别的未读统计

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

### 6.4 单个分类的未读统计

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

### 6.5 单个订阅的未读统计

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

### 6.6 数据库级别的缓存

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
}
```

该方法在以下时机被调用：
- 刷新订阅后
- 提交新文章后
- 标记已读后

### 6.7 侧边栏视图渲染

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

### 6.8 分类列表预加载

**核心方法**: `FreshRSS_CategoryDAO::listCategories()`

[CategoryDAO.php#L326-L358](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/CategoryDAO.php#L326-L358)

当 `prePopulateFeeds = true` 时，通过 LEFT JOIN 一次性查询所有分类及其订阅，避免 N+1 查询问题：

```sql
SELECT c.id AS c_id, c.name AS c_name, ..., f.id, f.name, ...
FROM `_category` c
LEFT OUTER JOIN `_feed` f ON f.category=c.id
GROUP BY f.id, c_id
ORDER BY c.name, f.name
```

---

## 七、三者之间的关系

### 7.1 层级结构关系

```
侧边栏 (Sidebar)
  ├── 全部文章 (total_unread)                    ← PRIORITY_MAIN_STREAM 及以上
  ├── 重要订阅 (total_important_unread)           ← PRIORITY_IMPORTANT 及以上
  ├── 收藏文章
  ├── 标签
  │     └── 标签1 (nbUnread)
  └── 分类1 (Category) ── nbNotRead()            ← 按 position + 名称排序
        ├── 订阅1 (Feed) ── nbNotRead()           ← 按名称排序
        ├── 订阅2 (Feed) ── nbNotRead()
        └── [隐藏订阅] ── 仅当前查看时显示
  └── 默认分类 (DEFAULTCATEGORYID=1) ── nbNotRead()  ← 名称始终为翻译值
        ├── 订阅3 (Feed) ── nbNotRead()
        └── ...
```

### 7.2 数据流关系

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
    ├── attributes JSON (show_unread_count, defaultSort, defaultOrder)
    │
    ▼
数据库表: _category (分类)
    │
    ├── id=1 为默认分类
    └── attributes JSON (position, show_unread_count, archiving, defaultSort, defaultOrder)
```

### 7.3 统计聚合关系

未读数统计是**自底向上**聚合的：

1. **文章级**：`_entry.is_read = 0` 表示未读
2. **订阅级**：`_feed.cache_nbUnreads` 缓存该订阅的未读文章数
3. **分类级**：遍历分类下所有订阅，累加满足优先级条件的未读数
4. **全局级**：遍历所有分类，累加满足优先级条件的未读数

**优先级过滤**：
- 主流视图 (`total_unread`)：只统计 `priority >= PRIORITY_MAIN_STREAM (10)` 的订阅
- 重要视图 (`total_important_unread`)：只统计 `priority >= PRIORITY_IMPORTANT (20)` 的订阅
- 分类视图 (`nbNotRead()`)：默认统计 `priority >= PRIORITY_FEED (-5)` 的订阅
- 隐藏订阅 (`PRIORITY_HIDDEN = -10`)：不贡献任何统计

### 7.4 移动订阅对统计的影响

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

### 7.5 默认分类的特殊角色

默认分类在整个系统中扮演着"回收站"和"安全网"的角色：

1. **新订阅默认归类**：添加订阅时如果未指定分类，放入默认分类
2. **删除分类的归宿**：删除分类时，该分类下所有订阅被移到默认分类
3. **无效分类的兜底**：移动订阅时如果目标分类不存在，使用默认分类
4. **不可删除**：默认分类不能被删除，保证系统至少有一个分类
5. **名称不可修改**：默认分类的名称始终使用翻译值，用户无法自定义

### 7.6 缓存与性能优化

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

## 八、关键代码文件索引

| 功能 | 文件 | 关键方法/行号 |
|------|------|--------------|
| 分类模型 | [Category.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php) | `isDefault()`, `nbNotRead()`, `feeds()`, `sortFeeds()`, `_id()`, `_name()` |
| 分类DAO | [CategoryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/CategoryDAO.php) | `DEFAULTCATEGORYID`, `checkDefault()`, `listSortedCategories()`, `listCategories()`, `countNotRead()`, `resetDefaultCategoryName()` |
| 订阅模型 | [Feed.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Feed.php) | `category()`, `nbNotRead()`, `priority()`, `showUnreadCount()` |
| 订阅DAO | [FeedDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/FeedDAO.php) | `changeCategory()`, `updateFeed()`, `updateCachedValues()`, `listByCategory()` |
| 属性Trait | [AttributesTrait.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/AttributesTrait.php) | `attributeInt()`, `attributeBoolean()`, `attributeString()`, `_attribute()` |
| 上下文 | [Context.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Context.php) | `categories()`, `updateUsingRequest()`, `$total_unread` |
| 分类控制器 | [categoryController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/categoryController.php) | `deleteAction()`, `updateAction()`（position 设置） |
| 订阅控制器 | [feedController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/feedController.php) | `moveFeed()`, `moveAction()`（含 TODO）, `addFeed()` |
| 订阅管理控制器 | [subscriptionController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/subscriptionController.php) | `feedAction()` |
| 侧边栏视图 | [aside_feed.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml) | 分类树渲染、隐藏过滤、未读数显示控制 |
| 翻译系统 | [Translate.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/lib/Minz/Translate.php) | `t()`, `resolveKey()`, `loadKey()` |
| 中文翻译 | [zh-CN/gen.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/i18n/zh-CN/gen.php) | `'default_category' => '未分类'` |
| 英语翻译 | [en/gen.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/i18n/en/gen.php) | `'default_category' => 'Uncategorized'` |
| 用户默认配置 | [config-user.default.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/config-user.default.php) | `show_unread_count`, `display_categories`, `hide_read_feeds` |
