# Kaku 自研终端 + AI Agent 开发工作流设计

## 1. 终端模拟器架构

### 1.1 核心分层

```
┌─────────────────────────────────────────────┐
│               UI 层 (Ratatui)                │
│  Tab 栏 │ 工具栏 │ AI 面板 │ 通知栏      │
├─────────────────────────────────────────────┤
│              渲染引擎 (wgpu/gpu)            │
│  字体渲染 │ 颜色/样式 │ Diff 高亮 │ 图像协议  │
├─────────────────────────────────────────────┤
│              终端层                          │
│  PTY 管理 │ 多 Tab │ 多 Pane │ SSH     │
├─────────────────────────────────────────────┤
│              AI Agent 引擎                    │
│  主循环 │ 工具系统 │ MCP │ 子代理 │ 上下文  │
├─────────────────────────────────────────────┤
│              基础设施                        │
│  配置 │ 日志 │ 遥测 │ 性能监控          │
└─────────────────────────────────────────────┘
```

### 1.2 技术栈

| 组件 | 技术 | 说明 |
|------|------|------|
| 语言 | Rust | 全栈 Rust，零 C 依赖 |
| PTY | `portable-pty` 或 `pty-process` | 跨平台伪终端 |
| 渲染 | `wgpu` + 自研 compositor | GPU 加速，支持 Wayland/X11/macOS Metal |
| 字体 | `cosmic-text` / `fontdb` | GPU 字体渲染，行内图片/Sixel |
| TUI 框架 | `ratatui` | 声/标签/滚动/布局，AI 面板的基础 |
| 异步 | `tokio` | Agent 引擎 + MCP + 网络IO |
| 序列化 | `serde` + `serde_json` | 配置、协议、工具结果 |
| 输入 | crossterm / glutin | 键盘/鼠标事件，原始模式 |

### 1.3 核心模块 (~12-15K LOC)

| 模块 | 职责 | 预估 LOC |
|------|------|----------|
| `pty/` | PTY fork/exec, 原始模式 I/O, shell 集成 | ~1500 |
| `render/` | GPU compositor, 字体光栅化, 滚动缓冲, 图片解码 | ~3000 |
| `term/` | 多 Tab/Pane, 终端状态 (title, cwd, size), 链接检测 | ~1500 |
| `input/` | 键盘映射, 鼠标事件, IME, Vi 模式 | ~1000 |
| `config/` | TOML 配置, 热加载, 分层 (global/project/session) | ~800 |
| `ssh/` | SSH 客户端, 连接管理, 代理跳转 | ~1500 |
| `ui/` | Tab 栏, 状态栏, 命令面板, 搜索, 通知 | ~2000 |
| **AI 引擎** | 见 §2 | ~8000+ |

---

## 2. AI Agent 引擎架构

### 2.1 主循环

```
用户输入 (自然语言)
  │
  ▼
┌──────────┐
│ Prompt   │ 构建 system message + 注入终端上下文
│ Builder  │
└────┬─────┘
     ▼
┌──────────┐     ┌─────────────┐
│ LLM      │────▶│ Token Stream │ 流式输出 + 工具调用解析
│ Client   │     └──────┬──────┘
└──────────┘            │
                    ▼
              ┌──────────────┐
              │ Tool Router   │ 工具分发: 并行只读 / 串行写入
              └──────┬───────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐
    │内置工具 │ │MCP工具  │ │子代理   │
    │(34个)  │ │(外部)   │ │(leaf)   │
    └────┬────┘ └────┬────┘ └────┬────┘
         └───────────┼───────────┘
                     ▼
              ┌──────────────┐
              │ Approval     │ 需要审批? 暂停等待用户
              │ Gate         │ 不需要? 直接执行
              └──────┬───────┘
                     ▼
              ┌──────────────┐
              │ Result       │ 工具结果 注入上下文
              │ Compositor   │ 上下文预算检查 压缩
              └──────┬───────┘
                     ▼
              ┌──────────────┐
              │ Response     │ 渲染: Diff 预览 / 纯文本 / 结构化
              │ Renderer    │ 终端内富文本展示
              └──────────────┘
```

### 2.2 Agent 状态机

```
      ┌────────┐
      │  IDLE  │◀──── 等待用户输入
      └───┬────┘
          │ 用户输入自然语言
          ▼
      ┌────────┐
      │ THINK  │──── LLM 正在思考/生成
      └───┬────┘
          │
     ┌────┴────┐
     ▼         ▼
┌────────┐ ┌────────┐
│ PLAN   │ │ EXEC   │──── 执行工具
└────┬───┘ └───┬────┘
     │         │
     ▼         ▼
┌────────┐ ┌────────┐
│ REVIEW │ │ APPROVE│──── 等待用户审批
└────┬───┘ └───┬────┘
     │         │
     └────┬────┘
          │ 所有步骤完成 / 用户中断
          ▼
      ┌────────┐
      │  IDLE  │
      └────────┘
```

### 2.3 终端上下文注入 (核心差异化)

Kaku 作为终端模拟器本身，拥有所有竞品无法获得的信息:

