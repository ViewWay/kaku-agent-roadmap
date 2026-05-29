# Kaku Agent 开发路线图 — 详细版本规划

## Context

基于 5 个 AI Coding Agent 项目源码深度分析 (Kaku, Hermes, Claude Code v2.0.2 fork + v2.1.156 CHANGELOG, OpenAI Codex)，制定 Kaku Agent 引擎增强路线图。

当前版本 `0.11.0`，有 7 个未提交 WIP 工具模块。发布策略: 每 Phase 一个版本。总预估 17 任务, ~4820 LOC, 8+ 周。

---

## 版本映射

| 版本 | Phase | 主题 | 周期 |
|------|-------|------|------|
| **0.12.0** | G | 核心性能与安全 | W1-2 |
| **0.13.0** | H | 子代理与编排 | W3-4 |
| **0.14.0** | I | 扩展性 | W5-7 |
| **0.15.0** | J | 终端独特优势 | W8+ |

---

## v0.12.0 — 核心性能与安全 (Phase G, W1-2)

### G0: 集成 WIP 工具模块 (前置)

已有 7 个未提交文件需集成到主分发: `git.rs`(286行), `nav.rs`(312行), `monitor.rs`(73行)

**文件**: `ai_tools/mod.rs` (内联改为调用模块), `ai_tools/registry.rs` (确认 ToolDef)
**验证**: `make check` + `make test` 通过

### G1: 工具并行执行 (~250 LOC)

**文件**: `ai_tools/registry.rs` (ToolDef 添加 `is_readonly: bool`), `ai_chat_engine/mod.rs` (分发重构), `overlay/ai_chat/render.rs` (batch 展示)

只读工具 (18): fs_read, fs_list, pwd, shell_poll, web_fetch, web_search, read_url, project_summary, file_tree, project_detect, symbol_search, grep_search, memory_read, soul_read, git_status, git_diff, git_log, git_show, git_branch(list/current), dir_overview, disk_usage, worktree_list, worktree_diff, system_stats, task_list, task_get, agent_poll

写入工具 (16): fs_write, fs_patch, fs_mkdir, fs_delete, shell_exec, shell_bg, git_commit, git_branch(create/delete), dir_jump, dir_bookmark(save/remove), worktree_create, worktree_remove, task_create, task_update, agent_spawn

**方案**: readonly 组 `futures::join_all` 并行, writable 组按序串行
**验证**: AI 同时读取 3 文件确认并行 (log 时间戳)

### G2: 迭代压缩 + 反抖动 (~200 LOC)

**文件**: `ai_chat_engine/mod.rs`, `ai_chat_engine/compact.rs`

**方案**: 30s 冷却期反抖动, 保留上次摘要 diff 增量追加, 结构化模板 `[goals|decisions|pending|completed]`
**验证**: 30 轮对话, 摘要可读, 间隔 >= 30s

### G3: Death Spiral Prevention (~120 LOC)

**文件**: `ai_chat_engine/mod.rs`

**方案**: 连续 3 次 LLM 错误→注入策略提醒, 连续 5 次→强制终止; 工具执行错误不计入
**验证**: 模拟连续参数错误, 3 次注入, 5 次终止

### G4: Bash 沙箱基础 (~300 LOC)

**文件**: 新建 `ai_tools/sandbox.rs`, 修改 `ai_tools/shell.rs`

**方案**: 网络隔离 (阻止 curl/wget/nc), 危险路径拦截 (rm -rf /), 写入白名单 (cwd 子目录)
**验证**: `rm -rf /` 拦截, `curl` 拦截, `touch` 通过

---

## v0.13.0 — 子代理与编排 (Phase H, W3-4)

### H0: 集成 WIP 子代理模块 (前置)

集成 `subagent.rs`(243行), `tasks.rs`(261行), `worktree.rs`(97行), `mcp.rs`(304行)
**验证**: agent_spawn, task_create, worktree_create 可调用

### H1: 角色区分 + 深度限制 (~250 LOC)

**文件**: `ai_tools/subagent.rs`

**方案**: `AgentRole::Leaf` (不可再创建子代理) / `Orchestrator` (可创建 leaf), `max_depth: 3`
- Leaf 工具集: 去掉 agent_spawn
- Orchestrator 工具集: 保留 agent_spawn + task_*, agent_send
**验证**: orchestrator→leaf 通过, leaf→spawn 拒绝

### H2: 子代理间通信 (~200 LOC)

