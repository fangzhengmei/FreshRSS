# FreshRSS API 兼容层分析

## 一、API 兼容层整体架构

FreshRSS 提供了多套 API 兼容层，用于支持不同的 RSS 阅读器客户端。所有 API 入口均位于 `p/api/` 目录下。

| API 类型        | 入口文件                          | 主要用途                     | 兼容性参考              |
|-----------------|-----------------------------------|------------------------------|-------------------------|
| Google Reader   | [greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php)       | 移动客户端主 API             | Google Reader API v2    |
| Fever           | [fever.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/fever.php)           | Fever 协议客户端             | Feed af Fever API       |
| Query           | [query.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/query.php)           | 公开共享查询（只读）         | 基于 Token 的公开访问   |
| Misc/Extension  | [misc.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/misc.php)             | 扩展 API 入口                | 钩子机制                |
| API 信息页      | [index.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/index.php)           | API 地址与可用性测试         | -                       |

**系统级开关**：所有 API 受 [config.default.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/config.default.php#L79) 中 `api_enabled` 配置项控制，默认为 `false`。

---

## 二、鉴权方式

### 2.1 Google Reader API 鉴权

Google Reader API 采用 **双层认证机制**：请求级 Auth Header + 写操作 Token。

#### 2.1.1 认证入口：Authorization Header

**位置**：[greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L178-L207) `authorizationToUser()`

```
Authorization: GoogleLogin auth={username}/{sha1(salt + username + apiPasswordHash)}
```

- **解析方式**：从 HTTP Authorization 头中提取 `GoogleLogin auth` 值
- **格式**：`用户名 / 令牌`，以 `/` 分隔
- **令牌算法**：`sha1(system_salt + username + user_apiPasswordHash)`
- **校验失败**：返回 `401 Unauthorized`，并附 `Google-Bad-Token: true` 响应头

#### 2.1.2 ClientLogin 登录端点

**位置**：[greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L209-L232) `clientLogin()`

- **路径**：`/accounts/ClientLogin`
- **方法**：POST（推荐），也兼容 GET（有安全警告日志）
- **参数**：`Email`（用户名）、`Passwd`（API 密码）
- **密码校验**：使用 `password_verify()` 验证 `apiPasswordHash`
- **返回**：
  ```
  SID={auth_token}
  LSID=null
  Auth={auth_token}
  ```
- **auth_token 格式**：与 Authorization Header 中的 auth 值相同

#### 2.1.3 写操作 Token 校验

**位置**：[greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L234-L263) `token()` / `checkToken()`

- **获取 Token**：`GET /reader/api/0/token`
- **Token 格式**：`str_pad(sha1(salt + user + apiPasswordHash), 57, 'Z')`，固定 57 字符
- **Token 校验**：所有 POST 写操作（edit-tag、mark-all-as-read 等）需携带 `T` 参数
- **兼容特例**：
  - 空 token（FeedMe 客户端）视为通过
  - token 值为 `x`（Reeder 客户端）视为通过

### 2.2 Fever API 鉴权

**位置**：[fever.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/fever.php#L169-L194) `authenticate()`

#### 2.2.1 认证方式

- **参数**：POST `api_key`
- **算法**：`md5(username:apiPassword)`，全部小写
- **长度限制**：截取前 128 字符，仅接受十六进制字符

#### 2.2.2 Key 存储机制

**位置**：[feverUtil.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Utils/feverUtil.php)

- **存储方式**：文件系统映射，路径为 `data/fever/.key-{sha1(salt)}-{feverKey}.txt`
- **文件内容**：纯文本用户名
- **设计原因**：通过 key 反查用户名，避免遍历所有用户配置

### 2.3 Query API 鉴权（Token 模式）

**位置**：[query.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/query.php)

- **参数**：`t`（token）、`user`（用户名）、`f`（格式）
- **Token 来源**：用户自定义查询（User Query）的共享 token
- **权限**：只读，仅能访问用户主动共享的查询结果
- **格式**：支持 atom、greader、html、json、opml、rss 六种输出格式

### 2.4 通用安全机制

| 安全措施               | 说明                                                         |
|------------------------|--------------------------------------------------------------|
| 系统总开关             | `api_enabled` 全局禁用时返回 503                             |
| 用户启用状态检查       | 用户配置 `enabled` 为 false 时拒绝访问                       |
| API 密码独立           | 与 Web 登录密码分离，使用专用 `apiPasswordHash`              |
| 用户枚举缓解           | Query API 中使用随机 `usleep()` 延迟缓解用户扫描             |
| CSP 安全头             | API 响应设置严格 Content-Security-Policy                     |
| CORS 支持              | Google Reader API 支持跨域（Access-Control-Allow-Origin: *） |

---

## 三、参数裁剪

### 3.1 输入参数裁剪与校验

#### 3.1.1 Google Reader API 输入处理

**位置**：[greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php)

| 参数类型   | 处理方式                                                                 |
|------------|--------------------------------------------------------------------------|
| 多值参数   | `multiplePosts()` 直接解析原始 `php://input`，避免 PHP 自动覆盖同名参数 |
| 条目 ID    | 支持十进制数字和 `tag:google.com,2005:reader/item/{hex}` 格式，统一转十进制 |
| 时间戳     | `ot`/`nt` 参数强制转为 int，默认 0                                       |
| 分页 token | `c` (continuation) 参数必须为数字字符串，否则重置为 `'0'`                |
| 数量限制   | `n` 参数默认 20 条                                                       |
| 流 ID      | 支持 `feed/{id|url}`、`user/-/label/{name}` 等多种格式，统一内部化        |
| HTML 编码  | 所有用户输入的名称类参数使用 `htmlspecialchars()` 编码存入数据库         |

#### 3.1.2 Fever API 输入处理

**位置**：[fever.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/fever.php)

- `api_key`：截取前 128 字符，必须为十六进制
- `id`：必须为数字字符串
- `with_ids`：按逗号分割后过滤非数字项
- `feed_ids` / `group_ids`：按逗号分割后过滤非数字项
- `max_id` / `since_id`：非数字字符串重置为空

### 3.2 输出参数裁剪

#### 3.2.1 Google Reader 条目格式化

**位置**：[Entry.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/models/Entry.php#L1219-L1318) `toGReader()`

支持三种输出模式：

| 模式        | 触发条件       | 主要裁剪行为                                                                 |
|-------------|----------------|------------------------------------------------------------------------------|
| `compat`    | 兼容模式       | 内容截断为 500KB；特殊字符转义为全角 Unicode；移除 `alternate.type` 字段     |
| 默认模式    | 标准模式       | 完整内容输出；保留 `alternate.type` 字段                                     |
| `freshrss`  | FreshRSS 扩展  | 额外输出 `guid` 字段；输出 `origin.feedUrl`；输出 `unread` 状态标签          |

**核心裁剪常量**：
- `API_MAX_COMPAT_CONTENT_LENGTH = 500000`（约 500KB，兼容模式内容上限）

#### 3.2.2 特殊字符转义

**位置**：[lib_rss.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/lib/lib_rss.php#L166-L180) `escapeToUnicodeAlternative()`

将可能导致 XML/JSON 解析问题的字符替换为 Unicode 全角形式：
- 基础字符：`&` → `＆`、`<` → `＜`、`>` → `＞`
- 扩展字符（`$extended=true`）：`'`、`"`、`^`、`?`、`\`、`/`、`,`、`;` 等

#### 3.2.3 内部字段清理

- `frss:id`：内部使用的数字 ID，在 JSON 输出前通过 `unset()` 移除
- 隐藏 Feed：`priority <= PRIORITY_HIDDEN` 的订阅不出现在 API 列表中
- 用户标签：Query API 的 greader/json 格式不导出用户标签（隐私保护）

#### 3.2.4 Fever API 输出裁剪

**位置**：[fever.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/fever.php#L490-L563) `getItems()`

- 条目列表固定上限：50 条/页
- 作者名去除两端分号和空格
- 不支持的字段：`is_spark` 固定为 0，`links` 固定为空数组
- 时间戳统一为 Unix 时间戳格式

---

## 四、客户端状态同步边界

### 4.1 状态同步的核心数据模型

状态同步围绕 **条目（Entry）** 的两个核心布尔状态展开：

| 状态字段        | 数据库字段        | 说明                     | 同步方向 |
|-----------------|-------------------|--------------------------|----------|
| `is_read`       | `is_read`         | 已读/未读状态            | 双向     |
| `is_favorite`   | `is_favorite`     | 收藏/星标状态            | 双向     |

**用户修改时间戳**：
- `lastUserModified`：用户最后修改条目标记的时间戳，用于增量同步
- `FreshRSS_UserDAO::touch()`：每次批量状态修改后更新用户级修改时间

### 4.2 状态同步操作边界

#### 4.2.1 单条/批量标记

**位置**：[EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/models/EntryDAO.php)

| 操作            | 方法                       | 批量上限                    |
|-----------------|----------------------------|-----------------------------|
| 标记已读        | `markRead($ids, true)`     | 受数据库最大参数数限制，自动分批 |
| 标记未读        | `markRead($ids, false)`    | 同上                        |
| 标记收藏        | `markFavorite($ids, true)` | 同上                        |
| 取消收藏        | `markFavorite($ids, false)`| 同上                        |

**GReader API 端点**：`/reader/api/0/edit-tag`
- 参数 `a`：要添加的标签/状态（可重复）
- 参数 `r`：要移除的标签/状态（可重复）
- 参数 `i`：条目 ID（可重复）

**Fever API 端点**：`?mark=item&as=read|saved|unread|unsaved`
- 参数 `id`：单条 ID
- 参数 `with_ids`：逗号分隔的批量 ID

#### 4.2.2 范围标记（全部已读）

**位置**：[EntryDAO.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/models/EntryDAO.php#L585-L770)

| 范围    | 方法                    | GReader API 对应流 ID                       |
|---------|-------------------------|---------------------------------------------|
| 全局    | `markReadEntries()`     | `user/-/state/com.google/reading-list`      |
| 分类    | `markReadCat()`         | `user/-/label/{category}`                   |
| Feed    | `markReadFeed()`        | `feed/{id}`                                 |
| 标签    | `markReadTag()`         | `user/-/label/{tag}`                        |
| 收藏    | `markReadEntries()`     | `user/-/state/com.google/starred`           |

**截止条件**：所有范围标记均支持 `olderThanId` 参数（即 `ts`，纳秒级时间戳/ID），只标记指定 ID 之前的条目。

**Fever API 对应**：
- `?mark=feed&as=read&id={feed_id}&before={timestamp}`
- `?mark=group&as=read&id={group_id}&before={timestamp}`

#### 4.2.3 订阅管理同步

**位置**：[greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L390-L483) `subscriptionEdit()`

| 操作         | 动作参数     | 说明                                   |
|--------------|--------------|----------------------------------------|
| 订阅         | `subscribe`  | 添加 Feed 到指定分类                    |
| 退订         | `unsubscribe`| 删除 Feed                              |
| 编辑         | `edit`       | 修改 Feed 标题、移动到新分类            |
| 快速添加     | `quickadd`   | 快速订阅 URL，自动发现分类              |
| 导出 OPML    | `export`     | 导出订阅列表为 OPML 格式                |
| 导入 OPML    | `import`     | 从 OPML 导入订阅并触发刷新              |

**Fever API 对应**：只读获取 `feeds`、`groups`、`feeds_groups`，无写操作。

#### 4.2.4 标签/分类管理同步

**位置**：[greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L980-L1029)

| 操作       | 端点               | 说明                                       |
|------------|--------------------|--------------------------------------------|
| 重命名标签 | `rename-tag`       | 同时支持分类和标签重命名（先匹配分类）       |
| 禁用标签   | `disable-tag`      | 删除分类下的 Feed 移入默认分类，或删除标签   |
| 标签列表   | `tag/list`         | 返回所有系统标签（状态标签）、分类、用户标签 |

### 4.3 增量同步机制

#### 4.3.1 时间范围过滤

**位置**：[greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L609-L678) `streamContentsFilters()`

- `ot` (older than / start_time)：只返回该时间戳之后爬取或修改的条目
- `nt` (newer than / stop_time)：只返回该时间戳之前的条目
- **实现方式**：对 `date` 和 `lastUserModified` 分别构建搜索条件，OR 关系

#### 4.3.2 分页延续（Continuation）

**位置**：[greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L683-L760) `streamContents()`

- 使用条目 ID 作为分页游标
- `c` 参数传入上一页最后一条的 ID
- 响应包含 `continuation` 字段表示还有更多数据
- 实现细节：多取 1 条，跳过第 1 条（因为 continuation 是已返回的最后一条 ID）

#### 4.3.3 未读计数同步

**位置**：[greader.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/p/api/greader.php#L507-L567) `unreadCount()`

- 按 Feed、分类、标签、阅读列表分别统计未读数
- 附带 `newestItemTimestampUsec` 字段（最新条目时间戳，微秒级）
- 客户端可通过未读数变化判断是否需要拉取新内容

**Fever API 对应**：
- `unread_item_ids`：所有未读条目的 ID 逗号列表
- `saved_item_ids`：所有收藏条目的 ID 逗号列表

### 4.4 状态同步的边界约束

| 边界类型           | 约束条件                                                                 |
|--------------------|--------------------------------------------------------------------------|
| 优先级过滤         | PRIORITY_HIDDEN 的 Feed 及其条目不出现在 API 同步范围内                  |
| 状态原子性         | 单条标记使用事务和乐观锁（WHERE is_read <> ?）保证并发一致性             |
| 批量大小限制       | 受数据库最大参数数限制，超大批量自动分片处理                             |
| 扩展钩子           | `EntryBeforeDisplay` 钩子可在输出前修改条目；`EntriesFavorite` 钩子监听收藏变化 |
| 时间精度           | GReader API 使用微秒级时间戳；Fever API 使用秒级时间戳                   |
| ID 格式            | 内部为 64 位十进制 ID；对外兼容 Google Reader 的十六进制 tag URI 格式    |

---

## 五、核心代码参考

### 5.1 GReader API 入口流程

1. **初始化**：`GReaderAPI::parse()` → 系统初始化 → 检查 `api_enabled`
2. **认证**：`authorizationToUser()` 解析 Authorization Header
3. **路由**：根据 PATH_INFO 分发到对应处理方法
4. **执行**：调用 DAO 层进行数据操作
5. **输出**：JSON 流式输出（大响应避免内存问题）

### 5.2 Fever API 入口流程

1. **系统初始化**：检查 `api_enabled`
2. **认证**：`authenticate()` 验证 `api_key`
3. **组合响应**：根据请求参数组合不同数据块（groups, feeds, items 等）
4. **包装输出**：`wrap()` 方法统一添加 `api_version`、`auth`、`last_refreshed_on_time`

### 5.3 密码管理

**位置**：[apiController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/27-FreshRSS/app/Controllers/apiController.php#L13-L33) `updatePassword()`

- API 密码通过 Web 界面设置，使用 `FreshRSS_password_Util::hash()` 哈希存储
- 同时生成 Fever API key 并存入用户配置和文件映射
- API 密码与 Web 登录密码相互独立
