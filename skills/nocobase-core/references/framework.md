## 4. 框架

应用组装客户端、服务端、CLI 运行时；插件通过公开入口扩展。两端服务契约相同，容器、实例与生命周期各自独立；共享 Token 不会共享内存，跨端走 API。依赖安装、运行端注册和最终业务接入须分别确认：客户端注册不自动启用服务端，服务端注册也不会给页面安装组件。导入插件声明时不应该读业务数据、启动任务或建立连接；声明选项只放稳定组装参数，部署凭据放服务端配置，不能放入客户端插件选项。`plugin inspect` 只读静态接入面，不能证明业务已工作。复用宿主实例：数据库 Token 来自 `@nocobase/db`，缓存/文件/任务等宿主 Token 来自 `@nocobase/app-server/*`。独立脚本才自行创建并 `destroy()`/`dispose()`，不要在每个请求里创建。

### 运行时

- 客户端：`client/runtime.ts` → `resolveAppRuntime()` → `ClientApplication` → `app.start()` → `AppClientRoot`（Providers + routes）。宿主须等 `start()` 成功再渲染。
- 服务端：配置、Provider、路由、数据资源、插件由运行时接入。用 `app.paths` 解析路径。运行数据用持久位置，不写入可替换的构建目录。不要把“当前用户”写入单例。
- CLI：按 `package.json` 的 `nocobase.templateKind`（源码）或 `nocobase.buildTarget`（产物）识别；`NOCOBASE_APP_ROOT` 指定目标但不改 cwd。只有业务依赖完整生命周期时才 `app.start()`。面向程序的数据进 JSON `result`，进度走 stderr。
- 生命周期：按 Provider 钩子顺序启动与关闭；清理只关自己拥有的资源。`NOCOBASE_STRICT_STARTUP` 使启动失败严格退出。

### 服务容器

`createServiceToken<T>(name)` 每次创建新对象，字符串相同也不是同一 Token，必须导入提供方原始 Token。同一容器不能重复绑定。`instance` / `singleton`（同步工厂，可接收 resolver）/ `resolve`（缺失或循环依赖抛错）/ `has` / `resolveIfCreated`。工厂失败后绑定保留失败状态。无请求作用域。

客户端：`useService(token)`；服务端：`app.container.resolve(token)`。

### HTTP（框架层）

中间件用 `defineHttpMiddleware({ name, register })` 声明，应用组装阶段调用 `app.addHttpMiddleware(...)`，且应在启动和路由注册之前完成。`register` 收到应用根 Router，`/api/*` 是相对于应用的路径，宿主负责外部基础路径适配。先执行 `await next()` 前的代码，返回后再执行其后代码；拒绝时直接返回或抛 `ApiError`，不再调用后续。身份必须先于依赖身份的授权。避免把请求体消费两次（尤其流式上传）。进程内限流计数不是多实例全局计数。不要把未知异常包装成成功响应。

React Provider 用 `defineClientReactProviders([{ name, layer, component }])`（`layer` 如 `'extension'`），按 `layer` 与显式 `before`/`after` 从外到内排列。解析声明不会执行组件；`app.start()` 后渲染 `AppClientRoot` 才挂载。不要在 render 中新建共享连接。

Repository 数据接口用于“一个 Repository 操作即可完成”的列表/详情/字段编辑，须配置 `policy`；涉及复杂状态、多权限范围或副作用时用自定义路由。

```ts
import { defineRepositoryApiRoutes } from '@nocobase/app-server/router';
export default defineRepositoryApiRoutes({
  repositories: [{
    name: 'publicCategories',      // HTTP 名，亦为 api.repository(name)
    collection: 'categories',      // 省略则与 name 相同
    // connection 省略则用默认连接
    actions: { findMany: { maxLimit: 50 }, findOne: {}, count: {} },
    // 还可启用 exists、createOne、updateOne、deleteOne、aggregate、groupBy
    policy: { read: { scope: true, fields: ['id', 'title'] }, create: false, update: false, delete: false },
  }],
});
```

