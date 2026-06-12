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

排序发生在三个位置：

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

#### 2.4.1 `data-position` 的真实用途

经过对整个代码库的全面搜索（`**/*.js` 和 `**/*.php`），`data-position` 属性仅在 [aside_feed.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml) 中被输出到 HTML，在 FreshRSS 核心的 JavaScript 和 PHP 代码中均无任何读取或使用的引用。

其真实用途是：
- 作为 HTML5 `data-*` 自定义属性，将分类的 `attributes.position` 整数值暴露给前端 DOM
- 供第三方前端扩展（如自定义拖拽排序插件）读取和使用
- FreshRSS 核心代码当前未直接依赖该属性，属于预留的扩展钩子

---

## 三、分类侧边栏显隐链路

分类侧边栏的展开/折叠状态由 **PHP 后端**和 **JS 前端**协同控制，形成完整的显隐链路。

### 3.1 `display_categories` 配置

**核心配置**：`display_categories` 用户配置决定分类的默认展开行为。

[config-user.default.php#L41](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/config-user.default.php#L41)

```php
'display_categories' => 'active',    // { active, remember, all, none }
```

可选值说明：

| 值 | 说明 |
|----|------|
| `'all'` | 始终展开所有分类 |
| `'active'` | 仅展开当前选中的分类（默认） |
| `'remember'` | 展开当前选中的分类 + 记忆用户手动展开/折叠的状态 |
| `'none'` | 折叠所有分类 |

配置在上下文初始化时进行合法性校验：

[Context.php#L157-L158](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Context.php#L157-L158)

```php
if (!in_array(FreshRSS_Context::$user_conf->display_categories, [ 'active', 'remember', 'all', 'none' ], true)) {
    FreshRSS_Context::$user_conf->display_categories = FreshRSS_Context::$user_conf->display_categories === true ? 'all' : 'active';
}
```

### 3.2 PHP 端：初始渲染的展开状态

PHP 端在渲染侧边栏时决定每个分类的初始展开状态。

**分类展开判断逻辑**：

[aside_feed.phtml#L106-L108](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml#L106-L108)

```php
$c_active = FreshRSS_Context::isCurrentGet('c_' . $cat->id());
$c_show = ($c_active && in_array(FreshRSS_Context::userConf()->display_categories, ['active', 'remember'], true))
    || FreshRSS_Context::userConf()->display_categories === 'all';
```

逻辑拆解：

| `display_categories` | 当前分类是否选中 | `$c_show` 结果 | 说明 |
|---------------------|-----------------|---------------|------|
| `'all'` | 任意 | `true` | 始终展开所有分类 |
| `'active'` | 是 | `true` | 展开当前选中的分类 |
| `'active'` | 否 | `false` | 折叠其他分类 |
| `'remember'` | 是 | `true` | 展开当前选中的分类 |
| `'remember'` | 否 | `false` | 其他分类初始折叠，后续由 JS 恢复 |
| `'none'` | 任意 | `false` | 所有分类初始折叠 |

**标签的展开判断逻辑**（与分类一致）：

[aside_feed.phtml#L66-L67](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml#L66-L67)

```php
$t_active = FreshRSS_Context::isCurrentGet('T');
$t_show = ($t_active && in_array(FreshRSS_Context::userConf()->display_categories, ['active', 'remember'], true)) || FreshRSS_Context::userConf()->display_categories === 'all';
```

**渲染展开状态**：

通过 `<ul class="tree-folder-items">` 是否添加 `active` 类来控制展开/折叠：

[aside_feed.phtml#L122](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml#L122)

```php
<ul class="tree-folder-items<?= $c_show ? ' active' : '' ?>">
```

展开/折叠按钮的图标也由 `$c_show` 决定：

[aside_feed.phtml#L114](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml#L114)

```php
<button class="dropdown-toggle" title="<?= _t('sub.category.expand') ?>"><?= _i($c_show ? 'up' : 'down') ?></button>
```

### 3.3 JS 端：用户交互的状态切换

当用户点击分类的展开/折叠按钮时，由前端 JS 处理状态切换。

**点击事件处理**：

[main.js#L1093-L1126](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/p/scripts/main.js#L1093-L1126)

```javascript
asideFeed.onclick = function (ev) {
    let a = ev.target.closest('.tree-folder > .tree-folder-title > button.dropdown-toggle');
    if (a) {
        const icon = a.querySelector('.icon');
        const category_id = a.closest('.category').id;
        if (icon.alt === '🔽' || icon.innerHTML === '🔽') {
            // 展开操作
            if (icon.src) {
                icon.src = icon.src.replace('/icons/down.', '/icons/up.');
                icon.alt = '🔼';
            } else {
                icon.innerHTML = '🔼';
            }
            rememberOpenCategory(category_id, true);  // 记住状态
        } else {
            // 折叠操作
            if (icon.src) {
                icon.src = icon.src.replace('/icons/up.', '/icons/down.');
                icon.alt = '🔽';
            } else {
                icon.innerHTML = '🔽';
            }
            rememberOpenCategory(category_id, false);  // 记住状态
        }

        const ul = a.closest('li').querySelector('.tree-folder-items');
        // 计算可见项数量用于 CSS transition 动画
        let nbVisibleItems = 0;
        for (let i = ul.children.length - 1; i >= 0; i--) {
            if (ul.children[i].offsetHeight) {
                nbVisibleItems++;
            }
        }
        ul.classList.toggle('active');
        // CSS transition 不支持 max-height:auto，需手动设置
        ul.style.maxHeight = ul.classList.contains('active') ? (nbVisibleItems * 4) + 'em' : 0;
        return false;
    }
}
```

**状态切换的完整流程**：

```
用户点击折叠按钮
    │
    ▼
判断当前状态 (icon.alt 或 innerHTML)
    │
    ├─ 折叠状态 (🔽) → 切换为展开状态
    │       ├─ 替换图标 src 为 up
    │       ├─ 设置 alt 为 🔼
    │       └─ 调用 rememberOpenCategory(id, true)
    │
    └─ 展开状态 (🔼) → 切换为折叠状态
            ├─ 替换图标 src 为 down
            ├─ 设置 alt 为 🔽
            └─ 调用 rememberOpenCategory(id, false)
    │
    ▼
切换 .tree-folder-items 的 active 类
    │
    ▼
设置 max-height 实现过渡动画
```

### 3.4 `remember` 模式：记忆分类展开状态

当 `display_categories` 设置为 `'remember'` 时，系统会记住用户手动展开/折叠的分类状态。

**配置传递到前端**：

PHP 端通过 `javascript_vars.phtml` 将配置编码为 JSON 传递给前端 JS：

[javascript_vars.phtml#L17](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/views/helpers/javascript_vars.phtml#L17)

```php
'display_categories' => FreshRSS_Context::userConf()->display_categories,
```

该变量被赋值给 JS 的 `context.display_categories`。

**记忆状态的存储**：

[main.js#L1025-L1035](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/p/scripts/main.js#L1025-L1035)

```javascript
function rememberOpenCategory(category_id, isOpen) {
    if (context.display_categories === 'remember') {
        const open_categories = JSON.parse(localStorage.getItem('FreshRSS_open_categories') || '{}');
        if (isOpen) {
            open_categories[category_id] = true;
        } else {
            delete open_categories[category_id];
        }
        localStorage.setItem('FreshRSS_open_categories', JSON.stringify(open_categories));
    }
}
```

存储机制：
- **存储位置**：浏览器 `localStorage`（持久化存储，关闭浏览器后仍保留）
- **存储键名**：`FreshRSS_open_categories`
- **存储格式**：JSON 对象，键为分类 ID（如 `'c_1'`），值为 `true`（表示展开）
- **仅展开的分类被存储**，折叠的分类从对象中删除

**恢复记忆的展开状态**：

页面加载时，`init_column_categories()` 函数从 localStorage 恢复展开状态：

[main.js#L1058-L1069](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/p/scripts/main.js#L1058-L1069)

```javascript
function init_column_categories() {
    if (context.current_view !== 'normal' && context.current_view !== 'reader') {
        return;
    }

    // Restore open categories
    if (context.display_categories === 'remember') {
        const open_categories = JSON.parse(localStorage.getItem('FreshRSS_open_categories') || '{}');
        Object.keys(open_categories).forEach(function (category_id) {
            openCategory(category_id);
        });
    }
    // ... 后续的滚动位置恢复
}
```

恢复展开状态的 `openCategory()` 函数：

[main.js#L1037-L1045](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/p/scripts/main.js#L1037-L1045)

```javascript
function openCategory(category_id) {
    const category_element = document.getElementById(category_id);
    if (!category_element) return;
    category_element.querySelector('.tree-folder-items').classList.add('active');
    const img = category_element.querySelector('button.dropdown-toggle img');
    if (!img) return;
    img.src = img.src.replace('/icons/down.', '/icons/up.');
    img.alt = '🔼';
}
```

**清除记忆状态**：

当用户登出时，`extra.js` 会清除 localStorage 中的记忆状态：

[extra.js#L16](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/p/scripts/extra.js#L16)

```javascript
localStorage.removeItem('FreshRSS_open_categories');
```

### 3.5 显隐链路的完整时序图

```
页面请求
    │
    ▼
PHP 端: Context 初始化
    │  校验 display_categories 合法性
    │
    ▼
PHP 端: aside_feed.phtml 渲染
    │  计算 $c_active (当前选中的分类)
    │  计算 $c_show (初始展开状态)
    │  根据 $c_show 决定是否添加 'active' 类
    │  输出 HTML
    │
    ▼
浏览器: 解析 HTML + 加载 JS
    │
    ▼
JS 端: init_column_categories()
    │
    ├─ display_categories === 'remember'?
    │   ├─ 是: 从 localStorage 读取 FreshRSS_open_categories
    │   │    遍历对象，调用 openCategory() 展开每个分类
    │   └─ 否: 不做额外处理
    │
    ▼
JS 端: 绑定 click 事件
    │
    ▼
用户点击折叠按钮
    │
    ├─ 切换图标（up ↔ down）
    ├─ 切换 .tree-folder-items 的 active 类
    ├─ 设置 max-height 动画
    └─ display_categories === 'remember'?
        ├─ 是: 更新 localStorage 的 FreshRSS_open_categories
        └─ 否: 不存储
```

### 3.6 其他显隐控制

**"已读隐藏"功能（CSS 级过滤）**：

[aside_feed.phtml#L6-L10](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml#L6-L10)

```php
if (FreshRSS_Context::userConf()->hide_read_feeds &&
    (FreshRSS_Context::isStateEnabled(FreshRSS_Entry::STATE_NOT_READ) || FreshRSS_Context::isStateEnabled(FreshRSS_Entry::STATE_OR_NOT_READ)) &&
    !FreshRSS_Context::isStateEnabled(FreshRSS_Entry::STATE_READ)) {
    $class = ' state_unread';
}
```

当 `hide_read_feeds` 为 `true` 且当前筛选"仅未读"时，侧边栏添加 `state_unread` 类。CSS 通过此类名隐藏没有未读文章的分类和订阅。

---

## 四、隐藏订阅的显隐规则

### 4.1 优先级与显隐的对应关系

FreshRSS 的订阅优先级决定了订阅在不同视图中的可见性：

| 优先级常量 | 值 | 含义 | 侧边栏可见性 | 主视图可见性 |
|------------|-----|------|-------------|-------------|
| `PRIORITY_IMPORTANT` | 20 | 重要 | 始终显示 | 显示 |
| `PRIORITY_MAIN_STREAM` | 10 | 主流（默认） | 显示 | 显示 |
| `PRIORITY_CATEGORY` | 0 | 仅分类 | 显示 | 不显示 |
| `PRIORITY_FEED` | -5 | 普通 | 显示 | 不显示 |
| `PRIORITY_HIDDEN` | -10 | 隐藏 | 条件显示 | 不显示 |

### 4.2 侧边栏中的隐藏订阅显示条件

**核心过滤逻辑**: [aside_feed.phtml#L127-L131](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml#L127-L131)

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

### 4.3 隐藏订阅的统计贡献

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

## 五、未读数显示控制的三级体系

未读数显示控制分为**全局配置**、**分类级属性**、**订阅级属性**三级，通过 `data-unread-hide="1"` 属性标记隐藏。CSS 会将该属性的未读数替换为一个淡色的点。

**CSS 处理**：

[frss.css#L2310-L2316](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/p/themes/base-theme/frss.css#L2310-L2316)

```css
/* Faint dot shown in place of the count badge when [data-unread-hide] is present */
.aside .category .tree-folder-title .title[data-unread-hide]:not([data-unread="0"])::after,
.aside .feed .item-title[data-unread-hide]:not([data-unread="0"])::after {
    content: "·";
    opacity: 0.5;
    pointer-events: none;
}
```

### 5.1 全局级 `data-unread-hide` 标记

侧边栏顶部的全局导航项使用独立的判断逻辑：

[aside_feed.phtml#L23-L24](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml#L23-L24)

```php
$hideSucGlobal = FreshRSS_Context::userConf()->show_unread_count !== 'all' ? ' data-unread-hide="1"' : '';
$hideSucImportant = FreshRSS_Context::userConf()->show_unread_count !== 'none' ? '' : ' data-unread-hide="1"';
```

| 全局项 | 判断逻辑 | 说明 |
|--------|---------|------|
| 全部文章、收藏、标签、分类 | `show_unread_count !== 'all'` | 仅当全局设置为 `'all'` 时显示 |
| 重要订阅 | `show_unread_count !== 'none'` | 当全局设置不是 `'none'` 时显示 |

### 5.2 分类级 `showUnreadCount()`

**核心方法**：`FreshRSS_Category::showUnreadCount()`

[Category.php#L121-L124](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php#L121-L124)

```php
public function showUnreadCount(): bool {
    return $this->attributeBoolean('show_unread_count') ??
        (FreshRSS_Context::userConf()->show_unread_count === 'all');
}
```

**逻辑**：
1. 优先使用分类自身的 `attributes.show_unread_count` 属性（`true`/`false`）
2. 如果未设置（返回 `null`），回退到全局判断：`show_unread_count === 'all'`

| 分类属性 | 全局设置 | 结果 |
|---------|---------|------|
| `true` | 任意 | `true`（显示） |
| `false` | 任意 | `false`（隐藏） |
| 未设置 | `'all'` | `true`（显示） |
| 未设置 | `'important'` / `'none'` | `false`（隐藏） |

在视图中使用：

[aside_feed.phtml#L109](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml#L109)

```php
$hideSucCat = $cat->showUnreadCount() ? '' : ' data-unread-hide="1"';
```

### 5.3 订阅级 `showUnreadCount()`

**核心方法**：`FreshRSS_Feed::showUnreadCount()`

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

**完整逻辑解析**：

#### 第一步：Important 订阅特殊规则
```php
if ($isImportant && $sucGlobal !== 'none') {
    return true;
}
```
- **如果是重要订阅（priority >= 20）且全局设置不是 `'none'`** → 直接返回 `true`，强制显示未读数
- **重要订阅在 `sucGlobal === 'none'` 时不享受此特权**，会继续后续判断

#### 第二步：订阅级属性覆盖
```php
$this->attributeBoolean('show_unread_count') ??
```
- 如果订阅自身设置了 `attributes.show_unread_count`（`true` 或 `false`），使用该值

#### 第三步：分类级属性覆盖
```php
$this->category()?->attributeBoolean('show_unread_count') ??
```
- 如果订阅级未设置，使用所属分类的 `attributes.show_unread_count`
- 使用 `?->` 运算符防止分类为 `null` 的情况

#### 第四步：全局回退
```php
($sucGlobal === 'all' || ($sucGlobal === 'important' && $isImportant))
```
- 如果前三级都未设置（全部返回 `null`），使用全局判断
- `'all'` → 显示
- `'important'` 且是重要订阅 → 显示
- `'none'` → 不显示
- `'important'` 但不是重要订阅 → 不显示

### 5.4 重要订阅在 `none` 配置下的正确判断

**此前的误解需要纠正**：

| 全局设置 | 重要订阅？ | 第一步命中？ | 后续判断 | 最终结果 |
|---------|-----------|------------|---------|---------|
| `'all'` | 是 | 是（`sucGlobal !== 'none'`） | - | `true` ✅ |
| `'important'` | 是 | 是（`sucGlobal !== 'none'`） | - | `true` ✅ |
| `'none'` | 是 | **否**（`sucGlobal === 'none'`） | 进入后续判断 | 取决于属性设置 |

当 `show_unread_count === 'none'` 时，重要订阅的判断流程：
1. 第一步不命中（因为 `$sucGlobal !== 'none'` 为 `false`）
2. 检查订阅级属性：如果设置了 `true`/`false`，使用该值
3. 检查分类级属性：如果设置了 `true`/`false`，使用该值
4. 全局回退：`'none' === 'all'` → `false`，`'none' === 'important'` → `false`，返回 `false`

**结论**：当全局设置为 `'none'` 时，重要订阅**默认不显示**未读数，但可以通过订阅级或分类级的 `show_unread_count = true` 强制显示。

这与侧边栏"重要订阅"全局项的判断逻辑一致：
[aside_feed.phtml#L50-L53](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml#L50-L53)

```php
<a class="tree-folder-title" data-unread="<?= format_number(FreshRSS_Context::$total_important_unread) ?>"<?=
    $hideSucImportant ?> href="...">
```
其中 `$hideSucImportant` 在 `show_unread_count === 'none'` 时添加 `data-unread-hide="1"`。

### 5.5 三级覆盖关系完整真值表

| 全局设置 | 分类属性 | 订阅属性 | 是否 Important | 结果 | 说明 |
|---------|---------|---------|---------------|------|------|
| `'all'` | 未设置 | 未设置 | 否 | `true` | 全局 all 默认显示 |
| `'all'` | 未设置 | 未设置 | 是 | `true` | Important 特殊规则命中 |
| `'all'` | `false` | 未设置 | 否 | `false` | 分类级隐藏 |
| `'all'` | `false` | 未设置 | 是 | `true` | Important 特殊规则优先于分类级 |
| `'all'` | `false` | `true` | 否 | `true` | 订阅级显示优先于分类级 |
| `'all'` | `false` | `true` | 是 | `true` | Important 特殊规则命中 |
| `'important'` | 未设置 | 未设置 | 否 | `false` | 全局 important 模式，非 important 不显示 |
| `'important'` | 未设置 | 未设置 | 是 | `true` | Important 特殊规则命中 |
| `'important'` | `true` | 未设置 | 否 | `true` | 分类级显示 |
| `'important'` | `true` | `false` | 否 | `false` | 订阅级隐藏优先于分类级 |
| `'important'` | `false` | `true` | 否 | `true` | 订阅级显示优先于分类级 |
| `'none'` | 未设置 | 未设置 | 否 | `false` | 全局 none 默认隐藏 |
| `'none'` | 未设置 | 未设置 | 是 | `false` | Important 特殊规则在 none 下不命中 |
| `'none'` | `true` | 未设置 | 否 | `true` | 分类级显示 |
| `'none'` | `true` | 未设置 | 是 | `true` | 分类级显示 |
| `'none'` | 未设置 | `true` | 否 | `true` | 订阅级显示 |
| `'none'` | 未设置 | `true` | 是 | `true` | 订阅级显示 |
| `'none'` | `false` | `true` | 否 | `true` | 订阅级显示优先于分类级 |

### 5.6 覆盖优先级总结

```
Important 特殊规则 (sucGlobal !== 'none')
    ↓ (优先级最高，仅对 Important 订阅有效)
订阅级 attributes.show_unread_count
    ↓
分类级 attributes.show_unread_count
    ↓ (优先级最低)
全局回退 (all / important / none)
```

**特殊注意**：
- Important 特殊规则**只在 `sucGlobal !== 'none'` 时生效**
- 当 `sucGlobal === 'none'` 时，Important 订阅与普通订阅遵循相同的三级覆盖规则
- 订阅级属性可以覆盖分类级属性
- 分类级属性可以覆盖全局默认
- 任何显式设置的属性（`true` 或 `false`）都会终止后续判断

---

## 六、订阅移动后分类内顺序为何仍留有待处理

### 6.1 代码中的 TODO 标注

在 `moveAction()` 方法中有一个明确的 TODO：

[feedController.php#L1081](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/feedController.php#L1081)

```php
/**
 * @todo should handle order of the feed inside the category.
 */
public function moveAction(): void {
```

### 6.2 问题分析

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

### 6.3 当前排序的实际处理方式

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

### 6.4 为何仍有 TODO

问题在于：FreshRSS **没有提供订阅级别的 position 属性**来支持用户自定义订阅顺序。

对比分类排序：分类有 `attributes.position` 字段，用户可以手动指定分类的显示顺序。但订阅没有类似的 position 机制——所有订阅都是固定的按名称排序。

这意味着：

1. **用户无法自定义订阅在分类内的显示顺序**——只能按名称字母序
2. **移动订阅后，如果用户期望订阅出现在分类的特定位置**（如最顶部或最底部），当前机制无法满足
3. TODO 的意图是：为 `moveAction()` 添加类似分类 position 的支持，让移动操作可以指定订阅在目标分类中的位置

目前，订阅的 `attributes` JSON 字段中只有 `defaultSort` 和 `defaultOrder`，它们控制的是**文章列表**的排序，而非订阅自身在侧边栏中的排序。

### 6.5 相关的用户配置

虽然不能自定义订阅位置，但有一个相关的简化配置：

[config-user.default.php#L112](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/config-user.default.php#L112)

```php
'simplify_over_n_feeds' => 1000,
```

当订阅总数超过此阈值时，侧边栏会简化显示（隐藏 favicon、隐藏配置下拉菜单等），以提升性能。

---

## 七、默认分类在中文界面的名称来源

### 7.1 名称的运行时覆盖机制

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

### 7.2 翻译键的解析路径

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

### 7.3 各语言的翻译值

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

### 7.4 数据库中的名称与显示名称的差异

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

---

## 八、侧边栏统计机制

### 8.1 统计数据结构

**核心文件**: [Context.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Context.php)

全局统计变量：

| 变量 | 类型 | 说明 |
|------|------|------|
| `$total_unread` | int | 主视图未读总数（仅统计 PRIORITY_MAIN_STREAM 及以上） |
| `$total_important_unread` | int | 重要订阅未读数（统计 PRIORITY_IMPORTANT 及以上） |
| `$total_starred` | array | 收藏文章统计（all/read/unread） |
| `$get_unread` | int | 当前视图的未读数 |

### 8.2 统计数据初始化

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

### 8.3 分类级别的未读统计

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

### 8.4 单个分类的未读统计

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

### 8.5 单个订阅的未读统计

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

### 8.6 数据库级别的缓存

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

### 8.7 侧边栏视图渲染

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

### 8.8 分类列表预加载

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

## 九、三者之间的关系

### 9.1 层级结构关系

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

### 9.2 显隐控制数据流

```
用户配置 (config.php)
  ├── display_categories: 'active' | 'remember' | 'all' | 'none'
  │     │
  │     ├─ PHP 端: 控制初始展开状态 ($c_show)
  │     └─ JS 端: context.display_categories 控制 remember 模式
  │
  ├── show_unread_count: 'all' | 'important' | 'none'
  │     │
  │     ├─ 全局导航项: $hideSucGlobal / $hideSucImportant
  │     ├─ 分类级: showUnreadCount() 的回退值
  │     └─ 订阅级: showUnreadCount() 的 Important 特殊规则和回退值
  │
  └── hide_read_feeds: true | false
        │
        └─ CSS 级过滤: 添加 state_unread 类隐藏已读项
```

### 9.3 统计聚合关系

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

### 9.4 移动订阅对统计的影响

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

### 9.5 默认分类的特殊角色

默认分类在整个系统中扮演着"回收站"和"安全网"的角色：

1. **新订阅默认归类**：添加订阅时如果未指定分类，放入默认分类
2. **删除分类的归宿**：删除分类时，该分类下所有订阅被移到默认分类
3. **无效分类的兜底**：移动订阅时如果目标分类不存在，使用默认分类
4. **不可删除**：默认分类不能被删除，保证系统至少有一个分类
5. **名称不可修改**：默认分类的名称始终使用翻译值，用户无法自定义

### 9.6 缓存与性能优化

为了避免频繁的数据库查询，系统使用了多层缓存：

1. **数据库缓存**：`_feed.cache_nbUnreads` 和 `_feed.cache_nbEntries`
2. **对象缓存**：`FreshRSS_Feed::$nbNotRead` 和 `FreshRSS_Category::$nbNotRead`
3. **上下文缓存**：`FreshRSS_Context::$categories` 和 `FreshRSS_Context::$total_unread`
4. **前端状态缓存**：`localStorage.FreshRSS_open_categories`（remember 模式）

缓存更新时机：
- 刷新订阅后 (`actualizeFeedsAndCommit`)
- 标记文章已读后
- 清理旧文章后
- 新增/删除文章后
- 用户手动展开/折叠分类后（仅前端缓存）

---

## 十、关键代码文件索引

| 功能 | 文件 | 关键方法/行号 |
|------|------|--------------|
| 分类模型 | [Category.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Category.php) | `isDefault()`, `nbNotRead()`, `feeds()`, `sortFeeds()`, `_id()`, `_name()`, `showUnreadCount()` |
| 分类DAO | [CategoryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/CategoryDAO.php) | `DEFAULTCATEGORYID`, `checkDefault()`, `listSortedCategories()`, `listCategories()`, `countNotRead()` |
| 订阅模型 | [Feed.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Feed.php) | `category()`, `nbNotRead()`, `priority()`, `showUnreadCount()` |
| 订阅DAO | [FeedDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/FeedDAO.php) | `changeCategory()`, `updateFeed()`, `updateCachedValues()`, `listByCategory()` |
| 属性Trait | [AttributesTrait.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/AttributesTrait.php) | `attributeInt()`, `attributeBoolean()`, `attributeString()`, `_attribute()` |
| 上下文 | [Context.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/Context.php) | `categories()`, `updateUsingRequest()`, `$total_unread`, `display_categories` 校验 |
| 用户配置 | [UserConfiguration.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Models/UserConfiguration.php) | 属性声明 (PHPDoc) |
| 分类控制器 | [categoryController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/categoryController.php) | `deleteAction()`, `updateAction()`（position 设置） |
| 订阅控制器 | [feedController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/feedController.php) | `moveFeed()`, `moveAction()`（含 TODO）, `addFeed()` |
| 订阅管理控制器 | [subscriptionController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/subscriptionController.php) | `feedAction()` |
| 配置控制器 | [configureController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/Controllers/configureController.php) | `display_categories` 配置保存 |
| 侧边栏视图 | [aside_feed.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/layout/aside_feed.phtml) | 分类树渲染、隐藏过滤、未读数显示控制、展开状态初始渲染 |
| JS 变量模板 | [javascript_vars.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/views/helpers/javascript_vars.phtml) | `context.display_categories` 传递到前端 |
| 主 JS | [main.js](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/p/scripts/main.js) | `rememberOpenCategory()`, `openCategory()`, `init_column_categories()`, 点击事件处理 |
| 额外 JS | [extra.js](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/p/scripts/extra.js) | 登出时清除 localStorage 记忆 |
| 主题 CSS | [frss.css](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/p/themes/base-theme/frss.css) | `data-unread-hide` 样式处理 |
| 翻译系统 | [Translate.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/lib/Minz/Translate.php) | `t()`, `resolveKey()`, `loadKey()` |
| 中文翻译 | [zh-CN/gen.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/i18n/zh-CN/gen.php) | `'default_category' => '未分类'` |
| 英语翻译 | [en/gen.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/app/i18n/en/gen.php) | `'default_category' => 'Uncategorized'` |
| 用户默认配置 | [config-user.default.php](file:///d:/fz/0601-1/solo-dogfeeding/code/22-FreshRSS/config-user.default.php) | `show_unread_count`, `display_categories`, `hide_read_feeds` |