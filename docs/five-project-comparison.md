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

### Phase G1 — 核心性能 (W1-2)
**目标**: 消除核心性能瓶颈，从串行进化到智能并行

| 任务 | 参考来源 | 预估 LOC | 优先级 |
|------|---------|----------|--------|
| 工具并行执行 (三级安全分级 + 只读并行) | Claude Code: Streaming Tool Execution, Hermes: 三级安全分级 | ~250 | P0 |
| 流式工具执行 (模型输出时即开始执行) | Claude Code: Streaming Tool Execution | ~200 | P0 |
| 迭代上下文压缩 (12段模板 + 反抖动 + 迭代更新) | Hermes: 5阶段压缩 + 12段模板 | ~350 | P0 |
| 工具循环防护 (death spiral + 工具超时追踪) | Claude Code: withheld errors | ~120 | P0 |
| 工具结果缓存 (LRU + 文件mtime失效) | Claude Code: cache_deleted_input_tokens | ~150 | P0 |
| 工具结果大小控制 (per-tool + per-message 双层阈值) | Claude Code: 双层工具结果存储 | ~150 | P1 |
| 上下文预算系统 (动态 token 预算替代固定轮次) | Hermes: IterationBudget, Claude Code: per-tool 阈值 | ~200 | P1 |
| Prompt Cache Stability (工具排序优化缓存命中) | Claude Code: prompt cache stability | ~100 | P1 |

### Phase G2 — 安全基线 (W2-3)
**目标**: 建立沙箱安全、权限增强、工具超时，终端感知前置

| 任务 | 参考来源 | 预估 LOC | 优先级 |
|------|---------|----------|--------|
| Bash 沙箱 (网络隔离 + 路径控制 + 环境过滤 + 资源限制) | Codex: bwrap/SandboxManager, Claude Code: sandbox | ~450 | P0 |
| 权限增强 (glob + 三级策略 + 持久化 + 会话级) | Claude Code: 权限系统, Codex: PermissionProfile | ~250 | P0 |
| 工具超时防护 (per-tool 可配置超时) | 所有竞品标配 | ~80 | P0 |
| 终端上下文注入 (cwd + 命令历史 + 环境变量，前置版) | Kaku 独有, Codex: Shell Snapshot | ~200 | P0 |

### Phase H — 子代理与编排 (W3-5)
**目标**: 从单 Agent 进化到多 Agent 协作，完整生命周期

| 任务 | 参考来源 | 预估 LOC | 优先级 |
|------|---------|----------|--------|
| 子代理角色区分 + 深度限制 + 细粒度工具控制 | Hermes: leaf/orchestrator, Claude Code: AgentDefinition | ~250 | P1 |
| 子代理间通信 (channel + 持久化日志防丢消息) | Codex: inter-agent communication | ~250 | P1 |
| Worktree 隔离 (Drop guard + 失败回滚 + 环境继承) | Hermes: workspace isolation, Claude Code: worktree | ~200 | P1 |
| 子代理心跳与超时回收 | Hermes: 心跳机制, Claude Code: 孤儿进程处理 | ~150 | P0 |
| 子代理 Transcript (独立对话记录 + 压缩 + 导出) | Claude Code: 独立 transcript | ~200 | P1 |
| 子代理结果聚合 (排序 + 去重 + 失败标记) | 通用需求 | ~200 | P1 |

### Phase I — 扩展性 (W5-7)
**目标**: 建立可扩展架构，降低第三方集成门槛

| 任务 | 参考来源 | 预估 LOC | 优先级 |
|------|---------|----------|--------|
| Hook 系统 (22 事件 + 5 种类型 + 条件匹配) | Claude Code: 28 hooks + 6 类型 | ~700 | P1 |
| MCP 增强 (HTTP/SSE/WS + OAuth + 采样 + 熔断 + 命名空间 + 会话恢复) | Hermes: MCP 增强, Claude Code: MCP Elicitation | ~800 | P1 |
| Skill 系统 (目录格式 + 条件激活 + 工具过滤 + 动态加载) | Claude Code: Skills + Plugins | ~400 | P2 |
| LSP 工具集成 (桥接 WezTerm LSP → Agent 工具层) | Claude Code: LSP Tool | ~300 | P1 |
| 配置管理 (分层配置 + schema 验证) | Claude Code: settings.json | ~200 | P1 |

