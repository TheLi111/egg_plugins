# DeepSeek Harness 架构研究（阶段 6）：可迁移设计模式、代价与重设计方向

## 结论摘要

DSH 最有价值的不是“有很多 plugin”，而是把 agent harness 中最易相互污染的四件事明确分开：能力接口与提供者、durable log 与 live stream、模型投影与 canonical result、root composition 与 agent-scoped composition。下表总结可迁移的具体决策。

## 12 个可迁移设计决策

| # | 问题 | DSH 的做法 | 代价与权衡 | 适用条件 |
|---|---|---|---|---|
| 1 | 如何让 agent loop、LLM、工具和存储可替换，而不是让一个核心类长成上帝对象？ | 用 Cordis `ctx.<key>` service definition / provider / consumer 三角色；例如 loop 只消费 `ctx.llm`，DeepSeek/Pi AI 作为 adapter provider 注册 route。 | interface、provider、consumer 要同时设计；过细 seam 增加 package、config 与诊断成本。 | 多 provider、多运行环境、或产品允许后续第三方能力实现时。 | 
| 2 | 配置改变/插件卸载时怎样避免残留 listener、tool 或 resource？ | 将 registration 放入 fiber-owned `ctx.effect()` / `ctx.on()`；fiber unload 按 disposer lifecycle 回收。 | plugin 代码必须有正确 disposer；异步 teardown、HMR 和依赖变化更难测试。 | 支持动态 composition、热加载、per-agent capability 时。 |
| 3 | 如何让同一进程的 Session 使用不同工具/提示词，而不污染彼此？ | agent preset 作为 standing composition，agent scope parent 到 preset mount；Cordis `isolate` 为相同 service key 建独立 realm。 | preset generation 不自动回收；README 明示每次 composition file 更新再新建 session 会增加 live watcher/tree。 | 多 persona、多权限模板、多租户/多项目 agent，但可接受 session 生命周期内 capability 固定。 |
| 4 | 如何让“模型看到了什么”可精确 replay、fork、审计？ | 先把 system/user/tool/assistant surface 写入 append-only Session log，再从 `deriveMessages()` 构造 frozen LLM request；dynamic stream 另走 live event。 | surface replacement/provenance/format migration 很复杂，write timing 与 cancellation 有严格要求。 | 高重现性、审计、会话迁移和 crash recovery 比最小实现更重要时。 |
| 5 | 流式 UI 怎样不破坏 durable model transcript？ | `agent/assistant-stream` 只传 start/chunk/end live frames；settle 时只写一个 `assistant/message` 或 `assistant/attempt`，其内嵌 compact stream。 | 硬崩溃发生在 settle 前无 durable partial attempt；UI follow protocol 需处理 frame gap/reopen。 | 有 Web/SDK streaming UX，但仍需 offline replay/导出时。 |
| 6 | 工具 API 如何同时服务模型、执行器、程序化调用和 UI？ | 一个 `ToolDefinition` 分出 model schema、canonical JSON output、`ContentBlock` renderer、durable presentation meta 和 optional host render intent。 | 定义较重，工具作者要理解多投影；Web 卡不直接复用 host presenter。 | 工具结果既有结构化下游使用，又有模型摘要、跨端 UI 与 replay 需求时。 |
| 7 | 如何允许策略插件扩展 tool pipeline，同时防止“后注册 listener 放行先前拒绝”？ | `tools/pre-execute` 是 reorderable waterfall；安全不可逆规则放 `ctx.tools.guard()` monotonic guard；timeout/retry/metrics 用 `tools/execute` around wrapper。 | 作者必须选对插入点；post transformer 的相对顺序仍是 composition policy。 | 有 approval、sandbox、retry、observability、内容脱敏等正交 policy 时。 |
| 8 | 工具并发怎样既提高吞吐又让 transcript 可理解？ | pre-policy 有序；允许的 body 进 bounded rolling pool；finalize 和 `tool/result` 按 model call order commit。 | 一个慢的前序 tool 会拖延后序结果对模型可见；并发安全只能由 tool author declaration 证明。 | 一个 model response 常产生多个独立 read-only call，且 transcript ordering 比 completion-order 更重要时。 |
| 9 | 如何控制长上下文而不把总结变成不可审计的黑盒？ | compaction 用 log-only start/summary/end 记录 transaction，真正模型 surface 是有 provenance 的 `user/message` replacement；先尝试 deterministic tool-result pruner，再可能 LLM summarize。 | 早期 replacement 改写前缀并损失原文可见性；字符预算不等于 token budget；summarization 增加模型调用。 | 长任务、context window 有限、且愿意用 summary 替换历史而保留完整 log 时。 |
| 10 | 如何让前端高效读取 Session 状态而不复制可变业务真相？ | `sessionProjections` 同步 fold committed event，`session-projection-cache` 持久 checkpoint；log leads cache，cache stale 只能落后不领先。 | 每个 event 驱动所有 unit；cache schema/version/lifecycle identity 维护成本高，cold refold仍由 consumer 负责。 | UI/listing 常读 todo/goal/title 等 current state，且会重启/冷读大量 session 时。 |
| 11 | 持久化格式演进如何避免“新版本覆盖旧文件后无法回退/诊断”？ | immutable version-named JSONL generations，adjacent migration packages，write-open 后才独占发布 final successor；旧 generation 不改名/不删除。 | migration catalog 与 strict historical validation 负担很重；每一格式变化需要边、fixture、publication discipline。 | 用户数据已发布、需要 crash-safe migration、可能保留历史 artifact 时。 |
| 12 | UI/外部协议如何扩展而不反向耦合 loop？ | external protocol/UI driver 通过 `ctx.agents` 与 `followup/steer/cancel` 交互；浏览器 business UI 使用 typed slot/conversation definition，renderer 独占 Cordis→React binding。 | Host/Client 有两套 type faces 和 Remote/Typert build complexity；特殊 UI 卡需 host metadata + client renderer 两处协作。 | 需要同一 agent engine 同时支持 Web、CLI、SDK、hooks、webhook 或 ACP 时。 |

