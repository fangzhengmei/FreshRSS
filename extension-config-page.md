# FreshRSS 扩展加载与配置页机制

## 一、扩展发现（Extension Discovery）

扩展发现的核心逻辑在 [ExtensionManager::init()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/ExtensionManager.php#L57-L102)。

### 1.1 扫描目录

FreshRSS 从两个固定目录扫描扩展：

| 常量 | 路径 | 含义 |
|---|---|---|
| `CORE_EXTENSIONS_PATH` | `lib/core-extensions/` | 内置核心扩展 |
| `THIRDPARTY_EXTENSIONS_PATH` | `extensions/` | 第三方扩展 |

扫描过程（[第 60-65 行](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/ExtensionManager.php#L60-L65)）：

```php
$list_core_extensions = array_diff(scandir(CORE_EXTENSIONS_PATH) ?: [], [ '..', '.' ]);
$list_thirdparty_extensions = array_diff(scandir(THIRDPARTY_EXTENSIONS_PATH) ?: [], [ '..', '.' ], $list_core_extensions);
```

核心扩展优先扫描，第三方扩展会排除与核心扩展同名的目录，避免重复加载。

### 1.2 metadata.json 校验

每个子目录必须包含 `metadata.json`，否则跳过。校验规则见 [isValidMetadata()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/ExtensionManager.php#L116-L119)：

- **必填字段**：`name`（扩展名）和 `entrypoint`（入口类名前缀）
- **`entrypoint`** 只允许字母、数字和下划线（`ctype_alnum` + 允许 `_`）
- 可选字段：`author`、`description`、`version`、`type`（`system` | `user`，默认 `user`）

示例（[UserCSS/metadata.json](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/core-extensions/UserCSS/metadata.json)）：

```json
{
    "name": "User CSS",
    "author": "hkcomori, Marien Fressinaud",
    "description": "Give possibility to overwrite the CSS with a user-specific rules.",
    "version": "1.1.1",
    "entrypoint": "UserCSS",
    "type": "user"
}
```

### 1.3 加载扩展类

通过 [load()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/ExtensionManager.php#L127-L156) 方法完成：

1. `include_once` 加载 `extension.php` 文件
2. 根据命名约定拼接类名：`{entrypoint}Extension`（如 `UserCSSExtension`）
3. 用 `class_exists()` 检查类是否存在
4. 实例化该类，传入 metadata 数组
5. 用 `instanceof Minz_Extension` 验证继承关系

**命名约定**：如果 `entrypoint` 是 `UserCSS`，则扩展类名必须为 `UserCSSExtension`，文件必须为 `extension.php`。

### 1.4 注册到列表

[register()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/ExtensionManager.php#L166-L173) 将扩展对象存入 `$ext_list` 静态数组（以 `name` 为 key）。同时，如果该扩展的类型为 `system` 且在系统配置的 `extensions_enabled` 中被标记为启用，则立即启用它。

---

## 二、扩展启停（Enable / Disable）

### 2.1 启用流程

启用扩展有两个入口：

#### A. 应用启动时自动启用

在 [FreshRSS::init()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/app/FreshRSS.php#L21-L70) 中，按以下顺序：

1. **系统扩展**：`Minz_ExtensionManager::init()` 内部，在 [register()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/ExtensionManager.php#L170-L172) 时检查 `system_conf->extensions_enabled`，自动启用系统扩展
2. **用户扩展**：在用户认证和配置初始化完成后，通过 [enableByList()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/ExtensionManager.php#L214-L223) 启用用户扩展（[第 61-62 行](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/app/FreshRSS.php#L61-L62)）

#### B. 用户手动启用

通过 [extensionController::enableAction()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/app/Controllers/extensionController.php#L162-L217) 处理：

1. 必须是 POST 请求
2. 查找扩展 → 检查是否已启用 → 检查权限（系统扩展需 admin）
3. 调用 `$ext->install()`（扩展可覆写此方法做数据库初始化等）
4. install() 返回 `true` 后，将扩展名写入对应配置的 `extensions_enabled` 数组并保存
5. 同时清理已消失或改变类型的扩展（`array_filter`）

### 2.2 禁用流程

通过 [extensionController::disableAction()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/app/Controllers/extensionController.php#L228-L283)：

1. 同样必须是 POST，检查权限
2. 调用 `$ext->uninstall()`（扩展可覆写做清理）
3. 在 `extensions_enabled` 中将该扩展设为 `false`（注意：不是删除 key，而是值改为 `false`）
4. 同样过滤已失效的扩展

### 2.3 运行时启用细节

[enable()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/ExtensionManager.php#L183-L206) 私有方法的执行步骤：

```
1. 加入 $ext_list_enabled
2. 注册 autoload（如果扩展定义了 autoload 方法）
3. 调用 $ext->enable() 设置 is_enabled 标志
4. 调用 $ext->init() 执行扩展初始化逻辑（注册 hook、加载翻译等）
5. 如果 init() 抛出 Minz_Exception → 记录日志、disable()、从启用列表移除
```

### 2.4 配置存储位置

启用状态存储在 `Minz_Configuration` 中：

| 扩展类型 | 存储位置 | 字段 |
|---|---|---|
| `system` | `data/config.php`（系统配置） | `extensions_enabled` |
| `user` | `data/users/{username}/config.php`（用户配置） | `extensions_enabled` |

`extensions_enabled` 结构为 `array<string, bool>`，key 是扩展名，value 表示启用（`true`）/禁用（`false`）。

对应默认配置：
- 系统默认：[config.default.php#L228-L231](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/config.default.php#L228-L231)
- 用户默认：[config-user.default.php#L143-L148](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/config-user.default.php#L143-L148)

### 2.5 系统扩展与用户扩展分两阶段启用的深度分析

在 [FreshRSS::init()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/app/FreshRSS.php#L21-L70) 中，系统扩展和用户扩展是分开两次启用的，中间穿插了若干关键初始化步骤。其代码顺序严格定义如下：

```
第 43 行:  Minz_ExtensionManager::init()          ← 阶段一：发现扩展 + 启用系统扩展
第 47 行:  self::initAuth()                        ← 认证系统初始化（创建 $_SESSION['currentUser']）
第 49 行:  FreshRSS_Context::initUser()            ← 用户配置加载
第 58 行:  self::initI18n()                        ← 国际化（依赖用户配置 language）
第 61-62 行: Minz_ExtensionManager::enableByList() ← 阶段二：启用用户扩展
```

#### 原因一：依赖的前置条件不同

系统扩展和用户扩展所依赖的系统基础设施完全不同，必须在不同的启动阶段运行：

| 扩展类型 | 可依赖的初始化步骤 | 不允许依赖 |
|---|---|---|
| **系统扩展** | `FreshRSS_Context::initSystem()`（系统配置已就绪） | 用户会话、用户配置、i18n、Auth |
| **用户扩展** | 系统配置 + Auth + 用户配置 + i18n + 共享系统 | 无额外限制 |

`ExtensionManager::init()` 在 [第 68-71 行](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/ExtensionManager.php#L68-L71) 读取系统配置中的 `extensions_enabled` 来启用系统扩展——此时用户配置甚至还不存在（因为 `initUser()` 还没执行）。

而用户扩展的启用调用在 [第 61-62 行](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/app/FreshRSS.php#L61-L62) 读取 `FreshRSS_Context::userConf()->extensions_enabled`，这显然要求用户上下文已经建立。

#### 原因二：扩展 `init()` 的典型操作对时机有要求

观察扩展 `init()` 可能执行的操作：

```php
// 在扩展 init() 中常见的调用
$this->registerHook(Minz_HookType::EntryBeforeDisplay, ...);   // 无需用户上下文
$this->registerTranslates();                                     // 无需用户上下文
Minz_View::appendStyle(...)                                      // 无需用户上下文
$this->getUserConfigurationValue(...)                            // 必须在用户配置就绪之后
Minz_User::name()                                                // 必须在 Auth 就绪之后
```

系统扩展（如修改认证流程、自定义登录页的扩展）必须在认证系统之前介入。例如 `FreshrssInit` Hook 在 [第 69 行](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/app/FreshRSS.php#L69) 被触发时，系统扩展和用户扩展都已可用——但系统扩展需要在更早时刻已经被注册才能影响中间的认证流程。

#### 原因三：权限与隔离

系统扩展由 admin 在系统配置中统一控制，对所有用户生效；用户扩展由每个用户独立控制。如果混合到一个阶段启用，就无法在运行时区分"系统层面强制启用"和"用户自行选择启用"的权限边界。

在 `enable()` 私有方法中，`$onlyOfType` 参数（[第 181-190 行](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/ExtensionManager.php#L181-L190)）确保每次 `enableByList()` 只启用指定类型的扩展，避免用户配置意外启用系统扩展。

#### 原因四：启动失败时的影响域可控

若系统扩展启动失败（抛异常），因为此时用户上下文尚未建立，失败仅影响全局级别，不会污染任何用户会话；若用户扩展启动失败，其影响仅局限于单个用户的启用状态（见 2.6 节的隔离机制）。两个阶段之间的间隔使得故障定位清晰。

#### 其他入口点的一致性

在 Web API（[p/api/greader.php#L1123-L1124](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/p/api/greader.php#L1123-L1124)、[p/api/pshb.php#L137-L155](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/p/api/pshb.php#L137-L155)、[p/api/query.php#L63-L64](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/p/api/query.php#L63-L64)）以及 CLI（[cli/_cli.php#L18-L43](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/cli/_cli.php#L18-L43)）中，都严格遵循同样的两阶段模式：先 `ExtensionManager::init()`（系统扩展），在用户上下文就绪后再 `enableByList(userConf->extensions_enabled, 'user')`。这不是偶然，而是架构级约束。

### 2.6 启用失败的影响隔离机制

启用失败发生在 `ExtensionManager::enable()` 的 `$ext->init()` 调用处（[第 198-204 行](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/ExtensionManager.php#L198-L204)），其隔离措施是多层的：

```php
try {
    $ext->init();
} catch (Minz_Exception $e) {
    Minz_Log::warning('Error while enabling extension ' . $ext->getName() . ': ' . $e->getMessage());
    $ext->disable();                         // 1. 标记扩展为禁用
    unset(self::$ext_list_enabled[$ext_name]); // 2. 从内存启用列表移除
}
```

#### 隔离层一：内存态隔离

异常被 catch 后，执行两个关键操作：
1. `$ext->disable()`：将扩展对象内部的 `is_enabled` 标志重置为 `false`
2. `unset(self::$ext_list_enabled[$ext_name])`：从静态启用列表中移除该扩展

这保证后续调用 `listExtensions(true)` 和 `isExtensionEnabled()` 都看不到这个扩展。其他扩展的启用不受影响（`enableByList()` 使用 foreach 循环逐个启用，单个的异常被限制在那一次 `enable()` 调用内）。

#### 隔离层二：持久化与内存态不一致（保护性设计）

注意：**此处 catch 块不会修改持久化配置中的 `extensions_enabled`**。这是有意为之：

- 如果扩展因为临时原因（依赖服务未就绪）而 init() 失败，下次请求可能成功。配置中仍然标记为"已启用"，意味着下次启动时会再次尝试启用。
- 如果扩展永久损坏，用户（或管理员）仍然可以在扩展管理界面看到它（因为它在 `$ext_list` 中仍存在），并通过 `disableAction()` 显式地将配置中的状态改为 `false`，同时看到错误日志链接进行排查。

这实现了"**启用失败不自动回滚配置**"的策略——把是否永久禁用的决定权交还给管理员，而不是静默地把扩展从配置中清除。

#### 隔离层三：仅捕获 Minz_Exception

catch 语句只捕获 `Minz_Exception`（而不是更宽泛的 `Exception` 或 `Throwable`）。这是一种契约：扩展开发者应使用 `Minz_ExtensionException`（继承自 `Minz_Exception`）抛出预期内的错误。原生 PHP Error（如语法错误、类型错误）将穿透到上层错误处理，从而暴露真正需要修复的代码缺陷，而不是被静默吞掉。

#### 不同启动阶段的故障影响范围

| 阶段 | 启用失败的影响范围 | 恢复手段 |
|---|---|---|
| 系统扩展阶段一 | 全局：该系统扩展对所有用户不可用，其他系统扩展不受影响 | 管理员修复扩展或在配置中关闭它 |
| 用户扩展阶段二 | 仅该用户：该扩展对当前用户不可用，不影响其他用户 | 用户在扩展管理界面操作 |

---

## 三、配置表单（Configuration Page）

### 3.1 配置页面入口

[extensionController::configureAction()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/app/Controllers/extensionController.php#L123-L151) 是配置页的控制器方法：

```
参数：e = 扩展名（URL 编码）
权限：系统扩展需 admin 权限
```

流程：
1. 解码扩展名 → `findExtension()` 查找扩展
2. 找不到 → 404；系统扩展非 admin → 403
3. **调用 `$ext->handleConfigureAction()`**：这是扩展处理 POST 请求、保存配置的入口
4. 渲染配置视图

### 3.2 配置视图渲染

[helpers/extension/configure.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/app/views/helpers/extension/configure.phtml) 是配置页的通用框架：

1. 显示扩展名、版本号、启用/禁用状态
2. 显示扩展描述和作者
3. Admin 用户可看到"删除扩展"按钮
4. **调用 `$this->extension->getConfigureView()`**：获取扩展自定义配置表单 HTML

[getConfigureView()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/Extension.php#L114-L123) 的实现：

```php
final public function getConfigureView(): string|false {
    $filename = $this->path . '/configure.phtml';
    if (!file_exists($filename)) {
        return false;   // 没有配置视图文件
    }
    ob_start();
    include $filename;
    return ob_get_clean();  // 返回渲染后的 HTML
}
```

关键：`configure.phtml` 在扩展类的作用域内 `include`，因此 `$this` 指向扩展对象本身，模板可直接访问扩展的属性和方法。

### 3.3 配置表单实践

以 [UserCSSExtension](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/core-extensions/UserCSS/extension.php) 为例：

**extension.php** — 处理逻辑：

```php
public function handleConfigureAction(): void {
    parent::init();
    $this->registerTranslates();

    if (Minz_Request::isPost()) {
        $css_rules = Minz_Request::paramString('css-rules', plaintext: true);
        $this->saveFile(self::FILENAME, $css_rules);      // 保存到用户数据目录
        FreshRSS_UserDAO::touch();
    }

    $this->css_rules = '';
    if ($this->hasFile(self::FILENAME)) {
        $this->css_rules = htmlspecialchars($this->getFile(self::FILENAME) ?? '', ENT_NOQUOTES, 'UTF-8');
    }
}
```

**configure.phtml** — 表单模板（`$this` 是扩展对象）：

```php
<form action="<?= _url('extension', 'configure', 'e', urlencode($this->getName())) ?>" method="post">
    <input type="hidden" name="_csrf" value="<?= FreshRSS_Auth::csrfToken() ?>" />
    <textarea name="css-rules"><?= $this->css_rules ?></textarea>
    <button type="submit">Submit</button>
</form>
```

### 3.4 扩展配置数据的读写

扩展配置有两种存储机制：

#### A. 结构化配置（extensions 字段）

通过 [Minz_Extension](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/Extension.php#L308-L502) 提供的 API：

| 方法 | 说明 |
|---|---|
| `getUserConfigurationValue($key)` | 读取用户级配置 |
| `setUserConfigurationValue($key, $value)` | 写入用户级配置 |
| `getUserConfigurationBool/String/Int/Array($key)` | 类型安全读取 |
| `getSystemConfigurationValue($key)` | 读取系统级配置 |
| `setSystemConfigurationValue($key, $value)` | 写入系统级配置 |
| `removeUserConfiguration()` | 清除用户配置 |
| `removeSystemConfiguration()` | 清除系统配置 |

配置存储在 `conf->extensions[扩展名]` 下，即：

```php
// 用户配置文件 data/users/{username}/config.php
'extensions' => [
    'User CSS' => ['key1' => 'value1', ...],
]
```

#### B. 文件存储（用户数据目录）

通过文件 API：

| 方法 | 说明 |
|---|---|
| `saveFile($filename, $content)` | 保存到 `data/users/{user}/extensions/{entrypoint}/` |
| `getFile($filename)` | 读取文件内容 |
| `hasFile($filename)` | 检查文件是否存在 |
| `removeFile($filename)` | 删除文件 |
| `getFileUrl($filename)` | 获取文件 URL |

### 3.5 静态资源服务

扩展的静态文件通过 [p/ext.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/p/ext.php) 提供，安全限制：

- 文件必须在 `ext_dir/static/` 目录下
- 禁止路径遍历（`..` 检测）
- 支持的 MIME 类型：css、js、gif、jpeg、jpg、png、svg
- 支持 HTTP 条件缓存（304）

用户私有文件通过 [serveAction()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/app/Controllers/extensionController.php#L336-L359) 提供，需要扩展已启用且文件存在。

### 3.6 配置页的两种显示模式

在 [configureAction()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/app/Controllers/extensionController.php#L124-L129) 中：

- **AJAX 模式**（`ajax=1`）：无布局，仅返回配置内容
- **Slider 模式**（`slider=1`）：在扩展列表页右侧的 slider 面板中展示
- **默认**：独立页面，带 `aside_configure` 侧边栏

### 3.9 扩展列表页

[indexAction()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/app/Controllers/extensionController.php#L25-L41) 展示：

1. **系统扩展** 和 **用户扩展** 分区显示
2. 每个扩展显示启用/禁用开关、配置按钮（[details.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/app/views/helpers/extension/details.phtml)）
3. **社区扩展** 列表：从 GitHub 拉取 `extensions.json`，缓存 24 小时，显示版本、兼容性等信息

---

## 四、异常隔离（Exception Isolation）

FreshRSS 在多处对扩展代码做了异常隔离，确保单个扩展的故障不会影响整个应用。

### 4.1 扩展加载阶段

[load()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/ExtensionManager.php#L140-L147) 中，实例化扩展类时用 try-catch 包裹：

```php
try {
    $extension = new $ext_class_name($info);
} catch (Exception $e) {
    Minz_Log::warning("Invalid extension `{$ext_class_name}`: " . $e->getMessage());
    return null;   // 加载失败，跳过此扩展
}
```

metadata.json 无效、类不存在、类非 Minz_Extension 子类等情况均只记日志并跳过。

### 4.2 扩展启用阶段

[enable()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/ExtensionManager.php#L198-L204) 中，init() 调用被 try-catch 包裹：

```php
try {
    $ext->init();
} catch (Minz_Exception $e) {
    Minz_Log::warning('Error while enabling extension ' . $ext->getName() . ': ' . $e->getMessage());
    $ext->disable();
    unset(self::$ext_list_enabled[$ext_name]);
}
```

init() 抛出异常 → 扩展被自动禁用、从启用列表移除，不影响其他扩展。

### 4.3 配置处理阶段

[configureAction()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/app/Controllers/extensionController.php#L145-L150) 中：

```php
try {
    $this->view->extension->handleConfigureAction();
} catch (Minz_Exception $e) {
    Minz_Log::error('Error while configuring extension ' . $ext->getName() . ': ' . $e->getMessage());
    Minz_Request::bad(_t('feedback.extensions.enable.ko', $ext_name, _url('index', 'logs')), ...);
}
```

配置处理抛出异常 → 记录错误日志、向用户显示错误反馈。

### 4.4 启停操作阶段

[enableAction()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/app/Controllers/extensionController.php#L191-L213) 和 [disableAction()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/app/Controllers/extensionController.php#L257-L279) 中：

扩展的 `install()` / `uninstall()` 返回值用于判断成败，返回非 `true` 的字符串作为错误信息。如果失败则不修改 `extensions_enabled` 配置。

### 4.5 专用异常类

[Minz_ExtensionException](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/ExtensionException.php) 是扩展专用异常，自动在消息中附加扩展名，便于日志定位：

```php
class Minz_ExtensionException extends Minz_Exception {
    public function __construct(string $message, string $extension_name = '', int $code = self::ERROR) {
        if ($extension_name !== '') {
            $message = 'An error occurred in `' . $extension_name . '` extension with the message: ' . $message;
        }
        parent::__construct($message, $code);
    }
}
```

在 [Minz_Extension::setType()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/Extension.php#L160-L165) 中使用：

```php
if (!in_array($type, ['user', 'system'], true)) {
    throw new Minz_ExtensionException('invalid `type` info', $this->name);
}
```

### 4.6 隔离策略总结

| 阶段 | 隔离方式 | 失败后果 |
|---|---|---|
| 发现/加载 | 跳过 + 日志 | 扩展不出现 |
| 实例化 | try-catch + 返回 null | 扩展不注册 |
| init() | try-catch + 自动禁用 | 扩展被禁用 |
| install()/uninstall() | 返回值检查 | 操作回滚提示 |
| handleConfigureAction() | try-catch + 错误反馈 | 用户看到错误 |
| 静态文件服务 | 路径校验 + MIME 白名单 | 400/404 拒绝 |

---

## 五、Hook 系统（扩展与核心通信）

扩展通过 Hook 注册回调，与核心逻辑交互。

### 5.1 Hook 类型

定义在 [Minz_HookType](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/HookType.php) 枚举中，包含 30+ 种钩子，例如：

| Hook | 签名 | 说明 |
|---|---|---|
| `EntryBeforeDisplay` | OneToOne | 显示前修改文章 |
| `FeedBeforeActualize` | OneToOne | 刷新前修改 Feed |
| `FreshrssInit` | NoneToNone | 应用初始化完成 |
| `MenuConfigurationEntry` | NoneToString | 添加配置菜单项 |
| `JsVars` | OneToOne | 注入 JS 变量 |

### 5.2 四种签名模式

定义在 [Minz_HookSignature](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/HookSignature.php)：

| 签名 | 输入 | 输出 | 链式行为 |
|---|---|---|---|
| `NoneToNone` | 无 | 无 | 逐个调用，无交互 |
| `NoneToString` | 无 | string | 拼接所有返回值 |
| `OneToOne` | 一个参数 | 修改后的值 | 链式传递（null/false 中断） |
| `PassArguments` | 透传参数 | 任意 | 首个非 null 返回值终止 |

### 5.3 注册 Hook

扩展在 `init()` 中注册：

```php
$this->registerHook(Minz_HookType::EntryBeforeDisplay, [$this, 'myHookMethod']);
```

[registerHook()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/Extension.php#L265-L267) 最终调用 [ExtensionManager::addHook()](file:///d:/fz/0601-1/solo-dogfeeding/code/28-FreshRSS/lib/Minz/ExtensionManager.php#L262-L271)，支持优先级排序。

---

## 六、端到端流程图

```
应用启动
  │
  ├─ FreshRSS_Context::initSystem()
  │
  ├─ Minz_ExtensionManager::init()          ← 发现 & 加载
  │   ├─ scandir(CORE_EXTENSIONS_PATH)
  │   ├─ scandir(THIRDPARTY_EXTENSIONS_PATH)
  │   ├─ 对每个目录:
  │   │   ├─ 读取 metadata.json → 校验
  │   │   ├─ include extension.php
  │   │   ├─ new {Entrypoint}Extension($meta)
  │   │   └─ register() → $ext_list
  │   │       └─ 系统扩展 + 已启用 → 自动 enable()
  │   └─ reset() 清理旧状态
  │
  ├─ FreshRSS_Auth::init()
  ├─ FreshRSS_Context::initUser()
  ├─ Minz_Translate::init()
  │
  ├─ ExtensionManager::enableByList()       ← 启用用户扩展
  │   └─ 对每个 extensions_enabled[key]=true:
  │       └─ enable() → init() → registerHook() 等
  │
  └─ callHookVoid(FreshrssInit)             ← 通知扩展初始化完成

用户操作: 点击"配置"
  │
  ├─ extensionController::configureAction()
  │   ├─ findExtension($name)
  │   ├─ $ext->handleConfigureAction()      ← 扩展处理 POST / 准备数据
  │   └─ 渲染 configure.phtml
  │       └─ $ext->getConfigureView()        ← include 扩展的 configure.phtml

用户操作: 点击"启用"
  │
  ├─ extensionController::enableAction()
  │   ├─ $ext->install()                    ← 扩展安装钩子
  │   ├─ conf->extensions_enabled[name] = true
  │   └─ conf->save()
```
