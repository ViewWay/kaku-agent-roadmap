# Kaku Agent 开发路线图 — 详细版本规划

## Context

基于 5 个 AI Coding Agent 项目源码深度分析 (Kaku, Hermes, Claude Code v2.0.2 fork + v2.1.156 CHANGELOG, OpenAI Codex)，制定 Kaku Agent 引擎增强路线图。

当前版本 `0.11.0`，有 7 个未提交 WIP 工具模块。发布策略: 每 Phase 一个版本。总预估 37 任务, ~10700 LOC, 12+ 周。

### 相关设计文档

- [终端+Agent 工作流设计](docs/terminal-agent-design.md) — 自研终端架构、AI Agent 开发工作流交互设计、Phase 0 终端 MVP (~17800 LOC)
- [多平台产品架构](docs/multiplatform-design.md) — Core/Shell/Interface 三层分离、全平台 UI、LLM 多 Provider 策略

### 终端策略说明

本路线图的 Agent 引擎任务 (G1-K) 与终端选择解耦。当前基于 WezTerm 实现，J 阶段任务中的终端渲染/API 引用以 WezTerm 为例，W9 决策节点后可能迁移到 Ghostty 或自研终端。完整的自研终端设计见 [terminal-agent-design.md](docs/terminal-agent-design.md)，多平台架构见 [multiplatform-design.md](docs/multiplatform-design.md)。

---

## 三个战略决策

基于 2025-2026 市场调研 (Cursor $29B / Warp Oz / MCP 21K Server / RMCP SDK v1.7.0)，确立三个关键决策:

### 决策一: 终端方案 — 短期锁定, 中期评估

**现状风险**: WezTerm 超过 2 年未更新 (最后 release 2024-02-03)，作者定位为 "spare time project"。

| 方案 | 短期 (Phase G-H) | 中期 (Phase J 时点) |
|------|-----------------|-------------------|
| 继续 WezTerm | 锁定 API 边界层，不深度绑定渲染内部 | 评估迁移到 Ghostty (45K stars, 极高活跃度) 或自研轻量终端 (crossterm/ratatui, ~5-8K LOC) |
| 不推荐: Fork WezTerm | 15K 渲染引擎架构复杂，独自维护不可控 | — |

**决策节点**: Phase J 完成时 (W9)，根据产品-market fit 决定终端策略。

### 决策二: MCP 模块迁移到 RMCP SDK — 立即执行

**理由**: 自建 mcp.rs 仅 304 LOC 且仅 stdio，官方 RMCP v1.7.0 (3,468 stars) 已完整覆盖 Tools/Resources/Prompts/Sampling/OAuth/Streamable HTTP，且通过 87.5% conformance 测试。

**迁移范围**:
- 删除: mcp.rs (304 LOC) + mcp_types.rs (~100 LOC) = -404 LOC
- 新增: MCP Client wrapper (~80 LOC) + MCP Server (~250 LOC) + tokio 桥接 (~100 LOC) = +430 LOC
- 净变化: +26 LOC — 代码量不变，能力大幅提升

**双模架构**: 迁移后 Kaku 同时作为 MCP Client (消费 21K+ 外部 Server) 和 MCP Server (暴露 34 内置工具给其他 Agent)。

**路线图影响**: H0 (集成 mcp.rs) 替换为 RMCP 迁移；I2 (MCP 增强 800 LOC) 缩减为 ~300 LOC (RMCP 已内置 HTTP/OAuth/Sampling，只需配置和熔断器)。

### 决策三: Warp Oz 互操作 — 轻量优先, 分阶段

**Oz 平台**: Warp 的云端 Agent 编排平台，HTTP REST API，支持 schedule/event/CI 触发，700K+ 开发者。

| 路径 | 方式 | 依赖 | 时机 |
|------|------|------|------|
| **路径 A (推荐): MCP Server 对接** | Kaku 作为 MCP Server，Oz 通过标准 MCP 调用 | RMCP 迁移完成后自动具备 | Phase H (RMCP 迁移后) |
| **路径 B: Oz CLI Agent** | 通过 `oz agent run` 提交任务到 Oz 编排器 | 需要 Oz 账号认证 | Phase K (评估 Oz 开放程度后) |
| **不做: 深度 Oz 集成** | 注册为 Oz harness，深度绑定 Warp 平台 | 失去独立性 | — |

**决策节点**: Phase K 时评估 Oz 平台开放程度，决定是否接入 Oz CLI。

---

## 版本映射

