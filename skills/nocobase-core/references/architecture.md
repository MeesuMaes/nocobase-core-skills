## 5. 应用架构

多门户、多应用、多租户可组合：多入口共用一套应用与版本用**多门户**；要独立数据/配置/版本用**多应用**；面向多客户时可为每租户一个独立 App（租户 App 内仍可多门户）。统一登录 ≠ 拥有所有应用权限，也不会自动打通业务数据。多门户管理、门户生成器、多门户模板，以及多应用相关插件，属于商业能力。

**多门户：** 同一应用内的业务入口，共享登录与运行环境，随应用统一升级；数据访问仍由权限控制，也可用多数据库连接。创建与开发见 `architecture/multi-portal/create-app`，`develop`，`access`，`manage`。

**多应用：** **Main** 是平台入口（工作台、应用管理、平台身份）；**子应用** 有自己的页面、API、数据库、配置和业务权限。平台决定能进哪些应用，子应用决定进去后能做什么。平台管理员不会自动成为子应用管理员。项目模板常把 Main 放根目录、默认子应用在 `apps/sub/`，共享依赖与锁文件，但分别开发、构建、部署；子应用也可在独立仓库。

平台中的**应用模板**管理某业务的部署版本；**版本**是一次构建的部署归档；**实例**是使用某版本运行的应用（如东/西门店）。发布新版本不会自动升级已有实例。托管应用由平台经 Releases 与 App Host 创建实例；独立接入则自行部署，平台提供入口和可选统一登录。源码位置不决定部署方式。托管时 Main 使用子应用部署归档，不直接跑其业务源码。

所需商业插件（项目模板不含实现也不提供授权）：`@nocobase/app-plugin-multi-app`（Main）、`@nocobase/app-plugin-identity-provider`（Main）、`@nocobase/app-plugin-platform-sso`（需平台登录的 NocoBase 子应用）。统一登录 Web 回调要求可信 HTTPS 和非回环主机名，不能用 `localhost`。启用平台身份后，新实例会登记登录客户端，但不会自动允许所有平台用户进入。只拥有应用访问资格不能因此管理实例。

运维要点：更新 Main 不会发布子应用；发布子应用版本也不会自动升级实例。实例「归档/恢复归档」是管理状态，不是数据库备份/恢复。代码回退不会自动撤销已执行迁移。不要只备份 Main 库。更换平台域名或挂载路径时，还要处理已保存的环境地址、回调、独立入口和身份绑定。外部应用接入 SSO 须支持授权码、PKCE S256、`client_secret_basic` 及标准 issuer/audience/签名校验；用户映射与业务权限由第三方负责。演练环境不要发生产通知或与生产争抢队列。

**出处：** `architecture/overview`，`architecture/multi-portal/create-app`，`develop`，`access`，`manage`，`architecture/multi-app/overview`，`quickstart`，`development`，`management`，`deployment`，`single-sign-on`，`external-apps`，`migration`

---

