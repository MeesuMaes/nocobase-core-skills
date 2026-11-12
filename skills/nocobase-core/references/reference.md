## 7. 参考

### 公开导入入口

| 入口 | 常用导出 |
| --- | --- |
| `@nocobase/app-client` | `createApp`、`defineClientPlugin`、`defineAppRoutes`、`useService`、`useApiClient` |
| `@nocobase/app-server` | `defineServerPlugin`、`defineServerPlugins` |
| `@nocobase/app-server/router` | `defineApiRoutes`、`defineRepositoryApiRoutes`、`defineRootRoutes`、`defineHttpMiddleware`、`apiValidator`、`ApiError` |
| `@nocobase/app-server/config` | `defineAppConfig`、`defaultAppConfigs`、`envString`、`envInteger`、`envBoolean` |
| `@nocobase/service-provider` | `ServiceProvider`、`ServiceContainer`、`createServiceToken` |
| `@nocobase/api-client` | `createApiClient`、`ApiClientError` |
| `@nocobase/db` | `createDatabaseManager`、`databaseManagerToken` |

数据库完整签名另见 Core 源码 `packages/libs/db/docs/api-reference/`（修订 `015404ecd0db`）。

### 命令速查

“两者”= 源码与部署产物均有。插件还会贡献命令，以 `commands` 输出为准。`dev`/`build`/`start` 等透传命令不统一支持 JSON。

| 命令 | 环境 | 要点 |
| --- | --- | --- |
| `dev` / `build` / `start` | 源码 | 开发、构建、启动产物 |
| `info` / `commands` | 两者 | `commands --json` 含参数与执行能力 |
| `config init` | 两者 | `--dialect`、`--config`；不安装驱动 |
| `config set` | 两者 | `key=value`；`--from-env` 把当前环境变量值写入文件（不是动态引用） |
| `config check` | 两者 | 默认尝试非 SQLite 连接；`--no-connect` 离线；`--connect` 也验证 SQLite；`--strict` 警告即失败；不执行迁移 |
| `config env` / `variables` | 两者 | 映射 / 部署清单；`variables --out` 写文件 |
| `db apply` | 两者 | 待运行迁移与初始数据；`--dry-run`；`--connection` 与 `--all` 互斥 |
| `db rollback` / `redo` | 两者 | 回退最近一批 / 回退后重做 |
| `db reset` | 两者 | 重置受管理表，删数据 |
| `db repair` | 两者 | 只修迁移校验和记录 |
| `db unlock` | 两者 | 处理残留锁；须先确认无迁移进程 |
| `db sample` | 仅开发 | 部署环境拒绝执行 |
| `collections generate` / `doctor` | 两者 | 生成快照 / 诊断；`doctor --fix` 会修改内容 |
| `locales check` | 两者 | 检查多语言资源 |
| `secrets status` / `rotate` | 两者 | `rotate --dry-run`、`--batch-size` |
| `plugin inspect/register/unregister/update` | 源码 | register/update 可 `--dry-run` |
| `package remove` | 源码 | 移除依赖及相关接入 |
| `skills sync` | 源码 | 可按包选择、`--dry-run` |
| `dist check` / `retarget` | 源码 | 检查/调整部署产物 |
| `cli build` / `cli link` | 源码 | 可分发业务 CLI / 本地链接 |

JSON 成功信封含 `schemaVersion`、`ok`、`command`、`status`、`result`、`warnings`。同时检查退出码、`ok`、`status`。`success-noop` 无需变更；`partial-success` 须看 warnings。`NOCOBASE_CONTENT_TYPE=json` 只对支持该模式的命令生效。

### 常用配置与环境变量

字段：`server.host`/`port`（模板代码默认 `127.0.0.1`/`13000`）、`server.startLog`、`app.publicOrigin`、`app.sampleData`、`database.default`（模板 `main`）、`database.connections.<name>`、`api.bodyLimit`/`timeout`/`rateLimit`、`secrets.keys`、`session`/`auth`、`users.initialAdmin`、`i18n.defaultLocale`、`logging`、`client.app.title`/`defaultColorScheme`/`defaultTheme`、`jobs`/`queue`/`caching`/`drive`，以及随插件出现的 `mail`/`notification`/`scheduler`/`knowledge`。局域网监听可改 `0.0.0.0`，但浏览器须用实际 IP；`pnpm dev` 可能还有独立前端端口。

