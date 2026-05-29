---
name: kaku-agent-constraints
description: Kaku AI Agent 当前架构约束 — 引擎循环、工具分发、子代理、MCP、审批、客户端
metadata: 
  node_type: memory
  type: project
  originSessionId: b7d96830-1507-491a-aa53-f3ef1588c554
---

# Kaku Agent 架构约束

## 核心模块
- `ai_chat_engine/mod.rs` — 引擎主循环 (~1080 LOC)，25 轮上限，串行工具执行，同步审批阻塞
- `ai_tools/mod.rs` — 工具分发 (806 LOC)，34 内置工具硬编码 match，MCP 兜底
- `ai_tools/registry.rs` — 工具定义 (831 LOC)，手写 JSON schema，三档预算 (brief/default/full)
- `ai_tools/subagent.rs` — 子代理 (243 LOC)，简单 fork，无角色区分、无深度限制
- `ai_tools/mcp.rs` — MCP (304 LOC)，仅 stdio 传输，无 OAuth/采样/熔断
- `ai_tools/approval.rs` — 审批 (diff 预览 + auto-approve glob 匹配)
- `ai_client.rs` — API 客户端 (1504 LOC)，流式处理、错误重试
- `overlay/ai_chat/state.rs` — UI 状态 (2243 LOC)

## 通信
- mpsc channel (StreamMsg enum: AssistantStart/Token/Reasoning/RoundStart/ToolStart/ToolDone/ToolFailed/ApprovalRequired/Done/Err)
- 引擎在 OS 后台线程运行，UI 在主线程

## 上下文管理
- `micro_compact()` — 截断到 72KB
- `summarize_in_place()` — LLM 驱动摘要（72KB 阈值触发）
- 无迭代更新、无反抖动、无结构化模板

## 已实现功能 (Phase F)
- Diff 预览审批、auto-approve glob 匹配、/fork /export 命令、Plan 模式工具过滤

## 相关记忆
[[hermes-agent-analysis]] — Hermes 对比分析和移植建议
