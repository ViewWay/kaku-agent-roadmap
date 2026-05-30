# Kaku Agent 待办跟踪

> 基于 AGENT_ROADMAP.md 6 Phase / 37 任务路线图
> 最后同步: 2026-05-30 (v0.15.0 发布后)

## Phase G1 — 核心性能 (v0.12.0, W1-2)

- [x] G1.0: 集成 WIP 工具模块 (git.rs, nav.rs, monitor.rs) ✅ v0.12.0
- [x] G1.1: 工具并行执行 (~250 LOC) ✅ v0.12.0 — std::thread::scope, 18只读并行
- [ ] G1.2: 流式工具执行 (~200 LOC) — 未实现
- [x] G1.3: 迭代压缩 + 反抖动 (~350 LOC) ✅ v0.12.0 — 30s冷却期, 4段模板
- [x] G1.4: Death Spiral Prevention (~120 LOC) ✅ v0.12.0 — 3次提醒, 5次终止
- [ ] G1.5: 工具结果缓存 (~150 LOC) — 未实现 (无 result_cache.rs)
- [ ] G1.6: 工具结果大小控制 (~150 LOC) — 未实现
- [ ] G1.7: 上下文预算系统 (~200 LOC) — 未实现 (无 budget.rs)
- [ ] G1.8: Prompt Cache Stability (~100 LOC) — 未实现

**Phase G1 完成度: 4/9 (44%)**

## Phase G2 — 安全基线 (v0.13.0, W2-3)

- [x] G2.0: Bash 沙箱 (~450 LOC) ✅ v0.12.0 — sandbox.rs, 网络拦截+路径防护
- [x] G2.1: 权限增强 (~250 LOC) ✅ v0.14.0 — permissions.json, Bash(git*) glob
- [ ] G2.2: 工具超时防护 (~80 LOC) — 未明确实现
- [x] G2.3: 终端上下文注入 前置版 (~200 LOC) ✅ v0.15.0 — terminal_context.rs

**Phase G2 完成度: 3/4 (75%)**

## Phase H — 子代理与编排 (v0.14.0, W3-5)

- [ ] H0: 集成 WIP 模块 + RMCP 迁移 (~430 LOC) — WIP 已集成; **RMCP 未迁移** (mcp.rs 仍为自建实现)
- [x] H1: 角色区分 + 深度限制 (~250 LOC) ✅ v0.13.0 — Leaf/Orchestrator, max_depth=3
- [x] H2: 子代理间通信 (~250 LOC) ✅ v0.13.0 — agent_send, inbox 机制
- [x] H3: Worktree 隔离增强 (~200 LOC) ✅ v0.13.0 — RAII Drop 自动清理
- [ ] H4: 子代理心跳与超时回收 (~150 LOC) — 未实现
- [ ] H5: 子代理 Transcript (~200 LOC) — 未实现 (无 subagent_transcript.rs)
- [ ] H6: 子代理结果聚合 (~200 LOC) — 未实现
- [x] ~~K3: Plan 模式增强~~ ✅ v0.13.0 — plan.rs 提前实现 (归属 Phase K 但已交付)

**Phase H 完成度: 4/7 (57%)** (H0 RMCP 部分未完成)

## Phase I — 扩展性 (v0.15.0, W5-7)

- [x] I1: Hook 系统 (~700 LOC) ✅ v0.14.0 — hooks.rs, 8事件 (计划22个, 已实现8个)
- [ ] I2: MCP 配置与运维层 (~300 LOC) — HTTP/SSE 已实现但用 reqwest::blocking, **非 RMCP SDK**
- [x] I3: Skill 系统 (~400 LOC) ✅ v0.14.0 — skills.rs, SKILL.md frontmatter
- [ ] I4: LSP 工具集成 (~300 LOC) — 未实现 (无 lsp_tool.rs)
- [ ] I5: 配置管理 (~200 LOC) — 部分实现 (permissions.json), 缺统一 settings.json

**Phase I 完成度: 2/5 (40%)** (I1 事件数未达标, I2 未用 RMCP)

## Phase J — 终端独特优势 (v0.16.0, W7-9)

- [x] J1: 终端会话上下文注入 增强版 (~250 LOC) ✅ v0.15.0 — cwd+命令+环境变量+进程
- [x] J2: 实时命令流监控 (~350 LOC) ✅ v0.15.0 — terminal_watch, 环形缓冲区
- [x] J3: 内联 Diff 编辑器 (~500 LOC) ✅ v0.15.0 — hunk级导航, 红绿高亮
- [ ] J4: 多 Tab AI 协作 (~500 LOC) — 未实现
- [ ] J5: Agent 通知系统 (~200 LOC) — 未实现
- [x] J6: 智能命令建议 (~300 LOC) ✅ v0.12.0 — suggestion.rs (提前实现)
- [ ] **决策节点 (W9)**: 终端方案评估 — 未到时间点

**Phase J 完成度: 4/6 (67%)**

## Phase K — 生态进阶 (v0.17.0, W9+)

