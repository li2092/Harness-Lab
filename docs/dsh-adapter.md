# dsh 适配设计（草案，2026-09）

> 上位文档：[产品定义](product-definition.md) §四–§六；[论文方案](paper-proposal.md) §四。
> 调研基线：`deepseek-ai/deepseek-harness` @ `477b4f4`（2026-09-24，版本 `0.1.7-rc.2`，MIT）。dsh 处于 developer preview，官方明确会有不兼容变更，下文所有接口引用都以这个提交为准。

---

## 一、结论先行

| 需求 | dsh 现状 | 适配难度 |
|---|---|---|
| 机制可开关 | 原生支持。每个机制是一个 Cordis 插件，用 `--patch` 的 `disabled` 或 `config` 按次切换 | 低 |
| 影子模式 | 可以做。waterfall 事件里只观察、调用 `next()`，把"本来会介入"写成 `ignorable` 的自定义会话事件 | 低 |
| 会话分叉 | 日志层原生支持：`buildForkSeed` 和 `ctx.agents.create({seed, …})` 可以在任意事件序号处分叉，并换用另一套配置 | 中 |
| 分叉时的环境状态 | **不支持**。分叉的会话与父会话共用同一个工作区，没有文件系统或工具状态快照 | **高：需要自建** |
| trace 导出为 OTel | **不支持** span。只有会话事件流，以及仅在用户反馈后才导出的 OTLP logs | 中：写一个导出插件 |
| 公开基准 | **没有** τ² / τ³ / Terminal-Bench 集成，`benchmarks/` 下只有性能基准 | **高：需要自建** |
| 随机性控制 | `temperature` 只能经 `agent/request` 设置；**没有 seed** | 靠对照分叉兜底 |

**判断：dsh 适合做首个适配目标。** 插件化和日志分叉这两项是影子机制方法最需要的能力，dsh 已经具备，而且做得比多数框架干净。真正的工作量集中在三处：**环境快照**、**基准接入**、**OTel 导出**。这三项本来就应该属于 Harness Lab 自己，不属于 dsh。

## 二、机制 → dsh 插件

每个机制是一个独立的 Cordis 函数插件（具名导出 `name` / `inject` / `Config` / `apply`），统一接收下列配置：

```ts
Config = z.object({
  mode: z.union(['on', 'off', 'shadow']).default('on'),
  // 机制自身的参数……
})
```

各模式的行为：

- `off`：插件不加载，由 patch 里的 `disabled: true` 实现。
- `on`：正常介入。
- `shadow`：注册与 `on` 完全相同的监听器，但判定为"要介入"时只写一条 `hlab/activation` 事件，然后原样调用 `next()`。

**dsh 的约束：** 仓库有一条运行时不变式，"模型可见 ⟺ 已记入日志"。所以影子模式**绝不能**注入任何模型可见的内容。`hlab/activation` 通过声明合并加入 `SessionEventMap`，并标记 `ignorable: true`，这样不会进入 `deriveMessages()`。

### 挂载点对照