`policy` 也可按请求身份生成。`principal` 用于解析身份。HTTP 调用示例为 `POST /api/publicCategories/findMany`。流式响应一旦开始发送，不能再改成普通 JSON 错误。超时不自动撤销已提交的库操作。

### 数据库（框架层）

入口：DatabaseManager / Connection；CollectionBuilder；Repository；Query；Collections / SchemaInspector；Migrator / Seeder。业务通常从 Repository 开始。Migration 里用 Builder 改结构、用 Query 做集合式回填。底层驱动能力不自动成为各方言公共能力。

```ts
const database = app.container.resolve(databaseManagerToken);
const orders = database.repository('orders');
// db.connection(name?)、connect、builder（方法）、query、collections
// db.transaction(callback, name?)、destroy()
// CollectionBuilder 仅作类型导出，不要 new CollectionBuilder()
// connection.builder 是属性；db.builder() 是方法
```

`context` 查询选项是变量上下文，不是事务句柄。Builder 的 `transaction()` 当前不开真实事务。

**建模：** 框架不自动附加 `id`/`createdAt`/`updatedAt`。`increments()` 才是自增主键，字段仅名为 `id` 不会成为主键。`createCollection(name, callback | { fields, indexes, constraints })`，不能把字段数组当第二参数。字段：`string`/`text`/`boolean`/`integer`/`decimal`/`bigInt`（超安全整数用十进制字符串）/`date`/`time`/`datetime`/`datetimeTz`/`json`/`enum`/`char`/`float`/`double`/`blob`/`uuid`/`native`。默认值不回写旧记录。`alterCollection`/`alterField`/`addField`/`dropField`/`dropCollection`/`renameCollection`（有关系/外键/视图依赖时可能拒绝；不要用删建模拟重命名）。Builder 使用逻辑名；物理名由 `underscored`、`tablePrefix` 决定，不要向 Builder 填 `tableName`/`columnName`。`dryRun`/`previewSql` 的操作级预览 ≠ `db apply --dry-run`。`strict: true` 让能力警告变错误，不是删除确认。`ifNotExists`/`ifExists` 不校正已有对象结构。external 连接不能执行 Builder DDL。

**关联：** 必须显式指定键，不推断。`belongsTo`：`foreignKey`+`targetKey`；`hasOne`/`hasMany`：`sourceKey`+`foreignKey`；`belongsToMany`：`sourceKey`+`targetKey`+`through`+`foreignKey`+`otherKey`。关联声明不授权、不自动建中间表。只有 `constraints: true` 才自动建外键；已有显式外键时用 `constraints: false`。`belongsTo` 默认补索引，可用 `index: false`。`onDelete`/`onUpdate`：`restrict`/`cascade`/`set null`/`set default`/`no action`，能力因方言而异。

**Repository：** `findMany` 惰性，同一对象不要混合 await 与异步迭代或重复消费；`findOne` 不证明唯一；`updateOne`/`deleteOne` 必须恰好一条否则失败；`upsertOne` 筛选须唯一定位；批量写/删须筛选或 `all: true`。`createOne` 等从 `.record` 取记录。金额用精确字符串。`select` 限制字段；不设 select 通常读全部标量，仍不可泄漏密码。需要权限时 `withPolicy()`。

**Query：** 面向物理表与结果集，不读 Collection 字段映射或表前缀。`updateTable`/`deleteFrom` 要求条件，全表须 `allowAllRows()`。

**过滤、排序、分页：** 列表、计数和导出应基于同一授权范围。简单等值可写 `{ status: 'draft' }`；`{ amount: { gte: 100 } }` 不是当前等值简写。空筛选直接省略，不传空对象。复杂条件用 Filter Builder（`f.and`/`f.or`，`f.string('title').includes(q)` 等）或 JSON AST。须限制 page/pageSize（示例拒绝非安全整数或 pageSize>100）。列表与 `count` 是两次查询，不天然同一快照。`cursor` 是排序值的排他边界，不要求边界记录仍存在；必须显式排序，轴只能用非空直接标量字段，并含完整主键或唯一约束作决胜项；游标键须与最终排序轴一致，不能与 offset 同用。`direction: 'backward'` 取边界之前最近一页，返回仍按原 sort 顺序。返回没有自动 pageInfo/nextCursor。分页不保证跨请求快照。

