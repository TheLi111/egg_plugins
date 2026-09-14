# 问题一：DeepSeek Harness 如何让“一切皆可替换”

## 结论

dsh 的可替换性来自三层共同作用，而不只是“把代码拆成插件”：

1. Cordis 将运行时能力暴露为 `ctx.<service>`，consumer 依赖 service key，不依赖具体实现。
2. dsh 将可替换能力设计成 **Service Definition / Service Provider / Consumer** 三种角色；替换 provider 不要求修改模型看到的 Consumer API。
3. profile、bundle 与 patch 在启动时组装整棵 plugin tree；`agent-loop`、Session、工具、LLM adapter 也都是这棵树中的普通 plugin，没有绕过 Loader 的特权入口。

架构文档直接声明 model adapter、tool registry、session log 和 agent loop 都是 plugin，并且 registration 会在 plugin unload 时撤销。[`docs/architecture.md:9-13`](../deepseek-harness/docs/architecture.md#L9-L13) Microkernel 设计笔记进一步要求 hooks、goal、workflow、compaction、sandbox、permission、UI、persistence、MCP 和 skill 均能在不修改 core loop 的情况下实现。[`.agents/notes/implemented/architecture/2026-06-11-microkernel-event-taxonomy.md:7-20`](../deepseek-harness/.agents/notes/implemented/architecture/2026-06-11-microkernel-event-taxonomy.md#L7-L20)

## 1. “可替换”的准确含义

### 1.1 运行时替换：按 service key 查找能力

Cordis `Context` 是一个 Proxy；普通属性读取经过 service resolver。`extend()`、`isolate()` 会生成 child context，使不同 scope 可以解析到同一 service key 的不同实现。[`vendor/cordis/src/context.ts:35-41`](../deepseek-harness/vendor/cordis/src/context.ts#L35-L41) [`vendor/cordis/src/context.ts:90-125`](../deepseek-harness/vendor/cordis/src/context.ts#L90-L125)

一个 `Service` 子类在构造时调用 `ctx.reflect.provide(name, instance, check)`，其注册归 owning fiber 所有；fiber 卸载后实现自动从 Context 消失。[`vendor/cordis/src/service.ts:32-58`](../deepseek-harness/vendor/cordis/src/service.ts#L32-L58) Consumer 用 `inject` 声明所需 service；Cordis 只有在实现存在时才激活 fiber，并在依赖实现变化时 unload/reload。[`vendor/cordis/src/registry.ts:164-185`](../deepseek-harness/vendor/cordis/src/registry.ts#L164-L185) [`vendor/cordis/src/fiber.ts:611-673`](../deepseek-harness/vendor/cordis/src/fiber.ts#L611-L673)

这意味着“替换”不是全局变量覆写，而是改变 composition 中提供某个 `ctx` key 的 plugin；依赖这个 key 的 consumers 随 Cordis 生命周期重新收敛。

### 1.2 包级替换：Definition / Provider / Consumer 三角色

Capability-seam 设计笔记指出，能力的接口、实现和模型/插件消费 API 变化速率不同；把三者绑在一个包中，会让替换本地执行器牵连模型工具 schema。[`.agents/notes/implemented/architecture/2026-06-13-capability-seams.md:7-11`](../deepseek-harness/.agents/notes/implemented/architecture/2026-06-13-capability-seams.md#L7-L11)

因此 dsh 使用三角色：

| 角色 | 职责 | shell 例子 |
|---|---|---|
| Service Definition | 定义 `ctx.<key>`、方法和共享类型 | `packages/shell/shell` 定义 `ctx.shell` |
| Service Provider | 提供实际执行实现 | `bash-local`、`bash-sandbox`、`pwsh-local` |
| Consumer | 暴露模型或其他插件使用的稳定 API | `tool-bash`、`tool-pwsh` |

角色定义、拆包条件及“不应预先过度拆分”的例外见 capability-seam 笔记。[`.agents/notes/implemented/architecture/2026-06-13-capability-seams.md:13-29`](../deepseek-harness/.agents/notes/implemented/architecture/2026-06-13-capability-seams.md#L13-L29) 其直接收益是 sandboxed executor 可替换 local executor，而 tool schema 无需改变；代价是更多 package、manifest、README 与 injection wiring。[`.agents/notes/implemented/architecture/2026-06-13-capability-seams.md:31-38`](../deepseek-harness/.agents/notes/implemented/architecture/2026-06-13-capability-seams.md#L31-L38)

### 1.3 装配级替换：profile → bundle → patch

`web` profile 明确由 `dsh-base` 与 `dsh-web-app` 两个 bundle 构成，并启用 live patch reload。[`packages/boot/app-boot/src/profile.ts:104-126`](../deepseek-harness/packages/boot/app-boot/src/profile.ts#L104-L126) 启动器将层按 bundle → profile patch → home patch → `--patch` overlay → telemetry overlay 的顺序组合。[`apps/cli/src/profile-boot.ts:194-243`](../deepseek-harness/apps/cli/src/profile-boot.ts#L194-L243)

最终根配置是空 entry list；所有运行条目由 patch 插入。[`apps/cli/src/profile-boot.ts:83-91`](../deepseek-harness/apps/cli/src/profile-boot.ts#L83-L91) `mountRootInclude()` 再把合成结果交给 Cordis Loader。[`packages/boot/app-boot/src/index.ts:516-553`](../deepseek-harness/packages/boot/app-boot/src/index.ts#L516-L553)

因此部署者可以在配置层替换：

- `ctx.llm` 的 adapter owner；默认 base 同时装载 LLM registry 与 DeepSeek/Pi-AI adapter rows。[`packages/bundle/base/cordis.patch.yml:27-31`](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L27-L31) [`packages/bundle/base/cordis.patch.yml:482-487`](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L482-L487)
- `ctx.sessionPersistence` 的后端；默认是 JSONL。[`packages/bundle/base/cordis.patch.yml:110-113`](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L110-L113)
- `ctx.subprocess`、`ctx.sandbox` 与 `ctx.fs` 的实现。[`packages/bundle/base/cordis.patch.yml:199-230`](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L199-L230) [`packages/bundle/base/cordis.patch.yml:477-480`](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L477-L480)
- `ctx.agentLoop` 本身；base 只是选择当前唯一的 concrete loop plugin，并不使它成为 framework 内核。[`packages/bundle/base/cordis.patch.yml:470-475`](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L470-L475)

## 2. 为什么 agent-loop 仍然可替换

`dsh-agent` 包定义公共 `Agent` 与 `agent/*` event vocabulary；`dsh-agent-loop` 是实现该接口的 concrete driver。Microkernel 笔记明确禁止其他包依赖 concrete loop，应只依赖 contract package。[`.agents/notes/implemented/architecture/2026-06-11-microkernel-event-taxonomy.md:13-20`](../deepseek-harness/.agents/notes/implemented/architecture/2026-06-11-microkernel-event-taxonomy.md#L13-L20)

`AgentRegistry` 的 factory 由 loop 注册，外部 consumer 通过 `ctx.agents.create()/resume()` 创建 Agent，因此将来可以由另一个 loop implementation 替换 factory，而不改变 Web、ACP 或 subagent consumer。该关系由 subsystem 文档说明。[`docs/subsystems/core.md:22-53`](../deepseek-harness/docs/subsystems/core.md#L22-L53)

## 3. 可替换性不是无限自由

### 3.1 一个 provider key 通常只有一个 owner

LLM adapter 以 provider name 为 route ownership key；同一 Context 中一个 provider 只能有一个 adapter owner。加载两个都声称 `deepseek` 的 adapter 是配置错误，不使用隐式优先级或 fallback。[`.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md:19-27`](../deepseek-harness/.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md#L19-L27)

### 3.2 并非每个包都需要三拆分

设计笔记明确拒绝预先拆分只有一个实现和一个 consumer 的能力；LLM seam 也允许 Definition 与 consumer 共处于 `dsh-llm`，只把 adapters 分离。[`.agents/notes/implemented/architecture/2026-06-13-capability-seams.md:23-25`](../deepseek-harness/.agents/notes/implemented/architecture/2026-06-13-capability-seams.md#L23-L25)

### 3.3 插件处于同一个可信进程

可替换不等于安全隔离。普通 Cordis plugin 能获得其 Context 中公开的能力；安全性来自 sandbox/approval/provider boundaries，而不是“plugin”这一称谓本身。特别是 workflow worker 的设计笔记明确指出 worker thread 与 vm context 不是 hostile-code security boundary。[`.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.md:25-31`](../deepseek-harness/.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.md#L25-L31)

## 4. 对其他 Agent Harness 的启示

可迁移的核心不是选择 Cordis，而是同时保留四条约束：

1. Consumer 依赖稳定 service identity，不依赖具体 provider class。
2. 模型 API 与 provider implementation 分开演进。
3. 所有 registration 都具有明确 owner 和 disposer。
4. application composition 是数据化、可检查的树，而不是分散在入口文件中的条件分支。

若系统只有一个模型、一个存储后端且不允许第三方扩展，完整三角色拆包的成本可能超过收益；此时仍可保留 service interface 和 disposer lifecycle，延迟物理拆包。

## 文档与代码核对

架构文档把 patch 顺序写作 bundle → profile → home → `--patch`。[`docs/architecture.md:15-29`](../deepseek-harness/docs/architecture.md#L15-L29) 源码还会在其后加入 telemetry environment patch。[`apps/cli/src/profile-boot.ts:231-243`](../deepseek-harness/apps/cli/src/profile-boot.ts#L231-L243) 这是文档省略的启动器派生层；本报告以源码顺序为准。

## 用于制作 PPT 的提示词

```text
请制作一份中文技术研究 PPT，题目为《DeepSeek Harness：如何让一切皆可替换》。面向熟悉 TypeScript 和 Agent 框架的研究者，共 10 页，16:9，深色简洁工程风格。不要使用营销措辞，不要声称插件天然安全。

页面结构：
1. 标题页：核心结论——可替换性来自 Context service、capability seam、profile composition 三层。
2. 问题页：传统 Agent 内核中 model、session、tools、loop、storage 的耦合。
3. Cordis Context 图：ctx.<key>、Service、inject、Fiber 的关系。
4. 生命周期图：provider 出现→consumer fiber 激活；provider 消失→effect/disposer 回收。
5. Capability seam 三角色图：Definition / Provider / Consumer，以 shell 为例。
6. Profile 装配图：web profile → base/web-app bundles → profile/home/CLI patches → Loader。
7. agent-loop 为什么不是特权内核：dsh-agent contract + replaceable factory。
8. LLM、filesystem、persistence 三个替换实例。
9. 代价与边界：包数量、配置错误、single provider owner、共享可信进程。
10. 可迁移设计原则总结。

每页页脚必须放 1～2 个源码证据，格式为“相对路径:行号”；优先使用：docs/architecture.md:9-29、packages/boot/app-boot/src/profile.ts:104-126、apps/cli/src/profile-boot.ts:194-243、vendor/cordis/src/service.ts:32-58、.agents/notes/implemented/architecture/2026-06-13-capability-seams.md:7-38。图中使用准确 API 名称，不使用“高度模块化”之类空话。为每页附 80～120 字演讲备注。
```