| 机制（概念稿） | stage | kind | dsh 挂载点 | 能否影子 | 备注 |
|---|---|---|---|---|---|
| Schema Fix Adapter | tool_call | reactive | 见下方"参数改写" | 能 | **dsh 禁止在 `tools/pre-execute` 改写参数**，需要换一种实现 |
| Loop Breaker | tool_call | reactive | `tools/pre-execute` → `deny`，或 `tools/post-execute` → `additionalContexts` | 能 | 可以参考 dsh 自带的 `guard/repeat-tool-reminder` |
| Explicit Shell Args | tool_call | reactive | `ctx.tools.guard()`（单调，只能拒绝） | 能 | |
| Tenant Isolation | tool_call | reactive | `ctx.tools.guard()` | 能 | 脱敏：租户字段名要参数化 |
| Null ≠ {} Semantic | tool_call | reactive | `tools/post-execute` → 替换 `content` | 能 | |
| Lazy-tool State | tool_call | reactive | `tools/post-execute` | 能 | |
| Pre-flight Policy | policy | persistent | `agent/pre-step`（`enter` 时改写传入的消息） | 否（常驻） | 用配对 A/B |
| Dynamic Tool Scope | policy | persistent | 挂在 `agent.ctx` 上的工具注册表，按 agent 生效 | 否 | |
| Sub-agent Depth Limit | policy | reactive | `tools/pre-execute`，针对 subagent 工具 | 能 | |
| Context Slimming | agent | reactive（按阈值触发） | 包装 `ctx.compaction`，或在 `agent/pre-step` 上 `prepend` | 能 | dsh 没有"压缩前否决"的事件 |
| Structured Checklist | agent | persistent | `agent/pre-step` | 否 | |
| Sysprompt Rebuild | agent | persistent | `agent-preset` 或 `developer/message` | 否 | |
| Param Auto-Complete / Lifecycle Hook / Premature FE State（负向） | agent | 待定 | 逐个确认 | —— | 负向机制是衰减研究的重点对象 |
| 声明门（新增） | claim | reactive | `agent/turn-stopping` → 用 `agent.steer()` 要求补证据 | 能 | dsh 没有专门的"完成声明"事件；判定依据是 `assistant/message` 加上 `turn/end` |
| 推理强度 / 模型路由 | profile | persistent | `agent/request` → `LlmCallConfig` | 否 | |

**参数改写（Schema Fix 这一类）的三种实现方式**：

1. **包装工具**：在工具注册表里用一个"宽容版"替换原工具，在工具内部规范化参数。模型看到的是同一个工具，日志里记的是原始参数，符合 dsh 的不变式。**推荐这一种。**
2. **拒绝并提示**：`deny` 并附上修正提示，让模型自己重发。语义变了，从"静默修复"变成"反馈修复"，要当作另一个机制登记。
3. **包装 `llm/stream`**：在流上改写工具调用。能做到，但动的是模型输出，会违背"日志即真值"，**不采用**。

这一点要在论文里如实写出来：同一个"机制"在不同框架里的实现语义可能不同。这正是 M4"跨实现对照"要回答的问题。

## 三、运行器（Runner）

dsh 的分叉能力**没有**暴露在 Python SDK、JSON-RPC 或 ACP 协议里（ACP 的 README 明确写着不支持 fork）。所以 Harness Lab 的 dsh 运行器要做成一个**进程内的 TypeScript 驱动**：启动 dsh 应用，直接调用 `ctx.agents.create` 与 `ctx.sessions`。

```
packages/hlab-dsh/            (放在 Harness Lab 仓库，依赖锁定到具体版本的 @deepseek-ai/dsh-*)
  mechanisms/                 各机制插件（脱敏后开源）
  activation/                 hlab/activation 事件声明 + 影子模式的公共工具函数
  runner/                     启动 dsh（profile=headless 或 sdk-minimal）、跑 trial、分叉
  env/                        环境快照与恢复（见 §四）
  bench/tau/                  τ² / τ³ 的工具插件 + 用户模拟器 + 判分
  bench/terminal/             Terminal-Bench 子集（每个分支一个容器）
  export/otel/                会话事件 → OTel GenAI span
```

**一次 trial 的流程：**
1. 生成配置 patch：各机制的 `mode`、模型、`reasoningEffort`、`temperature`。
2. 用**独立的** `DSH_HOME` 和工作区启动。
3. 由 bench 驱动多轮对话。τ 系列里，用户模拟器发出 `session/prompt`。
4. 读取 JSONL 会话日志，判分，产出 Trial 与 Activation 记录。

**一次分叉的流程：**
1. 在影子 trial 里找到第一个 `hlab/activation`，拿到它的 `seq`。
2. 用 `buildForkSeed(events, seq)` 生成三份种子：ON、OFF、CTRL。
3. 把环境恢复到该 `seq` 时的快照（§四），每个分支一份。
4. 每个分支调用一次 `ctx.agents.create({ seed, inheritedEventCount, agentOptions, setup })`。`setup` 负责按分支装载或卸载机制插件。
5. 三个分支各自跑完，然后判分。

