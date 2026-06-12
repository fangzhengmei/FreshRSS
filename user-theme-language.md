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
| `base-theme` | 开发用基础主题（`name` 为空，不出现在选择列表中） |

**注意**：[FreshRSS_Themes::get()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Models/Themes.php#L25-L36) 通过 `trim($theme['name']) !== ''` 过滤掉了 `base-theme`（其 `name` 为空字符串），因此它不会出现在主题选择列表中，但 `load()` 仍可通过 `get_infos()` 直接加载。

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

**此问题在所有三种 HTML 输出布局中均存在**（layout.phtml、simple.phtml、contentSelectorPreview.phtml 均直接输出 `userConf()->language`），见第 6 节。

**API 信息页特例**：`p/api/index.php` 中 `lang="en-GB"` 是硬编码的，与 `getLanguage()` 的结果无关。

#### RTL 检测层

RTL 检测依赖翻译词条 `_t('gen.dir')`。无效语言回退到 `en` 翻译后，`gen.dir` 返回 `'ltr'`，所以 RTL 检测始终正确。

#### 结论：翻译加载一致，lang 属性不一致

**翻译加载**：所有路径对无效语言的表现一致——回退到英文。
**lang 属性**：三个 HTML 布局直接输出 `userConf()->language` 的原始无效值，与实际翻译语言不一致。这是一个潜在的无障碍访问问题。

### 5.2 无效主题值

**场景**：用户配置中 `theme` 被设为不存在的主题名（如 `'NonExistent'`），无论通过何种方式写入（例如主题被删除后配置仍保留旧值）。

#### 已证实的现象

以下结论通过阅读源码直接验证，确定性高。

**1. CSS 文件加载回退到 Origine ✅ 已证实**

[FreshRSS::loadStylesAndScripts()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/FreshRSS.php#L110-L149) 调用 `FreshRSS_Themes::load(userConf()->theme)`：

```
FreshRSS_Themes::load('NonExistent')
  → get_infos('NonExistent') 返回 false
  → 回退到 self::load('Origine')
  → 返回 Origine 的元数据（id='Origine', files=['_frss.css','origine.css'], theme-color={...}）
  → loadStylesAndScripts() 用 $theme['id']（='Origine'）拼接 CSS/JS 路径
  → 页面正常加载 Origine 的样式和脚本
```

**2. theme-color meta 标签回退到 Origine ✅ 已证实**

回退后 `FreshRSS_View::appendThemeColors($theme['theme-color'])` 拿到 Origine 的 `{"dark": "#1f1f1f", "light": "#f0f0f0"}`，`metaThemeColor()` 输出正确。

**3. HTML class 输出无效值，与实际加载主题不一致 ✅ 已证实**

三个 HTML 布局中：

- [layout.phtml L16](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/layout/layout.phtml#L16)：`$class[] = 'theme_' . FreshRSS_Context::userConf()->theme;` → 输出 `theme_NonExistent`
- [simple.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/layout/simple.phtml)：**无 `theme_*` class**（即便主题有效也没有）
- [contentSelectorPreview.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/views/feed/contentSelectorPreview.phtml)：**无 `theme_*` class**（同 simple）

**4. 内置主题 CSS 完全不使用 `.theme_*` 选择器 ✅ 已证实**

对全部 `p/themes/` 下的 `.css` 文件搜索 `.theme_[A-Za-z]` 选择器，**0 条匹配**。所有内置主题（Origine、Dark、Flat、Swage、Nord 等）的 CSS 中没有任何 `.theme_*` 前缀的选择器。`theme_*` class 纯粹是 HTML 元数据标识，**不影响任何内置主题的视觉表现**。

**5. darkMode_auto 选择器不受影响 ✅ 已证实**

`darkMode_auto` 是独立的 CSS class，作用于 `<html>` 元素的 `darkMode_*` 前缀（见第 4.5 节），与 `theme_*` class 是两套机制。

**普通样式（LTR）中的分布**：
- 搜索全部 `p/themes/**/*.css`，`:root.darkMode_auto` 选择器**仅出现在 Origine 主题**
- [origine.css](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/p/themes/Origine/origine.css) **共 12 行**，集中在 [L1239](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/p/themes/Origine/origine.css#L1239)、[L1327-L1358](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/p/themes/Origine/origine.css#L1327-L1358)
- 这 12 行对应 8 条 CSS 规则（其中 L1335-L1337 是同一条规则的 3 个逗号分隔选择器，L1353-L1354 是同一条规则的 2 个逗号分隔选择器）
- `base-theme/frss.css`、Dark、Flat、Swage、Nord 等**所有其他内置主题的普通 CSS 中 `darkMode_auto` 出现次数均为 0**
- 覆盖的 UI 元素：`:root.darkMode_auto` 自身的 CSS 变量定义、`.nav_menu .btn`（含 hover/active/dropdown-target 共 4 种状态）、`.nav_menu.dropdown-menu`、`.nav_menu`、`.header`、`.btn.active .icon` + `.btn:active .icon`、`.spinner`，共 6 类元素

**RTL 样式中的分布**：
- [origine.rtl.css](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/p/themes/Origine/origine.rtl.css) **共 12 行**，集中在 [L1239](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/p/themes/Origine/origine.rtl.css#L1239)、[L1327-L1358](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/p/themes/Origine/origine.rtl.css#L1327-L1358)
- 行号、选择器、规则结构与 origine.css **逐行一一对应**，同样 12 行 / 8 条规则
- 所有其他内置主题的 `.rtl.css` 中 `darkMode_auto` 出现次数均为 0
- 由于 RTL 替换发生在 CSS 文件名层面（`origine.css` → `origine.rtl.css`，见 [FreshRSS.php L132-L135](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/FreshRSS.php#L132-L135)），RTL 页面的 `darkMode_auto` 选择器与 LTR 页面完全等价，只是应用在 RTL 布局下

**合计**：全部内置主题的全部 CSS 文件中 `darkMode_auto` 共 24 行（普通 12 + RTL 12），且全部位于 Origine 主题。

**无效主题值时的表现**：
- 无效主题回退到 Origine 后，`origine.css` / `origine.rtl.css` 被正常加载，`darkMode_auto` 相关选择器**全部生效**
- `darkMode_auto` 不依赖任何 `.theme_*` 选择器前缀，与回退机制完全独立
- 但若用户的 `darkMode` 配置为 `'no'`，则 `<html>` 不会有 `darkMode_*` class，这些选择器也不会匹配（这是预期行为，与主题有效性无关）

**6. 设置页显示 `theme_not_available` 提示 ✅ 已证实**

[display.phtml L86-L93](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/views/configure/display.phtml#L86-L93)：当遍历所有可用主题后仍找不到当前配置的主题时（`$themeAvailable = false`），显示红色错误提示 `theme_not_available`，告知用户选择其他主题。这给了用户修正的入口。

**7. load() 返回 false 时完全不加载主题 CSS ✅ 已证实**

若 `load()` 连 Origine 和第一个可用主题都无法加载（极端情况），返回 `false`，则 `loadStylesAndScripts()` 中 `if (is_array($theme))` 不成立，**不会加载任何主题 CSS/JS**，仅加载 `main.js` 和 `extra.js`。页面将完全没有样式。

#### 仍需谨慎判断的影响

以下推断基于代码分析，但涉及代码库外部或未来因素，无法在当前代码中严格证实或证伪。

**1. 第三方自定义主题可能依赖 `.theme_*` 选择器 ⚠️**

若第三方主题在其 CSS 中使用 `.theme_{id}` 选择器（如 `.theme_MyCustomTheme .header { ... }`），则无效主题时 class 不匹配会导致这些样式规则不生效。但当前代码库中无法验证这一点——已确认的是所有内置主题均不使用该选择器，FreshRSS 的主题开发文档也未要求使用此 class。

**2. 浏览器扩展 / 用户脚本可能读取 `theme_*` class ⚠️**

用户脚本可能通过 `document.documentElement.classList.contains('theme_Origine')` 判断主题并注入自定义 CSS。无效主题时会误判。这属于代码库外逻辑，无法验证。

**3. `simple.phtml` 和 `contentSelectorPreview.phtml` 本身就缺少 `theme_*` class ⚠️**

这是一个独立于无效主题值的问题：即便主题有效，简版布局和预览页也不输出 `theme_*` class。若第三方主题依赖此 class，在这两个页面同样会样式缺失。该问题**与无效主题值无关**，但影响面重合。

**4. 未来版本可能引入基于 `theme_*` 的样式架构 ⚠️**

若后续版本改为通过 `.theme_*` 选择器实现主题差异化（类似当前 `:root.darkMode_auto` 的做法），则 class 不一致将演变为实际 Bug。但这是对代码演进的预测，非当前事实。

**5. `theme_*` class 用于辅助技术的可能性 ⚠️**

辅助技术一般不读取非标准 class，HTML 规范也未将此类 class 用于无障碍目的。但不能排除某些定制工具使用它。影响极小但无法绝对排除。

#### 无效主题值表现总结

| 方面 | 状态 | 说明 |
|------|------|------|
| CSS 文件加载 | ✅ 已证实回退到 Origine | 视觉正常 |
| `theme-color` meta | ✅ 已证实回退到 Origine | 浏览器 UI 色正常 |
| `darkMode_auto` 暗色模式 | ✅ 已证实正常工作 | 使用 `:root.darkMode_auto` 选择器，不受 theme class 影响 |
| 内置主题视觉表现 | ✅ 已证实无影响 | 所有内置主题 CSS 不使用 `.theme_*` 选择器 |
| 设置页提示 | ✅ 已证实显示错误提示 | 用户可修正 |
| HTML `theme_*` class | ✅ 已证实为无效值 | 与实际加载的 Origine 不一致，但当前不影响样式 |
| 第三方自定义主题 CSS | ⚠️ 需谨慎判断 | 可能受影响（取决于是否使用 `.theme_*` 选择器） |
| 简版/预览布局的 theme class 缺失 | ⚠️ 需谨慎判断 | 与无效主题值无关，但第三方主题同样受此影响 |
| load() 返回 false | ✅ 已证实无主题 CSS/JS | 极端情况，`main.js` 一定保留，`extra.js` 按 controller 条件追加 |

### 5.3 无效 darkMode 值

`darkMode` 没有验证和回退逻辑。若设为任意字符串（如 `'foobar'`），三个 HTML 布局均输出 `darkMode_foobar` 作为 class。Origine 主题中只有 `:root.darkMode_auto` 选择器，`darkMode_foobar` 不匹配任何规则，效果等同于不触发暗色模式 CSS。这不是回退，而是**静默失效**。

### 5.4 无效值如何产生

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

### 6.1 渲染管线总览

所有 HTML 页面请求经历相同的初始化阶段，分歧点在 Controller action 中对 `_layout()` 的调用：

```
HTTP 请求 → p/i/index.php
  ↓
FreshRSS::init()                     ← 所有页面共用
  ├─ FreshRSS_Context::initSystem()  ← 加载系统配置
  ├─ FreshRSS_Auth::init()           ← 认证（可能触发 HTTP 认证自动建号）
  ├─ FreshRSS_Context::initUser()    ← 加载当前用户配置（语言、主题、darkMode）
  └─ FreshRSS::initI18n()            ← 确定语言，加载翻译
  ↓
Minz_Dispatcher::run()
  ├─ Controller->firstAction()
  ├─ Controller->{action}Action()    ← 业务逻辑 + _layout() 决定布局
  └─ Controller->lastAction()
  ↓
Minz_View::build()                   ← 根据布局文件名选择渲染路径
  │
  ├─ layout_filename !== '' → buildLayout()  ← 有布局
  │   ├─ 'layout'  → layout.phtml           ← 主布局
  │   └─ 'simple'  → simple.phtml           ← 简版布局
  │
  └─ layout_filename === '' → render()       ← 无布局（裸输出视图模板）
      ├─ 大部分：HTML 片段（Ajax 响应）
      └─ 特例：contentSelectorPreview.phtml（自包含 HTML 文档）
```

### 6.2 主布局（layout.phtml）— 默认路径

**适用范围**：绝大多数页面，包括：
- 主阅读页（index/index）
- 所有设置页（configure/display、configure/reading、configure/archiving 等）
- 订阅管理页（subscription/feed、subscription/add 等）
- 用户资料页（user/profile，邮箱已验证时）
- 标签管理页
- 扩展管理页
- 日志页
- 关于页

**触发条件**：Controller action 中**未调用** `_layout()`，或调用了 `$this->view->_layout('layout')`。

**渲染流程**：

```
[ layout.phtml ]
  │
  ├─ FreshRSS::preLayout()
  │   └─ loadStylesAndScripts()
  │       └─ FreshRSS_Themes::load(userConf()->theme) → 加载回退后的主题 CSS/JS
  │       └─ FreshRSS_View::appendThemeColors()       → 主题色
  │
  ├─ <html> 元素属性与 class：
  │   ├─ lang="userConf()->language"                   ← 直接取配置值
  │   ├─ xml:lang="userConf()->language"               ← 直接取配置值
  │   ├─ dir="rtl"                                     ← RTL 语言时
  │   ├─ class="controller_{name}"                     ← 当前 controller 名
  │   ├─ class="theme_{userConf()->theme}"             ← ⚠️ 直接取配置值，不做回退
  │   ├─ class="darkMode_{userConf()->darkMode}"       ← darkMode !== 'no' 时
  │   ├─ class="rtl"                                   ← RTL 语言时
  │   └─ class="logged_in"                             ← 已登录时
  │
  ├─ <head> 内容：
  │   ├─ FreshRSS_View::metaThemeColor()               ← 主题色 meta（已回退）
  │   ├─ FreshRSS_View::headStyle()                    ← CSS <link>（已回退）
  │   ├─ renderHelper('javascript_vars')               ← JS 配置变量
  │   ├─ FreshRSS_View::headScript()                   ← JS <script>（含 main.js、extra.js）
  │   ├─ <link rel="manifest">、favicon、apple-touch-icon
  │   ├─ FreshRSS_View::headTitle()
  │   └─ RSS/OPML <link>、robots meta
  │
  ├─ <body class="{actionName}">
  │   ├─ partial('header')                             ← 顶部导航栏
  │   ├─ <div id="global">
  │   │   ├─ aside_feed / aside_configure / aside_subscription  ← 侧边栏
  │   │   └─ $this->render()                           ← 视图内容
  │   └─ <div id="notification">                       ← 通知提示
  │
  └─ </html>
```

**关键观察**：
- `theme_*` class 取 `userConf()->theme` **原始值**（可能无效），而 CSS 文件加载取 `load()` **回退后的值**
- `lang` 属性取 `userConf()->language` **原始值**（可能无效），而翻译文本取 `initI18n()` **回退后的值**
- 这是无效值时 DOM 属性与实际渲染不一致的根因

### 6.3 简版布局（simple.phtml）

**适用范围**：

| Controller | Action | 触发条件 | 代码位置 |
|-----------|--------|---------|---------|
| userController | `profileAction()` | 邮箱未验证时（`email_validation_token != ''`） | [userController.php L160](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/userController.php#L160) |
| userController | `validateEmailAction()` | 始终使用 simple 布局 | [userController.php L577](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/userController.php#L577) |

**触发条件**：Controller action 中调用 `$this->view->_layout('simple')`。

**渲染流程**：

```
[ simple.phtml ]
  │
  ├─ FreshRSS::preLayout()                            ← 与主布局完全相同
  │   └─ loadStylesAndScripts() → 同上
  │
  ├─ <html> 元素属性与 class：
  │   ├─ lang="userConf()->language"                   ← 与主布局相同
  │   ├─ xml:lang="userConf()->language"               ← 与主布局相同
  │   ├─ dir="rtl"                                     ← RTL 语言时
  │   ├─ class="rtl"                                   ← RTL 语言时（注意：拼接在字符串前部）
  │   ├─ class="darkMode_{userConf()->darkMode}"       ← 与主布局相同
  │   └─ ❌ 无 theme_* class                          ← 与主布局不同
  │
  ├─ <head> 内容：
  │   ├─ FreshRSS_View::metaThemeColor()               ← 与主布局相同
  │   ├─ FreshRSS_View::headStyle()                    ← 与主布局相同
  │   ├─ renderHelper('javascript_vars')               ← 与主布局相同
  │   ├─ FreshRSS_View::headScript()                   ← 与主布局相同
  │   ├─ <link rel="manifest">、favicon、apple-touch-icon
  │   ├─ FreshRSS_View::headTitle()
  │   └─ robots meta（始终 noindex,nofollow）
  │
  ├─ <body>
  │   ├─ 精简 header（仅 logo + 登录/登出按钮）
  │   ├─ <div class="app-layout app-layout-simple">
  │   │   └─ $this->render()                           ← 视图内容
  │   └─ <div id="notification">                       ← 与主布局相同
  │
  └─ </html>
```

**与主布局的差异（主题/语言维度）**：

| 项目 | layout.phtml | simple.phtml | 一致性 |
|------|-------------|--------------|--------|
| `lang` / `xml:lang` | `userConf()->language` | `userConf()->language` | ✅ |
| `darkMode_*` class | 有 | 有 | ✅ |
| `rtl` class + `dir` | 有 | 有 | ✅ |
| `theme_*` class | `theme_{userConf()->theme}` | **无** | ❌ |
| `preLayout()` / CSS | 有 | 有 | ✅ |
| `metaThemeColor()` | 有 | 有 | ✅ |
| `headScript()` | 完整 | 完整 | ✅ |
| 侧边栏 | 有 | 无 | —（功能差异） |
| 导航栏 | 完整 header.phtml | 仅 logo + 登录/登出 | —（功能差异） |

**`theme_*` class 缺失的影响**：已证实所有内置主题 CSS 不使用 `.theme_*` 选择器，因此对内置主题无视觉影响。若第三方主题依赖此 class，则简版布局中该主题的特有样式会缺失。

### 6.4 内容选择器预览页（contentSelectorPreview.phtml）

**适用范围**：

| Controller | Action | 触发条件 | 代码位置 |
|-----------|--------|---------|---------|
| feedController | `contentSelectorPreviewAction()` | 在 feed 配置页点击 CSS 路径旁的「预览」按钮 | [feedController.php L1274](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/feedController.php#L1274) |

**实现方式**：
1. Controller：`$this->view->_layout(null)` — 禁用布局
2. 视图模板自身包含完整 HTML 文档结构（`<html>`、`<head>`、`<body>`）

**渲染流程**：

```
[ contentSelectorPreview.phtml ] 自包含 HTML 文档
  │
  ├─ FreshRSS::preLayout()                            ← 与主布局完全相同
  │   └─ loadStylesAndScripts() → 同上
  │
  ├─ <html> 元素属性与 class：
  │   ├─ lang="userConf()->language"                   ← 与主布局相同
  │   ├─ xml:lang="userConf()->language"               ← 与主布局相同
  │   ├─ dir="rtl"                                     ← RTL 语言时
  │   ├─ class="preview_background"                    ← 预览专用 class
  │   ├─ class="rtl"                                   ← RTL 语言时
  │   ├─ class="darkMode_{userConf()->darkMode}"       ← 与主布局相同
  │   └─ ❌ 无 theme_* class                          ← 与主布局不同
  │
  ├─ <head> 内容：
  │   ├─ ❌ 无 metaThemeColor()                       ← 与主布局不同
  │   ├─ FreshRSS_View::headStyle()                    ← CSS <link>（已回退）
  │   └─ 仅 preview.js（无 headScript() 的 main.js/extra.js）
  │
  ├─ <body class="preview_background">
  │   ├─ 错误提示 / 预览内容（rendered/raw 切换）
  │   └─ 使用翻译函数 _t()
  │
  └─ </html>
```

**与主布局的差异（主题/语言维度）**：

| 项目 | layout.phtml | contentSelectorPreview.phtml | 一致性 |
|------|-------------|------------------------------|--------|
| `lang` / `xml:lang` | `userConf()->language` | `userConf()->language` | ✅ |
| `darkMode_*` class | 有 | 有 | ✅ |
| `rtl` class + `dir` | 有 | 有 | ✅ |
| `theme_*` class | `theme_{userConf()->theme}` | **无** | ❌ |
| `preLayout()` / CSS | 有 | 有 | ✅ |
| `metaThemeColor()` | 有 | **无** | ❌ |
| `headScript()` | main.js + extra.js | 仅 `preview.js` | ❌ |
| `controller_*` class | 有 | **无** | —（功能性） |

**预览页特殊性**：
- 这是一个嵌入在 iframe 中的预览页面，由 CSP 头限制了 `frame-ancestors: 'self'`
- 缺少 `metaThemeColor()` 和完整的 JS 脚本对预览功能本身没有影响
- 但缺少 `theme_*` class 与 simple.phtml 同理，若第三方主题依赖此 class 则样式缺失

### 6.5 设置页（configure/）

**适用范围**：所有 `/configure/*` 路由：

| Action | 视图模板 | 说明 |
|--------|---------|------|
| `display` | [configure/display.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/views/configure/display.phtml) | 显示配置（语言、主题、darkMode） |
| `reading` | configure/reading.phtml | 阅读配置 |
| `archiving` | configure/archiving.phtml | 归档配置 |
| `integration` | configure/integration.phtml | 分享/集成配置 |
| `shortcut` | configure/shortcut.phtml | 快捷键配置 |
| `queries` / `query` | configure/queries.phtml / query.phtml | 自定义查询配置 |
| `privacy` | configure/privacy.phtml | 隐私配置 |
| `system` | configure/system.phtml | 系统配置（管理员） |

**布局**：使用默认 `layout.phtml`，**无任何 configure action 调用 `_layout()`**。

**唯一特例**：`queryAction()` 在 `ajax=1` 时调用 `_layout(null)`，仅返回查询配置的 HTML 片段。

**渲染流程**：与第 6.2 节主布局完全相同。侧边栏使用 [aside_configure.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/layout/aside_configure.phtml)，提供设置页导航菜单。

**主题/语言配置保存**：`displayAction()` 是唯一的保存入口，见第 2.3 节和第 4.6 节。

**无效主题的提示**：当配置的主题不可用时，[display.phtml L86-L93](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/views/configure/display.phtml#L86-L93) 在主题预览列表末尾显示红色错误提示：

```php
<?php if (!$themeAvailable) {?>
    <li class="preview-container picked">
        <div class="preview"></div>
        <div class="properties alert-error">
            <div><?= _t('conf.display.theme_not_available', FreshRSS_Context::userConf()->theme)?></div>
        </div>
    </li>
<?php }?>
```

`$themeAvailable` 在遍历可用主题列表时判断：若某个主题的 `$theme['id']` 与 `userConf()->theme` 相等，则设为 `true`。否则保持初始值 `false`，显示提示。

### 6.6 无布局页面（_layout(null)）

**适用范围**：Ajax 请求、数据导出、JS 配置等。这些页面**不输出完整 HTML 文档**，无需关心 `lang`/`theme_*` class。

| Controller | Action | 触发条件 | 输出内容 |
|-----------|--------|---------|---------|
| entryController | 所有 action | `ajax=1` 时 | HTML 片段 |
| feedController | `actualizeAction` | `ajax=1` 时 | HTML 片段 |
| feedController | `contentSelectorPreviewAction` | 始终 | 自包含 HTML 文档（见 6.4） |
| categoryController | 多个 action | `ajax=1` 时 | HTML 片段 |
| subscriptionController | `feedAction` | `ajax=1` 时 | HTML 片段 |
| tagController | 多个 action | `ajax=1` 时 | HTML 片段 |
| extensionController | 多个 action | `ajax=1` 时 | HTML 片段 |
| importExportController | 多个 action | 始终 | JSON/文件 |
| javascriptController | `nonceAction` | 始终 | JS |
| indexController | `rssAction` | 始终 | RSS XML |
| indexController | `opmlAction` | 始终 | OPML XML |
| userController | `deleteAction` | 始终 | HTML 片段 |
| configureController | `queryAction` | `ajax=1` 时 | HTML 片段 |

**注意**：Ajax HTML 片段会被插入主页面 DOM，此时使用的 CSS/JS 和 `lang`/class 均来自主布局。片段本身不需要独立处理主题和语言。

### 6.7 三种 HTML 布局的主题/语言表现对比

| 维度 | layout.phtml（默认） | simple.phtml | contentSelectorPreview.phtml | 无布局（Ajax/RSS/JSON） |
|------|---------------------|-------------|-----------------------------|----------------------|
| `<html lang>` | `userConf()->language` | `userConf()->language` | `userConf()->language` | N/A（无 HTML 文档） |
| `<html xml:lang>` | `userConf()->language` | `userConf()->language` | `userConf()->language` | N/A |
| `dir="rtl"` | RTL 语言时 | RTL 语言时 | RTL 语言时 | N/A |
| `rtl` class | RTL 语言时 | RTL 语言时 | RTL 语言时 | N/A |
| `theme_*` class | ✅ `theme_{theme}` | ❌ **无** | ❌ **无** | N/A |
| `darkMode_*` class | ✅ `darkMode_{value}` | ✅ `darkMode_{value}` | ✅ `darkMode_{value}` | N/A |
| `controller_*` class | ✅ `controller_{name}` | ❌ 无 | ❌ 无 | N/A |
| `preview_background` class | ❌ 无 | ❌ 无 | ✅ 有 | N/A |
| `metaThemeColor()` | ✅ 有 | ✅ 有 | ❌ **无** | N/A |
| `preLayout()` / CSS | ✅ 完整 | ✅ 完整 | ✅ 完整 | N/A |
| `headScript()` | ✅ 完整 | ✅ 完整 | ❌ 仅 `preview.js` | N/A |
| 翻译 `_t()` | ✅ 可用 | ✅ 可用 | ✅ 可用 | ✅ 可用（数据导出也用） |

**一致性总结**：
- `lang` / `darkMode_*` / `rtl`：三种布局**一致**
- `theme_*` class：仅 layout.phtml 输出，simple 和 preview **缺失**（但已证实不影响内置主题视觉）
- `metaThemeColor()`：仅 preview 缺失（对 iframe 预览无实际影响）
- `headScript()`：preview 仅加载 `preview.js`（预览页不需要主脚本）

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
| [Minz/View.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/lib/Minz/View.php) | 视图基类，管理 styles/scripts/themeColors，build() 分发布局 |
| [Minz/Dispatcher.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/lib/Minz/Dispatcher.php) | 调度器，调用 controller → view.build() |
| [layout.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/layout/layout.phtml) | 主布局模板，完整 DOM（theme class + lang + darkMode） |
| [simple.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/layout/simple.phtml) | 简版布局模板，精简 DOM（无 theme class） |
| [contentSelectorPreview.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/views/feed/contentSelectorPreview.phtml) | 预览页自包含 HTML（无 theme class + 无 metaThemeColor） |
| [aside_configure.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/layout/aside_configure.phtml) | 设置页侧边栏导航 |
| [configureController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/configureController.php) | 显示配置页控制器，保存主题/语言/暗色模式 |
| [display.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/views/configure/display.phtml) | 显示配置页视图，含 theme_not_available 提示 |
| [Auth.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Models/Auth.php) | 认证系统，含 HTTP 认证自动建号的语言处理 |
| [authController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/authController.php) | 登录/注册控制器 |
| [userController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/userController.php) | 用户管理控制器，含 createUser 建号逻辑 |
| [feedController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/feedController.php) | Feed 控制器，含 contentSelectorPreviewAction |
| [p/api/index.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/p/api/index.php) | API 信息页 |
| [p/api/greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/p/api/greader.php) | Google Reader 兼容 API |
| [p/api/fever.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/p/api/fever.php) | Fever 兼容 API |
| [p/api/query.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/p/api/query.php) | 共享查询 API |
| [p/api/misc.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/p/api/misc.php) | 扩展 API |
| [install.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/install.php) | 安装向导，自有语言选择逻辑 |