| 版本 | Phase | 主题 | 周期 |
|------|-------|------|------|
| **0.12.0** | G1 | 核心性能 | W1-2 |
| **0.13.0** | G2 | 安全基线 | W2-3 |
| **0.14.0** | H | 子代理与编排 | W3-5 |
| **0.15.0** | I | 扩展性 | W5-7 |
| **0.16.0** | J | 终端独特优势 | W7-9 |
| **0.17.0** | K | 生态进阶 | W9+ |

---

## v0.12.0 — 核心性能 (Phase G1, W1-2)

> 目标: 消除核心性能瓶颈，从串行单线程进化到智能并行。

### G1.0: 集成 WIP 工具模块 (前置)

已有 7 个未提交文件需集成到主分发: `git.rs`(286行), `nav.rs`(312行), `monitor.rs`(73行)

**文件**: `ai_tools/mod.rs` (内联改为调用模块), `ai_tools/registry.rs` (确认 ToolDef)
**验证**: `make check` + `make test` 通过

### G1.1: 工具并行执行 (~250 LOC)

**文件**: `ai_tools/registry.rs` (ToolDef 添加 `parallel_safety: ParallelSafety`), `ai_chat_engine/mod.rs` (分发重构), `overlay/ai_chat/render.rs` (batch 展示)

```rust
enum ParallelSafety { ReadOnly, PathScoped, Writable }
```

- **ReadOnly** (18 工具): fs_read, fs_list, pwd, shell_poll, web_fetch, web_search, read_url, project_summary, file_tree, project_detect, symbol_search, grep_search, memory_read, soul_read, git_status, git_diff, git_log, git_show, git_branch(list/current), dir_overview, disk_usage, worktree_list, worktree_diff, system_stats, task_list, task_get, agent_poll → `futures::join_all` 并行
- **Writable** (16 工具): fs_write, fs_patch, fs_mkdir, fs_delete, shell_exec, shell_bg, git_commit, git_branch(create/delete), dir_jump, dir_bookmark(save/remove), worktree_create, worktree_remove, task_create, task_update, agent_spawn → 按序串行
- **PathScoped** (预留): 写入但路径受限的工具，同类路径可并行

参考: Claude Code (只读并行 max10 + 写入串行), Hermes (NEVER_PARALLEL / PARALLEL_SAFE / PATH_SCOPED 三级)

**验证**: AI 同时读取 3 文件确认并行 (log 时间戳)

### G1.2: 流式工具执行 (~200 LOC) [新增]

**文件**: `ai_chat_engine/mod.rs`, `ai_client.rs`

**方案**: 模型流式输出时，一旦识别出完整工具调用 JSON，立即启动执行。不需要等模型完全停止生成。
- 解析器在 SSE 流中增量匹配 tool_use 块
- 完整块到达后立即提交到并行执行器
- 写入工具仍等待模型输出完成

参考: Claude Code Streaming Tool Execution

**验证**: 模型输出 3 个只读工具调用，第一个在模型输出完成前就开始执行

### G1.3: 迭代压缩 + 反抖动 (~350 LOC)

**文件**: `ai_chat_engine/mod.rs`, `ai_chat_engine/compact.rs`

**方案**:
- 30s 冷却期反抖动 (Hermes 有，当前 Kaku 无)
- 保留上次摘要 diff 增量追加 (迭代更新，非一次性生成)
- 结构化模板 12 段: `[goals|decisions|pending|completed|active_files|recent_changes|errors|constraints|user_preferences|context_summary|token_budget|open_questions]`
- focus_topic: 摘要时保留当前任务焦点，避免泛化

参考: Hermes 5 阶段压缩 + 12 段结构化模板 + 迭代摘要更新 + 反抖动

**验证**: 30 轮对话，摘要可读，间隔 >= 30s，焦点不丢失

### G1.4: Death Spiral Prevention (~120 LOC)

**文件**: `ai_chat_engine/mod.rs`

**方案**:
- 连续 3 次 LLM 错误 → 注入策略提醒 (降级 prompt 或切换模型)
- 连续 5 次 LLM 错误 → 强制终止
- 连续 3 次工具超时 → 标记可疑工具，后续自动跳过
- 工具执行错误不计入 (仅 LLM 返回错误和超时)

参考: Claude Code withheld errors + fallback model

**验证**: 模拟连续参数错误，3 次注入，5 次终止

### G1.5: 工具结果缓存 (~150 LOC) [新增]

**文件**: `ai_tools/registry.rs`, 新建 `ai_tools/result_cache.rs`