### Phase J — 终端独特优势 (W7-9)
**目标**: 放大"终端内嵌 AI"独特优势，建立护城河

| 任务 | 描述 | 预估 LOC | 优先级 |
|------|------|----------|--------|
| 终端会话上下文增强 (命令语义 + 进程状态 + 布局感知) | 基于 G2.3 前置版增强 | ~250 | P0 |
| 实时命令流监控 (自适应限流 + 结构化解析) | Agent 能"看到"终端正在发生什么 | ~350 | P1 |
| 内联 Diff 编辑器 (hunk 级操作 + 并行 diff + 统计) | 结合 WezTerm 的渲染能力 | ~500 | P1 |
| 多 Tab AI 协作 (跨 Tab 上下文共享 + 消息总线) | Kaku 独有，所有竞品无此能力 | ~500 | P0 |
| Agent 通知系统 (终端内通知 + 级别 + 历史队列) | 利用 WezTerm bell + 自定义通知栏 | ~200 | P1 |
| 智能命令建议 (历史模式 + 状态感知 + Tab 接受) | Agent 基于上下文建议下一步命令 | ~300 | P2 |

### Phase K — 生态进阶 (W9+)
**目标**: 建立 Plugin 生态，编排进阶，Auto Mode

| 任务 | 参考来源 | 预估 LOC | 优先级 |
|------|---------|----------|--------|
| Plugin 架构 (Skills + Hooks + MCP 可打包) | Claude Code: Plugin System | ~500 | P2 |
| Auto Mode 审批分类器 (风险分级 + 学习机制) | Claude Code: Auto Mode | ~250 | P2 |
| Plan 模式增强 (结构化步骤 + 逐条审批 + 验证循环) | Codex: PlanHandler + delta | ~300 | P2 |
| 会话持久化 (/resume + 快照 + 自动归档) | Claude Code: /resume | ~200 | P1 |
| Feature Flag 体系 (boolean + 灰度 + A/B) | Codex: 50+ Feature Flags | ~150 | P3 |
| Dynamic Workflows (YAML 编排 + DAG 可视化) | Claude Code: Dynamic Workflows, Codex: CSV Jobs | ~600 | P2 |

### 出圈策略总结

**Kaku 不应该**:
- 和 Claude Code 比 Hook/Plugin 生态丰富度 (对手已有 400+ 命令)
- 和 Codex 比企业沙箱安全 (对手有 4 平台隔离 + 密码学身份)
- 和 Hermes 比工具数量 (对手有 75+ 工具 + 18 平台适配器)
- 做一个"又一个 CLI Agent"
- 自研大模型 (调用智谱/OpenAI/Anthropic/Grok/Gemini/DeepSeek/OpenRouter/本地模型)

**Kaku 应该**:
- **放大"终端即 AI 工作站"定位** — 这是所有竞品都没有的
- **让 Agent 拥有终端感知能力** — 读取 shell 状态、观察命令输出、理解终端布局
- **自研终端渲染引擎** — GPU 加速 (wgpu)，富文本 diff 预览、内联编辑、可视化计划
- **全平台覆盖** — Terminal/Web/Mobile/IDE/CLI 共享 Agent 引擎
- **LLM 多 Provider** — 不绑定单一模型供应商，用户自由切换
- **从终端出发构建 AI 体验** — 而非在终端中塞入一个独立 AI

**出圈路径**:
1. 先在 Rust 终端 Agent 领域建立技术领先 (Phase G)
2. 再展示终端内嵌 AI 的独特体验 (Phase J)
3. 然后建立生态 (Phase H-I)
4. 扩展到全平台 (Web/Mobile/IDE)
5. 最终成为"AI 原生终端"品类定义者

## 相关记忆
[[kaku-agent-constraints]] — Kaku 当前架构约束
[[hermes-agent-analysis]] — Hermes 详细分析
[[claude-code-analysis]] — Claude Code fork 详细分析
