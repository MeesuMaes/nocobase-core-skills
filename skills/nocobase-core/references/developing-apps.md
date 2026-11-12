## 1. 开发应用

NocoBase Core 与配套插件提供数据库、服务端、客户端、认证与权限等基础能力，业务代码专注功能本身。应用由客户端、服务端、数据库、插件和配置按需组合。可与 Agent 协作：先读项目 `AGENTS.md`、应用规则和随包 Skills。

### 目录约定

目录随模板与功能变化，不必为匹配目录树补齐空目录。`services/`、`commands/`、`packages/`、外部库 `metadata/` 均按需创建。

```text
my-app/
├── .agents/rules/          # 应用自有规则
├── .agents/skills/         # 从依赖同步的 Skills（可再生成）
├── cli/commands/           # 应用命令，调用为 pnpm nocobase app …
├── cli/plugins.ts
├── client/pages|layouts|components|hooks|locales|theme/
├── client/routes.ts        # 路由、菜单、权限
├── client/plugins.ts
├── client/runtime.ts
├── database/<连接名>/migrations|seeds|metadata/
├── database/<连接名>/collections/   # 生成快照，不手改，不作为迁移输入
├── packages/               # 可选本地插件/共享包/UI Registry
├── public/
├── server/routes|services|providers|config/
├── server/plugins.ts
├── server/runtime.ts
├── tests/
├── storage/                # 运行数据（库文件、上传、日志）；生成≠可删
├── dist/  node_modules/    # 可重建
├── AGENTS.md
├── config.example.yml      # 可提交示例
├── config.yml              # 含密钥，不提交 Git
├── components.json         # shadcn
├── package.json  pnpm-lock.yaml  pnpm-workspace.yaml
```

接入：页面 → `client/routes.ts`；HTTP → `server/routes/index.ts`；服务 → 各端 ServiceProvider 聚合入口；配置分区 → `server/config/index.ts`；插件按实际提供的入口写入三端 `plugins.ts`。`runtime.ts` 组装运行时。模板自带的布局、主题和运行时入口是应用源码，升级时保留业务定制。

### 配置

合并顺序（后者覆盖前者）：`server/config/` 代码默认值 → `config.yml` → 已声明映射的环境变量。普通对象递归合并，数组和标量整体替换。YAML 中 `${NAME}` 是普通字符串，不会自动插值。环境变量须显式声明映射。`APP_CONFIG_FILE` 可改用另一份配置（相对路径相对应用根）。`dist/variables.json` 是部署配置元数据清单，不是密钥备份。`config init` 在已有配置时通常保持原文件，覆盖需 `--force`；已有应用应保留原密钥，不要靠重新初始化升级。改配置后部分服务需重启或自行处理 `reload()`。`users.initialAdmin` 只用于空用户库初始化，不重置已有账号。`client` 分区会写入页面；其他服务端分区仅 `defineAppConfig()` 的 `public` 叶子路径可公开。密码、密钥、邮件凭证不得放入 `client` 或公开路径。

`schemaManagement: managed` 表示框架可管理结构；`external` 限制框架结构变更，不是业务只读开关。默认 `DB_*` 只映射 `main` 连接。改方言或地址不会自动搬迁数据。SQLite 同时存在 `database` 与 `filename` 时 `database` 优先。

常用字段与环境变量见第 7 节。完整清单：`pnpm nocobase config variables --json`。

### 客户端

