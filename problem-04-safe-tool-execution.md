# 问题四：DeepSeek Harness 如何让工具执行安全可控

## 结论

dsh 不把工具安全寄托在单一“沙箱开关”上，而是采用分层控制：

1. Tool registry 对 arguments、canonical output 和 cancellation 做统一 materialization。
2. `tools/pre-execute`、approval、monotonic guards、`tools/execute`、`tools/post-execute` 形成策略流水线。
3. shell、filesystem、code-runtime 在各自正确的执行层实施隔离。
4. Session log 在 body 前记录 `tool/call`，并在最终 normalization 后记录 `tool/result`，保持审计与 replay。

## 1. 工具定义将“模型接口”和“执行能力”分开

`ToolDefinition` 包含 `ToolSchema`、canonical output declaration、`execute()`、`timeoutMs`、`isConcurrencySafe()`、`finalizeContent()` 和 UI presenters；但模型请求只得到 name、description 和 parameters。[`docs/subsystems/tools.md:9-96`](../deepseek-harness/docs/subsystems/tools.md#L9-L96)

first-party tool 通过 `defineTool()` 将 parameter DSL 编译为 JSON Schema，并在 body 前验证 args；body 返回值再按 output schema 验证、deep-freeze，然后由 `output.render()` 转为模型 `ContentBlock[]`。[`docs/subsystems/tools.md:98-151`](../deepseek-harness/docs/subsystems/tools.md#L98-L151) [`packages/core/tools/src/index.ts:1782-1813`](../deepseek-harness/packages/core/tools/src/index.ts#L1782-L1813)

这防止工具把“执行成功但返回不可序列化/不符合声明的对象”悄悄写入会话。

## 2. 固定执行流水线

```text
ToolExecutionInput
  -> snapshot/freeze arguments + execution token
  -> tools/pre-execute waterfall
  -> approval ask（若需要）
  -> monotonic guards
  -> tools/execute waterfall
  -> ToolDefinition.execute()
  -> output validation + render
  -> tools/post-execute waterfall
  -> finalizeContent()
  -> immutable tools/result notification
  -> Session tool/result
```

生成的 pipeline 图明确标出 deny/approval/guard/around/body/post/finalization 顺序。[`docs/tool-execution-pipeline.md:4-60`](../deepseek-harness/docs/tool-execution-pipeline.md#L4-L60)

代码中 `prepareExecution()` 先执行 `tools/pre-execute`，把 `ask` 交给 approval，再执行 guard；deny 被 materialize 为普通 `isError` result，不让一个预期的策略拒绝摧毁整个 turn。[`packages/core/tools/src/index.ts:1453-1496`](../deepseek-harness/packages/core/tools/src/index.ts#L1453-L1496)

已允许的调用经 `tools/execute` around waterfall 后执行 body；wrapper 可以替换 signal，但 registry 会将其与 caller signal fuse。[`packages/core/tools/src/index.ts:1517-1568`](../deepseek-harness/packages/core/tools/src/index.ts#L1517-L1568)

## 3. 为什么需要 monotonic guard

普通 waterfall listener 可 short-circuit，也可包装下游结果；如果不可撤销的安全决策只作为可重排 listener，后续 plugin 可能改变结论。`ctx.tools.guard()` 因此位于 pre waterfall 之后，只允许返回 deny reason 或 abstain；一旦任何 guard deny，后续无法重新放行。[`packages/core/tools/README.md:79-85`](../deepseek-harness/packages/core/tools/README.md#L79-L85)

适合 monotonic guard 的是 owner-level invariant，例如 structured-output 已 capture 后禁止额外 side effects。适合 `tools/pre-execute` 的是可组合的人类审批、hook policy 与部署策略。

## 4. Approval：一次调用的一次授权

`ctx.approval` 在 agent scope dispatch `approval/request`。若 policy 为 `never`，service 在任何 listener 之前直接拒绝；若没有 answerer、answerer throw 或返回非法值，则 fail closed 为 `unavailable`；caller abort 返回 `cancelled`。[`packages/interaction/user-approval/src/index.ts:260-299`](../deepseek-harness/packages/interaction/user-approval/src/index.ts#L260-L299)

Shell escalation 必须同时给 `sandbox_permissions` 与 `justification`，并且 requested mode 必须严格宽于 effective mode；grant 只改变这一 call 的 mode，不持久修改 session policy。[`.agents/notes/implemented/feature/2026-07-14-cross-family-fs-sandbox.md:42-46`](../deepseek-harness/.agents/notes/implemented/feature/2026-07-14-cross-family-fs-sandbox.md#L42-L46)

## 5. Sandbox：共享策略，按 capability family 执行

早期只让 shell 执行 SandboxMode 会留下绕过路径：模型可以改用 `write`/`edit` 在进程内修改同一文件。Cross-family sandbox 决策因此建立唯一的 `ctx.sandboxPolicy`，并由 shell 和 fs consumers 在每次调用中解析同一 session mode 与 workspace root。[`.agents/notes/implemented/feature/2026-07-14-cross-family-fs-sandbox.md:7-28`](../deepseek-harness/.agents/notes/implemented/feature/2026-07-14-cross-family-fs-sandbox.md#L7-L28)

Policy 优先级是 explicit approved mode → session `sandbox/mode` projection → deployment default；workspace root 优先 session cwd。[`packages/sandbox/sandbox-policy/src/index.ts:154-179`](../deepseek-harness/packages/sandbox/sandbox-policy/src/index.ts#L154-L179)

### Shell 路径

`tool-pwsh` 把 `{ command, workdir, timeoutMs, dshEnv, sandboxPolicy, signal }` 交给 `ctx.shell` provider；真正的 PowerShell process confinement 由 `pwsh-sandbox`/shell provider 承担。[`packages/shell/tool-pwsh/src/index.ts:346-398`](../deepseek-harness/packages/shell/tool-pwsh/src/index.ts#L346-L398)

### Filesystem 路径

`dsh-fs-sandbox` 在 `ctx.fs` provider 内执行：read operations 放行；`read-only` 拒绝 mutation；`workspace-write` 对 canonical target 做 root containment，并在实际委托写入前再次 canonicalize；`danger-full-access` 不加 fence。[`.agents/notes/implemented/feature/2026-07-14-cross-family-fs-sandbox.md:30-40`](../deepseek-harness/.agents/notes/implemented/feature/2026-07-14-cross-family-fs-sandbox.md#L30-L40)

这是一条可信代码中的 path-policy boundary，不是 kernel boundary；设计笔记明确保留 resolve-to-syscall TOCTOU，只有 `openat2 RESOLVE_BENEATH` 等平台 primitive 才能彻底消除。[`.agents/notes/implemented/feature/2026-07-14-cross-family-fs-sandbox.md:38-40`](../deepseek-harness/.agents/notes/implemented/feature/2026-07-14-cross-family-fs-sandbox.md#L38-L40)

## 6. Timeout 与 cancellation

`ToolDefinition.timeoutMs` 只是声明。真正 enforcement 来自 `dsh-tool-call-timeout-policy` 的 `tools/execute` wrapper；它用 deadline signal 包住 `next()`，到期返回 `TOOL_TIMEOUT` result。[`docs/subsystems/tools.md:53-60`](../deepseek-harness/docs/subsystems/tools.md#L53-L60) [`packages/guard/timeout-policy/src/index.ts:51-77`](../deepseek-harness/packages/guard/timeout-policy/src/index.ts#L51-L77)

Cancellation 是 cooperative：body 必须观察/转发 `exec.signal`，registry 不会 abandon 已开始 promise；取消后先等待 work quiescence，再将成功 outcome 规范为 `ABORTED`。[`packages/core/tools/src/index.ts:1319-1327`](../deepseek-harness/packages/core/tools/src/index.ts#L1319-L1327) [`packages/core/tools/src/index.ts:1517-1549`](../deepseek-harness/packages/core/tools/src/index.ts#L1517-L1549)

所以 timeout 不能硬杀任意同进程 JavaScript；process tool 能否被真正停止取决于 shell/subprocess provider 的 kill semantics。

## 7. 并发执行与安全声明

每个 tool 可用纯同步 `isConcurrencySafe(args)` opt in。只有明确返回 `true` 才并行；missing、throw、invalid 或其他返回都 fail closed 为 exclusive。[`.agents/notes/implemented/feature/2026-07-10-parallel-tool-call-execution.md:15-25`](../deepseek-harness/.agents/notes/implemented/feature/2026-07-10-parallel-tool-call-execution.md#L15-L25)

Scheduler 只并发 dispatch/body；pre 和 post 按模型顺序执行。exclusive call 是 barrier；parallel group 使用 `maxParallelToolCalls` rolling pool。[`.agents/notes/implemented/feature/2026-07-10-parallel-tool-call-execution.md:27-49`](../deepseek-harness/.agents/notes/implemented/feature/2026-07-10-parallel-tool-call-execution.md#L27-L49)

局限是 classifier 只看单个 args，无法表达“两个写操作只有路径不同时才可并行”。设计选择牺牲一部分并行度以换取简单 fail-closed contract。[`.agents/notes/implemented/feature/2026-07-10-parallel-tool-call-execution.md:73-101`](../deepseek-harness/.agents/notes/implemented/feature/2026-07-10-parallel-tool-call-execution.md#L73-L101)

## 8. 输出控制：post policy、spill 与 compaction

`tools/post-execute` 可接受、block、替换 value/content 或附加 `additionalContexts`；失败 result 不允许被替换成成功 value。[`packages/core/tools/src/index.ts:1721-1770`](../deepseek-harness/packages/core/tools/src/index.ts#L1721-L1770)

Spill policy 在 post stage 将 oversized plain-text result 改为有界 preview + locator，完整文本交给 `ctx.spillStore`；save 失败时保留原 result，不把成功调用变失败。[`packages/spill/spill-policy/README.md:47-69`](../deepseek-harness/packages/spill/spill-policy/README.md#L47-L69)

Compaction pruner 则只在 context-pressure trigger 已成立时追加 replayable replacement；它不是执行期保密机制。[`packages/compaction/compaction-tool-result-pruner/README.md:58-82`](../deepseek-harness/packages/compaction/compaction-tool-result-pruner/README.md#L58-L82)

## 9. 审计顺序

Agent-loop 在 pre-execute 前 append `tool/call`，body 可并发，但 `tool/result` 按模型 call order commit，并以 `sourceEventSeqs: [callSeq]` 指向调用。[`packages/core/agent-loop/src/tool-calls.ts:146-230`](../deepseek-harness/packages/core/agent-loop/src/tool-calls.ts#L146-L230) [`packages/core/agent-loop/src/tool-calls.ts:262-290`](../deepseek-harness/packages/core/agent-loop/src/tool-calls.ts#L262-L290)

因此安全拒绝、异常、timeout 和正常结果都能进入同一模型历史/审计结构。

## 用于制作 PPT 的提示词

```text
制作中文技术 PPT《DeepSeek Harness 的安全工具执行流水线》，12 页，面向 Agent 安全和 runtime 工程师，16:9。强调“多层控制”而非“有沙箱所以安全”，明确 fs policy fence 与 kernel process sandbox 的差别。

页面：
1. 标题与四层结论。
2. ToolDefinition：model schema、canonical value、execute、UI projection 分离。
3. 完整 pipeline 总图。
4. pre-execute、approval、monotonic guard 的职责对照。
5. tools/execute around wrapper 与 timeout/cancellation。
6. output validation、post rewrite、finalizeContent、tools/result。
7. ctx.sandboxPolicy 的 mode/root 优先级。
8. Shell sandbox 路径。
9. Filesystem sandbox 路径与 TOCTOU 边界。
10. 并发 classifier、exclusive barrier、rolling pool、ordered commit。
11. spill 与 compaction：输出容量控制但不是保密边界。
12. 风险清单：同进程 cooperative cancellation、恶意插件、错误 classifier、provider kill 能力。

每页页脚必须列源码证据“相对路径:行号”，重点使用 docs/tool-execution-pipeline.md:4-60、packages/core/tools/src/index.ts:1453-1568、packages/interaction/user-approval/src/index.ts:260-299、.agents/notes/implemented/feature/2026-07-14-cross-family-fs-sandbox.md:7-46、.agents/notes/implemented/feature/2026-07-10-parallel-tool-call-execution.md:15-49。每页附 80～120 字讲稿。
```
