# 问题三：DeepSeek Harness 如何让会话可持久化、可恢复

## 结论

dsh 使用 append-only `SessionEvent` log 作为会话唯一事实来源：Agent loop 先记录模型可见输入，再从日志推导请求；JSONL provider 异步持久化同一事件流；resume、fork、projection、query 与 UI replay 都消费该日志或其验证过的投影。

这是一种事件溯源设计，但应准确描述能力边界：当前代码提供恢复、fork、历史回放、surface replacement trace 和 immutable generation migration；它不提供“任意选中某个旧时点、原地回滚当前 Session 并继续写”的通用时间旅行 API。

## 1. 为什么 Session log 是唯一事实源

架构文档明确规定 Session log 是模型所见上下文的来源，`deriveMessages()` 从日志构造 model history；assistant 的完整 compact stream 嵌入最终 `assistant/message` 或 `assistant/attempt`。[`docs/architecture.md:115-121`](../deepseek-harness/docs/architecture.md#L115-L121)

在真实 turn 中，loop 的顺序是：

1. `prepareCall()` 成功；
2. append `system/message` 和首次 `user/message`；
3. append `request/header` / `request/context`；
4. `session.deriveMessages()`；
5. 将冻结请求交给 adapter。

代码证据位于 [`packages/core/agent-loop/src/agent.ts:352-379`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L352-L379) 和 [`packages/core/agent-loop/src/agent.ts:552-617`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L552-L617)。因此 log 不是事后审计副本，而是 request construction 的输入。

## 2. 事件模型：结构事件、surface 事件和 log-only 事件

Core Session 至少使用以下三类事实：

- 生命周期：`turn/start`、`turn/end`、`step/start`、`step/end`；
- 模型 surface：`system/message`、`user/message`、`assistant/message`、`tool/result`；
- 请求和控制事实：`request/header`、`request/context`、`assistant/attempt`、`tool/call`、`session/end-seed` 等。

架构文档列出 turn/step/tool 的耐久事件与 live `agent/*` extension points。[`docs/architecture.md:80-105`](../deepseek-harness/docs/architecture.md#L80-L105) `deriveMessages()` 只把当前 surface 投影为 provider history；structural/log-only events 不直接成为 LLM message。[`docs/subsystems/session.md:597-606`](../deepseek-harness/docs/subsystems/session.md#L597-L606)

Surface replacement 不删除旧记录。新 event 使用 `surfaceOp: replace` 与 `sourceEventSeqs` 指向被替换来源，原记录仍留在 append-only log。Compaction、system prompt 更新和 tool-result pruning 因此能够改变后续模型所见 surface，同时保留完整来源。[`docs/subsystems/session.md:277-340`](../deepseek-harness/docs/subsystems/session.md#L277-L340)

## 3. 不可变存储边界

Session 在接受 event 时执行一次 lossless JSON snapshot，拒绝不支持的值，断开 caller-owned reference，然后 deep-freeze event 及其所有后代；`append()`、`session/event` observer 和 event read 返回同一不可变记录。[`.agents/notes/implemented/architecture/2026-06-11-dev-invariants-over-deep-readonly.md:17-27`](../deepseek-harness/.agents/notes/implemented/architecture/2026-06-11-dev-invariants-over-deep-readonly.md#L17-L27)

`deriveMessages()` 也返回 detached、deep-frozen Message 和 fresh array snapshot，避免 request-side mutation 回写日志。[`.agents/notes/implemented/architecture/2026-06-11-dev-invariants-over-deep-readonly.md:29-31`](../deepseek-harness/.agents/notes/implemented/architecture/2026-06-11-dev-invariants-over-deep-readonly.md#L29-L31)

代价是每次 event admission 有一次递归 snapshot/freeze 成本；之后历史读与缓存 projection 可以复用安全记录。[`.agents/notes/implemented/architecture/2026-06-11-dev-invariants-over-deep-readonly.md:53-60`](../deepseek-harness/.agents/notes/implemented/architecture/2026-06-11-dev-invariants-over-deep-readonly.md#L53-L60)

## 4. 内存 Session 与 JSONL persistence 的关系

`ctx.sessions` 维护 live in-memory Session；`ctx.sessionPersistence` 是持久化 seam。默认 base 选择 `session-persistence-jsonl`，根目录为 `$DSH_HOME/sessions`。[`packages/bundle/base/cordis.patch.yml:33-37`](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L33-L37) [`packages/bundle/base/cordis.patch.yml:110-113`](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L110-L113)

JSONL backend 监听 `session/event`，把 live event enqueue 到对应 active writer；显式 flush 会 drain buffer 并同步持久化。[`packages/session/session-persistence-jsonl/src/storage.ts:527-551`](../deepseek-harness/packages/session/session-persistence-jsonl/src/storage.ts#L527-L551) backend 用 writer map 拒绝同一实例内的第二 writer，并用跨进程 lease 保护 write open。[`packages/session/session-persistence-jsonl/src/storage.ts:386-430`](../deepseek-harness/packages/session/session-persistence-jsonl/src/storage.ts#L386-L430)

这形成两层一致性：

- Session append 是同步的 logical commit 和 live publication；
- persistence 是有序 write-behind，在 request/turn/checkpoint 等位置由显式 flush 收紧 durability window。

因此“崩溃不丢任何已 append event”不能无条件成立；必须区分 logical committed 与已经 flush 的 durable prefix。架构通过 checkpoint policy、turn boundaries 和 persistence flush 明确控制该窗口，而不是假装每个内存 append 都等同于 fsync。[`packages/core/session/src/types.ts:276-284`](../deepseek-harness/packages/core/session/src/types.ts#L276-L284)

## 5. 崩溃恢复和 open 语义

Session query 的 cold read 会通过 read handle 加载并 replay-validate 完整 logical log，而不会激活 Agent；若 writer 崩溃留下 open trailing turn，read-side 可以在内存中用 interrupted closer 平衡，不修改 persistence。[`packages/session-query/session-query/README.md:105-115`](../deepseek-harness/packages/session-query/session-query/README.md#L105-L115)

Resume path 则创建 live Agent，并在请求前重建 prompt、request route、goal/preset/sandbox 等 projection state。用于模型的历史仍来自 `deriveMessages()`，不是从 UI snapshot 反序列化。

## 6. Fork、回放与“时间旅行”的真实边界

`ctx.sessions.create(id, { seed, meta })` 是低层 seed/fork primitive；Agent 创建 API可在 completed turn boundary 复制父 Session prefix，并写 `parentSession` / seed metadata。[`docs/subsystems/session.md:608-616`](../deepseek-harness/docs/subsystems/session.md#L608-L616) In-process fork subagent 只复制 completed parent turns，排除正在执行且尚无 tool result 的 in-flight turn，以保持 valid replay history。[`.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md:51-57`](../deepseek-harness/.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md#L51-L57)

Session query 可：

- `readSession()` replay-validate 完整 log；
- `readSurface()` 折叠 current surface；
- `traceSession()` 遍历 parent/descendant；
- `traceEvent()` 追踪 positional replacements 与 cited source events。

这些 API 见 [`packages/session-query/session-query/README.md:26-51`](../deepseek-harness/packages/session-query/session-query/README.md#L26-L51)。它们支持历史研究与 lineage navigation，但没有提供就地改写过去的 mutating rewind。

## 7. 格式版本与不可变 generation

`SESSION_FORMAT_VERSION` 是单调整数；只有旧 runtime 无法语义正确读取新 log 的结构变化才 bump。普通 plugin event vocabulary 增长使用 per-event `ignorable`，默认未知 event 必须拒绝，避免静默丢失影响重建的事实。[`.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.md:11-20`](../deepseek-harness/.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.md#L11-L20)

读取规则：

- 相同版本：正常读取；
- 存储版本更高：以 `unsupported/newer harness` 拒绝，并保留 raw artifact；
- 存储版本更低：在内存中运行完整 adjacent migration chain；read open 可直接消费结果，write open 先独占发布 current generation。

完整规则见 [` .agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.md:13-23`](../deepseek-harness/.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.md#L13-L23)。

Generation 文件名为 v0 `session.jsonl[.zstd]`、正版本 `session.vN.jsonl[.zstd]`。迁移只写最终 current target；绝不移动、覆盖或删除 committed predecessor，也不持久化中间版本。[`.agents/notes/implemented/architecture/2026-08-31-released-session-format-migrations.md:103-109`](../deepseek-harness/.agents/notes/implemented/architecture/2026-08-31-released-session-format-migrations.md#L103-L109)

### v0 → v1 → v2 → v3

| Edge | 主要目的 |
|---|---|
| v0→v1 | 读取早期 packed layout，规范 legacy event names/wrappers/ids，升级到 shared v1 layout。 |
| v1→v2 | 消费 top-level assistant chunks，将 stream 嵌入 `assistant/message`，无 surface settlement 的 attempt 转为 `assistant/attempt`。 |
| v2→v3 | 将 request-header system prompt 迁为 explicit `system/message` head/replacement；规范 PTC 名称与 surface envelope references。 |

各 edge 的变换/拒绝范围分别由 [`packages/session/session-format-v0-to-v1/README.md:10-47`](../deepseek-harness/packages/session/session-format-v0-to-v1/README.md#L10-L47)、[`packages/session/session-format-v1-to-v2/README.md:10-52`](../deepseek-harness/packages/session/session-format-v1-to-v2/README.md#L10-L52)、[`packages/session/session-format-v2-to-v3/README.md:54-110`](../deepseek-harness/packages/session/session-format-v2-to-v3/README.md#L54-L110) 定义。

## 8. Projection/cache 为什么不成为第二事实源

`ctx.sessionProjections` 对 committed `session/event` 执行同步 fold，生成带 `asOfSeq` 的 complete snapshot。[`packages/session/session-projection/README.md:28-69`](../deepseek-harness/packages/session/session-projection/README.md#L28-L69) Projection cache 在写 checkpoint 前先 flush Session log，因此 cache 可以落后但不能领先；identity/version/schema 不匹配时丢弃 cache row 并从 log refold。[`packages/session/session-projection-cache/README.md:55-67`](../deepseek-harness/packages/session/session-projection-cache/README.md#L55-L67)

所以 cache 是 replay acceleration，不是独立数据库权威。

## 9. 代价与限制

- Event schema、surface replacement、reference remapping 和 migration test burden 很高。
- Exact read/filter/trace 仍需加载并验证完整 logical log；超大 Session 成本不低。[`packages/session-query/session-query/README.md:143-153`](../deepseek-harness/packages/session-query/session-query/README.md#L143-L153)
- Write-behind 意味着 durability 必须靠显式 checkpoint 定义，不能只看 `Session.append()`。
- Unknown required event 选择 fail closed，会牺牲部分兼容便利以避免静默错误恢复。
- Immutable generations 增加磁盘占用，但保留取证和人工恢复材料。

## 用于制作 PPT 的提示词

```text
请制作中文技术研究 PPT《DeepSeek Harness 的事件溯源会话：持久化、恢复与版本迁移》，12 页，16:9，面向研究 Agent runtime 的工程师。必须区分 logical commit、durable flush、live stream、durable event；不要把 fork/replay 夸大成任意时间旅行。

结构：
1. 标题与核心结论。
2. 为什么普通 mutable conversation state 不足。
3. SessionEvent 分类：lifecycle、surface、request/control。
4. “model-visible means logged”时序：append→deriveMessages→LLM。
5. immutable snapshot/deep-freeze 边界。
6. in-memory Session 与 JSONL write-behind/flush。
7. crash recovery、read open、resume 的差异。
8. fork/seed、lineage、traceEvent 能力与时间旅行边界。
9. projection/cache：log leads, cache follows。
10. 单调 format version 与 unknown-event ignorable 策略。
11. v0→v1→v2→v3 迁移时间线与 immutable generations。
12. 代价、风险与可迁移原则。

每页放 1～2 个相对路径:行号证据。重点引用 docs/architecture.md:80-121、packages/core/agent-loop/src/agent.ts:352-379、packages/core/agent-loop/src/agent.ts:552-617、.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.md:7-23、.agents/notes/implemented/architecture/2026-08-31-released-session-format-migrations.md:103-109。为每页提供 100 字左右演讲备注。
```
