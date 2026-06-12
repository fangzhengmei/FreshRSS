# FreshRSS 标签模型与文章关联机制分析

## 一、概述

FreshRSS 中存在**两套独立的标签体系**：

1. **Feed 源标签（Entry Tags）**：RSS/ATOM 源自带的分类标签，存储在文章表的 `tags` 字段中，属于文章的固有属性
2. **用户自定义标签（User Tags / Labels）**：用户手动创建和管理的标签，通过关联表与文章建立多对多关系，支持打标、取消打标、标签管理等操作

本文档主要分析**用户自定义标签**（以下简称"标签"）的模型与关联机制。

---

## 二、数据库表结构

### 2.1 标签表 `_tag`

存储标签的元数据定义。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | INT (PK, AUTO_INCREMENT) | 标签 ID |
| `name` | VARCHAR(191) (UNIQUE) | 标签名称（唯一，与分类名不可重名） |
| `attributes` | TEXT | 扩展属性（JSON 格式），存储过滤动作等配置 |

> 代码参考：[app/SQL/install.sql.mysql.php](app/SQL/install.sql.mysql.php#L96-L103)

### 2.2 文章-标签关联表 `_entrytag`

实现文章与标签的**多对多关系**，通过联合主键保证唯一性。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id_tag` | INT (PK, FK) | 标签 ID，外键关联 `_tag.id` |
| `id_entry` | BIGINT (PK, FK) | 文章 ID，外键关联 `_entry.id` |

**外键约束（级联删除/更新）**：

```sql
FOREIGN KEY (id_tag) REFERENCES _tag(id) ON DELETE CASCADE ON UPDATE CASCADE
FOREIGN KEY (id_entry) REFERENCES _entry(id) ON DELETE CASCADE ON UPDATE CASCADE
```

> 代码参考：[app/SQL/install.sql.mysql.php](app/SQL/install.sql.mysql.php#L105-L113)

### 2.3 与 Feed 源标签的区别

文章表 `_entry` 中还有一个 `tags` 字段（VARCHAR(2048)），用于存储 RSS 源自带的标签，两者的区别：

| 特性 | Feed 源标签 (`_entry.tags`) | 用户自定义标签 (`_tag` + `_entrytag`) |
|------|----------------------------|--------------------------------------|
| 来源 | RSS/ATOM 源数据 | 用户手动创建或自动打标规则生成 |
| 可修改性 | 只读（随源更新） | 可增删改 |
| 存储方式 | 字符串（井号分隔，如 `#tag1 #tag2`） | 规范化关联表（多对多） |
| 模型属性 | `FreshRSS_Entry::$tags` | `FreshRSS_Tag` 实体类 + `_entrytag` 关联 |
| 搜索过滤 | 直接在 `_entry.tags` 字段上做 LIKE/REGEXP | 通过 JOIN `_entrytag` + `_tag` 表查询 |
| 存储限制 | VARCHAR(2048)，标签数量有限制 | 无限制，每行一条关联 |

> 代码参考：[app/Models/Entry.php](app/Models/Entry.php#L38-L39)、[app/Models/EntryDAO.php](app/Models/EntryDAO.php#L231-L233)

---

## 三、核心模型与 DAO

### 3.1 标签模型 `FreshRSS_Tag`

位置：[app/Models/Tag.php](app/Models/Tag.php)

**核心属性**：
- `$id` (int) - 标签 ID
- `$name` (string) - 标签名称
- `$nbEntries` (int) - 关联文章数（懒加载，首次访问时查询数据库）
- `$nbUnread` (int) - 未读文章数（懒加载）

**使用的 Trait**：
- `FreshRSS_AttributesTrait` - 提供属性扩展机制（存储 `attributes` JSON 字段）
- `FreshRSS_FilterActionsTrait` - 提供过滤动作配置（自动打标规则）

### 3.2 标签 DAO 体系

使用工厂模式根据数据库类型选择实现：

```
FreshRSS_TagDAO (基类，MySQL 默认)
  ├── FreshRSS_TagDAOSQLite (SQLite 覆盖)
  └── FreshRSS_TagDAOPGSQL (PostgreSQL 覆盖)
```

工厂创建方式：`FreshRSS_Factory::createTagDao()`

**基类**：[app/Models/TagDAO.php](app/Models/TagDAO.php)

**核心方法一览**：

| 方法 | 功能 |
|------|------|
| `addTag($values)` | 新增标签，自动检查名称不与分类重名 |
| `addTagObject($tag)` | 从对象新增标签，已存在则返回已有 ID |
| `deleteTag($id)` | 删除标签（关联由 DB 级联清理） |
| `updateTagName($id, $name)` | 更新标签名称 |
| `updateTagAttributes($id, $attributes)` | 更新标签扩展属性 JSON |
| `searchById($id)` | 按 ID 查找标签 |
| `searchByName($name)` | 按名称查找标签 |
| `listTags($precounts)` | 列出所有标签（可选预计算未读数） |
| `tagEntry($id_tag, $id_entry, $checked)` | 给单篇文章打/取消标签 |
| `tagEntries($addLabels)` | 批量打标（多篇文章 × 多个标签） |
| `getTagsForEntry($id_entry)` | 获取单篇文章的所有标签及打标状态 |
| `getTagsForEntries($entries)` | 批量获取多篇文章的标签 |
| `getEntryIdsTagNames($entries)` | 返回 `[e_id => [标签名列表]]`，用于 API 和导出 |
| `updateEntryTag($oldTagId, $newTagId)` | 迁移标签关联（合并标签用） |
| `countEntries($id)` | 统计标签关联的文章数 |
| `countNotRead($id)` | 统计标签下的未读文章数 |

**SQLite 子类**：[app/Models/TagDAOSQLite.php](app/Models/TagDAOSQLite.php) - 仅覆盖 `sqlIgnore()` 返回 `OR IGNORE`

**PostgreSQL 子类**：[app/Models/TagDAOPGSQL.php](app/Models/TagDAOPGSQL.php) - 覆盖 `sqlIgnore()` 和 `sqlResetSequence()`

---

## 四、标签创建流程

### 4.1 主动创建（标签管理页面）

**入口 Controller**：`tag/addAction`

位置：[app/Controllers/tagController.php](app/Controllers/tagController.php#L159-L186)

**流程**：
1. 用户在标签管理页面提交表单（POST 请求，参数 `name`）
2. 校验名称不能与**分类**重名（调用 `CategoryDAO::searchByName()`）
3. 校验名称不能与**已有标签**重名（调用 `TagDAO::searchByName()`）
4. 调用 `addTag(['name' => $name])` 插入 `_tag` 表
5. 返回成功/失败通知，重定向回标签列表页

> **注意**：标签名称不能与分类名称重名，因为二者在搜索和导航中使用相同的前缀语法。

### 4.2 隐式创建（打标时自动创建）

**入口 Controller**：`tag/tagEntryAction`

位置：[app/Controllers/tagController.php](app/Controllers/tagController.php#L31-L64)

**流程**：
1. 用户在文章上打一个**不存在的标签**
2. 传入 `name_tag` 参数（标签名称）和 `checked=true`
3. 先按名称 `searchByName($name_tag)` 查找标签
4. 不存在则自动调用 `addTag(['name' => $name_tag])` 创建新标签
5. 获得 `$id_tag` 后再调用 `tagEntry($id_tag, $id_entry, $checked)` 建立关联

这是最常用的标签创建方式——用户无需预先管理标签，打标时自动按需创建。

### 4.3 API 创建（GReader 兼容 API）

**入口**：`p/api/greader.php` 的 `edit-tag` 接口

客户端调用时传入 `a=user/-/label/标签名` 参数，服务端解析标签名后：
1. 按名称查找标签
2. 不存在则自动创建
3. 对指定文章列表批量打标

---

## 五、打标入口（文章关联标签的途径）

### 5.1 Web UI 手动打标 - 完整流程

#### 5.1.1 标签下拉菜单的触发与加载

**HTML 入口**：每篇文章头部有一个标签图标按钮

位置：[app/views/helpers/index/normal/entry_header.phtml](app/views/helpers/index/normal/entry_header.phtml#L83-L89)

```html
<div class="item-element dropdown dynamictags">
    <div id="dropdown-labels2-{entryId}" class="dropdown-target"></div>
    <a class="dropdown-toggle" href="#dropdown-labels2-{entryId}">
        <?= _i('label') ?>  <!-- 标签图标 -->
    </a>
</div>
```

**点击触发流程**：

```
用户点击标签图标
    ↓
JS 事件捕获：点击 .item.labels a.dropdown-toggle
    ↓
调用 show_labels_menu(el)  [main.js L780]
    ↓
检查：下拉菜单是否已存在？且 forceReloadLabelsList=false？
    ├── 否（首次打开或强制重载）→ 加载模板 + 调用 loadDynamicTags()
    └── 是 → 直接显示已缓存的菜单（无需重复请求）
```

> 代码参考：[p/scripts/main.js](p/scripts/main.js#L780-L797)

#### 5.1.2 `loadDynamicTags()` - 标签列表 AJAX 加载

位置：[p/scripts/main.js](p/scripts/main.js#L1696-L1795)

**加载流程详解**：

```
1. 清空下拉菜单中已有的 <li.item> 元素
2. 从当前文章 DOM 获取文章 ID：flux_{id} → id
3. 发送 GET 请求：
   URL: ./?c=tag&a=getTagsForEntry&id_entry={entryId}
   Header: Accept: application/json
    ↓
4. 后端处理：getTagsForEntryAction()
   调用 TagDAO::getTagsForEntry($id_entry)
   SQL: SELECT t.id, t.name, et.id_entry IS NOT NULL as checked
        FROM _tag t
        LEFT OUTER JOIN _entrytag et ON et.id_tag = t.id AND et.id_entry=:id_entry
        ORDER BY t.name
    ↓
5. 接收 JSON 数组：
   [{id:1, name:"重要", checked:true}, {id:2, name:"待读", checked:false}, ...]
    ↓
6. 动态构建 DOM：
   a. 登录用户：添加"新建标签"行
      - 隐藏的 checkbox (name="t_0", class="checkboxTag checkboxNewTag")
      - 文本输入框 (class="newTag") + 自动补全 datalist
      - "+" 按钮
   b. 遍历 JSON，为每个标签生成一行：
      <li class="item">
        <label>
          <input class="checkboxTag" name="t_{id}" type="checkbox" [checked]>
          {标签名称}
        </label>
      </li>
   c. 填充 datalist-labels 自动补全选项
```

**后端返回格式**（[app/views/tag/getTagsForEntry.phtml](app/views/tag/getTagsForEntry.phtml)）：
```php
echo json_encode($this->tagsForEntry);
```

**为什么使用 `LEFT OUTER JOIN` + `checked` 字段？**
- `LEFT JOIN` 保证返回**所有**标签，即使文章没有打这个标签
- `et.id_entry IS NOT NULL as checked` 直接在 SQL 层计算出该文章是否已打此标签
- 前端直接用 `checked` 字段设置 checkbox 的勾选状态，无需额外计算

#### 5.1.3 勾选标签 - 请求字段详解

**事件绑定**：`stream.onchange` 监听 `.checkboxTag` 的变化

位置：[p/scripts/main.js](p/scripts/main.js#L1592-L1654)

**勾选后的处理流程**：

```
checkbox 状态变化
    ↓
1. 解析参数：
   tagId   = checkbox.name.replace(/^t_/, '')   // "t_5" → "5"，新建标签为 "0"
   tagName = checkbox 后续输入框的值             // 仅新建标签时有值
   isChecked = checkbox.checked                  // true/false
   entryId   = 从最近 .flux 元素 id 中提取         // "flux_123456" → "123456"
    ↓
2. 参数合法性校验：
   if ((tagId == 0 && tagName.length > 0) || tagId != 0)
   ├── 新建标签场景：tagId=0，必须填写 tagName
   └── 已有标签场景：tagId≠0，不需要 tagName
    ↓
3. 禁用 checkbox，防止重复提交
    ↓
4. 发送 POST 请求：
   URL: ./?c=tag&a=tagEntry&ajax=1
   Method: POST
   Content-Type: application/json; charset=utf-8
   Body: JSON.stringify({
       _csrf:     context.csrf,        // CSRF 令牌
       id_tag:    tagId,              // 标签ID（0=新建）
       name_tag:  tagId==0 ? tagName : '',  // 新标签名称
       id_entry:  entryId,            // 文章ID
       checked:   isChecked,          // true=打标/false=取消
       ajax:      1                    // AJAX 请求标记
   })
```

**请求字段对照表**：

| 字段 | 类型 | 说明 | 示例值 |
|------|------|------|--------|
| `_csrf` | string | CSRF 防跨站伪造令牌 | `"abc123..."` |
| `id_tag` | int | 标签 ID。**0 表示新建标签**，非 0 为已有标签 | `5` 或 `0` |
| `name_tag` | string | 新建标签时的名称。**仅当 id_tag=0 时有效** | `"重要"` 或 `""` |
| `id_entry` | string | 文章 ID（微秒时间戳字符串） | `"1718123456000000"` |
| `checked` | bool | 打标动作：`true`=打标，`false`=取消打标 | `true` |
| `ajax` | int | 是否为 AJAX 请求（影响响应方式） | `1` |

#### 5.1.4 后端处理 `tagEntryAction`

位置：[app/Controllers/tagController.php](app/Controllers/tagController.php#L31-L64)

**处理逻辑**：

```php
// 1. 权限检查
if (!FreshRSS_Auth::hasAccess()) { Minz_Error::error(403); }

// 2. 仅接受 POST
if (Minz_Request::isPost()) {
    $id_tag    = Minz_Request::paramInt('id_tag');        // 0 或 >0
    $name_tag  = Minz_Request::paramString('name_tag');   // 新建时的名称
    $id_entry  = Minz_Request::paramString('id_entry');   // 文章ID
    $checked   = Minz_Request::paramBoolean('checked');   // 打/取消标

    if ($id_entry != '') {
        $tagDAO = FreshRSS_Factory::createTagDao();

        // 3. 隐式创建：id_tag=0 且 要打标（非取消）且 有名称
        if ($id_tag == 0 && $name_tag !== '' && $checked) {
            $existing_tag = $tagDAO->searchByName($name_tag);
            if ($existing_tag !== null) {
                // 标签名已存在 → 复用已有ID
                $tagDAO->tagEntry($existing_tag->id(), $id_entry, $checked);
            } else {
                // 标签不存在 → 自动创建，返回新ID
                $id_tag = $tagDAO->addTag(['name' => $name_tag]);
            }
        }

        // 4. 建立/删除关联
        if ($id_tag != false) {
            $tagDAO->tagEntry($id_tag, $id_entry, $checked);
        }
    }
}
```

**底层 SQL（`TagDAO::tagEntry()`）**：
- `checked=true`: `INSERT IGNORE INTO _entrytag(id_tag, id_entry) VALUES(:id_tag, :id_entry)`
- `checked=false`: `DELETE FROM _entrytag WHERE id_tag=:id_tag AND id_entry=:id_entry`

> 代码参考：[app/Models/TagDAO.php](app/Models/TagDAO.php#L302-L324)

#### 5.1.5 勾选后的回写 - 未读数与标签列表

**成功响应回调 `req.onload`**（[p/scripts/main.js](p/scripts/main.js#L1611-L1617)）：

```js
req.onload = function (e) {
    if (this.status != 200) { return req.onerror(e); }

    // ★ 核心回写逻辑：侧栏标签未读数 +1/-1
    if (entry.classList.contains('not_read')) {
        // 文章当前是未读状态，打标会影响未读计数
        // isChecked=true → +1（给标签增加1个未读）
        // isChecked=false → -1（给标签减少1个未读）
        incUnreadsTag('t_' + tagId, isChecked ? 1 : -1);
    }
};
```

**`incUnreadsTag()` 函数**（[p/scripts/main.js](p/scripts/main.js#L184-L196)）：

```js
function incUnreadsTag(tag_id, nb) {
    // 1. 更新侧栏中具体标签的未读数
    //    DOM: <li id="t_5" ...> <span class="item-title" data-unread="3">
    let t = document.getElementById(tag_id);  // "t_5"
    if (t) {
        const unreads = str2int(t.getAttribute('data-unread'));
        t.setAttribute('data-unread', unreads + nb);
        t.querySelector('.item-title').setAttribute(
            'data-unread', numberFormat(unreads + nb)
        );
    }

    // 2. 更新侧栏中"我的标签"总文件夹的未读数
    //    DOM: <li class="category tags ..."> <span class="title" data-unread="10">
    t = document.querySelector('.category.tags .title');
    if (t) {
        const unreads = str2int(t.getAttribute('data-unread'));
        t.setAttribute('data-unread', numberFormat(unreads + nb));
    }
}
```

> **关键前提**：仅当文章当前是**未读**（`.not_read`）时才更新未读数。已读文章的打标/取消打标不影响未读统计。

**请求结束回调 `req.onloadend`**（[p/scripts/main.js](p/scripts/main.js#L1619-L1641)）：

```js
req.onloadend = function (e) {
    checkboxTag.disabled = false;  // 重新启用 checkbox

    if (tagId == 0) {
        // ★ 新建标签分支
        forceReloadLabelsList = true;              // ① 设置全局强制刷新标志
        loadDynamicTags(checkboxTag.closest('div.dropdown'));  // ② 立即刷新当前菜单
    } else {
        // 已有标签分支：清理其他重复的下拉菜单 DOM
        // （一篇文章可能有多个标签入口：头部和底部）
        const dropdownmenu_current = ev.target.closest('.dropdown-menu');
        const flux = ev.target.closest('.flux');
        const dropdownmenu_all = flux.querySelectorAll('.dynamictags .dropdown-menu');
        if (dropdownmenu_all.length > 1) {
            dropdownmenu_all.forEach(function (currentValue) {
                if (currentValue !== dropdownmenu_current) {
                    // 删除非当前的菜单，下次打开时会重新拉取（保证状态一致）
                    currentValue.nextElementSibling.remove();
                    currentValue.parentNode.removeChild(currentValue);
                }
            });
        }
    }
};
```

### 5.2 新增标签后的刷新策略与 `forceReloadLabelsList`

**全局变量定义**（[p/scripts/main.js](p/scripts/main.js#L1691-L1694)）：

```js
// forceReloadLabelsList default is false, so that the list does need a reload after opening it a second time.
// will be set to true, if a new tag is added. Then the labels list will be reloaded each opening.
// purpose of this flag: minimize the network traffic.
let forceReloadLabelsList = false;
```

#### 代码中仅有的 3 个引用点

| 位置 | 代码 | 作用 |
|------|------|------|
| L1694 | `let forceReloadLabelsList = false;` | **声明并初始化**为 `false` |
| L1623 | `forceReloadLabelsList = true;` | **唯一置位点**：新建标签后设置为 `true` |
| L784 | `if (!dropdownMenu \|\| forceReloadLabelsList)` | **唯一消费点**：`show_labels_menu()` 中判断是否重建菜单 |

> ⚠️ **关键事实：代码中没有任何地方将 `forceReloadLabelsList` 重置回 `false`**。一旦被置为 `true`，它在当前页面的整个生命周期内将永远保持 `true`。

#### 从置位到消费的完整路径

```
① 用户勾选"新建标签"(tagId=0)，输入标签名后提交
    ↓
② XHR POST ./?c=tag&a=tagEntry&ajax=1 请求成功
    ↓
③ req.onloadend 回调（L1619-1641）：
    if (tagId == 0) {
        forceReloadLabelsList = true;                           // ← 唯一置位点
        loadDynamicTags(checkboxTag.closest('div.dropdown'));   // 立即刷新当前文章的菜单
    }
    ↓
④ 立即触发的 loadDynamicTags()：
    - 每次调用都无条件发送 GET ./?c=tag&a=getTagsForEntry&id_entry={entryId}
    - 没有任何客户端缓存层，每次都是全新的 AJAX 请求
    ↓
⑤ 后续用户打开**任意**文章的标签菜单：
    调用 show_labels_menu(el)
    ↓
⑥ show_labels_menu 的判断逻辑（L780-797）：
    const dropdownMenu = div.querySelector('.dropdown-menu');

    if (!dropdownMenu || forceReloadLabelsList) {
        // 满足任一条件就重建：
        //   条件A: dropdownMenu 不存在（首次打开，DOM中没有菜单）
        //   条件B: forceReloadLabelsList === true（已置位）
        if (dropdownMenu) {
            dropdownMenu.nextElementSibling.remove();   // 删除关闭按钮
            dropdownMenu.remove();                       // 删除旧菜单 DOM
        }
        // 插入空模板 → 调用 loadDynamicTags() 发 AJAX 请求
        const template = document.getElementById('labels_article_template').innerHTML;
        div.insertAdjacentHTML('beforeend', template);
        return loadDynamicTags(div.closest('.dynamictags'));
    }
    return true;  // 仅当 dropdownMenu 存在 且 forceReloadLabelsList===false 时走此分支
```

#### `forceReloadLabelsList` 的真实生命周期（单页面会话内）

```
页面加载: forceReloadLabelsList = false
  ↓
阶段一：从未新建过标签
  ├─ 打开文章A菜单 → dropdownMenu不存在 → 条件A满足 → AJAX请求 → DOM写入
  ├─ 关闭后再打开A菜单 → dropdownMenu存在 + flag=false → 条件均不满足 → 复用DOM ★零请求
  ├─ 打开文章B菜单 → dropdownMenu不存在 → 条件A满足 → AJAX请求 → DOM写入
  └─ 关闭后再打开B菜单 → dropdownMenu存在 + flag=false → 复用DOM ★零请求
  ↓
用户在某篇文章中新建了一个标签
  ↓
onloadend: forceReloadLabelsList = true  ← 永久置位，不会回退
  ↓
阶段二：forceReloadLabelsList === true（永久）
  ├─ 打开文章C菜单 → dropdownMenu不存在 → 条件A满足 → AJAX请求
  ├─ 关闭后再打开C菜单 → dropdownMenu存在 + flag=true → 条件B满足 → 删除旧DOM → AJAX请求
  ├─ 打开文章A菜单 → dropdownMenu存在 + flag=true → 条件B满足 → 删除旧DOM → AJAX请求
  └─ 每次打开任何文章菜单 → 必然删除旧DOM → 必然AJAX请求 ★无法复用
  ↓
  （直到页面刷新/导航离开，forceReloadLabelsList 才会随 JS 上下文销毁重置为 false）
```

#### 对一致性和请求次数的影响

| 场景 | 请求次数 | 说明 |
|------|---------|------|
| **从未新建标签** | 每篇文章首次打开 1 次，后续复用 0 次 | 条件A控制：首次没有 DOM 必须请求，之后有 DOM 且 flag=false 可复用 |
| **新建标签后** | **每次打开菜单都 1 次** | 条件B控制：flag=true 导致每次都删除旧 DOM 并重新请求 |
| **页面刷新后** | 回到"从未新建标签"状态 | flag 随页面重新初始化为 false |

**一致性影响**：
- 阶段一（flag=false）：标签列表依赖 DOM 缓存，如果其他用户/会话创建了新标签，当前页面不会感知。但在单用户场景下，缓存期内的列表是自洽的（只有自己能创建标签，自己刚操作过）。
- 阶段二（flag=true）：每次打开菜单都是最新的服务端数据，**一致性最强**，但代价是每次打开都发网络请求。

**请求次数影响**：
- 这是**单向不可逆**的设计：一旦用户在当前会话中新建过标签，后续所有菜单打开都变成实时请求。
- 注释中的 `purpose of this flag: minimize the network traffic` 指的是阶段一的优化——在用户没有新建标签时，通过 DOM 缓存避免重复请求。
- 阶段二的"每次请求"并非设计疏忽，而是**正确性优先**的权衡：新建标签后，每篇文章的标签列表内容和勾选状态都可能不同（因为新标签对当前文章是 checked，对其他文章是 unchecked），只有按文章 ID 逐个查询 `getTagsForEntry` 才能拿到准确的 `checked` 状态。这意味着没有可靠的客户端缓存策略可以替代逐篇文章请求。

#### 为什么不复用缓存？根本原因是 `checked` 状态因文章而异

`getTagsForEntry` 返回的不仅是标签列表，还有每篇文章的 `checked` 状态：

```json
// 文章A（已打了"紧急"标签）
[{id:1, name:"重要", checked:true}, {id:12, name:"紧急", checked:true}]

// 文章B（没有打"紧急"标签）
[{id:1, name:"重要", checked:false}, {id:12, name:"紧急", checked:false}]
```

两篇文章的返回结果不同——同样的标签列表，`checked` 布尔值不一样。因此：
- **不能**用一个全局缓存给所有文章共用（checked 不对）
- **不能**给每篇文章分别缓存（checked 可能因用户操作而变化）
- 唯一安全的做法是每次打开菜单时重新向服务端请求

所以 `forceReloadLabelsList=true` 后每次都发请求，本质上是 `checked` 状态差异化导致缓存不可复用的正确处理。

#### 侧栏标签列表不受影响

注意：侧栏的标签列表（[app/layout/aside_feed.phtml](app/layout/aside_feed.phtml#L69-L94)）是在页面初始渲染时由 PHP 生成的静态 HTML，`forceReloadLabelsList` 机制仅影响文章内的标签下拉菜单，不会触发侧栏的重新加载。侧栏的未读数更新通过 `incUnreadsTag()` 直接操作 DOM 实现，不依赖标签菜单的刷新。

### 5.3 侧栏标签列表的 HTML 结构（未读数回写目标）

位置：[app/layout/aside_feed.phtml](app/layout/aside_feed.phtml#L69-L94)

```html
<li id="tags" class="tree-folder category tags" data-unread="10">
  <a href="...?get=T" class="tree-folder-title">
    <button class="dropdown-toggle">...</button>
    <span class="title" data-unread="10">我的标签</span>
    <!-- ★ 总未读数回写点：.category.tags .title 的 data-unread 属性 -->
  </a>
  <ul class="tree-folder-items">
    <?php foreach ($this->tags as $tag): ?>
    <li id="t_<?= $tag->id() ?>" class="item feed" data-unread="<?= $tag->nbUnread() ?>">
      <!-- ★ 单个标签未读数回写点：#t_{id} 的 data-unread 属性 -->
      <a class="item-title" data-unread="<?= format_number($tag->nbUnread()) ?>"
         href="...?get=t_<?= $tag->id() ?>">
        <?= _i('label') ?> <?= $tag->name() ?>
        <!-- ★ item-title 子元素也有 data-unread，用于显示格式化后的数字 -->
      </a>
    </li>
    <?php endforeach; ?>
  </ul>
</li>
```

### 5.4 批量打标 `tagEntries()`

位置：[app/Models/TagDAO.php](app/Models/TagDAO.php#L330-L356)

支持一次对多篇文章打多个标签，使用 `INSERT IGNORE` 批量插入。

**输入格式**：
```php
[
    ['id_tag' => 1, 'id_entry' => '123456'],
    ['id_tag' => 2, 'id_entry' => '123456'],
    ['id_tag' => 1, 'id_entry' => '789012'],
]
```

### 5.5 自动打标（过滤规则驱动）

**核心机制**：通过 `FreshRSS_FilterActionsTrait` 为标签配置过滤规则，当新文章入库时自动匹配并打标。

#### 自动打标触发流程

```
feed 更新 → 新文章写入 _entrytmp → commitNewEntries()
    → applyLabelActions($nbNewEntries)
        → 遍历所有配置了 label 过滤动作的标签
            → 遍历最新 $nbNewEntries 篇文章
                → $label->applyFilterActions($entry, $applyLabel)
                    → 若文章匹配标签的过滤规则 → $applyLabel=true
                → 收集匹配的 [id_tag, id_entry] 对
        → tagEntries($applyLabels) 批量写入 _entrytag
```

**关键代码**：
- 触发入口：[app/Controllers/feedController.php](app/Controllers/feedController.php#L879-L901)
- 过滤规则配置存储在标签的 `attributes.filters` JSON 字段中

#### 过滤规则匹配动作

`applyFilterActions()` 支持三种动作：
- `read` - 自动标记为已读
- `star` - 自动加星标
- `label` - 自动打标签（设置 `$applyLabel=true`）

> **重要**：自动打标只对**新入库的文章**生效（`!$entry->isUpdated()`），避免覆盖用户手动操作。

### 5.6 GReader API 打标

**接口**：`/p/api/greader.php` 的 `edit-tag` 端点

**参数**：
- `a` - 要添加的标签（可重复），格式 `user/-/label/标签名`
- `r` - 要移除的标签（可重复）
- `i` - 文章 ID（可重复）

---

## 六、标签删除后的文章影响

### 6.1 数据库级联删除（核心机制）

`_entrytag` 表的外键约束设置了 **`ON DELETE CASCADE`**，这是理解删除影响的关键：

```sql
FOREIGN KEY (id_tag) REFERENCES _tag(id) ON DELETE CASCADE ON UPDATE CASCADE
FOREIGN KEY (id_entry) REFERENCES _entry(id) ON DELETE CASCADE ON UPDATE CASCADE
```

**级联删除语义**：
- **删除标签 `_tag` 记录时**：数据库**自动删除** `_entrytag` 中所有 `id_tag` 等于被删标签 ID 的关联记录
- **删除文章 `_entry` 记录时**：数据库**自动删除** `_entrytag` 中所有 `id_entry` 等于被删文章 ID 的关联记录

**这意味着**：
1. 标签删除后，所有曾经打了该标签的文章**自动失去该标签关联**
2. 文章删除后，其所有标签关联**自动清理**，不会产生孤儿记录
3. **应用层代码无需手动维护关联关系的删除**，全部由数据库保证引用完整性

> 代码参考：[app/SQL/install.sql.mysql.php](app/SQL/install.sql.mysql.php#L109-L110)

### 6.2 应用层删除逻辑

**DAO 方法**：`FreshRSS_TagDAO::deleteTag($id)`

位置：[app/Models/TagDAO.php](app/Models/TagDAO.php#L122-L139)

代码非常简洁，仅执行一条 DELETE SQL：

```php
public function deleteTag(int $id): int|false {
    if ($id <= 0) {
        return false;
    }
    $sql = 'DELETE FROM `_tag` WHERE id=:id';
    // 仅执行这一条 SQL，关联表清理由 DB 外键级联自动完成
    ...
}
```

### 6.3 控制器删除入口

**动作**：`tag/deleteAction`

位置：[app/Controllers/tagController.php](app/Controllers/tagController.php#L66-L85)

**流程**：
1. 接收 POST 参数 `id_tag`
2. 调用 `TagDAO::deleteTag($id_tag)`
3. 非 AJAX 请求则重定向回标签列表页

### 6.4 标签合并（重命名到已有标签时）

**动作**：`tag/renameAction`

位置：[app/Controllers/tagController.php](app/Controllers/tagController.php#L192-L225)

当用户将标签 A 重命名为标签 B（标签 B 已存在）时，执行**合并操作**：

```
目标标签不存在 → 直接 updateTagName 修改源标签名称
目标标签已存在 → 合并：
    1. updateEntryTag(sourceId, targetId) 迁移关联
    2. deleteTag(sourceId) 删除源标签
```

`updateEntryTag()` 的内部逻辑（避免违反联合主键唯一性）：

1. **先删重复**：删除那些同一篇文章同时打了源标签和目标标签的源标签关联（否则 UPDATE 会产生重复主键）
2. **再迁移**：将剩余的源标签关联的 `id_tag` 更新为目标 ID

> 代码参考：[app/Models/TagDAO.php](app/Models/TagDAO.php#L173-L202)

### 6.5 删除对 Feed 源标签无影响

注意：删除用户自定义标签**不会影响** `_entry.tags` 字段中存储的 Feed 源标签。两套标签体系独立存储、互不干扰。

---

## 七、标签查询与统计

### 7.1 获取文章的标签

- `getTagsForEntry($id_entry)` - 获取单篇文章的所有标签列表，并附带 `checked` 字段标识该文章是否已打此标签

  位置：[app/Models/TagDAO.php](app/Models/TagDAO.php#L361-L384)

- `getTagsForEntries($entries)` - 批量获取多篇文章的标签，支持大数据量时分批查询（防止超过 SQL 最大参数数量）

  位置：[app/Models/TagDAO.php](app/Models/TagDAO.php#L390-L429)

- `getEntryIdsTagNames($entries)` - 返回 `[e_{entryId} => [标签名1, 标签名2]]` 格式的关联数组，主要用于 API 和 JSON 导出

  位置：[app/Models/TagDAO.php](app/Models/TagDAO.php#L437-L448)

### 7.2 标签统计

- `countEntries($id)` - 统计标签关联的文章总数
- `countNotRead($id)` - 统计标签下的未读文章数
- `listTags($precounts)` - 列出所有标签，`$precounts=true` 时通过 LEFT JOIN 一次查询出未读数，避免 N+1 问题

### 7.3 按标签筛选文章

在 `EntryDAO` 中：
- 通过 JOIN `_entrytag` 表实现按标签筛选文章列表
- `markReadTag()` - 将指定标签下的文章批量标记为已读

> 代码参考：[app/Models/EntryDAO.php](app/Models/EntryDAO.php#L750-L784)

搜索查询中通过 `#标签名` 语法搜索标签时，会在 EntryDAO 中生成：

```sql
-- 按标签 ID 过滤
AND e.id IN (SELECT et.id_entry FROM _entrytag et WHERE et.id_tag IN (...))

-- 或按标签名过滤
AND e.id IN (SELECT et.id_entry FROM _entrytag et, _tag t WHERE et.id_tag = t.id AND t.name IN (...))
```

> 代码参考：[app/Models/EntryDAO.php](app/Models/EntryDAO.php#L1130-L1174)

### 7.4 全局标签缓存

位置：[app/Models/Context.php](app/Models/Context.php#L217-L222)

`FreshRSS_Context::labels()` 使用静态变量缓存标签列表，避免同一次请求内重复查询。

---

## 八、标签属性与过滤动作

标签的 `attributes` 字段（JSON 格式）存储扩展配置，主要用于：

### 8.1 过滤动作配置

使用 `FreshRSS_FilterActionsTrait` 提供的方法管理：

- `filtersAction('label')` - 获取该标签配置的所有过滤规则（`FreshRSS_BooleanSearch` 对象数组）
- `_filtersAction('label', $filters)` - 设置该标签的过滤规则

标签的过滤规则配置在标签更新页面设置，存储为：

```json
{
  "filters": [
    {"search": "intitle:关键词", "actions": ["label"]}
  ]
}
```

> 代码参考：[app/Controllers/tagController.php](app/Controllers/tagController.php#L120-L123)

### 8.2 属性更新方法

- `updateTagAttributes($id, $attributes)` - 整体替换 attributes JSON
- `updateTagAttribute($tag, $key, $value)` - 更新单个属性键值对

---

## 九、Web UI 打标完整时序图

```
┌─────────────┐              ┌──────────────┐              ┌──────────────┐              ┌──────────────┐
│   用户浏览器  │              │  main.js 前端 │              │ tagController │              │  TagDAO 后端  │
└──────┬──────┘              └──────┬───────┘              └──────┬───────┘              └──────┬───────┘
       │ 点击标签图标               │                            │                            │
       │──────────────────────────>│                            │                            │
       │                            │ 1. show_labels_menu()       │                            │
       │                            │    判断是否需要重新加载      │                            │
       │                            │ 2. loadDynamicTags()        │                            │
       │                            │    GET getTagsForEntry      │                            │
       │                            │───────────────────────────>│                            │
       │                            │                            │ getTagsForEntryAction()    │
       │                            │                            │───────────────────────────>│
       │                            │                            │                            │ SELECT LEFT JOIN _tag _entrytag
       │                            │                            │                            │ 返回 [{id,name,checked}]
       │                            │                            │<───────────────────────────│
       │                            │<───────────────────────────│                            │
       │                            │ 3. 渲染 checkbox 列表        │                            │
       │                            │    构建 DOM                 │                            │
       │ 看到标签列表               │<───────────────────────────│                            │
       │<───────────────────────────│                            │                            │
       │                            │                            │                            │
       │ 勾选已有标签 #5             │                            │                            │
       │──────────────────────────>│                            │                            │
       │                            │ 4. stream.onchange 触发      │                            │
       │                            │    解析 tagId=5, entryId=xxx │                            │
       │                            │    POST tagEntry JSON Body   │                            │
       │                            │    {_csrf,id_tag:5,         │                            │
       │                            │     id_entry,checked:true}  │                            │
       │                            │───────────────────────────>│                            │
       │                            │                            │ tagEntryAction()           │
       │                            │                            │───────────────────────────>│
       │                            │                            │                            │ tagEntry(): INSERT IGNORE
       │                            │                            │                            │ INTO _entrytag
       │                            │                            │<───────────────────────────│
       │                            │ 5. onload 回调              │                            │
       │                            │    文章未读? → incUnreadsTag │                            │
       │                            │    #t_5 data-unread +1       │                            │
       │                            │ 6. onloadend 回调            │                            │
       │                            │    移除其他重复下拉菜单 DOM   │                            │
       │ 侧边栏标签 #5 未读数+1     │                            │                            │
       │<───────────────────────────│                            │                            │
       │                            │                            │                            │
       │ 勾选"新建标签"输入"紧急"    │                            │                            │
       │──────────────────────────>│                            │                            │
       │                            │ 7. 解析 tagId=0,            │                            │
       │                            │    name_tag="紧急"          │                            │
       │                            │    POST tagEntry            │                            │
       │                            │    {_csrf,id_tag:0,         │                            │
       │                            │     name_tag:"紧急",...}    │                            │
       │                            │───────────────────────────>│                            │
       │                            │                            │ 8. searchByName("紧急")    │
       │                            │                            │    不存在 → addTag()        │
       │                            │                            │───────────────────────────>│
       │                            │                            │                            │ INSERT _tag → 返回新ID=12
       │                            │                            │<───────────────────────────│
       │                            │                            │ 9. tagEntry(id=12,...)     │
       │                            │                            │───────────────────────────>│
       │                            │                            │                            │ INSERT _entrytag
       │                            │                            │<───────────────────────────│
       │                            │ 10. onloadend 回调          │                            │
       │                            │     forceReloadLabelsList=true │                          │
       │                            │     loadDynamicTags() → 刷新 │                            │
       │                            │     → GET getTagsForEntry  │ │                            │
       │                            │    （现在返回包含 id:12 的列表）│                            │
       │ 菜单显示新标签"紧急"已勾选  │                            │                            │
       │<───────────────────────────│                            │                            │
       │                            │                            │                            │
       │ 打开另一篇文章的标签菜单    │                            │                            │
       │──────────────────────────>│                            │                            │
       │                            │ 11. show_labels_menu()      │                            │
       │                            │     检查 forceReloadLabelsList │                          │
       │                            │     = true → 删除旧缓存     │                            │
       │                            │     → 重新 loadDynamicTags  │                            │
       │                            │     （包含新标签"紧急"）     │                            │
       │ 看到新标签"紧急"            │                            │                            │
       │<───────────────────────────│                            │                            │
```

---

## 十、总结

### 10.1 关键设计特点

1. **双标签体系并存**：Feed 源标签（只读，存储在 `_entry.tags`）与用户自定义标签（可管理，`_tag` + `_entrytag` 关联表）独立运作
2. **规范化多对多设计**：用户标签使用独立实体表 + 关联表，符合数据库第三范式，支持一篇文章多标签、一标签多文章
3. **外键级联删除**：通过 `ON DELETE CASCADE` 自动维护引用完整性，删除标签或文章时无需手动清理关联
4. **懒加载统计**：`nbEntries` 和 `nbUnread` 属性按需查询，标签列表支持预计算（`$precounts=true`）避免 N+1
5. **打标时自动创建**：用户打不存在的标签时自动创建，无需预先在管理页维护
6. **过滤规则驱动自动打标**：标签可配置布尔搜索规则，新文章入库时自动匹配并打标
7. **多数据库适配**：通过工厂模式 + DAO 继承，支持 MySQL/SQLite/PostgreSQL，差异仅在 SQL 方言细节
8. **多操作入口**：Web UI、Controller 直接调用、GReader API 三种方式管理标签和打标
9. **前端下拉菜单惰性加载 + 缓存**：首次打开才 AJAX 拉取，复用缓存；仅新建标签时触发全局强制刷新
10. **未读数乐观回写**：AJAX 成功后直接更新 DOM 中 `data-unread` 属性，不等待页面刷新，保证用户体验流畅

### 10.2 数据流向图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           标签创建                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  标签管理页 ──POST──→ tag/addAction ──→ addTag() ──→ _tag 表                 │
│                        ↑ 检查不与分类重名                                    │
│                                                                             │
│  文章打标 ──POST──→ tag/tagEntryAction ──→ id_tag==0? ──否──→ searchByName() │
│                        │                              ↓（不存在）            │
│                        │                           addTag() → _tag           │
│                        └──是/否──→ tagEntry() ──────────────→ _entrytag      │
│                                                                             │
│  GReader API ──→ edit-tag ──→ 解析 label/xxx ──→ 自动创建+批量打标            │
│                                                                             │
│  Feed 更新 ──→ commitNewEntries() ──→ applyLabelActions()                   │
│                        │                               │                    │
│                        │                               └→ 过滤规则匹配       │
│                        │                                     ↓               │
│                        └──────────────────────────→ tagEntries() 批量打标    │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        标签删除                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  tag/deleteAction ──→ deleteTag(id) ──→ DELETE FROM _tag                    │
│                                                │                             │
│                                                └── DB 外键级联 ──→           │
│                                                    DELETE FROM               │
│                                                    _entrytag WHERE           │
│                                                    id_tag = 被删ID           │
│                                                                             │
│  tag/renameAction(重名时) ──→ updateEntryTag(迁移关联)                      │
│                              └──→ deleteTag(删除源标签)                      │
└─────────────────────────────────────────────────────────────────────────────┘
```
