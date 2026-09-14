# DeepSeek Harness 架构研究（阶段 1）：全局装配与 Cordis 运行时

## 范围与结论边界

本阶段回答“`dsh web` 如何成为一个可用 agent 会话系统”、Cordis 的最小运行语义，以及 Web 默认组合中的核心服务。这里的“提供者”指实际提供 `ctx.<key>` 实例或向某个 seam 注册实现的包；并不把每一个 UI renderer 都误称为核心 agent 服务。工具、会话格式、单 turn 与扩展机制会在后续阶段展开。

所有事实均以当前源码为准。除明确标记为「推测」的段落外，均有仓库内路径和行号证据。

## 1. 从 `dsh web` 到可用会话的装配路径

```text
dsh web
  -> CLI 选择 profile = web
  -> PROFILE_TEMPLATES.web
       dsh-base patch
       dsh-web-app patch
  -> profile cordis.patch.yml
  -> $DSH_HOME/cordis.patch.yml
  -> --patch overlays
  -> telemetry environment overlay
  -> Cordis Loader 的 root Include
  -> 按 inject 服务可用性激活 plugin fibers
  -> Host HTTP/API/Client graph + agent preset roster
  -> 用户创建 Session 时，agent-presets 将 standard agent.cordis.yml 挂到该 agent scope
  -> AgentRegistry / AgentLoop 可接收消息并开始 turn
```

