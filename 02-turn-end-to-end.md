# DeepSeek Harness 架构研究（阶段 2）：一次 Web → PowerShell turn 的端到端链路

## 场景与边界

场景：Windows Web UI 的用户提交“列出当前目录文件”的文本，模型选择 `pwsh` 工具。本文跟踪 Host 接收到 `SessionPromptRequest` 以后至同一 turn 的 tool result 被写回 Session；浏览器组件自身的 React click handler 不在本阶段断言范围内。Web Client 经 Gateway Remote 调用 Host `ctx.sessionController` 是包 README 明示的架构，而 `prompt()` 是 Host 入口。[`packages/api/session-controller/README.md:9-12`](../deepseek-harness/packages/api/session-controller/README.md#L9-L12)

## 调用链总览

```text
Web Session Remote
  SessionController.prompt(SessionPromptRequest)
    -> Agent.followup(UserMessage)
      -> AgentLoop.ReactLoopAgent.send(... next-turn, wakeup=true)
        -> Session turn/start
        -> Inbox.claim + SystemPrompt.assemble + agent/pre-step
        -> Session step/start
        -> agent/request + LlmRuntime.prepareCall
        -> Session system/user/request events -> deriveMessages()
        -> PreparedLlmCall.stream() -> llm/stream waterfall -> DeepSeek SSE chunks
        -> live agent/assistant-stream frames + durable assistant/message
        -> executeToolCalls()
          -> durable tool/call
          -> tools/pre-execute -> approval/guards -> tools/execute -> pwsh body
          -> tools/post-execute -> finalizeContent -> tools/result notification
          -> durable tool/result
        -> step/end; if more tool/user context, next step; otherwise turn/end
```

## 逐步追踪：输入、输出与持久化边界

| # | 调用与输入 | 输出 / 状态变化 | 证据 |
|---|---|---|---|
| 1 | `SessionController.prompt(request: SessionPromptRequest)` 接受 Session id、`content`、`requestId`、delivery `mode`。 | 先拒绝既无非空文本也无附件的提交；再 resolve live/resumable agent。 | [`packages/api/session-controller/src/commands.ts:297-321`](../deepseek-harness/packages/api/session-controller/src/commands.ts#L297-L321) |
| 2 | `content` 经 file-upload receipt、attachment admission 转为 `UserMessage`；`source` 含 `kind: 'user'` 与 `rpcId: requestId`。 | `mode === 'steer'` 调 `agent.steer(message)`，否则默认调 `agent.followup(message)`；本场景是后者。 | [`packages/api/session-controller/src/commands.ts:330-365`](../deepseek-harness/packages/api/session-controller/src/commands.ts#L330-L365) |
| 3 | `followup(input: UserMessage)`。 | 固定映射为 `send(input, 'next-turn', true)`；消息进入 Inbox，wake driver。 | [`packages/core/agent-loop/src/agent.ts:128-146`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L128-L146) |
| 4 | idle agent 被 wake。 | 生成新的 `AbortController`、进入 `running` phase，在 `ctx.agents.withInitiator(this, ...)` 中运行 driver；这使工具/事件可从 ambient initiator 找到此 agent。 | [`packages/core/agent-loop/src/agent.ts:179-208`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L179-L208) |
| 5 | `turn()`。 | 先 append durable `turn/start { turn }`，然后从 `next-turn` inbox claim 首批输入。 | [`packages/core/agent-loop/src/agent.ts:268-303`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L268-L303) |
| 6 | `preStep(target, {turn, step})`。输入是 claimed `UserMessage[]` 和 cancellation `AbortSignal`。 | 先 `systemPrompt.assemble(assembleContextFor(agent, signal))`，再生成 runtime context，最后经 agent-scoped `agent/pre-step` waterfall 给 listener 接受、改写或拒绝。输出为 `PreparedStep = reject | enter(messages, assembly, ...)`。 | [`packages/core/agent-loop/src/agent.ts:240-259`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L240-L259) |
| 7 | `SystemPrompt.assemble(AssembleContext)`。 | 叠加 global 与 scope-chain variables、sections、contexts 和 tool schema providers，排序工具后走 `system-prompt/assemble` waterfall；输出 `PromptAssembly { sections, contexts, tools, variables }`。 | [`packages/core/system-prompt/src/index.ts:552-627`](../deepseek-harness/packages/core/system-prompt/src/index.ts#L552-L627) |
| 8 | accepted `PreparedStep`。 | append durable `step/start {turn, step}`；若第一个 batch 被改为空，则 turn 不花 model step 即完成。 | [`packages/core/agent-loop/src/agent.ts:286-313`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L286-L313) |
| 9 | `prepareRequest(turn, step, signal)`。 | `agent/request` waterfall 产生/改写 `LlmCallConfig`；必须有 provider 和 model；`ctx.llm.prepareCall()` 绑定一次 adapter generation 与精确 model metadata。此 async 阶段被取消时尚未写 system/user 事件。 | [`packages/core/agent-loop/src/agent.ts:500-549`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L500-L549) |
| 10 | render 后的 prompt，accepted users，prepared config，`assembly.tools`。 | `SystemPromptProjection` 生成 `system/message` replacement/append；首次 attempt append `user/message`；随后记录 `request/header` / `request/context`，由 `session.deriveMessages()` 构建并 freeze `GenerateOptions`。 | [`packages/core/agent-loop/src/agent.ts:352-379`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L352-L379) [`packages/core/agent-loop/src/agent.ts:552-617`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L552-L617) |
| 11 | frozen `GenerateOptions`，或已绑定的 `PreparedLlmCall`。 | `preparedCall.stream(request)`（无 prepared call 时为 `ctx.llm.stream(request)`）返回 `AsyncIterable<StreamChunk>`；每个 chunk 推入 `AssistantStreamAttempt` 并 emit transient `agent/assistant-stream` frame。 | [`packages/core/agent-loop/src/agent.ts:379-397`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L379-L397) |
| 12 | LLM runtime 的 `stream()`。 | 使用 `llm/stream` waterfall 包住最终 adapter dispatch，因此 retry/metrics 等 plugin 可绕行包裹 streaming，而不改 loop。 | [`packages/llm/llm/src/index.ts:1106-1120`](../deepseek-harness/packages/llm/llm/src/index.ts#L1106-L1120) |
| 13 | DeepSeek adapter 的 bound connection/options。 | 一次 stream 固定 connection/credential snapshot；建立 SSE request 并通过 async iterator yield `StreamChunk`。caller abort、idle timeout、transport failures 均规整为 `LlmError`。 | [`packages/llm/llm-deepseek/src/adapter.ts:440-527`](../deepseek-harness/packages/llm/llm-deepseek/src/adapter.ts#L440-L527) |
| 14 | 完整 Assistant stream。 | 正常结束 append 一条 durable `assistant/message`，其中含 assembled `AssistantMessage`、usage 与完整 compact `stream`；错误/中断没有可见内容时写 `assistant/attempt`。live chunk 不是 durable record。 | [`packages/core/agent-loop/src/agent.ts:399-483`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L399-L483) [`docs/agent-lifecycle.md:83-89`](../deepseek-harness/docs/agent-lifecycle.md#L83-L89) |
| 15 | 从 assistant content 过滤出的 `ToolCallBlock[]`。 | 若为空，step outcome 是 `completed`；否则 `executeToolCalls()` 以本 turn/step/signal 调度，tool 产生的 `additionalContexts` 插入 `next-step` inbox。 | [`packages/core/agent-loop/src/agent.ts:484-492`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L484-L492) |
| 16 | 每个即将开始的 tool block `{ id, name, arguments }`。 | 在 body 前 append durable `tool/call {turn, step, callId, name, arguments}`；pre-policy 是有序的，而允许 parallel 的 body 可在 bounded pool 内并发。 | [`packages/core/agent-loop/src/tool-calls.ts:165-218`](../deepseek-harness/packages/core/agent-loop/src/tool-calls.ts#L165-L218) [`packages/core/agent-loop/src/tool-calls.ts:262-265`](../deepseek-harness/packages/core/agent-loop/src/tool-calls.ts#L262-L265) |
| 17 | `ToolRuntime.prepare()` 的 `ToolExecutionInput`，含不可变 caller `signal`、agent、call id、name、raw JSON args。 | snapshot/freeze JSON arguments；走 scoped `tools/pre-execute` waterfall；`ask` 会先调用 approval；之后执行不可被 later listener 放行的 monotonic `guard`。deny 转正常 `isError` result，不会中止整个 turn。 | [`packages/core/tools/src/index.ts:1453-1496`](../deepseek-harness/packages/core/tools/src/index.ts#L1453-L1496) |
| 18 | `pwsh` pre-policy 已允许。 | `tools/execute` waterfall 包住 `tool.execute()`，并把 wrapper signal 与 caller signal fuse；tool body 的异常、unknown tool 都规范化为 result。 | [`packages/core/tools/src/index.ts:1517-1568`](../deepseek-harness/packages/core/tools/src/index.ts#L1517-L1568) |
| 19 | `pwsh.execute(args, exec)`。 | 解析 session policy，若提供 `sandbox_permissions + justification` 先走 approval；构造 `{command, workdir, timeoutMs, dshEnv, sandboxPolicy, signal}`，经 `ctx.shell.resolve/run` 执行。 | [`packages/shell/tool-pwsh/src/index.ts:203-240`](../deepseek-harness/packages/shell/tool-pwsh/src/index.ts#L203-L240) [`packages/shell/tool-pwsh/src/index.ts:346-398`](../deepseek-harness/packages/shell/tool-pwsh/src/index.ts#L346-L398) |
| 20 | `SandboxPolicyService.resolve({session})`。 | per-call policy 优先级为 approved explicit mode → 最新 durable `sandbox/mode` → deployment default；workspace root 来自 session cwd 或 deployment fallback。 | [`packages/sandbox/sandbox-policy/src/index.ts:154-179`](../deepseek-harness/packages/sandbox/sandbox-policy/src/index.ts#L154-L179) |
| 21 | `ctx.approval` request。 | policy 是 `never` 时在 service 内、在 event dispatch 前直接拒绝；否则在 agent scope 走 `approval/request` waterfall，缺 answerer、throw 或非法返回都会 fail closed 为 `unavailable`。 | [`packages/interaction/user-approval/src/index.ts:260-299`](../deepseek-harness/packages/interaction/user-approval/src/index.ts#L260-L299) |
| 22 | tool body 的 canonical `ToolExecutionResult`。 | `tools/post-execute` 可接受、替换、block 或附加 context；result 再经 value schema、render/presentation-meta snapshot，最终以 frozen outcome 通知 `tools/result`。 | [`packages/core/tools/src/index.ts:1721-1813`](../deepseek-harness/packages/core/tools/src/index.ts#L1721-L1813) |
| 23 | model-order settled result 与先前 `tool/call` seq。 | append durable `tool/result`，`sourceEventSeqs: [callSeq]` 保持 call/result provenance；私有 UI `meta` 同时持久化。 | [`packages/core/agent-loop/src/tool-calls.ts:268-290`](../deepseek-harness/packages/core/agent-loop/src/tool-calls.ts#L268-L290) |
| 24 | 当前 step 的所有 results，以及 next-step inbox。 | append `step/end`；有 next-step work 时循环进入下一 step，否则 serial dispatch `agent/turn-stopping`、append `turn/end` 并变 idle。 | [`packages/core/agent-loop/src/agent.ts:304-349`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L304-L349) |

## 两个关键时序保证

### A. “模型可见即已记录”不是事后日志

`prepareRequest()` 在 `agent/request` 与 `prepareCall()` 成功前不写 system/user；随后 `buildRequest()` 先 append `request/header`、`request/context`，再从 Session 派生冻结的 `messages`。因此模型请求读取的是已提交 log projection，而不是临时 prompt array。[`packages/core/agent-loop/src/agent.ts:530-549`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L530-L549) [`packages/core/agent-loop/src/agent.ts:552-617`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L552-L617)

### B. 流式 UI 与 durable transcript 分离

`agent/assistant-stream` 用于过程中的 start/chunk/end；Session 只在 stream settle 时得到一个 `assistant/message` 或 `assistant/attempt`。所以正常重放不依赖 UI chunk 传输，但硬进程丢失发生在 settle 前时不会凭空拥有 durable partial attempt。[`docs/architecture.md:105`](../deepseek-harness/docs/architecture.md#L105) [`docs/architecture.md:115-121`](../deepseek-harness/docs/architecture.md#L115-L121)

## 最小关键片段

先准备 provider call，后写模型可见 events：

```ts
const { config, preparedCall } = await this.prepareRequest(turn, step, signal)
for (const { message, intent } of commits) {
  this.session.append('system/message', { turn, step, message }, intent)
}
if (firstAttempt) {
  for (const message of decision.messages) {
    this.session.append('user/message', message, { surfaceOp: 'append' })
  }
}
```

[`packages/core/agent-loop/src/agent.ts:362-378`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L362-L378)

工具调用 event 先于 pipeline，结果按模型顺序写回：

```ts
callSeqs[index] = appendToolCall(session, turn, step, call.block)
const prepared = await ctx.tools[TOOL_RUNTIME_SCHEDULER].prepare(call.exec)
// ... dispatch may overlap
const result = slot.needsPost
  ? await ctx.tools[TOOL_RUNTIME_SCHEDULER].finalize(slot.exec, slot.result)
  : ctx.tools[TOOL_RUNTIME_SCHEDULER].finish(slot.exec, slot.result)
appendToolResult(session, turn, step, call.block, result, callSeqs[committed]!)
```

[`packages/core/agent-loop/src/tool-calls.ts:147-176`](../deepseek-harness/packages/core/agent-loop/src/tool-calls.ts#L147-L176)

## 我读了哪些文件

- `docs/agent-lifecycle.md`
- `docs/tool-execution-pipeline.md`
- `docs/{architecture,capability-seams}.md`
- `packages/api/session-controller/{README.md,src/commands.ts}`
- `packages/core/{agent,agent-loop,system-prompt,tools}` 的 README 与本报告引用的 `src/*.ts`
- `packages/llm/{llm,llm-deepseek}` 的 README 与本报告引用的 `src/*.ts`
- `packages/shell/tool-pwsh/{README.md,src/index.ts}`
- `packages/sandbox/sandbox-policy/{README.md,src/index.ts}`
- `packages/interaction/user-approval/{README.md,src/index.ts}`

## 还有哪些没搞清楚

- `api-gateway` / `client-connection` 的具体 HTTP RPC framing、SSE follow framing，尚未逐函数追踪；它不影响本报告从 `SessionController.prompt()` 开始的 Host turn 链。
- `llm-deepseek` 的 SSE JSON frame 到每种 `StreamChunk` 的 translation（例如 reasoning/tool-call delta 的精确字段）尚未读 `sse.ts`、`translate.ts`；留给流式协议细节或阶段 3。
- `pwsh-sandbox` provider 如何在 Windows 上实际构造受限进程，尚未读 provider 实现；此处只验证了 tool 把 resolved `sandboxPolicy` 交给 shell seam。