```rust
struct TerminalContext {
    // Shell 状态
    cwd: PathBuf,
    last_commands: Vec<CommandRecord>,  // 最近 50 条命令
    env_vars: HashMap<String, String>, // 关键环境变量

    // 终端状态
    window_size: (u16, u16),
    tab_count: usize,
    active_pane: PaneId,

    // 前台进程
    foreground_process: Option<ProcessInfo>,
    // 编译/测试/部署等长时间运行的任务

    // Git 状态
    branch: String,
    dirty_files: Vec<PathBuf>,
    recent_commits: Vec<CommitInfo>,

    // Agent 历史 (跨轮次)
    completed_tasks: Vec<TaskSummary>,
    active_plan: Option<Plan>,
}
```

注入策略: 每轮对话开始时，将 TerminalContext 序列化为结构化 JSON 注入 system message。变化时才更新 (diff 注入，非全量)。

---

## 3. 完整开发工作流设计

### 3.1 工作流一: Feature 实现

```
用户: "给用户登录添加手机号验证"

┌── THINK ──────────────────────────────────────────────┐
│ Agent:                                                   │
│ 1. 注入终端上下文 (cwd=src/auth, git branch=feat/sms)  │
│ 2. 搜索相关文件: symbol_search("login"), grep("phone")   │
│ 3. 并行读取: fs_read(login.rs), fs_read(sms.rs),        │
│    fs_read(user_model.rs)                                │
│ 4. 理解架构: auth 模块分层 (handler - service - repo)   │
│ 5. 生成 Plan:                                             │
│    Step 1: 新建 SMS 验证 service (src/auth/sms_verify.rs) │
│    Step 2: 添加手机号字段到 User model                  │
│    Step 3: 修改登录 handler 集成 SMS 验证               │
│    Step 4: 添加配置 (SMS provider, 模板)                 │
│    Step 5: 编写单元测试                                   │
└──────────────────────────────────────────────────────┘

┌── PLAN (工具集过滤为只读) ────────────────────────────┐
│ [1] 新建 SMS 验证 service                 [待确认]   │
│ [2] User model 添加 phone 字段             [待确认]   │
│ [3] 修改登录 handler                     [待确认]   │
│ [4] 添加 SMS 配置                         [待确认]   │
│ [5] 编写测试                             [待确认]   │
│                                                     │
│ 操作: y(通过) n(拒绝) e(编辑) a(全部通过)        │
└──────────────────────────────────────────────────────┘

用户: "y" (通过 Step 1)

┌── EXEC ───────────────────────────────────────────────┐
│ Agent: 执行 Step 1                                     │
│ 1. fs_write(src/auth/sms_verify.rs, ...)             │
│ 2. 展示 Diff 预览:                                   │
│    +  ... 新增 SMS 验证逻辑 ...                       │
│    + 53 行, 2 处修改                                │
│ 3. 等待审批                                         │
└──────────────────────────────────────────────────────┘

┌── APPROVE (内联 Diff 编辑器) ─────────────────────────┐
│ src/auth/sms_verify.rs                     +53 lines │
│ ┌────────────────────────────────────────┐              │
│ + use crate::sms::SmsProvider;         │              │
│ + pub struct SmsVerifyService {          │              │
│ +     provider: SmsProvider,            │              │
│ + }                                   │              │
│ +                                     │              │
│ + pub async fn verify(&self,           │              │
│ +     phone: &str, code: &str            │              │
│ + ) -> Result<VerifyResult>            │              │
│ + }                                   │              │
│ └────────────────────────────────────────┘              │
│                                                     │
│ y: accept hunk  n: reject  e: edit  a: all accept      │
└──────────────────────────────────────────────────────┘

用户: "a" (接受全部)

→ 继续执行 Step 2-5，每步展示 Diff 预览
→ 全部完成后自动运行测试
→ 测试通过: git commit (自动生成 message)
→ 测试失败: Agent 自动分析失败原因，提出修复方案
```

### 3.2 工作流二: Bug 修复 (观察模式)

```
用户: "cargo test 报错 auth 模块 panic"

┌── THINK (观察模式) ──────────────────────────────────┐
│ Agent:                                                   │
│ 1. 终端上下文: 检测到最近的 cargo test 输出           │
│ 2. 解析错误: thread panicked at src/auth/handler.rs   │
│ 3. 定位: 读取 handler.rs, grep panic 消息              │
│ 4. 根因: unwrap() 在 Option::None 时 panic            │
│ 5. 修复: 改为 ? 或 .unwrap_or(default)                   │
│ 6. 自动执行修复                                         │
│ 7. 重新运行 cargo test 验证                            │
└──────────────────────────────────────────────────────┘
```

### 3.3 工作流三: 代码审查

