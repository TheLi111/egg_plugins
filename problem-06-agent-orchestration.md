# 问题六：DeepSeek Harness 如何让 Agent 行为可编排

## 结论

dsh 没有抽象出一个包办所有场景的“Workflow DSL 内核”。它提供多层编排原语：

- AgentLoop：一个 Session 内 turn/step 与 inbox 的基础驱动；
- Agent public API：followup、steer、inject、cancel、whenIdle；
- Subagent seam：一次任务委派与 continuable child；
- Workflow seam：用脚本 fan-out/fan-in 多个 subagent；
- Goal domain + round driver：让一个目标跨多个 turn 持久存在并受预算约束；
- Jobs：将 background work 的控制统一为 list/output/kill。

这些模块通过 `Agent`、Session events 和 capability services 组合，避免把具体编排策略写进 `agent-loop`。

## 1. 基础原语：turn、step 与 inbox

一个 step 是一次模型请求及其工具调用；一个 turn 是零个或多个 step，从输入被 claim 前开始，到系统不再欠下一步工作时结束。[`docs/architecture.md:80-103`](../deepseek-harness/docs/architecture.md#L80-L103)

Agent 的三个消息动作具有不同调度语义：

| 动作 | Inbox target | 是否 wake | 用途 |
|---|---|---|---|
| `followup()` | `next-turn` | 是 | 普通新用户轮次 |
| `steer()` | `next-step` | 是 | 在当前 turn 的下一模型 step 注入纠偏 |
| `inject()` | `next-step` | 否 | 添加上下文但不唤醒 idle agent |

源码映射见 [`packages/core/agent-loop/src/agent.ts:128-147`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L128-L147)。Driver 在每个 step 后检查 next-step inbox；若为空则运行 `agent/turn-stopping` serial checkpoint，再结束 turn。[`packages/core/agent-loop/src/agent.ts:304-349`](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L304-L349)

这一层足以表达多轮对话、工具反馈导致的下一 step、hook 强制继续和异步 context 注入。

## 2. Subagent：把“委派”建成可替换 transport

Subagent 需要多个实现同时存在：便宜的 in-process child、fork child、ACP 外部进程、Codex、Claude Code 等。因此它不是 single provider service，而是 `ctx.subagents` named-provider registry。[`.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md:9-24`](../deepseek-harness/.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md#L9-L24)

三角色如下：

- Definition：`dsh-subagent` 定义 `SubagentRuntime`、`SubagentProvider`、`SubagentRun` 与 lifecycle events；
- Providers：spawn/fork in-process、ACP、Codex、Claude Code；
- Consumer：`dsh-tool-subagent` 将选定 provider 暴露为 model tool。

包边界见 [` .agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md:26-40`](../deepseek-harness/.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md#L26-L40)。

核心 protocol 是 `start(request) -> Promise<SubagentRun>`：

- publication 前失败 reject，由 provider rollback 未发布资源；
- publication 后 outcome 通过 `run.result` settle，并保留 child id；
- caller 持有 run 的 dispose authority；
- `dispose()` 取消剩余工作并等待 quiescence。

该 ownership boundary 见 [` .agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md:42-49`](../deepseek-harness/.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md#L42-L49)。

Fresh 与 fork 是不同 provider，不是 request flag。Fork 只 seed completed turn prefix；in-flight turn 因尚缺 subagent tool result，不能形成有效 replay history。[`.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md:51-57`](../deepseek-harness/.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md#L51-L57)

## 3. Capability negotiation：请求必须先被证明可支持

`agentOptions`、`outputSchema`、`depthLimit`、`toolFilter`、`persona` 是 provider 的 static capabilities。Runtime 在创建 run 前检查，缺 capability 就 `UNSUPPORTED_CAPABILITY` fail loud，禁止 accepted-then-ignored。[`.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md:46-49`](../deepseek-harness/.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md#L46-L49)

Continuable child 则以 `provider.prepareContinuable` 方法是否存在来表示 capability，避免 flag 和实现漂移。One-shot run 不假装支持 steer/resume。

## 4. 子会话隔离与父会话可见性

In-process child 有自己的 Session id 和 `parentSession` lineage，独立持久化；父 Session 只记录 subagent tool call/result，不复制 child 的每个 step/tool event。Remote provider 可只有 parent-scoped lifecycle id，而无本地 child Session。[`.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md:55-61`](../deepseek-harness/.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md#L55-L61)

这控制父上下文增长，也让 child transcript 可独立导航。代价是父模型默认只看到最终 child output，live chunk-to-parent progress 不是 seam 的保证。

## 5. Workflow：让脚本持有 fan-out plan

单个 subagent tool 只能委派一项任务。对多文件审计、批量迁移等 fan-out 工作，若父模型逐轮协调，所有中间结果都进入父上下文，并增加模型 round-trip。Dynamic workflow 决策因此让模型编写 JavaScript orchestration script，由 runtime 执行并持有 loop/branch/intermediate results。[`.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.md:7-19`](../deepseek-harness/.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.md#L7-L19)

`ctx.workflowEngine` 的 script API 包括：

- `agent(prompt, options)`；
- `parallel(thunks)`；
- `pipeline(items, ...stages)`；
- `phase(title)`、`log(message)`、`args`。

Workflow Definition/Provider/Consumer 分离：`dsh-workflow` 定义 contract，worker-thread engine 执行，`tool-workflow` 暴露 model schema。[`.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.md:21-41`](../deepseek-harness/.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.md#L21-L41)

Worker thread 提供非阻塞 host、serialization boundary 和可强制 termination，但不是 hostile-code sandbox；脚本拥有与模型 bash access 相同的信任前提。[`.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.md:25-37`](../deepseek-harness/.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.md#L25-L37)

## 6. Structured output：让 child 结果成为可组合数据

Workflow 的 `agent({schema})` 依赖 subagent `outputSchema` capability。In-process child 获得 scoped capture tool、instruction 和 monotonic guard；只有 schema-valid committed capture 才算成功，capture 后不再允许 side effect，并在 commit 后停止 child 而不额外请求模型。[`.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.md:45-51`](../deepseek-harness/.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.md#L45-L51)

这避免 workflow 从自然语言猜测结构化中间值，同时保持失败为可重试的 tool error。

## 7. Goal：状态与继续执行权限分离

长任务目标若只是内存 loop variable，会在 restart 丢失；若 UI 独有，模型行为无法重建。但持久化“自动继续”权限会导致打开 Session 就自动花费资源。Goal 决策因此将 durable lifecycle 与 process-local activation 分开。[`.agents/notes/implemented/feature/2026-07-19-persisted-same-session-goal-domain.md:7-17`](../deepseek-harness/.agents/notes/implemented/feature/2026-07-19-persisted-same-session-goal-domain.md#L7-L17)

每次 goal mutation append 完整 `goal/change` snapshot 或 tombstone；phase 为 `active | paused | blocked | complete`，更新使用 id+revision compare-and-set。Resume/fork 后 durable state 存在，但 activation 默认 disarmed，只有显式 resume 才重新授权继续。[`.agents/notes/implemented/feature/2026-07-19-persisted-same-session-goal-domain.md:19-35`](../deepseek-harness/.agents/notes/implemented/feature/2026-07-19-persisted-same-session-goal-domain.md#L19-L35)

`dsh-goal` 只存状态；`goal-round-driver`、command、tool 是独立 consumers。因此 goal 不污染 concrete loop。[`packages/goal/goal/README.md:26-32`](../deepseek-harness/packages/goal/goal/README.md#L26-L32)

## 8. 各编排原语的选择

| 需求 | 选择 |
|---|---|
| 一条新用户消息 | `followup()` |
| 当前 turn 后追加纠偏 | `steer()` |
| 只注入下一请求上下文 | `inject()` |
| 一至两个独立子任务 | `subagent` tool |
| 多项 fan-out/fan-in、pipeline | `workflow` tool |
| 跨 turn、restart 的单一目标 | `goal` + round driver |
| 已发布后台进程/子任务 | `ctx.jobs` + job tools |

Workflow README 也明确建议一两个 delegation 使用普通 subagent，复杂分解才用 workflow。[`packages/workflow/workflow/README.md:26-36`](../deepseek-harness/packages/workflow/workflow/README.md#L26-L36)

## 9. 限制

- Foreground subagent/workflow 会占住父 step，直到完整结果 settle。
- Subagent recursion 需要 depthLimit/toolFilter；某些 remote providers 不支持这些 capability。[`.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md:71-76`](../deepseek-harness/.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md#L71-L76)
- Worker workflow 无 saved/nested workflow、journaling/resume、token budget 或 hostile-code isolation。[`.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.md:57-65`](../deepseek-harness/.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.md#L57-L65)
- Goal 只有一个 current goal，且 round cap 不等于 token/currency/time budget。[`packages/goal/goal/README.md:151-162`](../deepseek-harness/packages/goal/goal/README.md#L151-L162)
- 目标是否真正 complete 由记录该状态的 actor 决定，独立 evaluator/certificate 尚未实现。[`.agents/notes/implemented/feature/2026-07-19-persisted-same-session-goal-domain.md:57-63`](../deepseek-harness/.agents/notes/implemented/feature/2026-07-19-persisted-same-session-goal-domain.md#L57-L63)

## 用于制作 PPT 的提示词

```text
生成中文技术 PPT《DeepSeek Harness 的 Agent 编排原语》，12 页，16:9，面向 Agent 系统研究者。用层级图说明 AgentLoop、Subagent、Workflow、Goal、Jobs 的职责，不要把它们描述成一个统一工作流引擎。

页面：
1. 标题与编排层级总图。
2. turn/step/inbox 基础状态机。
3. followup、steer、inject、cancel 的差异。
4. Subagent named-provider seam 与多 transport coexistence。
5. Subagent start→publication→result→dispose ownership 时序。
6. fresh 与 fork、parent/child Session lineage。
7. capability negotiation 和 fail-loud。
8. Workflow 解决多任务 fan-out 的问题。
9. worker-thread engine、script API、信任边界。
10. structured-output capture tool 与 monotonic commit。
11. Goal 的 durable phase 与 process-local activation。
12. 选择矩阵、限制与适用场景。

页脚引用相对路径:行号。重点证据：docs/architecture.md:80-103、packages/core/agent-loop/src/agent.ts:128-147、.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md:26-65、.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.md:7-51、.agents/notes/implemented/feature/2026-07-19-persisted-same-session-goal-domain.md:7-35。每页附 80～120 字演讲备注。
```