- 路由：`defineAppRoutes()`（`@nocobase/app-client` / `@nocobase/app-client/plugins`）。页面必填 `name`（同包内唯一，以字母或数字开头，可含点/下划线/连字符）、`path`、`componentLoader`；可选 `navigation`（`{ title, icon?, order? }`）、`breadcrumb`、`auth`、`authz`、`parent`（仅贡献根条目）、`children`。纯分组必填 `name`、`children`（可空数组）、`navigation`，不可设 `componentLoader` 与 `authz`，`path` 可选（作子路由前缀）。`auth` 顶层默认 `required`，子层继承。标题为字符串，图标为组件，不支持按用户执行配置回调。底层 React Router 的 `index`、`element`、`loader`、`action`、`errorElement` 不是该声明 API 的字段。`authz` 例：`{ resource: { type: 'page', id: 'projects' }, action: 'access' }` 或 `'skip'`。插件路由的 `parent` 可带包前缀。覆盖插件页：替换页面加载器，保留插件定义的路径、身份与权限元数据（`client/route-overrides.ts` 或插件提供的方式）。
- 路径规则：顶层路径以 `/` 开头；子路径追加到父层，`/new` 并不跳到根。不在 `path` 中写查询串、hash、`*`、可选参数 `?`、反斜杠或 `.`/`..` 段。页面地址在整个应用和所有插件之间唯一；冲突检查忽略静态段大小写和参数名（`/Projects` 与 `/projects` 冲突，`/:id` 与 `/:projectId` 冲突；`new` 与 `:projectId` 可共存）。不同包可同名，但地址不能相同。`/` 只能由应用提供；`/login`、`/register`、`/forgot-password`、`/reset-password` 必须 `auth: 'guest'`。

- UI：统一 shadcn，组件源码在应用中；页面连接数据与操作。品牌、侧栏、全局操作由应用外框管理；业务页组合已有容器、标题和内容区。主题用语义变量，改主题文件即可让列表/表单/弹窗一起适配；用户已保存的配色优先于 `client.app.defaultColorScheme`（`light`/`dark`/`system`）。`client.app.defaultTheme` 须在主题源码中存在。模块化区块优先参考 NocoBase UI Library（https://ui.nocobase.com/）。
- 请求：组件用 `useApiClient()`；`api.request<T>(options)` 返回完整响应体，不自动解包 `data`，泛型不做运行时校验。选项：`path`、`method`（默认 GET）、`query`、`headers`、`signal`、`credentials`（默认 `include`）；JSON 用 `json`，其他用 `body`，二者互斥。`api.stream()`、`api.repository(name)`（需服务端已开放对应接口；`name` 可以与 Collection 名不同；远程查询也走 POST，并自动解开 `data`）。失败为 `ApiClientError`。不要把浏览器缓存的权限当服务端授权依据。登录切换应清理旧身份状态。
- 多语言：`client/locales/` 与 `server/locales/` 独立加载；页面能翻译的 key，服务端仍需声明。
- 客户端服务：跨页面复用操作或应用级资源时用 ServiceProvider 注册到容器；`useService(token)` 只解析、不订阅变化，订阅需 `useSyncExternalStore` 等。React Provider ≠ ServiceProvider。

### 服务端

- 路由：`defineApiRoutes(factory)` 挂到 API 前缀（普通应用为 `/api`，还需考虑基础路径）；`defineRootRoutes()` 挂到应用根；`defineHttpMiddleware({ name, register })`；`apiValidator(target, schema)`；`throw new ApiError({ status, reason, domain, message })`，框架转为含 `error` 的失败响应。`message` 不能作为客户端分支条件；字段问题用 `fieldViolations`。自定义接口可统一 `{ data, meta }`。OpenAPI 辅助从 `@nocobase/app-server/router` 导入。路由工厂接收应用实例、返回 Hono Router，还须经应用或插件扩展入口接入。
- 服务：不必继承框架基类。HTTP、命令、后台任务复用同一服务。不要在单例字段保存当前用户、事务或本次输入。事务内用回调连接上的 Repository。邮件、HTTP、队列通常不随库回滚，放在 `afterCommit()` 或 Outbox。CLI/任务不能默认管理员权限。
- 认证与权限：认证、操作权限、数据范围职责分离。接口先认证再取当前用户；业务操作需注册，授权决策产生策略后 `withPolicy(policy)` 绑到 Repository。用户提交的过滤条件不是授权策略，只能缩小范围。页面权限与接口权限要分别配置和验证。不要把请求里的 `ownerId` 直接当当前用户。
- 后台：`JobExecutor` 先返回受理回执；`ScheduleExecutor` 按 Cron 或间隔触发。业务负责幂等与结果保存。需要管理页和执行记录时用定时任务插件。
- 工作流：基于状态机（`@nocobase/lifecycle`）定义状态、允许流转与条件，适合跨多次请求、由人员/自动/外部系统共同推进的流程。

### 数据库（应用侧）

应用资源在 `database/<连接名>/`。连接名对应目录，不是业务名。Examples 模板常用 `main` + 演示用 `analytics`。

