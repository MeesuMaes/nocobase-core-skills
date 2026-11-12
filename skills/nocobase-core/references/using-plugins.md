## 2. 使用插件

先确认应用实际注册的插件与版本，再按篇接入。包已安装不等于已启用；部分插件还需应用组合工厂，或由应用声明业务数据、页面和权限。各篇 `/client`、`/server` 表示该包公开导出。使用文档四章：提供的能力、场景与示例、原理与实现、AI 开发约束与验证。

文档覆盖 21 个插件（主仓库核对 24 个应用插件 + 商业 2 个，见第 7 节）。按功能：

| 功能 | 路由 | 包（摘要） |
| --- | --- | --- |
| 用户 / 登录 / API Key | `using-plugins/authentication/users`，`sign-in`，`api-keys` | `app-plugin-users`（管理接口和页面）、`authentication`（登录与会话）、`api-keys`（内置） |
| 权限 / 默认范围 / 共享 / 限制 | `authorization/permissions`，`default-access`，`sharing-rules`，`restriction-rules` | `authorization`、`authz-default-access`、`authz-sharing-rules`、`authz-restriction-rules`（均内置） |
| 库浏览（只读） | `data-and-files/database-explorer` | `database-explorer`（内置） |
| 文件 | `data-and-files/file-management` | `file`（内置）；可编辑文件界面由 UI Library 提供 |
| 通知 / 渠道 / 站内信 / Web Push / 邮件 | `notifications/notification`，`providers`，`in-app`，`web-push`，`mail` | `notification*`（Web Push 非内置）；`mail`（内置） |
| Agents / 知识库 | `agents-and-knowledge/agents`，`knowledge` | 均在默认依赖中但默认不注册；`agent-sessions` 非内置 |
| 审批（商业） | `collaboration/approval` | `app-plugin-approval`；另有商业 `task-center` |
| 定时任务 / 模板打印 | `automation/scheduler`，`template-print` | `scheduler` 内置；`template-print` 仅 Skills |
| 多语言 / 应用底座 | `application-support/i18n`，`foundation` | 均内置 |

各插件文档中的开发约束（细节回各篇）：

- 用户：表单选项不能替代授权；接口仍检查 create 与 assign-role。邀请要区分“已创建账号”和“邮件投递失败”。删除不能当作物理清除用户历史。
- 登录：关闭注册后仍须检查直接注册请求；密码找回要验证实际投递与链接。业务接口在认证之后继续授权。
- API Key：管理密钥用登录会话，不能用 API Key 再管理其他密钥。无范围密钥 ≠ 只读；报表接口仍须落实操作和记录范围。轮换须立即使旧 secret 失效。
- 权限：先声明业务能力再授权；不能把未接入权限模型的查询当作已受保护。避免一个布尔 can 之后做无限制查询。Seed 角色时检查是否意外扩大现有用户权限。
- 默认数据范围：一个资源只维护一条默认规则；默认范围是追加范围，不是“仅无范围时生效的兜底”。规则只改变记录范围，不授予页面/操作/字段权限。
- 共享规则：按指定主体与记录保存；选择不支持 all，也不能顺带开放所有关联记录。
- 限制规则：选择表示仍允许的记录，多条取交集；删除规则可能扩大访问。
- 库浏览：只读；改表结构仍走迁移。展示连接摘要时不额外暴露主机、密码或数据库文件路径。第一页不是全部 Collection。
- 文件：检查内容路由和元数据接口各自的授权；默认 `accessPath` 不受 Repository policy 保护。上传只接受约定字段，业务归属由服务端处理，不信任客户端任意订单 ID。`deleteOne` 不等于物理清理。
- 通知：业务成功与投递分别处理；重试保留稳定事件标识，不能每次重试生成新幂等键。不能把用户 ID 当邮件地址。一次性凭据走 transient 路径，不能依赖其持久重试。
- 渠道：渠道名 ≠ `type`（`smtp` / `resend` 需原生邮箱与 subject 及 text 或 html；`feishu-webhook` / `dingtalk-webhook` 需 text，可选完整 HTTP(S) target URL）。YAML 不展开环境变量。不要为测试关闭提供方 Webhook 校验。该插件只有 Server 入口，无配置页。
- 站内信：接口使用当前登录身份，不能新增按任意 userId 的不受保护查询。实时事件只触发刷新，操作结果以 HTTP 为准。内部路径不能重复拼接 `/main`。
- 邮件：管理员邮箱概览不代替用户授权连接；participant 不扩大访问范围。结果未知时先核实 submission，避免重复发信。
- Web Push：不能每次启动生成新 VAPID 密钥；设备 API 不可暴露 endpoint 与加密密钥。URL 须指向应用内并含挂载前缀。`dedupeKey` 只替换可见通知，不替代事件幂等。排队成功 ≠ 已展示。
- 知识库：空间列表、读取与搜索使用同一访问边界；页面可进 ≠ 所有空间可读。写入携带预期版本，冲突时重新读取，不去掉版本检查强行覆盖。扫描件不可搜索不能伪造提取成功。
- 定时任务：声明计划 ≠ 已实现业务目标。不能把单进程 memory 验证当作分布式验收。`finalize` 仅使用完整清单，避免误停任务。
- 模板打印：包只提供开发指引（Skills），不得导入不存在的运行时入口或声称已有通用打印服务。下载接口须重新检查记录与字段权限。
- 审批：复用业务记录作为流程标识，不要维护脱节的第二份状态。可见按钮 ≠ 有权审批；服务端同时校验用户、记录、任务资格和内容版本。
- 多语言：语言选项由应用加载器决定，不另硬编码一份与资源不一致的列表。服务端回退不能被误认为浏览器切换已撤销。
- 应用底座：页面和接口保留 unrestricted 要求，不为让入口出现而放宽为“普通登录即可访问”。不能用底座统计代替业务测试。

**出处：** `using-plugins/overview` 及上表各路径；清单与是否内置：`reference/plugins`

---

