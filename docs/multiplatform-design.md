# Kaku 多平台产品架构设计

## 1. 产品愿景

**一个 AI Agent 引擎，所有入口统一体验。**

```
                  ┌─────────────┐
                  │ Kaku Core  │
                  │  Agent 引擎 │
                  └──────┬──────┘
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
    │ Terminal │   │  Web App  │   │  Mobile  │   │   IDE    │
    │ (自研)   │   │ (浏览器)  │   │  App     │   │ 插件     │
    └──────────┘   └──────────┘   └──────────┘   └──────────┘
          ▼               ▼               ▼               ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
    │ macOS    │   │  PWA      │   │ iOS     │   │ VS Code  │
    │ Linux    │   │ 桌面端    │   │ Android │   │ JetBrains│
    │ Windows  │   │ 移动端    │   │ React   │   │ Neovim   │
    └──────────┘   └──────────┘   └──────────┘   └──────────┘
```

## 2. 架构分层: Core / Shell / Interface

```
┌─────────────────────────────────────────────────┐
│              Interface 层 (UI)                    │
│  Terminal UI │ Web UI │ Mobile UI │ IDE Plugin   │
│  Ratatui     │ React  │ Flutter│ LSP Bridge    │
└──────────────┬──────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────┐
│              Shell 层 (业务逻辑)                  │
│  工具系统 │ 审批 │ Plan │ Hook │ Skill          │
│  MCP Client/Server │ 权限 │ 配置 │ 通知       │
└────────────┬──────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────┐
│              Core 层 (Agent 引擎)                   │
│  LLM Client │ 主循环 │ 上下文管理 │ 子代理       │
│  Token 预算 │ 压缩 │ 流式解析 │ 错误恢复     │
│  RMCP 集成 │ Tool Registry │ Prompt Builder      │
└─────────────────────────────────────────────────────┘
```

## 3. crate 结构

```
kaku/
├── kaku-core/           # Core 层 (~6K LOC)
│   ├── src/
│   │   ├── lib.rs          # Agent 主循环
│   │   ├── llm/            # LLM client (多供应商)
│   │   ├── context/         # 上下文管理 + 压缩
│   │   ├── tools/          # Tool trait + Registry
│   │   ├── subagent/       # 子代理框架
│   │   ├── approval/       # 审批 gate
│   │   ├── budget/         # Token 预算
│   │   └── mcp/            # RMCP Client/Server
│
├── kaku-shell/          # Shell 层 (~5K LOC)
│   ├── src/
│   │   ├── tools/          # 34 内置工具实现
│   │   ├── approval/        # 审批实现
│   │   ├── hooks/           # Hook 执行器
│   │   ├── config/          # 配置管理
│   │   └── permissions/     # 权限系统
│
├── kaku-term/           # Terminal 界面 (~15K LOC, 自研终端)
├── kaku-web/            # Web 界面 (React + WASM bridge)
├── kaku-mobile/         # Mobile 界面 (UniFFI + 平台壳)
├── kaku-ide/            # IDE 插件 (VS Code / JetBrains / Neovim)
└── kaku-cli/             # Headless CLI (CI/脚本)
```

Feature Gates 实现跨平台:
```toml
# kaku-shell/Cargo.toml
[features]
default = []
terminal = ["kaku-term"]
web = ["kaku-web"]
mobile = ["kaku-mobile"]
ide = ["kaku-ide"]
cli = []
```

kaku-core 零平台依赖: tokio, rmcp, serde (无任何 UI/终端依赖)

## 4. 各平台设计

### 4.1 Terminal (自研终端)
- wgpu GPU 渲染, ratatui TUI
- AI Panel 嵌入 Tab 栏, Diff 内联编辑
- 完整终端上下文 (cwd, 命令历史, 前台进程, Tab)
- macOS, Linux, Windows

### 4.2 Web App (浏览器)
- React/Next.js SPA + Rust 后端
- WebSocket 双向通信 (流式输出, 工具调用)
- xterm.js 终端模拟器 (浏览器内运行命令)
- Monaco Editor diff viewer
- PWA 离线可用, 自托管或 Cloudflare

### 4.3 Mobile App (手机)
- UniFFI + 平台原生壳
- kaku-core + kaku-shell 全部编译为 UniFFI
- 对话 + Diff 查看 + 代码编辑器 + 终端
- 推送通知 (Agent 完成, 测试失败, PR 状态)
- iOS 16+, Android 10+

### 4.4 IDE 插件
- VS Code: Webview Panel + LSP Bridge
- JetBrains: Tool Window + LSP Bridge
- Neovim: Float terminal + Lua API
- 通过 Shell 层调用已有 LSP Server

### 4.5 Headless CLI
- stdin/stdout JSON, 类似 Claude Code / Aider
- GitHub Actions 评论/PR, GitLab CI, Jenkins
- 适用于所有有 Rust 工具链的平台

## 5. 跨平台共享与差异

### 完全共享
Agent 主循环, LLM 调用, 工具执行, 审批, 上下文管理, 子代理, MCP, Hook, 配置, Token 预算

