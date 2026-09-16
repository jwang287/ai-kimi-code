# Kimi Code 整体架构分析

> 本文基于对仓库源码的走读整理，覆盖 desktop / web / CLI 三种宿主形态、kap-server 服务器、agent-core-v2 引擎、transcript 数据层与持久化布局。以代码事实为准，所有引用均给出文件路径。

## 一、进程拓扑：单引擎库、三种宿主形态

agent-core-v2 引擎以**库**形式被三种进程各自内嵌，进程之间不共享引擎状态；只有 desktop/web 这条线存在真正的网络边界。

```
终端                     浏览器 / Electron 桌面端
┌──────────────┐   ┌────────────────────────────────────┐
│ kimi TUI     │   │ kimi-desktop (Electron, 薄壳)       │
│  └ node-sdk  │   │   └ spawn/attach sidecar ─────────┐│
│     └ klient │   │      (SEA 二进制 `kimi web`)       ││
│       /memory│   │           │                        ││
└──────┼───────┘   │  web UI (dist-web) ◀─REST+WS──┐    ││
       │           └───────────────────────────────│────┘│
       │                     kap-server (Fastify) ◀┘    │
       │                     └ bootstrap()             │
       ▼                       ▼
   ┌──────────────────────────────────┐
   │ agent-core-v2 (DI×Scope 引擎)     │ ← 各进程各自内嵌一份
   └──────────────┬───────────────────┘
                  ▼ 文件 I/O
        ~/.kimi-code/ (sessions, minidb, wire.jsonl…)
```

三个要点：

- **TUI 不走网络**：node-sdk 在同一进程内 bootstrap 引擎，经 `klient/memory` transport 调用（`packages/node-sdk/src/sdk-rpc-client-v2.ts`）。
- **desktop 壳零业务逻辑**：渲染进程直连 `http://127.0.0.1:<port>`，壳只负责 sidecar 生命周期管理（`apps/kimi-desktop/src/server/`）。
- **sidecar 契约**：kap-server 写 `~/.kimi-code/server/instances/<id>.json`（15s 心跳 + pid 探测，`packages/kap-server/src/instanceRegistry.ts`），desktop 轮询发现实例拿端口；token 从 `server.token` 读出后以 URL fragment `#token=` 传给窗口（不上行、不进日志）。
- 第三种形态 `kimi acp`（`apps/kimi-code/src/cli/sub/acp.ts`）走 stdio 的 Agent Client Protocol，同样 in-process bootstrap 引擎 + klient memory transport。

## 二、引擎核心：DI × Scope（agent-core-v2）

仿 VS Code instantiation 模型的依赖注入引擎：

- **三级 LifecycleScope**（`packages/agent-core-v2/src/app/scopes.ts`）：`App → Session → Agent`，通过 `createScopedChildHandle` 派生子容器，解析不到的服务向父 scope 冒泡；父→子上下文靠 `seeds` 注入（如 `ISessionContext`、telemetry 绑定）。
- **Workspace 不是 DI scope**：它是 App 级 `IWorkspaceInstanceManager` 持有的 `WorkspaceInstance = Program 组合根 + RuntimeRegistry`（`workspace/workspaceInstance/workspaceInstance.ts`）。每个 Program generation 手写组装 state/fs/git/mcp/trust/skills 等单元。
- **创建链**：`bootstrap()` → App scope（~128 个 OnScopeCreated 服务）→ `ISessionManager.create/resume()` → per-workspace `SessionLifecycleService.materializeSession()` → Session scope → `IAgentLifecycleService.create()` → Agent scope。
- **扩展点靠 collection contribution**：`collection(name)` 生成 token，多方 `provide` 汇聚为 `CollectionView`（工具 `AgentToolContribution`、配置节、命令、agent profile 都走这机制）。
- **Feature seam**：`src/features/` 下的特性包（skill/plan/todo/goal/swarm/cron/notify…）都是 `Feature extends Service`，不改核心、只往 collection token 供稿；`FeatureAssemblyService` 在 App scope 创建时统一实例化为 Fiber（带 Pending→Active→Unloading 状态机的单元容器）。

## 三、kap-server 内部分层

`packages/kap-server/src/start.ts` 严格串行组装：

```
instanceRegistry → auth(tokenStore/bcrypt) → remoteControlManager → bootstrap(engine)
→ telemetry → IWorkspaceService.list() → ISessionIndex.prepare()   ← listen 前预热
→ Fastify + middleware → OpenAPI → REST v1/v2 → WS v1/v3/debug → webAssets → listen
```

- **routes/ 是薄适配层**：zod schema + OpenAPI 文档，handler 内 `core.accessor.get(I…)` 直连引擎服务，统一 `{code,msg,data,request_id}` 信封。
- **services/ 是进程级对象**（不进 DI）：`TranscriptService`（transcript 生产者）、`ProjectionService`（v3 生产者）、`SessionEventBroadcaster`+磁盘 journal、`GlobalSearchService`→worker 线程。
- **两条 WebSocket 栈并存**（`start.ts` 按 URL 分流）：

