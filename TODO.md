# Kaku Agent 待办跟踪

> 基于 AGENT_ROADMAP.md 6 Phase / 37 任务路线图

## Phase G1 — 核心性能 (v0.12.0, W1-2)

- [ ] G1.0: 集成 WIP 工具模块 (git.rs, nav.rs, monitor.rs)
- [ ] G1.1: 工具并行执行 (~250 LOC)
- [ ] G1.2: 流式工具执行 (~200 LOC)
- [ ] G1.3: 迭代压缩 + 反抖动 (~350 LOC)
- [ ] G1.4: Death Spiral Prevention (~120 LOC)
- [ ] G1.5: 工具结果缓存 (~150 LOC)
- [ ] G1.6: 工具结果大小控制 (~150 LOC)
- [ ] G1.7: 上下文预算系统 (~200 LOC)
- [ ] G1.8: Prompt Cache Stability (~100 LOC)

## Phase G2 — 安全基线 (v0.13.0, W2-3)

- [ ] G2.0: Bash 沙箱 (~450 LOC)
- [ ] G2.1: 权限增强 (~250 LOC)
- [ ] G2.2: 工具超时防护 (~80 LOC)
- [ ] G2.3: 终端上下文注入 前置版 (~200 LOC)

## Phase H — 子代理与编排 (v0.14.0, W3-5)

- [ ] H0: 集成 WIP 模块 + RMCP 迁移 (~430 LOC)
- [ ] H1: 角色区分 + 深度限制 (~250 LOC)
- [ ] H2: 子代理间通信 (~250 LOC)
- [ ] H3: Worktree 隔离增强 (~200 LOC)
- [ ] H4: 子代理心跳与超时回收 (~150 LOC)
- [ ] H5: 子代理 Transcript (~200 LOC)
- [ ] H6: 子代理结果聚合 (~200 LOC)

## Phase I — 扩展性 (v0.15.0, W5-7)

- [ ] I1: Hook 系统 (~700 LOC)
- [ ] I2: MCP 配置与运维层 (~300 LOC)
- [ ] I3: Skill 系统 (~400 LOC)
- [ ] I4: LSP 工具集成 (~300 LOC)
- [ ] I5: 配置管理 (~200 LOC)

## Phase J — 终端独特优势 (v0.16.0, W7-9)

- [ ] J1: 终端会话上下文注入 增强版 (~250 LOC)
- [ ] J2: 实时命令流监控 (~350 LOC)
- [ ] J3: 内联 Diff 编辑器 (~500 LOC)
- [ ] J4: 多 Tab AI 协作 (~500 LOC)
- [ ] J5: Agent 通知系统 (~200 LOC)
- [ ] J6: 智能命令建议 (~300 LOC)
- [ ] **决策节点 (W9)**: 终端方案评估 — 继续 WezTerm / Ghostty / 自研

## Phase K — 生态进阶 (v0.17.0, W9+)

- [ ] K1: Plugin 架构 (~500 LOC)
- [ ] K2: Auto Mode 审批分类器 (~250 LOC)
- [ ] K3: Plan 模式增强 (~300 LOC)
- [ ] K4: 会话持久化 (~200 LOC)
- [ ] K5: Feature Flag 体系 (~150 LOC)
- [ ] K6: Dynamic Workflows (~600 LOC)

## 自研终端 MVP (terminal-agent-design.md, ~17800 LOC)

- [ ] PTY 管理 + 原始模式 I/O (~1500 LOC)
- [ ] GPU 渲染引擎 wgpu + 字体 (~3000 LOC)
- [ ] 多 Tab/Pane 管理 (~1500 LOC)
- [ ] 键盘/鼠标输入 + Vi 模式 (~1000 LOC)
- [ ] 基础 UI Tab 栏 + 状态栏 + 搜索 (~2000 LOC)
- [ ] 配置系统 TOML 热加载 (~800 LOC)
- [ ] AI Agent 引擎迁移 (~8000 LOC)

## 多平台扩展 (multiplatform-design.md)

- [ ] kaku-cli Headless CLI (~500 LOC)
- [ ] kaku-web Web App (~8000 LOC)
- [ ] kaku-ide IDE 插件 (~3000 LOC)
- [ ] kaku-mobile Mobile App (~6000 LOC)
- [ ] 云端同步 SQLite + PostgreSQL (~2000 LOC)

## 里程碑

| # | 里程碑 | 预计时间 | 状态 |
|---|--------|---------|------|
| M1 | G1 核心性能完成 | W2 | 待开始 |
| M2 | G2 安全基线完成 | W3 | 待开始 |
| M3 | H 子代理 + RMCP 迁移完成 | W5 | 待开始 |
| M4 | I 扩展性完成 (Hook/Skill/MCP) | W7 | 待开始 |
| M5 | J 终端独特优势 + 终端方案决策 | W9 | 待开始 |
| M6 | K 生态进阶完成 | W11 | 待开始 |
| M7 | 自研终端 MVP 可用 | W10 | 待开始 |
| M8 | 全平台 UI 覆盖 (Web/IDE/Mobile) | W16 | 待开始 |
| M9 | 商业化准备 | W18+ | 待开始 |
