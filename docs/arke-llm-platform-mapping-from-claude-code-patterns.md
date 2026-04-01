# Arke 大模型功能平台与 Claude Code 模式的对照（可参考 / 可抽象复用的设计）

**Mapping Arke’s LLM platform to patterns observed in the Claude Code source snapshot — design-level reuse only, not code copying.**

> 本文档将此前对话中的分析整理成文，与 [`claude-code-architecture-analysis.md`](./claude-code-architecture-analysis.md)、[`build-your-own-agent-from-claude-code-patterns.md`](./build-your-own-agent-from-claude-code-patterns.md) 互为补充。  
> **路径约定：** 文中 `/home/blueyi/workspace/repos/arke` 指 Arke 仓库根目录；若你只在 Arke 侧维护文档，可将本文复制到 `arke/docs/design/` 并调整交叉链接。

---

## 1. Arke 与 Claude Code 已同构的部分

两者均为 **「LLM 调用 → 解析 tool_use → 在环境中执行 → 写回 tool_result → 再调用」** 的 agentic loop。

- **Claude Code：** `query.ts` 中 `queryLoop` + `queryModelWithStreaming` + `runTools`。  
- **Arke：** `arke/agent/runner.py` 中 `LLMRunner.optimize()` 的 `for turn in range(max_turns)` + `OptimizationSession.run_tool()`。

Arke 仓库内 `docs/design/deprecated/cc-inspired-update.md` 已用对照表说明这一同构性。

**English:** Arke’s loop is already the same *shape* as Claude Code’s agent loop.

---

## 2. 值得参考或抽象复用的设计（按优先级）

### 2.1 单一核心循环 + 依赖注入（Single loop + injectable `callModel`）

| Claude Code | Arke 侧落地 |
|-------------|--------------|
| `query()` / `queryLoop` + `productionDeps().callModel` 可替换 | 将「HTTP + 解析响应 + usage 累计」抽成 **`LLMClient` 协议**（或 `TypedDict` + 函数），`optimize()` 只负责消息与 tool 编排；测试注入 fake client |

**可复用度：** 高（结构调整，不必抄实现）。

### 2.2 工具执行的并发语义（Partition + concurrency-safe）

| Claude Code | Arke 侧落地 |
|-------------|--------------|
| `partitionToolCalls`：连续且 `isConcurrencySafe` 的 tool 可并行；写路径串行 | 当前 `runner.py` 对 tool 多为 **for 循环串行**。可对 **纯观察、不改变 `ArkeEnv`** 的调用（如只读分析）标注 **只读**，批量并行；**修改 strategy / compile / profile** 仍串行 |

**可复用度：** 中高（需为每个 tool 标注副作用，与 `legal_actions` 一致）。

### 2.3 流式与长任务（Streaming + long-job completion）

| Claude Code | Arke 侧落地 |
|-------------|--------------|
| 流式 delta + `StreamingToolExecutor`（尽早执行、abort 时补齐 tool_result） | 编译/剖分可能较慢：可采用 **异步 job + 进度事件**，或对 API 暴露 **SSE**；思想上与「长工具需收口、避免悬空 tool_use」一致 |

**可复用度：** 中（取决于是否做实时 CLI/API 流式输出）。

### 2.4 上下文与预算（Compact / token budget）

| Claude Code | Arke 侧落地 |
|-------------|--------------|
| auto compact、prompt-too-long 恢复、`maxTurns`、task_budget | Arke 已有 `OptimizationBudget`、`max_turns`、轨迹；长会话可增 **轨迹摘要**（早期步压缩为摘要，保留近 k 步原始 tool 结果），与 CC 的 compact 同构 |

**可复用度：** 高（评测与多 kernel 连续优化阶段尤其重要）。

### 2.5 Skill / Agent 分层与仓库内 SOUL / AGENTS / skills

| Claude Code | Arke 侧落地 |
|-------------|--------------|
| `SKILL.md` frontmatter + Agent `skills[]` **预载**到首轮上下文 | Arke 已有 `SOUL.md`、`AGENTS.md`、`skills/**/SKILL.md`；可在 **构建 system prompt 或首条 user** 时按配置 **预注入** skill 正文，对齐「预载 playbook」而非仅靠模型记忆 |

**可复用度：** 高（偏文档与加载逻辑）。

### 2.6 权限与审批（`canUseTool`）

| Claude Code | Arke 侧落地 |
|-------------|--------------|
| 每次 tool 可走策略/人工批准 | 若未来开放写盘、远程执行、多租户，可对 **高敏 tool** 增加 **policy 回调**（允许/拒绝/需确认） |

**可复用度：** 中高（视产品边界而定）。

### 2.7 子代理 / 多角色（可选）

| Claude Code | Arke 侧落地 |
|-------------|--------------|
| `runAgent` 嵌套 `query`，独立 tool 池与 transcript | 可设想 **「探索策略」** 与 **「严格验证」** 等 profile（不同 system prompt + 不同 tool 白名单），由 orchestrator 调度，与 Semantic/Strategy 分离哲学一致 |

**可复用度：** 中（与 Roadmap 阶段权衡）。

### 2.8 可观测与特性开关

| Claude Code | Arke 侧落地 |
|-------------|--------------|
| GrowthBook、埋点、`bun:bundle` 的 `feature()` 裁剪 | 实验性 codegen 路径可用 **环境变量或小型 flag 表**；`RunResult` 已含 tokens/decisions 等，可扩展统一遥测字段 |

**可复用度：** 中。

---

## 3. 不必强行对齐的部分（Lower ROI）

- **Ink/React TUI、IDE Bridge、MCP 全链路**：Arke 核心是 **编译器 + Python agent**，除非做 IDE 插件，否则不必对齐整套终端产品形态。  
- **巨型单文件**：不必模仿超大单模块；保持 **`runner` / `session` / `tools_schema` / `prompts` 分离** 更清晰。

**English:** Skip porting the terminal IDE surface; keep Arke’s modular Python layout.

---

## 4. 与 Arke 已有文档的衔接

| Claude Code 概念 | Arke 落地抓手 |
|------------------|----------------|
| `query()` 异步生成器 | `cc-inspired-update.md` 草案中的 `optimization_loop` → `AsyncGenerator[OptimizationEvent]` |
| `QueryDeps` | `LLMClient` 协议 + 测试用 fake |
| `runTools` 分区 | `TOOL_METADATA` 增加 `side_effect: read \| write` 后并行只读步骤 |
| Skill 预载 | `build_system_prompt` 或首轮 user 拼接已解析的 `SKILL.md` |
| compact | 对 `session` 对话/轨迹做摘要 + 保留尾部原始步 |
| permission | `run_tool` 前增加 policy 钩子 |

详细代码级草案见：**`/home/blueyi/workspace/repos/arke/docs/design/deprecated/cc-inspired-update.md`**（若迁移到非 deprecated 目录，建议更新此表链接）。

---

## 5. 一句话总结

**English:** Arke’s compiler-verified `ArkeEnv` is the differentiator; the main transferable ideas from the Claude Code snapshot are **engineering patterns**: one injectable agentic loop, safe concurrency for read-only tools, streaming/long-job handling, trajectory compaction for long sessions, skill/playbook preloading, and optional permission hooks — not the UI stack or vendor-specific internals.

---

## 6. 相关文档（本仓库）

- [架构与端到端调用链](./claude-code-architecture-analysis.md)  
- [Agent / Skill 配置语义](./build-your-own-agent-from-claude-code-patterns.md)