### 平台差异
| 终端上下文 | Terminal 完整 | Web 部分(xterm.js) | Mobile 无 | IDE 部分(内嵌终端) |
| Diff 编辑 | 内联 TUI | Monaco (浏览器级) | 基础滚动 | IDE 原生 |
| 文件编辑 | 内置编辑器 | Monaco | 代码编辑器 App | IDE 编辑器 |
| 通知 | bell + 通知栏 | 浏览器通知 | 推送通知 | IDE 通知 |
| 快捷键 | 自定义 Vi 模式 | 键盘快捷键 | 手势 + 键盘 | IDE 快捷键 |
| 离线 | 不支持 | PWA | App 本地 | 不适用 |

## 6. 数据同步 (多设备)

用户在桌面终端开始任务 → 手机上审批 → Web 上查看结果。

同步架构: Session Store (SQLite 本地 + PostgreSQL 云端) + CRDT-like 增量同步。

| 数据 | 同步策略 | 实时性 |
|------|---------|--------|
| 会话历史 | 全量同步 | 实时 |
| 工具/权限/MCP 配置 | 增量 | 准实时 |
| Terminal 上下文 | 仅 Terminal 发送 | 仅活跃时 |

## 7. LLM Provider 策略

**不自研大模型，调用第三方模型提供商。** 不依赖单一供应商，用户可自由切换。

### 7.1 支持的 Provider

| Provider | 海外端点 | 国内端点 | 模型示例 |
|----------|---------|---------|---------|
| 智谱 (Zhipu) | open.bigmodel.cn | open.bigmodel.cn | GLM-5.1, GLM-4.7 |
| OpenAI | api.openai.com | 代理 | GPT-4o, o3 |
| Anthropic | api.anthropic.com | 代理 | Claude 4.7, Claude 4.6 |
| Google | generativelanguage.googleapis.com | 代理 | Gemini 2.5 Pro |
| xAI (Grok) | api.x.ai | 代理 | Grok 3 |
| DeepSeek | api.deepseek.com | api.deepseek.com | DeepSeek-V3, R1 |
| OpenRouter | openrouter.ai/api | 代理 | 聚合 200+ 模型 |
| 本地模型 | localhost (Ollama/llama.cpp/vLLM) | — | Qwen3, Llama 4, DeepSeek |

### 7.2 架构设计

```
┌────────────────────────────────────────────┐
│          kaku-core/src/llm/                 │
│                                            │
│  trait LLMProvider {                       │
│      async fn chat(messages) -> Stream;   │
│      async fn models() -> Vec<ModelInfo>;  │
│      fn supports_tool_use() -> bool;       │
│      fn supports_vision() -> bool;         │
│  }                                         │
│                                            │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐     │
│  │ OpenAI  │ │ Anthro  │ │ Zhipu   │     │
│  │ Impl    │ │ Impl    │ │ Impl    │     │
│  └─────────┘ └─────────┘ └─────────┘     │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐     │
│  │ Gemini  │ │ DeepSeek│ │OpenRoutr│     │
│  │ Impl    │ │ Impl    │ │ Impl    │     │
│  └─────────┘ └─────────┘ └─────────┘     │
│  ┌─────────────────────────────────┐      │
│  │ LocalModel (Ollama/llama.cpp)   │      │
│  └─────────────────────────────────┘      │
│                                            │
│  LLMRouter: 模型选择 + 回退 + 负载均衡    │
│  TokenCounter: 统一 token 计数接口         │
│  StreamParser: 统一流式解析 + 工具调用提取   │
└────────────────────────────────────────────┘
```

### 7.3 关键设计

- **统一 trait 接口**: 所有 Provider 实现相同 trait，上层代码无感知
- **工具调用兼容层**: 不同 Provider 的 tool_use 格式差异在 adapter 层抹平
- **模型选择策略**: 用户配置默认模型 + 任务类型自动路由 (coding 用强模型, 摘要用快模型)
- **回退链**: 主模型不可用时自动切换备用模型
- **本地模型优先**: 检测到本地 Ollama/vLLM 时自动发现可用模型
- **代理/直连自动**: 国内端点走直连，海外端点走用户配置的代理

## 8. 开发优先级

### Phase 0: 架构重构 (W1-2)
Core/Shell 分离重构 + LLM 多 Provider + CLI 入口 + Feature gates。~3200 LOC。

### Phase 1: Terminal MVP (W2-10)
自研终端 MVP + Agent 引擎完整实现 (原 G1-G2-H-I)。

### Phase 2-6: 各平台 UI 叠加
Web (W10-12), IDE (W12-13), Mobile iOS+Android (W13-16), 云同步 (W16-17)。

### Phase 7: 商业化 (W18+)
Pro 订阅, 自托管企业版。

## 9. 总预估

| 组件 | LOC | 周期 |
|------|-----|------|
| kaku-core | ~6000 | W1-2 |
| kaku-shell | ~5000 | W2-10 (逐步) |
| kaku-term | ~15000 | W2-10 |
| kaku-cli | ~500 | W2 |
| kaku-web | ~8000 | W10-12 |
| kaku-mobile | ~6000 | W13-16 |
| kaku-ide | ~3000 | W12-13 |
| 云端同步 | ~2000 | W16-17 |
| **总计** | **~45500** | **~17 周** |

---

## 相关记忆

[[terminal-agent-design]] — 终端 + Agent 工作流详细设计
[[AGENT_ROADMAP]] — 开发路线图
