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

> 代码参考：[install.sql.mysql.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/SQL/install.sql.mysql.php#L96-L103)

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

> 代码参考：[install.sql.mysql.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/SQL/install.sql.mysql.php#L105-L113)

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

> 代码参考：[Entry.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/Entry.php#L38-L39)、[EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/EntryDAO.php#L231-L233)

---

## 三、核心模型与 DAO

### 3.1 标签模型 `FreshRSS_Tag`

位置：[app/Models/Tag.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/Tag.php)

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

**基类**：[app/Models/TagDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/TagDAO.php)

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

**SQLite 子类**：[TagDAOSQLite.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/TagDAOSQLite.php) - 仅覆盖 `sqlIgnore()` 返回 `OR IGNORE`

**PostgreSQL 子类**：[TagDAOPGSQL.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/TagDAOPGSQL.php) - 覆盖 `sqlIgnore()` 和 `sqlResetSequence()`

---

## 四、标签创建流程

### 4.1 主动创建（标签管理页面）

**入口 Controller**：`tag/addAction`

位置：[tagController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Controllers/tagController.php#L159-L186)

**流程**：
1. 用户在标签管理页面提交表单（POST 请求，参数 `name`）
2. 校验名称不能与**分类**重名（调用 `CategoryDAO::searchByName()`）
3. 校验名称不能与**已有标签**重名（调用 `TagDAO::searchByName()`）
4. 调用 `addTag(['name' => $name])` 插入 `_tag` 表
5. 返回成功/失败通知，重定向回标签列表页

> **注意**：标签名称不能与分类名称重名，因为二者在搜索和导航中使用相同的前缀语法。

### 4.2 隐式创建（打标时自动创建）

**入口 Controller**：`tag/tagEntryAction`

位置：[tagController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Controllers/tagController.php#L31-L64)

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

### 5.1 Web UI 手动打标

**视图**：文章详情中的标签管理区域

相关视图：
- [getTagsForEntry.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/views/tag/getTagsForEntry.phtml) - AJAX 获取文章标签 JSON
- [tag/index.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/views/tag/index.phtml) - 标签管理页面

**交互流程**：
```
文章底部 → 我的标签下拉 → 勾选/取消勾选标签 → AJAX POST 到 tag/tagEntryAction
```

### 5.2 控制器打标动作 `tagEntryAction`

位置：[tagController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Controllers/tagController.php#L31-L64)

**请求参数**：
- `id_tag` (int) - 标签 ID（可选，与 `name_tag` 二选一）
- `name_tag` (string) - 标签名称（可选，不存在则自动创建）
- `id_entry` (string) - 文章 ID
- `checked` (bool) - `true`=打标，`false`=取消打标
- `ajax` (bool) - 是否 AJAX 请求（影响响应方式）

**底层调用 `TagDAO::tagEntry()`**：

```php
// checked=true: INSERT IGNORE INTO _entrytag(id_tag, id_entry) VALUES(...)
// checked=false: DELETE FROM _entrytag WHERE id_tag=... AND id_entry=...
public function tagEntry(int $id_tag, string $id_entry, bool $checked = true): bool
```

> 代码参考：[TagDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/TagDAO.php#L302-L324)

### 5.3 批量打标 `tagEntries()`

位置：[TagDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/TagDAO.php#L330-L356)

支持一次对多篇文章打多个标签，使用 `INSERT IGNORE` 批量插入。

**输入格式**：
```php
[
    ['id_tag' => 1, 'id_entry' => '123456'],
    ['id_tag' => 2, 'id_entry' => '123456'],
    ['id_tag' => 1, 'id_entry' => '789012'],
]
```

### 5.4 自动打标（过滤规则驱动）

**核心机制**：通过 `FreshRSS_FilterActionsTrait` 为标签配置过滤规则，当新文章入库时自动匹配并打标。

位置：[FilterActionsTrait.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/FilterActionsTrait.php)

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
- 触发入口：[feedController.php::applyLabelActions()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Controllers/feedController.php#L879-L901)
- 过滤匹配：[FilterActionsTrait.php::applyFilterActions()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/FilterActionsTrait.php#L125-L153)
- 过滤规则配置存储在标签的 `attributes.filters` JSON 字段中

#### 过滤规则匹配动作

`applyFilterActions()` 支持三种动作：
- `read` - 自动标记为已读
- `star` - 自动加星标
- `label` - 自动打标签（设置 `$applyLabel=true`）

> **重要**：自动打标只对**新入库的文章**生效（`!$entry->isUpdated()`），避免覆盖用户手动操作。

### 5.5 GReader API 打标

**接口**：`/api/greader.php` 的 `edit-tag` 端点

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

> 代码参考：[install.sql.mysql.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/SQL/install.sql.mysql.php#L109-L110)

### 6.2 应用层删除逻辑

**DAO 方法**：`FreshRSS_TagDAO::deleteTag($id)`

位置：[TagDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/TagDAO.php#L122-L139)

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

位置：[tagController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Controllers/tagController.php#L66-L85)

**流程**：
1. 接收 POST 参数 `id_tag`
2. 调用 `TagDAO::deleteTag($id_tag)`
3. 非 AJAX 请求则重定向回标签列表页

### 6.4 标签合并（重命名到已有标签时）

**动作**：`tag/renameAction`

位置：[tagController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Controllers/tagController.php#L192-L225)

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

> 代码参考：[TagDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/TagDAO.php#L173-L202)

### 6.5 删除对 Feed 源标签无影响

注意：删除用户自定义标签**不会影响** `_entry.tags` 字段中存储的 Feed 源标签。两套标签体系独立存储、互不干扰。

---

## 七、标签查询与统计

### 7.1 获取文章的标签

- `getTagsForEntry($id_entry)` - 获取单篇文章的所有标签列表，并附带 `checked` 字段标识该文章是否已打此标签

  位置：[TagDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/TagDAO.php#L361-L384)

- `getTagsForEntries($entries)` - 批量获取多篇文章的标签，支持大数据量时分批查询（防止超过 SQL 最大参数数量）

  位置：[TagDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/TagDAO.php#L390-L429)

- `getEntryIdsTagNames($entries)` - 返回 `[e_{entryId} => [标签名1, 标签名2]]` 格式的关联数组，主要用于 API 和 JSON 导出

  位置：[TagDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/TagDAO.php#L437-L448)

### 7.2 标签统计

- `countEntries($id)` - 统计标签关联的文章总数
- `countNotRead($id)` - 统计标签下的未读文章数
- `listTags($precounts)` - 列出所有标签，`$precounts=true` 时通过 LEFT JOIN 一次查询出未读数，避免 N+1 问题

### 7.3 按标签筛选文章

在 `EntryDAO` 中：
- 通过 JOIN `_entrytag` 表实现按标签筛选文章列表
- `markReadTag()` - 将指定标签下的文章批量标记为已读

> 代码参考：[EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/EntryDAO.php#L750-L784)

搜索查询中通过 `#标签名` 语法搜索标签时，会在 EntryDAO 中生成：

```sql
-- 按标签 ID 过滤
AND e.id IN (SELECT et.id_entry FROM _entrytag et WHERE et.id_tag IN (...))

-- 或按标签名过滤
AND e.id IN (SELECT et.id_entry FROM _entrytag et, _tag t WHERE et.id_tag = t.id AND t.name IN (...))
```

> 代码参考：[EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/EntryDAO.php#L1130-L1174)

### 7.4 全局标签缓存

位置：[Context.php::labels()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Models/Context.php#L217-L222)

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

> 代码参考：[tagController.php::updateAction()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-FreshRSS/app/Controllers/tagController.php#L120-L123)

### 8.2 属性更新方法

- `updateTagAttributes($id, $attributes)` - 整体替换 attributes JSON
- `updateTagAttribute($tag, $key, $value)` - 更新单个属性键值对

---

## 九、总结

### 9.1 关键设计特点

1. **双标签体系并存**：Feed 源标签（只读，存储在 `_entry.tags`）与用户自定义标签（可管理，`_tag` + `_entrytag` 关联表）独立运作
2. **规范化多对多设计**：用户标签使用独立实体表 + 关联表，符合数据库第三范式，支持一篇文章多标签、一标签多文章
3. **外键级联删除**：通过 `ON DELETE CASCADE` 自动维护引用完整性，删除标签或文章时无需手动清理关联
4. **懒加载统计**：`nbEntries` 和 `nbUnread` 属性按需查询，标签列表支持预计算（`$precounts=true`）避免 N+1
5. **打标时自动创建**：用户打不存在的标签时自动创建，无需预先在管理页维护
6. **过滤规则驱动自动打标**：标签可配置布尔搜索规则，新文章入库时自动匹配并打标
7. **多数据库适配**：通过工厂模式 + DAO 继承，支持 MySQL/SQLite/PostgreSQL，差异仅在 SQL 方言细节
8. **多操作入口**：Web UI、Controller 直接调用、GReader API 三种方式管理标签和打标

### 9.2 数据流向图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        标签创建                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  标签管理页 ──POST──→ tag/addAction ──→ addTag() ──→ _tag 表        │
│                        ↑ 检查不与分类重名                            │
│                                                                     │
│  文章打标 ──POST──→ tag/tagEntryAction ──→ 已存在? ──否──→ addTag() │
│                        │                                    ↓       │
│                        └──是──→ tagEntry() ──────────────→ _entrytag│
│                                                                     │
│  GReader API ──→ edit-tag ──→ 解析 label/xxx ──→ 自动创建+批量打标  │
│                                                                     │
│  Feed 更新 ──→ commitNewEntries() ──→ applyLabelActions()           │
│                        │                         │                  │
│                        │                         └→ 过滤规则匹配    │
│                        │                               ↓            │
│                        └──────────────────────→ tagEntries() 批量打标│
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                        标签删除                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  tag/deleteAction ──→ deleteTag(id) ──→ DELETE FROM _tag            │
│                                                │                     │
│                                                └── DB 级联 ──→       │
│                                                    DELETE FROM       │
│                                                    _entrytag WHERE   │
│                                                    id_tag = 被删ID   │
│                                                                     │
│  tag/renameAction(重名时) ──→ updateEntryTag(迁移关联)               │
│                              └──→ deleteTag(删除源标签)              │
└─────────────────────────────────────────────────────────────────────┘
```
