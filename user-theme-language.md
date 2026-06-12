# FreshRSS 用户配置、主题与语言选择机制

## 1. 用户配置加载机制

### 1.1 配置层级

FreshRSS 的配置采用 **默认值 → 用户自定义覆盖** 的两层合并策略，由 `Minz_Configuration` 基类实现。

| 层级 | 文件路径 | 说明 |
|------|----------|------|
| 默认值 | `config-user.default.php` | 代码仓库内置，定义所有用户配置项的缺省值 |
| 用户覆盖 | `data/users/{username}/config.php` | 运行时由用户操作生成，仅包含被修改的项 |

合并逻辑位于 [Minz_Configuration::__construct()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/lib/Minz/Configuration.php#L108-L129)：

1. 先加载默认文件 `$this->data = self::load($this->default_filename);`
2. 再用 `array_replace_recursive($this->data, self::load($this->config_filename))` 将用户配置覆盖到默认值上
3. 最终只保留字符串键：`array_filter($overloaded, 'is_string', ARRAY_FILTER_USE_KEY)`
4. 若用户配置文件不存在且无默认文件，则抛出 `Minz_FileNotExistException`

### 1.2 配置访问入口

- **系统配置**：`FreshRSS_Context::systemConf()` → 来自 `data/config.php`（覆盖 `config.default.php`）
- **用户配置**：`FreshRSS_Context::userConf()` → 来自 `data/users/{username}/config.php`（覆盖 `config-user.default.php`）

初始化顺序见 [FreshRSS::init()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/FreshRSS.php#L21-L70)：

```
1. FreshRSS_Context::initSystem()        → 加载系统配置
2. Minz_ExtensionManager::init()         → 初始化扩展管理器
3. self::initAuth()                      → 初始化认证
4. FreshRSS_Context::initUser()          → 加载当前用户配置
5. self::initI18n()                      → 初始化国际化（依赖用户配置中的 language）
6. 启用用户扩展
```

### 1.3 用户配置初始化细节

[FreshRSS_Context::initUser()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Models/Context.php#L101-L175) 的关键步骤：

1. 调用 `FreshRSS_UserConfiguration::init()` 加载用户配置
2. 对 `language` 字段做规范化：将 `-xx` 后缀转为大写（如 `zh-cn` → `zh-CN`）
3. 处理旧版配置兼容（archiving、display_categories、shortcuts 等）

### 1.4 建号时的配置合并

[FreshRSS_user_Controller::createUser()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/userController.php#L347-L406) 在创建新用户时，按以下顺序决定初始配置：

1. 若 `data/config-user.custom.php` 存在，先加载其中的数组作为基础
2. 再用调用方传入的 `$userConfigOverride` 做 `array_merge` 覆盖
3. 对 `language` 字段做验证：若语言不存在于翻译目录，强制回退为 `'en'`
4. 最终写入 `data/users/{username}/config.php`

```php
// userController.php L370-L372
if (!Minz_Translate::exists(is_string($userConfig['language'] ?? null) ? $userConfig['language'] : '')) {
    $userConfig['language'] = Minz_Translate::DEFAULT_LANGUAGE;  // 回退为 'en'
}
```

---

## 2. 语言选择机制

### 2.1 语言优先级

语言由 [FreshRSS::initI18n()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/FreshRSS.php#L90-L103) 确定，核心调用链：

```php
$userLanguage = FreshRSS_Context::hasUserConf() ? FreshRSS_Context::userConf()->language : null;
$systemLanguage = FreshRSS_Context::hasSystemConf() ? FreshRSS_Context::systemConf()->language : null;
$language = Minz_Translate::getLanguage($userLanguage, Minz_Request::getPreferredLanguages(), $systemLanguage);
```

[Minz_Translate::getLanguage()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/lib/Minz/Translate.php#L126-L141) 的优先级从高到低：

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1 | `$user`（用户配置 `language` 字段） | 已登录用户在「显示配置」页面设置的语言 |
| 2 | `$preferred`（浏览器 `Accept-Language` 头） | 用户未登录时，按浏览器偏好列表依次匹配 |
| 3 | `$default`（系统配置 `language` 字段） | 管理员在系统配置中设定的语言 |
| 4 | `Minz_Translate::DEFAULT_LANGUAGE` = `'en'` | 硬编码的最终兜底值 |

**注意**：优先级 1 中，如果用户配置的 `$user` 语言在 `app/i18n/` 目录中不存在对应翻译文件夹，则直接回退到 `'en'`，**不会**继续尝试优先级 2 和 3。

### 2.2 浏览器语言解析

[Minz_Request::getPreferredLanguages()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/lib/Minz/Request.php#L583-L588) 解析 `HTTP_ACCEPT_LANGUAGE` 头，返回语言列表；若无该头则返回 `['en']`。

### 2.3 语言切换生效路径

用户在「显示配置」页面提交语言设置后（[configureController::displayAction()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/configureController.php#L46-L103)）：

```
1. Minz_Request::paramString('language') → 从 POST 取得新语言值
2. Minz_Translate::exists($language)     → 验证该语言有对应翻译
3. userConf()->language = $language      → 写入用户配置对象
4. userConf()->save()                    → 持久化到 data/users/{user}/config.php
5. Minz_Session::_param('language', ...) → 更新 Session
6. Minz_Translate::reset($language)      → 立即重新加载翻译
7. 页面重定向 → 下次请求时 initI18n() 从 userConf()->language 读取
```

### 2.4 翻译文件结构

翻译文件位于 `app/i18n/{lang}/`，每个子目录对应一种语言，内部按模块分文件：

- `gen.php` — 通用词条
- `conf.php` — 配置页词条
- `admin.php` — 管理页词条
- `sub.php` — 订阅页词条
- `plurals.php` — 复数形式规则

加载逻辑：[Minz_Translate::loadLang()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/lib/Minz/Translate.php#L158-L196) 若所选语言目录不存在，自动回退到 `en/`。

---

## 3. 各入口的语言选择路径详解

不同入口点（匿名访问、HTTP 认证自动建号、注册页、各 API）触发语言选择的方式和优先级不同，以下逐条分清。

### 3.1 匿名访问（allow_anonymous）

**触发条件**：系统配置 `allow_anonymous = true`，未登录用户访问主页面。

**代码路径**：

1. [FreshRSS_Auth::init()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Models/Auth.php#L18-L44)：未登录时 `Minz_User::name()` 为 null，Session 中 `CURRENT_USER` 被设为 `systemConf()->default_user`，但 `login_ok = false`
2. [FreshRSS_Context::initUser()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Models/Context.php#L101-L175)：以 `default_user` 加载其用户配置 → **匿名用户看到的语言/主题来自默认用户的配置**
3. [FreshRSS::initI18n()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/FreshRSS.php#L90-L103)：此时 `hasUserConf() = true`（默认用户的配置已加载），走主流程优先级

**语言优先级**：

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1 | 默认用户的 `userConf()->language` | 匿名用户实质上继承了默认用户的语言设置 |
| 2 | 浏览器 `Accept-Language` | 仅在默认用户的 language 为 null 时（正常不会） |
| 3 | 系统配置 `systemConf()->language` | 优先级 2 仍无匹配时 |
| 4 | `'en'` | 最终兜底 |

**关键点**：匿名用户**没有**独立的浏览器偏好覆盖机制。即使匿名用户的浏览器偏好是 `zh-CN`，只要默认用户配置了 `fr`，页面就显示法语。

### 3.2 HTTP 认证自动建号（http_auth_auto_register）

**触发条件**：系统配置 `auth_type = 'http_auth'` 且 `http_auth_auto_register = true`，HTTP 头中提供了合法用户名但该用户在 FreshRSS 中不存在。

**代码路径**：

1. [FreshRSS_Auth::accessControl()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Models/Auth.php#L69-L86)：检测到 HTTP 认证用户不存在
2. 在建号前**提前**做语言选择：

```php
// Auth.php L81-L84
$language = Minz_Translate::getLanguage(null, Minz_Request::getPreferredLanguages(), FreshRSS_Context::systemConf()->language);
Minz_Translate::init($language);
$login_ok = FreshRSS_user_Controller::createUser($current_user, $email, '', [
    'language' => $language,
]);
```

**语言优先级**（此时尚无用户配置，`$user = null`）：

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1 | 浏览器 `Accept-Language` | 依次匹配可用语言 |
| 2 | 系统配置 `systemConf()->language` | 浏览器偏好无匹配时 |
| 3 | `'en'` | 最终兜底 |

**关键点**：
- 此时 `Minz_Translate::init($language)` 在 `FreshRSS::initI18n()` **之前**被调用，因为 `initAuth()` 先于 `initI18n()` 执行
- 建号成功后，`giveAccess()` → `initUser()` 加载新用户配置，之后 `initI18n()` 会按主流程再走一遍，此时 `$userLanguage` 已有值，与刚设的相同
- [createUser()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/userController.php#L370-L372) 还会对 language 做二次校验：若浏览器返回了不存在的语言代码，会被强制修正为 `'en'`

### 3.3 注册页（registerAction）

**触发条件**：未登录用户访问注册页面，系统未达到最大注册数。

**代码路径**：

1. [authController::registerAction()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/authController.php#L250-L263)
2. 注册页面**展示**时语言由 [FreshRSS::initI18n()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/FreshRSS.php#L90-L103) 主流程决定（此时 `login_ok = false`，Session 中的 `CURRENT_USER` 是 `default_user`）

**注册页展示时的语言**：

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1 | 默认用户的 `userConf()->language` | `initUser(default_user)` 已完成，`hasUserConf() = true` |
| 2 | 浏览器 `Accept-Language` | 优先级 1 无效时 |
| 3 | 系统配置 `systemConf()->language` | 优先级 2 无匹配时 |
| 4 | `'en'` | 最终兜底 |

**注册页传递给模板的偏好语言**：

```php
// authController.php L261
$this->view->preferred_language = Minz_Translate::getLanguage(null, Minz_Request::getPreferredLanguages(), FreshRSS_Context::systemConf()->language);
```

这是**独立于页面展示语言**的，用于模板中预选语言下拉框的值。其优先级：

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1 | 浏览器 `Accept-Language` | `$user = null` |
| 2 | 系统配置 `systemConf()->language` | 浏览器偏好无匹配时 |
| 3 | `'en'` | 最终兜底 |

**提交注册时的语言**：

```php
// userController.php L488-L489
$ok = self::createUser($new_user_name, $email, $passwordPlain, [
    'language' => Minz_Request::paramString('new_user_language') ?: FreshRSS_Context::userConf()->language,
    ...
]);
```

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1 | POST 参数 `new_user_language` | 注册表单提交的语言 |
| 2 | 当前 `userConf()->language`（默认用户的语言） | POST 参数为空时的兜底 |

`createUser()` 中会再次验证语言是否存在，不存在则强制设为 `'en'`。

### 3.4 API 请求

各 API 端点的语言初始化方式不同，需要分开看。

#### 3.4.1 API 信息页（p/api/index.php）

**触发条件**：直接访问 `/p/api/` 路径。

```php
// p/api/index.php L15
Minz_Translate::init(Minz_Translate::getLanguage(null, Minz_Request::getPreferredLanguages(), null));
```

**语言优先级**：

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1 | 浏览器 `Accept-Language` | `$user = null` |
| 2 | `'en'` | `$default = null`，直接到 `DEFAULT_LANGUAGE` |

**关键点**：`$default = null`，没有系统配置参与。信息页是静态 HTML，不走 FreshRSS 主初始化流程，也没有用户上下文。HTML 中硬编码了 `lang="en-GB"`。

#### 3.4.2 GReader API（p/api/greader.php）

**触发条件**：移动客户端通过 Google Reader 兼容 API 访问。

```php
// greader.php L1121-L1127
if (FreshRSS_Context::hasUserConf()) {
    Minz_Translate::init(FreshRSS_Context::userConf()->language);
} else {
    Minz_Translate::init();
}
```

**语言优先级**：

| 场景 | 优先级 | 来源 | 说明 |
|------|--------|------|------|
| 认证成功（accounts 之后的路径） | 1 | `userConf()->language` | 直接用用户配置语言初始化 |
| 认证成功但语言无效 | 1→2 | `userConf()->language` → 回退到 `en/` | Translate::init 接受任意字符串，loadLang 时目录不存在则自动回退 en |
| 未认证（accounts 路径） | 1 | `''`（空字符串） | `Minz_Translate::init()` 默认参数，loadLang 回退到 en |

**关键点**：
- GReader API **不走** `getLanguage()` 三级优先级逻辑，直接用 `userConf()->language` 初始化
- 若用户配置的语言无效，`Minz_Translate::init('invalid_lang')` 不会报错，`loadLang()` 找不到目录会自动回退到 `en/`
- 未认证时 `Minz_Translate::init()`（空字符串），等效于加载英文翻译

#### 3.4.3 Fever API（p/api/fever.php）

**触发条件**：移动客户端通过 Fever 兼容 API 访问。

```php
// fever.php L178-L186
FreshRSS_Context::initUser($username);
if ($feverKey === FreshRSS_Context::userConf()->feverKey && FreshRSS_Context::userConf()->enabled) {
    Minz_Translate::init(FreshRSS_Context::userConf()->language);
    return true;
} else {
    Minz_Translate::init();
}
```

**语言优先级**：

| 场景 | 优先级 | 来源 | 说明 |
|------|--------|------|------|
| 认证成功 | 1 | `userConf()->language` | 同 GReader API |
| 认证失败 | 1 | `''`（空字符串） | 回退到英文 |

#### 3.4.4 共享查询 API（p/api/query.php）

**触发条件**：通过 token 访问用户的共享查询结果。

```php
// query.php L41-L62
FreshRSS_Context::initUser($user);
Minz_Translate::init(FreshRSS_Context::userConf()->language);
```

**语言优先级**：直接使用目标用户的 `userConf()->language`，与 GReader API 认证成功后的行为一致。

#### 3.4.5 扩展 API（p/api/misc.php）

**触发条件**：通过 `/api/misc.php/{ExtensionName}/` 路径调用扩展。

```php
// misc.php L64
Minz_Translate::init();
```

**语言优先级**：始终为英文（空字符串初始化，回退到 `en/`），无用户上下文参与。

### 3.5 安装向导（install.php）

**触发条件**：系统尚未完成安装（`data/config.php` 不存在）。

```php
// install.php L34-L47
function initTranslate(): void {
    Minz_Translate::init();
    $available_languages = Minz_Translate::availableLanguages();
    if (Minz_Session::paramString('language') == '') {
        Minz_Session::_param('language', get_best_language());
    }
    if (!in_array(Minz_Session::paramString('language'), $available_languages, true)) {
        Minz_Session::_param('language', Minz_Translate::DEFAULT_LANGUAGE);
    }
    Minz_Translate::reset(Minz_Session::paramString('language'));
}
```

**语言优先级**：

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1 | Session 中的 `language`（来自上次安装步骤选择） | 安装步骤间保持语言一致 |
| 2 | `get_best_language()`：取 `Accept-Language` 前 2 位 | Session 为空时 |
| 3 | `'en'` | 浏览器语言不在可用列表中时 |

### 3.6 Form 登录成功后

**触发条件**：表单登录认证通过。

```php
// authController.php L169
Minz_Translate::init(FreshRSS_Context::userConf()->language);
```

直接用用户配置语言初始化，与 GReader API 行为一致。

### 3.7 各路径语言优先级对比总表

| 入口 | `$user` | `$preferred`（浏览器） | `$default`（系统配置） | 特殊逻辑 |
|------|---------|----------------------|---------------------|---------|
| 主流程（已登录 / 匿名） | `userConf()->language` | ✅ 无效时回退 | ✅ 无匹配时回退 | `getLanguage()` 三级优先级 |
| HTTP 认证自动建号 | `null` | ✅ 优先 | ✅ 次选 | 建号前提前算语言，写入新用户配置 |
| 注册页展示 | `default_user` 的 language | ✅ 无效时回退 | ✅ 无匹配时回退 | 模板偏好语言单独算（`$user=null`） |
| 注册提交 | POST `new_user_language` | ❌ | ❌ | 回退到 `default_user` 的语言 |
| API 信息页 | `null` | ✅ 优先 | `null`（无系统配置） | 硬编码 `lang="en-GB"` |
| GReader API（已认证） | 直接 `userConf()->language` | ❌ | ❌ | 不走 `getLanguage()`，无效语言回退到 en 翻译 |
| GReader API（未认证） | `''` | ❌ | ❌ | `init()` 空参数，回退到 en |
| Fever API | 直接 `userConf()->language` | ❌ | ❌ | 同 GReader |
| 共享查询 API | 直接 `userConf()->language` | ❌ | ❌ | 同 GReader |
| 扩展 API | `''` | ❌ | ❌ | 始终英文 |
| 安装向导 | Session → 浏览器 → `'en'` | ✅ 取前 2 位 | ❌ | 自有逻辑，不在 `getLanguage()` 体系内 |

---

## 4. 主题选择机制

### 4.1 主题优先级

主题选择比语言简单，**没有浏览器偏好或系统级配置参与**，仅由用户配置决定：

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1 | `userConf()->theme` | 用户在「显示配置」页面选择的主题名 |
| 2 | `FreshRSS_Themes::$defaultTheme` = `'Origine'` | 用户配置为空或无效时的硬编码默认值 |
| 3 | 第一个可用主题 | `Origine` 也无效时，取 `p/themes/` 下第一个有效主题 |
| 4 | `false` | 完全没有可用主题（极端情况） |

回退逻辑见 [FreshRSS_Themes::load()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Models/Themes.php#L81-L101)：

```php
if (empty($infos)) {
    if ($theme_id !== self::$defaultTheme) {   // 回退到 Origine
        return self::load(self::$defaultTheme);
    }
    $themes_list = self::getList();
    if (!empty($themes_list)) {
        if ($theme_id !== $themes_list[0]) {   // 回退到第一个主题
            return self::load($themes_list[0]);
        }
    }
    return false;
}
```

### 4.2 主题验证

[FreshRSS_Themes::exists()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Models/Themes.php#L18-L22) 检查：
- 主题 ID 不含 `..`、`/` 或目录分隔符（防路径穿越）
- `p/themes/{theme_id}/metadata.json` 文件存在

### 4.3 内置主题列表

| 主题名 | 类型 |
|--------|------|
| `Origine` | 默认亮色主题 |
| `Origine-compact` | 紧凑版 Origine |
| `Ansum` | 亮色 |
| `Mapco` | 亮色 |
| `Flat` | 扁平化 |
| `Nord` | Nord 色系 |
| `Swage` | 现代风格 |
| `Pafat` | 亮色 |
| `Dark` | 暗色 |
| `Dark-pink` | 暗色粉色 |
| `Alternative-Dark` | 备选暗色 |
| `base-theme` | 开发用基础主题 |

### 4.4 主题配置文件 metadata.json

每个主题目录下必须有 `metadata.json`，结构示例（Origine）：

```json
{
    "name": "Origine",
    "author": "Marien Fressinaud",
    "description": "Le thème par défaut pour FreshRSS",
    "version": 0.2,
    "files": ["_frss.css", "origine.css"],
    "theme-color": {"dark": "#1f1f1f", "light": "#f0f0f0"}
}
```

- `files`：CSS/JS 文件列表，以 `_` 前缀表示引用 `base-theme` 目录下的文件（如 `_frss.css` → `base-theme/frss.css`）
- `theme-color`：可为字符串（单色）或对象（dark/light/default 分别设置）

### 4.5 Dark Mode

`darkMode` 是独立于主题的配置项，取值为 `'auto'` | `'no'` | 其他自定义值：

| 值 | 效果 |
|----|------|
| `'auto'` | 跟随系统暗色模式偏好 |
| `'no'` | 不启用暗色模式 |
| 其他值 | 自定义暗色模式标识 |

在 layout.phtml 中，`darkMode` 会作为 CSS class 附加到 `<html>` 元素上：

```php
if (FreshRSS_Context::userConf()->darkMode !== 'no') {
    $class[] = 'darkMode_' . FreshRSS_Context::userConf()->darkMode;
}
```

### 4.6 主题切换生效路径

用户在「显示配置」页面提交主题设置后：

```
1. Minz_Request::paramString('theme')    → 从 POST 取得新主题名
2. FreshRSS_Themes::exists($theme)       → 验证主题存在
3. userConf()->theme = $theme            → 写入用户配置对象
4. userConf()->save()                    → 持久化
5. 页面重定向 → 下次请求时 loadStylesAndScripts() 从 userConf()->theme 读取
```

---

## 5. 无效配置值的表现与一致性分析

### 5.1 无效语言值

**场景**：用户配置文件中 `language` 被设为不存在的值（如 `'xx'`），无论通过何种方式写入。

#### 翻译加载层

| 调用方式 | 代码 | 结果 |
|---------|------|------|
| 主流程 `initI18n()` | `getLanguage('xx', ...)` → `Minz_Translate::exists('xx')` = false → 直接返回 `'en'` | ✅ 翻译显示英文 |
| API 直接初始化 `Minz_Translate::init('xx')` | `loadLang()` 找不到 `app/i18n/xx/` → 回退到 `app/i18n/en/` | ✅ 翻译显示英文 |
| `Minz_Translate::init()`（空字符串） | `loadLang()` 跳过，回退到 `app/i18n/en/` | ✅ 翻译显示英文 |

**结论**：翻译加载在所有路径下对无效语言值的表现一致——回退到英文翻译。

#### HTML lang 属性层

[layout.phtml L28](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/layout/layout.phtml#L28)：

```php
<html lang="<?= FreshRSS_Context::userConf()->language ?>"
      xml:lang="<?= FreshRSS_Context::userConf()->language ?>">
```

**问题**：`lang` 属性直接输出 `userConf()->language` 的原始值，**不做任何回退处理**。

| 实际翻译语言 | `lang` 属性值 | 是否一致 |
|------------|-------------|---------|
| 英文（回退） | `xx`（无效值） | ❌ 不一致 |

**示例**：若 `userConf()->language = 'xx'`：
- 主流程中 `getLanguage('xx', ...)` 返回 `'en'`，翻译显示英文
- 但 `layout.phtml` 仍输出 `<html lang="xx" xml:lang="xx">`
- 浏览器/屏幕阅读器会认为页面语言是 `xx`，可能影响语音合成和字体回退

**API 信息页特例**：`p/api/index.php` 中 `lang="en-GB"` 是硬编码的，与 `getLanguage()` 的结果无关。

#### RTL 检测层

RTL 检测依赖翻译词条 `_t('gen.dir')`。无效语言回退到 `en` 翻译后，`gen.dir` 返回 `'ltr'`，所以 RTL 检测始终正确。

#### 结论：翻译加载一致，lang 属性不一致

**翻译加载**：所有路径对无效语言的表现一致——回退到英文。
**lang 属性**：layout.phtml 直接输出 `userConf()->language` 的原始无效值，与实际翻译语言不一致。这是一个潜在的无障碍访问问题。

### 5.2 无效主题值

**场景**：用户配置中 `theme` 被设为不存在的主题名（如 `'NonExistent'`），无论通过何种方式写入。

#### 样式加载层

[FreshRSS::loadStylesAndScripts()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/FreshRSS.php#L110-L149) 调用 `FreshRSS_Themes::load(userConf()->theme)`：

```
FreshRSS_Themes::load('NonExistent')
  → get_infos('NonExistent') 返回 false/空
  → 回退到 FreshRSS_Themes::load('Origine')
  → 返回 Origine 的元数据和文件列表
  → 加载 Origine 的 CSS/JS
```

✅ 页面样式回退到 Origine，视觉表现正常。

#### theme-color meta 标签层

回退到 Origine 后，`theme-color` 使用 Origine 的 `{"dark": "#1f1f1f", "light": "#f0f0f0"}`。✅ 一致。

#### HTML class 属性层

[layout.phtml L16](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/layout/layout.phtml#L16)：

```php
$class[] = 'theme_' . FreshRSS_Context::userConf()->theme;
```

**问题**：class 直接输出 `userConf()->theme` 的原始值，**不做回退**。

| 实际加载的主题 CSS | HTML class | 是否一致 |
|------------------|-----------|---------|
| Origine（回退） | `theme_NonExistent` | ❌ 不一致 |

**影响**：
- CSS 中以 `.theme_Origine` 为选择器的样式不会生效（class 名不匹配）
- 页面可能缺少主题特定的 CSS 覆盖，导致样式不完整
- 依赖 `theme_*` class 的自定义 CSS 或 JavaScript 也会失效

#### 结论：样式部分回退但不完整

**CSS 文件加载**：回退到 Origine 的文件列表，基础样式正常。
**CSS class 选择器**：`theme_NonExistent` 与实际加载的 Origine CSS 不匹配，主题特有样式可能丢失。
**根因**：`loadStylesAndScripts()` 和 `layout.phtml` 对主题回退的处理不一致——前者使用了回退后的主题，后者仍使用配置中的原始值。

### 5.3 无效 darkMode 值

`darkMode` 没有验证和回退逻辑。若设为任意字符串（如 `'foobar'`），layout.phtml 会输出 `darkMode_foobar` 作为 class。没有对应 CSS 规则，效果等同于 `darkMode_auto` 的缺失——即不触发任何暗色模式 CSS。

### 5.4 无效值场景汇总

| 配置项 | 翻译/样式是否回退 | DOM 属性是否回退 | 两者是否一致 |
|--------|-----------------|----------------|------------|
| `language`（无效值） | ✅ 回退到 `en` 翻译 | ❌ `lang` 属性仍为无效值 | ❌ 不一致 |
| `theme`（无效值） | ✅ CSS 文件回退到 Origine | ❌ class 仍为 `theme_{无效值}` | ❌ 不一致 |
| `darkMode`（无效值） | N/A（无回退逻辑） | ❌ class 为 `darkMode_{无效值}` | N/A（无验证机制） |

### 5.5 无效值如何产生

正常 Web UI 操作不会写入无效值，因为：

- **语言**：[configureController::displayAction()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/configureController.php#L48-L51) 在保存前调用 `Minz_Translate::exists($language)` 验证
- **主题**：[configureController::displayAction()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/configureController.php#L53-L56) 在保存前调用 `FreshRSS_Themes::exists($theme)` 验证
- **建号**：[createUser()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/userController.php#L370-L372) 验证语言存在性

但以下场景可能产生无效值：
- 直接编辑 `data/users/{username}/config.php` 文件
- 主题被删除后，用户配置中仍保留该主题名
- 翻译目录被删除后，用户配置中仍保留该语言代码
- `data/config-user.custom.php` 中写入了无效值
- HTTP 认证自动建号时浏览器返回了不存在的语言代码（被 `createUser()` 修正为 `'en'`，不会产生）

---

## 6. 页面渲染生效路径

### 6.1 完整请求生命周期

```
HTTP 请求
  ↓
p/i/index.php（入口）
  ↓
FreshRSS->init()                         ← 初始化系统配置、用户配置、i18n
  ├─ FreshRSS_Context::initSystem()      ← 加载 data/config.php
  ├─ FreshRSS_Context::initUser()        ← 加载 data/users/{user}/config.php
  └─ FreshRSS::initI18n()                ← 确定语言，加载翻译
  ↓
Minz_FrontController->run()
  ↓
Controller->firstAction()
Controller->{action}Action()             ← 业务逻辑
  ↓
layout.phtml（布局渲染）
  ├─ FreshRSS::preLayout()               ← 加载样式和脚本
  │   └─ FreshRSS::loadStylesAndScripts() ← 根据 userConf()->theme 加载 CSS/JS
  ├─ <html lang="userConf()->language">  ← 语言写入 HTML lang 属性
  ├─ class="theme_{userConf()->theme}"   ← 主题名写入 HTML class
  ├─ class="darkMode_{userConf()->darkMode}" ← 暗色模式 class
  ├─ FreshRSS_View::metaThemeColor()     ← 主题色 meta 标签
  ├─ FreshRSS_View::headStyle()          ← 输出 <link> 标签
  └─ FreshRSS_View::headScript()         ← 输出 <script> 标签
```

### 6.2 主题生效的关键代码

[FreshRSS::loadStylesAndScripts()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/FreshRSS.php#L110-L149)：

1. 调用 `FreshRSS_Themes::load(FreshRSS_Context::userConf()->theme)` 获取主题元数据
2. 按逆序遍历 `theme['files']`，通过 `FreshRSS_View::prependStyle()` 注入 CSS
3. `_` 前缀文件从 `base-theme` 目录加载，其余从主题自身目录加载
4. RTL 语言时自动替换为 `.rtl.css` 版本
5. 附加 `theme-color` meta 标签

[layout.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/layout/layout.phtml#L16-L19) 中将主题和暗色模式写入 HTML class：

```php
$class[] = 'theme_' . FreshRSS_Context::userConf()->theme;          // 如 theme_Origine
if (FreshRSS_Context::userConf()->darkMode !== 'no') {
    $class[] = 'darkMode_' . FreshRSS_Context::userConf()->darkMode; // 如 darkMode_auto
}
```

### 6.3 语言生效的关键代码

[layout.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/layout/layout.phtml#L28) 中语言写入 HTML 属性：

```php
<html lang="<?= FreshRSS_Context::userConf()->language ?>"
      xml:lang="<?= FreshRSS_Context::userConf()->language ?>">
```

翻译文本通过 `_t('key.subkey')` 函数在所有 .phtml 视图模板中使用，该函数是 `Minz_Translate::t()` 的别名。

### 6.4 RTL 支持

语言为 RTL（如 `he` 希伯来语）时，翻译词条 `gen.dir` 返回 `'rtl'`，layout.phtml 会：

- 在 `<html>` 添加 `dir="rtl"` 属性
- 在 HTML class 中添加 `rtl`
- CSS 文件自动替换为 `.rtl.css` 版本

---

## 7. 配置项速查表

| 配置项 | 用户配置键 | 默认值 | 影响范围 |
|--------|-----------|--------|----------|
| 语言 | `language` | `'en'` | HTML lang、翻译文本、RTL 检测 |
| 主题 | `theme` | `'Origine'` | CSS class `theme_*`、加载的 CSS/JS 文件 |
| 暗色模式 | `darkMode` | `'auto'` | CSS class `darkMode_*` |
| 内容宽度 | `content_width` | `'thin'` | CSS class |
| 时区 | `timezone` | `''`（服务器默认） | 日期时间显示 |

---

## 8. 关键源文件索引

| 文件 | 职责 |
|------|------|
| [config-user.default.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/config-user.default.php) | 用户配置默认值定义 |
| [config.default.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/config.default.php) | 系统配置默认值定义（含系统级 language） |
| [Minz/Configuration.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/lib/Minz/Configuration.php) | 配置加载与合并的基类 |
| [UserConfiguration.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Models/UserConfiguration.php) | 用户配置子类 |
| [Context.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Models/Context.php) | 运行时上下文，持有 systemConf 和 userConf |
| [FreshRSS.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/FreshRSS.php) | 前端控制器，初始化 i18n 和加载样式/脚本 |
| [Translate.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/lib/Minz/Translate.php) | 翻译引擎，语言选择与翻译加载 |
| [Themes.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Models/Themes.php) | 主题管理，加载与回退 |
| [Minz/View.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/lib/Minz/View.php) | 视图基类，管理 styles/scripts/themeColors |
| [layout.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/layout/layout.phtml) | HTML 布局模板，主题和语言写入 DOM |
| [configureController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/configureController.php) | 显示配置页控制器，保存主题/语言/暗色模式 |
| [Auth.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Models/Auth.php) | 认证系统，含 HTTP 认证自动建号的语言处理 |
| [authController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/authController.php) | 登录/注册控制器 |
| [userController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/userController.php) | 用户管理控制器，含 createUser 建号逻辑 |
| [p/api/index.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/p/api/index.php) | API 信息页 |
| [p/api/greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/p/api/greader.php) | Google Reader 兼容 API |
| [p/api/fever.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/p/api/fever.php) | Fever 兼容 API |
| [p/api/query.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/p/api/query.php) | 共享查询 API |
| [p/api/misc.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/p/api/misc.php) | 扩展 API |
| [install.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/install.php) | 安装向导，自有语言选择逻辑 |