- Migration：`defineMigration()`，`name` 必须与去扩展名的文件名一致，默认导出其返回值。必须有 `down()` 或 `irreversible: true`。已合入/发布的迁移不改内容（格式/注释也会改校验和），后续变化写新迁移。不要导入随应用变化的模型或服务。
- Seed：`defineSeed({ name, run, transaction?, sample? })`，必须默认导出，`name` 与文件名一致且在同一连接全部 Seed 来源中唯一。没有 `down()`、`shouldRun()` 或重复执行模式。用稳定业务键判断是否已存在再写入（或 `upsertOne`）。Seed 无 `builder`，结构须先由迁移建立。同一连接 Seed 按名称排序，位于迁移之后。`sample: true` 标记可选示例；普通 Seed 不会因记录“看起来像示例”而跳过。应用仅在连接首次安装且 `app.sampleData` 为 `true` 时自动执行 sample Seed（首次安装＝执行前没有迁移和 Seed 历史，或刚完成受管理结构重置）；否则记为 `skipped`，以后只打开配置也不会自动跑。开发可用 `pnpm nocobase db sample` 补载被跳过的示例（部署产物拒绝执行）。示例若必须调用其他插件业务服务，应使用 `sampleDataToken` 在应用就绪后生成，不能把业务服务塞进 Seed Context。迁移回退不删 Seed 历史。不要让迁移依赖同一次执行里尚未运行的 Seed。

- `collections/` 用 `pnpm nocobase collections generate` 更新（`--check` 只检查）。
- `schemaManagement: 'external'`：框架读已有结构并叠加应用元数据，不通过该连接跑迁移/Seed；`metadata/` 是源码，不能创建表。缺字段找结构维护方，不能给 external 连接加应用迁移来“修表”。元数据不能代替物理约束。增加连接或元数据不会自动开放 API。
- 取得数据库：`app.container.resolve(databaseManagerToken)`，再 `database.repository('orders')`。操作在服务端执行。

### 命令与插件管理

应用命令放在 `cli/commands/`，例如 `cli/commands/orders/sync.ts` → `pnpm nocobase app orders sync`。继承 `AppCommand`，可用 `withApp()`；不要在模块顶层连库。`cli link` / `cli build` 用于通过 HTTP 调用已运行应用的独立客户端（`@nocobase/app-cli-client`）。

管理插件：先查 `package.json`、锁文件和三端 `plugins.ts`，不要只根据 `node_modules` 推断已启用。`plugin register` / `update` / `unregister`（可 `--dry-run`）；`unregister` 不等于删除库中业务数据；省略名称的 `plugin update` 升级全部已注册插件并同步 Skills。`skills sync` 更新 `.agents/skills/`；维护插件 Skill 时改插件 `skills/` 源文件。`template-print` 仅分发 Skills，不必加入注册数组。`agents` 与 `knowledge` 在默认模板依赖中，但没有默认注册。扩展界面：优先插件公开 options/组件；需改源码则从 UI Library 安装由应用维护的副本。

升级 Core：先备份库、配置和密钥，保留锁文件；按文档逐项合并模板与依赖，同步 Skills，检查并执行迁移。回退代码不会自动撤销结构变更。不要用新模板覆盖原应用来“升级”。

测试：正常路径、权限拒绝、失败重试都要覆盖。迁移测试用 `@nocobase/app-testing/server` 的 `describeMigration()`。常用脚本：`pnpm dev`、`pnpm test`、`pnpm lint`、`pnpm typecheck`、`pnpm build`、`pnpm start`（以应用 `package.json` 为准）。

**出处：** `developing-apps/overview`，`project-structure`，`configuration-and-settings`，`testing-and-acceptance`，`pages-routes-and-navigation`，`pages-and-interactions`，`data-requests-and-state`，`page-layouts`，`localization`，`themes-and-styles`，`client-business-services`，`apis-and-services`，`server-business-services`，`request-validation-and-errors`，`authentication-and-permissions`，`background-tasks`，`business-workflows`，`server-localization`，`data-models-and-migrations`，`querying-data`，`transactions-and-consistency`，`multiple-databases`，`existing-databases`，`application-commands`，`plugin-commands`，`cli-client`，`manage-plugins`，`upgrade-nocobase-core`

---

