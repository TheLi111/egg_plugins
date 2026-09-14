# DeepSeek Harness 架构研究（阶段 5）：扩展点清单

## 总则

DSH 的 extension point 不是“允许 plugin import internals”。作者应把新行为挂在公开的 Context service、typed event 或 client slot；architecture 的 feature map 明确要求新行为附着到 documented extension point，而非修改 agent loop。[`docs/architecture.md:133-160`](../deepseek-harness/docs/architecture.md#L133-L160) Cookbook 进一步指出：每个 product feature 应能映射到 documented listener/service，且不修改 loop。[`docs/cookbook/extension-cookbook.md:98-102`](../deepseek-harness/docs/cookbook/extension-cookbook.md#L98-L102)

## 扩展面清单

| 维度 | 作者使用的 API / protocol | 最小已有例子 | 生命周期与注意点 |
|---|---|---|---|
| tool | `ctx.tools.register(defineTool(...))` | [`docs/cookbook/adding-a-tool.md:7-38`](../deepseek-harness/docs/cookbook/adding-a-tool.md#L7-L38)；生产例 `packages/shell/tool-pwsh/src/index.ts:243-346` | registration effect dispose 后 schema 与 body 一同消失；schema 自动流入 prompt assembly。 |
| service | `class X extends Service { super(ctx, 'x') }` 或 `ctx.provide()` | `WebhookRuntime extends Service`，key 为 `ctx.webhookRuntime` [`packages/webhook/webhook/src/index.ts:58-75`](../deepseek-harness/packages/webhook/webhook/src/index.ts#L58-L75) | service instance 与 fiber 绑定，consumer 用 `inject` 声明依赖；避免 consumer 直接 import provider。 |
| event hook | `ctx.on(event, listener)`；waterfall 必须传 `next()` | `tools/pre-execute` deny hook [`docs/cookbook/extension-cookbook.md:11-33`](../deepseek-harness/docs/cookbook/extension-cookbook.md#L11-L33) | 选择 pre/guard/execute/post/result 应按是否可改写、是否必须单调拒绝、是否包裹 lifetime 区分。 |
| agent driver/UI driver | `ctx.agents` 创建/恢复，`agent.followup()` / `steer()` / `cancel()`；观察 `session/event` + `agent/assistant-stream` | cookbook 的 external protocol driver/UI pattern [`docs/cookbook/extension-cookbook.md:35-89`](../deepseek-harness/docs/cookbook/extension-cookbook.md#L35-L89) | durable transcript 用 `session/event`；live token 用 `agent/assistant-stream`；不要把 transient frame 当 replay source。 |
| session durable state | declaration-merge `SessionEventMap` + append event + projection fold | `sandbox/mode` 被 `sandboxPolicy` projection fold 成 `sandboxMode` [`packages/sandbox/sandbox-policy/src/index.ts:132-138`](../deepseek-harness/packages/sandbox/sandbox-policy/src/index.ts#L132-L138) | 任何 model-visible input 都必须由 durable event 重建；有 client state 需求则加 `ctx.sessionProjections` unit。 |
| system prompt | `ctx.systemPrompt.section/context/tools/variable` | `tool-pwsh` 注册 `tool:pwsh` section [`packages/shell/tool-pwsh/src/index.ts:243-248`](../deepseek-harness/packages/shell/tool-pwsh/src/index.ts#L243-L248) | registration 支持 scope shadow；assembly 是 waterfall；section order 用 service-owned order name。 |
| client UI slot | browser plugin 向 typed slot 注册 component/renderer | tool card 走 `tool.call.toolview` keyed slot；Conversation target 注册 definition + keyed Chat renderer [`docs/cookbook/adding-a-tool.md:93-97`](../deepseek-harness/docs/cookbook/adding-a-tool.md#L93-L97) [`docs/cookbook/extension-cookbook.md:128-129`](../deepseek-harness/docs/cookbook/extension-cookbook.md#L128-L129) | browser business plugin 接收 typed props，不能自行 wire session subscription；`ui-renderer` 是唯一 runtime→React bridge。 |
| UI conversation model | `ctx.uiConversation.events.register()` / `.views.register()` | Chat-like target 的 declaration merge + registration [`packages/client/ui-conversation/README.md:26-36`](../deepseek-harness/packages/client/ui-conversation/README.md#L26-L36) | Definitions 同时处理 durable events 和 transient assistant chunks；replacement/gap 时 assembler rebuild。 |
| skill provider | `ctx.skills.registerProvider()`，或直接 `ctx.skills.register()` runtime skill | `SkillRegistry` 的 provider registration 及 `skills/change` invalidation [`packages/skill/skill/src/index.ts:346-458`](../deepseek-harness/packages/skill/skill/src/index.ts#L346-L458) | provider 提供 candidate list + on-demand body；服务按 rank、provider order 合并，provider 错误可被跳过而不抹掉可用技能。 |
| subagent driver | `ctx.subagents.registerProvider(provider)` | `SubagentRuntime.registerProvider()` [`packages/subagent/subagent/src/index.ts:503-541`](../deepseek-harness/packages/subagent/subagent/src/index.ts#L503-L541) | provider 用 name/capabilities/start 实现 transport；unload 仅阻断新 start，不撤销已发布 child run。 |
| MCP server adapter | 一 server 一 plugin，discover tool 后 `ctx.tools.register()` | cookbook 的明确规则 [`docs/cookbook/extension-cookbook.md:123-125`](../deepseek-harness/docs/cookbook/extension-cookbook.md#L123-L125)；`mcp-client` 的 entry 以 server-qualified name 注册 tools [`packages/mcp/mcp-client/src/index.ts:1-3`](../deepseek-harness/packages/mcp/mcp-client/src/index.ts#L1-L3) | 外部 raw schema 直接成为 ToolDefinition；不是通过 Typert 扩展模型工具。 |
| Claude Code hook bridge | 把 command hook config 映射到 `agent/*`、`tools/*` listener | `hooks-claude-code` 将 PreToolUse 映射为 `tools/pre-execute`，Stop 映射到 `agent/turn-stopping` [`packages/hooks/hooks-claude-code/src/index.ts:95-188`](../deepseek-harness/packages/hooks/hooks-claude-code/src/index.ts#L95-L188) | hook invocation/result 作为 session facts 记录；bridge 本身依赖 shell/subprocess capability。 |
| Codex hook bridge | 读取 Codex `hooks.json`，映射到相同 interception points | `hooks-codex` 的 `agent/pre-step`、`tools/pre-execute`、`tools/post-execute`、`agent/turn-stopping` bindings [`packages/hooks/hooks-codex/src/index.ts:187-276`](../deepseek-harness/packages/hooks/hooks-codex/src/index.ts#L187-L276) | 当前只运行同步 command hooks；代码明确有未实现的 `systemMessage`/`updatedInput` 和 stop-loop TODO，不能假定与 Codex 完全等价。 |
| webhook | `ctx.webhookRuntime.register({id, kind, run})`；provider 调 `dispatch(VerifiedWebhookDelivery)` | `WebhookRuntime.register/dispatch` [`packages/webhook/webhook/src/index.ts:86-142`](../deepseek-harness/packages/webhook/webhook/src/index.ts#L86-L142) | 认证/验证由 provider adapter 前置；runtime snapshot delivery、异步运行 rule，non-null request 才创建 Workspace-backed Session。 |
| LLM provider | `ctx.llm.registerAdapter()` / `LlmAdapter` | DeepSeek/Pi AI adapter 是 registered providers [`docs/capability-seams.md:479`](../deepseek-harness/docs/capability-seams.md#L479) | loop 固定依赖 `ctx.llm` seam，provider 必须实现 prepare/stream route contract。 |
| persistence backend | 实现 `SessionPersistence` 的 `create/open/stat/list/export` | architecture 指定 JSONL provider 并列出 persistence seam contract [`docs/architecture.md:156-157`](../deepseek-harness/docs/architecture.md#L156-L157) | 必须遵守 current logical format、migration 和 write ownership；不是只实现 append file。 |
| dynamic in-product Cordis package | `ctx.dynamicCordisRunner` Remote methods / tool-cordis | `DynamicCordisRunnerService` 可触发 `cordis/request-run`、执行 Host half、停用并 emit retract [`packages/extensions/cordis-host-runner/src/index.ts:124-143`](../deepseek-harness/packages/extensions/cordis-host-runner/src/index.ts#L124-L143) [`packages/extensions/cordis-host-runner/src/index.ts:291-324`](../deepseek-harness/packages/extensions/cordis-host-runner/src/index.ts#L291-L324) | 这是受 runner guard/approval 约束的 runtime package system，不等同于任意 npm plugin install。 |

## 三个常用的“最小插件”形状

### 1. 模型工具

```ts
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'my_tool',
    description: '...',
    parameters: { input: { type: 'string', required: true } },
    output: { schema: { type: 'string' }, render: (_args, value) => [{ type: 'text', text: value }] },
    async execute({ input }, exec) { /* observe exec.signal */ return input },
  }))
}
```

这是项目 cookbook 的 prescribed shape，省略的关键约束是 `execute` 必须 cooperative cancellation、canonical output 必须过 schema、presentation 必须 pure/replayable。[`docs/cookbook/adding-a-tool.md:7-48`](../deepseek-harness/docs/cookbook/adding-a-tool.md#L7-L48)

### 2. 不可被后续 plugin 推翻的 deny

```ts
ctx.tools.guard((exec) => {
  return exec.name === 'dangerous_operation' ? 'not allowed by deployment policy' : undefined
})
```

这里不应使用普通 pre waterfall listener，因为其选择 rule 是：`tools/pre-execute` 用于 reorderable allow/deny/ask，`ctx.tools.guard()` 用于 final monotonic deny。[`docs/cookbook/adding-a-tool.md:57-60`](../deepseek-harness/docs/cookbook/adding-a-tool.md#L57-L60)

### 3. 可替换 provider

```ts
export function apply(ctx: Context) {
  ctx.subagents.registerProvider({
    name: 'my-driver',
    capabilities: { /* declared feature flags */ },
    async start(request) { /* return published child run */ },
  })
}
```

实际 interface 还要求完整 provider contract；此片段只展示注册点。runtime 会按 name 查找 provider、先检查 capabilities、再 invoke `start()`。[`packages/subagent/subagent/src/index.ts:545-614`](../deepseek-harness/packages/subagent/subagent/src/index.ts#L545-L614)

## 前端扩展的边界

Web renderer 明确把 `ctx`/Cordis state 到 React 的绑定集中在一个位置：business plugin 注册 slot component，获得 standard props，不直接接触 runtime subscription 或 React internals。[`packages/client/ui-renderer/README.md:34-58`](../deepseek-harness/packages/client/ui-renderer/README.md#L34-L58) Conversation assembly 则把 durable `SessionEventLikeEntry` 与 transient `assistant/live-chunk` 一并处理；target plugin 只注册 Definition/View，而不会再打开第二个 Session event source。[`packages/client/ui-conversation/README.md:26-36`](../deepseek-harness/packages/client/ui-conversation/README.md#L26-L36)

这意味着“给 Web 增加一个工具卡”通常是两项 work：Host tool 用 `presentationMeta` 持久化必要结果事实，Client plugin 占据 `tool.call.toolview` key 并从 raw event 导出 props；只实现 `presentResult()` 不会自动得到 Web 专用卡。[`docs/cookbook/adding-a-tool.md:93-97`](../deepseek-harness/docs/cookbook/adding-a-tool.md#L93-L97)

## 我读了哪些文件

- `docs/cookbook/{extension-cookbook,adding-a-tool,adding-a-package}.md`
- `packages/{skill/skill,subagent/subagent,mcp/mcp-client,webhook/webhook}` 的 README 与入口实现
- `packages/hooks/{hooks-claude-code,hooks-codex}` 的 README 与入口实现
- `packages/extensions/{cordis-host-runner,tool-cordis}` 的 README 与入口实现
- `packages/client/{ui-renderer,ui-conversation}` 的 README 与入口/contract 定位
- `docs/{architecture,capability-seams}.md`

## 还有哪些没搞清楚

- MCP client 的 transport lifecycle、server restart 与 JSON-schema conversion failure paths 没有逐行分析。
- Claude/Codex hook protocol 的 payload translator/command parser 位于共享 `hook-protocol` 包，本阶段只验证了 bridge 的 extension-point mapping。
- client slot system 的完整 generic type chain 和 `ConversationNodeDefinition` 的实际 Chat package example 尚未逐 source 展开；需要做 UI 插件时应继续读 `ui-slots` 与 `ui-chat`。
