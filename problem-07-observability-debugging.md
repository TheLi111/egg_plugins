# 问题七：DeepSeek Harness 如何让框架可观测、可调试

## 结论

dsh 的可观测性建立在四条互补路径上：

1. **Session event log**：记录模型可见输入、assistant stream settlement、工具调用/结果和控制生命周期，是 replay 与因果追踪的权威。
2. **Live events**：`agent/*`、`workflow/*`、`subagent/*`、`tools/result` 等提供当前进程中的增量状态。
3. **Projection 与 SessionQuery**：把 raw log 转为一致 point-in-time state、surface、lineage、replacement/source trace 和全文搜索。
4. **Runtime invariants 与 telemetry**：前者在进程内检查不可能状态，后者在显式反馈授权下导出经过处理的 Session prefix。

“精确理解 Agent 为什么这么做”需要限定：dsh 可以重建模型收到什么、返回什么、调用了什么、哪些策略生效以及事件之间的 provenance；它不能恢复模型没有输出的私有 chain-of-thought。`reasoning` block 若 provider 实际输出并进入 stream，则属于记录内容；未输出的内部推理不存在可观测证据。

## 1. Session log：行为证据的主轴

架构要求任何进入 model request 的输入都能从 Session log 重建；新 model-visible input 必须新增 Session event，而不能只存在于 transient Context。[`docs/architecture.md:115-123`](../deepseek-harness/docs/architecture.md#L115-L123)

一次 turn 会记录：

- `turn/start/end` 与 `step/start/end`；
- `system/message`、`user/message`；
- `request/header`、`request/context`；
- `assistant/message` 或失败的 `assistant/attempt`，包含 compact timed stream；
- `tool/call` 与 `tool/result`，后者引用前者的 event seq。

完整顺序见 [`docs/architecture.md:80-105`](../deepseek-harness/docs/architecture.md#L80-L105)。工具结果通过 `sourceEventSeqs: [callSeq]` 保存 provenance，并保留 structured error info 与 UI metadata。[`packages/core/agent-loop/src/tool-calls.ts:262-290`](../deepseek-harness/packages/core/agent-loop/src/tool-calls.ts#L262-L290)

因此调试者能回答：

- 哪一 request route/model 产生了输出；
- 当时的 system/user/tool history 是什么；
- model 发出了哪些 tool arguments；
- tool 是否被 deny、timeout、abort 或正常完成；
- compact/replacement 后哪些 source events 不再进入当前 surface；
- turn 最终因 completed、blocked、aborted、max-tokens 或 error 等原因结束。

## 2. Durable settlement 与 live stream 的分工

`agent/assistant-stream` 发送 process-local start/chunk/end frame；正常完成后，loop 只写一条 durable `assistant/message`，失败或重试 attempt 则写 `assistant/attempt`。[`packages/core/agent-loop/src/agent.ts:379-483`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L379-L483)

这两条数据流服务不同问题：

- UI 用 live frames 显示正在生成的 token；
- replay/export/query 使用 durable settlement；
- crash 恰好发生在 settlement 前时，未落盘的 partial stream 不被伪造成完整历史。

Agent lifecycle 文档明确建议 SDK 的 replayable transcript 消费 `session/event`，把 `agent/*` 留给 live coordination。[`docs/agent-lifecycle.md:83-89`](../deepseek-harness/docs/agent-lifecycle.md#L83-L89)

## 3. SessionProjection：把事件转为可消费状态

Raw event 对 UI 和查询并不总是便利。`ctx.sessionProjections` 允许 domain 注册同步 `init/apply/view` unit；每个 committed event 驱动 fold，snapshot 用 `asOfSeq` 标记所有返回值共同反映的 history cut。[`packages/session/session-projection/README.md:28-69`](../deepseek-harness/packages/session/session-projection/README.md#L28-L69)

适合 projection 的事实包括 title、model selection、agent preset、subagent identity/timing、goal、todo 等。它们由各自 domain package 定义，Session transport 和 Client store 不理解具体语义。[`.agents/notes/implemented/architecture/2026-08-25-session-observations-and-projection-owned-client-state.md:64-76`](../deepseek-harness/.agents/notes/implemented/architecture/2026-08-25-session-observations-and-projection-owned-client-state.md#L64-L76)

Projection cache 提供 cold-read shortcut，但 log 仍为 authority：cache 写前先 flush log，故 cache 只能落后；identity/version/schema 不匹配时重新 fold。[`packages/session/session-projection-cache/README.md:55-67`](../deepseek-harness/packages/session/session-projection-cache/README.md#L55-L67)

## 4. SessionObservation：一个一致的 point-in-time read

多个 endpoint 若分别从 live Session、persistence、cache 拼接 header/events/projection，可能得到不同时间点。设计笔记因此定义 retained `SessionObservation`：一次 observation 绑定 source kind、header、contiguous event prefix、cursor、projection snapshot 与 durable revision。[`.agents/notes/implemented/architecture/2026-08-25-session-observations-and-projection-owned-client-state.md:7-21`](../deepseek-harness/.agents/notes/implemented/architecture/2026-08-25-session-observations-and-projection-owned-client-state.md#L7-L21)

Live source 优先；否则使用由 persistence revision keyed 的 prepared cold cache。所有字段来自同一 lifecycle witness，caller 必须 dispose observation lease。[`.agents/notes/implemented/architecture/2026-08-25-session-observations-and-projection-owned-client-state.md:46-76`](../deepseek-harness/.agents/notes/implemented/architecture/2026-08-25-session-observations-and-projection-owned-client-state.md#L46-L76)

这避免调试页面出现“header 是旧的、events 是新的、projection 又来自另一个 live cut”的混合观察。

## 5. SessionQuery：读取、过滤、trace 与全文搜索

`ctx.sessionQuery` 将 live Session 与 persistence 统一成 live-preferred logical corpus。公共能力包括：

| 能力 | 用途 |
|---|---|
| `readSession(id)` | replay-validate 完整 raw event log，不激活 Agent |
| `readSurface(id)` | 返回当前模型/用户可见 surface |
| `listEvents` / `filterEvents` | 结构与语义文本过滤 |
| `traceSession(id)` | parent lineage 和 descendant tree |
| `traceEvent(request)` | replacement 与 cited-source 关系 |
| `searchSessions` / `searchEvents` | 由 SQLite FTS5 backend 实现 ranked search |

API 概览见 [`packages/session-query/session-query/README.md:26-51`](../deepseek-harness/packages/session-query/session-query/README.md#L26-L51)。Exact read/filter/trace 在 Definition 内实现，full-text search 是 provider-specific abstract surface。[`packages/session-query/session-query/README.md:77-115`](../deepseek-harness/packages/session-query/session-query/README.md#L77-L115)

`readSurface`、`listEvents` 与 `traceEvent` 共享同一个 `foldSurface` validation，区分 `current | shadowed | log-only`，校验 seq、replacement 和 citations。因此搜索结果与 model-history derivation 不使用互相矛盾的 surface 算法。[`packages/session-query/session-query/README.md:111-115`](../deepseek-harness/packages/session-query/session-query/README.md#L111-L115)

SQLite backend 是 derived index，不是 Session authority；搜索 index 可重建，exact read 仍来自 logical corpus。[`packages/session-query/session-query-sqlite/README.md:10-12`](../deepseek-harness/packages/session-query/session-query-sqlite/README.md#L10-L12)

## 6. Runtime invariants：发现“每条记录都合法但组合不可能”的状态

类型与单条 schema 无法发现这些关系错误：

- seq 不单调；
- step 不在 turn 内；
- tool result 没有同 step call；
- stream delta 指向未打开 block；
- Agent status 出现非法 transition；
- loop-built request 与其 log prefix 无法重建为同一请求。

`dsh-invariants` 让拥有这些关系的 package 发布 `./invariant` companion；中央 registry 只负责 selection、child fiber、rollback 与 package-attributed `InvariantError`。[`packages/runtime-diagnostics/invariants/README.md:26-36`](../deepseek-harness/packages/runtime-diagnostics/invariants/README.md#L26-L36) [`packages/runtime-diagnostics/invariants/README.md:88-118`](../deepseek-harness/packages/runtime-diagnostics/invariants/README.md#L88-L118)

设计规则是只检查独立可观察的 runtime relationship，不重复 method existence、plugin name 或 TypeScript contract。[`.agents/notes/implemented/architecture/2026-07-19-package-invariant-runtime-contracts.md:7-26`](../deepseek-harness/.agents/notes/implemented/architecture/2026-07-19-package-invariant-runtime-contracts.md#L7-L26)

代表性 checks 涵盖 Session、Agent、Scope、AgentLoop、LLM、tools、compaction、hooks、sandbox、fs、subagent、workflow、goal 和 jobs。[`.agents/notes/implemented/architecture/2026-07-19-package-invariant-runtime-contracts.md:28-56`](../deepseek-harness/.agents/notes/implemented/architecture/2026-07-19-package-invariant-runtime-contracts.md#L28-L56)

注意：`dsh-base` 默认不加载 diagnostics；registry 本身也不带产品 checks。自定义部署必须显式装配 companions。[`packages/runtime-diagnostics/invariants/README.md:26-36`](../deepseek-harness/packages/runtime-diagnostics/invariants/README.md#L26-L36)

## 7. Telemetry：可观测性与数据授权分离

默认 base 将 Session telemetry 配成 feedback-only，而不是持续上传。普通活动不触发 capture；显式 feedback 才释放一个 bounded canonical prefix。[`packages/bundle/base/cordis.patch.yml:168-197`](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L168-L197)

反馈门控设计的原因是：用户报告问题时需要同时分享发生问题的 Session，但不应默认持续采集所有 session。[`.agents/notes/implemented/feature/2026-08-25-feedback-gated-telemetry-default.md:7-17`](../deepseek-harness/.agents/notes/implemented/feature/2026-08-25-feedback-gated-telemetry-default.md#L7-L17)

边界如下：

- `DSH_TELEMETRY_DISABLED` 非空值在 plugin load 前 hard opt out；
- ordinary request/lifecycle 不触发 export；
- feedback 时复制并 redact canonical log；
- 若部署无 redaction rule，export 仍可能包含 message text、tool args/results 和 workspace paths；
- command acknowledgement 只确认已记录，不保证网络交付成功。

这些 consequence 由设计笔记明确指出。[`.agents/notes/implemented/feature/2026-08-25-feedback-gated-telemetry-default.md:27-31`](../deepseek-harness/.agents/notes/implemented/feature/2026-08-25-feedback-gated-telemetry-default.md#L27-L31)

## 8. “为什么”能够回答到什么程度

dsh 可以用证据回答：

- **上下文原因**：模型当时收到的 system/user/tool history；
- **路由原因**：request header 中 provider/model/reasoning config；
- **控制原因**：pre-step、request retry、approval、sandbox、goal/plan/compaction 的 durable/live records；
- **行为结果**：assistant stream、tool calls/results、error codes、turn end reason；
- **历史变换原因**：surface replacement 及 `sourceEventSeqs`。

dsh 不能仅凭这些记录证明：

- 模型未输出的内部思维过程；
- 某段自然语言 reasoning 是否真实反映模型内部计算；
- 外部 provider 未返回的 server-side routing/cache decision；
- 一个 goal 的 completion claim 是否客观正确（当前无独立 certificate）。

因此更准确的表述是“重建可观察的决策上下文与执行因果链”，不是“读取 Agent 的真实内心理由”。

## 9. 成本与缺口

- Exact read/filter/trace 会读取并验证完整 logical log，大 Session 有明显成本。[`packages/session-query/session-query/README.md:143-153`](../deepseek-harness/packages/session-query/session-query/README.md#L143-L153)
- Projection eager drive 每个 event 触及所有 units；当前依赖同步 cheap fold 与 reference gate。[`packages/session/session-projection/README.md:128-134`](../deepseek-harness/packages/session/session-projection/README.md#L128-L134)
- Live-only invariant companion 在 reload 前的操作无法重建；只有 session-backed companion 可 replay baseline。[`packages/runtime-diagnostics/invariants/README.md:145-155`](../deepseek-harness/packages/runtime-diagnostics/invariants/README.md#L145-L155)
- Telemetry 的可诊断性与隐私/交付可靠性有显式权衡。
- Log 完整不等于 UI 一定显示全部；Client/Host presentation 是另一个投影层，调试时应保留 raw events。

## 用于制作 PPT 的提示词

```text
请制作中文技术 PPT《DeepSeek Harness 的可观测性与调试架构》，12 页，16:9，面向 Agent 研究者和平台 SRE。风格克制、证据导向。必须在结论中区分“可观察执行因果链”与“模型私有思维链”。

页面：
1. 标题与四条观测路径总览。
2. Session log 记录哪些事实。
3. model-visible means logged：请求重建链。
4. durable assistant settlement 与 live stream 双通道。
5. surface replacement/sourceEventSeqs 的 provenance 图。
6. SessionProjection 与 asOfSeq。
7. SessionObservation：live/cold source 的一致 point-in-time cut。
8. SessionQuery：read、surface、filter、trace、SQLite FTS search。
9. Runtime invariants：单条合法但跨事件不可能的例子。
10. Telemetry 的 feedback-only 授权边界。
11. “为什么”能回答与不能回答的两列表。
12. 性能、隐私、默认未启用 diagnostics 等限制。

每页页脚放相对路径:行号。重点引用 docs/architecture.md:80-123、packages/core/agent-loop/src/tool-calls.ts:262-290、.agents/notes/implemented/architecture/2026-08-25-session-observations-and-projection-owned-client-state.md:15-76、packages/session-query/session-query/README.md:26-51、.agents/notes/implemented/architecture/2026-07-19-package-invariant-runtime-contracts.md:28-56、.agents/notes/implemented/feature/2026-08-25-feedback-gated-telemetry-default.md:7-31。每页附 80～120 字讲稿。
```
