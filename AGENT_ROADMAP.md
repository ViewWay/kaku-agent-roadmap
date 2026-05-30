# Kaku Agent 开发路线图 — 详细版本规划

## Context

基于 5 个 AI Coding Agent 项目源码深度分析 (Kaku, Hermes, Claude Code v2.0.2 fork + v2.1.156 CHANGELOG, OpenAI Codex)，制定 Kaku Agent 引擎增强路线图。

当前版本 `0.11.0`，有 7 个未提交 WIP 工具模块。发布策略: 每 Phase 一个版本。总预估 37 任务, ~10700 LOC, 12+ 周。

### 相关设计文档

- [终端+Agent 工作流设计](docs/terminal-agent-design.md) — 自研终端架构、AI Agent 开发工作流交互设计、Phase 0 终端 MVP (~17800 LOC)
- [多平台产品架构](docs/multiplatform-design.md) — Core/Shell/Interface 三层分离、全平台 UI、LLM 多 Provider 策略

### 终端策略说明

本路线图的 Agent 引擎任务 (G1-K) 与终端选择解耦。当前基于 WezTerm 实现，J 阶段任务中的终端渲染/API 引用以 WezTerm 为例，W9 决策节点后可能迁移到 Ghostty 或自研终端。完整的自研终端设计见 [terminal-agent-design.md](docs/terminal-agent-design.md)，多平台架构见 [multiplatform-design.md](docs/multiplatform-design.md)。

---

## 三个战略决策

基于 2025-2026 市场调研 (Cursor $29B / Warp Oz / MCP 21K Server / RMCP SDK v1.7.0)，确立三个关键决策:

### 决策一: 终端方案 — 短期锁定, 中期评估

**现状风险**: WezTerm 超过 2 年未更新 (最后 release 2024-02-03)，作者定位为 "spare time project"。

| 方案 | 短期 (Phase G-H) | 中期 (Phase J 时点) |
|------|-----------------|-------------------|
| 继续 WezTerm | 锁定 API 边界层，不深度绑定渲染内部 | 评估迁移到 Ghostty (45K stars, 极高活跃度) 或自研轻量终端 (crossterm/ratatui, ~5-8K LOC) |
| 不推荐: Fork WezTerm | 15K 渲染引擎架构复杂，独自维护不可控 | — |

**决策节点**: Phase J 完成时 (W9)，根据产品-market fit 决定终端策略。

### 决策二: MCP 模块 — 分阶段增强，Core/Shell 分离时迁移 RMCP

**当前状态 (v0.15.0)**: 自建 mcp.rs 304 LOC + mcp_types.rs ~100 LOC，使用 reqwest::blocking 实现 stdio + HTTP/SSE，无 OAuth/Sampling。

