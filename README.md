# NocoBase 3 Core Skills

这是一套按 [Agent Skills](https://agentskills.io/specification) 规范打包的 NocoBase 3 Core 答疑技能。内容提炼自 [nocobase-core-docs](https://github.com/MeesuMaes/nocobase-core-docs)（文档站点：https://meesumaes.github.io/nocobase-core-docs/），把 134 篇 Core 中文文档收成一个技能：`skills/nocobase-core`。

技能本身只提供答疑依据，不代替官方文档。回答时应给出文档站点路由作为出处；手册不够时，再打开 `https://meesumaes.github.io/nocobase-core-docs/<route>.md` 核对原文。

## 仓库结构

```
.
├── LICENSE
├── README.md
├── .claude-plugin/marketplace.json
└── skills/nocobase-core/
    ├── SKILL.md
    └── references/
        ├── developing-apps.md
        ├── using-plugins.md
        ├── extension-development.md
        ├── framework.md
        ├── architecture.md
        ├── deployment.md
        └── reference.md
```

`SKILL.md` 里是触发说明、问题类型到文档路由的索引，以及答题规则。七个分组的细节在 `references/` 中，按需打开，不必一次读完。

## 安装

### 直接复制

把 `skills/nocobase-core` 目录复制到本机技能目录，文件夹名保持 `nocobase-core`：

```bash
# Claude Code
cp -R skills/nocobase-core ~/.claude/skills/nocobase-core

# Cursor
cp -R skills/nocobase-core ~/.cursor/skills/nocobase-core
```

复制完成后，在对话里提到 NocoBase 3 的应用开发、插件、框架 API、架构或部署问题时，代理应能自动选用该技能。

### 作为 Claude Code 插件市场安装

在 Claude Code 中把本仓库登记为插件市场，再安装插件：

```
/plugin marketplace add MeesuMaes/nocobase-core-skills
/plugin install nocobase-core@nocobase-core-skills
```

## 来源与许可

本仓库采用 MIT License，与 [nocobase-core-docs](https://github.com/MeesuMaes/nocobase-core-docs) 相同。见 [LICENSE](./LICENSE)。

- 整理与结构化：MeesuMaes（Copyright (c) 2026）
- 文档内容源自 NocoBase 官方文档：https://docs.nocobase.com/
- 中文镜像仓库：https://github.com/MeesuMaes/nocobase-core-docs
- 文档站点：https://meesumaes.github.io/nocobase-core-docs/

NocoBase 3 目前处于 beta，各包独立编号。手册中的接口基线以镜像仓库当时的 Core 修订为准，实际项目请以当前锁文件和本地类型声明为准。
