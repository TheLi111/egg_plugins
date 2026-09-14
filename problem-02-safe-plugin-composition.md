# 问题二：DeepSeek Harness 如何让插件安全组合

## 结论

dsh 所谓“安全组合”包含三种不同安全性，不能混为一谈：

- **生命周期一致性**：service、listener、tool、prompt registration 都有 owner，plugin unload 时撤销。
- **交互语义可预测**：依赖由 `inject` 驱动；事件具有明确 dispatch mode；waterfall 的 `next()` 决定继续还是 short-circuit。
- **运行时关系可检查**：Session、agent、LLM stream、tool pipeline 等包可发布 invariant companion，检查跨事件关系而不是仅靠 TypeScript 类型。

它不等价于恶意插件隔离。普通插件仍在同一进程和信任域中；sandbox 保护的是受控 capability path，不是 Cordis 插件之间的权限隔离。

## 1. 依赖关系由运行时服务状态驱动

Plugin 可以是 function、class 或 `{ apply }` object，并可声明 `inject`、`provide`、`Config`。[`vendor/cordis/src/registry.ts:91-145`](../deepseek-harness/vendor/cordis/src/registry.ts#L91-L145) `ctx.plugin()` 创建 Fiber；`ctx.inject()` 是带依赖声明的 plugin shortcut。[`vendor/cordis/src/registry.ts:164-185`](../deepseek-harness/vendor/cordis/src/registry.ts#L164-L185) [`vendor/cordis/src/registry.ts:293-335`](../deepseek-harness/vendor/cordis/src/registry.ts#L293-L335)

Fiber 用 inject map 建立当前 dependency epoch。缺服务时为 `PENDING`；服务齐备时 `LOADING → ACTIVE`；依赖变更时 `UNLOADING` 后根据新 epoch 再加载。[`vendor/cordis/src/fiber.ts:139-154`](../deepseek-harness/vendor/cordis/src/fiber.ts#L139-L154) [`vendor/cordis/src/fiber.ts:611-695`](../deepseek-harness/vendor/cordis/src/fiber.ts#L611-L695)

这消除了手写“先启动 A，再启动 B”的大部分顺序控制。Base patch 也明确说明 row order 不携带 load semantics，activation 由 service availability 驱动。[`packages/bundle/base/cordis.patch.yml:1-13`](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L1-L13)

## 2. 可逆副作用是组合一致性的基础

`ctx.effect()` 接受同步/异步 disposer 或 disposer iterable，owner unload 时执行清理。[`vendor/cordis/src/fiber.ts:68-101`](../deepseek-harness/vendor/cordis/src/fiber.ts#L68-L101) `ctx.on()` 将 listener registration 作为当前 fiber 的 effect，因此无需插件额外维护全局 listener table。[`vendor/cordis/src/events.ts:245-301`](../deepseek-harness/vendor/cordis/src/events.ts#L245-L301)

Service construction 也通过 `reflect.provide()` 绑定到 owning fiber。[`vendor/cordis/src/service.ts:32-58`](../deepseek-harness/vendor/cordis/src/service.ts#L32-L58) 所以一次正常卸载会一起撤销：

- `ctx.<service>` implementation；
- typed event listener；
- tool schema/body registration；
- prompt section/context/provider；
- provider-registry entry；
- watcher、server、process 等显式 effect resource。

Microkernel 笔记选择 Cordis event system 而没有自建 Koa middleware stack，原因之一就是复用 dispatch、disposal 与 reload semantics。[`.agents/notes/implemented/architecture/2026-06-11-microkernel-event-taxonomy.md:22-30`](../deepseek-harness/.agents/notes/implemented/architecture/2026-06-11-microkernel-event-taxonomy.md#L22-L30)

## 3. Typed event mode 约束插件如何合作

| Mode | 语义 | dsh 例子 |
|---|---|---|
| `waterfall` | around middleware；listener 可 transform、recover、wrap 或不调用 `next()` 来 veto | `agent/pre-step`、`agent/request`、`tools/*`、`llm/stream`、`system-prompt/assemble` |
| `serial` | 按注册顺序 await，首个 bail value 停止 | `agent/turn-stopping` |
| `parallel` | 所有 listener 都获得机会并被 await | `session/flush` |
| `emit` | 同步通知，不等待返回 | inbox/lifecycle/error、`tools/result` observation |

该 taxonomy 由 microkernel 决策固定。[`.agents/notes/implemented/architecture/2026-06-11-microkernel-event-taxonomy.md:11-20`](../deepseek-harness/.agents/notes/implemented/architecture/2026-06-11-microkernel-event-taxonomy.md#L11-L20) Cordis 实际实现五类 dispatch，并明确 `waterfall` 的最后参数为 `next`。[`vendor/cordis/src/events.ts:24-32`](../deepseek-harness/vendor/cordis/src/events.ts#L24-L32) [`vendor/cordis/src/events.ts:76-88`](../deepseek-harness/vendor/cordis/src/events.ts#L76-L88)

Waterfall 的危险点也很明确：listener 忘记 `next()` 会 short-circuit。设计笔记把这一点列为必须文档化并进行 composition test 的非显然语义。[`.agents/notes/implemented/architecture/2026-06-11-microkernel-event-taxonomy.md:26-31`](../deepseek-harness/.agents/notes/implemented/architecture/2026-06-11-microkernel-event-taxonomy.md#L26-L31)

## 4. 多插件策略冲突如何处理

dsh 不假设所有策略都是同一种 middleware：

- 可重排的 allow/deny/ask 放在 `tools/pre-execute` waterfall。
- 绝不能被后续 listener 重新放行的规则使用 `ctx.tools.guard()` monotonic guard。
- timeout、retry、metrics 使用 `tools/execute` around wrapper。
- result rewrite 或 additional context 使用 `tools/post-execute`。
- 不修改结果的审计使用 immutable `tools/result` notification。

这些插入点与选择原则在工具 subsystem 和 cookbook 中定义。[`docs/subsystems/tools.md:170-208`](../deepseek-harness/docs/subsystems/tools.md#L170-L208) [`docs/cookbook/adding-a-tool.md:57-60`](../deepseek-harness/docs/cookbook/adding-a-tool.md#L57-L60)

Registry 的代码顺序是 pre waterfall → approval ask → monotonic guards → dispatch。[`packages/core/tools/src/index.ts:1453-1496`](../deepseek-harness/packages/core/tools/src/index.ts#L1453-L1496) 因此 security owner 可选择不可逆 guard，不依赖 listener registration order。

## 5. Scope 防止 per-agent registration 泄漏

`Context.isolate(name)` 为一个 service key 建立独立 label；同 label 可 join，其他 scope 不可见。[`vendor/cordis/src/context.ts:109-125`](../deepseek-harness/vendor/cordis/src/context.ts#L109-L125) Web 的 agent-presets 使用 standing mount + agent scope，使一个 preset 的 tools/prompt listeners 只覆盖加入该 mount 的 agents。[`packages/preset/agent-presets/README.md:98-130`](../deepseek-harness/packages/preset/agent-presets/README.md#L98-L130)

Preset mount 还执行审计：拒绝 unscoped target、永远等待缺失服务的 row、以及把 service 泄漏到 root realm 的 row。[`packages/preset/agent-presets/README.md:120-138`](../deepseek-harness/packages/preset/agent-presets/README.md#L120-L138)

## 6. Runtime invariant 检查跨时间关系

TypeScript `readonly` 不能阻止 runtime cast，也无法表达 turn/step nesting、tool-call/result pairing、stream block grammar等关系。设计笔记因此将职责分为：Session boundary 永久在线 snapshot+deep-freeze；可选 invariant companion 检查跨记录/跨服务关系。[`.agents/notes/implemented/architecture/2026-06-11-dev-invariants-over-deep-readonly.md:7-27`](../deepseek-harness/.agents/notes/implemented/architecture/2026-06-11-dev-invariants-over-deep-readonly.md#L7-L27) [` .agents/notes/implemented/architecture/2026-06-11-dev-invariants-over-deep-readonly.md:29-37`](../deepseek-harness/.agents/notes/implemented/architecture/2026-06-11-dev-invariants-over-deep-readonly.md#L29-L37)

Invariant 必须检查独立可观察关系；不能只是检查 plugin name、method existence 或固定样例。中央 `dsh-invariants` 只负责 registry、selection、child fiber 与 package-attributed failure，不导入产品包。[`.agents/notes/implemented/architecture/2026-07-19-package-invariant-runtime-contracts.md:7-26`](../deepseek-harness/.agents/notes/implemented/architecture/2026-07-19-package-invariant-runtime-contracts.md#L7-L26)

代表性检查包括：

- Session seq、turn/step enclosure、tool pair；
- Agent status transition；
- scope subject consistency；
- loop request 与 log reconstruction 相等；
- LLM stream grammar；
- tool pipeline stage monotonicity 与 immutable result。

这些已实现关系见设计笔记清单。[`.agents/notes/implemented/architecture/2026-07-19-package-invariant-runtime-contracts.md:28-56`](../deepseek-harness/.agents/notes/implemented/architecture/2026-07-19-package-invariant-runtime-contracts.md#L28-L56)

## 7. 安全组合的边界和成本

- **不是恶意插件沙箱。** Cordis 解决生命周期与依赖组合，不限制可信 plugin 直接访问同进程资源。
- **Waterfall 易误用。** 不调用 `next()` 可能是 policy veto，也可能是 bug；类型无法区分意图。
- **HMR 需要持有正确 disposer。** 未放入 effect 的外部资源不会自动回收。
- **Invariant 是 opt-in composition。** `dsh-base` 默认不加载 runtime diagnostics；只加载 registry 而无 companions 也不会检查产品关系。[`packages/runtime-diagnostics/invariants/README.md:26-36`](../deepseek-harness/packages/runtime-diagnostics/invariants/README.md#L26-L36)
- **Scope 泄漏属于配置/实现错误。** preset mount audit 能抓一部分 service leak，但 shared process 的任意副作用仍需 plugin 自律与测试。

## 用于制作 PPT 的提示词

```text
请制作一份中文技术 PPT，标题《DeepSeek Harness：几十个插件如何安全组合》。受众为 Agent 平台架构师，共 11 页，16:9，使用清晰的状态图和时序图，不使用泛泛的“模块化良好”表述。

页面安排：
1. 标题与结论：生命周期一致性、交互语义、运行时关系三层安全。
2. 风险模型：依赖缺失、卸载残留、listener 冲突、scope 泄漏、跨事件不一致。
3. Plugin/Fiber 状态机：PENDING→LOADING→ACTIVE→UNLOADING/FAILED/DISPOSED。
4. inject 驱动的依赖激活图。
5. ctx.effect 与 disposer 的资源所有权图。
6. 五种 event dispatch mode 对照表，突出 waterfall 的 next()。
7. 工具策略分层：pre、approval、guard、execute、post、result。
8. Agent scope 与 preset isolate 图。
9. Session immutable boundary 与 runtime invariant companion 的分工。
10. 已知边界：不是恶意插件隔离、waterfall 易误用、diagnostics 为可选。
11. 对其他 harness 的落地检查表。

每页必须附“相对路径:行号”证据。重点证据：vendor/cordis/src/fiber.ts:139-154、vendor/cordis/src/fiber.ts:611-695、vendor/cordis/src/events.ts:24-88、.agents/notes/implemented/architecture/2026-06-11-microkernel-event-taxonomy.md:7-31、.agents/notes/implemented/architecture/2026-07-19-package-invariant-runtime-contracts.md:15-56。为每页生成 80～120 字演讲备注，并在最后一页区分“组合安全”与“恶意代码安全隔离”。
```
