# FreshRSS 搜索语法与过滤动作协作分析

## 1. 整体架构概述

FreshRSS 的搜索过滤系统采用清晰的三层架构设计，各层职责明确，协作边界清晰：

```
┌─────────────────────────────────────────────────────────┐
│                  结果呈现层 (Presentation)              │
│  [indexController.php] [Context.php] [*.phtml]         │
│  • HTTP 请求处理                                        │
│  • 上下文状态管理                                       │
│  • 视图渲染与分页                                       │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼  FreshRSS_BooleanSearch 对象
┌─────────────────────────────────────────────────────────┐
│                  条件构造层 (DAO)                       │
│  [EntryDAO.php]                                         │
│  • SQL 查询构建                                         │
│  • 数据库语法适配                                       │
│  • 参数绑定与安全                                       │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼  结构化搜索条件
┌─────────────────────────────────────────────────────────┐
│                  查询解析层 (Model)                     │
│  [Search.php] [BooleanSearch.php] [FilterAction.php]   │
│  • 搜索语法解析                                         │
│  • 布尔逻辑处理                                         │
│  • 正则表达式支持                                       │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼  原始搜索字符串
                     用户输入 / HTTP 请求
```

---

## 2. 查询解析层 (Query Parsing Layer)

### 2.1 核心文件与职责

| 文件 | 核心类 | 主要职责 |
|------|--------|----------|
| [Search.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/Search.php) | `FreshRSS_Search` | 原子搜索条件解析 |
| [BooleanSearch.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/BooleanSearch.php) | `FreshRSS_BooleanSearch` | 布尔逻辑与嵌套结构解析 |
| [FilterAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/FilterAction.php) | `FreshRSS_FilterAction` | 过滤动作与搜索条件绑定 |

### 2.2 支持的搜索语法

#### 基础搜索语法
```
intitle:keyword       # 标题搜索
intext:keyword        # 正文搜索
author:name           # 作者搜索
inurl:domain          # URL搜索
#tagname              # 标签搜索
f:1,2,3               # 按Feed ID过滤
c:4,5                 # 按分类ID过滤
L:10,11               # 按标签ID过滤
S:1,2                 # 按保存的查询ID过滤
search:"query name"   # 按保存的查询名称过滤
date:2024-01-01/      # 日期范围（接收日期）
pubdate:2024-01/      # 发布日期
userdate:P7D/         # 用户修改日期
mdate:PT24H/          # 服务器修改日期
e:12345               # 按条目ID
```

#### 高级语法
```
/pattern/i            # 正则表达式（支持i/m修饰符）
"exact phrase"        # 精确短语匹配
-keyword              # 排除（NOT）
!keyword              # 排除（NOT，与-等价）
A OR B                # 逻辑或
(A AND B) OR C        # 括号分组
```

### 2.3 查询解析实现流程

#### 2.3.1 `FreshRSS_Search` 解析流程