**方案**:
- LRU 缓存 (容量 50)，key = (tool_name, params_hash, file_mtime)
- 命中时返回 `[Content cleared, result cached]` 占位符 + 缓存结果
- 文件读取类工具自动利用 (fs_read, symbol_search, grep_search)
- 写入工具自动失效相关缓存

参考: Claude Code `cache_deleted_input_tokens` 优化

**验证**: 同一文件读取两次，第二次从缓存返回，token 消耗降低

### G1.6: 工具结果大小控制 (~150 LOC) [新增]

**文件**: `ai_tools/registry.rs`, `ai_chat_engine/mod.rs`

**方案**:
- per-tool 阈值: 单个工具结果 > 10KB 截断为摘要 (保留首尾各 2KB + 中间省略)
- per-message 聚合预算: 单轮所有工具结果总和 > 50KB 触发压缩
- 大结果标记为 `[truncated, {original_size} bytes]`

参考: Claude Code 双层工具结果存储

**验证**: 工具返回 100KB 日志，截断为摘要，后续轮次引用不重复展开

### G1.7: 上下文预算系统 (~200 LOC) [新增]

**文件**: `ai_chat_engine/mod.rs`, `ai_chat_engine/budget.rs`

**方案**:
- 替换固定 25 轮硬上限为动态 token 预算 (默认 200K tokens)
- per-tool 预算: 读取类 2K，写入类 5K，搜索类 10K
- 每轮开始检查剩余预算，不足时主动触发压缩
- 预算耗尽 → 摘要 + 终止，而非硬截断

参考: Hermes IterationBudget (可退款机制), Claude Code per-tool 持久化阈值

**验证**: 高消耗会话（多文件搜索 + 大文件读取），预算用尽时优雅结束

### G1.8: Prompt Cache Stability (~100 LOC) [新增]

**文件**: `ai_tools/registry.rs`, `ai_chat_engine/mod.rs`

**方案**:
- 工具 schema 排序: 按调用频率降序排列，高频工具 schema 构成连续前缀
- MCP 工具追加到末尾 (避免打破内置工具的缓存前缀)
- 会话间保持排序一致性

参考: Claude Code prompt cache stability

**验证**: 连续两轮调用，prompt 前缀 byte 完全相同

---

## v0.13.0 — 安全基线 (Phase G2, W2-3)

> 目标: 建立沙箱安全、权限增强、工具超时，形成安全基线。

### G2.0: Bash 沙箱 (~450 LOC)

**文件**: 新建 `ai_tools/sandbox.rs`, 修改 `ai_tools/shell.rs`

**方案**:
- **网络隔离**: 拦截 curl/wget/nc/ssh 等网络命令 (应用层拦截，非 OS 沙箱)
- **危险路径拦截**: `rm -rf /`, 任何修改 `/etc`、`/usr`、`/System`、`~/` 根目录的操作
- **写入白名单**: 默认只允许 cwd 子目录写入，例外: `/tmp/kaku-*`, `$XDG_CONFIG_HOME`
- **环境变量过滤**: 注入命令前移除 `__CF_*`, `AWS_*`, `GITHUB_TOKEN` 等敏感变量 (除非显式授权)
- **进程资源限制**: CPU 时间 30s, 内存 512MB, 输出 1MB (ulimit wrapper)
- **macOS 适配**: 可选 `sandbox-exec` 集成 (profile: `kaku-agent.sb`)

参考: Codex (bwrap/Landlock/Seatbelt 四平台), Claude Code Bash sandbox + 网络控制

**验证**: `rm -rf /` 拦截, `curl` 拦截, `touch ./newfile` 通过, `echo $GITHUB_TOKEN` 空输出

### G2.1: 权限增强 (~250 LOC)

**文件**: `ai_tools/approval.rs`, 新建 `.claude/permissions.json`

**方案**:
- glob 匹配: `Bash(git *)`, `Write(src/**/*.rs)`, `Read(.env*)`
- 三级策略: `allow` (自动通过), `deny` (自动拒绝), `ask` (弹审批)
- 持久化到 `.claude/permissions.json`
- 工具级别默认策略: 只读工具默认 allow, 写入工具默认 ask
- 会话级临时规则: 本次会话有效，不持久化

参考: Claude Code 权限系统 + auto-mode, Codex PermissionProfile

**验证**: `Bash(git *)` 规则, git 不弹审批, `rm` 仍弹

### G2.2: 工具超时防护 (~80 LOC) [新增]

**文件**: `ai_tools/registry.rs`, `ai_chat_engine/mod.rs`

