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

### 各命令/入口的互斥保护缺失矩阵

上面列出的是通用的并发风险。如果聚焦到**单用户刷新、清理、库优化、全量刷新**这几个维护命令之间的并行关系，会发现它们在上层几乎没有统一互斥保护——只有最底层的 Feed 级锁和数据库事务是共享的。

#### 命令保护对照表

| 命令/入口 | 全局刷新锁 | 用户级互斥锁 | Feed 级锁 | 数据库事务 | 数据库优化锁 | 并发写入保护 |
|----------|----------|-----------|----------|----------|-----------|-----------|
| **actualize_script.php（全量刷新）** | ✅ 有 | ✅ 间接（全局锁） | ✅ 有 | ✅ 有 | ❌ 无 | ✅ LOCK_EX |
| **actualize-user.php（单用户刷新）** | ❌ **无** | ❌ **无** | ✅ 有 | ✅ 有 | ❌ 无 | ✅ LOCK_EX |
| **purge.php（清理旧条目）** | ❌ **无** | ❌ **无** | ❌ **无** | ✅ 有 | ❌ 无 | ✅ LOCK_EX |
| **db-optimize.php（库优化）** | ❌ **无** | ❌ **无** | ❌ **无** | ❌ **无** | ❌ **无** | ✅ LOCK_EX |
| feedAction（Web 刷新） | ❌ **无** | ❌ **无** | ✅ 有 | ✅ 有 | ❌ 无 | ✅ LOCK_EX |
| JS 轮询（仅维护） | ❌ **无** | ❌ **无** | N/A | ❌ 无 | ❌ 无 | ✅ LOCK_EX |
| WebSub pshb.php | ❌ **无** | ❌ **无** | ✅ 有 | ✅ 有 | ❌ 无 | ✅ LOCK_EX |
| greader API 导入 | ❌ **无** | ❌ **无** | ✅ 有 | ✅ 有 | ❌ 无 | ✅ LOCK_EX |

#### 四个核心命令缺少的统一互斥保护逐项分析

##### 1. actualize-user.php（单用户刷新）缺少的保护

- **不检查全局刷新锁**：`actualize_script.php` 持有 `actualize.freshrss.lock` 全局锁时，`actualize-user.php` 可以无视它直接运行，两个进程可能同时刷新同一用户的 feed（靠 Feed 级锁兜底，不同 feed 仍会并行造成资源浪费）
- **无用户级互斥锁**：两个 `actualize-user.php --user alice` 进程可以同时跑

##### 2. purge.php（清理旧条目）缺少的保护

- **不检查全局刷新锁**：全量刷新运行时可以同时执行清理
- **无用户级互斥锁**：可与同一用户的刷新、优化并行
- **无 Feed 级锁**：清理某个 feed 的旧条目时，该 feed 可能正在被刷新，写入和删除同时操作 `_entry` 表

##### 3. db-optimize.php（库优化）缺少的保护

- **不检查全局刷新锁**：全量刷新运行时可以同时执行优化
- **无用户级互斥锁**：可与同一用户的刷新、清理并行
- **无数据库级保护**：
  - MySQL：`OPTIMIZE TABLE` 会隐式锁表，但脚本本身不做前置检查
  - SQLite：`VACUUM` 需要独占写锁，与任何写入并发都会导致 `database is locked` 错误
- **无事务**：优化操作本身不在事务中

##### 4. actualize_script.php（全量刷新）的相对完备

- 这是**唯一**具备全局互斥锁的入口
- 但该锁只防止自身重复执行，不被其他任何命令/入口检查
- 与 `purge.php`、`db-optimize.php` 同时运行时，靠数据库事务和行锁兜底

#### 典型并行冲突场景

| 并行组合 | 现象 | 后果 | 风险等级 |
|---------|------|------|---------|
| **actualize-user + actualize_script** | 全量刷新持有全局锁，但单用户刷新不检查 | 两进程同时刷新同一用户，依赖 Feed 级锁（同 feed 串行、不同 feed 并行） | 中 |
| **purge + db-optimize** | 两者均无用户级锁，同时操作同一数据库 | MySQL `OPTIMIZE TABLE` 锁表阻塞 purge；SQLite `VACUUM` 与写入冲突报错 | **高** |
| **purge + actualize-\*** | 同操作 `_entry` 表不同行范围 | 功能基本安全，但事务持有时间变长，增加死锁概率 | 中 |
| **actualize-user + actualize-user** | 同用户无互斥锁 | 同一用户多个刷新进程并行，数据库连接压力倍增 | 中 |
| **WebSub + actualize_script** | 两条路径同时刷新同一 feed | Feed 级锁有效保护，后到的请求跳过 | 低 |

#### 缺失保护的完整建议列表

