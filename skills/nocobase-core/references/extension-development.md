## 3. 开发插件

只服务当前业务、常随页面改的代码，直接放应用的 `client/`、`server/`、`database/`、`cli/`。需独立维护、提供扩展入口或供多应用接入时，再拆 `packages/app-plugin-<name>/`。本地插件随应用构建，不必先发 npm。

### 创建

```bash
pnpm create @nocobase/plugin@beta <name> --with <capability>
```

`--with` 是要生成的代码能力，不是安装已有插件。必须至少一种能力，或 `--empty` 只生成包结构。多个能力重复 `--with`，不用逗号拼接。

| `--with` | 生成 | 不会自动做的事 |
| --- | --- | --- |
| `client.routes` | 路由骨架与测试 | 不自动生成业务页面 |
| `client.components` | 示例组件与导出 | 需自己实现界面 |
| `client.service-providers` | 客户端 Provider 骨架 | 服务与生命周期 |
| `client.react-providers` | Context 与声明 | 状态与组合 |
| `client.locales` | 语言文件 | 实际译文 |
| `server.service-providers` | services/providers/tokens | 业务逻辑 |
| `server.routes` | 路由骨架 | 不自动生成 CRUD |
| `server.jobs` | 示例 Job 与执行器 Provider | 提交点、幂等、重试 |
| `server.locales` | 服务端语言文件 | 实际译文 |
| `database` | migrations/seeds 的 `.ts.example`、资源声明 | 改为 `.ts` 后才纳入任务 |
| `cli` | 示例 `info` 命令 | 实际命令 |
| `skills` | `skills/<名>/SKILL.md` 草稿 | 按真实实现完善后再同步 |

有任一项 client.*（除单独 components 的细则见原文）会推导 `client/index.ts`、`client/plugin.ts` 与 `./client` 导出；server 侧类似。不必写 `--with client` / `--with server`。带客户端时还会生成 `components.json` 与 `client/styles.css`。

创建后在应用中 `plugin register <包名>`（先 `--dry-run`），并检查三端注册文件。

### 结构与声明

插件按需提供 Client、Server、CLI、database、skills，不要求目录齐全。链路：

```text
client: pages → routes.ts → plugin.ts → index.ts → package.json 的 ./client → 应用 client/plugins.ts
server: routes/providers → plugin.ts → index.ts → ./server → server/plugins.ts
cli:    defineCliPlugin() 默认导出 → ./cli → cli/plugins.ts
```

```ts
// client/plugin.ts
import { defineClientPlugin } from '@nocobase/app-client/plugins';
export default defineClientPlugin({
  packageName: '@nocobase/app-plugin-announcements',
  routes,
  locales,
  // 还可贡献 serviceProviders、react-providers 等，以声明文档为准
});

// server/plugin.ts
import { defineServerPlugin } from '@nocobase/app-server/plugins';
export default defineServerPlugin({
  packageName: '@nocobase/app-plugin-announcements',
  baseDir: path.resolve(import.meta.dirname, '..'),
  serviceProviders,
  routes,
  locales,
  database: { migrations: './database/migrations', seeds: './database/seeds' },
});
```

跨插件调用必须走公开导出与原始 Token，并在依赖中声明该包；不能相对引用旁边插件的私有实现。UI 作为可编辑源码资产时，用独立 Registry 包（`registry/`、`registry.json`），构建 item JSON 后交给 `shadcn add`；本地不必启动 Registry 服务。资产不能依赖发布者私有路径。

`AGENTS.md` 面向维护插件源码的 Agent；`skills/` 面向在应用中使用该插件的 Agent。`.agents/skills/` 是同步副本。

### ServiceProvider

前后端都可用，容器各自独立。`register()` 同步注册；`boot()` / `start()` / `ready()` / `shutdown()` 为异步钩子。使用宿主 `app.container`，不要另建全局容器或销毁宿主共享的 DatabaseManager、Queue、Logging、Caching。工厂是同步的，异步准备放在生命周期里。

### 插件数据与命令

插件 `database/migrations`、`seeds` 经 `server/plugin.ts` 的 `database` 声明由宿主发现。同一连接上应用与插件迁移合并后按 **名称** 排序，插件注册顺序不是迁移顺序。重名会检查失败。构建后生成 `.manifest.json` 保留源校验和，不手改。命令命名空间通常由包名去掉 scope 与 `app-plugin-` 得到，冲突报错，不能靠注册顺序覆盖。开发命令（`devCommands`）只在源码环境保留。

发布、测试与 Skill 写法见 `publish-plugin`、`testing-and-verification`、`write-plugin-skills`。第一个完整示例见 `write-your-first-plugin`。

**出处：** `extension-development/when-to-write-a-plugin`，`plugin-development-guidance`，`create-plugin`，`plugin-structure`，`plugin-declarations`，`write-your-first-plugin`，`service-provider`，`client-services`，`server-services`，`server-apis`，`pages-and-menus`，`frontend-components`，`react-context-and-providers`，`database-and-migrations`，`background-jobs`，`extend-command-line`，`localization`，`server-localization`，`write-plugin-skills`，`publish-plugin`，`testing-and-verification`

---