- [ ] K1: Plugin 架构 (~500 LOC) — 未开始
- [ ] K2: Auto Mode 审批分类器 (~250 LOC) — 未开始
- [x] K3: Plan 模式增强 (~300 LOC) ✅ v0.13.0 — 已提前实现于 Phase H
- [ ] K4: 会话持久化 (~200 LOC) — 未开始
- [ ] K5: Feature Flag 体系 (~150 LOC) — 未开始
- [ ] K6: Dynamic Workflows (~600 LOC) — 未开始

**Phase K 完成度: 1/6 (17%)** (K3 已提前交付)

## 自研终端 MVP (terminal-agent-design.md, ~17800 LOC)

- [ ] PTY 管理 + 原始模式 I/O (~1500 LOC) — 未开始
- [ ] GPU 渲染引擎 wgpu + 字体 (~3000 LOC) — 未开始
- [ ] 多 Tab/Pane 管理 (~1500 LOC) — 未开始
- [ ] 键盘/鼠标输入 + Vi 模式 (~1000 LOC) — 未开始
- [ ] 基础 UI Tab 栏 + 状态栏 + 搜索 (~2000 LOC) — 未开始
- [ ] 配置系统 TOML 热加载 (~800 LOC) — 未开始
- [ ] AI Agent 引擎迁移 (~8000 LOC) — 未开始

**自研终端完成度: 0/7 (0%)**

## 多平台扩展 (multiplatform-design.md)

- [ ] kaku-cli Headless CLI (~500 LOC) — 未开始
- [ ] kaku-web Web App (~8000 LOC) — 未开始
- [ ] kaku-ide IDE 插件 (~3000 LOC) — 未开始
- [ ] kaku-mobile Mobile App (~6000 LOC) — 未开始
- [ ] 云端同步 SQLite + PostgreSQL (~2000 LOC) — 未开始

**多平台完成度: 0/5 (0%)**

## 总进度汇总

| Phase | 计划 | 完成 | 完成率 | 版本 |
|-------|------|------|--------|------|
| G1 核心性能 | 9 | 4 | 44% | v0.12.0 |
| G2 安全基线 | 4 | 3 | 75% | v0.12.0 |
| H 子代理编排 | 7 | 4 | 57% | v0.13.0 |
| I 扩展性 | 5 | 2 | 40% | v0.14.0 |
| J 终端优势 | 6 | 4 | 67% | v0.15.0 |
| K 生态进阶 | 6 | 1 | 17% | — |
| **Agent 引擎小计** | **37** | **18** | **49%** | |
| 自研终端 MVP | 7 | 0 | 0% | — |
| 多平台扩展 | 5 | 0 | 0% | — |
| **总计** | **49** | **18** | **37%** | |

## 关键偏差

| 偏差项 | 路线图要求 | 实际状态 | 影响 |
|--------|-----------|---------|------|
| RMCP 迁移 | H0: 删除 mcp.rs, 用 RMCP SDK | mcp.rs 仍存在, HTTP/SSE 用 reqwest::blocking | 无法消费 Streamable HTTP MCP Server |
| Hook 事件数 | I1: 22 个事件 | 仅实现 8 个 | 生命周期覆盖不足 |
| 工具结果缓存 | G1.5: LRU 缓存 50 条 | 未实现 | 重复读取浪费 token |
| 上下文预算 | G1.7: 动态 token 预算 | 未实现 (仍为固定轮次) | 无法精细控制 token 消耗 |
| 流式工具执行 | G1.2: 模型输出时启动 | 未实现 | 只读工具延迟执行 |

## 里程碑

| # | 里程碑 | 预计时间 | 状态 |
|---|--------|---------|------|
| M1 | G1 核心性能完成 | W2 | 🟡 部分完成 (4/9) |
| M2 | G2 安全基线完成 | W3 | 🟡 部分完成 (3/4) |
| M3 | H 子代理 + RMCP 迁移完成 | W5 | 🟡 部分完成 (4/7, RMCP 未迁) |
| M4 | I 扩展性完成 (Hook/Skill/MCP) | W7 | 🔴 未达标 (2/5) |
| M5 | J 终端独特优势 + 终端方案决策 | W9 | 🟡 部分完成 (4/6) |
| M6 | K 生态进阶完成 | W11 | 🔴 未开始 |
| M7 | 自研终端 MVP 可用 | W10 | 🔴 未开始 |
| M8 | 全平台 UI 覆盖 (Web/IDE/Mobile) | W16 | 🔴 未开始 |
| M9 | 商业化准备 | W18+ | 🔴 未开始 |

## v0.15.0 源文件清单 (kaku-gui/src/)

**ai_tools/**: fs.rs, git.rs, hooks.rs, mcp.rs, mcp_types.rs, mod.rs, monitor.rs, nav.rs, paths.rs, plan.rs, project.rs, registry.rs, sandbox.rs, search.rs, shell.rs, skills.rs, soul.rs, subagent.rs, tasks.rs, terminal_context.rs, web.rs, worktree.rs (22 files)

**ai_chat_engine/**: approval.rs, compact.rs, mod.rs, suggestion.rs, summarize.rs, title.rs (6 files)

**overlay/ai_chat/**: input.rs, layout.rs, markdown.rs, mod.rs, prompt_context.rs, render.rs, state.rs, strings.rs, syntax.rs, tests.rs, types.rs, waza.rs (12 files)

**测试**: 1201 tests passed, 0 failed (v0.15.0)
