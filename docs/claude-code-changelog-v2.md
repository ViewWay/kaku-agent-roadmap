---
name: claude-code-changelog-v2
description: Claude Code v2.0.1→v2.1.156 完整变更记录 — 600 条变更分类、Top 10 新功能、Top 10 修复、废弃列表
metadata:
  type: project
---

# Claude Code v2.0.1 → v2.1.156 变更分析

## 统计
- 总变更: ~600 条, 跨 156 版本
- 新功能: ~350, 缺陷修复: ~230, 废弃: ~20

## Top 10 新功能

| # | 功能 | 版本 | 说明 |
|---|------|------|------|
| 1 | Dynamic Workflows | v2.1.154 | 编排数十到数百后台 Agent |
| 2 | Agent View | v2.1.139 | 统一会话仪表板 |
| 3 | Plugin System | v2.0.12 | 命令+Agent+Hook+MCP 可扩展 |
| 4 | 1M Context Window | Opus 4.6+ | 超大上下文 |
| 5 | Auto Mode | v2.1.154 | 无需 opt-in 权限分类器 |
| 6 | LSP Tool | v2.0.74 | 语言服务器代码智能 |
| 7 | Remote Control | v2.1.79 | 桥接移动端/Web |
| 8 | Claude in Chrome | v2.0.72 | 直接控制浏览器 |
| 9 | MCP Elicitation | v2.1.76 | 交互式结构化输入 |
| 10 | Fullscreen 渲染器 | v2.1.89 | 无闪烁虚拟化回滚 |

## 新功能分类

### Agent 循环
- Thinking 默认开启、Plan 模式增强、Auto Mode 去 opt-in
- Auto-compact 改进、/context 命令、Effort 等级 (low~xhigh)
- Memory 系统 (MEMORY.md 索引)、Streaming Tool Execution

### 多代理
- Dynamic Workflows、Agent View、Background Sessions (claude --bg)
- Agent Teams、Worktree 隔离、Task 依赖追踪

### 工具 & MCP
- LSP 工具、MCP Tool Search 上下文启用
- HTTP/SSE/WS 传输、OAuth step-up、Marketplace
- Binary 内容 (PDF/Office/音频)

### Hook & 扩展
- 28+ Hook 事件 (含 SubagentStart/Stop, MessageDisplay, Elicitation)
- Prompt-based Hooks、HTTP Hooks、once:true、continueOnBlock

### UI/UX
- Fullscreen 渲染器、自定义主题、Vim 模式 (含 visual)
- 多行输入、IME、外部编辑器 (Ctrl+G)、Focus Mode
- 桌面通知、Voice Mode (20语言)

### 安全
- Bash 沙箱、网络隔离、Permission 通配符
- Managed Settings 企业策略、auto-mode 安全分类器

### IDE & 平台
- VS Code 原生扩展、JetBrains、Chrome 扩展
- Windows PowerShell 工具、macOS sandbox、Linux bwrap

## Top 10 关键修复

| # | 修复 | 严重性 |
|---|------|--------|
| 1 | Bash 命令注入 (v2.1.7/98/117) | 严重 |
| 2 | 权限规则通配符绕过 (v2.1.117) | 严重 |
| 3 | 长会话内存泄漏 (v2.1.70/121) | 高 |
| 4 | OAuth Token 刷新竞态 (v2.1.117/136) | 高 |
| 5 | /resume 大会话崩溃 (v2.1.116/149) | 高 |
| 6 | 子代理模型继承错误 (v2.1.74/117) | 高 |
| 7 | Windows 路径处理 (v2.1.50/106) | 中 |
| 8 | MCP 工具结果截断 (v2.1.89) | 中 |
| 9 | Plan 模式状态丢失 (v2.1.121/122) | 中 |
| 10 | 后台 Agent 孤儿进程 (v2.1.50/144) | 中 |

## 废弃/移除
- /fork → /branch、output styles → --system-prompt-file
- .claude.json → settings.json、TaskOutput → Read 文件
- Legacy SDK → @anthropic-ai/claude-agent-sdk、--tag 移除

## 对 Kaku 的启示
1. 安全修复频率高 (3 次命令注入) — Kaku 也需定期安全审计
2. Dynamic Workflows 是多代理终局形态 — Kaku 的子代理需向此演进
3. Plugin System 是生态基础 — Kaku 应尽早设计可扩展架构
4. LSP 集成是标配 — Kaku 可利用 WezTerm 已有的 LSP 支持
5. Fullscreen 渲染器投入大 — Kaku 已有 WezTerm 渲染引擎，可直接利用

## 相关记忆
[[five-project-comparison]] — 5 项目全面对比
[[claude-code-analysis]] — Claude Code fork 源码分析
