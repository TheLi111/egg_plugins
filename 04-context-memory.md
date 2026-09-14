# DeepSeek Harness 架构研究（阶段 4）：上下文、压缩与会话记忆

## 1. System prompt 的来源与落盘方式

### 1.1 Assembly 的四类输入

一次 `SystemPrompt.assemble(AssembleContext)` 解析以下输入：

| 输入 | 注册 API | scope 规则 | 产物 |
|---|---|---|---|
| 静态/动态 prompt section | `ctx.systemPrompt.section()` | scoped 同名 section shadow global | `PromptAssembly.sections` |
| dynamic context | `ctx.systemPrompt.context()` | scoped 同名 context shadow global；可被 suppress | `PromptAssembly.contexts` |
| 模型工具定义 | `ctx.systemPrompt.tools()`（由 ToolRuntime 注册） | global + matching scope providers | `PromptAssembly.tools` |
| prompt variable | `ctx.systemPrompt.variable()` | nearest scoped value shadow global | `PromptAssembly.variables` |

`AssembleContext` 本身至少有 `{ scope?: ScopeKey, signal?: AbortSignal }`；agent loop 会加入其 agent scope 和本 turn signal。[`docs/subsystems/system-prompt.md:9-23`](../deepseek-harness/docs/subsystems/system-prompt.md#L9-L23) section 按 `(order, name)` 排序，文本可在 assembly 时求值并通过 `{{variable}}` 渲染；一个 effective `complete` section 会成为唯一的 system prompt。[`docs/subsystems/system-prompt.md:40-69`](../deepseek-harness/docs/subsystems/system-prompt.md#L40-L69) 工具 schema provider 的 `schemas` 是当前 assembly 唯一模型可见的 tool 集合。[`docs/subsystems/system-prompt.md:26-37`](../deepseek-harness/docs/subsystems/system-prompt.md#L26-L37)

代码的实际合并顺序是：先 global variable、再由远到近 scope variable；再 merge sections/contexts；收集 global 与 scope tool providers；最后将 assembly 交给 `system-prompt/assemble` waterfall。[`packages/core/system-prompt/src/index.ts:552-627`](../deepseek-harness/packages/core/system-prompt/src/index.ts#L552-L627)

### 1.2 prompt 如何进入模型历史

prompt 不作为 `GenerateOptions.system` 独立字段存在。agent-loop 将 `renderPrompt(assembly)` 经 `SystemPromptProjection` 变成 durable `system/message` surface node，再调用 `session.deriveMessages()` 得到 provider request history。[`packages/core/agent-loop/src/agent.ts:358-379`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L358-L379) [`packages/core/agent-loop/src/agent.ts:601-617`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L601-L617)

第一 step 的 system node 预留 head；prompt 改变时通常替换该节点。当 adapter prepared call 声明 `systemPromptUpdate: 'in-history'` 时，持续 request series 的非空更新可以追加在 cached history 后。这一语义由 subsystem 文档明确记录。[`docs/subsystems/system-prompt.md:40-45`](../deepseek-harness/docs/subsystems/system-prompt.md#L40-L45)

### 1.3 `agent-instructions`

`dsh-agent-instructions` 是 workspace instruction loader，目标是 `AGENTS.md`-compatible files：baseline instruction 在首次 request 前进入 durable context；之后成功的 fs read/write/edit touch 会把嵌套、变化或删除的 instruction 更新放进 inbox。[`packages/context/agent-instructions/src/index.ts:2-5`](../deepseek-harness/packages/context/agent-instructions/src/index.ts#L2-L5) [`packages/context/agent-instructions/src/index.ts:74-75`](../deepseek-harness/packages/context/agent-instructions/src/index.ts#L74-L75) 它不是一个“每次请求临时读文件”的 system-prompt plugin；其行为依赖 session durable context/inbox，从而让后续模型请求可由日志重建。

## 2. Compaction：何时触发、如何不破坏日志语义

`ctx.compaction` 是 optional capability seam，不属于 agent-loop spine；interface 依赖 Session 和 LLM，但 backend 可替换。[`docs/subsystems/compaction.md:1-7`](../deepseek-harness/docs/subsystems/compaction.md#L1-L7)

`compaction-basic` 有三条入口：

1. `auto: true` 时的 serial `agent/pre-step`：request derivation 前由 `ctx.tokenMeter` 按当前 routed model 的 capacity 计算压力。
2. `agent/request-error` 收到 provider 确认的 `CONTEXT_WINDOW_EXCEEDED`：绕开常规 threshold，尝试 maximal balanced head reduction；只有 surface replacement generation 变化才允许 retry。
3. `/compact` 触发的 standalone/manual compaction。

这些 trigger、先 prune 后 summarize 的顺序，以及 overflow-retry 条件由 backend README 明确说明。[`packages/compaction/compaction-basic/README.md:110-124`](../deepseek-harness/packages/compaction/compaction-basic/README.md#L110-L124) 它不会压缩 protected system head、tools 或无法安全切分的单一单位；也不能缩短 system prompt / tool schema / session prefix。[`packages/compaction/compaction-basic/README.md:10-12`](../deepseek-harness/packages/compaction/compaction-basic/README.md#L10-L12)

一次 summary 是一个日志事务样式的 bracket：`compaction/start` → LLM summary → `compaction/summary` → replacement `user/message` → `compaction/end`。三个 `compaction/*` 事件均 log-only；真正影响模型 surface 的是 replacement `user/message`，并带 `surfaceOp: { op: 'replace', ... }`。[`docs/subsystems/compaction.md:9-16`](../deepseek-harness/docs/subsystems/compaction.md#L9-L16) 这使 summary 既可被模型看到，也可保留 source range、原始模型输出、usage 等审计事实。

大 tool result 的先行 pruner 不是 summarizer：trigger 成立后，它用 head + marker + tail 生成可重放 replacement，并以 `compaction/prune` 为 token shadow-price 事件；若裁剪已解除压力，backend 不必再调用 summary model。[`packages/compaction/compaction-tool-result-pruner/README.md:74-82`](../deepseek-harness/packages/compaction/compaction-tool-result-pruner/README.md#L74-L82) [`packages/compaction/compaction-basic/README.md:87-89`](../deepseek-harness/packages/compaction/compaction-basic/README.md#L87-L89)

## 3. Session 持久化：append log、generation 与版本

当前 writer 常量是 `SESSION_FORMAT_VERSION = 3`。[`packages/core/session/src/types.ts:84-98`](../deepseek-harness/packages/core/session/src/types.ts#L84-L98) 文档 release record 同样声明当前已发布的 latestReleasedVersion 为 3（`dsh-v0.1.5-alpha.1`）。[`docs/session-format-status.md:16-33`](../deepseek-harness/docs/session-format-status.md#L16-L33)

JSONL provider 对一个 write handle 保持 per-handle mutation chain，并由 backend 内 `writers: Map<SessionId, ...>` 保证单进程单 writer；跨进程 lock 在 materializing write 前取得。live `session/event` 会被写入该 handle 的 buffered write-behind 队列。[`packages/session/session-persistence-jsonl/src/storage.ts:63-75`](../deepseek-harness/packages/session/session-persistence-jsonl/src/storage.ts#L63-L75) [`packages/session/session-persistence-jsonl/src/storage.ts:386-551`](../deepseek-harness/packages/session/session-persistence-jsonl/src/storage.ts#L386-L551)

generation 规则：v0 使用 `session.jsonl[.zstd]`；v1+ 使用 `session.vN.jsonl[.zstd]`。读取选数字最大的 canonical generation；write-open 以 adjacent migration chain 解码为 current logical events 后，独占发布 final version-named successor，旧 generation 不移动、不覆盖、不删除。[`docs/architecture.md:115-123`](../deepseek-harness/docs/architecture.md#L115-L123)

## 4. v0 → v1 → v2 → v3：每一边解决什么

| edge | 设计动机 / 主要变换 | 重要代价与约束 |
|---|---|---|
| v0 → v1 | 从 released-v0 physical JSONL/packed rows 读取到 shared-layout v1；对有限 legacy vocabulary 做 canonicalization，例如 `steering/message → user/message`、`compact/* → compaction/*`、补 message wrapper/deterministic IDs。 | source inventory 是 closed；unknown（即使 `ignorable`）和未审计 payload 拒绝，不猜测修复。 | 
| v1 → v2 | 将 top-level `assistant/chunk` 收拢为 `assistant/message.stream`；没有 surface message 的失败/取消/retry attempt 变 `assistant/attempt`。 | event cardinality 变化，必须 dense-remap declared same-session references；每个 v2 event 一行并从 `session/end-seed` 推 inherited cut。 |
| v2 → v3 | 将 historical request header 的 system prompt 移为 explicit system head/replacement `system/message`；将 `code` preset/PTC vocabulary 改为 `ptc`；canonicalize surface envelope endpoint 为 `startSeq/endSeq`。 | 插入 system events 改变 seq 和 inherited cut，因此 mapping 仅重写明确定义的 reference fields；事件内容和不相关数字字段不递归改写。 |

v0→v1 的 details 与拒绝原则见 [`packages/session/session-format-v0-to-v1/README.md:10-47`](../deepseek-harness/packages/session/session-format-v0-to-v1/README.md#L10-L47)。v1→v2 对 embedded assistant stream、attempt settlement 和 dense references 的说明见 [`packages/session/session-format-v1-to-v2/README.md:10-52`](../deepseek-harness/packages/session/session-format-v1-to-v2/README.md#L10-L52)。v2→v3 的 system head transformation、PTC vocabulary 和 envelope canonicalization 见 [`packages/session/session-format-v2-to-v3/README.md:54-110`](../deepseek-harness/packages/session/session-format-v2-to-v3/README.md#L54-L110)。

这三条 edge 都是状态化、adjacent、source-preserving transformer；迁移包不自行发布文件。full restore 的 caller 不能把 stage 中间产物当成功，最终 target validation / immutable successor publication 属于 catalog 与 JSONL persistence。[`packages/session/session-format-v2-to-v3/README.md:39-49`](../deepseek-harness/packages/session/session-format-v2-to-v3/README.md#L39-L49)

## 5. Projection 与 cache：把日志变成可读状态，但不制造第二真相

`ctx.sessionProjections` 的 unit 是同步 fold：`{ key, stateVersion, stateSchema, init, apply, wire? }`。它订阅一次 `session/event`，每个 committed event 驱动已注册 unit；`Object.is` reference gate 决定是否做下游 view/publish。`snapshot(session)` 产出 `{ asOfSeq, values }`，其中 `asOfSeq` 表示每个 value 都已反映的共同历史 cut。[`packages/session/session-projection/README.md:28-69`](../deepseek-harness/packages/session/session-projection/README.md#L28-L69) [`packages/session/session-projection/README.md:81-101`](../deepseek-harness/packages/session/session-projection/README.md#L81-L101)

`session-projection-cache` 将每个 session 的 checkpoint 存为 version-stamped record：创建、`turn/end`、dispose 是 mandatory write；中间由 `writeEveryEvents` / `writeIntervalMs` throttle。写入前 flush log，因此 crash 最多让 cache 落后，绝不能领先 committed log。[`packages/session/session-projection-cache/README.md:55-67`](../deepseek-harness/packages/session/session-projection-cache/README.md#L55-L67)

冷读遵循两层：identity/version/schema 匹配时 `cachedSnapshot()` zero-I/O；否则 consumer 读取 persistence log，通过 `restoreFloor()` 从 checkpoint 后 refold。cache 自己不读 session persistence，malformed/stale record 视作 absent 或 backup-and-skip。[`packages/session/session-projection-cache/README.md:61-67`](../deepseek-harness/packages/session/session-projection-cache/README.md#L61-L67) [`packages/session/session-projection-cache/README.md:79-93`](../deepseek-harness/packages/session/session-projection-cache/README.md#L79-L93)

## 6. 研究判断

- DSH 的“记忆”主要是 append-only Session event log，而不是向量检索或一个 mutable conversation object；model history、projection、cache、fork、telemetry 都是它的消费者。[`docs/architecture.md:115-121`](../deepseek-harness/docs/architecture.md#L115-L121)
- prompt、tool result compaction、instruction update 都通过 durable surface/event 进入后续模型 history，这统一了 replay 与审计，但使 surface replacement 的 provenance/sequence 规则成为格式演进的高复杂度区域。
- **推测：**v3 将 prompt 从 request header 移到 explicit `system/message`，除了提升 replay fidelity，也为 system prompt 的 in-history append/replacement 和 compaction interaction 提供统一 surface mechanism；直接证据是 v3 migration 的 system-head insertion 和 current architecture 对 `system/message` history 的描述，但“主要动机权重”需要设计记录才能完全确认。[`packages/session/session-format-v2-to-v3/README.md:65-74`](../deepseek-harness/packages/session/session-format-v2-to-v3/README.md#L65-L74) [`docs/subsystems/system-prompt.md:40-45`](../deepseek-harness/docs/subsystems/system-prompt.md#L40-L45)

## 我读了哪些文件

- `docs/{session-format-status,architecture}.md`
- `docs/subsystems/{system-prompt,compaction,session}.md`
- `packages/core/{session,system-prompt}` 的 README 与引用源码
- `packages/context/agent-instructions/{README.md,src/index.ts}`
- `packages/compaction/{compaction,compaction-basic,compaction-tool-result-pruner}` 的 README 与引用源码
- `packages/session/{session-persistence,session-persistence-jsonl,session-projection,session-projection-cache}` 的 README 与引用源码
- `packages/session/session-format-{v0-to-v1,v1-to-v2,v2-to-v3}/README.md` 及 migration entry 定位

## 还有哪些没搞清楚

- JSONL 的 zstd framing、recoverable torn-tail decode 和 successor atomic publication 的精确系统调用路径未逐函数检查；本报告只确认 handle ownership 和 generation 语义。
- agent-instructions 的具体 instruction-file search order、digest/version comparison 和 fs-event reconciliation 尚未逐行梳理。
- compaction summarizer 的实际 prompt template 与 `summarize()` result validation 没有展开；下一轮如需研究 prompt quality，应读 `compaction-basic/src/summarizer.ts`。
