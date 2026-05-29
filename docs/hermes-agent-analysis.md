---
name: hermes-agent-analysis
description: Hermes Agent 全面分析 — 架构、12 个核心模式、与 Kaku 的对比、推荐移植方向
metadata:
  type: project
---

# Hermes Agent 分析

## 项目概况
- Nous Research 开源 AI Agent，Python 3.11+ + TypeScript (TUI)
- 75+ 工具、18+ 平台适配器、25 技能类别、14 插件
- 核心文件: run_agent.py (~12K LOC)、tools/ (75 文件)、agent/ (52 文件)

## 核心创新模式 (Kaku 缺失)

### Tier 1 (高价值)
1. **PTC (Programmatic Tool Calling)** — LLM 写 Python 脚本调用工具，UDS/文件 RPC，迭代预算退款 (~1622 LOC)
2. **子代理编排** — leaf/orchestrator 角色、深度限制、凭证继承、心跳、跨代理文件协调 (~2598 LOC)
3. **Kanban 多代理任务板** — SQLite 持久化、worker 所有权、幻觉检测、工作区隔离 (~872 LOC)
4. **上下文压缩管线** — 5 阶段、迭代摘要更新、反抖动、12 段结构化模板、focus_topic (~1480 LOC)
5. **Curator 自改进** — 空闲触发、技能生命周期、LLM 驱动伞形合并、dry-run、备份 (~1675 LOC)

### Tier 2 (中等价值)
6. 并行工具执行安全分级 (NEVER_PARALLEL / PARALLEL_SAFE / PATH_SCOPED)
7. Hook 系统 (YAML 清单 + emit/emit_collect 决策型钩子)
8. MCP 增强 (HTTP/SSE 传输 + OAuth 2.1 + 采样 + 熔断器 + 会话恢复)
9. 提示词构建器 (注入扫描 + 双层缓存 + 条件可见性)

## Kaku 推荐移植方向
- P0: 并行工具执行 (~300 LOC)、迭代上下文压缩 (~200 LOC)、工具循环防护 (~150 LOC)
- P1: 子代理增强 (~500 LOC)、MCP HTTP 传输 (~400 LOC)、上下文文件注入 (~300 LOC)
- P2: Kanban 任务板、Hook 系统、自改进循环
- 不移植: PTC (Rust 不兼容)、18+ 平台适配器、React/Ink TUI

## 相关记忆
[[kaku-agent-constraints]] — Kaku 当前 agent 架构的约束和瓶颈