**文件**: 新建 `ai_tools/agent_comm.rs`, 修改 `subagent.rs`, `registry.rs`

**方案**: crossbeam channel, `agent_send(id, message)` 发送, `agent_messages()` 接收
**验证**: 两个 leaf 通过消息交换文件路径

### H3: Worktree 隔离增强 (~200 LOC)

**文件**: `ai_tools/worktree.rs`, `ai_tools/subagent.rs`

**方案**: 子代理默认在 git worktree 运行, Drop guard 自动清理, 失败回滚
**验证**: 子代理 worktree 修改文件, 清理后父代理不受影响

### H4: Plan 模式增强 (~300 LOC)

**文件**: `ai_chat_engine/mod.rs`, `overlay/ai_chat/state.rs`, `render.rs`, `input.rs`

**方案**: Plan 模式工具集过滤为只读, 结构化步骤输出, 用户 a/r/e 逐条审批
**验证**: 5 步计划, approve 3, reject 1, edit 1

---

## v0.14.0 — 扩展性 (Phase I, W5-7)

### I1: Hook 系统 (~500 LOC)

**文件**: 新建 `ai_tools/hooks.rs`, 修改 `ai_chat_engine/mod.rs`

8 个核心事件: SessionStart/End, UserPromptSubmit, Pre/PostToolUse, PermissionRequest, AgentStart/Stop
配置: `.claude/hooks.json`, JSON stdin/stdout 协议, 5s 超时
**验证**: PreToolUse hook 拦截 `rm` 命令

### I2: MCP HTTP/SSE 传输 (~350 LOC)

**文件**: `ai_tools/mcp.rs`, `ai_tools/mcp_types.rs`

`McpTransport::HttpSse`, POST JSON-RPC + SSE 解析, 指数退避重连, 30s 心跳
**验证**: HTTP/SSE MCP server 连接, 调用工具, 断线重连

### I3: Skill 系统 (~400 LOC)

**文件**: 新建 `ai_tools/skills.rs`, 修改引擎和 UI

`.claude/skills/{name}/SKILL.md` + YAML frontmatter (name, trigger, allowed/disallowed-tools, effort)
**验证**: code-review skill, `/code-review` 触发后工具集过滤

### I4: 审批增强 (~250 LOC)

**文件**: `ai_chat_engine/approval.rs`, `overlay/ai_chat/state.rs`

`Bash(git *)` 通配符, `Write(src/**/*.rs)` 路径 glob, `.claude/permissions.json` 持久化
**验证**: `Bash(git *)` 规则, git 不弹审批, `rm` 仍弹

---

## v0.15.0 — 终端独特优势 (Phase J, W8+)

### J1: 终端会话上下文注入 (~200 LOC)

**文件**: 新建 `ai_tools/terminal_context.rs`, 修改 `ai_chat_engine/mod.rs`

捕获 cwd + 最近 20 条命令 + 环境变量 + 前台进程, 注入 system message, 每轮刷新
**验证**: Agent 自动知道当前目录和最近命令

### J2: 实时命令流监控 (~350 LOC)

**文件**: `ai_tools/terminal_context.rs`, `overlay/termwindow/mod.rs`

`terminal_watch(cols, lines)` 返回终端文本快照, 每轮限 3 次
**验证**: `cargo test` 后 Agent 看到 test 输出

### J3: 内联 Diff 编辑器 (~500 LOC)

**文件**: `overlay/ai_chat/render.rs`, `state.rs`, `input.rs`

富文本 diff 红/绿高亮, j/k/J/K 导航, a/y/n/e 操作, hunk 级选择
**验证**: 3 文件变更, accept 2 hunk, reject 1

---

## 依赖关系

```
G0 ── G1 ── G2 ── G3 ── G4
      │
      └── H0 ── H1 ── H2
               ├── H3
               └── H4
G4 ────────────┤
               ├── I1 ── I2 ── I3
G1+G3 ─────────┤    └── I4
               │
H+I ────────── J1 ── J2 ── J3
```

## 总预估

| 版本 | 任务 | LOC | 周期 |
|------|------|-----|------|
| 0.12.0 | 5 (G0-G4) | ~1070 | W1-2 |
| 0.13.0 | 5 (H0-H4) | ~1200 | W3-4 |
| 0.14.0 | 4 (I1-I4) | ~1500 | W5-7 |
| 0.15.0 | 3 (J1-J3) | ~1050 | W8+ |
| **合计** | **17** | **~4820** | **8+ 周** |
