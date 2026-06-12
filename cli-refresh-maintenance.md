# FreshRSS CLI 刷新与维护命令深度分析

## 目录

1. [CLI 命令概览](#cli-命令概览)
2. [命令解析机制](#命令解析机制)
3. [用户上下文管理](#用户上下文管理)
4. [日志记录机制](#日志记录机制)
5. [并发风险与锁机制](#并发风险与锁机制)
6. [核心刷新命令详解](#核心刷新命令详解)
7. [数据库维护命令详解](#数据库维护命令详解)

---

## CLI 命令概览

FreshRSS 的 CLI 命令位于 [`cli/`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli) 目录，所有命令均以 PHP 脚本形式存在，可通过命令行直接执行。

### 刷新类命令

| 命令 | 文件 | 功能 |
|------|------|------|
| 实际化脚本 | [actualize_script.php](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/app/actualize_script.php) | 全量用户订阅刷新（cron 主脚本） |
| 用户刷新 | [actualize-user.php](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/actualize-user.php) | 单个用户订阅刷新 |

### 维护类命令

| 命令 | 文件 | 功能 |
|------|------|------|
| 清理旧条目 | [purge.php](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/purge.php) | 清理过期文章条目 |
| 数据库优化 | [db-optimize.php](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/db-optimize.php) | 优化数据库表结构 |
| 数据库备份 | [db-backup.php](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/db-backup.php) | 备份为 SQLite |
| 数据库恢复 | [db-restore.php](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/db-restore.php) | 从 SQLite 恢复 |
| 环境准备 | [prepare.php](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/prepare.php) | 初始化数据目录结构 |
| 健康检查 | [health.php](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/health.php) | API 健康检查 |

---

## 命令解析机制

### 入口点与基础架构

所有 CLI 命令都以 [`_cli.php`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/_cli.php) 作为公共入口文件，提供统一的初始化和工具函数。

**执行流程：**

```
CLI 脚本 → require _cli.php → 初始化系统 → 解析参数 → 执行业务逻辑 → done() 退出
```

### 命令行选项解析器

#### CliOption 类

[`CliOption.php`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/CliOption.php) 定义单个命令行选项的配置：

- **值类型**：`VALUE_NONE`（标志位）、`VALUE_REQUIRED`（必须传值）、`VALUE_OPTIONAL`（可选值）
- **数据类型**：`string`、`int`、`bool`、数组类型
- **别名系统**：支持长别名（`--user`）、短别名（`-u`）、废弃别名（带 deprecation 警告）

**构造器流式 API 示例：**
```php
(new CliOption('user', 'u'))
    ->withValueRequired()      // 必须传值
    ->typeOfString()           // 字符串类型
    ->deprecatedAs('username') // 旧别名，使用时警告
```

#### CliOptionsParser 类

[`CliOptionsParser.php`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/CliOptionsParser.php) 是抽象基类，采用匿名类扩展模式：

**解析流程：**

1. `parseInput()` - 调用 PHP 原生 `getopt()` 解析参数
2. `getoptOutputTransformer()` - 将 getopt 输出转换为内部格式
3. `checkForDeprecatedAliasUse()` - 检测废弃别名并输出警告
4. `appendUnknownAliases()` - 收集未知选项错误
5. `appendInvalidValues()` - 验证值的类型合法性
6. `appendTypedValidValues()` - 类型转换并设置动态属性

**使用模式：**
```php
$cliOptions = new class extends CliOptionsParser {
    public string $user;    // 动态属性，由解析器自动赋值

    public function __construct() {
        $this->addRequiredOption('user', (new CliOption('user')));
        parent::__construct();
    }
};

if (!empty($cliOptions->errors)) {
    fail('Error: ' . array_shift($cliOptions->errors) . "\n" . $cliOptions->usage);
}
```

**错误处理：**
- 未知选项 → 收集到 `$errors` 数组
- 类型不匹配（非整数、非法布尔值）→ 收集到 `$errors`
- 必填选项缺失 → 收集到 `$errors`
- 废弃选项 → 输出 STDERR 警告但不报错

---

## 用户上下文管理

### 初始化流程

CLI 环境下的用户上下文通过 [`cliInitUser()`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/_cli.php#L28-L46) 函数建立：

```php
function cliInitUser(string $username): string {
    // 1. 校验用户名格式
    FreshRSS_user_Controller::checkUsername($username);
    // 2. 校验用户存在性
    FreshRSS_user_Controller::userExists($username);
    // 3. 初始化用户上下文
    FreshRSS_Context::initUser($username);
    // 4. 加载用户启用的扩展
    $ext_list = FreshRSS_Context::userConf()->extensions_enabled;
    Minz_ExtensionManager::enableByList($ext_list, 'user');
}
```

### Context 初始化详解

[`FreshRSS_Context::initUser()`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/app/Models/Context.php#L98-L140) 的核心逻辑：

1. **会话初始化**：`Minz_Session::init('FreshRSS')` - 即使在 CLI 模式也维持 Session 抽象
2. **会话锁定**：读写 Session 前加锁，防止并发
3. **配置加载**：从 `data/users/{username}/config.php` 加载用户配置，以 `config-user.default.php` 为默认值
4. **用户切换**：`Minz_User::change($username)` - 设置当前用户标识
5. **搜索初始化**：创建 `FreshRSS_BooleanSearch` 实例
6. **旧配置迁移**：处理 `old_entries` 等历史配置项的兼容性

### 系统级初始化

在 `_cli.php` 顶部执行的全局初始化：

```php
Minz_Session::init('FreshRSS', true);      // 初始化 Session
FreshRSS_Context::initSystem();             // 加载系统配置
Minz_ExtensionManager::init();              // 初始化扩展管理器
Minz_Translate::init(Minz_Translate::DEFAULT_LANGUAGE); // 初始化翻译
FreshRSS_Context::$isCli = true;            // 标记 CLI 模式
```

### 多用户遍历模式

在 [`actualize_script.php`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/app/actualize_script.php#L67-L116) 等全用户脚本中：

1. 获取所有用户列表：`FreshRSS_user_Controller::listUsers()`
2. 随机打乱顺序：`shuffle($users)` - 分散负载
3. 默认用户优先：`array_unshift($users, $default_user)`
4. 跳过检查：
   - 无效用户配置
   - 被禁用的用户（`userConf()->enabled === false`）
   - 非活跃用户（超过 `max_inactivity` 时间无活动）
5. 循环内重新初始化上下文：每个用户独立调用 `$app->init()`

---

## 日志记录机制

### Minz_Log 核心类

[`Minz_Log`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/lib/Minz/Log.php) 提供统一的日志记录接口。

### 日志级别

| 级别 | 方法 | 生产环境是否记录 |
|------|------|-----------------|
| ERROR | `Minz_Log::error()` | ✅ 是 |
| WARNING | `Minz_Log::warning()` | ✅ 是 |
| NOTICE | `Minz_Log::notice()` | ❌ 否 |
| DEBUG | `Minz_Log::debug()` | ❌ 否 |

### 日志文件位置

- **用户日志**：`data/users/{username}/log.txt` - 用户相关操作
- **管理日志**：`data/users/_/log.txt` - 系统级事件（通过 `ADMIN_LOG` 常量）
- **PSHB 日志**：`data/PubSubHubbub/log.txt` - WebSub 推送日志

### 日志格式

```
[日期时间] [级别] --- 日志消息
```

示例：
```
[Wed, 12 Jun 2026 10:30:00 +0000] [warning] --- Feed http://example.com/feed.xml returned HTTP 404
```

### 日志轮转机制

[`ensureMaxSize()`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/lib/Minz/Log.php#L73-L99) 实现自动轮转：

- 默认最大 1MB（`MAX_LOG_SIZE` 常量可配置）
- 超过限制时保留后半部分内容（截断前半部分）
- 使用 `flock(LOCK_EX)` 进行文件级并发保护
- 轮转后插入一条 `Log rotate.` 记录

### 日志写入并发控制

日志写入使用 `FILE_APPEND | LOCK_EX` 标志：
```php
file_put_contents($file_name, $log, FILE_APPEND | LOCK_EX)
```

`LOCK_EX` 确保写入时获取独占锁，防止多进程并发写入导致的日志内容交错。

### Syslog 集成

通过 `COPY_LOG_TO_SYSLOG` 常量控制是否同时输出到系统 syslog。

### CLI 特定的日志输出

在 [`actualize_script.php`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/app/actualize_script.php#L25-L37) 中定义了 `notice()` 辅助函数：

```php
function notice(string $message): void {
    Minz_Log::notice($message, ADMIN_LOG);  // 写入管理日志
    if (!COPY_LOG_TO_SYSLOG && SIMPLEPIE_SYSLOG_ENABLED) {
        syslog(LOG_NOTICE, $message);        // 可选输出到 syslog
    }
    if (defined('STDOUT') && !COPY_SYSLOG_TO_STDERR) {
        fwrite(STDOUT, $message . "\n");     // 输出到标准输出
    }
}
```

---

## 并发风险与锁机制

### 全局刷新互斥锁

在 [`actualize_script.php`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/app/actualize_script.php#L39-L56) 中实现了全局级别的互斥锁：

**锁文件位置**：`data/tmp/actualize.freshrss.lock`

**锁机制：**
```php
$mutexFile = TMP_PATH . '/actualize.freshrss.lock';
$mutexTtl = 900; // 15 分钟 TTL

// 1. 过期锁自动清理
if (file_exists($mutexFile) && (time() - filemtime($mutexFile) > $mutexTtl)) {
    unlink($mutexFile);
}

// 2. 尝试获取锁（使用 fopen 'x' 模式，文件已存在则失败）
if (($handle = @fopen($mutexFile, 'x')) === false) {
    notice('Actualization already running, aborting...');
    die();
}
fclose($handle);

// 3. 注册 shutdown 函数确保退出时释放锁
register_shutdown_function(static function () use ($mutexFile) {
    unlink($mutexFile);
});
```

**锁续期机制：**
在每个 feed 刷新前通过 hook 更新锁文件 mtime：
```php
Minz_ExtensionManager::addHook(Minz_HookType::FeedBeforeActualize, 
    static function (FreshRSS_Feed $feed) use ($mutexFile) {
        touch($mutexFile);  // 续期
        return $feed;
    });
```

### 单 Feed 级别的锁

[`FreshRSS_Feed::lock()`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/app/Models/Feed.php#L1339-L1350) 实现单个订阅的刷新锁：

**锁文件位置**：`data/tmp/{feed_hash}.freshrss.lock`

**锁机制：**
- 同样使用 `fopen('x')` 排他创建模式
- TTL 为 3600 秒（1 小时）
- 刷新失败或完成后调用 `$feed->unlock()` 释放

**代码位置**：在 `FreshRSS_feed_Controller::actualizeFeeds()` 的 feed 循环中使用。

### 数据库事务锁

#### 条目提交事务

在 [`actualizeFeedsAndCommit()`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/app/Controllers/feedController.php#L918-L942) 中：

```php
if ($nbNewArticles > 0) {
    $entryDAO->beginTransaction();
    FreshRSS_feed_Controller::commitNewEntries();
}
// ... 缓存更新 ...
if ($entryDAO->inTransaction()) {
    $entryDAO->commit();
}
```

#### 清理操作事务

在 [`purge.php`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/purge.php#L32-L37) 中：

```php
$feedDAO->beginTransaction();
foreach ($feeds as $feed) {
    $nb_total += ($feed->cleanOldEntries() ?: 0);
}
$feedDAO->updateCachedValues();
$feedDAO->commit();
```

### 会话锁

[`Minz_Session::lock()`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/lib/Minz/Session.php#L19-L22) 使用 PHP 原生 session 锁机制，防止会话数据并发读写冲突。

### 配置文件锁

[`Minz_ModelArray::loadArray()`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/lib/Minz/ModelArray.php#L27-L48) 对配置文件使用自定义文件锁。

### 并发风险分析

| 风险场景 | 保护机制 | 风险等级 |
|---------|---------|---------|
| 重复执行 actualize_script | 全局互斥锁 + TTL 续期 | 低 |
| 同一 Feed 并发刷新 | Feed 级文件锁 | 低 |
| 并发写入日志文件 | `LOCK_EX` 文件锁 | 低 |
| 数据库并发写入 | 事务 + 行锁 | 中 |
| 缓存值并发更新 | 数据库事务保护 | 中 |
| 用户配置并发修改 | 会话锁 + 文件锁 | 中 |
| HTTP 缓存失效并发 | 无特殊保护 | 低 |

---

## 核心刷新命令详解

### actualize-user.php - 单用户刷新

[`cli/actualize-user.php`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/actualize-user.php)

**参数：**
- `--user` (必填)：用户名

**执行流程：**

1. 检查系统需求
2. 初始化用户上下文
3. 触发用户维护钩子：`Minz_ExtensionManager::callHookVoid(FreshrssUserMaintenance)`
4. 执行轻量数据库维护：`$databaseDAO->minorDbMaintenance()`
5. 提交暂存新条目：`FreshRSS_feed_Controller::commitNewEntries()`
6. 更新 Feed 缓存值：`$feedDAO->updateCachedValues()`
7. 刷新动态 OPML：`FreshRSS_category_Controller::refreshDynamicOpmls()`
8. 执行实际 Feed 刷新：`FreshRSS_feed_Controller::actualizeFeedsAndCommit()`
9. 失效 HTTP 缓存：`invalidateHttpCache($username)`
10. 输出结果并退出

**返回值：**
- 退出码 0：有 feed 被更新
- 退出码 1：无 feed 更新或出错

### actualize_script.php - 全量刷新脚本

[`app/actualize_script.php`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/app/actualize_script.php)

**特点：**
- 这是 cron 任务调用的主要脚本
- 遍历所有用户依次刷新
- 内置全局互斥锁防止重复执行
- 跳过非活跃用户（超过 `max_inactivity` 秒）

**执行流程：**

1. 初始化 CLI 环境，设置 `auth_type = 'none'` 绕过认证
2. 设置请求参数模拟 Web 调用（`c=feed`, `a=actualize`, `maxFeeds=PHP_INT_MAX`）
3. 获取全局刷新锁
4. 记录开始时间，遍历用户列表
5. 每个用户：
   - 初始化用户上下文
   - 检查用户是否启用、是否活跃
   - 调用 `$app->init()` 完整初始化
   - 注册 Feed 刷新 hook 用于锁续期
   - 调用 `$app->run()` 执行刷新
   - 失效 HTTP 缓存
   - 触发垃圾回收：`gc_collect_cycles()`
6. 输出总览统计（用户数、内存峰值、耗时）

### actualizeFeeds() 核心刷新逻辑

[`FreshRSS_feed_Controller::actualizeFeeds()`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/app/Controllers/feedController.php#L420-L900)

**刷新策略：**
- 按 `lastUpdate` 升序排列，优先刷新最久未更新的 feed
- 支持按优先级过滤（`PRIORITY_MAIN_STREAM`）
- 每个 feed 刷新前检查 TTL，未过期则跳过

**单 Feed 刷新步骤：**

1. 获取 feed 锁：`$feed->lock()`
2. 检查 TTL 是否过期
3. 加载 feed 内容（使用 SimplePie 或 XPath 抓取）
4. 计算文章 GUID，识别新文章
5. 过滤和处理：
   - 标题重复标记为已读
   - 应用过滤器
   - 处理更新的文章（hash 变化）
6. 新条目先写入 `entrytmp` 临时表
7. 更新 `lastSeen` 时间戳
8. 随机触发旧条目清理（1/30 概率）
9. 更新 feed 的 `lastUpdate` 时间
10. 释放 feed 锁

### actualizeFeedsAndCommit() 提交逻辑

[`FreshRSS_feed_Controller::actualizeFeedsAndCommit()`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/app/Controllers/feedController.php#L918-L942)

1. 调用 `actualizeFeeds()` 获取临时表中的新条目
2. 开启事务
3. 提交新条目到主表：`commitNewEntries()`
4. 执行标签动作
5. 更新 feed 缓存计数
6. 随机触发缓存清理（1/30 概率）
7. 提交事务

---

## 数据库维护命令详解

### db-optimize.php - 数据库优化

[`cli/db-optimize.php`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/db-optimize.php)

**参数：**
- `--user` (必填)：用户名

**数据库差异：**
- **MySQL**：对所有表执行 `OPTIMIZE TABLE` - 重组表和索引
- **PostgreSQL**：对所有表执行 `VACUUM` - 回收死元组空间
- **SQLite**：执行 `VACUUM` - 重建整个数据库文件

**涉及表：** `category`, `feed`, `entry`, `entrytmp`, `tag`, `entrytag`

### purge.php - 清理旧条目

[`cli/purge.php`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/purge.php)

**参数：**
- `--user` (必填)：用户名

**清理策略：**
每个 feed 的 `cleanOldEntries()` 根据以下规则清理：
- `keep_period`：保留最近 N 天的条目
- `keep_max`：最多保留 N 条条目
- 两者同时设置时，满足任一条件即删除（OR 逻辑）

**执行流程：**
1. 轻量数据库维护
2. 列出所有 feed
3. 开启事务
4. 遍历每个 feed 调用 `cleanOldEntries()`
5. 更新缓存值
6. 提交事务
7. 失效 HTTP 缓存

### minorDbMaintenance() - 轻量维护

[`FreshRSS_DatabaseDAO::minorDbMaintenance()`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/app/Models/DatabaseDAO.php#L321-L349)

**用途：** 每次刷新前自动执行的快速维护

**执行内容：**
1. 重置默认分类名称
2. 执行 `SQL_UPDATE_MINOR` 中的增量 SQL 更新
3. 处理 MySQL 和 MariaDB 的语法差异（`DROP INDEX IF EXISTS`）

**特点：**
- 幂等安全，可重复执行
- 失败时仅记录日志不中断流程
- 只处理 minor 版本的 schema 更新

### db-backup.php - 数据库备份

[`cli/db-backup.php`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/db-backup.php)

**参数：**
- `--quiet` / `-q` (可选)：静默模式，减少输出

**功能：**
- 遍历所有用户
- 将数据库导出为 SQLite 格式
- 备份文件位置：`data/users/{username}/backup.sqlite`
- 支持 MySQL → SQLite、PostgreSQL → SQLite 的跨数据库导出

### prepare.php - 环境准备

[`cli/prepare.php`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/prepare.php)

**功能：**
创建必要的数据目录结构和保护文件：
- `cache/` - 缓存目录
- `extensions-data/` - 扩展数据
- `favicons/` - 网站图标缓存
- `fever/` - Fever API 数据
- `PubSubHubbub/` - WebSub 相关
- `Retry-After/` - 重试延迟记录
- `tokens/` - 令牌存储
- `users/` - 用户数据
- `users/_/` - 系统用户目录

每个目录自动创建 `index.html` 防止目录遍历，并创建 `.htaccess` 禁止访问。

---

## 附录：HTTP 缓存失效

### invalidateHttpCache() 函数

[`lib/lib_rss.php`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/lib/lib_rss.php#L329-L342)

```php
function invalidateHttpCache(string $username = ''): bool {
    if (!FreshRSS_user_Controller::checkUsername($username)) {
        Minz_Session::_param('touch', uTimeString());
        $username = Minz_User::name() ?? Minz_User::INTERNAL_USER;
    }
    return FreshRSS_UserDAO::ctouch($username);
}
```

**原理：** 通过更新用户数据目录的 mtime 时间戳，使 HTTP 条件请求（`If-Modified-Since`、`ETag`）失效，强制客户端重新获取数据。

**使用场景：**
- 刷新订阅后
- 清理旧条目后
- 数据库优化后
- 任何修改用户数据的操作后