环境变量映射（已核对清单，精确以 `config variables` 为准）：`APP_SERVER_HOST`/`PORT`/`START_LOG`，`APP_PUBLIC_ORIGIN`，`APP_SAMPLE_DATA`，`DB_DIALECT`/`HOST`/`PORT`/`DATABASE`/`FILENAME`/`USERNAME`/`PASSWORD`/`SSL`（仅 main），`API_BODY_LIMIT`/`TIMEOUT`，`SECRETS_KEYS`（`版本:密钥` 逗号分隔，第一项为当前密钥，轮换须保留旧项），`AUTH_SECRET`/`SESSION_SECRET`，`INITIAL_ADMIN_*`，`APP_DEFAULT_LOCALE`，`SNOWFLAKE_WORKER_ID`，`APP_VITE_DEV_URL`（`spa.viteDevUrl`），以及 mail/notification/scheduler 相关项。运行时另读：`APP_BASE_PATH`、`APP_CONFIG_FILE`、`NOCOBASE_STRICT_STARTUP`。会话优先用显式 `session.secret`，未设时才尝试从 Secrets 派生。

自定义分区示例：`defineAppConfig({ defaults, env: { ANNOUNCEMENTS_PAGE_SIZE: envInteger('pageSize', { description }) }, public: ['pageSize'], validate })`，在 `defaultAppConfigs({ … })` 中登记。`validate` 在启动、reload、`config check` 时运行，不要在其中发网络请求或写数据。

### 数据库方言

8 种方言；MariaDB 与 MySQL 共用 `mysql`，不要写成 `mariadb`。

| 数据库 | 包 | dialect | 要点 |
| --- | --- | --- | --- |
| SQLite | `@nocobase/db-sqlite` | `sqlite` | 默认模板已装；`database` 为文件路径，相对路径按 `storage/` 解析 |
| PostgreSQL | `@nocobase/db-postgres` | `postgres` | 可 `schema`、`ssl`；专用 CA 用 `ssl.ca` 填 PEM 内容而非路径 |
| MySQL / MariaDB | `@nocobase/db-mysql` | `mysql` | `charset` 如 `utf8mb4`；JSON 过滤需 MySQL ≥8.0.17 或 MariaDB ≥10.9 |
| SQL Server | `@nocobase/db-mssql` | `mssql` | `encrypt`、`trustServerCertificate` |
| Oracle | `@nocobase/db-oracle` | `oracle` | 用 `serviceName`（不是 SID，也不用 `database`） |
| 达梦（Pro，实验性） | `@nocobase/db-dameng` | `dameng` | 用 `schema`，初始化不生成 `database` |
| 人大金仓（Pro） | `@nocobase/db-kingbase` | `kingbase` | 须 `DB_MODE=pg`，仍用本方言与驱动 |
| OceanBase CE（Pro） | `@nocobase/db-oceanbase` | `oceanbase` | MySQL 兼容租户；TEXT 默认值由 Repository 在创建时填充，直接 SQL 插入须自行给值 |

商业驱动经 Pro registry：`pnpm login --registry="<url>" --scope=@nocobase`，令牌作密码，不写入文档/参数/Git。`config init --dialect … --json` 生成配置和密钥但不连库、不装驱动、不建账号。改已有 `config.yml` 时保留原密钥。集成测试曾用 PG16、MySQL 8.4、MariaDB 10.11、SQL Server 2022、Oracle Free 23，这不是完整兼容范围承诺。

### 包索引（按职责，非必装清单）

运行时：`app-client`、`app-server`、`app-cli`、`app-websocket`。基础库：`api-client`、`authorization`、`caching`、`config`、`db`、`drive`、`i18n`、`jobs`、`lifecycle`、`logging`、`mailer`、`queue`、`realtime`、`repository-input`、`secrets`、`service-provider`、`session`、`snowflake`、`document-extractor`、`markdown-mermaid`。工具：`app-skills`、`app-testing`、`db-testing`、`db-testkit`、`dev-config`、`create-app`、`create-plugin`。部署：`app-installer`、`app-host`、`app-host-docker`。Agent/远程命令：`acp-bridge`、`agent-protocol`、`agent-runner`、`app-cli-client`、`cli-envelope`。创建插件：`pnpm create @nocobase/plugin@beta`。

### FAQ 要点

功能写应用还是插件、安装后无页面、组件不必固定结构、不必为复用而发 UI Registry、本地 UI 资产不必启 Registry 服务、ServiceProvider 非服务端专用、config 不生效原因、初始管理员密码不改已有账号、命令与教程不符时先确认是 v3 且入口正确、建表不自动生成 API、校验和不一致时通常新增迁移、换方言不搬数据、包版本不必强行对齐、`storage/` 不能当构建缓存删、反馈问题时提供脱敏错误与包版本而不提供密钥。详见 `reference/faq`。

**出处：** `reference/api`，`cli`，`configuration`，`project-conventions`，`packages`，`plugins`，`compatibility`，`database-configuration`，`faq`
