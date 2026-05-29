---
name: five-project-comparison
description: 5 大 AI Coding Agent 全面对比矩阵 — Kaku vs Hermes vs Claude Code vs OpenAI Codex，含独特功能识别和出圈路线图
metadata:
  type: project
---

# 5 大 AI Coding Agent 全面对比

## 项目概览

| 维度 | Kaku | Hermes Agent | Claude Code (fork v2.0.2) | Claude Code (official v2.1.156) | OpenAI Codex |
|------|------|-------------|--------------------------|-------------------------------|-------------|
| 语言 | Rust | Python 3.11+ | TypeScript | TypeScript (闭源) | Rust + TypeScript |
| LOC | ~15K (agent相关) | ~35K (agent+tools) | 505K | 未知 (闭源) | ~770K (codex-rs) |
| 许可 | 开源 | 开源 | fork 源码可见 | 闭源商业 | 开源 |
| LLM | 多供应商 | 多供应商 | Anthropic | Anthropic | OpenAI (紧耦合) |
| 工具数 | 34 内置 + MCP | 75+ 内置 + MCP | 61 内置 + MCP | 未知 | 20+ 内置 + MCP |
| 定位 | 终端内嵌 AI | 通用 Agent | CLI Agent | CLI Agent | 企业级 CLI/IDE Agent |

## 核心架构对比

### 主循环
| | Kaku | Hermes | Claude Code | Codex |
|--|------|---------|-------------|-------|
| 模式 | OS 线程 + mpsc | asyncio 事件循环 | async generator | tokio + submission queue |
| 最大轮次 | 25 | IterationBudget (可退款) | 无硬限制 | 无硬限制 |
| 工具执行 | 串行 | 安全分级并行 | 只读并行(max10)+写入串行 | 并行 (CoreToolRuntime trait) |
| 流式 | StreamMsg enum | EventEmitter | yield 流事件 | WebSocket SSE |
| 审批 | 同步阻塞 | Hook 集成 | Hook + 权限系统 | Approval + Guardian Review |

### 上下文压缩
| | Kaku | Hermes | Claude Code | Codex |
|--|------|---------|-------------|-------|
| 策略数 | 2 层 | 5 阶段 | 6 层管线 | 3 种 (local/remote v1/v2) |
| 触发 | 72KB 阈值 | 反抖动 + 迭代更新 | 80% 预测 + 紧急 | Pre-turn + Mid-turn |
| 摘要 | LLM 驱动 | 12 段结构化模板 | commit log replay | LLM 本体 + 服务端 |
| 反抖动 | 无 | 有 | 有 (circuit breaker) | 无 |
| 缓存利用 | 无 | 无 | cache_deleted_input_tokens | 无 |

### 工具系统
| | Kaku | Hermes | Claude Code | Codex |
|--|------|---------|-------------|-------|
| 注册方式 | 硬编码 match | 声明式 YAML | 声明式 + 动态 | HashMap + trait |
| Schema | 手写 JSON | YAML 自动生成 | 自动推断 | 显式 spec |
| 并行安全 | 无分级 | 3 级 (NEVER/SAFE/PATH) | 只读/写入 | trait 标注 |
| 结果存储 | 直接传递 | 持久化 | 双层阈值 | per-turn |

### MCP
| | Kaku | Hermes | Claude Code | Codex |
|--|------|---------|-------------|-------|
| 传输 | stdio only | stdio + HTTP/SSE | 7 种 (stdio/sse/http/ws等) | stdio + HTTP |
| OAuth | 无 | OAuth 2.1 | 无 | 有 (ElicitationRequestManager) |
| 采样 | 无 | 有 | 无 | 无 |
| 熔断 | 无 | 有 | 无 | 无 |
| 工具前缀 | 无 | 无 | 无 | 有 (server name prefix) |