**方案**:
- 每个工具默认超时: 读取 10s, 写入 30s, 搜索 15s, shell 120s
- 可在 ToolDef 中自定义
- 超时后强制终止进程，返回 `[Tool timed out after {N}s]`
- 超时不计入 death spiral 计数 (G1.4 独立追踪)

**验证**: 工具挂起 10s 后自动终止，agent 继续运行

### G2.3: 终端上下文注入 (前置版) (~200 LOC) [从 J 前置]

**文件**: 新建 `ai_tools/terminal_context.rs`, 修改 `ai_chat_engine/mod.rs`

**方案**:
- 注入到 system message: cwd + 最近 20 条命令 + 关键环境变量 (PATH, PWD, SHELL, HOME)
- 每轮刷新 (不缓存，命令历史会变)
- 命令标记成功/失败状态 (exit code)
- **前置理由**: 让"终端感知"从一开始就成为产品的一部分，而非 8 周后的附加功能

参考: Kaku 独有差异化: 终端内嵌 AI, Codex Shell Snapshot

**验证**: Agent 自动知道当前目录和最近命令的执行结果

---

## v0.14.0 — 子代理与编排 (Phase H, W3-5)

> 目标: 从单 Agent 进化到多 Agent 协作，建立编排基础。

### H0: 集成 WIP 模块 + RMCP 迁移 (前置)

集成 `subagent.rs`(243行), `tasks.rs`(261行), `worktree.rs`(97行)。
**不集成 mcp.rs** — 改为迁移到官方 RMCP SDK (v1.7.0)。

**RMCP 迁移** (~430 LOC 新增 / 404 LOC 删除):
- 删除: mcp.rs (304 LOC), mcp_types.rs (~100 LOC) — 自定义 JSON-RPC 实现已过时
- 新增: `ai_tools/mcp_client.rs` (~80 LOC) — RMCP Client wrapper，支持 stdio + Streamable HTTP
- 新增: `ai_tools/mcp_server.rs` (~250 LOC) — RMCP Server，暴露 34 内置工具给外部 Agent
- 新增: `ai_tools/tokio_bridge.rs` (~100 LOC) — tokio runtime 桥接层 (Kaku 主架构保持 OS 线程 + mpsc)
- Cargo.toml: 添加 `rmcp` (git dep, branch=main), `tokio` (features=["sync","macros","rt","time"])

**双模架构**: 迁移后 Kaku 同时是 MCP Client (消费 21K+ 外部 Server) 和 MCP Server (暴露工具)。

集成时一并设计 H1-H3 的接口，避免集成后再改。

