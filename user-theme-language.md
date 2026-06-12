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

### 2.2 特殊场景的语言选择

| 场景 | 代码位置 | 语言来源 |
|------|----------|----------|
| 未登录用户首次访问 | [FreshRSS_Auth](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Models/Auth.php#L81-L84) | `getLanguage(null, getPreferredLanguages(), systemConf()->language)` |
| 注册/登录页面 | [authController](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Controllers/authController.php#L261) | `getLanguage(null, getPreferredLanguages(), systemConf()->language)` |
| API 请求 | [p/api/index.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/p/api/index.php#L15) | `getLanguage(null, getPreferredLanguages(), null)` |

### 2.3 浏览器语言解析

[Minz_Request::getPreferredLanguages()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/lib/Minz/Request.php#L583-L588) 解析 `HTTP_ACCEPT_LANGUAGE` 头，返回语言列表；若无该头则返回 `['en']`。

### 2.4 语言切换生效路径

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

### 2.5 翻译文件结构

翻译文件位于 `app/i18n/{lang}/`，每个子目录对应一种语言，内部按模块分文件：

- `gen.php` — 通用词条
- `conf.php` — 配置页词条
- `admin.php` — 管理页词条
- `sub.php` — 订阅页词条
- `plurals.php` — 复数形式规则

加载逻辑：[Minz_Translate::loadLang()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/lib/Minz/Translate.php#L158-L196) 若所选语言目录不存在，自动回退到 `en/`。

---

## 3. 主题选择机制

### 3.1 主题优先级

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

### 3.2 主题验证

[FreshRSS_Themes::exists()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/Models/Themes.php#L18-L22) 检查：
- 主题 ID 不含 `..`、`/` 或目录分隔符（防路径穿越）
- `p/themes/{theme_id}/metadata.json` 文件存在

### 3.3 内置主题列表

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

### 3.4 主题配置文件 metadata.json

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

### 3.5 Dark Mode

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

### 3.6 主题切换生效路径

用户在「显示配置」页面提交主题设置后：

```
1. Minz_Request::paramString('theme')    → 从 POST 取得新主题名
2. FreshRSS_Themes::exists($theme)       → 验证主题存在
3. userConf()->theme = $theme            → 写入用户配置对象
4. userConf()->save()                    → 持久化
5. 页面重定向 → 下次请求时 loadStylesAndScripts() 从 userConf()->theme 读取
```

---

## 4. 页面渲染生效路径

### 4.1 完整请求生命周期

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

### 4.2 主题生效的关键代码

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

### 4.3 语言生效的关键代码

[layout.phtml](file:///d:/fz/0601-1/solo-dogfeeding/code/29-FreshRSS/app/layout/layout.phtml#L28) 中语言写入 HTML 属性：

```php
<html lang="<?= FreshRSS_Context::userConf()->language ?>"
      xml:lang="<?= FreshRSS_Context::userConf()->language ?>">
```

翻译文本通过 `_t('key.subkey')` 函数在所有 .phtml 视图模板中使用，该函数是 `Minz_Translate::t()` 的别名。

### 4.4 RTL 支持

语言为 RTL（如 `he` 希伯来语）时，翻译词条 `gen.dir` 返回 `'rtl'`，layout.phtml 会：

- 在 `<html>` 添加 `dir="rtl"` 属性
- 在 HTML class 中添加 `rtl`
- CSS 文件自动替换为 `.rtl.css` 版本

---

## 5. 配置项速查表

| 配置项 | 用户配置键 | 默认值 | 影响范围 |
|--------|-----------|--------|----------|
| 语言 | `language` | `'en'` | HTML lang、翻译文本、RTL 检测 |
| 主题 | `theme` | `'Origine'` | CSS class `theme_*`、加载的 CSS/JS 文件 |
| 暗色模式 | `darkMode` | `'auto'` | CSS class `darkMode_*` |
| 内容宽度 | `content_width` | `'thin'` | CSS class |
| 时区 | `timezone` | `''`（服务器默认） | 日期时间显示 |

---

## 6. 关键源文件索引

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
