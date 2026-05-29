# Kaku Agent 开发路线图

Kaku AI Agent 引擎增强计划 — 基于 5 个顶级 AI Coding Agent 源码深度分析。

## 版本规划

| 版本 | Phase | 主题 | 周期 |
|------|-------|------|------|
| v0.12.0 | G | 核心性能与安全 | W1-2 |
| v0.13.0 | H | 子代理与编排 | W3-4 |
| v0.14.0 | I | 扩展性 | W5-7 |
| v0.15.0 | J | 终端独特优势 | W8+ |

## 分析项目

| 项目 | 语言 | 定位 | LOC |
|------|------|------|-----|
| Kaku | Rust | 终端内嵌 AI | ~15K |
| Hermes Agent | Python | 通用 Agent | ~35K |
| Claude Code (fork) | TypeScript | CLI Agent | 505K |
| Claude Code (official) | TypeScript | CLI Agent (闭源) | 未知 |
| OpenAI Codex | Rust+TS | 企业级 Agent | ~770K |

## 文档

- [AGENT_ROADMAP.md](AGENT_ROADMAP.md) — 详细版本规划和任务分解
- [5 项目全面对比](docs/five-project-comparison.md) — 对比矩阵和出圈策略
- [Claude Code 分析](docs/claude-code-analysis.md) — fork 源码分析
- [Claude Code 变更记录](docs/claude-code-changelog-v2.md) — v2.0.1→v2.1.156 变更
- [Hermes Agent 分析](docs/hermes-agent-analysis.md) — 核心模式和移植建议
- [Kaku 架构约束](docs/kaku-agent-constraints.md) — 当前架构瓶颈

## 核心策略

**不做"又一个 CLI Agent"，成为 "AI 原生终端" 品类定义者。**

Kaku 的根本差异化是唯一将 AI 深度嵌入终端的产品。
出圈策略: 让终端成为 AI 的第一界面，而非 AI 的附属品。