### 子代理/多代理
| | Kaku | Hermes | Claude Code | Codex |
|--|------|---------|-------------|-------|
| 模式 | 简单 fork | leaf/orchestrator 角色 | Swarm 4 后端 | v1 + v2 两代 |
| 角色区分 | 无 | leaf/orchestrator | AgentDefinition | 无 (扁平) |
| 深度限制 | 无 | 有 | 无 | 有 (v2) |
| 通信 | 无 | 文件协调 + 凭证继承 | enqueue notification | Channel 通信 |
| 隔离 | 无 | 工作区隔离 | worktree | 无明确隔离 |
| 批量 | 无 | Kanban 任务板 | 无 | CSV 批量 spawn |

### 沙箱/安全
| | Kaku | Hermes | Claude Code | Codex |
|--|------|---------|-------------|-------|
| 沙箱 | 无 (终端内运行) | 无 | Bash sandbox + 网络控制 | bwrap/Landlock/Seatbelt/Windows |
| 审批 | Diff 预览 + glob | Hook 集成 | 权限系统 + auto-mode | Guardian Review + PermissionProfile |
| 身份认证 | API key | API key | API key | Ed25519 + JWT + Curve25519 |

## 各项目独有杀手功能

### Kaku 独有
1. **终端内嵌 AI** — 唯一将 AI Agent 直接嵌入终端模拟器的产品
2. **实时终端上下文注入** — 可读取当前 shell 会话、命令历史、文件系统状态
3. **Diff 预览审批** — 在终端内直接展示文件变更 diff，用户确认后执行
4. **Rust 高性能** — 所有项目中唯一的 Rust 原生终端 Agent
5. **SmartPrompt 模式** — 终端感知的智能提示
6. **零上下文切换** — AI 和终端在同一窗口，无需切换应用

### Hermes 独有
1. **PTC (Programmatic Tool Calling)** — LLM 写 Python 脚本调用工具，UDS RPC
2. **Kanban 多代理任务板** — SQLite 持久化、worker 所有权、幻觉检测
3. **Curator 自改进** — 空闲触发、技能生命周期、LLM 驱动合并
4. **18+ 平台适配器** — 覆盖 GitHub/GitLab/Jira/Linear/Notion 等
5. **StreamingContextScrubber** — 实时上下文清洗
6. **迭代摘要更新** — 摘要可随对话演进而非一次性生成

### Claude Code 独有
1. **6 层压缩管线** — 最成熟的上下文管理 (Snip/Micro/Collapse/Auto/Predictive/Reactive)
2. **28 Hook 事件** — 最完整的生命周期钩子系统
3. **Swarm 多代理 4 后端** — tmux/iTerm2/WinTerminal/InProcess
4. **Skills + Plugins 生态** — 400+ slash 命令，marketplace
5. **Dynamic Workflows** — 编排数十到数百个后台 Agent (official v2.1.154)
6. **Prompt Cache Stability** — 工具排序优化缓存命中
7. **Streaming Tool Execution** — 模型流式输出时即开始执行只读工具

### OpenAI Codex 独有
1. **多平台沙箱引擎** — bwrap/Landlock/Seatbelt/Windows 四平台隔离
2. **Agent Identity 密码学体系** — Ed25519 + JWT + Curve25519 加密任务注册
3. **Guardian Review** — 自主代码审查子代理系统
4. **App Server RPC** — 完整 IDE 集成协议 (~50+ RPC 方法)
5. **Agent Jobs CSV 批量** — 从 CSV 批量生成子代理
6. **Feature Flag 体系** — 50+ 特性开关控制功能发布
7. **Shell Snapshot** — 会话级 shell 环境快照注入沙箱
8. **Codex Apps** — OpenAI 托管的 MCP 服务集成

## Kaku 出圈路线图

### 核心差异化: "终端即 AI 工作站"
Kaku 的根本差异化不是"又一个 CLI Agent"，而是**唯一将 AI 深度嵌入终端的产品**。
出圈策略: 让终端成为 AI 的第一界面，而非 AI 的附属品。

### Phase G — 并行与安全 (2-3 周)
**目标**: 消除核心性能瓶颈，建立安全基线