**必须的默认设置：**
- `session-log-deepseek` 默认会把会话日志**上传到 DeepSeek 官方 API**。Harness Lab 的基础 patch 必须设置 `enabled: false`。这一条也属于脱敏要求。
- 其他遥测插件保持关闭，只用本地 JSONL，压缩设为 `none`，方便读取。

## 四、环境快照（自建，最关键）

dsh 分叉时不复制工作区，因此"同一前缀、不同配置"的前提要由 Harness Lab 保证。

| 基准 | 环境状态 | 快照方式 |
|---|---|---|
| τ² / τ³ | 业务数据库（订单、航班……）+ 用户模拟器的对话状态 | 工具插件把状态保存在可序列化的内存 DB 里，每次工具调用后写入以 `seq` 为键的快照。用户模拟器的历史可以从会话日志重建，但模拟器下一步的输出是随机的，要记录下来或用对照分叉兜底 |
| Terminal-Bench | 容器文件系统 + 进程 | 在分叉点做 `docker commit` 或 overlay 快照，每个分支起一个新容器。进程状态不保存，只在工具调用边界分叉（dsh 的 checkpoint 策略本来就在工具调用前后刷盘） |

**原则：只在工具调用边界或步骤边界分叉。** 这些位置正好是 dsh 会话 checkpoint 的刷盘点，也是反应式机制的天然触发点。

## 五、trace 导出

dsh 没有 OTel span。在 `export/otel/` 中实现一个 `session/event` 监听器，或一个 `SessionTelemetryBackend`，做如下映射：

| dsh 会话事件 | OTel |
|---|---|
| `turn/start` ~ `turn/end` | agent 调用 span |
| `step/start` ~ `step/end` | step span |
| `request/*` + `assistant/message` | `gen_ai` chat span（模型、token、推理强度） |
| `tool/call` ~ `tool/result` | `execute_tool` span |
| `hlab/activation` | span event，属性包括 `hlab.mechanism.id`、`hlab.mechanism.mode`、`hlab.would_intervene` |
| trial 或分叉的元数据 | resource 属性：`hlab.config.id`、`hlab.trial.id`、`hlab.fork.parent_seq`、`hlab.fork.arm` |

JSONL 原始日志始终是真值源，OTel 只是导出视图。这与 README 第八节的原则一致。

## 六、风险

| 风险 | 应对 |
|---|---|
| dsh 不兼容升级 | 锁定提交；adapter 层只依赖 §二、§三列出的事件和服务；在 CI 里对锁定版本跑一次冒烟 trial |
| `agent-preset` 在会话开始后被锁定，分叉后无法换 preset | 不用 preset 切换机制，改在 `agents.create` 的 `setup` 里按分支装载插件 |
| 没有 seed，同配置分叉也会发散 | CTRL 分叉是方法的必需部分，用它测噪声地板；记录服务精度（参见 Replay Gap 的 FP8 发现） |
| 首次运行时 dsh 默认上传会话日志 | 基础 patch 强制关闭，并在 runner 启动时检查它是否已关闭，未关闭就拒绝运行 |
| 自建基准接入与官方判分不一致 | τ 系列直接复用官方的任务与判分代码，只替换执行器 |

## 七、与 P0–P3 的衔接

- **P0（旗舰实验复现）不依赖 dsh**：继续用原来的 τ³-airline 调试环境做严格协议的复现，拿到可信基线。
- **P1**：`hlab-dsh` 骨架，包括 activation 事件、三种模式、基础 patch（关闭上传），先迁移 3–4 个代表性机制：Schema Fix（包装工具方式）、Loop Breaker、Pre-flight Policy、声明门。然后实现 OTel 导出。
- **P2**：τ² / τ³ bench 插件、环境快照、分叉驱动。验收标准：同一任务上，ON / OFF / CTRL 三个分支可以从同一 `seq` 跑完，并能导出 trace。
- **P3**：Terminal-Bench 子集与容器快照；把剩余机制迁移完。