| 缺失项 | 影响的命令 | 建议补充的保护 |
|-------|----------|--------------|
| **单用户级互斥锁** | actualize-user、feedAction、purge、db-optimize | 在用户目录创建 `{username}.maintenance.lock` |
| **actualize-user 检查全局锁** | actualize-user.php | 读取 `actualize.freshrss.lock` 并检查 TTL，锁存在时退出 |
| **db-optimize 用户级锁** | db-optimize.php | 优化期间阻止其他读写操作 |
| **purge Feed 级锁** | purge.php | 清理 feed 时对该 feed 加锁，避免与刷新冲突 |
| **WebSub 检查全局锁** | pshb.php | 全局刷新运行时延迟处理推送，存入 Retry-After |
| **跨入口统一协调层** | 所有入口 | 抽象刷新/维护调度层，统一管理锁与优先级 |

#### SQLite 的特殊风险

SQLite 采用文件级写锁（单写多读模型），以下操作在 SQLite 后端下并发风险显著高于 MySQL/PostgreSQL：

- `db-optimize.php` 执行 `VACUUM` 需要独占写锁，会阻塞所有其他数据库操作
- `purge.php` 的批量 DELETE 会长时间持有写锁
- 多个用户并发刷新时，SQLite 串行化所有写操作，性能急剧下降

**建议**：SQLite 后端避免并行执行任何维护命令，使用 `flock` 或任务队列串行化。

---

## 核心刷新命令详解

### 全量刷新的多入口体系——不只是 CLI 脚本

在展开具体命令之前，需要先明确：刷新订阅**不是只有两组 CLI 脚本**。FreshRSS 实际包含 **6 类独立入口**，它们共用最底层的 `actualizeFeeds()` 函数和 Feed 级锁，但上层没有统一的互斥协调。

#### 6 类入口全景

