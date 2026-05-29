---
name: claude-code-analysis
description: Claude Code 源码深度分析 — 505K LOC TypeScript，核心循环、61 工具、28 hook 事件、6 层压缩管线、Swarm 多代理
metadata:
  type: project
---

# Claude Code 源码分析

## 项目概况
- TypeScript/TSX, 2147 文件, 505K LOC
- 61 内置工具, 7 MCP 传输, 28 hook 事件, 6 层上下文压缩
- 核心: query.ts (~1946 LOC), QueryEngine.ts (~1370 LOC), hooks.ts (~5200 LOC)

## 核心架构
### 主循环
- **async generator** 模式 (非 class)，yield 流事件 + 工具结果
- 混合并行+串行工具分发: 只读工具并行 (max 10)，写入工具串行
- **Streaming Tool Execution**: 模型流式输出时即可开始执行只读工具
- Withheld errors + fallback model + death spiral prevention

### 上下文压缩 (6 层管线)
1. Snip Compact — 清理僵尸消息
2. Microcompact — 旧工具结果替换为 [Content cleared]，利用服务端 cache_deleted_input_tokens
3. Context Collapse — 粒度化摘要，commit log replay 恢复
4. Autocompact — 80% 阈值触发 LLM 摘要，有 circuit breaker
5. Predictive Autocompact — 预测当前轮增长会超限则主动压缩
6. Reactive Compact — prompt-too-long 紧急压缩（一次性）

### Hook 系统 (28 事件, 6 种类型)
- 事件: PreToolUse, PostToolUse, UserPromptSubmit, SessionStart/End, Stop, PermissionRequest 等 28 个
- 类型: command(shell), prompt(LLM), agent(LLM+上下文), HTTP(webhook), callback(进程内), function(会话级)
- 并行执行、异步 hook、条件匹配 (if: "Bash(git *)")、JSON 决策协议
- 5200 LOC hooks.ts

### 子代理
- 同进程内，独立 AbortController + transcript + agentId + ToolUseContext
- 父->子: prompt + ToolUseContext; 子->父: enqueuePendingNotification
- mid-turn 消息注入 (queuePendingMessage / drainPendingMessages)
- 无角色区分，自定义来自 AgentDefinition (.claude/agents/)

### Swarm 多代理
- 4 后端: tmux, iTerm2, Windows Terminal, InProcess
- 权限同步: file-based + mailbox 双系统，leader UI 处理 worker 权限请求
- 每个 teammate 可有独立 git worktree
- 隐藏/显示 pane (tmux)，dead pane recovery (iTerm2)
- 8475 LOC

### 工具系统 (61 工具)
- 核心: Agent, Bash, Read, Write, Edit, Grep, Glob, LSP, WebFetch, WebSearch 等
- 任务: TaskCreate/Get/List/Update, TodoWrite, TaskOutput/Stop
- 企业: CronCreate/Delete/List, TeamCreate/Delete, PowerShell, Workflow, SubscribePR
- MCP: 7 传输 (stdio, sse, sse-ide, http, ws, sdk, claudeai-proxy)
- 双层工具结果存储: per-tool 持久化阈值 + per-message 聚合预算
- prompt cache stability: built-in 排序为连续前缀，MCP 追加

### Skills + Plugins
- Skills: 目录格式 (skill-name/SKILL.md)，YAML frontmatter，条件激活 (paths 匹配)
- Plugins: 内置 + marketplace，可提供 skills + hooks + MCP servers
- 400+ slash 命令，feature flag 驱动 tree-shaking

## 相关记忆
[[hermes-agent-analysis]] — Hermes 对比
[[kaku-agent-constraints]] — Kaku 当前架构