**查询输入：** `@nocobase/repository-input` 的 `buildFilter`/`buildSelect`/`buildSort`/`buildAggregate` 把回调或 AST 物化为可序列化 JSON；也可 `buildCreateValues`/`buildUpdateValues`，远程请求用 API Client 的 `build*Options`。构造输入不连库、不验证字段权限。函数不能直接经 JSON 传输。日期转 ISO，bigint 转十进制字符串。不要使用 `internal/*`。服务端仍须核对 Collection、字段、关系深度、分页和策略。

**聚合：** `aggregate()` 返回别名到数值的对象，不带 record 包装；`groupBy({ by, aggregate, having, sort })` 返回含分组字段与聚合别名的数组。先 filter，再分组，再 having。`count()` 空集为 0；`count(field)` 忽略 SQL NULL；`sum`/`avg` 对整数与 DECIMAL 保留精度字符串，浮点返回 number；`min`/`max` 保留逻辑类型；空集的 sum/avg/min/max 为 `null`。不要对金额或大整数使用 `Number()`。`by` 必须非空且不重复；聚合别名不能覆盖分组字段；having/sort 不能引用未分组字段。统计与明细使用同一权限范围。

**关联查询：** `select: (s) => s.fields(...).include('orders', (orders) => orders.fields(...).filter(...).sort(...).limit(10))`。关系局部 filter 只影响关联内容，不会移除没有匹配子记录的父行；筛父行需根级关系条件。belongsTo/hasOne 无匹配为 `null`，hasMany/belongsToMany 为 `[]`。不传 select 时默认读标量、不自动展开关系。`fields()` 无参数表示零个根字段。重复 include 同一关系会被拒绝。to-many 支持局部排序、limit、游标；to-one 不支持这些。关系 Builder 不可变，必须使用链式返回值。`combine()` 可在同一关系下分别返回记录、计数和求和，各分支条件与分页独立。局部 limit 不保证数据库只读取每个父记录的限定行数。关联不是绕过目标记录权限的通道。

**连接：** 一个管理器可含多个连接，各有方言、事务和模型解析。`db.repository('events', 'analytics')` 或 `db.connection('analytics')` 选命名连接。核心管理器不自动扫描方言包；独立代码须显式提供方言工厂或在 `drivers` 注册后用声明式配置。应用运行时按配置异步加载已安装官方方言。连接不是跨库事务协调器。不要在每次请求中创建连接池。

**事务：** 回调内必须用 `connection.repository()`，不要用外层 `db.repository()`。回调返回后连接结束。`afterCommit()` 在回滚时丢弃；其抛错不改变已提交结果。嵌套成功时回调并入最外层提交。需要可靠投递应持久化待发送记录。一个连接事务不覆盖其他连接或外部系统。

**迁移执行：** 只跑无成功历史的迁移，本次记入同一新批次，但各任务自有事务。`transaction: 'auto'`/`true` 使用连接事务；`false` 失败可能留下部分修改。不能假设所有方言 DDL 可回滚。`shouldRun` 在锁内、任务事务前执行，返回 `false` 不写成功历史；`--dry-run` 不执行该回调。`db rollback` 回退最新整批（可能含应用+插件），任一条不可逆则整批不能开始；非交互需 `--force`。`db redo` 回退后重做迁移与 Seed，重建表不等于已执行 Seed 会重装。`db reset` 删除受管理结构和历史后从空库安装，不调用 `down()`，只用于可丢弃数据。`db repair` 只修历史校验和，不跑迁移、不修结构。迁移 Context 的 `connection` 不是完整运行时连接；不要从外部 DatabaseManager 另取 Repository（会离开任务事务）。Context `container` 在应用层目前只允许 `idGeneratorToken`。迁移/Seed 写入不触发运行时 Repository 变更事件。

**Seed：** `name` 与文件名一致；执行历史与失败重试见原页。示例数据用 `sample: true` + `db sample`（仅开发）。

