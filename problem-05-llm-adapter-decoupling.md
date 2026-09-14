# 问题五：DeepSeek Harness 如何让 LLM 适配解耦

## 结论

dsh 的 LLM 解耦不是简单定义一个 `generate(messages)` 方法，而是同时统一：

- request route：显式 `provider + model`；
- provider-neutral message/content/tool schema；
- provider-neutral streaming grammar `StreamChunk`；
- adapter-owned exact model capacity/defaults；
- provider-specific replay state；
- retry、request interception 和 stream observation 的独立插件点。

上层 AgentLoop 只面向 `ctx.llm` 和 `GenerateOptions`。Adapter 负责把稳定语义映射到 DeepSeek SSE、pi-ai/OpenAI/Anthropic 等 provider protocol，并在无法无损表达时明确拒绝，而不是静默忽略。

## 1. 中立请求与流协议

`dsh-llm` 定义 provider-neutral streaming vocabulary：`block-start`、`text-delta`、`reasoning-delta`、`tool-call-delta`、`block-end`、`usage`、`finish`。[`.agents/notes/implemented/architecture/2026-06-13-twin-llm-adapters.md:7-18`](../deepseek-harness/.agents/notes/implemented/architecture/2026-06-13-twin-llm-adapters.md#L7-L18)

这套 grammar 允许 loop 统一执行：

1. 按 block index 组装 text/reasoning/tool-call；
2. 在完成时记录 usage 与 finish reason；
3. 将 provider stream 存进 `assistant/message.stream`；
4. 对 error/aborted 产生一致的 `LlmFailure`。

`LlmRuntime.stream()` 本身通过 `llm/stream` waterfall 包装最终 adapter stream。[`packages/llm/llm/src/index.ts:1106-1120`](../deepseek-harness/packages/llm/llm/src/index.ts#L1106-L1120) 因此 metrics、mock/replay 或其他 stream middleware 不需要修改 AgentLoop。

## 2. 为什么同时需要 provider 和 model

早期以 model name 作为 adapter registration key，会混淆“谁提供请求”和“该 provider 运行什么模型”：同一 model id 可存在于多个 provider，gateway 又可能拥有启动后变化的开放模型目录。[`.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md:7-16`](../deepseek-harness/.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md#L7-L16)

当前 `GenerateOptions`、`LlmCallConfig` 与 request header 都携带显式 `provider` 和 `model`。`LlmRuntime` 以 provider 注册 adapter，model id 由选中的 adapter 解析/验证/转发。[`.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md:17-27`](../deepseek-harness/.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md#L17-L27)

一个 provider 在一个 Context 中只能有一个 owner。这样避免 exact/wildcard fallback ordering 和隐式优先级；代价是 native DeepSeek adapter 与 pi-ai 的 DeepSeek implementation 不能同时注册同一 provider route。[`.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md:23-27`](../deepseek-harness/.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md#L23-L27)

## 3. `prepareCall()`：能力解析与 dispatch 绑定

AgentLoop 在写入本次 system/user input 前调用 `ctx.llm.prepareCall(proposedConfig, signal)`；返回值包含 resolved config、exact model context、adapter defaults、system-prompt update mode、retry policy 和只能 dispatch 一次的 bound stream handle。[`packages/core/agent-loop/src/agent.ts:500-549`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L500-L549) [`packages/llm/llm/src/index.ts:900-958`](../deepseek-harness/packages/llm/llm/src/index.ts#L900-L958)

它解决了 adapter registry 在异步 request preparation 与真正 stream 之间变化的竞态：model capacity、defaults 与 dispatch 都来自同一 adapter generation。否则 request header 可能记录旧 adapter 的能力，却向新 adapter endpoint 发请求。

## 4. Provider-specific translation 留在 adapter 内

DeepSeek adapter 的 `prepareCall()` snapshot 当前 connection options，并将其绑定到后续 `streamWithConnection()`。[`packages/llm/llm-deepseek/src/adapter.ts:440-459`](../deepseek-harness/packages/llm/llm-deepseek/src/adapter.ts#L440-L459) 每个 stream 冻结 endpoint、credential 与 model configuration，设置 idle watchdog，并将 caller signal 与内部 consumer signal 合并。[`packages/llm/llm-deepseek/src/adapter.ts:461-527`](../deepseek-harness/packages/llm/llm-deepseek/src/adapter.ts#L461-L527)

实际 HTTP request 使用 `accept: text/event-stream`，带 provider-specific authorization 与 Harness attribution headers。[`packages/llm/llm-deepseek/src/adapter.ts:530-551`](../deepseek-harness/packages/llm/llm-deepseek/src/adapter.ts#L530-L551)

Pi-AI adapter 则根据 provider profile 选择 OpenAI Responses、Anthropic Messages、Google Generative AI 等 protocol。共同 stream API 无法表达 stop sequences 时，它返回 `UNSUPPORTED_OPTION`，而不是忽略；native DeepSeek adapter 仍可支持自身 serializer 的 stop。[`.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md:29-37`](../deepseek-harness/.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md#L29-L37)

## 5. 为什么设计两个真实 adapter

仅有一个 DeepSeek adapter 时，所谓“provider-neutral” vocabulary 可能只是 DeepSeek wire protocol 的改名。设计笔记因此用 direct-fetch DeepSeek adapter 与基于 pi-ai 的第二 adapter 互相验证；任何无法同时表达两者语义的部分都视为 core vocabulary 缺陷。[`.agents/notes/implemented/architecture/2026-06-13-twin-llm-adapters.md:11-18`](../deepseek-harness/.agents/notes/implemented/architecture/2026-06-13-twin-llm-adapters.md#L11-L18)

该 twin 验证了 usage-before-finish、finish 后无事件、tool-call arguments 保留 raw JSON string，以及 throw/terminal error chunk 两种错误路径。代价是 adapter 与 real-API e2e 维护量增加一倍。[`.agents/notes/implemented/architecture/2026-06-13-twin-llm-adapters.md:18-27`](../deepseek-harness/.agents/notes/implemented/architecture/2026-06-13-twin-llm-adapters.md#L18-L27)

## 6. Provider-specific replay state

仅存最终文本会丢失 provider continuation 所需的 response id、reasoning signature、tool signature 等。当前 assistant message 因此记录 authoritative `provider`、`model` 和 optional opaque adapter replay state。[`.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md:39-47`](../deepseek-harness/.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md#L39-L47)

规则是：

- Harness content 始终是 durable authoritative content；
- replay state 只在同一 adapter instance 同时拥有 historical provider 与 target provider 时交给 adapter；
- adapter 无法使用、版本未知或 block 对不上时，降级为 provider-neutral conversion 并给 diagnostic；
- 切换到另一个 adapter 时不会泄漏 opaque replay state。

这保留 provider 特有能力，又不让 core loop 理解 provider payload。

## 7. 前缀缓存如何被架构支持

dsh 没有一个统一“cache API”来模拟所有 provider，而是通过稳定 request construction 提升复用可能性：

- system prompt 是 protected surface head；
- tool schemas 有确定顺序；
-历史 messages 从 immutable log 派生；
- compaction summarizer 重放相同 system/tool/history prefix，再在尾部追加 instruction；
- fork subagent 默认继承同 provider/model，避免切 route 后重新 prefill。

Compaction 的 warm-prefix 设计见 [`packages/compaction/compaction-basic/README.md:101-119`](../deepseek-harness/packages/compaction/compaction-basic/README.md#L101-L119)。Fork 保留 route 以维持 KV-cache reuse 的决定见 [` .agents/notes/implemented/feature/2026-08-18-model-selected-subagent-routes.md:21-23`](../deepseek-harness/.agents/notes/implemented/feature/2026-08-18-model-selected-subagent-routes.md#L21-L23)。

必须注意：代码只能保持输入前缀稳定，不能保证 provider 一定 cache hit；cache 命中、retention 和计费仍属于 provider 行为。

## 8. Retry 与 provider attempt 可见性

Pi-AI 内部 retries 被强制为零；provider-owned retry policy 由 `dsh-llm-retry` 在 `agent/request-error` extension point 执行，从而每个 attempt 都可在 Session 中结算，而不是被 SDK 内部隐藏。[`.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md:29-35`](../deepseek-harness/.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md#L29-L35)

AgentLoop 对 terminal error/aborted stream append `assistant/attempt`，再询问 `agent/request-error` 是否 retry；retry 不重新运行 prompt assembly 或再次写 user message。[`packages/core/agent-loop/src/agent.ts:441-463`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L441-L463)

## 9. 代价与限制

- Core stream grammar 必须覆盖多 provider 的交集，同时允许 replay state 承载差异；设计和验证复杂。
- One-provider-one-owner 简化路由，但不能在同一 Context 中 A/B 两个实现同一 upstream。
- Adapter-specific unsupported option 必须显式暴露，应用层不能假定所有 config 跨 provider 可用。
- Provider catalog、credential profile、endpoint protocol 的配置错误可能在 request 前 fail loud。
- Opaque replay state 影响长期格式和隐私；设计笔记明确指出可能持久化 encrypted reasoning signatures，但不在普通 transcript 渲染。[`.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md:73-91`](../deepseek-harness/.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md#L73-L91)

## 用于制作 PPT 的提示词

```text
请生成中文技术 PPT《DeepSeek Harness 的 LLM 能力接缝》，11 页，16:9，面向 LLM 平台和 Agent runtime 工程师。重点解释“统一语义、保留 provider 特性”，不要宣称所有 provider 功能完全等价。

页面：
1. 标题与结论：route、stream grammar、replay state 三层解耦。
2. 单一 generate() 抽象为什么不足。
3. provider + model 显式路由及 one-provider-one-owner。
4. GenerateOptions / Message / ToolSchema / StreamChunk 统一词汇图。
5. prepareCall 绑定 adapter generation、capacity、defaults、dispatch。
6. AgentLoop → llm/stream waterfall → adapter 的时序图。
7. DeepSeek direct-fetch/SSE translation。
8. pi-ai 多协议 adapter 与 unsupported-option fail loud。
9. 双真实 adapter 如何验证中立性。
10. provider-specific replay state 与跨 adapter 降级。
11. prefix-cache 支持、retry 可见性、代价与限制。

页脚证据使用相对路径:行号，重点引用 packages/llm/llm/src/index.ts:900-958、packages/llm/llm/src/index.ts:1106-1120、packages/llm/llm-deepseek/src/adapter.ts:440-527、.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md:17-47、.agents/notes/implemented/architecture/2026-06-13-twin-llm-adapters.md:7-27。每页给 80～120 字讲稿。
```