| 任务 | 参考来源 | 预估 LOC | 优先级 |
|------|---------|----------|--------|
| 工具并行执行 (只读并行 + 写入串行) | Claude Code: Streaming Tool Execution | ~300 | P0 |
| 迭代上下文压缩 (摘要可更新 + 反抖动) | Hermes: 5阶段压缩 | ~200 | P0 |
| 工具循环防护 (death spiral prevention) | Claude Code: withheld errors | ~150 | P0 |
| Bash 沙箱 (网络隔离 + 路径控制) | Codex: bwrap/SandboxManager | ~500 | P1 |

### Phase H — 多代理与编排 (3-4 周)
**目标**: 从单 Agent 进化到多 Agent 协作

| 任务 | 参考来源 | 预估 LOC | 优先级 |
|------|---------|----------|--------|
| 子代理增强 (角色区分 + 深度限制 + 心跳) | Hermes: leaf/orchestrator | ~500 | P1 |
| 子代理间通信 (channel 或文件协调) | Codex: inter-agent communication | ~300 | P1 |
| 子代理沙箱隔离 (独立工作区 + 凭证继承) | Hermes: workspace isolation | ~400 | P1 |
| Plan 模式增强 (结构化计划 + 验证循环) | Codex: PlanHandler + delta | ~300 | P2 |

### Phase I — 生态与可扩展性 (4-5 周)
**目标**: 建立插件生态，降低第三方集成门槛

| 任务 | 参考来源 | 预估 LOC | 优先级 |
|------|---------|----------|--------|
| Hook 系统 (Pre/Post Tool + Session 事件) | Claude Code: 28 hooks | ~600 | P1 |
| MCP HTTP/SSE 传输 + OAuth | Hermes: MCP 增强 | ~400 | P1 |
| Skill/Plugin 系统 (目录格式 + 前置条件) | Claude Code: Skills + Plugins | ~500 | P2 |
| Plugin Marketplace 基础 | Claude Code official: marketplace | ~800 | P3 |

### Phase J — 终端独特优势 (持续)
**目标**: 放大"终端内嵌 AI"的独特优势，建立护城河

| 任务 | 描述 | 预估 LOC | 优先级 |
|------|------|----------|--------|
| 终端会话注入 (命令历史 + 环境变量 + cwd) | 将当前 shell 状态作为 Agent 上下文 | ~200 | P0 |
| 实时命令流监控 (Agent 可观察 shell 输出) | Agent 能"看到"终端正在发生什么 | ~400 | P1 |
| 内联 Diff 编辑器 (在终端直接编辑 AI 提议的变更) | 结合 WezTerm 的渲染能力 | ~600 | P1 |
| 终端上下文文件 (.terminal-context) | 自动捕获终端状态供 Agent 使用 | ~200 | P2 |
| 多 Tab AI 协作 (不同 Tab 的 Agent 共享上下文) | 利用 WezTerm 的多 Tab 能力 | ~500 | P2 |

### 出圈策略总结

**Kaku 不应该**:
- 和 Claude Code 比 Hook/Plugin 生态丰富度 (对手已有 400+ 命令)
- 和 Codex 比企业沙箱安全 (对手有 4 平台隔离 + 密码学身份)
- 和 Hermes 比工具数量 (对手有 75+ 工具 + 18 平台适配器)
- 做一个"又一个 CLI Agent"

**Kaku 应该**:
- **放大"终端即 AI 工作站"定位** — 这是所有竞品都没有的
- **让 Agent 拥有终端感知能力** — 读取 shell 状态、观察命令输出、理解终端布局
- **利用 WezTerm 的渲染能力** — 富文本 diff 预览、内联编辑、可视化计划
- **从终端出发构建 AI 体验** — 而非在终端中塞入一个独立 AI

**出圈路径**:
1. 先在 Rust 终端 Agent 领域建立技术领先 (Phase G)
2. 再展示终端内嵌 AI 的独特体验 (Phase J)
3. 然后建立生态 (Phase H-I)
4. 最终成为"AI 原生终端"品类定义者

## 相关记忆
[[kaku-agent-constraints]] — Kaku 当前架构约束
[[hermes-agent-analysis]] — Hermes 详细分析
[[claude-code-analysis]] — Claude Code fork 详细分析