**市场调研 (2026-05)**:
- MCP Server 19,388 个，177,436 工具，SDK 月下载 9700 万次
- 仅 8.5% MCP Server 使用 OAuth，52% Server 已废弃
- RMCP v1.7.0: Server 87.5%, Client 80% 合规 (Tier 2，非 Tier 1)
- 已知问题: reqwest 版本冲突 (#299), StreamableHTTP bug (#468)
- 替代 SDK: prism-mcp-rs, ultrafast-mcp, rust-mcp-sdk (均需 tokio)

**短期 (v0.16.0): 保持自建，补充关键能力**
- 在 mcp.rs 添加 OAuth 2.1 支持 (参考 MCP Authorization spec)
- 添加 Streamable HTTP 支持 (参考 MCP Transports spec 2025-11-25)
- 理由: 仅 8.5% Server 用 OAuth，52% 已废弃，短期迁移 ROI 低

**中期 (v0.17.0+): RMCP 迁移时机**
- 触发条件: RMCP 达到 Tier 1 合规 + reqwest 冲突修复
- 最佳窗口: Core/Shell 分离重构时 (多平台设计已规划引入 tokio)
- 届时 tokio 桥接成本最低，避免重复引入 async 运行时

**迁移范围 (中期执行)**:
- 删除: mcp.rs (304 LOC) + mcp_types.rs (~100 LOC) = -404 LOC
- 新增: MCP Client wrapper (~80 LOC) + MCP Server (~250 LOC) + tokio 桥接 (~100 LOC) = +430 LOC
- 净变化: +26 LOC

**双模架构**: 迁移后 Kaku 同时作为 MCP Client (消费 19K+ 外部 Server) 和 MCP Server (暴露 34 内置工具给其他 Agent)。

**路线图影响**: I2 (MCP 增强) 拆分为短期自建增强 + 中期 RMCP 迁移。

### 决策三: Warp Oz 互操作 — 轻量优先, 分阶段

**Oz 平台**: Warp 的云端 Agent 编排平台，HTTP REST API，支持 schedule/event/CI 触发，700K+ 开发者。

| 路径 | 方式 | 依赖 | 时机 |
|------|------|------|------|
| **路径 A (推荐): MCP Server 对接** | Kaku 作为 MCP Server，Oz 通过标准 MCP 调用 | RMCP 迁移完成后自动具备 | Phase H (RMCP 迁移后) |
| **路径 B: Oz CLI Agent** | 通过 `oz agent run` 提交任务到 Oz 编排器 | 需要 Oz 账号认证 | Phase K (评估 Oz 开放程度后) |
| **不做: 深度 Oz 集成** | 注册为 Oz harness，深度绑定 Warp 平台 | 失去独立性 | — |

**决策节点**: Phase K 时评估 Oz 平台开放程度，决定是否接入 Oz CLI。

---

## 版本映射

| 版本 | Phase | 主题 | 周期 |
|------|-------|------|------|
| **0.12.0** | G1 | 核心性能 | W1-2 |
| **0.13.0** | G2 | 安全基线 | W2-3 |
| **0.14.0** | H | 子代理与编排 | W3-5 |
| **0.15.0** | I | 扩展性 | W5-7 |
| **0.16.0** | J | 终端独特优势 | W7-9 |
| **0.17.0** | K | 生态进阶 | W9+ |

---

## 用户交互设计

### 通用交互规范

#### 快捷键体系

| 快捷键 | 上下文 | 功能 | 平台 |
|--------|--------|------|------|
| `Ctrl+K` | Kaku 终端内任意时刻 | 切换 AI 面板 (展开/收起) | 全平台 |
| `Esc` | AI 面板激活时 | 退出 AI 对话回到终端 (AI 进后台继续) | 全平台 |
| `Ctrl+Shift+Enter` | AI 输入框内 | 提交当前输入并执行 | 全平台 |
| `Enter` | AI 输入框内 | 换行 (多行输入) | 全平台 |
| `Tab` | 命令建议浮层 | 接受建议命令 | 全平台 |
| `Shift+Tab` | 命令建议浮层 | 下一条建议 | 全平台 |
| `Esc` | 命令建议浮层 | 忽略建议 | 全平台 |
| `↑/↓` | AI 输入框内 (空) | 浏览历史消息 | 全平台 |
| `j/k` | Diff 编辑器内 | 逐行导航 | 全平台 |
| `J/K` | Diff 编辑器内 | 按 hunk 跳转 | 全平台 |
| `g/G` | Diff 编辑器内 | 跳到首/尾 | 全平台 |
| `y` | Diff 编辑器/Plan 审批 | 接受当前 hunk/步骤 | 全平台 |
| `n` | Diff 编辑器/Plan 审批 | 拒绝当前 hunk/步骤 | 全平台 |
| `a` | Diff 编辑器/Plan 审批 | 全部接受 | 全平台 |
| `e` | Diff 编辑器/Plan 审批 | 编辑当前 hunk/步骤 | 全平台 |
| `Ctrl+/` | AI 面板内 | 唤起命令面板 (slash commands) | 全平台 |
| `Ctrl+L` | AI 面板内 | 清屏 | 全平台 |
| `Ctrl+C` | AI 执行中 | 中断当前操作 (保留已完成步骤) | 全平台 |
| `Ctrl+C` (2x) | AI 执行中 (快速双击) | 强制终止整个 Agent 任务 | 全平台 |
| `?` | AI 面板内 | 显示帮助面板 | 全平台 |

#### 全局快捷键 (vibecoding 入口)

| 平台 | 默认 | 备选 | 冲突检测 |
|------|------|------|----------|
| macOS | `Cmd+Shift+K` | `Cmd+\.` | Carbon/IOKit 扫描 |
| Windows | `Ctrl+Shift+K` | `Win+K` | RegisterHotKey 检测 |
| Linux X11 | `Ctrl+Shift+K` | `Super+K` | X11 GrabKey 检测 |
| Linux Wayland | `Ctrl+Shift+K` | `Super+K` | Wayland portal 检测 |

冲突检测逻辑: 启动时扫描已注册快捷键 → 冲突则日志警告 → 自动切换到备选 → 提示用户可在 `settings.json` 自定义。

#### 输入模式

| 模式 | 触发方式 | 行为 |
|------|----------|------|
| **对话模式** | 默认 | `Enter`=换行, `Ctrl+Shift+Enter`=提交。支持 `@` 引用、`/` 命令、`!` 终端直执行 |
| **命令模式** | `Ctrl+/` | 显示 slash 命令列表，可用 `↑↓` 选择，`Enter` 执行 |
| **Vi 模式** | 可选, `settings.json` → `input.vi_mode: true` | AI 面板内 `hjkl` 导航，`dd` 删行，`i` 插入，`Esc` 返回 normal |

#### @ 引用语法

| 语法 | 功能 | 示例 |
|------|------|------|
| `@file.rs` | 将文件内容注入上下文 | `@src/auth/handler.rs 这个文件有什么问题` |
| `@folder/` | 将目录结构注入上下文 | `@src/auth/ 审查这个模块` |
| `@git-diff` | 当前 git diff 注入上下文 | `@git-diff review 这些改动` |
| `@git-diff~3` | 最近 N 个 commit 的 diff | `@git-diff~3 总结最近改动` |
| `@terminal` | 当前终端输出注入上下文 | `@terminal 刚才 cargo build 失败了` |
| `@image.png` | 图片注入上下文 (多模态模型) | `@screenshot.png 这个 UI 有什么问题` |
| `@url` | 获取 URL 内容注入上下文 | `@https://... 参考这个文档实现` |

#### Slash 命令完整列表

| 命令 | 功能 | 参数 | Phase |
|------|------|------|-------|
| `/help` | 显示帮助面板 | 可选: 主题关键词 | G1 |
| `/clear` | 清除当前会话上下文 | — | G1 |
| `/compact` | 手动触发上下文压缩 | 可选: `--focus <topic>` | G1 |
| `/model` | 切换 LLM 模型 | 模型名称或别名 | G1 |
| `/resume` | 恢复上次会话 | 可选: session ID | K4 |
| `/plan` | 进入 Plan 模式 | 可选: 任务描述 | K3 |
| `/auto` | 切换 Auto Mode | 可选: `on/off/toggle` | K2 |
| `/settings` | 打开设置面板/编辑配置 | 可选: JSON 路径 | I5 |
| `/hooks` | 查看已加载 Hook 及状态 | — | I1 |
| `/mcp` | MCP 服务器管理 | 子命令: `list/add/remove/restart` | I2 |
| `/skills` | 查看已加载 Skills | 可选: `--reload` | I3 |
| `/agents` | 子代理状态面板 | 子命令: `list/kill/transcript` | H |
| `/export` | 导出会话为 Markdown | 可选: 输出路径 | K4 |
| `/budget` | 查看当前 token 预算 | 可选: `--set <N>` | G1 |
| `/permissions` | 查看/编辑权限规则 | 子命令: `list/add/remove` | G2 |

#### 审批流程 UI 规范

**单步审批** (工具执行需要确认时):

```
┌─────────────────────────────────────────────┐
│ ⚠ 工具: fs_write                             │
│ 目标: src/auth/sms_verify.rs (新建, +53 行)   │
│                                               │
│ [y] 接受  [n] 拒绝  [e] 编辑  [a] 后续全部接受│
│ [d] 查看完整 diff  [?] 帮助                   │
└─────────────────────────────────────────────┘
```

**批量审批** (Plan 模式下):

```
┌─────────────────────────────────────────────┐
│ 📋 Plan: 给用户登录添加手机号验证             │
│                                               │
│ [1] ✓ 新建 SMS 验证 service           已接受   │
│ [2] ○ User model 添加 phone 字段      待确认   │
│ [3] ○ 修改登录 handler                 待确认   │
│ [4] ○ 添加 SMS 配置                   待确认   │
│ [5] ○ 编写测试                         待确认   │
│                                               │
│ [y] 逐步审批  [a] 全部通过  [r] 拒绝全部      │
└─────────────────────────────────────────────┘
```

**权限记忆** (首次审批后):

```
┌─────────────────────────────────────────────┐
│ ⚠ Bash: npm install lodash                  │
│                                               │
│ [y] 接受  [n] 拒绝  [e] 编辑                 │
│ [s] 本次会话内自动通过  [p] 永久自动通过      │
└─────────────────────────────────────────────┘
```
- `s` (session): 写入 session 级权限规则，会话结束失效
- `p` (permanent): 写入 `permissions.json`，跨会话生效

#### 状态指示器体系

| Agent 状态 | 面板展示 | 状态栏格式 |
|------------|----------|-----------|
| IDLE | 半透明, 显示 token 统计和模型 | `[model: GLM-5.1] [tokens: 12K/200K] [session: 3m]` |
| THINKING | 脉冲动画, 流式文本逐字显示 | `[🧠 思考中...] [tokens: ━━━░ 65%]` |
| PLAN | 结构化步骤列表, 逐条高亮 | `[📋 Plan 模式] [Step 2/5] [y/n/e/a]` |
| EXECUTING | 当前工具名 + 进度 + 并行数 | `[🔧 fs_read] [并行: 3] [Step 2/5] [tokens: ━━░]` |
| APPROVAL | Diff 预览 + 操作按钮高亮 | `[⏳ 等待审批] [Step 2/5] [Esc 跳过]` |
| ERROR | 红色 banner + 重试选项 | `[❌ LLM 错误 3/5] [r 重试] [m 切换模型]` |
| SUGGESTING | 底部浮层, 灰色建议命令 | `[💡 建议命令] [Tab 接受] [Esc 忽略]` |
| COMPACTING | 蓝色进度条, 压缩中提示 | `[🔄 压缩上下文...] [保留: goals, active_files]` |

#### 错误展示规范

| 错误类型 | 展示样式 | 用户可选操作 |
|----------|----------|-------------|
| LLM API 错误 | 🔴 红色 banner + 错误码 + 建议原因 | `[r] 重试 [m] 切换模型 [s] 跳过` |
| 工具超时 | 🟡 黄色提示 + 已用时间 + 工具名 | `[r] 重试 [c] 取消 [i] 忽略此工具` |
| 工具执行失败 | 🟠 橙色提示 + 错误输出摘要 (首 500 字) | `[r] 重试 [s] 跳过 [v] 查看详细输出` |
| 沙箱拦截 | 🔴 红色 banner + 被拦截命令 + 拦截原因 | `[e] 编辑命令 [s] 跳过 [a] 授权一次` |
| 权限拒绝 | ⚪ 灰色提示 + 拒绝规则匹配 | `[o] 修改权限 [s] 跳过` |
| 上下文预算耗尽 | 🔵 蓝色提示 + 压缩摘要 + 会话将终止 | `[c] 强制压缩继续 [e] 导出会话` |
| Death Spiral 触发 | 🔴 闪烁 banner + 连续错误计数 + 降级建议 | `[c] 继续 (已降级) [m] 切换模型 [x] 终止` |
| MCP 连接失败 | 🟡 黄色提示 + server 名 + 失败原因 | `[r] 重连 [d] 禁用此 server [i] 忽略` |
| 子代理异常 | 🟠 橙色提示 + agent ID + 错误摘要 | `[v] 查看 transcript [r] 重启 agent [k] 终止` |
| Hook 执行失败 | 🟡 黄色提示 + hook 名 + 错误输出 | `[s] 跳过 [r] 重试 [d] 禁用此 hook` |

#### 配置入口体系

| 入口 | 路径 | 用途 | 持久化 |
|------|------|------|--------|
| 全局设置 | `~/.kaku/settings.json` | 模型、权限、Hook、通用偏好 | 永久 |
| 项目设置 | `.kaku/settings.json` | 项目级 MCP、Skills、权限覆盖 | 永久, git 可跟踪 |
| 权限规则 | `.kaku/permissions.json` | 工具执行权限 glob 规则 | 永久 |
| Hook 配置 | `.kaku/hooks.json` | Hook 事件绑定 (22 事件 × 5 类型) | 永久 |
| MCP 服务器 | `.kaku/mcp_servers.json` | MCP Server 声明式配置 | 永久 |
| 快捷键 | `settings.json` → `shortcuts` 字段 | 全局/终端内快捷键自定义 | 永久 |
| 会话内临时 | `/settings` 命令 | 临时修改 (退出后失效) | 会话级 |
| Feature Flags | `.kaku/feature_flags.json` | 功能开关 (boolean / percentage / user_list) | 永久 |

---

## v0.12.0 — 核心性能 (Phase G1, W1-2)

> 目标: 消除核心性能瓶颈，从串行单线程进化到智能并行。

### G1.0: 集成 WIP 工具模块 (前置)

已有 7 个未提交文件需集成到主分发: `git.rs`(286行), `nav.rs`(312行), `monitor.rs`(73行)

**文件**: `ai_tools/mod.rs` (内联改为调用模块), `ai_tools/registry.rs` (确认 ToolDef)
**验证**: `make check` + `make test` 通过

### G1.1: 工具并行执行 (~250 LOC)

**文件**: `ai_tools/registry.rs` (ToolDef 添加 `parallel_safety: ParallelSafety`), `ai_chat_engine/mod.rs` (分发重构), `overlay/ai_chat/render.rs` (batch 展示)

```rust
enum ParallelSafety { ReadOnly, PathScoped, Writable }
```

- **ReadOnly** (18 工具): fs_read, fs_list, pwd, shell_poll, web_fetch, web_search, read_url, project_summary, file_tree, project_detect, symbol_search, grep_search, memory_read, soul_read, git_status, git_diff, git_log, git_show, git_branch(list/current), dir_overview, disk_usage, worktree_list, worktree_diff, system_stats, task_list, task_get, agent_poll → `futures::join_all` 并行
- **Writable** (16 工具): fs_write, fs_patch, fs_mkdir, fs_delete, shell_exec, shell_bg, git_commit, git_branch(create/delete), dir_jump, dir_bookmark(save/remove), worktree_create, worktree_remove, task_create, task_update, agent_spawn → 按序串行
- **PathScoped** (预留): 写入但路径受限的工具，同类路径可并行

参考: Claude Code (只读并行 max10 + 写入串行), Hermes (NEVER_PARALLEL / PARALLEL_SAFE / PATH_SCOPED 三级)

**验证**: AI 同时读取 3 文件确认并行 (log 时间戳)

### G1.2: 流式工具执行 (~200 LOC) [新增]

**文件**: `ai_chat_engine/mod.rs`, `ai_client.rs`

**方案**: 模型流式输出时，一旦识别出完整工具调用 JSON，立即启动执行。不需要等模型完全停止生成。
- 解析器在 SSE 流中增量匹配 tool_use 块
- 完整块到达后立即提交到并行执行器
- 写入工具仍等待模型输出完成

参考: Claude Code Streaming Tool Execution

**验证**: 模型输出 3 个只读工具调用，第一个在模型输出完成前就开始执行

### G1.3: 迭代压缩 + 反抖动 (~350 LOC)

**文件**: `ai_chat_engine/mod.rs`, `ai_chat_engine/compact.rs`

**方案**:
- 30s 冷却期反抖动 (Hermes 有，当前 Kaku 无)
- 保留上次摘要 diff 增量追加 (迭代更新，非一次性生成)
- 结构化模板 12 段: `[goals|decisions|pending|completed|active_files|recent_changes|errors|constraints|user_preferences|context_summary|token_budget|open_questions]`
- focus_topic: 摘要时保留当前任务焦点，避免泛化

参考: Hermes 5 阶段压缩 + 12 段结构化模板 + 迭代摘要更新 + 反抖动

**验证**: 30 轮对话，摘要可读，间隔 >= 30s，焦点不丢失

### G1.4: Death Spiral Prevention (~120 LOC)

**文件**: `ai_chat_engine/mod.rs`

**方案**:
- 连续 3 次 LLM 错误 → 注入策略提醒 (降级 prompt 或切换模型)
- 连续 5 次 LLM 错误 → 强制终止
- 连续 3 次工具超时 → 标记可疑工具，后续自动跳过
- 工具执行错误不计入 (仅 LLM 返回错误和超时)

参考: Claude Code withheld errors + fallback model

**验证**: 模拟连续参数错误，3 次注入，5 次终止

### G1.5: 工具结果缓存 (~150 LOC) [新增]

**文件**: `ai_tools/registry.rs`, 新建 `ai_tools/result_cache.rs`

**方案**:
- LRU 缓存 (容量 50)，key = (tool_name, params_hash, file_mtime)
- 命中时返回 `[Content cleared, result cached]` 占位符 + 缓存结果
- 文件读取类工具自动利用 (fs_read, symbol_search, grep_search)
- 写入工具自动失效相关缓存

参考: Claude Code `cache_deleted_input_tokens` 优化

**验证**: 同一文件读取两次，第二次从缓存返回，token 消耗降低

### G1.6: 工具结果大小控制 (~150 LOC) [新增]

**文件**: `ai_tools/registry.rs`, `ai_chat_engine/mod.rs`

**方案**:
- per-tool 阈值: 单个工具结果 > 10KB 截断为摘要 (保留首尾各 2KB + 中间省略)
- per-message 聚合预算: 单轮所有工具结果总和 > 50KB 触发压缩
- 大结果标记为 `[truncated, {original_size} bytes]`

参考: Claude Code 双层工具结果存储

**验证**: 工具返回 100KB 日志，截断为摘要，后续轮次引用不重复展开

### G1.7: 上下文预算系统 (~200 LOC) [新增]

**文件**: `ai_chat_engine/mod.rs`, `ai_chat_engine/budget.rs`

**方案**:
- 替换固定 25 轮硬上限为动态 token 预算 (默认 200K tokens)
- per-tool 预算: 读取类 2K，写入类 5K，搜索类 10K
- 每轮开始检查剩余预算，不足时主动触发压缩
- 预算耗尽 → 摘要 + 终止，而非硬截断

参考: Hermes IterationBudget (可退款机制), Claude Code per-tool 持久化阈值

**验证**: 高消耗会话（多文件搜索 + 大文件读取），预算用尽时优雅结束

### G1.8: Prompt Cache Stability (~100 LOC) [新增]

**文件**: `ai_tools/registry.rs`, `ai_chat_engine/mod.rs`

**方案**:
- 工具 schema 排序: 按调用频率降序排列，高频工具 schema 构成连续前缀
- MCP 工具追加到末尾 (避免打破内置工具的缓存前缀)
- 会话间保持排序一致性

参考: Claude Code prompt cache stability

**验证**: 连续两轮调用，prompt 前缀 byte 完全相同

### G1 交互细节

#### G1.1 工具并行执行 — 用户感知

| 场景 | 用户看到的 | 用户可操作的 |
|------|-----------|-------------|
| Agent 并行读取 3 文件 | 状态栏: `[🔧 并行: 3] [fs_read, grep, symbol]` + 3 个进度指示同时推进 | 等待，或 `Ctrl+C` 中断 |
| 并行完成 | 工具结果逐个显示 (按完成顺序)，非等待全部 | — |
| 写入串行等待 | 状态栏: `[🔧 等待写入锁] [fs_write 排队中]` | 可查看队列 |
| PathScoped 冲突 | 黄色提示: `fs_write(/src/a.rs) 等待 fs_write(/src/a.rs) 完成` | — |

#### G1.2 流式工具执行 — 用户感知

- 用户看到: 模型仍在输出文本时，工具栏已有 `[🔧 fs_read: 执行中]` 提前出现
- 状态栏显示流式进度: `[🧠 输出中 + 🔧 2 工具已提交]`
- 第一个只读工具结果返回后立即显示，不等模型输出完成

#### G1.3 迭代压缩 — 用户感知

| 触发条件 | 用户看到的 |
|----------|-----------|
| 上下文超预算 80% | 自动触发: `[🔄 正在压缩上下文...]` 蓝色进度条 → `[✅ 压缩完成] 200K → 80K, 保留焦点: SMS 验证` |
| 30s 冷却期后 | 自动触发 (反抖动): 同上，但避免频繁压缩 |
| 手动 `/compact` | 同上，支持 `--focus <topic>` 指定保留焦点 |

压缩摘要结构 (12 段): `[goals|decisions|pending|completed|active_files|recent_changes|errors|constraints|user_preferences|context_summary|token_budget|open_questions]`，用户可通过 `/budget --verbose` 查看各段大小。

#### G1.4 Death Spiral — 用户感知

| 连续错误次数 | 用户看到的 | 系统行为 |
|-------------|-----------|---------|
| 1-2 次 | 🔴 单条错误 banner `[r] 重试` | 正常重试 |
| 3 次 | 🔴 banner `连续 3 次 LLM 错误 — 已注入降级策略` + 蓝色提示 | 注入策略提醒, 简化后续 prompt |
| 4 次 | 🔴 闪烁 `⚠ 即将终止 (1 次机会剩余)` | 最后一次尝试 |
| 5 次 | 🔴 大 banner `❌ Agent 任务终止 — 连续 5 次 LLM 错误` | 强制终止, 保留已完成步骤 |

用户操作: `[m] 手动切换模型 [r] 重试 [x] 提前终止`

#### G1.5 工具结果缓存 — 用户感知

- 缓存命中: 工具结果标注 `[cache hit]`，状态栏 `[✅ 缓存命中: fs_read(handler.rs) 节省 2K tokens]`
- 写入操作自动失效相关缓存: 写入 `a.rs` 后，`a.rs` 的缓存条目自动清除
- `/budget` 命令可查看缓存命中率统计

#### G1.6 工具结果大小控制 — 用户感知

| 场景 | 展示 |
|------|------|
| 单工具 > 10KB | `[truncated, 32KB → 4KB]` + 首尾各 2KB + `...省略 {N} 行...` |
| 单轮 > 50KB | `[results compressed, 67KB → 12KB]` |
| 需要完整结果 | 点击截断部分 → 展开完整内容 (仅此次，不持久化) |

#### G1.7 上下文预算系统 — 用户感知

| 预算状态 | 状态栏展示 |
|----------|-----------|
| 正常 (0-80%) | `[tokens: 45K/200K ████████░░░░░░░░ 22%]` |
| 警告 (80-95%) | `[tokens: 165K/200K ████████████████░ 82% ⚠]` |
| 不足 (95%+) | `[tokens: 190K/200K █████████████████░ 95% 🔵]` → 下一轮自动压缩 |
| 耗尽 | `[❌ 预算耗尽]` → `[c] 继续压缩 [e] 导出会话` |

命令: `/budget` 查看分配; `/budget --set 300K` 调整; `/budget --verbose` 含 cache 命中率

#### G1.8 Prompt Cache — 用户感知

- 用户不可见 (后台优化)
- `/budget --verbose` 查看: `[prompt cache: 92% hit, 节省 15K tokens]`

---

## v0.13.0 — 安全基线 (Phase G2, W2-3)

> 目标: 建立沙箱安全、权限增强、工具超时，形成安全基线。

### G2.0: Bash 沙箱 (~450 LOC)

**文件**: 新建 `ai_tools/sandbox.rs`, 修改 `ai_tools/shell.rs`

**方案**:
- **网络隔离**: 拦截 curl/wget/nc/ssh 等网络命令 (应用层拦截，非 OS 沙箱)
- **危险路径拦截**: `rm -rf /`, 任何修改 `/etc`、`/usr`、`/System`、`~/` 根目录的操作
- **写入白名单**: 默认只允许 cwd 子目录写入，例外: `/tmp/kaku-*`, `$XDG_CONFIG_HOME`
- **环境变量过滤**: 注入命令前移除 `__CF_*`, `AWS_*`, `GITHUB_TOKEN` 等敏感变量 (除非显式授权)
- **进程资源限制**: CPU 时间 30s, 内存 512MB, 输出 1MB (ulimit wrapper)
- **macOS 适配**: 可选 `sandbox-exec` 集成 (profile: `kaku-agent.sb`)

参考: Codex (bwrap/Landlock/Seatbelt 四平台), Claude Code Bash sandbox + 网络控制

**验证**: `rm -rf /` 拦截, `curl` 拦截, `touch ./newfile` 通过, `echo $GITHUB_TOKEN` 空输出

### G2.1: 权限增强 (~250 LOC)

**文件**: `ai_tools/approval.rs`, 新建 `.claude/permissions.json`

**方案**:
- glob 匹配: `Bash(git *)`, `Write(src/**/*.rs)`, `Read(.env*)`
- 三级策略: `allow` (自动通过), `deny` (自动拒绝), `ask` (弹审批)
- 持久化到 `.claude/permissions.json`
- 工具级别默认策略: 只读工具默认 allow, 写入工具默认 ask
- 会话级临时规则: 本次会话有效，不持久化

参考: Claude Code 权限系统 + auto-mode, Codex PermissionProfile

**验证**: `Bash(git *)` 规则, git 不弹审批, `rm` 仍弹

### G2.2: 工具超时防护 (~80 LOC) [新增]

**文件**: `ai_tools/registry.rs`, `ai_chat_engine/mod.rs`

**方案**:
- 每个工具默认超时: 读取 10s, 写入 30s, 搜索 15s, shell 120s
- 可在 ToolDef 中自定义
- 超时后强制终止进程，返回 `[Tool timed out after {N}s]`
- 超时不计入 death spiral 计数 (G1.4 独立追踪)

**验证**: 工具挂起 10s 后自动终止，agent 继续运行

### G2.3: 终端上下文注入 (前置版) (~200 LOC) [从 J 前置]

**文件**: 新建 `ai_tools/terminal_context.rs`, 修改 `ai_chat_engine/mod.rs`

**方案**:
- 注入到 system message: cwd + 最近 20 条命令 + 关键环境变量 (PATH, PWD, SHELL, HOME)
- 每轮刷新 (不缓存，命令历史会变)
- 命令标记成功/失败状态 (exit code)
- **前置理由**: 让"终端感知"从一开始就成为产品的一部分，而非 8 周后的附加功能

参考: Kaku 独有差异化: 终端内嵌 AI, Codex Shell Snapshot

**验证**: Agent 自动知道当前目录和最近命令的执行结果

### G2 交互细节

#### G2.0 Bash 沙箱 — 用户感知

| 拦截场景 | 用户看到的 | 可选操作 |
|----------|-----------|---------|
| 网络命令 (`curl`, `wget`) | 🔴 `⛔ 沙箱拦截: curl 被禁止 (网络隔离策略)` | `[e] 编辑命令 [s] 跳过 [a] 临时授权网络` |
| 危险路径 (`rm -rf /`, 修改 `/etc`) | 🔴 `⛔ 沙箱拦截: 不允许修改 /etc/ (路径防护)` | `[e] 编辑 [s] 跳过` |
| 敏感环境变量泄露 (`echo $GITHUB_TOKEN`) | 🔴 `⛔ 沙箱拦截: 环境变量 GITHUB_TOKEN 已过滤` | 命令正常执行但变量值为空 |
| 正常写入 cwd | ✅ 正常执行，无拦截 | — |
| 超出资源限制 (CPU 30s / 内存 512MB) | 🟡 `⚠ 进程超限: CPU 时间 30s 已到` → 自动终止 | `[r] 放宽限制 [s] 跳过` |

沙箱配置 (`settings.json`):
```json
{
  "sandbox": {
    "network": "deny",
    "allowed_paths": ["./", "/tmp/kaku-*"],
    "blocked_paths": ["/etc", "/usr", "/System", "~"],
    "env_filter": ["GITHUB_TOKEN", "AWS_*", "__CF_*"],
    "resource_limits": { "cpu_seconds": 30, "memory_mb": 512, "output_mb": 1 }
  }
}
```

#### G2.1 权限增强 — 用户感知

**权限规则配置** (`.kaku/permissions.json`):
```json
{
  "rules": [
    { "pattern": "Bash(git *)",       "action": "allow" },
    { "pattern": "Write(src/**/*.rs)", "action": "allow" },
    { "pattern": "Write(.env*)",       "action": "deny" },
    { "pattern": "Bash(rm *)",         "action": "ask" },
    { "pattern": "Bash(cargo *)",      "action": "ask" },
    { "pattern": "Read(*)",            "action": "allow" }
  ]
}
```

**权限管理命令**:
| 命令 | 功能 |
|------|------|
| `/permissions list` | 显示所有规则，按优先级排序 |
| `/permissions add "Bash(npm *)"` | 添加新规则 (交互式选择 action) |
| `/permissions remove 3` | 按编号删除规则 |
| `/permissions test "Bash(git push)"` | 测试规则匹配结果 |

**审批时的权限匹配提示**:
```
⚠ Bash: rm -rf ./target/
匹配规则: Bash(rm *) → action: ask
[y] 接受  [n] 拒绝  [s] 本次会话自动通过  [p] 永久自动通过
```

#### G2.2 工具超时 — 用户感知

| 工具类型 | 默认超时 | 超时展示 |
|----------|---------|---------|
| 读取 (fs_read, grep) | 10s | 🟡 `[⏱ fs_read 超时 (10s)] [r] 重试 [c] 跳过` |
| 写入 (fs_write, shell) | 30s | 🟡 `[⏱ fs_write 超时 (30s)] [r] 重试 [c] 跳过` |
| 搜索 (symbol, web) | 15s | 🟡 `[⏱ web_search 超时 (15s)] [r] 重试 [c] 跳过` |
| Shell 执行 | 120s | 🟡 `[⏱ shell_exec 超时 (120s)] [r] 重试 [c] 跳过 [e] 延长` |

超时后工具被强制终止，不计入 Death Spiral 计数。用户可 `[e] 延长超时` (输入新的秒数)。

#### G2.3 终端上下文注入 — 用户感知

注入到 system message 的内容:
```
┌─── 终端上下文 (每轮自动注入) ─────────────────────────┐
│                                                         │
│  📂 cwd: ~/project/src/auth                            │
│  🔀 branch: feat/sms-verify (3 commits ahead)           │
│  📁 dirty: [user_model.rs, handler.rs]                  │
│                                                         │
│  🖥 最近命令:                                           │
│    ✓ (0) cargo build          — 成功 (2.3s)            │
│    ✓ (0) cargo test --auth    — 成功, 5/5 通过          │
│    ✗ (1) cargo test           — 失败, auth::test_sms   │
│    ⏱ (130) vim handler.rs    — 超时 (vim 仍在运行)      │
│                                                         │
│  🔧 环境变量: PATH, PWD, SHELL, HOME, RUSTUP_HOME      │
│                                                         │
│  💡 前台进程: vim (PID 48291)                           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

用户可通过 `settings.json` → `terminal_context.max_commands: 20` 调整注入的命令条数。

---

## v0.14.0 — 子代理与编排 (Phase H, W3-5)

> 目标: 从单 Agent 进化到多 Agent 协作，建立编排基础。

### H0: 集成 WIP 模块 + RMCP 迁移 (前置)

集成 `subagent.rs`(243行), `tasks.rs`(261行), `worktree.rs`(97行)。
**不集成 mcp.rs** — 改为迁移到官方 RMCP SDK (v1.7.0)。

**RMCP 迁移** (~430 LOC 新增 / 404 LOC 删除):
- 删除: mcp.rs (304 LOC), mcp_types.rs (~100 LOC) — 自定义 JSON-RPC 实现已过时
- 新增: `ai_tools/mcp_client.rs` (~80 LOC) — RMCP Client wrapper，支持 stdio + Streamable HTTP
- 新增: `ai_tools/mcp_server.rs` (~250 LOC) — RMCP Server，暴露 34 内置工具给外部 Agent
- 新增: `ai_tools/tokio_bridge.rs` (~100 LOC) — tokio runtime 桥接层 (Kaku 主架构保持 OS 线程 + mpsc)
- Cargo.toml: 添加 `rmcp` (git dep, branch=main), `tokio` (features=["sync","macros","rt","time"])

**双模架构**: 迁移后 Kaku 同时是 MCP Client (消费 21K+ 外部 Server) 和 MCP Server (暴露工具)。

集成时一并设计 H1-H3 的接口，避免集成后再改。

参考: [RMCP Rust SDK](https://github.com/modelcontextprotocol/rust-sdk) (v1.7.0, 3,468 stars), [Building MCP Servers in Rust](https://rup12.net/posts/write-your-mcps-in-rust/)

**验证**: agent_spawn/task_create/worktree_create 可调用; 通过 MCP 连接外部 Server 并调用工具; 作为 MCP Server 被外部 Client 发现和调用

### H1: 角色区分 + 深度限制 (~250 LOC)

**文件**: `ai_tools/subagent.rs`

**方案**:
```rust
enum AgentRole { Leaf, Orchestrator }
struct SubagentConfig {
    role: AgentRole,
    max_depth: u8,        // 默认 3
    allowed_tools: Vec<String>,  // 细粒度工具控制
    timeout: Duration,     // 默认 120s
    heartbeat_interval: Duration,
}
```
- Leaf 工具集: 去掉 agent_spawn, task_create, task_update
- Orchestrator 工具集: 保留 agent_spawn + task_*, agent_send
- 深度限制: `max_depth: 3`，超限拒绝创建
- 自定义角色: 用户可指定允许的工具集

参考: Hermes leaf/orchestrator, Claude Code AgentDefinition

**验证**: orchestrator→leaf 通过, leaf→spawn 拒绝, depth=3 时拒绝第四层

### H2: 子代理间通信 (~250 LOC)

**文件**: 新建 `ai_tools/agent_comm.rs`, 修改 `subagent.rs`, `registry.rs`

**方案**:
- 传输: crossbeam channel (内存内) + 持久化日志 (防止断连丢消息)
- `agent_send(id, message)` 发送, `agent_messages()` 接收
- 消息格式: JSON `{from, to, type, payload, timestamp}`
- 消息类型: `info`, `result`, `request`, `error`
- 接收缓冲: 最多 100 条，溢出丢弃最旧

参考: Codex Channel 通信, Hermes 文件协调

**验证**: 两个 leaf 通过消息交换文件路径，一个崩溃后另一个仍能收到之前的消息

### H3: Worktree 隔离增强 (~200 LOC)

**文件**: `ai_tools/worktree.rs`, `ai_tools/subagent.rs`

**方案**:
- 子代理默认在 git worktree 运行
- Drop guard 自动清理: 子代理退出后 worktree 自动删除
- 失败回滚: 写入操作失败时自动 `git checkout .`
- 环境继承: 从父代理继承 PATH/PWD/SHELL，过滤敏感变量 (G2.0 策略)
- 凭证继承: 可选，默认不继承 (安全优先)

参考: Hermes workspace isolation + 凭证继承, Claude Code worktree 隔离

**验证**: 子代理 worktree 修改文件，清理后父代理不受影响，敏感环境变量未泄露

### H4: 子代理心跳与超时回收 (~150 LOC) [新增]

**文件**: `ai_tools/subagent.rs`

**方案**:
- 心跳间隔: 每 10s 发送心跳到父代理
- 超时检测: 3 次心跳未收到 → 标记为可能死亡
- 超时回收: 5 次心跳未收到 → 强制终止 + 日志记录
- 资源清理: 终止时自动清理 worktree + 临时文件

参考: Hermes 心跳机制, Claude Code 子代理孤儿进程处理

**验证**: 子代理挂起，5x 心跳后自动回收，资源已清理

### H5: 子代理 Transcript (~200 LOC) [新增]

**文件**: 新建 `ai_tools/subagent_transcript.rs`, 修改 `subagent.rs`

**方案**:
- 每个子代理独立记录完整对话历史 (prompt + tools + response)
- 父代理可通过 `agent_transcript(id)` 查询
- Transcript 有 token 上限 (默认 50K)，超限自动压缩
- 支持导出为 Markdown (用于审计和调试)

参考: Claude Code 独立 transcript

**验证**: 子代理执行 5 轮后，父代理获取完整 transcript

### H6: 子代理结果聚合 (~200 LOC) [新增]

**文件**: 修改 `subagent.rs`, `ai_chat_engine/mod.rs`

**方案**:
- 多子代理结果汇总: 按相关性排序，去重
- 结构化返回: `{results: [{agent_id, summary, artifacts, token_cost}], deduped: bool}`
- 失败子代理标记: `{status: "failed", error, partial_results}`
- 父代理可选择: 取最佳结果 / 合并所有结果 / 重新分配

**验证**: 3 个 leaf 执行同一文件的不同修改，父代理收到聚合结果

### H 交互细节

#### 子代理生命周期 — 用户交互全流程

```
┌─── 子代理完整交互流程 ──────────────────────────────────────────┐
│                                                                 │
│  1. 创建 (Agent 自动或用户手动)                                  │
│     用户看到: [🤖 子代理 #3 已创建 — 角色: leaf, 工具: 12个]     │
│     /agents list → 查看所有子代理状态                             │
│                                                                 │
│  2. 执行中                                                     │
│     状态栏: [🤖 #3: 🔧 fs_read(mod.rs) | tokens: 15K | ❤️ OK]  │
│     心跳: 每 10s 刷新状态, 3 次心跳丢失 → 🟡 可疑, 5 次 → 终止   │
│                                                                 │
│  3. 消息交互                                                   │
│     父代理想发送消息: agent_send(3, "检查这个文件的边界情况")     │
│     子代理收到: 📨 显示在 AI 面板                                │
│     /agents transcript 3 → 查看完整对话记录                     │
│                                                                 │
│  4. 结果返回                                                   │
│     父代理聚合结果:                                             │
│     [🤖 #1 ✅ 完成: handler.rs 审查 — 3 issues]                 │
│     [🤖 #2 ✅ 完成: model.rs 审查 — 1 issue]                    │
│     [🤖 #3 ❌ 失败: 超时, 部分结果: 前 2 文件已审查]             │
│                                                                 │
│  5. 清理                                                       │
│     子代理退出 → worktree 自动清理 → 临时文件删除               │
│     用户看到: [🤖 #3 已清理 — worktree .worktrees/agent-3 已移除] │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### H0 RMCP 迁移 — 用户感知

| 事件 | 用户看到的 |
|------|-----------|
| MCP Server 连接成功 | 🟢 `🔌 MCP: github-mcp 已连接 (12 工具可用)` |
| MCP Server 连接失败 | 🟡 `🔌 MCP: github-mcp 连接失败 — [r] 重连 [d] 禁用` |
| MCP 工具被调用 | 状态栏: `[🔧 mcp__github__create_issue: 执行中]` |
| MCP Server 断线 | 🟡 `🔌 MCP: github-mcp 已断线 — 60s 后自动重连` |
| 作为 MCP Server 被外部调用 | 🟢 `🌐 Kaku Server: 外部 Agent 调用了 fs_read` |

MCP 管理命令 (`/mcp`):
| 子命令 | 功能 |
|--------|------|
| `/mcp list` | 列出所有已配置 MCP Server 及连接状态 |
| `/mcp add` | 交互式添加 MCP Server (支持 stdio/HTTP/SSE) |
| `/mcp remove <name>` | 移除 MCP Server 配置 |
| `/mcp restart <name>` | 重连 MCP Server |
| `/mcp tools <name>` | 列出某 Server 提供的工具 |

#### H1 角色区分 — 用户感知

| 角色 | 允许工具 | 用户看到的创建提示 |
|------|---------|------------------|
| Leaf | 12 只读 + 8 写入 (无 agent_spawn/task_create) | `[🤖 子代理 #4 (leaf) — 只读+写入, 不能创建子代理]` |
| Orchestrator | 全部 (含 agent_spawn, task_*) | `[🤖 子代理 #5 (orchestrator) — 可创建子代理, 深度限制 2]` |
| Custom | 用户指定工具集 | `[🤖 子代理 #6 (custom) — 工具: fs_read, grep, symbol_search]` |

深度超限时: `❌ 无法创建子代理 — 已达最大深度 (3/3)`

#### H2 子代理通信 — 用户感知

```
┌─── 消息传递 UI ────────────────────────────────────────┐
│                                                         │
│  📨 agent_send(#2 → #3):                                │
│  "我在 handler.rs 发现了一个 unwrap，你在 model.rs 看看"  │
│                                                         │
│  📨 agent_send(#3 → #2):                                │
│  "收到，model.rs 的 phone 字段也需要处理"               │
│                                                         │
│  消息历史: /agents messages 2                            │
│  消息缓冲: 最多 100 条，溢出丢弃最旧                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### H3 Worktree 隔离 — 用户感知

| 事件 | 用户看到的 |
|------|-----------|
| 子代理创建 worktree | `[📁 worktree: .worktrees/agent-4 创建 (分支: agent/task-4)]` |
| 子代理在 worktree 中修改文件 | (正常执行，用户不感知 worktree 细节) |
| 子代理退出后清理 | `[📁 worktree: .worktrees/agent-4 已清理]` |
| 写入失败自动回滚 | `[🔄 worktree 回滚: fs_write 失败, 已 git checkout .]` |

#### H4 心跳超时 — 用户感知

| 心跳状态 | 用户看到的 |
|----------|-----------|
| 正常 (每 10s) | 状态栏 `❤️ OK` (不显示，静默) |
| 3 次丢失 | 🟡 `[⚠ 子代理 #3 心跳丢失 (3/5)]` |
| 5 次丢失 → 终止 | 🔴 `[❌ 子代理 #3 已终止 — 心跳超时] [v] 查看部分结果 [r] 重启` |

#### H5 Transcript — 用户感知

```
/agents transcript 3

┌─── 子代理 #3 Transcript ──────────────────────────────┐
│                                                         │
│  [轮次 1] 用户指令: 审查 src/auth/ 下的所有文件          │
│  🧠 思考: 需要读取 handler.rs, service.rs, model.rs     │
│  🔧 fs_read(handler.rs) → 420 行                        │
│  🔧 fs_read(service.rs) → 280 行                        │
│  📝 分析完成: 发现 3 个 issue                            │
│                                                         │
│  [轮次 2] 用户指令: 检查边界情况                         │
│  🧠 思考: 需要关注 unwrap() 和空值处理                   │
│  🔧 grep("unwrap()", "src/auth/") → 12 处匹配           │
│  ...                                                    │
│                                                         │
│  📊 Token 使用: 25K/50K | 轮次: 5/∞                    │
│  [e] 导出为 Markdown [c] 压缩 transcript                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### H6 结果聚合 — 用户感知

```
┌─── 聚合结果面板 ───────────────────────────────────────┐
│                                                         │
│  🤖 3 个子代理审查完成:                                  │
│                                                         │
│  ✅ #1 handler.rs — 3 issues (1 critical, 2 warning)    │
│     L42: unwrap() 可能 panic                             │
│     L89: SQL 拼接风险                                   │
│     L15: 建议提取为常量                                  │
│                                                         │
│  ✅ #2 model.rs — 1 issue (warning)                    │
│     L23: 硬编码 SMS provider secret                      │
│                                                         │
│  ❌ #3 tests/ — 超时, 部分完成                          │
│     完成: test_auth.rs (0 issues)                       │
│     未完成: test_sms.rs (超时)                           │
│                                                         │
│  📊 去重: 0 重复 | Token 总消耗: 45K                    │
│  [a] 自动修复  [f] 逐条修复  [d] 展开 diff              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## v0.15.0 — 扩展性 (Phase I, W5-7)

> 目标: 建立可扩展架构，降低第三方集成门槛。

### I1: Hook 系统 (~700 LOC)

**文件**: 新建 `ai_tools/hooks.rs`, 修改 `ai_chat_engine/mod.rs`

**配置**: `.claude/hooks.json`

**事件 (22 个)**:
- Session: `SessionStart`, `SessionEnd`, `SessionPause`, `SessionResume`
- Prompt: `UserPromptSubmit`, `AssistantResponse`
- Tool: `PreToolUse`, `PostToolUse`, `ToolTimeout`
- Agent: `SubagentStart`, `SubagentStop`, `AgentError`
- Context: `CompactStart`, `CompactEnd`, `BudgetWarning`
- System: `ModelSwitch`, `NotificationReceive`
- Memory: `MemoryRead`, `MemoryWrite`
- Approval: `PermissionRequest`

**类型 (5 种)**:
- `command`: shell 命令，JSON stdin/stdout，5s 超时
- `prompt`: LLM 判断，适合需要理解语义的场景
- `http`: Webhook POST，适合 CI/CD 集成
- `function`: 会话级 Rust 函数，适合高性能场景
- `callback`: 进程内回调，适合临时逻辑

**特性**: 并行执行、条件匹配 (`if: "Bash(git *)"`)、once 标记、continueOnBlock

参考: Claude Code 28 Hook 事件 + 6 种类型

**验证**: PreToolUse hook 拦截 `rm` 命令, SubagentStart hook 记录子代理创建

### I2: MCP 配置与运维层 (~300 LOC)

> 基于 H0 的 RMCP 迁移，Streamable HTTP/OAuth/Sampling 已由 RMCP 内置。此任务聚焦配置管理和运维策略。

**文件**: 修改 `ai_tools/mcp_client.rs`, 新建 `ai_tools/mcp_config.rs`

**方案**:
- MCP Server 配置管理: `.claude/mcp_servers.json` 声明式配置 (server 命令/URL/环境变量/OAuth)
- 熔断器策略: 连续 5 次失败自动断开，60s 后半开重试
- 工具命名空间: `mcp__{server}__{tool}` 格式，避免多 Server 命名冲突
- 会话恢复: 重连后恢复工具列表和会话状态
- 集成 Smithery/mcp.so: 支持 `npx @smithery/cli add <name>` 一键安装社区 MCP Server

参考: RMCP 内置 Streamable HTTP + OAuth 2.1 + Sampling; Hermes 熔断器策略

**验证**: HTTP/SSE MCP server 连接, 调用工具, 断线重连后状态恢复, 一键安装社区 MCP Server

### I3: Skill 系统 (~400 LOC)

**文件**: 新建 `ai_tools/skills.rs`, 修改引擎和 UI

**目录格式**: `.claude/skills/{name}/SKILL.md`
```yaml
---
name: code-review
trigger: ["/code-review", "code review"]
allowed-tools: [Read, Grep, Glob, LSP]
disallowed-tools: [Write, Edit, Bash]
effort: medium
paths: ["src/**/*.rs"]
---
```

**特性**:
- 条件激活: `paths` 字段匹配当前工作区文件类型时自动激活
- 工具过滤: `allowed-tools` / `disallowed-tools` 约束可用工具集
- 动态加载: 热加载/卸载，无需重启

参考: Claude Code Skills + Plugins

**验证**: code-review skill, `/code-review` 触发后工具集过滤

### I4: LSP 工具集成 (~300 LOC) [新增]

**文件**: 新建 `ai_tools/lsp_tool.rs`, 修改 `ai_tools/mod.rs`

**方案**:
- 利用 WezTerm 已有 LSP 支持，桥接到 Agent 工具层
- 5 个核心操作: `goToDefinition`, `findReferences`, `hover`, `documentSymbol`, `workspaceSymbol`
- 自动检测工作区的语言服务器配置
- 结果格式化为结构化文本注入上下文

参考: Claude Code LSP Tool (v2.0.74)

**验证**: Agent 使用 `workspaceSymbol("MyStruct")` 找到定义位置

### I5: 配置管理 (~200 LOC) [新增]

**文件**: 新建 `config/mod.rs`, `.claude/settings.json`

**方案**:
- 统一配置入口: `.claude/settings.json`
- 分层配置: global (`~/.claude/settings.json`) > project (`.claude/settings.json`) > session
- 配置项: 模型选择、权限规则、Hook 配置、MCP 服务器列表、工具预算
- 配置验证: 启动时校验 schema，无效配置告警

参考: Claude Code settings.json

**验证**: 项目级 settings 覆盖全局 settings，session 级临时设置生效

### I 交互细节

#### I1 Hook 系统 — 用户交互

**Hook 配置** (`.kaku/hooks.json`):
```json
{
  "hooks": [
    {
      "event": "PreToolUse",
      "condition": "Bash(rm *)",
      "type": "command",
      "command": "echo 'rm 命令执行前确认'",
      "timeout": 5,
      "continue_on_block": false
    },
    {
      "event": "PostToolUse",
      "type": "function",
      "name": "auto_lint",
      "continue_on_block": true
    },
    {
      "event": "SubagentStart",
      "type": "http",
      "url": "https://hooks.my-ci.com/agent-start",
      "headers": { "Auth": "${CI_TOKEN}" }
    }
  ]
}
```

**Hook 事件类型与交互**:

| 事件类型 | 触发时机 | 用户感知 |
|----------|---------|---------|
| `command` | 事件触发时执行 shell 命令 | 命令输出注入上下文；失败时 `[⚠ Hook "auto-lint" 执行失败] [s] 跳过` |
| `prompt` | 需要 LLM 判断时 | 同步等待 LLM 判断结果，状态栏 `[🔧 Hook: prompt 判断中...]` |
| `http` | Webhook POST | 异步发送，不阻塞；失败时 🟡 提示 |
| `function` | 进程内回调 | 无感知 (高性能) |
| `callback` | 临时逻辑注册 | 无感知 |

**Hook 管理命令** (`/hooks`):
```
/hooks              → 列出所有已加载 Hook + 触发次数 + 成功/失败统计
/hooks --reload     → 重新加载 hooks.json
/hooks --test <id>  → 手动触发指定 Hook 测试
/hooks --disable 2  → 临时禁用 #2 Hook
/hooks --enable 2   → 重新启用
```

**Hook 执行流程** (用户看到的):
```
PreToolUse 触发 → 匹配 "Bash(rm *)" → 执行 command hook
┌─────────────────────────────────────────┐
│ 🔧 Hook: pre-rm-check (command)         │
│ → 输出: "正在检查 rm 目标路径..."        │
│ → 结果: continue (允许执行)              │
└─────────────────────────────────────────┘
```

#### I2 MCP 配置与运维 — 用户交互

**MCP Server 配置** (`.kaku/mcp_servers.json`):
```json
{
  "servers": {
    "github-mcp": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" },
      "timeout": 30,
      "reconnect": { "max_retries": 3, "backoff": 60 }
    },
    "db-mcp": {
      "type": "streamable-http",
      "url": "http://localhost:3001/mcp",
      "oauth": { "client_id": "kaku", "scopes": ["read", "write"] }
    }
  }
}
```

**连接状态面板** (`/mcp list`):
```
┌─── MCP Servers ────────────────────────────────────────┐
│                                                         │
│  🟢 github-mcp  (stdio)    12 工具  ✅ 已连接          │
│  🟢 db-mcp       (HTTP)     8 工具  ✅ 已连接          │
│  🔴 old-server   (SSE)      —       ❌ 连接失败 (3次)    │
│     [r] 重连 [d] 禁用 [c] 查看配置                      │
│                                                         │
│  工具命名空间: mcp__github__create_issue                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**一键安装社区 MCP Server**:
```
/mcp add @smithery/cli/filesystem
→ 🔍 搜索: filesystem MCP Server
→ 📦 安装: @smithery/cli/filesystem@1.2.0
→ ✅ 已添加到 mcp_servers.json, 5 个工具可用
```

#### I3 Skill 系统 — 用户交互

**Skill 创建** (`/skills`):
```
/skills                    → 列出已加载 Skills
/skills --create           → 交互式创建新 Skill (模板向导)
/skills --reload           → 热重载所有 Skills
/skills code-review        → 查看 Skill 详情
/skills code-review --edit → 编辑 SKILL.md
```

**Skill 激活方式**:

| 激活方式 | 示例 | 用户看到的 |
|----------|------|-----------|
| Slash 命令 | `/code-review` | `[📋 Skill: code-review 已激活 — 工具集: 只读]` |
| 自然语言触发 | "请帮我审查代码" | `[📋 Skill: code-review 自动激活 (匹配: "审查代码")]` |
| 路径条件激活 | 进入 `src/**/*.rs` 目录 | `[📋 Skill: rust-reviewer 自动激活 (路径匹配)]` |
| 手动切换 | `/skills code-review --enable` | `[📋 Skill: code-review 已启用]` |

**Skill 执行中的工具过滤**:
```
/code-review src/auth/
→ Skill: code-review 已激活
→ 工具集: [Read, Grep, Glob, LSP] (4 个可用)
→ 工具集: [Write, Edit, Bash] (3 个已过滤)
→ 🔍 正在审查 src/auth/...
```

#### I4 LSP 工具集成 — 用户感知

| LSP 操作 | Agent 使用方式 | 用户感知 |
|----------|---------------|---------|
| `goToDefinition` | Agent 查询符号定义位置 | (用户不直接看到，结果注入 Agent 上下文) |
| `findReferences` | Agent 查找所有引用 | 同上 |
| `hover` | Agent 获取类型信息 | 同上 |
| `documentSymbol` | Agent 获取文件结构大纲 | 同上 |
| `workspaceSymbol` | Agent 全工作区搜索符号 | 同上 |

LSP 状态指示:
```
🟢 LSP: rust-analyzer 已连接 (workspace: 142 symbols)
🔴 LSP: 未检测到语言服务器 — 部分功能不可用
```

#### I5 配置管理 — 用户交互

**配置层次** (优先级从高到低):

```
会话级 (/settings 临时) > 项目级 (.kaku/settings.json) > 全局级 (~/.kaku/settings.json)
```

**settings.json 结构**:
```json
{
  "model": { "default": "GLM-5.1", "fallback": "GLM-4.7" },
  "sandbox": { "network": "deny", "allowed_paths": ["./"] },
  "terminal_context": { "max_commands": 20 },
  "shortcuts": { "global": "Cmd+Shift+K", "panel_toggle": "Ctrl+K" },
  "budget": { "default_tokens": 200000 },
  "suggest_commands": true,
  "input": { "vi_mode": false }
}
```

**配置验证**: 启动时校验 schema，无效配置:
```
⚠ settings.json 验证失败:
  Line 12: "budget.default_tokens" 应为数字, 实际为字符串
  已使用默认值: 200000
[?] 查看完整验证报告 [f] 自动修复 [i] 忽略
```

---

## v0.16.0 — 终端独特优势 (Phase J, W7-9)

> 目标: 放大"终端内嵌 AI"独特优势，建立竞品无法复制的护城河。
> **决策节点 (W9)**: Phase J 完成时评估终端方案 — 继续 WezTerm / 迁移 Ghostty (45K stars) / 自研轻量终端。

### J1: 终端会话上下文注入 (增强版) (~250 LOC)

**文件**: `ai_tools/terminal_context.rs`, `ai_chat_engine/mod.rs`

**方案** (基于 G2.3 前置版增强):
- 命令历史语义分析: 标记成功/失败/超时/中断
- 前台进程状态: 知道正在运行的编译/测试/部署进度
- 终端布局感知: 窗口大小、pane 布局、字体大小
- 智能摘要: 最近 50 条命令压缩为关键事件流 (而非原始列表)

参考: Kaku 独有差异化, Codex Shell Snapshot

**验证**: Agent 知道 `cargo build` 刚失败并看到错误摘要

### J2: 实时命令流监控 (~350 LOC)

**文件**: `ai_tools/terminal_context.rs`, `overlay/termwindow/mod.rs`

**方案**:
- `terminal_watch(cols, lines)` 返回终端文本快照
- 自适应限流: 空闲时每轮最多 3 次，有编译/测试运行时放宽到 5 次
- 智能截取: 只捕获变化部分 (diff against previous snapshot)
- 结构化解析: 从终端输出中提取编译错误、测试结果、进度信息

参考: Kaku 独有差异化

**验证**: `cargo test` 后 Agent 看到 test 输出和失败摘要

### J3: 内联 Diff 编辑器 (~500 LOC)

**文件**: `overlay/ai_chat/render.rs`, `state.rs`, `input.rs`

**方案**:
- 富文本 diff: 红/绿高亮, 利用 WezTerm 的终端颜色能力
- 导航: j/k 逐行, J/K 按 hunk 跳转, g/G 首尾
- 操作: a (accept all), y (accept hunk), n (reject hunk), e (edit in editor)
- 并行 diff: 多文件变更同时展示, Tab 切换
- 统计: 左上角显示 `+{added} -{removed} {file_count} files`

参考: Kaku 已有 Diff 预览审批的增强版

**验证**: 3 文件变更, accept 2 hunk, reject 1, e 打开编辑器

### J4: 多 Tab AI 协作 (~500 LOC) [新增]

**文件**: 新建 `ai_tools/tab_cooperation.rs`, 修改 `ai_tools/terminal_context.rs`

**方案**:
- 利用 WezTerm 多 Tab 能力，不同 Tab 可运行独立 Agent
- 跨 Tab 上下文共享: 当前 Tab、工作目录、git branch
- 消息总线: Tab A 的 Agent 可向 Tab B 发送结果
- 统一视图: `Tab` 工具列出所有 Tab 的 Agent 状态
- 权限控制: Agent A 不能读取 Tab B 的对话历史 (只共享元数据)

参考: Kaku 独有差异化 (所有竞品无此能力)

**验证**: Tab 1 运行测试 Agent，Tab 2 运行开发 Agent，开发 Agent 知道测试结果

### J5: Agent 通知系统 (~200 LOC) [新增]

**文件**: 新建 `ai_tools/notification.rs`, `overlay/termwindow/mod.rs`

**方案**:
- 后台 Agent 完成时终端内通知 (利用 WezTerm bell + 自定义通知栏)
- 通知级别: info (完成), warning (需关注), error (失败)
- 通知队列: 最多 20 条，可查看历史
- 快捷操作: 通知栏可直接跳转到对应 Agent/Tab

参考: Claude Code 桌面通知

**验证**: 后台 Agent 完成任务，用户在其他 Tab 收到通知，点击跳转

### J6: 智能命令建议 (~300 LOC) [新增]

**文件**: `ai_tools/terminal_context.rs`, `overlay/ai_chat/input.rs`

**方案**:
- Agent 基于上下文主动建议下一步命令
- 建议来源: 历史模式 (用户之前类似的操作流)、当前状态 (编译失败→建议 fix 命令)
- 展示: 在输入行上方显示灰色的建议命令，Tab 接受
- 可配置: `.claude/settings.json` 中 `suggest_commands: true/false`

**验证**: 编译失败后，Agent 建议正确的编译命令，用户 Tab 接受

### J7: AI 快速启动 — `ka` 命令 + 全局快捷键 (~350 LOC) [新增]

**文件**: 新建 `bin/ka.rs` (CLI 入口), `overlay/ai_chat/input.rs` (快捷键注册), 新建 `config/shortcut.rs` (冲突检测)

**方案**:

**`ka` 命令 (终端内)**:
- 在任意终端中输入 `ka` 即可唤起 Kaku AI 对话模式
- 支持直接带参数: `ka "帮我修这个 bug"` 跳过唤起直接执行
- 支持管道: `echo "cargo test failed" | ka` 从 stdin 注入上下文
- `ka --resume` 恢复上次会话

**全局快捷键 (多平台)**:

| 平台 | 默认快捷键 | 备选 | 说明 |
|------|-----------|------|------|
| macOS | Cmd+Shift+K | Cmd+\. | 检查 Xcode/IDE/系统级冲突 |
| Windows | Ctrl+Shift+K | Win+K | 检查 VS/IDE/Windows Terminal 冲突 |
| Linux (X11) | Ctrl+Shift+K | Super+K | 检查桌面环境 (GNOME/KDE/i3) 快捷键冲突 |
| Linux (Wayland) | Ctrl+Shift+K | Super+K | 需兼容 compositor 快捷键层 |

**冲突检测机制**:
- 启动时扫描系统已注册快捷键: macOS (Carbon/IOKit), Windows (RegisterHotKey), Linux (X11 GrabKey / Wayland portal)
- 检测到冲突时: 日志警告 + 自动切换到备选 + 提示用户可在 `settings.json` 自定义
- 用户可通过 `settings.json` 的 `ai.shortcut` 字段覆盖默认值

**终端内快捷键 (Kaku 终端)**:
- `Ctrl+K`: 在当前 Tab 内切换 AI 面板 (收起/展开)
- `Esc`: 退出 AI 对话回到终端 (不关闭 AI，后台继续)
- `Ctrl+Shift+Enter`: 提交当前输入并执行 (区分普通回车换行)

**体验目标**: 用户在任何终端、任何时刻，1 秒内进入 AI 对话模式 — 这是 vibecoding 的入口。

参考: Warp Cmd+K (AI 搜索), Cursor Cmd+K (AI 编辑), Claude Code `/` 前缀

**验证**: 三个平台各测试 — 默认快捷键无冲突; 手动冲突时自动降级到备选; `ka` 命令在非 Kaku 终端也能唤起

### J 交互细节

#### J1 终端上下文注入增强 — 用户感知

相比 G2.3 前置版，增强内容:

| 增强项 | 注入内容 | 用户可感知方式 |
|--------|---------|--------------|
| 命令语义分析 | 不仅列出命令，还标记: 成功(✓)/失败(✗)/超时(⏱)/中断(⚡) | Agent 回复中引用: "你上次 `cargo test` 失败了，让我看看原因" |
| 前台进程状态 | 知道当前正在运行的编译/测试/部署 | Agent: "`cargo build` 正在运行，我等它完成后看结果" |
| 终端布局感知 | 窗口大小、pane 布局 | Agent 生成代码时适配终端宽度 |
| 智能摘要 | 最近 50 条命令 → 关键事件流 | `/terminal-summary` 命令可手动查看 |

```
┌─── 增强版终端上下文 (注入到 Agent) ─────────────────────┐
│                                                         │
│  📂 cwd: ~/project/src/auth                            │
│  🔀 branch: feat/sms (3 ahead, 1 behind main)           │
│  📁 dirty: [user_model.rs M, handler.rs M]              │
│                                                         │
│  🖥 最近事件流 (智能摘要):                               │
│    ✓ cargo build — 成功, 2.3s, 0 warnings              │
│    ✗ cargo test --auth — 失败, test_sms::verify panic  │
│    ⏱ vim handler.rs — 编辑中 (130s, 仍在运行)           │
│    ✓ git add user_model.rs                             │
│                                                         │
│  💡 前台进程: vim (PID 48291, 编辑 handler.rs)           │
│  📐 终端: 120×40, 1 Tab, 无 split pane                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### J2 实时命令流监控 — 用户感知

| 场景 | Agent 行为 | 用户感知 |
|------|-----------|---------|
| `cargo test` 运行中 | Agent 调用 `terminal_watch()` 获取终端快照 | 状态栏: `[👀 监控终端: cargo test 输出中]` |
| 编译错误出现 | Agent 从终端输出提取错误 | Agent: "我看到 `cargo build` 报错了: missing field `phone`" |
| 测试通过 | Agent 解析测试结果 | Agent: "所有测试通过了 (15/15)" |
| 长时间无输出 | Agent 限流: 空闲 3 次 → 最多, 运行中放宽到 5 次 | 节省 token |

```
┌─── 命令流监控 (Agent 视角) ────────────────────────────┐
│                                                         │
│  terminal_watch() 返回 (变化部分):                      │
│                                                         │
│  diff (新增内容):                                       │
│  +   Compiling auth v0.1.0                              │
│  +   error[E0425]: cannot find value `phone` in scope  │
│  +     --> src/auth/handler.rs:42                       │
│  +      |                                              │
│  +   error: aborting due to 1 previous error          │
│                                                         │
│  Agent 解析: 1 编译错误, 位置 handler.rs:42, 缺少 phone │
│  自适应限流: 本次为编译运行, 下次 5s 后再检查           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### J3 内联 Diff 编辑器 — 详细交互

```
┌─── Diff 编辑器 (完整交互) ─────────────────────────────┐
│                                                         │
│  src/auth/user_model.rs                      +12 lines │
│  ═════════════════════════════════════════════════════  │
│                                                         │
│  ┃ 15 ┃ pub struct User {                               │
│  ┃ 16 ┃     pub id: Uuid,                               │
│ +┃    ┃     pub phone_number: Option<String>,  ← hunk 1 │
│  ┃ 17 ┃     pub email: String,                           │
│  ┃ 18 ┃     pub name: String,                           │
│  ┃    ┃                                                 │
│ +┃    ┃     pub fn verify_phone(&self) -> bool { ← hunk 2│
│ +┃    ┃         self.phone_number.is_some()              │
│ +┃    ┃     }                                           │
│  ┃ 19 ┃ }                                               │
│                                                         │
│  ─────────────────────────────────────────────────────  │
│  hunk [1/2] │ +3 -0 │ 文件统计: +12 -0 │ 1/3 files   │
│                                                         │
│  导航: [j/k] 逐行  [J/K] 按 hunk  [g/G] 首/尾         │
│  操作: [y] 接受  [n] 拒绝  [e] 编辑  [a] 全部接受      │
│  文件: [Tab] 下个文件  [Shift+Tab] 上个文件              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**并行 Diff 多文件**:
```
┌─── 多文件 Diff ────────────────────────────────────────┐
│                                                         │
│  [Tab 1/3] user_model.rs  +12   [Tab 2] handler.rs +28 │
│  ═════════════════════════════════════════════════════  │
│  (当前文件 Diff 内容...)                                │
│                                                         │
│  [Shift+Tab] 切换文件 | [a] 全部接受 | [n] 全部拒绝     │
│  总计: +53 -8 | 3 files | hunk: 2/7                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### J4 多 Tab AI 协作 — 用户感知

```
┌─── 多 Tab 协作 ────────────────────────────────────────┐
│                                                         │
│  Tab 栏:                                                │
│  [1: test-runner 🧪] [2: dev 🤖] [3: terminal 💻]      │
│                                                         │
│  Tab 1 (test-runner Agent):                             │
│    正在运行 cargo test --release                         │
│    Agent 知道 Tab 2 的开发进度                           │
│                                                         │
│  Tab 2 (dev Agent):                                     │
│    正在修复 test_sms::verify                            │
│    状态栏: [📡 收到 Tab 1 测试结果: 3/5 通过]           │
│                                                         │
│  跨 Tab 交互:                                           │
│    - Agent 自动获取其他 Tab 的 git branch / cwd / 状态   │
│    - Tab A 的 Agent 可发送消息给 Tab B 的 Agent           │
│    - 用户可切换 Tab 查看各 Agent 的执行情况               │
│                                                         │
│  权限:                                                  │
│    ✅ 共享: cwd, branch, 任务状态, 测试结果              │
│    ❌ 不共享: 对话历史, 工具结果, 用户输入               │
│                                                         │
│  /agents tab                                            │
│  → Tab 1: 🧪 test-runner | 运行中 | cargo test          │
│  → Tab 2: 🤖 dev | 运行中 | 修复 test_sms              │
│  → Tab 3: 💻 terminal | 空闲 | 用户手动操作              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### J5 通知系统 — 用户感知

```
┌─── 通知栏 (终端右上角) ────────────────────────────────┐
│                                                         │
│  🔔 3 条通知                                            │
│                                                         │
│  ┌─────────────────────────────────────────┐            │
│  │ ✅ [12:30] Tab 1: 测试全部通过 (15/15)   │ [查看]    │
│  │ ⚠ [12:25] Tab 2: 编译警告 (2 warnings)  │ [查看]    │
│  │ ❌ [12:20] agent #3: 超时终止            │ [查看]    │
│  └─────────────────────────────────────────┘            │
│                                                         │
│  通知级别:                                              │
│    info (完成) → 🔔 + 音效 (可关闭)                      │
│    warning (需关注) → 🔔 + 终端高亮                     │
│    error (失败) → 🔔 + bell + 终端闪烁                  │
│                                                         │
│  快捷操作:                                              │
│    [查看] → 跳转到对应 Tab / Agent                       │
│    [清除] → 关闭通知                                    │
│    [全部清除] → 清空通知队列 (最多 20 条)                │
│                                                         │
│  配置: settings.json → notifications.sound: true/false  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### J6 智能命令建议 — 详细交互

```
┌─── 建议浮层 (终端底部，输入行上方) ────────────────────┐
│                                                         │
│  场景: cargo build 失败, 缺少 phone 字段                │
│                                                         │
│  💡 1/3  cargo build --features sms 2>/dev/null         │
│         原因: User model 缺少 phone 字段导致编译失败     │
│                                                         │
│  [Tab] 执行  [Esc] 忽略  [Shift+Tab] 下一条建议         │
│                                                         │
│  ─────────────────────────────────────────────────     │
│  💡 2/3  cargo check --message-format=short             │
│         原因: 快速检查当前错误                           │
│                                                         │
│  💡 3/3  git diff --stat                                │
│         原因: 查看未提交改动                              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

建议来源优先级: 终端错误上下文 > 历史模式匹配 > 常用命令

#### J7 AI 快速启动 — 详细交互

**`ka` 命令交互**:

| 场景 | 命令 | 行为 |
|------|------|------|
| 基础唤起 | `ka` | 打开 AI 面板，光标在输入框，等待输入 |
| 带指令唤起 | `ka "帮我修这个 bug"` | 打开 AI 面板 + 自动提交指令 + 开始执行 |
| 管道输入 | `echo "cargo test failed" \| ka` | 打开 AI 面板 + 管道内容注入上下文 |
| 恢复会话 | `ka --resume` | 打开 AI 面板 + 恢复上次会话上下文 |
| 指定模型 | `ka --model GLM-4.7 "快速检查"` | 使用轻量模型执行简单任务 |
| 非交互 | `ka --non-interactive "fix lint errors"` | 无 UI, 直接执行, 结果输出到 stdout |

**快捷键唤起体验**:
1. 用户在任何终端、任何时刻按下 `Cmd+Shift+K`
2. Kaku 窗口立即前置 (如果未打开则启动)
3. AI 面板展开, 输入框获得焦点
4. 光标闪烁, 等待输入 — **1 秒内进入 vibecoding 模式**

**启动冲突处理**:
```
⚠ 快捷键冲突: Cmd+Shift+K 已被 Xcode 占用
→ 自动切换到备选: Cmd+\
→ 可在 ~/.kaku/settings.json 中自定义:
   "shortcuts": { "global": "Cmd+Shift+Space" }
```

---

## v0.17.0 — 生态进阶 (Phase K, W9+)

> 目标: 建立 Plugin 生态，编排进阶，Auto Mode。

### K1: Plugin 架构 (~500 LOC)

**文件**: 新建 `plugin/mod.rs`, `plugin/loader.rs`, `plugin/manifest.rs`

**方案**:
- Plugin 可提供: Skills + Hooks + MCP Servers + 工具定义
- 清单格式: `.claude/plugins/{name}/manifest.json`
- 生命周期: install → enable → load → unload → disable
- 沙箱: Plugin 代码在独立线程运行，异常不影响主 Agent

参考: Claude Code Plugin System (v2.0.12)

### K2: Auto Mode 审批分类器 (~250 LOC)

**文件**: `ai_tools/approval.rs`, 新建 `ai_tools/risk_classifier.rs`

**方案**:
- 基于规则的风险分类: 读操作→低风险(自动通过), 写操作→中风险(仅首次询问), 破坏性操作→高风险(始终询问)
- 学习机制: 用户手动审批过一次的操作，后续同类操作自动降级
- 全局开关: `auto_mode: {read: true, write: "learn", destructive: false}`

参考: Claude Code Auto Mode + 权限分类器

### K3: Plan 模式增强 (~300 LOC)

**文件**: `ai_chat_engine/mod.rs`, `overlay/ai_chat/state.rs`, `render.rs`, `input.rs`

**方案**:
- Plan 模式工具集过滤为只读
- 结构化步骤输出: `Step {id, description, files, tools, status}`
- 用户逐条审批: a(accept), r(reject), e(edit step)
- 验证循环: 执行后对比预期 vs 实际结果

参考: Codex PlanHandler + delta

### K4: 会话持久化 (~200 LOC)

**文件**: 新建 `ai_tools/session.rs`, 修改 `ai_chat_engine/mod.rs`

**方案**:
- 会话快照: 每轮结束保存对话状态到 `.claude/sessions/{id}.json`
- `/resume` 命令: 恢复上次会话，压缩后继续
- 自动清理: 超过 7 天的会话自动归档
- 导出: 会话可导出为 Markdown

参考: Claude Code /resume, Hermes 会话持久化

### K5: Feature Flag 体系 (~150 LOC)

**文件**: 新建 `config/feature_flags.rs`

**方案**:
- JSON 配置: `.claude/feature_flags.json`
- 开关类型: boolean, percentage (灰度), user_list
- 运行时切换: 无需重启
- 适用于: 新功能灰度发布、A/B 测试 prompt

参考: Codex 50+ Feature Flags

### K6: Dynamic Workflows (~600 LOC)

**文件**: 新建 `ai_tools/workflow.rs`, 修改 `subagent.rs`

**方案**:
- 声明式工作流: YAML 定义 Agent 依赖图和执行顺序
- 编排能力: 并行分支、条件分支、结果聚合、失败重试
- 可视化: 在终端中展示工作流 DAG 进度
- 适用于: CI/CD pipeline、批量代码审查、大规模重构

参考: Claude Code Dynamic Workflows (v2.1.154), Codex CSV Jobs

### K 交互细节

#### K1 Plugin 架构 — 用户交互

**Plugin 生命周期管理**:

| 操作 | 命令/方式 | 用户看到的 |
|------|----------|-----------|
| 安装 | `kaku plugin install <name>` 或 GUI | `📦 安装 plugin: rust-analyzer-bridge v1.2.0... ✅` |
| 启用 | 自动 (安装后) 或 `kaku plugin enable <name>` | `[📋 plugin: rust-analyzer-bridge 已启用]` |
| 禁用 | `kaku plugin disable <name>` | `[📋 plugin: rust-analyzer-bridge 已禁用 — 重启生效]` |
| 卸载 | `kaku plugin uninstall <name>` | `[🗑️ plugin: rust-analyzer-bridge 已卸载]` |
| 更新 | `kaku plugin update [name]` | `[📦 更新: 3 个 plugin 可用, 1 个需要重启]` |
| 列表 | `/plugins` 或 `kaku plugin list` | 显示所有 plugin 状态、版本、来源 |

**Plugin 清单** (`.kaku/plugins/{name}/manifest.json`):
```json
{
  "name": "rust-analyzer-bridge",
  "version": "1.2.0",
  "description": "Rust LSP 桥接工具",
  "provides": {
    "skills": ["rust-review", "rust-fix"],
    "hooks": ["PostToolUse:auto-clippy"],
    "mcp_servers": ["rust-analyzer-mcp"],
    "tools": ["cargo_check", "cargo_clippy"]
  },
  "permissions": ["fs_read", "fs_write(src/**/*.rs)"],
  "isolation": "thread"
}
```

**Plugin 出错隔离**:
```
⚠ Plugin "rust-analyzer-bridge" 执行异常 (线程已隔离)
→ 错误: LSP 连接超时
→ 主 Agent 不受影响, 继续
→ [r] 重启 plugin [d] 禁用 [i] 忽略
```

#### K2 Auto Mode — 用户交互

**Auto Mode 状态切换**:

```
/auto               → 切换 Auto Mode (on/off)
/auto on            → 开启
/auto off           → 关闭
/auto status        → 查看当前规则
```

**Auto Mode 配置** (`settings.json`):
```json
{
  "auto_mode": {
    "enabled": true,
    "rules": {
      "read": "allow",
      "write": "learn",
      "destructive": "ask"
    },
    "learned_actions": [
      { "pattern": "Bash(cargo test)", "action": "allow", "learned_at": "2026-05-30" }
    ]
  }
}
```

**Auto Mode 行为**:

| 操作类型 | 风险等级 | Auto Mode 行为 | 用户看到的 |
|----------|---------|----------------|-----------|
| 读取文件 (fs_read) | 低 | 自动通过 | 无审批, 直接执行 |
| 搜索 (grep, symbol) | 低 | 自动通过 | 无审批 |
| 写入文件 (fs_write) | 中 | 首次询问, 后续同类自动通过 | 首次: `[y] 接受 [s/p] 记住`; 后续: 静默执行 + 状态栏 `[auto]` |
| Shell 命令 (shell_exec) | 中 | 首次询问, 学习后自动 | 同上 |
| 删除文件 (fs_delete) | 高 | 始终询问 | `[⚠ 高风险] [y] 接受 [n] 拒绝 [p] 永久记住拒绝` |
| git push | 高 | 始终询问 | 同上 |

**学习机制**: 用户手动审批过一次 → 同类操作自动降级。可查看/清除已学习规则:
```
/auto status
已学习规则: 5 条
  1. Bash(cargo test) → allow (2026-05-30)
  2. Write(src/**/*.rs) → allow (2026-05-30)
  3. Bash(git commit) → allow (2026-05-29)
  4. Bash(rm target/) → deny (2026-05-28)
  5. Write(.env*) → deny (2026-05-27)

/auto clear-learned    → 清除所有已学习规则
/auto clear-learned 3  → 清除第 3 条
```

#### K3 Plan 模式增强 — 用户交互

**Plan 模式进入/退出**:

| 方式 | 命令 | 行为 |
|------|------|------|
| 进入 | `/plan` 或 Agent 自动建议 | 工具集过滤为只读, Agent 生成结构化 Plan |
| 带任务 | `/plan 重构 auth 模块` | 直接带任务进入 Plan 模式 |
| 退出 | `Esc` 两次 或 `/plan --exit` | 回到正常模式 |

**Plan 展示与交互**:
```
┌─── Plan 模式 (结构化步骤) ────────────────────────────┐
│                                                         │
│  📋 Plan: 重构 auth 模块分层架构                         │
│  📊 预估: 5 步 | ~200 LOC | 涉及 4 文件                 │
│                                                         │
│  Step 1: 创建 src/auth/service.rs (新建)     ⏳ 执行中  │
│    工具: fs_write → 展示 Diff                           │
│                                                         │
│  Step 2: 迁移验证逻辑到 service.rs        ○ 待确认      │
│    工具: fs_write + fs_delete → Diff + 删除确认          │
│    文件: src/auth/handler.rs, src/auth/service.rs        │
│                                                         │
│  Step 3: 更新 handler.rs 调用 service     ○ 待确认      │
│    工具: fs_write → Diff                                │
│                                                         │
│  Step 4: 添加单元测试                     ○ 待确认      │
│    工具: fs_write → Diff                                │
│                                                         │
│  Step 5: 运行测试验证                     ○ 待确认      │
│    工具: shell_exec(cargo test) → 无需审批               │
│                                                         │
│  ─────────────────────────────────────────────────────  │
│  当前: Step 1/5 | [y] 逐步  [a] 全部  [r] 拒绝        │
│  [e] 编辑步骤  [↑↓] 调整顺序  [+] 添加步骤             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**验证循环** (Plan 特有):
- 每步执行后自动对比预期 vs 实际结果
- 偏差时: `[⚠ Step 2 偏差: 预期 3 处修改, 实际 4 处] [v] 查看差异 [c] 继续`

#### K4 会话持久化 — 用户交互

| 操作 | 命令/方式 | 行为 |
|------|----------|------|
| 保存 | 自动 (每轮结束) + 手动 `/export` | 保存到 `.kaku/sessions/{id}.json` |
| 恢复 | `/resume` 或 `ka --resume` | 压缩后继续上次会话 |
| 查看列表 | `/resume --list` | 列出所有会话, 按时间倒序 |
| 恢复指定 | `/resume abc123` | 恢复指定 session |
| 删除 | `/resume --delete abc123` | 删除指定会话 |
| 清理 | 自动 (7 天后归档) | 归档到 `.kaku/sessions/archive/` |

```
/resume --list

┌─── 会话列表 ───────────────────────────────────────────┐
│                                                         │
│  #1  2026-05-30 14:30  "重构 auth 模块"                 │
│      5 轮对话 | tokens: 45K | 状态: 进行中              │
│                                                         │
│  #2  2026-05-30 10:15  "修复 SMS 验证 bug"               │
│      12 轮对话 | tokens: 120K | 状态: 已完成            │
│                                                         │
│  #3  2026-05-29 16:00  "添加单元测试"                   │
│      8 轮对话 | tokens: 65K | 状态: 已完成              │
│                                                         │
│  输入编号恢复, 或 [Enter] 恢复最近的                      │
│  [a] 归档所有已完成  [c] 清空全部                       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**恢复时的提示**:
```
/resume 1
→ [🔄 恢复会话 #1: "重构 auth 模块"]
→ [压缩: 5 轮 → 12 段摘要, 保留焦点: auth 分层]
→ [✅ 已恢复, 上下文: 200K → 80K]
→ [上次进度: Step 3/5 完成, 下一步: 添加单元测试]
```

#### K5 Feature Flag — 用户交互

**配置** (`.kaku/feature_flags.json`):
```json
{
  "flags": {
    "streaming_tool_execution": { "type": "boolean", "value": true },
    "dynamic_workflows": { "type": "percentage", "value": 50 },
    "advanced_sandbox": { "type": "user_list", "users": ["user_id_1", "user_id_2"] }
  }
}
```

**管理命令**:
```
/flags                     → 列出所有 flag 及当前值
/flags set <name> <value>  → 运行时修改 (无需重启)
/flags enable <name>       → 启用 boolean flag
/flags disable <name>      → 禁用 boolean flag
/flags reset <name>        → 重置为默认值
```

**灰度发布场景**:
```
# 对 50% 用户启用 Dynamic Workflows
/flags set dynamic_workflows 50

# 当前用户命中灰度:
[🧪 Feature Flag: dynamic_workflows = ON (50% rollout)]
[🧪 Feature Flag: advanced_sandbox = OFF (not in user_list)]
```

#### K6 Dynamic Workflows — 用户交互

**工作流定义** (`.kaku/workflows/code-review.yaml`):
```yaml
name: code-review
description: 自动代码审查工作流
steps:
  - id: analyze
    agent: leaf
    prompt: "分析 {files} 的代码质量"
    tools: [Read, Grep, LSP]
    parallel: true  # 多文件并行

  - id: security
    agent: leaf
    prompt: "安全审查 {files}"
    tools: [Read, Grep]
    depends_on: []  # 与 analyze 并行

  - id: aggregate
    agent: orchestrator
    prompt: "合并审查结果, 生成报告"
    tools: [Read, Write]
    depends_on: [analyze, security]
    condition: "analyze.result || security.result"

  - id: fix
    agent: leaf
    prompt: "修复 {aggregate.issues}"
    tools: [Edit, Write]
    depends_on: [aggregate]
    on_failure: retry(2)
```

**执行工作流**:
```
/workflow run code-review --files "src/auth/*.rs"

┌─── 工作流: code-review ──────────────────────────────┐
│                                                         │
│  ┌─ analyze (leaf, 并行) ──────────── [运行中] ──┐    │
│  │  📄 handler.rs  📄 service.rs  📄 model.rs     │    │
│  └────────────────────────────────────────────────┘    │
│  ┌─ security (leaf) ─────────────────── [运行中] ─┐    │
│  │  🔍 扫描安全问题...                             │    │
│  └────────────────────────────────────────────────┘    │
│  ┌─ aggregate ───────────────────────── [等待中] ──┐    │
│  │  ⏳ 等待 analyze + security 完成                │    │
│  └────────────────────────────────────────────────┘    │
│  ┌─ fix ─────────────────────────────── [等待中] ───┐    │
│  │  ⏳ 等待 aggregate 完成                          │    │
│  └────────────────────────────────────────────────┘    │
│                                                         │
│  进度: 2/4 步骤 | 运行中: 3 agents | tokens: 32K       │
│  [Ctrl+C] 中断  [v] 查看详细  [e] 编辑工作流            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**工作流管理命令**:
```
/workflows                     → 列出所有可用工作流
/workflow run <name>           → 执行工作流
/workflow list --running       → 查看运行中的工作流
/workflow cancel <id>          → 取消运行中的工作流
/workflow edit <name>          → 编辑工作流 YAML
```

**终端 DAG 可视化** (运行中):
```
  analyze ──┐
            ├──→ aggregate ──→ fix
  security ─┘

  ✅ analyze (3/3 完成)    ✅ security (1/1 完成)
  🔄 aggregate (执行中)     ⏳ fix (等待中)
```

---

## 依赖关系

```
G1.0 ── G1.1 ── G1.2 ── G1.3 ── G1.4
│         │         │
│         ├── G1.5 ──┤
│         ├── G1.6 ──┤
│         ├── G1.7 ──┤
│         └── G1.8 ──┘
│
G1.1+G1.4 ── G2.0 ── G2.1 ── G2.2
                           │
                           └── G2.3(终端上下文前置)
G2 ─────────────── H0(RMCP迁移) ── H1 ── H2
                                    ├── H3
                                    ├── H4
                                    ├── H5
                                    └── H6
H+RMCP ────────── I1 ── I2(配置层) ── I3
              ├── I4              └── I5
              └── I5
I ─────────────── J1 ── J2 ── J3
              ├── J4
              ├── J5
              ├── J6
              └── J7
J ─────────────── K1 ── K2 ── K3
              ├── K4
              ├── K5
              └── K6
```

---

## 总预估

| 版本 | Phase | 任务 | LOC | 周期 |
|------|-------|------|-----|------|
| 0.12.0 | G1 | 9 (G1.0-G1.8) | ~1320 | W1-2 |
| 0.13.0 | G2 | 4 (G2.0-G2.3) | ~980 | W2-3 |
| 0.14.0 | H | 7 (H0-H6) | ~1880 | W3-5 |
| 0.15.0 | I | 5 (I1-I5) | ~1900 | W5-7 |
| 0.16.0 | J | 7 (J1-J7) | ~2450 | W7-9 |
| 0.17.0 | K | 6 (K1-K6) | ~2000 | W9+ |
| **合计** | | **38** | **~11050** | **12+ 周** |

---

## 与竞品的功能对标

| 功能 | 当前 | G1 | G2 | H | I | J | K |
|------|------|----|----|---|---|---|---|
| 工具并行 | 串行 | G1.1 | | | | | |
| 迭代压缩 | 基础 | G1.3 | | | | | |
| 流式执行 | 无 | G1.2 | | | | | |
| 结果缓存 | 无 | G1.5 | | | | | |
| Bash 沙箱 | 无 | | G2.0 | | | | |
| 权限系统 | glob | | G2.1 | | | | |
| 终端上下文 | 无 | | G2.3 | | J1 | | |
| 子代理角色 | 无 | | | H1 | | | |
| 代理通信 | 无 | | | H2 | | | |
| Worktree 隔离 | 无 | | | H3 | | | |
| Hook 系统 | 无 | | | | I1 | | |
| MCP 增强 | stdio | | | | I2 | | |
| Skill 系统 | 无 | | | | I3 | | |
| LSP 工具 | 无 | | | | I4 | | |
| 命令流监控 | 无 | | | | | J2 | |
| 内联 Diff 编辑 | 基础 | | | | | J3 | |
| 多 Tab 协作 | 无 | | | | | J4 | |
| AI 快速启动 | 无 | | | | | J7 | |
| Plugin 生态 | 无 | | | | | | K1 |
| Auto Mode | 无 | | | | | | K2 |
| Dynamic Workflows | 无 | | | | | | K6 |
| 会话恢复 | 无 | | | | | | K4 |

**仍在竞品之下** (可考虑后续版本):
- Claude Code 6 层压缩管线 (Kaku 有 2 层→增强后约 4 层)
- Codex 4 平台沙箱 (Kaku 只有应用层拦截)
- Hermes PTC (Rust 不兼容，不移植)
- Claude Code 400+ slash 命令 (Kaku Skill 生态需要时间积累)
- Codex Agent Identity 密码学体系 (需要评估必要性)