[Search.php#L122-L169](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/Search.php#L122-L169)

```php
public function __construct(string $input) {
    $input = self::cleanSearch($input);          // 清理空白字符
    $input = self::unescape($input);             // 去除转义
    $input = FreshRSS_BooleanSearch::unescapeLiterals($input);  // 还原字面量
    $this->raw_input = $input;

    // 按优先级顺序解析各种搜索条件（先处理NOT条件，再处理正向条件）
    $input = $this->parseNotEntryIds($input);
    $input = $this->parseNotFeedIds($input);
    // ... 更多 parseNot* 方法 ...
    
    $input = $this->parseEntryIds($input);
    $input = $this->parseFeedIds($input);
    // ... 更多 parse* 方法 ...
    
    $input = $this->parseQuotedSearch($input);   // 解析引号包裹的精确搜索
    $input = $this->parseNotSearch($input);      // 解析排除搜索
    $this->parseSearch($input);                  // 解析剩余普通搜索词
}
```

**设计特点**：
- 每个 `parse*` 方法负责一种特定语法的解析
- 解析后从输入字符串中移除已匹配的部分
- 剩余内容继续传递给下一个解析方法
- 最终结果存储在类的属性中，通过 getter 方法访问

#### 2.3.2 单个条件解析示例 - 标题搜索

[Search.php#L1072-L1090](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/Search.php#L1072-L1090)

```php
private function parseIntitleSearch(string $input): string {
    // 1. 先匹配正则表达式形式：intitle:/pattern/i
    if (preg_match_all('#\bintitle:(?P<search>/.*?(?<!\\)/[im]*)#', $input, $matches)) {
        $this->intitle_regex = $matches['search'];
        $input = str_replace($matches[0], '', $input);
    }
    // 2. 再匹配引号包裹形式：intitle:"exact phrase"
    if (preg_match_all('/\bintitle:(?P<delim>[\'"])(?P<search>.*)(?P=delim)/U', $input, $matches)) {
        $this->intitle = $matches['search'];
        $input = str_replace($matches[0], '', $input);
    }
    // 3. 最后匹配普通单词形式：intitle:keyword
    if (preg_match_all('/\bintitle:(?P<search>[^\s"]*)/', $input, $matches)) {
        $this->intitle = array_merge($this->intitle ?? [], $matches['search']);
        $input = str_replace($matches[0], '', $input);
    }
    return $input;
}
```

#### 2.3.3 `FreshRSS_BooleanSearch` 解析流程

[BooleanSearch.php#L19-L48](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/BooleanSearch.php#L19-L48)

```php
public function __construct(
    string $input,
    int $level = 0,
    private readonly string $operator = 'AND',
    bool $allowUserQueries = true,
    bool $expandUserQueries = true
) {
    if ($level === 0) {
        $input = self::escapeLiterals($input);           // 转义字面量中的括号和OR
        $input = $this->parseUserQueryNames($input, $allowUserQueries);  // 展开保存的查询
        $input = $this->parseUserQueryIds($input, $allowUserQueries);
    }

    $input = self::consistentOrParentheses($input);     // 统一OR表达式的括号格式

    // 优先解析括号，否则解析OR分段
    $this->parseParentheses($input, $level) || $this->parseOrSegments($input);
}
```

**括号解析逻辑** [BooleanSearch.php#L277-L392](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/BooleanSearch.php#L277-L392)：
- 遍历字符串，识别未转义的括号
- 遇到 `(` 时递归创建子 `BooleanSearch` 对象
- 根据括号前的运算符（`AND`/`OR`/`AND NOT`/`OR NOT`）确定组合方式
- 支持复杂嵌套：`(A OR (B AND C)) AND NOT D`

### 2.4 数据结构

```
FreshRSS_BooleanSearch
├── operator: 'AND'|'OR'|'AND NOT'|'OR NOT'
├── searches: array<FreshRSS_BooleanSearch|FreshRSS_Search>
└── raw_input: string

FreshRSS_Search
├── entry_ids: list<string>|null
├── feed_ids: list<int>|null
├── category_ids: list<int>|null
├── label_ids: list<list<int>|'*'>|null
├── intitle: list<string>|null
├── intitle_regex: list<string>|null
├── intext: list<string>|null
├── intext_regex: list<string>|null
├── author: list<string>|null
├── author_regex: list<string>|null
├── tags: list<string>|null
├── tags_regex: list<string>|null
├── inurl: list<string>|null
├── inurl_regex: list<string>|null
├── search: list<string>|null        # 全字段搜索
├── search_regex: list<string>|null
├── min_date/max_date: int|false|null
├── min_pubdate/max_pubdate: int|false|null
├── ...（各字段均有对应的 not_* 版本）
└── raw_input: string
```

### 2.5 边界职责

**查询解析层只负责：**
- ✅ 字符串模式匹配与提取
- ✅ 语法验证与规范化
- ✅ 数据结构转换（字符串 → 对象）
- ✅ 用户查询展开（`S:` / `search:`）

**查询解析层不负责：**
- ❌ 数据库查询构建
- ❌ 业务逻辑判断
- ❌ 权限验证
- ❌ 数据渲染

---

## 3. 条件构造层 (Condition Construction Layer)

### 3.1 核心文件与职责

| 文件 | 核心类 | 主要职责 |
|------|--------|----------|
| [EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/EntryDAO.php) | `FreshRSS_EntryDAO` | SQL查询构建与执行 |

### 3.2 核心方法调用链

```
listWhere()
    ↓
listWhereRaw()
    ↓
sqlListWhere()
    ↓
sqlListEntriesWhere()
    ↓
sqlBooleanSearch()  ← 核心：将搜索对象转为SQL
    ↓
sqlRegex()         ← 正则表达式SQL适配
```

### 3.3 `sqlBooleanSearch` - 搜索条件转SQL

[EntryDAO.php#L950-L1381](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/EntryDAO.php#L950-L1381)

这是条件构造层的核心方法，负责将 `FreshRSS_BooleanSearch` 对象树转换为 SQL WHERE 子句。

#### 3.3.1 布尔逻辑处理

```php
public static function sqlBooleanSearch(string $alias, FreshRSS_BooleanSearch $filters, int $level = 0): array {
    $search = '';
    $values = [];

    foreach ($filters->searches() as $filter) {
        if ($filter instanceof FreshRSS_BooleanSearch) {
            // 递归处理嵌套布尔搜索
            [$filterValues, $filterSearch] = self::sqlBooleanSearch($alias, $filter, $level + 1);
            
            if ($filterSearch !== '') {
                // 根据运算符组合
                if ($search !== '') {
                    $search .= $filter->operator();  // AND / OR / AND NOT / OR NOT
                }
                $search .= ' (' . $filterSearch . ') ';
                $values = array_merge($values, $filterValues);
            }
            continue;
        }
        
        // 处理原子搜索条件（FreshRSS_Search）
        $sub_search = '';
        
        // 1. ID过滤
        if ($filter->getEntryIds() !== null) {
            $sub_search .= 'AND ' . $alias . 'id IN (';
            foreach ($filter->getEntryIds() as $entry_id) {
                $sub_search .= '?,';
                $values[] = $entry_id;
            }
            $sub_search = rtrim($sub_search, ',') . ') ';
        }
        
        // 2. 日期范围过滤
        if ($filter->getMinDate() !== null) {
            $sub_search .= 'AND ' . $alias . 'id >= ? ';
            $values[] = "{$filter->getMinDate()}000000";
        }
        if ($filter->getMaxDate() !== null) {
            $sub_search .= 'AND ' . $alias . 'id <= ? ';
            $values[] = "{$filter->getMaxDate()}000000";
        }
        
        // 3. Feed/Category过滤
        if ($filter->getFeedIds() !== null) {
            $sub_search .= 'AND ' . $alias . 'id_feed IN (';
            foreach ($filter->getFeedIds() as $feed_id) {
                $sub_search .= '?,';
                $values[] = $feed_id;
            }
            $sub_search = rtrim($sub_search, ',') . ') ';
        }
        
        // 4. 标签过滤（支持子查询）
        if ($filter->getLabelIds() !== null) {
            foreach ($filter->getLabelIds() as $label_ids) {
                if ($label_ids === '*') {
                    $sub_search .= 'AND EXISTS (SELECT et.id_tag FROM `_entrytag` et WHERE et.id_entry = ' . $alias . 'id) ';
                } else {
                    $sub_search .= 'AND ' . $alias . 'id IN (SELECT et.id_entry FROM `_entrytag` et WHERE et.id_tag IN (';
                    foreach ($label_ids as $label_id) {
                        $sub_search .= '?,';
                        $values[] = $label_id;
                    }
                    $sub_search = rtrim($sub_search, ',') . ')) ';
                }
            }
        }
        
        // 5. 文本搜索（支持LIKE和正则）
        if ($filter->getIntitle() !== null) {
            foreach ($filter->getIntitle() as $title) {
                $sub_search .= 'AND ' . $alias . 'title LIKE ? ';
                $values[] = "%{$title}%";
            }
        }
        if ($filter->getIntitleRegex() !== null) {
            foreach ($filter->getIntitleRegex() as $title) {
                $sub_search .= 'AND ' . static::sqlRegex($alias . 'title', $title, $values) . ' ';
            }
        }
        
        // 6. 全字段搜索
        if ($filter->getSearch() !== null) {
            foreach ($filter->getSearch() as $search_value) {
                if (static::isCompressed()) {  // MySQL压缩存储优化
                    $sub_search .= "AND CONCAT({$alias}title, '\n', UNCOMPRESS({$alias}content_bin)) LIKE ? ";
                    $values[] = "%{$search_value}%";
                } else {
                    $sub_search .= 'AND (' . $alias . 'title LIKE ? OR ' . $alias . 'content LIKE ?) ';
                    $values[] = "%{$search_value}%";
                    $values[] = "%{$search_value}%";
                }
            }
        }
        
        // ...（NOT条件处理类似，使用NOT LIKE/NOT IN等）
        
        // 组合OR条件（同一BooleanSearch内的多个Search对象为OR关系）
        if ($sub_search != '') {
            if ($isOpen) {
                $search .= ' OR ';
            } else {
                $isOpen = true;
            }
            $search .= '(' . trim(substr($sub_search, 4)) . ')';  // 移除开头多余的'AND '
        }
    }

    return [ $values, $search ];
}
```

#### 3.3.2 数据库适配 - 正则表达式

[EntryDAO.php#L50-L91](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/EntryDAO.php#L50-L91)

```php
protected static function sqlRegex(string $expression, string $regex, array &$values): string {
    $matches = static::regexToSql($regex);  // 解析 /pattern/im 格式
    
    if (isset($matches['pattern'])) {
        $matchType = $matches['matchType'] ?? '';
        
        if ($databaseDAOMySQL->isMariaDB()) {
            // MariaDB 使用 REGEXP 操作符，模式修饰符内嵌
            if (str_contains($matchType, 'm')) {
                $matches['pattern'] = '(?m)' . $matches['pattern'];
            }
            if (str_contains($matchType, 'i')) {
                $matches['pattern'] = '(?i)' . $matches['pattern'];
            }
            $values[] = $matches['pattern'];
            return "{$expression} REGEXP ?";
        } else {  // MySQL
            // MySQL 使用 REGEXP_LIKE 函数，修饰符作为第三个参数
            if (!str_contains($matchType, 'i')) {
                $matchType .= 'c';  // case-sensitive
            }
            $values[] = $matches['pattern'];
            return "REGEXP_LIKE({$expression},?,'{$matchType}')";
        }
    }
    return '';
}
```

### 3.4 `sqlListEntriesWhere` - 基础查询条件

[EntryDAO.php#L1394-L1519](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/EntryDAO.php#L1394-L1519)

处理通用查询条件，与搜索条件正交：

```php
protected function sqlListEntriesWhere(
    string $alias = '',
    int $state = FreshRSS_Entry::STATE_ALL,
    ?FreshRSS_BooleanSearch $filters = null,
    string $id_min = '0',
    string $id_max = '0',
    string $sort = 'id',
    string $order = 'DESC',
    string $continuation_id = '0',
    array $continuation_values = []
): array {
    $search = ' ';
    $values = [];

    // 1. 文章状态过滤（已读/未读/收藏）
    if ($state & FreshRSS_Entry::STATE_ANDS) {
        if ($state & FreshRSS_Entry::STATE_NOT_READ) {
            $search .= 'AND (' . $alias . 'is_read=0) ';
        }
        // ... 其他状态
    }

    // 2. ID范围过滤
    if ($id_max !== '0') {
        $search .= 'AND ' . $alias . 'id <= ? ';
        $values[] = $id_max;
    }
    if ($id_min !== '0') {
        $search .= 'AND ' . $alias . 'id >= ? ';
        $values[] = $id_min;
    }

    // 3. Keyset分页（基于排序字段的游标分页）
    if ($continuation_id !== '0' && in_array($sort, [...])) {
        $sign = $order === 'ASC' ? '>' : '<';
        // 处理复合排序的分页条件
        $search .= "AND ({$orderBy} {$sign} ? OR ({$orderBy} = ? AND {$alias}id {$sign}= ?)) ";
        $values[] = $continuation_values[0];
        $values[] = $continuation_values[0];
        $values[] = $continuation_id;
    }

    // 4. 集成搜索条件
    if ($filters !== null && count($filters->searches()) > 0) {
        [$filterValues, $filterSearch] = self::sqlBooleanSearch($alias, $filters);
        if ($filterSearch !== '') {
            $search .= 'AND (' . $filterSearch . ') ';
            $values = array_merge($values, $filterValues);
        }
    }

    return [$values, $search];
}
```

### 3.5 `sqlListWhere` - 完整SQL构建

[EntryDAO.php#L1535-L1643](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/EntryDAO.php#L1535-L1643)

整合所有查询条件，构建最终的SQL查询：

```php
private function sqlListWhere(
    string $type = 'a',    // 查询类型：a=全部, c=分类, f=Feed, t=标签...
    int $id = 0,
    int $state = FreshRSS_Entry::STATE_ALL,
    ?FreshRSS_BooleanSearch $filters = null,
    // ... 其他参数
): array {
    $where = '';
    $values = [];
    
    // 根据类型添加基础过滤条件
    switch ($type) {
        case 'a':  // 全部主流信息流
            $where .= 'f.priority >= ' . min(FreshRSS_Feed::PRIORITY_MAIN_STREAM, ...) . ' ';
            break;
        case 'c':  // 分类
            $where .= 'f.priority >= ' . min(FreshRSS_Feed::PRIORITY_CATEGORY, ...) . ' ';
            $where .= 'AND f.category=? ';
            $values[] = $id;
            break;
        case 'f':  // Feed
            $where .= 'e.id_feed=? ';
            $values[] = $id;
            break;
        case 't':  // 标签
            $where .= 'et.id_tag=? ';
            $values[] = $id;
            break;
        // ... 其他类型
    }

    // 构建排序表达式
    $orderBy = match ($sort) {
        'id' => 'e.id',
        'date' => 'e.date',
        'title' => 'e.title',
        'rand' => static::sqlRandom(),
        // ...
    };

    // 集成通用查询条件
    [$searchValues, $search] = $this->sqlListEntriesWhere(
        alias: 'e.', state: $state, filters: $filters, ...
    );

    // 构建最终SQL（先查ID，再关联查详情，性能优化）
    return [array_merge($values, $searchValues), 'SELECT e.id FROM `_entry` e '
        . 'INNER JOIN `_feed` f ON f.id = e.id_feed '
        . ($sort === 'c.name' ? 'INNER JOIN `_category` c ON c.id = f.category ' : '')
        . ($type === 't' ? 'INNER JOIN `_entrytag` et ON et.id_entry = e.id ' : '')
        . 'WHERE ' . $where . $search
        . 'ORDER BY ' . $orderBy . ' ' . $order
        . ($limit > 0 ? ' LIMIT ' . $limit : '')
    ];
}
```

### 3.6 边界职责

**条件构造层只负责：**
- ✅ SQL语法构建与数据库适配
- ✅ 参数绑定与SQL注入防护
- ✅ 查询性能优化（延迟关联、索引提示）
- ✅ 分页逻辑实现

**条件构造层不负责：**
- ❌ 搜索字符串解析
- ❌ 业务逻辑判断（除了必要的权限过滤）
- ❌ 结果数据格式化
- ❌ HTTP请求处理

---

## 4. 结果呈现层 (Result Rendering Layer)

### 4.1 核心文件与职责

| 文件 | 核心类 | 主要职责 |
|------|--------|----------|
| [indexController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Controllers/indexController.php) | `FreshRSS_index_Controller` | 主控制器，协调请求处理 |
| [Context.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/Context.php) | `FreshRSS_Context` | 全局状态容器 |
| [searchController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Controllers/searchController.php) | `FreshRSS_search_Controller` | 高级搜索表单处理 |
| [normal.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/views/index/normal.phtml) | - | 视图模板 |

### 4.2 请求处理流程

#### 4.2.1 高级搜索表单提交流程

[searchController.php#L71-L219](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Controllers/searchController.php#L71-L219)

```php
public function submitAction(): void {
    // 1. 获取表单参数
    $searchTerms = [];

    // 2. 构建OR子句（支持多行输入，每行一个值）
    $freeTextClause = self::buildOrClause(Minz_Request::paramString('free_text'));
    if ($freeTextClause !== '') {
        $searchTerms[] = $freeTextClause;
    }

    $titleClause = self::buildOrClause(Minz_Request::paramString('title'), 'intitle:');
    if ($titleClause !== '') {
        $searchTerms[] = $titleClause;
    }

    // 3. 构建日期范围
    if ($dateNumber > 0 && $dateUnit !== '') {
        $searchTerms[] = "date:{$prefix}{$dateNumber}{$dateUnit}";
    } elseif ($dateFrom !== '' || $dateTo !== '') {
        $searchTerms[] = "date:$dateFrom/$dateTo";
    }

    // 4. 构建ID过滤
    $feedIds = Minz_Request::paramArrayInt('feed_ids');
    if (!empty($feedIds)) {
        $searchTerms[] = 'f:' . implode(',', $feedIds);
    }

    // 5. 组合所有条件，重定向到主视图
    $searchQuery = implode(' ', $searchTerms);
    Minz_Request::forward([
        'c' => 'index',
        'a' => 'index',
        'params' => ['search' => $searchQuery],
    ], redirect: true);
}
```

**`buildOrClause` 辅助方法** [searchController.php#L43-L66](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Controllers/searchController.php#L43-L66)：
```php
private static function buildOrClause(string $rawValue, string $prefix = ''): string {
    $lines = preg_split('/[\r\n]+/', $rawValue);
    $terms = [];
    foreach ($lines as $line) {
        $line = trim($line, " \n\r\t\v\0\"'");
        if ($line === '') continue;
        $quoted = str_contains($line, ' ') && !str_starts_with($line, '/') ? "'$line'" : $line;
        $terms[] = $prefix . $quoted;
    }
    if (empty($terms)) return '';
    if (count($terms) === 1) return $terms[0];
    return '(' . implode(' OR ', $terms) . ')';
}
```

#### 4.2.2 主视图流程

[indexController.php#L104-L196](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Controllers/indexController.php#L104-L196)

```php
public function normalAction(): void {
    // 1. 权限检查
    if (!FreshRSS_Auth::hasAccess() && !$allow_anonymous) {
        Minz_Request::forward(['c' => 'auth', 'a' => 'login']);
        return;
    }

    // 2. 更新上下文（关键：解析搜索参数）
    try {
        FreshRSS_Context::updateUsingRequest(true);
    } catch (FreshRSS_Context_Exception $e) {
        Minz_Error::error(404);
    }

    // 3. 设置视图数据
    $this->view->categories = FreshRSS_Context::categories();
    $this->view->rss_title = FreshRSS_Context::$name . ' | ' . FreshRSS_View::title();
    
    // 4. 注册回调（惰性加载优化）
    $this->view->callbackBeforeEntries = static function (FreshRSS_View $view) {
        try {
            // +1 用于分页判断（是否有下一页）
            $view->entries = FreshRSS_index_Controller::listEntriesByContext(
                FreshRSS_Context::$number + 1
            );
            ob_start();  // 逐条输出缓冲
        } catch (FreshRSS_EntriesGetter_Exception $e) {
            Minz_Error::error(404);
        }
    };

    $this->view->callbackBeforePagination = static function (?FreshRSS_View $view, int $nbEntries, FreshRSS_Entry $lastEntry) {
        if ($nbEntries > FreshRSS_Context::$number) {
            // 有更多数据，设置下一页游标
            ob_clean();
            FreshRSS_Context::$continuation_id = $lastEntry->id();
        } else {
            FreshRSS_Context::$continuation_id = '0';
        }
        ob_end_flush();
    };
}
```

#### 4.2.3 上下文更新 - 搜索参数解析入口

[Context.php#L239-L265](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/Context.php#L239-L265)

```php
public static function updateUsingRequest(bool $computeStatistics): void {
    // ... 其他上下文初始化 ...
    
    // 关键：将HTTP请求中的search参数解析为搜索对象
    self::$search = new FreshRSS_BooleanSearch(
        Minz_Request::paramString('search', plaintext: true)
    );
    
    // ... 解析其他参数（sort, order, state等） ...
}
```

#### 4.2.4 数据获取

[indexController.php#L349-L413](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Controllers/indexController.php#L349-L413)

```php
public static function listEntriesByContext(?int $postsPerPage = null): Generator {
    $entryDAO = FreshRSS_Factory::createEntryDao();

    // 1. 获取当前浏览类型（全部/分类/Feed/标签）
    $get = FreshRSS_Context::currentGet(true);
    [$type, $id] = is_array($get) ? [$get[0], (int)$get[1]] : [$get, 0];

    // 2. 构建分页游标值
    $continuation_values = [];
    if (FreshRSS_Context::$continuation_id !== '0') {
        // 查询上一页最后一条记录，获取排序字段值用于keyset分页
        $pagingEntry = $entryDAO->searchById(FreshRSS_Context::$continuation_id);
        $continuation_values[] = match (FreshRSS_Context::$sort) {
            'date' => $pagingEntry->date(raw: true),
            'title' => $pagingEntry->title(),
            // ...
        };
    }

    // 3. 调用DAO层获取数据（返回Generator）
    yield from $entryDAO->listWhere(
        $type, $id, 
        FreshRSS_Context::$state, 
        FreshRSS_Context::$search,  // 传入搜索对象
        id_min: $id_min,
        id_max: FreshRSS_Context::$id_max,
        sort: FreshRSS_Context::$sort,
        order: FreshRSS_Context::$order,
        continuation_id: FreshRSS_Context::$continuation_id,
        continuation_values: $continuation_values,
        limit: $postsPerPage ?? FreshRSS_Context::$number,
        // ...
    );
}
```

#### 4.2.5 视图渲染

[normal.phtml#L11-L50](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/views/index/normal.phtml#L11-L50)

```php
<?php
// 1. 执行回调，获取数据生成器
call_user_func($this->callbackBeforeEntries, $this);

$last_transition = '';
$lastEntry = null;
$nbEntries = 0;

// 2. 遍历生成器，逐条渲染
foreach ($this->entries as $item):
    $item = Minz_ExtensionManager::callHook(Minz_HookType::EntryBeforeDisplay, $item);
    if ($item === null) continue;
    
    $nbEntries++;
    $lastEntry = $item;
    
    // 3. 显示分组过渡（按日期/分类/Feed分组）
    $transition = FreshRSS_index_Controller::transition($item);
    if ($transition !== '' && $transition !== $last_transition):
        $last_transition = $transition;
        ?>
        <div class="transition">
            <?= $transition ?>
            <a class="transition-link" href="<?= FreshRSS_index_Controller::transitionLink($item) ?>">¶</a>
        </div>
    <?php endif; ?>

    <!-- 4. 渲染单条文章 -->
    <article class="flux_item">
        <!-- 文章标题、内容、元数据等 -->
    </article>

<?php endforeach;

// 5. 执行分页回调
if ($nbEntries > 0) {
    call_user_func($this->callbackBeforePagination, $this, $nbEntries, $lastEntry);
}
?>
```

### 4.3 边界职责

**结果呈现层只负责：**
- ✅ HTTP请求参数接收与验证
- ✅ 全局上下文状态管理
- ✅ 调用DAO层获取数据
- ✅ 视图模板渲染
- ✅ 分页导航逻辑

**结果呈现层不负责：**
- ❌ 搜索字符串语法解析
- ❌ SQL查询构建
- ❌ 数据库操作细节
- ❌ 业务规则实现（除了UI相关逻辑）

---

## 5. 协作边界与数据流

### 5.1 完整调用链

```
用户输入搜索: "intitle:php AND author:john OR #programming"
        │
        ▼
HTTP GET /i/?search=intitle:php+AND+author%3ajohn+OR+%23programming
        │
        ▼
[indexController.php] normalAction()
        │
        ├─► [Context.php] updateUsingRequest()
        │      │
        │      └─► new FreshRSS_BooleanSearch("intitle:php AND author:john OR #programming")
        │              │
        │              ├─► escapeLiterals()  # 保护引号和正则中的括号/OR
        │              ├─► parseUserQueryNames()  # 展开保存的查询
        │              ├─► consistentOrParentheses()  # 标准化括号
        │              ├─► parseParentheses()  # 递归解析括号
        │              │      └─► 子BooleanSearch对象，operator='OR'
        │              │              ├─► 子BooleanSearch对象，operator='AND'
        │              │              │      ├─► FreshRSS_Search("intitle:php")
        │              │              │      │      └─► parseIntitleSearch() → intitle=['php']
        │              │              │      └─► FreshRSS_Search("author:john")
        │              │              │             └─► parseAuthorSearch() → author=['john']
        │              │              └─► FreshRSS_Search("#programming")
        │              │                     └─► parseTagsSearch() → tags=['programming']
        │              └─► 返回 BooleanSearch 对象树
        │
        └─► callbackBeforeEntries
                │
                └─► listEntriesByContext()
                        │
                        └─► [EntryDAO.php] listWhere()
                                │
                                ├─► sqlListWhere()
                                │      │
                                │      ├─► 基础WHERE条件（类型、权限）
                                │      └─► sqlListEntriesWhere()
                                │              │
                                │              ├─► 状态过滤（已读/未读）
                                │              ├─► ID范围
                                │              ├─► Keyset分页条件
                                │              └─► sqlBooleanSearch()  ← 核心转换
                                │                      │
                                │                      ├─► 遍历BooleanSearch对象树
                                │                      ├─► 递归处理嵌套结构
                                │                      ├─► 根据operator组合条件
                                │                      ├─► 每个Search对象转为OR条件组
                                │                      ├─► 构建SQL片段和参数数组
                                │                      └─► 返回 [values[], sqlFragment]
                                │
                                └─► listWhereRaw()
                                        │
                                        ├─► 构建最终SQL（先查ID再关联）
                                        ├─► 参数绑定（按类型绑定PDO::PARAM_INT/STR）
                                        ├─► 执行查询
                                        └─► yield FreshRSS_Entry 对象（Generator）
        │
        ▼
[normal.phtml] 视图渲染
        │
        ├─► 遍历Generator
        ├─► 按排序字段分组显示过渡标题
        ├─► 渲染每条文章
        └─► 生成分页导航（基于continuation_id）
```

### 5.2 层间接口契约

#### 5.2.1 查询解析层 → 条件构造层

**接口**：`FreshRSS_BooleanSearch` 对象

**契约**：
- 不可变对象（构造后只读）
- 通过 getter 方法访问所有解析后的条件
- 支持嵌套结构（BooleanSearch 包含 BooleanSearch 或 Search）
- 每个 Search 对象内的多个条件为 OR 关系
- BooleanSearch 的 operator 决定子条件的组合方式

#### 5.2.2 条件构造层 → 结果呈现层

**接口**：`Generator<FreshRSS_Entry>`

**契约**：
- 惰性加载，内存高效
- 按指定排序顺序返回
- 自动处理分页逻辑
- 每个条目是完整的 `FreshRSS_Entry` 域对象
- 异常通过 `FreshRSS_EntriesGetter_Exception` 传递

#### 5.2.3 结果呈现层 → 查询解析层

**接口**：原始搜索字符串

**契约**：
- URL编码的字符串
- 通过 `Minz_Request::paramString('search')` 获取
- 支持所有定义的搜索语法
- 空字符串表示无搜索条件

### 5.3 关键设计决策

#### 5.3.1 为什么使用两层解析（Search + BooleanSearch）？

**设计原因**：
1. **关注点分离**：`Search` 处理原子条件解析，`BooleanSearch` 处理逻辑组合
2. **递归解析**：布尔逻辑天然具有递归结构，需要独立的类来处理
3. **灵活组合**：可以单独使用 `Search`（简单搜索）或嵌套使用 `BooleanSearch`（复杂搜索）
4. **SQL构建优化**：`BooleanSearch` 的 operator 直接映射到 SQL 的 AND/OR

#### 5.3.2 为什么使用 Generator 而不是数组？

**设计原因**：
1. **内存效率**：避免一次性加载所有搜索结果到内存
2. **流式处理**：可以边查询边渲染，提升响应速度
3. **分页友好**：可以精确控制返回条数，+1判断是否有下一页
4. **灵活中止**：视图层可以随时停止遍历

#### 5.3.3 为什么使用延迟关联（先查ID再JOIN）？

**设计原因**：
1. **性能优化**：`ORDER BY ... LIMIT` 在大表上性能差，先查ID可以利用覆盖索引
2. **避免排序大表**：子查询只查ID，排序更快，再关联获取详情
3. **MySQL优化器适配**：`LIMIT` 推到子查询，减少需要处理的行数

#### 5.3.4 为什么使用 Keyset 分页而不是 OFFSET？

**设计原因**：
1. **性能稳定**：OFFSET 越大性能越差，Keyset 分页性能恒定
2. **数据一致性**：避免新数据插入导致的重复/遗漏
3. **适用场景**：Feed阅读器典型的"加载更多"交互模式

---

## 6. FilterAction 过滤动作

### 6.1 核心文件

[FilterAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/FilterAction.php)

### 6.2 作用与定位

`FreshRSS_FilterAction` 是搜索条件与动作的绑定器，用于自动化规则场景：

```php
class FreshRSS_FilterAction {
    private readonly FreshRSS_BooleanSearch $booleanSearch;  // 触发条件
    private ?array $actions;  // 匹配时执行的动作列表

    public function booleanSearch(): FreshRSS_BooleanSearch {
        return $this->booleanSearch;
    }

    /** @return list<string> */
    public function actions(): array {
        return $this->actions ?? [];
    }

    // JSON序列化/反序列化，用于持久化存储
    public function toJSON(): array { /* ... */ }
    public static function fromJSON($json): ?FreshRSS_FilterAction { /* ... */ }
}
```

### 6.3 协作边界

`FilterAction` 不属于三层架构中的任何一层，而是：
- **上游**：使用查询解析层（`BooleanSearch`）定义触发条件
- **下游**：被业务逻辑（如自动标记规则）使用来执行动作
- **本身**：只做条件与动作的绑定，不包含业务逻辑

---

## 7. 设计特点与最佳实践

### 7.1 优点

1. **清晰的分层架构**：每层职责单一，易于理解和维护
2. **面向对象设计**：搜索条件被建模为对象，而非字符串传递
3. **安全的参数绑定**：所有动态值通过参数化查询，无SQL注入风险
4. **数据库抽象**：通过DAO层隐藏数据库差异，支持多种后端
5. **性能优化**：延迟关联、Keyset分页、Generator惰性加载
6. **可扩展性**：新的搜索语法只需添加 `parse*` 方法和对应的SQL构建逻辑

### 7.2 潜在改进点

1. **解析方法顺序依赖**：`Search::__construct` 中 `parse*` 方法的调用顺序很重要，需要文档化
2. **正则解析重复**：多个 `parse*` 方法有相似的正则匹配逻辑，可抽象复用
3. **错误处理**：解析失败时静默忽略，可考虑添加警告机制
4. **SQL构建性能**：`sqlBooleanSearch` 方法较长（~400行），可按条件类型拆分

### 7.3 代码优化建议

**问题**：[EntryDAO.php#L1360](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/EntryDAO.php#L1360) 有笔误 `ANT NOT` 应为 `AND NOT`

```php
// 原代码（错误）
$sub_search .= 'AND NOT ' . static::sqlRegex($alias . 'title', $search_value, $values) .
    ' ANT NOT ' . static::sqlRegex("UNCOMPRESS({$alias}content_bin)", $search_value, $values) . ' ';

// 修正后
$sub_search .= 'AND NOT ' . static::sqlRegex($alias . 'title', $search_value, $values) .
    ' AND NOT ' . static::sqlRegex("UNCOMPRESS({$alias}content_bin)", $search_value, $values) . ' ';
```

---

## 8. 总结

FreshRSS 的搜索过滤系统展现了优秀的软件架构设计：

| 层级 | 输入 | 输出 | 核心能力 | 关键文件 |
|------|------|------|----------|----------|
| 查询解析层 | 原始搜索字符串 | `FreshRSS_BooleanSearch` 对象 | 语法解析、布尔逻辑、正则处理 | [Search.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/Search.php) [BooleanSearch.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/BooleanSearch.php) |
| 条件构造层 | 搜索对象 + 查询参数 | `Generator<FreshRSS_Entry>` | SQL构建、数据库适配、参数安全 | [EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/EntryDAO.php) |
| 结果呈现层 | HTTP请求参数 | 渲染后的HTML页面 | 请求处理、状态管理、视图渲染 | [indexController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Controllers/indexController.php) [Context.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/Context.php) |

**协作边界的核心原则**：
- 每层只通过定义良好的接口通信
- 不跨越层级直接访问内部实现
- 数据单向流动，无循环依赖
- 每层可独立测试和替换

---

## 9. 过滤动作与搜索条件的协作：自动规则体系

前面的三层架构分析侧重于"用户主动搜索→数据库查询→结果展示"这条路径。但 FreshRSS 还存在一条隐式路径：**自动规则（Filter Actions）**——它复用了同一套搜索条件模型，却在完全不同的时机和方式下工作。

### 9.1 自动规则的完整生命周期

自动规则由 `FreshRSS_FilterAction` 承载，将一条搜索条件与一组动作绑定：

```
FreshRSS_FilterAction {
    booleanSearch: FreshRSS_BooleanSearch   // 复用搜索语法描述"哪些条目"
    actions: ['read', 'star', 'label']     // 命中后执行的动作
}
```

**存储位置**：`FilterAction` 被序列化为 JSON，保存在宿主对象的 `attributes.filters` 属性中。

**宿主对象**（均使用 [FilterActionsTrait.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/FilterActionsTrait.php)）：

| 宿主 | 类 | 规则作用域 |
|------|------|-----------|
| 用户全局配置 | [UserConfiguration.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/UserConfiguration.php) | 所有 Feed 的条目 |
| 分类 | [Category.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/Category.php) | 该分类下所有 Feed 的条目 |
| 订阅源 | [Feed.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/Feed.php) | 仅该 Feed 的条目 |
| 标签 | [Tag.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/Tag.php) | 自动打标签规则 |

### 9.2 搜索条件的复用方式——两条路径，同一模型

同一套 `FreshRSS_BooleanSearch` + `FreshRSS_Search` 对象在两条路径中被复用，但匹配引擎截然不同：

```
                        FreshRSS_BooleanSearch
                       ┌──────────┴──────────┐
                       │                       │
               路径A: SQL查询               路径B: 内存匹配
               (用户主动搜索)             (自动规则)
                       │                       │
                       ▼                       ▼
              EntryDAO::sqlBooleanSearch   Entry::matches()
              将搜索对象转为SQL WHERE      将搜索对象逐字段与
              子句，由数据库引擎执行        Entry对象属性对比，
                                           由PHP在内存中执行
```

#### 路径A：SQL查询（用户主动搜索）

[EntryDAO.php#L950-L1381](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/EntryDAO.php#L950-L1381)

- **触发时机**：用户输入搜索词，请求文章列表
- **执行者**：数据库引擎
- **匹配方式**：搜索对象 → SQL WHERE 子句 → 数据库全表/索引扫描
- **输出**：匹配的条目集合（Generator）
- **特点**：批量高效，无需加载所有条目到内存

#### 路径B：内存匹配（自动规则）

[Entry.php#L645-L888](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/Entry.php#L645-L888)

- **触发时机**：新条目入库时逐条匹配
- **执行者**：PHP 运行时
- **匹配方式**：逐字段对比 Entry 对象属性与搜索条件
- **输出**：布尔值（是否匹配）
- **特点**：单条精确，无需数据库查询

**内存匹配的关键实现** [Entry.php#L645-L888](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/Entry.php#L645-L888)：

```php
public function matches(FreshRSS_BooleanSearch $booleanSearch): bool {
    $ok = true;
    foreach ($booleanSearch->searches() as $filter) {
        if ($filter instanceof FreshRSS_BooleanSearch) {
            // 递归处理嵌套布尔逻辑
            match ($filter->operator()) {
                'AND'     => $ok &= $this->matches($filter),
                'OR'      => $ok |= $this->matches($filter),
                'AND NOT' => $ok &= !$this->matches($filter),
                'OR NOT'  => $ok |= !$this->matches($filter),
            };
        } elseif ($filter instanceof FreshRSS_Search) {
            // 原子条件：逐字段对比
            $ok = true;
            if ($filter->getEntryIds() !== null) {
                $ok &= in_array($this->id, $filter->getEntryIds(), true);
            }
            if ($ok && $filter->getIntitle() !== null) {
                foreach ($filter->getIntitle() as $title) {
                    $ok &= $databaseDao::strilike($this->title, $title, contains: true);
                }
            }
            if ($ok && $filter->getIntext() !== null) {
                foreach ($filter->getIntext() as $content) {
                    $ok &= $databaseDao::strilike($this->content, $content, contains: true);
                }
            }
            // ... 所有搜索条件的字段逐一对比
            if ($ok) return true;  // OR语义：任一原子条件匹配即返回true
        }
    }
    return (bool)$ok;
}
```

**对比**：SQL 路径中 `intitle:php` 变成 `title LIKE '%php%'`，内存路径中同一条件变成 `$databaseDao::strilike($this->title, 'php', contains: true)`。语义一致，执行引擎不同。

### 9.3 自动规则的触发时机与流程

自动规则在两个关键时机触发：

#### 9.3.1 条目入库时——标记已读/收藏

[feedController.php#L703](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Controllers/feedController.php#L703)

当 Feed 刷新获取到新条目（或更新已有条目）时：

```
Feed刷新 → 解析新条目 → Extension Hook (EntryBeforeInsert)
                          │
                          ▼
               Entry::applyFilterActions()
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
   UserConfiguration   Category     Feed
   .applyFilterActions  .applyFilter  .applyFilter
   (全局规则)           (分类规则)     (Feed规则)
            │             │             │
            └──────┬──────┘             │
                   ▼                    ▼
         FilterActionsTrait::applyFilterActions()
                   │
                   ▼
         遍历所有 FilterAction
                   │
                   ▼
         Entry::matches(filterAction.booleanSearch())
                   │
            ┌──────┴──────┐
            ▼ 匹配         ▼ 不匹配
     执行 actions:     跳过
     • 'read'  → entry._isRead(true)
     • 'star'  → entry._isFavorite(true)
     • 'label' → 设置 applyLabel=true（延迟处理）
```

[FilterActionsTrait.php#L125-L153](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/FilterActionsTrait.php#L125-L153)

```php
public function applyFilterActions(FreshRSS_Entry $entry, ?bool &$applyLabel = null): void {
    $applyLabel = false;
    foreach ($this->filterActions() as $filterAction) {
        if ($entry->matches($filterAction->booleanSearch())) {
            foreach ($filterAction->actions() as $action) {
                switch ($action) {
                    case 'read':
                        if (!$entry->isRead()) {
                            $entry->_isRead(true);
                        }
                        break;
                    case 'star':
                        if (!$entry->isUpdated()) {
                            $entry->_isFavorite(true);
                        }
                        break;
                    case 'label':
                        if (!$entry->isUpdated()) {
                            $applyLabel = true;
                        }
                        break;
                }
            }
        }
    }
}
```

**优先级**：全局规则 → 分类规则 → Feed规则，三者依次执行，后者的结果可能覆盖前者。

**安全约束**：`isUpdated()` 检查确保已更新的文章不会被自动标记为收藏或打标签，防止覆盖用户手动操作。

#### 9.3.2 新条目提交后——自动打标签

[feedController.php#L879-L901](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Controllers/feedController.php#L879-L901)

标签动作的处理与其他动作不同——它不能在入库时立即执行（因为标签关联需要条目已持久化），所以延迟到 `commitNewEntries` 之后：

```
actualizeFeedsAndCommit()
    │
    ├─► actualizeFeeds()       ← 获取新条目，触发 applyFilterActions（read/star）
    │
    └─► commitNewEntries()     ← 将临时表数据提交到主表
              │
              └─► applyLabelActions()   ← 批量处理标签动作
                      │
                      ├─► 查询所有配置了 label 动作的 Tag
                      ├─► 遍历最近入库的条目
                      ├─► 对每个 Tag 调用 Tag.applyFilterActions(entry)
                      └─► 批量执行 tagDAO.tagEntries($applyLabels)
```

```php
private static function applyLabelActions(int $nbNewEntries): int|false {
    $tagDAO = FreshRSS_Factory::createTagDao();
    $labels = FreshRSS_Context::labels();
    $labels = array_filter($labels, static fn(FreshRSS_Tag $label) =>
        !empty($label->filtersAction('label')));
    if (count($labels) <= 0) return 0;

    $entryDAO = FreshRSS_Factory::createEntryDao();
    $applyLabels = [];
    foreach (FreshRSS_Entry::fromTraversable($entryDAO->selectAll(order: 'DESC', limit: $nbNewEntries)) as $entry) {
        foreach ($labels as $label) {
            $label->applyFilterActions($entry, $applyLabel);
            if ($applyLabel) {
                $applyLabels[] = ['id_tag' => $label->id(), 'id_entry' => $entry->id()];
            }
        }
    }
    return $tagDAO->tagEntries($applyLabels);
}
```

### 9.4 过滤动作与搜索结果的协作边界

```
┌──────────────────────────────────────────────────────────────────┐
│                    搜索条件模型 (共享)                           │
│         FreshRSS_BooleanSearch + FreshRSS_Search                │
│                                                                  │
│  定义"匹配什么"——与执行路径无关的纯条件描述                      │
└───────────────┬──────────────────────────┬───────────────────────┘
                │                          │
        ┌───────┴───────┐          ┌───────┴───────┐
        ▼               ▼          ▼               ▼
   用户搜索路径    自动规则路径   FilterAction    Entry::matches()
   (批量查询)     (逐条匹配)    (条件+动作绑定)  (内存匹配引擎)
        │               │          │               │
        ▼               ▼          ▼               ▼
   sqlBooleanSearch  applyFilter  toJSON/fromJSON  strilike/preg_match
   (SQL WHERE)       Actions()   (持久化)          (字段级对比)
```

**协作边界总结**：

| 维度 | 用户搜索路径 | 自动规则路径 |
|------|-------------|-------------|
| 搜索条件来源 | HTTP请求 `search` 参数 | 宿主对象 `attributes.filters` |
| 匹配引擎 | 数据库 SQL 引擎 | PHP 内存 `Entry::matches()` |
| 匹配时机 | 用户请求时 | 条目入库时 |
| 匹配粒度 | 批量（成千上万条） | 逐条 |
| 输出 | 匹配条目的 Generator | 布尔值 → 触发动作 |
| 可执行动作 | 无（纯筛选） | read / star / label |
| 搜索条件接口 | `FreshRSS_BooleanSearch` 对象 | 同左（完全复用） |
| 条件描述语言 | 同一套搜索语法 | 同一套搜索语法 |

**关键设计**：搜索条件模型是纯值对象，不包含任何执行语义。它是两条路径之间的"共享契约"——无论 SQL 引擎还是 PHP 引擎，对同一搜索条件的匹配结果必须一致。

---

## 10. 搜索请求→上下文→取数→列表渲染：概念级说明

### 10.1 端到端概念模型

一次搜索请求从用户输入到页面呈现，经历四个概念阶段。每个阶段完成一项明确的转换，产出一个供下一阶段消费的结构：

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  ① 请求解析  │────►│  ② 条件装配  │────►│  ③ 数据获取  │────►│  ④ 列表渲染  │
│             │     │             │     │             │     │             │
│ HTTP参数 →  │     │ 搜索对象 +  │     │ SQL →       │     │ Generator → │
│ 上下文状态  │     │ 查询上下文  │     │ Generator   │     │ HTML片段    │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
   Controller          Context             DAO                View
   负责                负责                负责                负责
```

### 10.2 阶段①：请求解析——从HTTP到上下文

**职责**：将散列的HTTP参数收拢为类型安全的全局状态

**入口**：[Context.php#L239-L317](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Models/Context.php#L239-L317) `updateUsingRequest()`

**转换过程**：

```
HTTP Query String
    ?search=intitle:php
    &get=f_12
    &state=2
    &sort=date
    &order=DESC
    &nb=20
    &cid=9876543210
        │
        ▼  逐一解析，赋值到静态属性
FreshRSS_Context {
    $search:    FreshRSS_BooleanSearch    ← 搜索语法解析在此触发
    $get:       ['f', 12]                ← 浏览范围：Feed #12
    $state:     STATE_NOT_READ           ← 文章状态过滤
    $sort:      'date'                   ← 排序字段
    $order:     'DESC'                   ← 排序方向
    $number:    20                       ← 每页条数
    $id_max:    '0'                      ← 安全上限（防无限回溯）
    $continuation_id: '9876543210'       ← 游标分页起点
    $sinceHours: 0                       ← 时间窗口限制
}
```

**概念要点**：
- **上下文是全局单例**：`FreshRSS_Context` 使用静态属性，整个请求生命周期共享
- **搜索解析是急切的**：构造 `BooleanSearch` 时就完成全部语法解析，不延迟
- **状态过滤与搜索条件正交**：`state`（已读/未读/收藏）与 `search`（关键词/字段过滤）是独立维度

### 10.3 阶段②：条件装配——搜索对象与查询参数汇合

**职责**：将上下文状态组装为DAO层可消费的参数集合

**入口**：[indexController.php#L349-L413](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/Controllers/indexController.php#L349-L413) `listEntriesByContext()`

**转换过程**：

```
FreshRSS_Context 静态属性
        │
        ├─ $get → ['f', 12]     → $type='f', $id=12
        │
        ├─ $sinceHours           → $id_min = (time()-hours*3600).'000000'
        │
        ├─ $continuation_id      → 需查DAO获取游标条目的排序字段值
        │   │
        │   └─► entryDAO.searchById('9876543210')
        │       → $pagingEntry → continuation_values = [1700000000]
        │
        └─ $search, $state, $sort, $order, $number...
                │
                ▼  原样传递给DAO
        entryDAO.listWhere(
            type:   'f',
            id:     12,
            state:  STATE_NOT_READ,
            filters: BooleanSearch{intitle:['php']},
            id_min: '1700000000000000',
            id_max: '0',
            sort:   'date',
            order:  'DESC',
            continuation_id: '9876543210',
            continuation_values: [1700000000],
            limit:  21          ← number+1，多取一条判断是否有下一页
        )
```

**概念要点**：
- **游标分页的额外查询**：使用非ID排序时，需要先查DAO获取上一页末条记录的排序字段值
- **+1策略**：请求 N 条数据但查询 N+1 条，多出的一条仅用于判断是否有后续页
- **搜索对象透传**：`BooleanSearch` 作为整体参数传入DAO，Controller不拆解其内部结构

### 10.4 阶段③：数据获取——搜索对象转为SQL，查询并流式返回

**职责**：将搜索对象和查询参数转换为安全的SQL，执行查询，以Generator返回

**核心调用链**：

```
listWhere(type, id, state, filters, ...)
    │
    └─► sqlListWhere(type, id, state, filters, ...)
            │
            ├─► 构建基础WHERE（type → 优先级/分类/Feed/标签过滤）
            │
            └─► sqlListEntriesWhere(state, filters, id_min, id_max, sort, ...)
                    │
                    ├─► 构建状态过滤（is_read / is_favorite）
                    ├─► 构建ID范围（id_min / id_max）
                    ├─► 构建Keyset分页条件（continuation_id + sort values）
                    │
                    └─► sqlBooleanSearch(alias, filters)
                            │
                            ├─► 递归遍历 BooleanSearch 树
                            │   ├─► BooleanSearch → AND/OR/AND NOT/OR NOT 组合
                            │   └─► Search → 逐字段转SQL片段
                            │       ├─ entry_ids    → id IN (?,?,?)
                            │       ├─ feed_ids     → id_feed IN (?,?,?)
                            │       ├─ category_ids → 子查询 IN (SELECT ...)
                            │       ├─ label_ids    → EXISTS / 子查询
                            │       ├─ intitle      → title LIKE ?
                            │       ├─ intitle_regex→ REGEXP_LIKE(?,?) / REGEXP ?
                            │       ├─ intext       → content LIKE ? / UNCOMPRESS + LIKE
                            │       ├─ author       → author LIKE ?
                            │       ├─ inurl        → link LIKE ?
                            │       ├─ tags         → CONCAT + LIKE
                            │       ├─ date         → id >= ? / id <= ?
                            │       ├─ search       → title LIKE ? OR content LIKE ?
                            │       └─ (not_* 版本→ NOT LIKE / NOT IN / NOT EXISTS)
                            │
                            └─► 返回 [values[], sqlFragment]
```

**最终SQL结构**（延迟关联模式）：

```sql
-- 外层：获取完整字段
SELECT e0.id, e0.title, e0.author, UNCOMPRESS(e0.content_bin) AS content, ...
FROM `_entry` e0
INNER JOIN (
    -- 内层：仅查ID，利用索引高效排序和分页
    SELECT e.id
    FROM `_entry` e
    INNER JOIN `_feed` f ON f.id = e.id_feed
    WHERE f.priority >= ?                    -- 优先级过滤
      AND e.id_feed = ?                     -- Feed过滤
      AND (e.is_read = 0)                   -- 状态过滤
      AND e.id <= ?                         -- 安全上限
      AND (e.date < ? OR (e.date = ? AND e.id <= ?))  -- Keyset分页
      AND (e.title LIKE ?)                  -- 搜索条件
    ORDER BY e.date DESC, e.id DESC
    LIMIT 21
) e2 ON e2.id = e0.id
ORDER BY e0.date DESC, e0.id DESC
```

**概念要点**：
- **延迟关联**：先在子查询中仅查ID（利用覆盖索引快速排序+分页），再关联外层获取完整数据
- **参数化安全**：所有动态值通过 `?` 占位符和 `$values[]` 数组绑定，杜绝SQL注入
- **数据库方言适配**：`sqlRegex()` 根据MariaDB/MySQL/SQLite/PostgreSQL选择不同正则实现
- **搜索条件以AND嵌入**：`sqlBooleanSearch` 的结果通过 `AND (搜索片段)` 整体拼入主WHERE

### 10.5 阶段④：列表渲染——Generator驱动流式输出

**职责**：消费Generator，逐条渲染HTML，管理分页状态

**入口**：[normal.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/24-FreshRSS/app/views/index/normal.phtml)

**转换过程**：

```
callbackBeforeEntries($view)
    │
    └─► listEntriesByContext(number+1)
            │
            └─► entryDAO.listWhere(...) → Generator<FreshRSS_Entry>
                    │
                    ▼ 赋值到 $view->entries
        ob_start()  ← 开启输出缓冲（逐条刷新）

foreach ($this->entries as $item):
    │
    ├─► Extension Hook: EntryBeforeDisplay    ← 扩展可修改/过滤条目
    │
    ├─► transition($item)                     ← 生成分组过渡标题
    │   │                                       （"今天 — 2024-01-15"）
    │   └─► transitionLink($item)              ← 过渡标题的锚点链接
    │
    ├─► 渲染 <article> HTML                    ← 单条文章的完整呈现
    │
    ├─► ob_flush()                             ← 逐条刷新到浏览器
    │
    └─► $nbEntries++; $lastEntry = $item;

callbackBeforePagination($view, $nbEntries, $lastEntry)
    │
    ├─► if ($nbEntries > $number):
    │       $continuation_id = $lastEntry->id()   ← 设置下一页游标
    │       ob_clean()                             ← 丢弃多取的那一条
    │   else:
    │       $continuation_id = '0'                 ← 无更多数据
    │
    └─► ob_end_flush()                            ← 关闭缓冲，输出全部内容

渲染分页导航栏 ← 基于 $continuation_id 生成"加载更多"链接
```

**概念要点**：
- **Generator驱动渲染**：视图模板不持有全部数据，而是在 `foreach` 中逐条从Generator拉取
- **输出缓冲策略**：`ob_start()` / `ob_clean()` / `ob_end_flush()` 配合 +1 策略，多取的条目被丢弃
- **过渡分组**：根据排序字段（日期/分类/Feed名称）自动生成分组标题
- **Keyset分页闭环**：`continuation_id` 取自最后一条的ID，嵌入下一页请求的 `cid` 参数

### 10.6 四阶段概念总结

```
     ① 请求解析           ② 条件装配           ③ 数据获取           ④ 列表渲染
     ─────────           ─────────           ─────────           ─────────
入   HTTP参数            Context状态          DAO参数集合          Generator
出   Context状态         DAO参数集合          Generator           HTML页面
核   参数类型化           参数聚合补充         SQL构建+执行        流式渲染+分页
心                                        参数绑定安全
动   search→解析          游标值补充          搜索→WHERE           过渡标题
作   get→类型+ID          +1策略              延迟关联             缓冲控制
     state→位掩码                             方言适配             分页导航
```

**端到端数据形态变迁**：

```
字符串 "intitle:php"          ← HTTP参数
    │
    ▼ 解析
BooleanSearch 对象树          ← 请求解析阶段产物
    │
    ▼ SQL转换
"AND (title LIKE '%php%')"    ← 数据获取阶段产物
    │
    ▼ 查询执行
PDOStatement → Generator     ← 数据获取阶段产物
    │
    ▼ 模板渲染
<article>…</article>          ← 列表渲染阶段产物
```