参考: [RMCP Rust SDK](https://github.com/modelcontextprotocol/rust-sdk) (v1.7.0, 3,468 stars), [Building MCP Servers in Rust](https://rup12.net/posts/write-your-mcps-in-rust/)

**验证**: agent_spawn/task_create/worktree_create 可调用; 通过 MCP 连接外部 Server 并调用工具; 作为 MCP Server 被外部 Client 发现和调用

### H1: 角色区分 + 深度限制 (~250 LOC)

**文件**: `ai_tools/subagent.rs`

**方案**:
```rust
enum AgentRole { Leaf, Orchestrator }
struct SubagentConfig {
    role: AgentRole,
    max_depth: u8,        // 默认 3
    allowed_tools: Vec<String>,  // 细粒度工具控制
    timeout: Duration,     // 默认 120s
    heartbeat_interval: Duration,
}
```
- Leaf 工具集: 去掉 agent_spawn, task_create, task_update
- Orchestrator 工具集: 保留 agent_spawn + task_*, agent_send
- 深度限制: `max_depth: 3`，超限拒绝创建
- 自定义角色: 用户可指定允许的工具集

参考: Hermes leaf/orchestrator, Claude Code AgentDefinition

**验证**: orchestrator→leaf 通过, leaf→spawn 拒绝, depth=3 时拒绝第四层

### H2: 子代理间通信 (~250 LOC)

**文件**: 新建 `ai_tools/agent_comm.rs`, 修改 `subagent.rs`, `registry.rs`

**方案**:
- 传输: crossbeam channel (内存内) + 持久化日志 (防止断连丢消息)
- `agent_send(id, message)` 发送, `agent_messages()` 接收
- 消息格式: JSON `{from, to, type, payload, timestamp}`
- 消息类型: `info`, `result`, `request`, `error`
- 接收缓冲: 最多 100 条，溢出丢弃最旧

参考: Codex Channel 通信, Hermes 文件协调

**验证**: 两个 leaf 通过消息交换文件路径，一个崩溃后另一个仍能收到之前的消息

### H3: Worktree 隔离增强 (~200 LOC)

**文件**: `ai_tools/worktree.rs`, `ai_tools/subagent.rs`

**方案**:
- 子代理默认在 git worktree 运行
- Drop guard 自动清理: 子代理退出后 worktree 自动删除
- 失败回滚: 写入操作失败时自动 `git checkout .`
- 环境继承: 从父代理继承 PATH/PWD/SHELL，过滤敏感变量 (G2.0 策略)
- 凭证继承: 可选，默认不继承 (安全优先)

参考: Hermes workspace isolation + 凭证继承, Claude Code worktree 隔离

**验证**: 子代理 worktree 修改文件，清理后父代理不受影响，敏感环境变量未泄露

### H4: 子代理心跳与超时回收 (~150 LOC) [新增]

**文件**: `ai_tools/subagent.rs`

**方案**:
- 心跳间隔: 每 10s 发送心跳到父代理
- 超时检测: 3 次心跳未收到 → 标记为可能死亡
- 超时回收: 5 次心跳未收到 → 强制终止 + 日志记录
- 资源清理: 终止时自动清理 worktree + 临时文件

参考: Hermes 心跳机制, Claude Code 子代理孤儿进程处理

**验证**: 子代理挂起，5x 心跳后自动回收，资源已清理

### H5: 子代理 Transcript (~200 LOC) [新增]

**文件**: 新建 `ai_tools/subagent_transcript.rs`, 修改 `subagent.rs`

**方案**:
- 每个子代理独立记录完整对话历史 (prompt + tools + response)
- 父代理可通过 `agent_transcript(id)` 查询
- Transcript 有 token 上限 (默认 50K)，超限自动压缩
- 支持导出为 Markdown (用于审计和调试)

参考: Claude Code 独立 transcript

**验证**: 子代理执行 5 轮后，父代理获取完整 transcript

### H6: 子代理结果聚合 (~200 LOC) [新增]

**文件**: 修改 `subagent.rs`, `ai_chat_engine/mod.rs`

**方案**:
- 多子代理结果汇总: 按相关性排序，去重
- 结构化返回: `{results: [{agent_id, summary, artifacts, token_cost}], deduped: bool}`
- 失败子代理标记: `{status: "failed", error, partial_results}`
- 父代理可选择: 取最佳结果 / 合并所有结果 / 重新分配

**验证**: 3 个 leaf 执行同一文件的不同修改，父代理收到聚合结果

---

## v0.15.0 — 扩展性 (Phase I, W5-7)

> 目标: 建立可扩展架构，降低第三方集成门槛。

### I1: Hook 系统 (~700 LOC)

**文件**: 新建 `ai_tools/hooks.rs`, 修改 `ai_chat_engine/mod.rs`

**配置**: `.claude/hooks.json`

**事件 (22 个)**:
- Session: `SessionStart`, `SessionEnd`, `SessionPause`, `SessionResume`
- Prompt: `UserPromptSubmit`, `AssistantResponse`
- Tool: `PreToolUse`, `PostToolUse`, `ToolTimeout`
- Agent: `SubagentStart`, `SubagentStop`, `AgentError`
- Context: `CompactStart`, `CompactEnd`, `BudgetWarning`
- System: `ModelSwitch`, `NotificationReceive`
- Memory: `MemoryRead`, `MemoryWrite`
- Approval: `PermissionRequest`

**类型 (5 种)**:
- `command`: shell 命令，JSON stdin/stdout，5s 超时
- `prompt`: LLM 判断，适合需要理解语义的场景
- `http`: Webhook POST，适合 CI/CD 集成
- `function`: 会话级 Rust 函数，适合高性能场景
- `callback`: 进程内回调，适合临时逻辑

**特性**: 并行执行、条件匹配 (`if: "Bash(git *)"`)、once 标记、continueOnBlock

参考: Claude Code 28 Hook 事件 + 6 种类型

**验证**: PreToolUse hook 拦截 `rm` 命令, SubagentStart hook 记录子代理创建

### I2: MCP 配置与运维层 (~300 LOC)

> 基于 H0 的 RMCP 迁移，Streamable HTTP/OAuth/Sampling 已由 RMCP 内置。此任务聚焦配置管理和运维策略。

**文件**: 修改 `ai_tools/mcp_client.rs`, 新建 `ai_tools/mcp_config.rs`

**方案**:
- MCP Server 配置管理: `.claude/mcp_servers.json` 声明式配置 (server 命令/URL/环境变量/OAuth)
- 熔断器策略: 连续 5 次失败自动断开，60s 后半开重试
- 工具命名空间: `mcp__{server}__{tool}` 格式，避免多 Server 命名冲突
- 会话恢复: 重连后恢复工具列表和会话状态
- 集成 Smithery/mcp.so: 支持 `npx @smithery/cli add <name>` 一键安装社区 MCP Server

参考: RMCP 内置 Streamable HTTP + OAuth 2.1 + Sampling; Hermes 熔断器策略

**验证**: HTTP/SSE MCP server 连接, 调用工具, 断线重连后状态恢复, 一键安装社区 MCP Server

### I3: Skill 系统 (~400 LOC)

**文件**: 新建 `ai_tools/skills.rs`, 修改引擎和 UI

**目录格式**: `.claude/skills/{name}/SKILL.md`
```yaml
---
name: code-review
trigger: ["/code-review", "code review"]
allowed-tools: [Read, Grep, Glob, LSP]
disallowed-tools: [Write, Edit, Bash]
effort: medium
paths: ["src/**/*.rs"]
---
```

**特性**:
- 条件激活: `paths` 字段匹配当前工作区文件类型时自动激活
- 工具过滤: `allowed-tools` / `disallowed-tools` 约束可用工具集
- 动态加载: 热加载/卸载，无需重启

参考: Claude Code Skills + Plugins

**验证**: code-review skill, `/code-review` 触发后工具集过滤

### I4: LSP 工具集成 (~300 LOC) [新增]

**文件**: 新建 `ai_tools/lsp_tool.rs`, 修改 `ai_tools/mod.rs`

**方案**:
- 利用 WezTerm 已有 LSP 支持，桥接到 Agent 工具层
- 5 个核心操作: `goToDefinition`, `findReferences`, `hover`, `documentSymbol`, `workspaceSymbol`
- 自动检测工作区的语言服务器配置
- 结果格式化为结构化文本注入上下文

参考: Claude Code LSP Tool (v2.0.74)

**验证**: Agent 使用 `workspaceSymbol("MyStruct")` 找到定义位置

### I5: 配置管理 (~200 LOC) [新增]

**文件**: 新建 `config/mod.rs`, `.claude/settings.json`

**方案**:
- 统一配置入口: `.claude/settings.json`
- 分层配置: global (`~/.claude/settings.json`) > project (`.claude/settings.json`) > session
- 配置项: 模型选择、权限规则、Hook 配置、MCP 服务器列表、工具预算
- 配置验证: 启动时校验 schema，无效配置告警

参考: Claude Code settings.json

**验证**: 项目级 settings 覆盖全局 settings，session 级临时设置生效

---

## v0.16.0 — 终端独特优势 (Phase J, W7-9)

> 目标: 放大"终端内嵌 AI"独特优势，建立竞品无法复制的护城河。
> **决策节点 (W9)**: Phase J 完成时评估终端方案 — 继续 WezTerm / 迁移 Ghostty (45K stars) / 自研轻量终端。

### J1: 终端会话上下文注入 (增强版) (~250 LOC)

**文件**: `ai_tools/terminal_context.rs`, `ai_chat_engine/mod.rs`

**方案** (基于 G2.3 前置版增强):
- 命令历史语义分析: 标记成功/失败/超时/中断
- 前台进程状态: 知道正在运行的编译/测试/部署进度
- 终端布局感知: 窗口大小、pane 布局、字体大小
- 智能摘要: 最近 50 条命令压缩为关键事件流 (而非原始列表)

参考: Kaku 独有差异化, Codex Shell Snapshot

**验证**: Agent 知道 `cargo build` 刚失败并看到错误摘要

### J2: 实时命令流监控 (~350 LOC)

**文件**: `ai_tools/terminal_context.rs`, `overlay/termwindow/mod.rs`

**方案**:
- `terminal_watch(cols, lines)` 返回终端文本快照
- 自适应限流: 空闲时每轮最多 3 次，有编译/测试运行时放宽到 5 次
- 智能截取: 只捕获变化部分 (diff against previous snapshot)
- 结构化解析: 从终端输出中提取编译错误、测试结果、进度信息

参考: Kaku 独有差异化

**验证**: `cargo test` 后 Agent 看到 test 输出和失败摘要

### J3: 内联 Diff 编辑器 (~500 LOC)

**文件**: `overlay/ai_chat/render.rs`, `state.rs`, `input.rs`

**方案**:
- 富文本 diff: 红/绿高亮, 利用 WezTerm 的终端颜色能力
- 导航: j/k 逐行, J/K 按 hunk 跳转, g/G 首尾
- 操作: a (accept all), y (accept hunk), n (reject hunk), e (edit in editor)
- 并行 diff: 多文件变更同时展示, Tab 切换
- 统计: 左上角显示 `+{added} -{removed} {file_count} files`

参考: Kaku 已有 Diff 预览审批的增强版

**验证**: 3 文件变更, accept 2 hunk, reject 1, e 打开编辑器

### J4: 多 Tab AI 协作 (~500 LOC) [新增]

**文件**: 新建 `ai_tools/tab_cooperation.rs`, 修改 `ai_tools/terminal_context.rs`

**方案**:
- 利用 WezTerm 多 Tab 能力，不同 Tab 可运行独立 Agent
- 跨 Tab 上下文共享: 当前 Tab、工作目录、git branch
- 消息总线: Tab A 的 Agent 可向 Tab B 发送结果
- 统一视图: `Tab` 工具列出所有 Tab 的 Agent 状态
- 权限控制: Agent A 不能读取 Tab B 的对话历史 (只共享元数据)

参考: Kaku 独有差异化 (所有竞品无此能力)

**验证**: Tab 1 运行测试 Agent，Tab 2 运行开发 Agent，开发 Agent 知道测试结果

### J5: Agent 通知系统 (~200 LOC) [新增]

**文件**: 新建 `ai_tools/notification.rs`, `overlay/termwindow/mod.rs`

**方案**:
- 后台 Agent 完成时终端内通知 (利用 WezTerm bell + 自定义通知栏)
- 通知级别: info (完成), warning (需关注), error (失败)
- 通知队列: 最多 20 条，可查看历史
- 快捷操作: 通知栏可直接跳转到对应 Agent/Tab

参考: Claude Code 桌面通知

**验证**: 后台 Agent 完成任务，用户在其他 Tab 收到通知，点击跳转

### J6: 智能命令建议 (~300 LOC) [新增]

**文件**: `ai_tools/terminal_context.rs`, `overlay/ai_chat/input.rs`

**方案**:
- Agent 基于上下文主动建议下一步命令
- 建议来源: 历史模式 (用户之前类似的操作流)、当前状态 (编译失败→建议 fix 命令)
- 展示: 在输入行上方显示灰色的建议命令，Tab 接受
- 可配置: `.claude/settings.json` 中 `suggest_commands: true/false`

**验证**: 编译失败后，Agent 建议正确的编译命令，用户 Tab 接受

---

## v0.17.0 — 生态进阶 (Phase K, W9+)

> 目标: 建立 Plugin 生态，编排进阶，Auto Mode。

### K1: Plugin 架构 (~500 LOC)

**文件**: 新建 `plugin/mod.rs`, `plugin/loader.rs`, `plugin/manifest.rs`

**方案**:
- Plugin 可提供: Skills + Hooks + MCP Servers + 工具定义
- 清单格式: `.claude/plugins/{name}/manifest.json`
- 生命周期: install → enable → load → unload → disable
- 沙箱: Plugin 代码在独立线程运行，异常不影响主 Agent

参考: Claude Code Plugin System (v2.0.12)

### K2: Auto Mode 审批分类器 (~250 LOC)

**文件**: `ai_tools/approval.rs`, 新建 `ai_tools/risk_classifier.rs`

**方案**:
- 基于规则的风险分类: 读操作→低风险(自动通过), 写操作→中风险(仅首次询问), 破坏性操作→高风险(始终询问)
- 学习机制: 用户手动审批过一次的操作，后续同类操作自动降级
- 全局开关: `auto_mode: {read: true, write: "learn", destructive: false}`

参考: Claude Code Auto Mode + 权限分类器

### K3: Plan 模式增强 (~300 LOC)

**文件**: `ai_chat_engine/mod.rs`, `overlay/ai_chat/state.rs`, `render.rs`, `input.rs`

**方案**:
- Plan 模式工具集过滤为只读
- 结构化步骤输出: `Step {id, description, files, tools, status}`
- 用户逐条审批: a(accept), r(reject), e(edit step)
- 验证循环: 执行后对比预期 vs 实际结果

参考: Codex PlanHandler + delta

### K4: 会话持久化 (~200 LOC)

**文件**: 新建 `ai_tools/session.rs`, 修改 `ai_chat_engine/mod.rs`

**方案**:
- 会话快照: 每轮结束保存对话状态到 `.claude/sessions/{id}.json`
- `/resume` 命令: 恢复上次会话，压缩后继续
- 自动清理: 超过 7 天的会话自动归档
- 导出: 会话可导出为 Markdown

参考: Claude Code /resume, Hermes 会话持久化

### K5: Feature Flag 体系 (~150 LOC)

**文件**: 新建 `config/feature_flags.rs`

**方案**:
- JSON 配置: `.claude/feature_flags.json`
- 开关类型: boolean, percentage (灰度), user_list
- 运行时切换: 无需重启
- 适用于: 新功能灰度发布、A/B 测试 prompt

参考: Codex 50+ Feature Flags

### K6: Dynamic Workflows (~600 LOC)

**文件**: 新建 `ai_tools/workflow.rs`, 修改 `subagent.rs`

**方案**:
- 声明式工作流: YAML 定义 Agent 依赖图和执行顺序
- 编排能力: 并行分支、条件分支、结果聚合、失败重试
- 可视化: 在终端中展示工作流 DAG 进度
- 适用于: CI/CD pipeline、批量代码审查、大规模重构

参考: Claude Code Dynamic Workflows (v2.1.154), Codex CSV Jobs

---

## 依赖关系

```
G1.0 ── G1.1 ── G1.2 ── G1.3 ── G1.4
│         │         │
│         ├── G1.5 ──┤
│         ├── G1.6 ──┤
│         ├── G1.7 ──┤
│         └── G1.8 ──┘
│
G1.1+G1.4 ── G2.0 ── G2.1 ── G2.2
                           │
                           └── G2.3(终端上下文前置)
G2 ─────────────── H0(RMCP迁移) ── H1 ── H2
                                    ├── H3
                                    ├── H4
                                    ├── H5
                                    └── H6
H+RMCP ────────── I1 ── I2(配置层) ── I3
              ├── I4              └── I5
              └── I5
I ─────────────── J1 ── J2 ── J3
              ├── J4
              ├── J5
              └── J6
J ─────────────── K1 ── K2 ── K3
              ├── K4
              ├── K5
              └── K6
```

---

## 总预估

| 版本 | Phase | 任务 | LOC | 周期 |
|------|-------|------|-----|------|
| 0.12.0 | G1 | 9 (G1.0-G1.8) | ~1320 | W1-2 |
| 0.13.0 | G2 | 4 (G2.0-G2.3) | ~980 | W2-3 |
| 0.14.0 | H | 7 (H0-H6) | ~1880 | W3-5 |
| 0.15.0 | I | 5 (I1-I5) | ~1900 | W5-7 |
| 0.16.0 | J | 6 (J1-J6) | ~2100 | W7-9 |
| 0.17.0 | K | 6 (K1-K6) | ~2000 | W9+ |
| **合计** | | **37** | **~10700** | **12+ 周** |

---

## 与竞品的功能对标

| 功能 | 当前 | G1 | G2 | H | I | J | K |
|------|------|----|----|---|---|---|---|
| 工具并行 | 串行 | G1.1 | | | | | |
| 迭代压缩 | 基础 | G1.3 | | | | | |
| 流式执行 | 无 | G1.2 | | | | | |
| 结果缓存 | 无 | G1.5 | | | | | |
| Bash 沙箱 | 无 | | G2.0 | | | | |
| 权限系统 | glob | | G2.1 | | | | |
| 终端上下文 | 无 | | G2.3 | | J1 | | |
| 子代理角色 | 无 | | | H1 | | | |
| 代理通信 | 无 | | | H2 | | | |
| Worktree 隔离 | 无 | | | H3 | | | |
| Hook 系统 | 无 | | | | I1 | | |
| MCP 增强 | stdio | | | | I2 | | |
| Skill 系统 | 无 | | | | I3 | | |
| LSP 工具 | 无 | | | | I4 | | |
| 命令流监控 | 无 | | | | | J2 | |
| 内联 Diff 编辑 | 基础 | | | | | J3 | |
| 多 Tab 协作 | 无 | | | | | J4 | |
| Plugin 生态 | 无 | | | | | | K1 |
| Auto Mode | 无 | | | | | | K2 |
| Dynamic Workflows | 无 | | | | | | K6 |
| 会话恢复 | 无 | | | | | | K4 |

**仍在竞品之下** (可考虑后续版本):
- Claude Code 6 层压缩管线 (Kaku 有 2 层→增强后约 4 层)
- Codex 4 平台沙箱 (Kaku 只有应用层拦截)
- Hermes PTC (Rust 不兼容，不移植)
- Claude Code 400+ slash 命令 (Kaku Skill 生态需要时间积累)
- Codex Agent Identity 密码学体系 (需要评估必要性)