### 后台、实时与支撑

- Jobs（`jobExecutorServiceToken`，`@nocobase/app-server/jobs`）：`Job` 子类，`static jobName`；载荷须可 JSON 持久化，不要传连接、容器、函数、`Date`、`bigint`、`undefined`（日期用 ISO 字符串）。所有消费进程须在 `setup()` 前 `registerJob`。回执 ≠ 处理成功。开发可用 memory，跨进程用 Redis（配置名与 queue 的 `inMemory`/`redis` 不同）。任务提交不与业务库事务原子绑定。
- 调度：`getScheduleExecutor(scope)`，`cron`（五/六段，默认 UTC，应显式 `tz`）与 `every`（毫秒）二选一；`startDate`/`endDate`/`limit`/`immediately`。更名不会自动删除旧规则；`removeJob()` 不要在每次进程关闭时调用。
- Queue（`queueServiceToken`）：多数业务先用 Jobs；需通道、延迟、优先级、批量或限流再用队列。`producer(name).publish(channel, payload)` 回执 ≠ 消费完成。同进程处理器并行，任一失败可能整条重试。`withChannel()` 只筛选通道。不要在处理器里等待注销自身。
- Realtime（`realtimeServiceToken`）：先 `defineTopic`/`defineTopicFamily`；`audience: 'user'` 只投递给连接认证后的指定用户。`publishFor` 的用户 ID 须服务端已确认。事件断线不补发；权威状态在数据库和 API。默认 `inMemory` broker 仅当前实例，多进程用 Redis Pub/Sub。页面用 `useRealtimeRefresh(topic, refresh, { delayMs })` 或 `useRealtimeTopic()`；不要为每个组件建连接。路径内部为 `/ws`，对外可能含挂载前缀。
- Drive（`driveManagerToken`）：`drive.use()`/`use('local')`；驱动 `fs`（`location`、`visibility`、可选 `url`）、`s3`、`oss`。`visibility: 'private'` 不会自动配 OS 权限或保护公开 HTTP 目录；`url` 只参与 URL 生成。`put`/`get`/`delete`。多实例用本地盘需共享卷或改对象存储。
- Caching（`cachingToken`）：`getCache({ namespace, defaultTtl })`，`wrap`/`get`/`set`/`delete`/`take`。缓存不能当唯一数据源；键须覆盖租户/用户/权限维度。内置 `memory` 仅当前进程。不能只把配置改成 `redis` 就获得未安装的驱动。`wrap` 合并不是分布式锁。布隆过滤器可能误判存在。
- Logging（`loggingToken`）：`getLogger(source)` 写入 `logger` 字段，不自动分文件；默认文件名 `app`，目录 `storage/logs/`。文件始终 JSON Lines；`console.pretty` 只影响终端。一个日志目录由一个进程管理轮换。不要写密码、Cookie、Token、完整个人资料。
- Snowflake（`idGeneratorToken`）：`generate()` / `generateString()`。`workerId` 0–31（`snowflake.workerId` / `SNOWFLAKE_WORKER_ID`），多实例必须不同且不会自动协调。同秒序列用尽、时钟回退或超出安全整数会抛错，不静默重复。不要当会话密钥或 API Token。浏览器新 ID 应由服务端生成。

**出处：** `framework/overview`，`file-storage`，`runtime/client`，`runtime/server`，`runtime/cli`，`runtime/plugins`，`runtime/lifecycle`，`runtime/react-providers`，`services/container`，`services/tokens`，`services/integration`，`http/routes`，`http/middleware`，`http/validation`，`http/repository-routes`，`http/api-client`，`database/overview`，`database/connections`，`database/schema`，`database/relations`，`database/repositories`，`database/query-inputs`，`database/filtering-pagination`，`database/association-queries`，`database/aggregation`，`database/transactions`，`database/migrations`，`database/seeds`，`background/jobs`，`background/scheduling`，`background/queues`，`realtime/topics`，`realtime/websocket`，`support/caching`，`support/logging`，`support/ids`

---

