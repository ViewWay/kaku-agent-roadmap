# Kaku Agent 开发路线图

Kaku AI Agent 引擎增强计划 — 基于 5 个顶级 AI Coding Agent 源码深度分析。

## 版本规划

| 版本 | Phase | 主题 | 周期 |
|------|-------|------|------|
| v0.12.0 | G1 | 核心性能 | W1-2 |
| v0.13.0 | G2 | 安全基线 | W2-3 |
| v0.14.0 | H | 子代理与编排 | W3-5 |
| v0.15.0 | I | 扩展性 | W5-7 |
| v0.16.0 | J | 终端独特优势 | W7-9 |
| v0.17.0 | K | 生态进阶 | W9+ |

**总计**: 37 任务, ~10250 LOC, 12+ 周

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

### 关键设计决策

1. **终端感知前置** — 终端上下文注入从 Phase G2 (W2) 就开始，而非等到最后
2. **安全独立成版** — 沙箱和权限从性能 Phase 中拆出，独立为 G2 版本
3. **子代理完整生命周期** — 不只是 fork，包含角色/通信/隔离/心跳/transcript/聚合
4. **扩展性对标 Claude Code** — Hook 22 事件 + 5 种类型，MCP 完整协议层
5. **终端优势最大化** — 多 Tab 协作、命令流监控、内联 Diff、智能建议
6. **MCP 迁移到官方 RMCP SDK** — 替换自建 304 LOC 实现，获得完整协议支持 (HTTP/OAuth/Sampling)
7. **终端方案 W9 评估** — WezTerm (2年未更新) 风险待观察，Phase J 时决策继续/迁移/自研
8. **Warp Oz 轻量互操作** — 通过 MCP Server 标准协议对接，不深度绑定 Warp 平台
