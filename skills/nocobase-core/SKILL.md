---
name: nocobase-core
description: 解答 NocoBase 3 Core 问题：应用开发、插件使用与开发、框架 API（HTTP/数据库/任务/实时）、应用架构、部署运维、CLI/配置/包索引与兼容性。在用户询问 NocoBase 3、创建应用、页面路由、权限、迁移、插件、ServiceProvider、部署，或查阅命令与配置时使用。依据 https://meesumaes.github.io/nocobase-core-docs/ 作答并给出出处路由；文档未记载的 API 或配置项不得编造。
license: MIT
metadata:
  author: MeesuMaes
  version: "1.0"
  source: https://github.com/MeesuMaes/nocobase-core-docs
---

# NocoBase 3 Core

本技能把 [NocoBase 3 Core 中文文档](https://github.com/MeesuMaes/nocobase-core-docs) 提炼为答疑手册，覆盖应用开发、插件使用与开发、框架、应用架构、部署运维和参考速查。先用本页定位问题并打开对应 `references/` 文件，手册不够再读文档站点原文核对细节与签名。

## 文档来源与环境

面向答疑：先用本手册定位，再打开出处页核对细节与签名。只陈述文档已写明的事实，不编造 API、配置项或命令。文档站点：`https://meesumaes.github.io/nocobase-core-docs/<route>`，原文 Markdown：同路径加 `.md`。官方原文：https://docs.nocobase.com/zh-CN/core/。核对此镜像时，常用接口基线为 Core 修订 `015404ecd0db`（2026-10-26/27），插件清单另有 `db419f1d4`；以当前应用锁文件与本地类型为准。NocoBase 3 处于 beta，各包独立编号，主版本号不等于产品版本。

运行环境：Node.js 24+；pnpm 版本以应用 `package.json` 的 `packageManager` 为准（文档示例为 pnpm 11 / 11.11.0）。源码根目录用 `pnpm nocobase …`；部署产物用 `node dist/cli/index.js …`。创建应用：`PNPM_CONFIG_MINIMUM_RELEASE_AGE=0 pnpm create @nocobase/app@beta my-app --template-tag=beta`（该环境变量仅让此次创建可安装刚发布的包）。Windows 使用 WSL，命令在 POSIX shell 中执行。

## 如何答题

按问题类型打开对应页面；手册不够就读原文，并在回答中给出相对路由作为出处。接口输入、响应和权限以运行中应用的 `/api/swagger`、`/api/swagger/docs` 为准（受身份与权限约束，默认不向匿名用户开放）。路径相对于应用挂载前缀（如 `APP_BASE_PATH=/main` 时，`/api/users` 实际为 `/main/api/users`）。

| 问题类型 | 先打开 |
| --- | --- |
| 目录该放哪、生成文件能不能删 | `developing-apps/project-structure`，`reference/project-conventions`，`reference/faq` |
| 配置不生效、密钥、环境变量、公开给浏览器的配置 | `developing-apps/configuration-and-settings`，`reference/configuration` |
| 命令怎么写、开发/部署命令差异、JSON 输出 | `reference/cli`，`framework/runtime/cli`，`developing-apps/application-commands` |
| 页面、菜单、路由、登录要求、覆盖插件页 | `developing-apps/pages-routes-and-navigation`，`developing-apps/page-layouts` |
| UI、shadcn、主题、组件复用、UI Registry | `developing-apps/pages-and-interactions`，`developing-apps/themes-and-styles`，`extension-development/frontend-components` |
| 客户端请求、状态、远程 Repository | `developing-apps/data-requests-and-state`，`framework/http/api-client`，`reference/api` |
| 服务端接口、中间件、校验、错误 | `developing-apps/apis-and-services`，`developing-apps/request-validation-and-errors`，`framework/http/routes`，`framework/http/middleware`，`framework/http/validation` |
| 自动 CRUD 接口 | `framework/http/repository-routes`（建表不会自动开放 HTTP） |
| 认证、角色、数据范围、withPolicy | `developing-apps/authentication-and-permissions`，`using-plugins/authorization/*` |
| 建表、迁移、Seed、校验和、回退 | `developing-apps/data-models-and-migrations`，`framework/database/schema`，`framework/database/migrations`，`framework/database/seeds` |
| 查询、关联、聚合、事务 | `developing-apps/querying-data`，`framework/database/repositories`，`framework/database/relations`，`framework/database/transactions` |
| 多连接、已有库、换方言 | `developing-apps/multiple-databases`，`developing-apps/existing-databases`，`reference/compatibility`，`reference/database-configuration` |
| 后台任务、定时、队列 | `developing-apps/background-tasks`，`framework/background/jobs`，`framework/background/scheduling`，`framework/background/queues`，`using-plugins/automation/scheduler` |
| 工作流/状态机 | `developing-apps/business-workflows`（包 `@nocobase/lifecycle`） |
| 文件存储与上传访问 | `framework/file-storage`，`using-plugins/data-and-files/file-management` |
| 实时推送、WebSocket | `framework/realtime/topics`，`framework/realtime/websocket` |
| 缓存、日志、Snowflake ID | `framework/support/caching`，`framework/support/logging`，`framework/support/ids` |
| 多语言 | `developing-apps/localization`，`developing-apps/server-localization`，`using-plugins/application-support/i18n` |
| 装/卸/升级插件，为何没有页面 | `developing-apps/manage-plugins`，`using-plugins/overview`，`reference/plugins`，`reference/faq` |
| 某内置/商业插件怎么用 | `using-plugins/` 对应篇，包名见 `reference/plugins` |
| 该不该写插件、怎么创建 | `extension-development/when-to-write-a-plugin`，`extension-development/create-plugin`，`extension-development/plugin-structure`，`extension-development/plugin-declarations` |
| 升级 Core、包版本为何不同 | `developing-apps/upgrade-nocobase-core`，`reference/faq`，`reference/packages` |
| 部署、发布新版本、健康检查 | `deployment/choose-a-deployment-method`，`deployment/publish-app-release`，`tutorials/deploy-with-docker`，`tutorials/deploy-with-app-installer` |
| 多门户 / 多应用 / 多租户 | `architecture/overview`，再进 `architecture/multi-portal/*` 或 `architecture/multi-app/*` |
| 公开 API 速查、从功能找包名 | `reference/api`，`reference/packages` |
| 测试怎么写 | `developing-apps/testing-and-acceptance`，`extension-development/testing-and-verification` |

答题规则：包已安装 ≠ 已注册 ≠ 页面已接入 ≠ 迁移已执行。不要从依赖的 `src/` 或私有构建文件导入，只用 `package.json` 声明的公开入口。完整参数以所安装版本的类型声明为准。文档没写的 API 或配置项，直接说文档未记载。

按问题所属分组打开对应参考文件，不要一次读完全部：

- `developing-apps/*` → [references/developing-apps.md](references/developing-apps.md)
- `using-plugins/*` → [references/using-plugins.md](references/using-plugins.md)
- `extension-development/*` → [references/extension-development.md](references/extension-development.md)
- `framework/*` → [references/framework.md](references/framework.md)
- `architecture/*` → [references/architecture.md](references/architecture.md)
- `deployment/*`、部署教程 → [references/deployment.md](references/deployment.md)
- `reference/*` → [references/reference.md](references/reference.md)

## 规则

- 只陈述文档已写明的事实，不编造 API、配置项或命令。
- 回答中给出出处链接：`https://meesumaes.github.io/nocobase-core-docs/<route>`。需要对照原文 Markdown 时，使用同路径加 `.md`。
- 文档没写的 API 或配置项，直接说文档未记载，不要自行补全。
- 接口输入、响应和权限以运行中应用的 `/api/swagger`、`/api/swagger/docs` 为准（受身份与权限约束，默认不向匿名用户开放）。
- 完整参数以所安装版本的类型声明为准；核对此镜像时的常用接口基线见上文修订号，但仍以当前应用锁文件与本地类型为准。