| | v1 `/api/v1/ws` | v3 `/api/v3/ws` |
|---|---|---|
| 模型 | 事件流 + seq/epoch 游标 + 磁盘 journal 重放 | 内存实体投影，重放快照恢复 |
| transcript | 内嵌 `subscribe_v2`（transcript.reset/ops） | 不带，走 REST |
| 实际消费者 | web UI 主栈 | kimi-inspect（尚未替代 v1） |

- **debug 反射 RPC**（`/api/v1/debug/*`，仅 loopback + `--debug-endpoints`）：服务名直接来自 DI 注册表反射，`dispatcher` 对 live 会话按 scope 调服务方法；kimi-inspect 是其客户端。
- **auth 分两层互不相关**：HTTP/WS bearer token 是 kap-server 进程级的；模型 provider 的 OAuth 是引擎 App scope 的 `IOAuthService`。

## 四、数据层契约（packages/transcript）

纯 TypeScript、无引擎依赖的四层结构：

- **L1 store**：per-session `TranscriptStore`（per-agent 的 turn/frame/item 有序列表）。
- **L2 operations**：幂等 `TranscriptOperation` + `applyOperation` 归约器，op 批序契约唯一归属本包。
- **L3 订阅粒度**：`off / turn / block / delta`，降级裁流、升级重发 reset。
- **L4 view registry**：framework-free 渲染器注册表（tool / input-origin / marker），由前端注入。
- **turn-cursor 分页**：`{beforeTurn?, afterTurn?, pageSize}` → `{items, hasMore}`。

kap-server 的 `TranscriptService` 是它在进程内的宿主（live 会话绑定 scope 事件折叠进 store；冷会话直接读 wire.jsonl/state.json）；web UI 经 v1 WS `subscribe_v2` 消费。

## 五、客户端 facade（packages/klient）

三层严格分层：**Facade**（`global.* / session(id).* / agent(id).*`，唯一公开 API，无 service-locator 逃生口）→ **契约**（每个 wire 方法的手写 zod schema，编译期 parity 断言钉住引擎类型）→ **Transport**（`memory` 供 node-sdk/acp-server 进程内使用；`ipc` 走 unix socket NDJSON，仓库内暂无生产消费者，属预留）。

## 六、持久化全景（~/.kimi-code）

| 数据 | 路径 | 写方（scope） | 读方 |
|---|---|---|---|
| 会话元数据 | `sessions/<ws>/<sid>/state.json` | ISessionMetadata（Session） | SessionIndex 扫描、冷读 transcript |
| agent 会话记录 | `sessions/<ws>/<sid>/agents/<aid>/wire.jsonl` | IWireService（Agent） | 冷读、搜索 sync、messageHistory |
| 会话-工作区索引 | `session_index.jsonl` | SessionLifecycleService | WorkspaceService merge/aliases |
| 工作区目录 | `workspaces.json` | FileWorkspacePersistence（App） | IWorkspaceService |
| 会话索引读模型 | `<cache>/query-store`（minidb 16 分片） | SessionIndexProjector | 列表 API 读模型 |
| 全文搜索索引 | `search-index/`（minidb + 文本索引） | kap-server worker 线程 | /api/v1/search |
| WS v1 事件 journal | `server/events/<sid>` | SessionEventBroadcaster | 断线 seq 重放 |
| 服务器实例/令牌 | `server/instances/*`、`server.token` | instanceRegistry / tokenStore | auth hook |

关键设计倾向：**权威数据在文件（state.json / wire.jsonl），minidb 只是可重建的读模型/索引**。

## 七、横切能力挂载点

- **telemetry**：引擎侧 `ITelemetryService`（App scope），Session/Agent scope 创建时以 seed 注入 `session_id`/`agent_id` 上下文，各级 `track2` 自动带上下文；kap-server 注册 `CloudAppender`。
- **remote-control**：进程级、不进 DI（`start.ts` 创建 `RemoteControlManager`），隧道把外部 HTTP/WS 转发到本地 server。
- **实验特性开关**：`registerFlagDefinition` 在 owning domain 声明，`IFlagService.enabled(id)` 检查，env > config > master env > default。

## 八、架构观察与已知性能关联

1. **「引擎以库内嵌」+「listen 前全量预热」**：桌面端启动 = 进程启动 + DI 组装 + 索引预热串行叠加，desktop 窗口只能排在链路最后创建（`apps/kimi-desktop/src/main.ts`）。
2. **「文件为权威 + 读模型可降级」**：minidb 锁竞争（多进程共享 ~/.kimi-code）或降级时，会话列表回落到 N+1 全磁盘扫描（`sessionIndexService.ts` 的 `listLegacy`/`collectAuthoritative`）。
3. **「冷会话无中间态」**：transcript 冷读全量 parse wire.jsonl 重建、无缓存；看消息还要 `resumeSessionById` 拉起整个 scope（连带 spawn 该 workspace 的 MCP 进程）。
4. **「v1/v3 并存」**：v3 的轻量投影栈已就位，但主客户端 web UI 仍在 v1，优化红利尚未兑现到主路径。
5. **SEA sidecar 资源校验**：`kimi web` 每次启动对 web/native 资源做全量 sha256 校验 + 同步写盘（`apps/kimi-code/src/native/web-assets.ts`、`native-assets.ts`），堵死 sidecar 事件循环。
