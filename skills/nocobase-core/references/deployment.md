## 6. 部署与运维

适用自己的 Core 应用。Docker：容器 + 镜像与持久化挂载。PM2（app-installer）：Linux 上托管 Node 进程，安装器管理安装、配置、版本切换、备份和回退。源码留在开发机，服务器运行配置和业务数据独立于发布版本。两种方式都用独立生产配置、数据库和文件存储；构建前确定平台、数据库和存储。

构建产物含页面、接口、迁移和运行依赖，**不会**自动携带开发数据库、上传文件或运行密钥。首次用新库会执行迁移和必需种子；示例应用可能初始化角色，但不会复制开发环境中的商品、用户或角色分配。通过管理页创建的其他角色也不会自动迁入。`node dist/server/standalone.js` 只能检查产物，正式部署还需进程托管、配置管理与数据持久化。

构建指定平台（原生依赖不能简单跨系统复制）：

```bash
pnpm build --target linux-x64 --node-version 24 --tar
# 归档写入 storage/exports/dist.tar.gz；目标如 linux-arm64-musl
```

**Docker 要点：** 开发机构建匹配架构的镜像，如 `docker build --platform linux/amd64 -t my-app:1 .`。部署目录含 `docker-compose.yml`、`config.yml`、`storage/`。启动：`docker compose -p my-app up -d --wait --wait-timeout 180`。健康检查示例：`curl --fail http://<IP>:13000/main/api/healthz`（保留应用路径）。容器内 CLI：`docker compose -p my-app run --rm --no-deps app node dist/cli/index.js config check --connect`（以及 `db apply --dry-run --json` 等）。更新：构建新标签，改 Compose 镜像版本并 `--force-recreate`，保留配置和存储挂载。`down` 删除容器但可保留主机配置与数据。更新前备份 `config.yml`、`storage`、`docker-compose.yml`。

**PM2 / app-installer：** 命令使用 `@nocobase/app-installer@beta`（或固定版本），在服务器执行；开发机负责构建、打包、上传。示例：应用名、独立目录/端口/进程名、端口 `13000`、路径 `/main`。服务器需 Node 24、全局 PM2 4.3+。首次安装设置 origin/host/port/name 等；**升级用 `upgrade`，不再传这些首次参数**，保留原配置、密钥、库和上传。预检可能返回 `CONFIRMATION_REQUIRED`（此时尚未切换）；重复同包可为 `success-noop`。安装器自动备份可识别的 SQLite、`config.yml`、`app.env`；上传文件、外部库、对象存储须另行备份。`--backup-done` 只确认外部库已备份，不执行备份。若 `CHECKSUM_REPAIR_REQUIRED`，普通 `--yes` 不会授权修复，须按计划加 `--repair-checksums <token>`。错误：`PORT_IN_USE`（换端口并同步 origin，不要停无关进程）、`DRIVER_MISSING`（在源码加驱动并重建）、`START_FAILED`（看返回的日志路径和安装目录 `logs/`）。

发布新版本：沿用生产库、配置和文件存储；涉及结构变更时把新迁移随代码发布，保留已执行迁移；先备份，留存旧包或镜像。切回旧代码不一定能恢复数据库。先按升级 Core 文档完成框架升级再部署。

公网访问须先配域名、HTTPS 和反向代理，再登录管理员。文档中的 IP+HTTP 示例仅用于可信网络验证。

**出处：** `deployment/choose-a-deployment-method`，`deployment/publish-app-release`，`tutorials/deploy-with-docker`，`tutorials/deploy-with-app-installer`；升级另见 `developing-apps/upgrade-nocobase-core`

---

