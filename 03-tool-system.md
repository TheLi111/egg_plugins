# DeepSeek Harness 架构研究（阶段 3）：工具系统

## 1. 一个工具的五个不同投影

DSH 没有把“工具”缩成一个 JSON Schema。`ToolDefinition` 在一个 registration 中同时拥有五类信息，但它们去往不同消费者：

| 层 | 载体 | 谁消费 | 是否模型可见 |
|---|---|---|---|
| 模型调用面 | `ToolSchema { name, description, parameters }` | `SystemPrompt.assemble()` → `GenerateOptions.tools` | 是 |
| canonical output | `output.schema`、`output.render()` | ToolRuntime 验证 body 返回值并生成 `ContentBlock[]` | 结果内容可见；schema 不直接给模型 |
| 执行面 | `execute(args, exec)`、`timeoutMs`、`isConcurrencySafe()` | registry + scheduler | 否 |
| Host 呈现面 | `presentCall()`、`presentResult()` | CLI/Host presenter | 否 |
| replay UI metadata | `output.presentationMeta()` → `tool/result.data.meta` | Web client 的 raw-event-derived renderer | 否（但 durable） |

`ToolDefinition` 的字段分工与“不泄露 execution/presentation 字段到模型”由 subsystem 文档明确规定。[`docs/subsystems/tools.md:9-12`](../deepseek-harness/docs/subsystems/tools.md#L9-L12) `ToolRuntime.schemas()` 在 prompt assembly 时才建立 model-facing projection，因此 registration、scoped restriction 或 mode 变化会反映到下一次 request。[`packages/core/tools/README.md:101-104`](../deepseek-harness/packages/core/tools/README.md#L101-L104)

## 2. 注册、schema 与 scoped visibility

### 2.1 `defineTool()` → `ctx.tools.register()`

first-party author使用 `defineTool({ name, description, parameters, output, execute, ... })`。它从 author-facing `ParameterSchemaSpec` 编译 raw JSON Schema、在 execute 前验证 arguments，并让 TypeScript 将 `execute`、`render`、`presentationMeta` 的类型连接到同一 output schema。[`packages/core/tools/README.md:28-60`](../deepseek-harness/packages/core/tools/README.md#L28-L60) [`docs/subsystems/tools.md:134-151`](../deepseek-harness/docs/subsystems/tools.md#L134-L151)

注册的 disposer 是 Cordis effect 的一部分；global 与 agent-scoped registration 都走 `ctx.tools.register()`，scoped registration 能 shadow global。每次 assembly 时 `ctx.systemPrompt` 收集 tool providers，复制 `name/description/parameters`，排序后形成 `PromptAssembly.tools`。[`packages/core/tools/README.md:79-85`](../deepseek-harness/packages/core/tools/README.md#L79-L85) [`packages/core/system-prompt/src/index.ts:568-615`](../deepseek-harness/packages/core/system-prompt/src/index.ts#L568-L615)

`ToolRestriction { allow?, deny? }` 是 scope 上继承工具的 live filter：多个 filter 相交，但该 scope 自己注册的工具不受 inherited filter 隐藏。[`docs/subsystems/tools.md:153-168`](../deepseek-harness/docs/subsystems/tools.md#L153-L168)

### 2.2 raw JSON Schema 与 Typert 的关系

两者职责不同：

- 工具参数/输出使用 DSH 的受限 JSON-value DSL，编译为 provider-neutral raw JSON Schema；它跨 tool、subagent、workflow、MCP 使用。[`packages/core/tools/README.md:60`](../deepseek-harness/packages/core/tools/README.md#L60)
- Typert 是 Host/Client Remote API 的 TypeScript type graph 和 runtime registry；`typert-loader` 会从已配置 package 的 `./typert` export 导入 `TYPERT` manifest 后 `ctx.typert.register(manifest)`，以供 gateway/remote 生成与调用，而不是用来定义 model tool parameters。[`packages/typert/loader/src/index.ts:5-25`](../deepseek-harness/packages/typert/loader/src/index.ts#L5-L25) [`packages/typert/loader/src/index.ts:370-383`](../deepseek-harness/packages/typert/loader/src/index.ts#L370-L383)

因此可把 Typert 理解为“UI/Host 业务 API 的 typed wire contract”，把 `ToolSchema` 理解为“模型 function-calling contract”。它们都可使用 schema，但没有同一个注册表。这一分工由各自 `ctx.typert` 与 `ctx.tools` service key 也可直接验证。[`docs/capability-seams.md:493`](../deepseek-harness/docs/capability-seams.md#L493) [`docs/capability-seams.md:511`](../deepseek-harness/docs/capability-seams.md#L511)

## 3. Native、PTC 与 presentation

`dsh-tools` 的 `mode` 有 `native`、`ptc`、`both`：native 将所有 visible tool schema 直接交给模型；ptc 仅暴露保留的 `run_code` 和生成 SDK；both 同时暴露两者。[`packages/core/tools/README.md:62-77`](../deepseek-harness/packages/core/tools/README.md#L62-L77)

PTC 并不让模型代码直接拥有 host capability。`run_code` 内 SDK binding 仍会回到 ToolRuntime pipeline；sub-call 带 parent token、可记录 `tool/ptc-dispatch`，而在 `ptc` 模式下模型直接调用普通 tool name 会在 policy 前以 `UNKNOWN_TOOL` 拒绝。[`docs/subsystems/tools.md:185-208`](../deepseek-harness/docs/subsystems/tools.md#L185-L208) [`packages/core/tools/README.md:123-132`](../deepseek-harness/packages/core/tools/README.md#L123-L132)

Host presentation 是 `card` tagged union：pending `ToolCallView` 可为 `generic | terminal | diff`，completed `ToolResultView` 还可为 `search | read | web` 等；它是 UI-neutral render intent。[`docs/subsystems/tools.md:459-468`](../deepseek-harness/docs/subsystems/tools.md#L459-L468) 但内建 Web Client 不读取 `presentCall`/`presentResult` 的返回值，它以 raw call args、durable result content/failure/meta 导出自己的 card props。[`packages/core/tools/README.md:87-89`](../deepseek-harness/packages/core/tools/README.md#L87-L89)

## 4. 执行 pipeline 与 policy 插入点

```text
ToolExecutionInput
  snapshot + freeze JSON arguments + opaque token
  -> tools/pre-execute (waterfall: allow | deny | ask)
  -> ctx.approval for ask
  -> monotonic ctx.tools.guard() checks
  -> tools/execute (waterfall around body; timeout/retry/metrics)
  -> definition.execute()
  -> output schema validation + output.render()
  -> tools/post-execute (accept | replace | block | additionalContexts)
  -> finalizeContent()
  -> frozen ToolExecutionResult + tools/result
  -> agent-loop appends durable tool/result
```

这不是仅由文档描述的顺序：registry 的 `prepareExecution()` 明确先调用 `tools/pre-execute`，为 `ask` 调 `serviceAsk()`，随后询问 monotonic guard；deny 成为 materialized error result。[`packages/core/tools/src/index.ts:1453-1496`](../deepseek-harness/packages/core/tools/src/index.ts#L1453-L1496) 已允许的 call 经过 `tools/execute` waterfall，body 使用 caller signal 与 wrapper signal 的 fused signal；body 已开始后取消也要等待其 quiescence。[`packages/core/tools/src/index.ts:1517-1568`](../deepseek-harness/packages/core/tools/src/index.ts#L1517-L1568)

post stage 可 block、replace value/content 或追加 FIFO context；成功 value 会用定义的 output schema 验证，再由 `render()` 变成 model content，top-level call 的 `presentationMeta` 会被 snapshot。[`packages/core/tools/src/index.ts:1721-1813`](../deepseek-harness/packages/core/tools/src/index.ts#L1721-L1813)

### 4.1 关键策略插件

| 关注点 | 插入点与行为 | 证据 |
|---|---|---|
| approval | `tools/pre-execute` 的 `ask` 调 `ctx.approval`；approval service 的 `never` 在 event listener 之前直接拒绝，缺 answerer 也为 `unavailable`。 | [`packages/core/tools/src/index.ts:1465-1489`](../deepseek-harness/packages/core/tools/src/index.ts#L1465-L1489) [`packages/interaction/user-approval/src/index.ts:260-299`](../deepseek-harness/packages/interaction/user-approval/src/index.ts#L260-L299) |
| timeout | definition 的 `timeoutMs` 只是声明；`dsh-tool-call-timeout-policy` 在 `tools/execute` wrapper 建 deadline，deadline 触发时返回 `TOOL_TIMEOUT` error result。 | [`docs/subsystems/tools.md:53-60`](../deepseek-harness/docs/subsystems/tools.md#L53-L60) [`packages/guard/timeout-policy/src/index.ts:51-77`](../deepseek-harness/packages/guard/timeout-policy/src/index.ts#L51-L77) |
| sandbox | `tool-pwsh` 每次 `execute` 从 `sandboxPolicy` resolve per-session policy；显式 escalation 在 shell body 前经 approval。实际 shell provider 接收 `sandboxPolicy`，从而把执行隔离留给 provider seam。 | [`packages/shell/tool-pwsh/src/index.ts:203-240`](../deepseek-harness/packages/shell/tool-pwsh/src/index.ts#L203-L240) [`packages/shell/tool-pwsh/src/index.ts:346-398`](../deepseek-harness/packages/shell/tool-pwsh/src/index.ts#L346-L398) |
| repeat reminder | `dsh-repeat-tool-reminder` 监听 `tools/post-execute`，因此它看到的是已执行的结果，而不是模型发出的原始 call；它只能附加提示，不负责阻断。 | [`packages/guard/repeat-tool-reminder/src/index.ts:162-233`](../deepseek-harness/packages/guard/repeat-tool-reminder/src/index.ts#L162-L233) |
| spill | `dsh-spill-policy` 是 prepend 的 `tools/post-execute` listener；只替换 oversized plain-text final content 为 preview+locator，full text 留给 `ctx.spillStore`，后端失败则保留原结果。 | [`packages/spill/spill-policy/README.md:47-69`](../deepseek-harness/packages/spill/spill-policy/README.md#L47-L69) [`packages/spill/spill-policy/README.md:81-105`](../deepseek-harness/packages/spill/spill-policy/README.md#L81-L105) |
| compaction pruner | 它不是每次 tool result 的 middleware；仅在 compaction trigger 已成立时，把旧 `tool/result` 追加 replacement，保留原 event、以 `sourceEventSeqs` 建 provenance，随后 request 看到裁剪后的 surface。 | [`packages/compaction/compaction-tool-result-pruner/README.md:58-60`](../deepseek-harness/packages/compaction/compaction-tool-result-pruner/README.md#L58-L60) [`packages/compaction/compaction-tool-result-pruner/README.md:74-82`](../deepseek-harness/packages/compaction/compaction-tool-result-pruner/README.md#L74-L82) |

## 5. 工具调度与 durable ordering

Agent-loop 在每个 call body 开始前写 `tool/call`，可对 concurrency-safe calls 建 bounded rolling pool；但在模型顺序上调用 `finalize` 和 append `tool/result`，所以快完成的后一个 call 不会抢在前一个 call 的 durable result 之前进入 transcript。[`packages/core/agent-loop/src/tool-calls.ts:146-230`](../deepseek-harness/packages/core/agent-loop/src/tool-calls.ts#L146-L230) `tool/result` 用 `sourceEventSeqs: [callSeq]` 引回其 call，并持久化 presentation meta。[`packages/core/agent-loop/src/tool-calls.ts:262-290`](../deepseek-harness/packages/core/agent-loop/src/tool-calls.ts#L262-L290)

## 6. 观察

- “工具插件”不等于“执行进程”：`tool-pwsh` 只声明 `pwsh` schema 和将 request 转到 `ctx.shell`；sandbox、shell backend、approval、timeout 都可以分别替换或插入。这是 capability seam 在工具层的实际用法，而非抽象口号。[`docs/capability-seams.md:525`](../deepseek-harness/docs/capability-seams.md#L525) [`docs/capability-seams.md:528-530`](../deepseek-harness/docs/capability-seams.md#L528-L530)
- DSH 特意让结果至少有两种表示：canonical JSON `value` 用于程序语义；`ContentBlock[]` 用于模型历史；`meta` 供 durable UI replay。将这些混成单个“tool output string”会失去 schema validation、presentation replay 或 prompt-size policy 的插入位置。
- **推测：**因为 `tools/post-execute` 可改 content/value 且 plugins 可按注册顺序叠加，部署方需要审查 middleware composition；系统通过 monotonic guard 解决“安全决策不得被后续放行”，但对多个非安全 post-transformer 的语义顺序没有通用冲突检测。该推测基于 post waterfall 的可替换语义，而非已发现的缺陷。[`packages/core/tools/src/index.ts:1721-1770`](../deepseek-harness/packages/core/tools/src/index.ts#L1721-L1770)

## 我读了哪些文件

- `docs/subsystems/tools.md`
- `docs/tool-execution-pipeline.md`
- `packages/core/tools/{README.md,src/index.ts,src/schema.ts,src/ptc.ts,src/presentation.ts}`
- `packages/typert/{registry,loader}` 的 README 与 `src` 实现
- `packages/guard/{timeout-policy,repeat-tool-reminder}` 的 README 与 `src/index.ts`
- `packages/spill/{spill-policy,spill-local}` 的 README 与 `spill-policy/src/index.ts`
- `packages/compaction/compaction-tool-result-pruner/{README.md,src/index.ts}`
- `packages/{shell/tool-pwsh,interaction/user-approval,sandbox/sandbox-policy}` 的相关实现

## 还有哪些没搞清楚

- `run_code` 在 `ctx.codeRuntime` 的 worker-thread provider 中如何执行 TypeScript/如何回调 SDK binding，尚未读 code-runtime provider；阶段 5 会以扩展点方式补充。
- MCP client 如何把外部 server tool schema 转为 `ToolDefinition`，尚未读 `mcp-client`；阶段 5 补。
- 每一种 first-party tool 的 `presentCall`/`presentationMeta` 与 Web renderer 的对应关系未逐工具枚举；这属于前端/扩展点深挖，不影响核心 pipeline。