```
用户: "review main..feature-branch"

┌── THINK ──────────────────────────────────────────────┐
│ Agent:                                                   │
│ 1. git diff --stat (文件列表)                         │
│ 2. git log (提交历史)                                  │
│ 3. 逐文件 diff 分析: 逻辑/安全/风格/性能               │
│ 4. 生成审查报告                                        │
└──────────────────────────────────────────────────────┘

┌── REVIEW REPORT ─────────────────────────────────────┐
│ src/auth/handler.rs (3 issues):                      │
│   L42  unwrap() 可能 panic                           │
│   L89  SQL 拼接风险 (参数化查询)                    │
│   L15  建议提取为常量                                  │
│ src/auth/sms_verify.rs (1 issue):                     │
│   L23  硬编码 SMS provider secret                      │
│                                                      │
│ Summary: 4 issues (1 critical, 2 warning, 1 info)      │
│ [a] 自动修复  [f] 逐条修复  [d] 展开 diff            │
└──────────────────────────────────────────────────────┘
```

### 3.4 工作流四: 智能命令建议

```
Agent (后台观察): cargo build 失败, 缺少字段 `phone`

┌── 建议浮层 (终端底部) ──────────────────────────────┐
│ 💡 cargo build --features sms 2>/dev/null             │
│ 原因: User model 缺少 phone 字段导致编译失败          │
│ [Tab] 执行  [Esc] 忽略  [↑↓] 更多建议                   │
└──────────────────────────────────────────────────────┘
```

### 3.5 工作流五: 多 Tab 协作

```
Tab 1: Agent A (测试运行者)
  cargo test --release → 监控测试输出

Tab 2: Agent B (开发)
  读取 Agent A 的测试结果 → 修复失败用例 → 提交测试任务

Tab 3: 用户手动操作
  git log, git diff, 手动验证
```

---

## 4. UI 布局设计

### 4.1 主界面

```
┌─ Tab 1 ── Tab 2 ── Tab 3 ──────────────────── [AI] ─┐
│                                                        │
│  ~/project/src/auth $                  cargo test      │
│                                                        │
│  ┌─ AI Panel (右侧, 可收起) ───────────────────────┐  │
│  │ 💬 正在执行: 给用户登录添加手机号验证           │  │
│  │ 📋 Step 2/5: 添加手机号字段到 User model       │  │
│  │ 🔍 已读取: handler.rs, user_model.rs, sms.rs   │  │
│  │ 🧪 测试: 3/5 通过                            │  │
│  │                                                 │  │
│  │ ┌─ Diff 预览 ──────────────────────────────┐  │  │
│  │ │ src/auth/user_model.rs              +12 lines │  │  │
│  │ │ + phone_number: Option<String>,           │  │  │
│  │ │ [y accept] [n reject] [e edit] [a all]    │  │  │
│  │ └──────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────┘  │
│                                                        │
│  $ 💬 输入消息...                                    │
│  [status: 75% budget | model: claude-4]              │
└────────────────────────────────────────────────────────┘
```

### 4.2 AI 面板状态

| 状态 | 展示 |
|------|------|
| IDLE | 半透明, token 统计 + 模型信息 |
| THINKING | 脉冲动画 |
| PLAN | 结构化步骤, 逐条审批 |
| EXECUTING | 当前工具 + 进度条 + 步骤计数 |
| APPROVAL | 内联 Diff + 操作按钮 |
| ERROR | 错误信息 + 重试计数 + 建议 |
| SUGGESTING | 底部浮层, 命令建议 + Tab 接受 |

---

## 5. 与竞品的交互对比

| 维度 | Claude Code | Cursor | Kaku (设计) |
|------|------------|--------|-------------|
| 代码展示 | 终端纯文本 | IDE 高亮 | 终端富文本 (语法+Diff颜色) |
| Diff 审批 | 纯文本 diff | IDE inline diff | 内联编辑器 (j/k导航, hunk选择) |
| 计划展示 | 文本列表 | IDE panel | 结构化步骤 (逐条审批) |
| 上下文感知 | git + 文件 | IDE 索引 | 完整终端状态 (命令+进程+Tab) |
| 通知 | 终端文本 | IDE notification | 终端内通知栏 + bell |
| 多 Agent | Swarm (独立进程) | 无 | 多 Tab 原生协作 |
| 命令建议 | 无 | Copilot inline | 底部浮层 + Tab 接受 |

---

## 6. 开发路线 (预览)

### Phase 0: 自研终端 MVP (在 Phase G1 之前)

| 任务 | 预估 LOC | 周期 |
|------|----------|------|
| PTY 管理 + 原始模式 I/O | ~1500 | W1 |
| GPU 渲染引擎 (wgpu + 字体) | ~3000 | W2-3 |
| 多 Tab/Pane 管理 | ~1500 | W3 |
| 键盘/鼠标输入 + Vi 模式 | ~1000 | W3 |
| 基础 UI (Tab 栏 + 状态栏 + 搜索) | ~2000 | W3-4 |
| 配置系统 (TOML, 热加载) | ~800 | W4 |
| AI Agent 引擎 (从现有 Kaku 迁移) | ~8000 | W4-8 |
| **合计** | **~17800** | **~8 周** |

### 后续 Phase

Phase G1 → J → K 不变，但所有终端特有功能 (J1-J6) 直接集成到渲染引擎，无需桥接。

---

## 相关记忆

[[kaku-agent-constraints]] — Kaku 当前架构
[[AGENT_ROADMAP]] — 开发路线图