## 每项的代码证据

1. capability seam 的三角色定义及 LLM provider/consumer map：[`docs/architecture.md:125-129`](../deepseek-harness/docs/architecture.md#L125-L129) [`docs/capability-seams.md:479`](../deepseek-harness/docs/capability-seams.md#L479)。
2. effect disposer 与 fiber teardown：[`vendor/cordis/src/fiber.ts:68-101`](../deepseek-harness/vendor/cordis/src/fiber.ts#L68-L101) [`vendor/cordis/src/fiber.ts:675-710`](../deepseek-harness/vendor/cordis/src/fiber.ts#L675-L710)。
3. standing preset mount / isolate：[`packages/preset/agent-presets/README.md:98-130`](../deepseek-harness/packages/preset/agent-presets/README.md#L98-L130) [`vendor/cordis/src/context.ts:109-125`](../deepseek-harness/vendor/cordis/src/context.ts#L109-L125)。
4. prepare-before-commit 与 log-derived request：[`packages/core/agent-loop/src/agent.ts:500-617`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L500-L617)。
5. stream settle semantics：[`docs/architecture.md:105`](../deepseek-harness/docs/architecture.md#L105) [`packages/core/agent-loop/src/agent.ts:399-483`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L399-L483)。
6. ToolDefinition field separation：[`docs/subsystems/tools.md:9-96`](../deepseek-harness/docs/subsystems/tools.md#L9-L96)。
7. pre/guard/execute/post pipeline：[`docs/subsystems/tools.md:170-208`](../deepseek-harness/docs/subsystems/tools.md#L170-L208) [`packages/core/tools/src/index.ts:1453-1568`](../deepseek-harness/packages/core/tools/src/index.ts#L1453-L1568)。
8. ordered model-facing commit despite pool dispatch：[`packages/core/agent-loop/src/tool-calls.ts:146-246`](../deepseek-harness/packages/core/agent-loop/src/tool-calls.ts#L146-L246)。
9. compaction log events/surface replacement 与 trigger：[`docs/subsystems/compaction.md:9-16`](../deepseek-harness/docs/subsystems/compaction.md#L9-L16) [`packages/compaction/compaction-basic/README.md:110-124`](../deepseek-harness/packages/compaction/compaction-basic/README.md#L110-L124)。
10. projection/cache consistency：[ `packages/session/session-projection/README.md:81-101`](../deepseek-harness/packages/session/session-projection/README.md#L81-L101) [`packages/session/session-projection-cache/README.md:55-67`](../deepseek-harness/packages/session/session-projection-cache/README.md#L55-L67)。
11. immutable generation/migration chain：[ `docs/architecture.md:115-123`](../deepseek-harness/docs/architecture.md#L115-L123)。
12. protocol/UI driver and renderer boundary：[ `docs/cookbook/extension-cookbook.md:35-89`](../deepseek-harness/docs/cookbook/extension-cookbook.md#L35-L89) [`packages/client/ui-renderer/README.md:34-58`](../deepseek-harness/packages/client/ui-renderer/README.md#L34-L58)。

## 观察到的弱点与技术债

以下前四项是源码/README 明示的限制，不是推测。

1. **preset generation 资源会累积。** Superseded preset generation 不回收，原因是没有 join count 来判断最后一个 session 是否离开；skill filesystem watcher 会随 edit-then-create cycle 叠加。[`packages/preset/agent-presets/README.md:171-181`](../deepseek-harness/packages/preset/agent-presets/README.md#L171-L181) 对长期运行、多次编辑 preset 的 daemon，这是真实的 resource-growth 风险。
2. **projection driver 是 eager all-unit-per-event。** README 明示每一个 registered unit 都会被每个 committed event 触及；当前依赖 cheap fold/reference identity gate，hot path 未来可能要 event-type prefilter。[`packages/session/session-projection/README.md:81-101`](../deepseek-harness/packages/session/session-projection/README.md#L81-L101) [`packages/session/session-projection/README.md:128-134`](../deepseek-harness/packages/session/session-projection/README.md#L128-L134)。
3. **compaction 的 pruner 是语法性裁剪。** head/tail 可以丢掉中间最关键内容，character budget 也不等于 provider token budget。[`packages/compaction/compaction-tool-result-pruner/README.md:127-136`](../deepseek-harness/packages/compaction/compaction-tool-result-pruner/README.md#L127-L136)。
4. **tool cancellation 是合作式的。** registry 不会硬杀同进程 body；tool 必须观察 `exec.signal` 并达到 quiescence。timeout 本身也只是一项需要 wrapper provider 才生效的声明。[`docs/subsystems/tools.md:25-60`](../deepseek-harness/docs/subsystems/tools.md#L25-L60)。
5. **hook compatibility 并非完全模拟外部产品。** Codex bridge 当前明示忽略 hook 的 `systemMessage`，并有 `updatedInput`、`continue:false` 的 TODO；因此“支持 Codex hooks”应理解为映射一个已选择的子集，而不是 drop-in behavioral equivalence。[`packages/hooks/hooks-codex/src/index.ts:161-171`](../deepseek-harness/packages/hooks/hooks-codex/src/index.ts#L161-L171) [`packages/hooks/hooks-codex/src/index.ts:254-276`](../deepseek-harness/packages/hooks/hooks-codex/src/index.ts#L254-L276)。
6. **Web tool presentation 有双投影成本。** Host `presentCall/presentResult` 不会被内建 Web 使用，Web 需在 `tool.call.toolview` 用 raw durable event 再导出；这是为跨端/replay 付出的重复逻辑成本。[`docs/cookbook/adding-a-tool.md:93-97`](../deepseek-harness/docs/cookbook/adding-a-tool.md#L93-L97)。

## 如果重新设计：三项推测性建议

### A. 为 preset mount 增加显式租约或 session refcount

**推测。** 可以让 agent join/handle dispose 维护 generation refcount，在计数归零且已被 supersede 时 dispose subtree/watcher；这能解决已知增长问题。代价是 agent creation rollback、fork/resume、异常进程退出和 standing composition 的 shared service 必须拥有严格的 lease lifetime，复杂度可能超过当前“进程结束统一释放”的策略。现有代码明确选择不做该回收，因此这应只在长驻、多 preset 编辑工作负载下考虑。[`packages/preset/agent-presets/README.md:171-177`](../deepseek-harness/packages/preset/agent-presets/README.md#L171-L177)

### B. 在 projection registry 编译 event-type interest index

**推测。** ProjectionDefinition 可以声明其消费的 event type set，由 registry 在 event 到来时只 drive relevant units。它会降低 all-units-per-event cost，但会破坏“任意 event 均可影响 fold”这一简单、通用的 API，并引入声明遗漏导致 stale projection 的风险。仅当 profiling 证明 projection drive 成为热路径时值得引入。[`packages/session/session-projection/README.md:128-134`](../deepseek-harness/packages/session/session-projection/README.md#L128-L134)

### C. 把 Web card derivation 提取为纯共享 projector library

**推测。** 可让 Host tool 与 Client card 从同一 browser-safe, JSON-only projection function 得到 card data，减少双实现。但它可能反而把 UI vocabulary 提升为 core tool ABI，并限制 Client 用 session-specific layout/interaction facts；DSH 现在明确倾向 Client 从 raw event 自主导出。适合多客户端确实需要相同 canonical card 的产品，而不一定适合 DSH 的可插拔 client 目标。[`packages/core/tools/README.md:87-89`](../deepseek-harness/packages/core/tools/README.md#L87-L89)

## 对其他 harness 的简短迁移建议

若要借鉴但不复制整个 DSH，优先按顺序采用：

1. 先建立“model-visible input 必有 durable event”的事件日志约束。
2. 将工具分成 schema、canonical value、model rendering、UI metadata 四层。
3. 用 around middleware 与 monotonic guard 分开可组合政策和不可撤销安全判定。
4. 把 per-agent capability set 放进 scope/preset，而不是在全局 registry 上临时开关。
5. 最后再引入 immutable format generations、projection/cache；这两项只有在 session 数据成为发布兼容性承诺后才值得承担复杂度。

## 我读了哪些文件

本阶段未新增大范围源码阅读；仅复核并综合前五阶段的 architecture、Cordis、agent-loop、tools、session、compaction、projection、preset、hook 与 client renderer 证据。关键引用均已在本文件“代码证据”和“弱点”部分列出。

## 还有哪些没搞清楚

- 没有使用真实 DeepSeek API 运行 profile，因此未测量 warm-prefix reuse、compaction quality、stream recovery 或 tool concurrency 的实际性能，只分析了实现契约。
- 未做跨项目对比；“可迁移”是由机制与代价推导的研究结论，不是对其他 harness 已验证的基准排名。
- Dynamic Cordis runner 的 VM guard 与 browser-half approval 仍值得单独做安全研究，尤其是 untrusted plugin code 的异步 escape 行为。