1. `web` 不是一个独立 Node 入口，而是 CLI 的 named profile；`PROFILE_TEMPLATES.web` 指定 bundle 顺序为 `@deepseek-ai/dsh-base`、`@deepseek-ai/dsh-web-app`，并启用 `live` patch reload。[`packages/boot/app-boot/src/profile.ts:104-126`](../deepseek-harness/packages/boot/app-boot/src/profile.ts#L104-L126)
2. CLI 对空的 root entry list 依次合并 bundle patch、profile patch、home patch、`--patch` overlay；最后还可加入由 `DSH_TELEMETRY_DISABLED` 派生的 telemetry patch。[`apps/cli/src/profile-boot.ts:205-243`](../deepseek-harness/apps/cli/src/profile-boot.ts#L205-L243)
3. 随后 `runProfile()` 在任何 config row 启动前提供 launch-environment 与 `cmdlineArgs`，再调用 `boot()` 将合成 patch 交给 Loader。[`apps/cli/src/profile-boot.ts:282-348`](../deepseek-harness/apps/cli/src/profile-boot.ts#L282-L348)
4. `mountRootInclude()` 将 `cordis:include`（固定 id 为 `include`）交给 Loader；Include 的 `path` 是 profile 目录中生成的空 `cordis.yml`，真正的条目来自 patch。它还内置 `cordis:group`，用于让同一 agent preset 的 provider 与 consumer 处在同一个 `isolate` realm。[`packages/boot/app-boot/src/index.ts:506-553`](../deepseek-harness/packages/boot/app-boot/src/index.ts#L506-L553)
5. base patch 首先插入 LLM、session、agent、settings、credentials 等全局行；例如 `llm`、`session`、`agent` 分别为 `@deepseek-ai/dsh-llm`、`@deepseek-ai/dsh-session`、`@deepseek-ai/dsh-agent`。[`packages/bundle/base/cordis.patch.yml:15-108`](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L15-L108)
6. base patch 还装配 JSONL 持久化、projection、local subprocess、sandbox、approval 与 permission presets。[`packages/bundle/base/cordis.patch.yml:110-240`](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L110-L240)
7. 它的末端装配 `ctx.tools`、`ctx.systemPrompt`、`ctx.agentLoop`、`ctx.fs` provider 和 DeepSeek LLM adapter；agent-loop 的初始 `agents` 数组为空，因此 Web 不在 boot 时静态创建 agent。[`packages/bundle/base/cordis.patch.yml:456-487`](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L456-L487)
8. web-app patch 新增 `webStartup`、`webServer`、`webRuntime`、connection、API gateway/remotes 与 client graph；`webServer` 明确注入 `webStartup`，而 `webRuntime` 也注入它。[`packages/bundle/web-app/cordis.patch.yml:122-199`](../deepseek-harness/packages/bundle/web-app/cordis.patch.yml#L122-L199)
9. Web bundle 禁用 base 中的若干全局 model-facing row（如 `tool-bash`、`agent-instructions`、`tool-web`），然后插入 `agent-presets`。[`packages/bundle/web-app/cordis.patch.yml:450-484`](../deepseek-harness/packages/bundle/web-app/cordis.patch.yml#L450-L484)
10. 每个 Session 最终从 preset 的 `agent.cordis.yml` 获得 tool、prompt section 和 skill；同 preset 的 Session 共用 standing composition，但状态隔离。[`packages/preset/agent-presets/README.md:28-36`](../deepseek-harness/packages/preset/agent-presets/README.md#L28-L36) `standard` preset 的 row 包含 persona、agent instructions、shell/fs 工具、skill 与 compaction/subagent 等组件。[`packages/preset/agent-presets/presets/standard/agent.cordis.yml:25-255`](../deepseek-harness/packages/preset/agent-presets/presets/standard/agent.cordis.yml#L25-L255)

### 文档与代码的一个差异

`docs/architecture.md` 将可见 patch 层描述为 bundle → profile patch → home patch → `--patch`。[`docs/architecture.md:27-29`](../deepseek-harness/docs/architecture.md#L27-L29) 当前 CLI 代码还在这些层之后追加一个由 `DSH_TELEMETRY_DISABLED` 生成的 telemetry overlay。[`apps/cli/src/profile-boot.ts:205-243`](../deepseek-harness/apps/cli/src/profile-boot.ts#L205-L243) 这不是配置文件层的冲突，而是文档未列出一个环境变量派生层；本报告按代码的五层顺序理解。

## 2. Cordis：DSH 实际使用的运行时模型

### 2.1 Context 与服务发现

Cordis 的 `Context` 是 Proxy。读取 `ctx.foo` 经 service resolver；`extend()` 创建从父 Context 继承的 child；`isolate(name)` 为指定 service key 创建独立解析 scope，而不改父 Context。[`vendor/cordis/src/context.ts:35-41`](../deepseek-harness/vendor/cordis/src/context.ts#L35-L41) [`vendor/cordis/src/context.ts:90-125`](../deepseek-harness/vendor/cordis/src/context.ts#L90-L125)

服务通常继承 `Service`，在构造函数中以名称调用 `ctx.reflect.provide()`；该实现与 owning fiber 绑定，fiber unload 时自动撤销。[`vendor/cordis/src/service.ts:5-10`](../deepseek-harness/vendor/cordis/src/service.ts#L5-L10) [`vendor/cordis/src/service.ts:32-58`](../deepseek-harness/vendor/cordis/src/service.ts#L32-L58)

### 2.2 Plugin、依赖注入与 Fiber 生命周期

Cordis 接受三种 plugin 入口：函数 `(ctx, config)`、class `(ctx, config)`、或带 `apply(ctx, config)` 的对象；每种入口都可声明 `inject`、`provide` 和 Config schema。[`vendor/cordis/src/registry.ts:91-145`](../deepseek-harness/vendor/cordis/src/registry.ts#L91-L145)

`ctx.inject(deps, callback)` 本质上是带依赖的 plugin；所需服务不存在时 fiber 保持 pending，存在时才执行。[`vendor/cordis/src/registry.ts:164-185`](../deepseek-harness/vendor/cordis/src/registry.ts#L164-L185) Fiber 的状态集合为 `PENDING`、`LOADING`、`ACTIVE`、`FAILED`、`UNLOADING`、`DISPOSED`。[`vendor/cordis/src/fiber.ts:139-154`](../deepseek-harness/vendor/cordis/src/fiber.ts#L139-L154) `_refresh()` 按 inject service 是否可解析决定 epoch；epoch 变化会 reload 或 unload plugin。[`vendor/cordis/src/fiber.ts:611-673`](../deepseek-harness/vendor/cordis/src/fiber.ts#L611-L673)

`ctx.effect()` 的 disposer 按反向注册顺序执行，并允许 async disposer；因此服务、event listener、prompt/tool registration 可以随同 plugin unload 被收回。[`vendor/cordis/src/fiber.ts:68-101`](../deepseek-harness/vendor/cordis/src/fiber.ts#L68-L101)

### 2.3 Event 与 middleware

Cordis 有 `emit`、`parallel`、`serial`、`bail`、`waterfall` 五种 dispatch mode。`waterfall` 把 listener 组成 around middleware：listener 不调用 `next()` 即 veto/short-circuit 后续链。[`vendor/cordis/src/events.ts:24-32`](../deepseek-harness/vendor/cordis/src/events.ts#L24-L32) [`vendor/cordis/src/events.ts:76-88`](../deepseek-harness/vendor/cordis/src/events.ts#L76-L88) Listener 的注册本身是 fiber-owned effect，因此卸载 plugin 会解除监听。[`vendor/cordis/src/events.ts:245-301`](../deepseek-harness/vendor/cordis/src/events.ts#L245-L301)

### 2.4 “everything is a plugin”的精确定义

这不是“所有代码都在独立 npm 包”的泛称，而是以下三个可验证性质的组合：

- 产品能力（含 model adapter、tool registry、session log、agent loop）由 plugin 贡献到 shared Context，而不是通过 privileged core 特判。[`docs/architecture.md:9-13`](../deepseek-harness/docs/architecture.md#L9-L13)
- 功能关系通过 `ctx.<service>`、`inject` 与 typed events 表示，而不是 consumer import 某个具体 provider；例如 agent-loop 调用 provider-neutral `ctx.llm`，DeepSeek/Pi AI adapter 在该 seam 注册 route。[`docs/capability-seams.md:479`](../deepseek-harness/docs/capability-seams.md#L479)
- 被注册的东西是 effect，有 disposer；因此 patch/HMR/agent scope 卸载会同步撤销服务与监听。[`docs/cordis-primer.md:9-13`](../deepseek-harness/docs/cordis-primer.md#L9-L13)

## 3. 核心服务与提供者（Web 默认组合）

下表以 `docs/capability-seams.md` 的生成图为服务名与 provider 的权威枚举，并以 base/web patch 确认 Web 默认组合会装配相关行。`Definition` 是 service 接口所有者，`Provider` 是默认运行实现；两者相同时意味着该包自身提供 service，而不表示其不可替换。

| 分层 | `ctx` key | Definition / 默认 provider |
|---|---|---|
| agent spine | `sessions` | `core/session` / 同包 [`docs/capability-seams.md:483`](../deepseek-harness/docs/capability-seams.md#L483) |
| agent spine | `agents` | `core/agent` / 同包 [`docs/capability-seams.md:519`](../deepseek-harness/docs/capability-seams.md#L519) |
| agent spine | `agentLoop` | `core/agent-loop` / bundle base 装配 [`docs/capability-seams.md:521`](../deepseek-harness/docs/capability-seams.md#L521) |
| prompt/tool | `systemPrompt` | `core/system-prompt` / 同包 [`docs/capability-seams.md:510`](../deepseek-harness/docs/capability-seams.md#L510) |
| prompt/tool | `tools` | `core/tools` / 同包 [`docs/capability-seams.md:511`](../deepseek-harness/docs/capability-seams.md#L511) |
| LLM | `llm` | `llm/llm` / `llm-deepseek`、`llm-pi-ai`（测试还可用 `llm-replay`）[`docs/capability-seams.md:479`](../deepseek-harness/docs/capability-seams.md#L479) |
| LLM | `deepseekLlmApiExtensions` | `deepseek-llm-api-extensions` / `session-log-deepseek`、`plugin-package-inventory-deepseek` [`docs/capability-seams.md:480`](../deepseek-harness/docs/capability-seams.md#L480) |
| LLM | `agentDefaultModel`、`tokenMeter` | `agent-default-model`、`token-meter` / 各自同包 [`docs/capability-seams.md:481`](../deepseek-harness/docs/capability-seams.md#L481) [`docs/capability-seams.md:520`](../deepseek-harness/docs/capability-seams.md#L520) |
| durable state | `sessionPersistence` | `session-persistence` / `session-persistence-jsonl` [`docs/capability-seams.md:495`](../deepseek-harness/docs/capability-seams.md#L495) |
| durable state | `sessionProjections`、`sessionProjectionCache` | `session-projection`、`session-projection-cache` / 各自同包 [`docs/capability-seams.md:516-517`](../deepseek-harness/docs/capability-seams.md#L516-L517) |
| durable state | `sessionQuery` | `session-query` / `session-query-sqlite` [`docs/capability-seams.md:506`](../deepseek-harness/docs/capability-seams.md#L506) |
| durable state | `storage`、`storageDomain` | `storage` / `storage-json`（也可 sqlite）；`storage-domain` / 同包 [`docs/capability-seams.md:501-502`](../deepseek-harness/docs/capability-seams.md#L501-L502) |
| durable state | `sessionTitle`、`attachments` | `session-title` / first/all-prompts LLM provider；`attachment` / `attachment-local` [`docs/capability-seams.md:477`](../deepseek-harness/docs/capability-seams.md#L477) [`docs/capability-seams.md:509`](../deepseek-harness/docs/capability-seams.md#L509) |
| configuration | `settings`、`credentials`、`authorization` | `settings` / `settings-file`；`credentials` / `credentials-local`；`authorization` 的 flow registry 无默认 provider [`docs/capability-seams.md:496`](../deepseek-harness/docs/capability-seams.md#L496) [`docs/capability-seams.md:498-499`](../deepseek-harness/docs/capability-seams.md#L498-L499) |
| per-session composition | `agentPresets`、`skills`、`commands`、`planMode`、`goals` | `agent-presets`；`skill` / filesystem 等 provider；`commands`；`plan-mode`；`goal`，均由各自同名包提供 registry/domain [`docs/capability-seams.md:513-518`](../deepseek-harness/docs/capability-seams.md#L513-L518) [`docs/capability-seams.md:522`](../deepseek-harness/docs/capability-seams.md#L522) |
| process execution | `subprocess` | `subprocess` / `subprocess-local` 或 E2B [`docs/capability-seams.md:524`](../deepseek-harness/docs/capability-seams.md#L524) |
| process execution | `shell`、`shellEnv` | `shell` / `bash-local`、`bash-sandbox`、`pwsh-local`；`shell-env` / 同包 [`docs/capability-seams.md:525-526`](../deepseek-harness/docs/capability-seams.md#L525-L526) |
| policy/security | `sandbox`、`sandboxPolicy`、`approval`、`permissionPresets` | `sandbox` / `sandbox-local`；`sandbox-policy`；`user-approval`；`permission-presets` [`docs/capability-seams.md:528-531`](../deepseek-harness/docs/capability-seams.md#L528-L531) |
| filesystem | `fs` | `fs` / `fs-local`、`fs-sandbox` 或 `fs-e2b`；`fs-observation-policy` 以 event policy 介入 [`docs/capability-seams.md:533`](../deepseek-harness/docs/capability-seams.md#L533) |
| context reduction | `compaction`、`toolResultPruner`、`spillStore` | `compaction` / `compaction-basic`；`compaction-tool-result-pruner`；`spill` / `spill-local` [`docs/capability-seams.md:482`](../deepseek-harness/docs/capability-seams.md#L482) [`docs/capability-seams.md:534`](../deepseek-harness/docs/capability-seams.md#L534) [`docs/capability-seams.md:540`](../deepseek-harness/docs/capability-seams.md#L540) |
| delegation/work | `subagents`、`jobs`、`workflowEngine` | `subagent` / in-process、ACP、Codex、Claude Code、SDK drivers；`jobs` / `jobs-local`；`workflow` / worker-thread provider [`docs/capability-seams.md:535`](../deepseek-harness/docs/capability-seams.md#L535) [`docs/capability-seams.md:538`](../deepseek-harness/docs/capability-seams.md#L538) [`docs/capability-seams.md:544`](../deepseek-harness/docs/capability-seams.md#L544) |
| network/extensibility | `web`、`webhookRuntime`、`lsp` | `web` / DeepSeek、Exa、Perplexity search 和 HTTP fetch；`webhook`；`lsp` / stdio provider [`docs/capability-seams.md:539`](../deepseek-harness/docs/capability-seams.md#L539) [`docs/capability-seams.md:545-546`](../deepseek-harness/docs/capability-seams.md#L545-L546) |
| host/API | `typert`、`typertGateway` | `typert-registry`；`api-gateway` [`docs/capability-seams.md:493-494`](../deepseek-harness/docs/capability-seams.md#L493-L494) |
| host/API | `sessionController`、`settingsController`、`workspaceController` | `api-session-controller`、`api-settings-controller`、`api-workspace-controller` [`docs/capability-seams.md:484-491`](../deepseek-harness/docs/capability-seams.md#L484-L491) |
| host/Web transport | `webServer`、`clientModules` | `host-webserver`、`client-modules`；connection/HMR 等以此注册 routes/client graph [`docs/capability-seams.md:542-543`](../deepseek-harness/docs/capability-seams.md#L542-L543) |
| dynamic extension | `dynamicCordisRunner`、`cordisInspect` | `extensions/cordis-host-runner` / 同包 [`docs/capability-seams.md:547-548`](../deepseek-harness/docs/capability-seams.md#L547-L548) |

完整 capability 图还枚举了与 Web product composition 可选或实验相关的 `fileUploads`、`workspaceRegistry`、`messageFeedback`、`sessionFeedback`、`directoryPicker`、`terminals`、`codeRuntime`、`e2b`、`agentTeams`、`inspector` 等服务；它们的 Definition/provider/consumer 三元组见 [`docs/capability-seams.md:477-548`](../deepseek-harness/docs/capability-seams.md#L477-L548)。这份图明确区分 `core`、`seam`、`bundle` 分类，因此不应把所有 `ctx` key 都当成可替换 provider seam。

## 4. 分层图

```text
CLI / app boot
  profile + patch layers ──> Cordis Loader / fibers
                               │
Host control plane             │                     Per-agent realm
webServer ─ connection ─ API ──┼── SessionController ─ AgentRegistry ─ AgentLoop
Typert / Remote                │            │                │
                               │            └── Session log ←┴→ SystemPrompt + Tools
                               │                                  │          │
Shared capability plane         ├── LLM registry/adapters          │          ├─ tool definitions
settings / credentials          ├── persistence/projections        │          └─ policy waterfalls
sandbox / fs / subprocess       ├── sandbox / approval / fs        │
skills / web / subagents        └── preset standing mounts ────────┘
                               │
Browser plane                  └── client modules / connection / UI plugins
```

该图的关键不是单纯“分层”，而是作用域：base/web 的 host services 位于 process root；`agent-presets` 将 session 所选 composition 挂到 agent scope，使同一进程的 Session 可拥有不同工具与 prompt，而 preset 外的 root service 不会被其 isolate provider 覆盖。[`packages/preset/agent-presets/README.md:98-130`](../deepseek-harness/packages/preset/agent-presets/README.md#L98-L130) [`vendor/cordis/src/context.ts:109-125`](../deepseek-harness/vendor/cordis/src/context.ts#L109-L125)

## 5. 阶段结论

- DSH 的“产品定义”是 profile patch 后的 Cordis plugin tree，而非一个固定的 agent runtime 类。启动器生成空 root，然后用 Include/Loader 让 patch 后的 row 逐个成为 fiber。[`apps/cli/src/profile-boot.ts:83-91`](../deepseek-harness/apps/cli/src/profile-boot.ts#L83-L91) [`packages/boot/app-boot/src/index.ts:516-553`](../deepseek-harness/packages/boot/app-boot/src/index.ts#L516-L553)
- `dsh-base` 提供共享宿主能力；Web 不是简单在 base 上“加 UI”，而是把多项 model-facing row 从 root 移到 session preset，因而工具集合是 per-agent composition 的属性。[`packages/bundle/base/cordis.patch.yml:240-284`](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L240-L284) [`packages/bundle/web-app/cordis.patch.yml:450-484`](../deepseek-harness/packages/bundle/web-app/cordis.patch.yml#L450-L484)
- Cordis 的替换能力来自三个联合机制：`ctx` 的服务解析、`inject` 驱动的 fiber activation/reload、fiber-owned effects 的 teardown；仅把模块拆成 package 并不足以获得这种行为。[`vendor/cordis/src/registry.ts:294-335`](../deepseek-harness/vendor/cordis/src/registry.ts#L294-L335) [`vendor/cordis/src/fiber.ts:675-710`](../deepseek-harness/vendor/cordis/src/fiber.ts#L675-L710)

## 我读了哪些文件

- `docs/architecture.md`
- `docs/development.md`
- `docs/cordis-primer.md`
- `docs/subsystems/core.md`（架构 spine 与服务目录部分）
- `docs/subsystems/{session,tools,system-prompt,llm-streaming,scope}.md`（目录与 API 定位部分）
- `docs/capability-seams.md`
- `packages/boot/app-boot/src/{index,profile}.ts`
- `apps/cli/src/profile-boot.ts`
- `packages/bundle/{base,web-app}/cordis.patch.yml`
- `packages/preset/agent-presets/{README.md,presets/standard/agent.cordis.yml}`
- `vendor/cordis/src/{context,service,registry,events,fiber}.ts`

## 还有哪些没搞清楚

- Web Client 从浏览器输入到 `api-session-controller` 的具体 transport（RPC/SSE）边界尚未逐调用点验证；留待阶段 2。
- `cordis:include` 对 id-targeted patch 的替换、insert 和 group child patch 的全部算法尚未从 Include 源码逐行验证；本阶段只验证了 DSH 对它的调用点。
- `agent-presets` 如何将新建 session 的 Context parent 到 standing mount，以及 blank-session switching 的 commit 顺序，尚未读取 `mount.ts`/`index.ts` 的实现；留待阶段 2/5。
- 上表列举的是当前 capability graph 所定义的核心 Web product 服务；具体某个 optional/experimental row 是否在默认 `web` patch 中启用，仍应以该 profile 的最终 dump-config 为准。