| 入口类型 | 入口文件/类 | 触发方式 | 触发对象 | 是否有全局锁 | 是否有用户级锁 |
|---------|-----------|---------|---------|------------|--------------|
| **CLI 全量脚本** | [actualize_script.php](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/app/actualize_script.php) | Cron / 手动 | 所有用户 | ✅ 有 | 间接（通过全局锁） |
| **CLI 单用户** | [actualize-user.php](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/actualize-user.php) | 手动 / 脚本 | 指定用户 | ❌ 无 | ❌ 无 |
| **Web 在线 Cron** | [feedController::actualizeAction()](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/app/Controllers/feedController.php#L954-L1033) | HTTP 请求 `?c=feed&a=actualize` | 当前用户 | ❌ 无 | ❌ 无 |
| **Web JavaScript 轮询** | [javascriptController::actualizeAction()](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/app/Controllers/javascriptController.php#L21-L42) | AJAX 后台轮询 | 当前用户 | ❌ 无 | ❌ 无 |
| **Google Reader API** | [greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/p/api/greader.php#L325-L332) | API 订阅导入后 | 当前用户 | ❌ 无 | ❌ 无 |
| **WebSub 实时推送** | [pshb.php](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/p/api/pshb.php#L140-L168) | 外部 Hub 推送 | 订阅该 feed 的所有用户 | ❌ 无 | ❌ 无 |

#### 入口间调用关系

```
                    ┌─────────────────────────┐
                    │  actualize_script.php   │── Cron
                    │  (全局互斥锁 + 多用户)  │
                    └───────────┬─────────────┘
                                │ 模拟 Web 请求
                                ▼
                    ┌─────────────────────────┐
CLI 手动 ──────────▶│ actualize-user.php      │── 两次维护钩子
                    │  (直接函数调用)         │
                    └───────────┬─────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          ▼                     ▼                     ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ feedController   │  │ javascriptCtrl   │  │ WebSub pshb.php  │
│ actualizeAction  │  │ (仅维护钩子)     │  │ (推送多用户)     │
└──────────┬───────┘  └──────────────────┘  └──────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ actualizeFeedsAndCommit()    │
│ + Feed 级锁（所有入口共享）  │
└──────────┬───────────────────┘
           ▲
           │
┌──────────┴──────────────┐
│ greader API import 后  │
│ 自动调用刷新            │
└────────────────────────┘
```

各入口关键差异：
- **actualize_script.php**：唯一拥有全局互斥锁的入口，通过模拟 Web 请求走 MVC 路由调用 `actualizeAction()`
- **actualize-user.php**：不走 MVC 路由，直接调用底层核心函数，在 `minorDbMaintenance()` 前后**两次**触发维护钩子
- **feedController::actualizeAction()**：Web/在线 Cron 入口，只在 `minorDbMaintenance()` 后触发一次钩子，支持 `id/url/maxFeeds` 参数
- **javascriptController::actualizeAction()**：浏览器 AJAX 轮询入口，**不实际刷新 feed**，只执行维护钩子和 minorDbMaintenance，返回 feed 更新时间排序供前端决策
- **WebSub pshb.php**：唯一被动入口，接收外部 Hub 推送的 XML，一次推送可刷新多个用户的同一 feed
- **greader API**：OPML 导入成功后自动触发，不经过维护钩子和数据库维护

---

### actualize-user.php - 单用户刷新

[`cli/actualize-user.php`](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/actualize-user.php)

**参数：**
- `--user` (必填)：用户名

**执行流程（精确代码顺序 + 两次维护钩子的前后关系）：**

1. 检查系统需求：`performRequirementCheck()`
2. 初始化用户上下文：`cliInitUser($cliOptions->user)`
3. **第 1 次触发用户维护钩子**（`minorDbMaintenance` **之前**）：`callHookVoid(Minz_HookType::FreshrssUserMaintenance)` — [第 23 行](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/actualize-user.php#L23)
4. 执行轻量数据库维护：`$databaseDAO->minorDbMaintenance()`
5. **第 2 次触发用户维护钩子**（`minorDbMaintenance` **之后**）：`callHookVoid(Minz_HookType::FreshrssUserMaintenance)` — [第 29 行](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/actualize-user.php#L29)
6. 提交暂存新条目：`FreshRSS_feed_Controller::commitNewEntries()`
7. 更新 Feed 缓存值：`$feedDAO->updateCachedValues()`
8. 刷新动态 OPML：`FreshRSS_category_Controller::refreshDynamicOpmls()`
9. 执行实际 Feed 刷新：`FreshRSS_feed_Controller::actualizeFeedsAndCommit()`
10. 失效 HTTP 缓存：`invalidateHttpCache($username)`
11. 输出结果并退出：`done($nbUpdatedFeeds > 0)`

**两次维护钩子的前后顺序与各入口对比：**

这是 `actualize-user.php` **独有的行为**。所有入口的钩子调用情况对比如下：

| 入口 | 钩子调用次数 | 调用位置相对 minorDbMaintenance |
|-----|------------|------------------------------|
| **actualize-user.php** | **2 次** | **之前 1 次 + 之后 1 次** |
| actualize_script.php（via actualizeAction） | 1 次 | 之后（[feedController.php:971](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/app/Controllers/feedController.php#L971)） |
| Web 在线 cron（actualizeAction） | 1 次 | 之后 |
| JavaScript 轮询（javascriptController） | 1 次 | 之后（[javascriptController.php:35](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/app/Controllers/javascriptController.php#L35)） |
| greader API、WebSub、purge、db-optimize | 0 次 | 不触发 |

代码对照：

```php
// cli/actualize-user.php - 独有两次调用模式
Minz_ExtensionManager::callHookVoid(Minz_HookType::FreshrssUserMaintenance);  // 第 1 次：之前
$databaseDAO->minorDbMaintenance();
Minz_ExtensionManager::callHookVoid(Minz_HookType::FreshrssUserMaintenance);  // 第 2 次：之后

// app/Controllers/feedController.php + javascriptController.php - 标准一次调用模式
$databaseDAO = FreshRSS_Factory::createDatabaseDAO();
$databaseDAO->minorDbMaintenance();
Minz_ExtensionManager::callHookVoid(Minz_HookType::FreshrssUserMaintenance);  // 仅之后
```

推测两次钩子的设计意图：
- **第 1 次（之前）**：允许扩展在数据库 schema 更新前做准备/迁移工作
- **第 2 次（之后）**：允许扩展在 schema 更新后执行依赖新结构的维护任务

对扩展开发者的实际影响：
1. 扩展需自行确保幂等性——`actualize-user.php` 钩子会跑两次，Web 入口只跑一次
2. 如果主要更新渠道是 WebSub 或 greader API，维护钩子可能**永不执行**
3. 用户每打开一次 Web 页面，JS 轮询就会触发一次钩子，可能执行过于频繁
4. `purge.php` 和 `db-optimize.php` 虽修改数据库，但不通知扩展

**返回值：**
- 退出码 0：有 feed 被更新
- 退出码 1：无 feed 更新或出错

---

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

---

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

---

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

> ⚠️ **不会失效 HTTP 缓存**：`db-optimize.php` 执行完 `optimize()` 后直接调用 `done($ok)` 退出，**没有调用 `invalidateHttpCache()`**。这意味着优化后用户的 Web 客户端不会感知到数据变化——浏览器仍使用旧的 `If-Modified-Since` / `ETag` 条件请求命中缓存，直到其他操作（刷新、清理）触发缓存失效。对比其他命令：`actualize-user.php`（[L49](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/actualize-user.php#L49)）、`purge.php`（[L39](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/cli/purge.php#L39)）、`actualize_script.php`（[L108](file:///d:/fz/0601-1/solo-dogfeeding/code/30-FreshRSS/app/actualize_script.php#L108)）均在执行完毕后调用了 `invalidateHttpCache()`。如果需要在优化后立即让客户端看到变化，应手动执行 `touch data/users/{username}/` 或在优化后触发一次刷新。

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
