# 02 - 系统架构

## 整体架构图

```mermaid
graph TB
    subgraph "分发层 (Distribution)"
        NPM["@openai/codex<br/>npm launcher"]
        BREW["Homebrew"]
        SCRIPTS["安装脚本"]
        SDK_TS["@openai/codex-sdk<br/>TypeScript"]
        SDK_PY["cursor-sdk<br/>Python"]
    end

    subgraph "入口层 (Entry Points)"
        CLI["codex-cli<br/>子命令路由器"]
        CLI --> TUI["codex-tui<br/>Ratatui 交互式"]
        CLI --> EXEC["codex-exec<br/>无头模式"]
        CLI --> APPS["codex-app-server<br/>JSON-RPC 服务"]
        CLI --> MCPS["codex-mcp-server<br/>MCP stdio 服务"]
        CLI --> SANDBOX_CMD["codex sandbox<br/>调试沙箱"]
    end

    subgraph "协议层 (Protocol)"
        APP_PROTO["codex-app-server-protocol<br/>v2 RPC 定义"]
        PROTO["codex-protocol<br/>内部事件/类型"]
        TRANSPORT["codex-app-server-transport<br/>stdio/WS/Unix"]
    end

    subgraph "核心层 (Core)"
        CORE["codex-core<br/>会话引擎"]
        CONFIG["codex-config<br/>配置加载"]
        TOOLS_REG["codex-tools<br/>工具注册"]
        STATE["codex-state<br/>SQLite 持久化"]
        ROLLOUT["codex-rollout<br/>会话录制"]
    end

    subgraph "模型层 (Model)"
        API["codex-api<br/>Responses API"]
        PROVIDER["codex-model-provider<br/>提供商抽象"]
        OLLAMA["codex-ollama"]
        LMSTUDIO["codex-lmstudio"]
    end

    subgraph "工具层 (Tools)"
        SHELL_TOOL["Shell 工具"]
        PATCH_TOOL["Apply Patch"]
        MCP_CLIENT["codex-mcp<br/>MCP 客户端"]
        UNIFIED["Unified Exec<br/>交互终端"]
        CODE_MODE["Code Mode (V8)"]
    end

    subgraph "安全层 (Security)"
        SANDBOXING["codex-sandboxing<br/>平台抽象"]
        LINUX_SB["codex-linux-sandbox<br/>bwrap+seccomp"]
        SEATBELT["Seatbelt<br/>macOS"]
        WIN_SB["Windows Sandbox"]
        EXECPOLICY["codex-execpolicy<br/>命令策略"]
        GUARDIAN["Guardian<br/>自动审批"]
        NET_PROXY["codex-network-proxy<br/>网络代理"]
    end

    subgraph "扩展层 (Extensions)"
        PLUGINS["codex-plugin"]
        SKILLS["codex-skills"]
        HOOKS["codex-hooks"]
        MEMORIES["codex-memories-*"]
    end

    NPM --> CLI
    SDK_TS --> APPS
    SDK_PY --> APPS

    TUI --> APPS
    APPS --> CORE
    EXEC --> CORE
    MCPS --> CORE

    APPS --> APP_PROTO
    APPS --> TRANSPORT
    CORE --> PROTO
    CORE --> CONFIG

    CORE --> API
    API --> PROVIDER
    PROVIDER --> OLLAMA
    PROVIDER --> LMSTUDIO

    CORE --> TOOLS_REG
    TOOLS_REG --> SHELL_TOOL
    TOOLS_REG --> PATCH_TOOL
    TOOLS_REG --> MCP_CLIENT
    TOOLS_REG --> UNIFIED

    SHELL_TOOL --> SANDBOXING
    SANDBOXING --> LINUX_SB
    SANDBOXING --> SEATBELT
    SANDBOXING --> WIN_SB
    CORE --> EXECPOLICY
    CORE --> GUARDIAN
    SANDBOXING --> NET_PROXY

    CORE --> STATE
    CORE --> ROLLOUT
    CORE --> PLUGINS
    CORE --> SKILLS
    CORE --> HOOKS
```

## Crate 依赖关系

### 核心依赖链

```mermaid
graph LR
    CLI[codex-cli] --> TUI[codex-tui]
    CLI --> EXEC[codex-exec]
    CLI --> APPS[codex-app-server]

    TUI --> ASC[app-server-client]
    EXEC --> ASC
    ASC --> CORE[codex-core]

    APPS --> CORE
    CORE --> PROTO[codex-protocol]
    CORE --> CONFIG[codex-config]
    CORE --> TOOLS[codex-tools]
    CORE --> API[codex-api]
    CORE --> SANDBOX[codex-sandboxing]
    CORE --> MCP[codex-mcp]
    CORE --> STATE[codex-state]

    TOOLS --> SANDBOX
    MCP --> RMCP[codex-rmcp-client]
    SANDBOX --> LINUX[codex-linux-sandbox]

    style CORE fill:#f96,stroke:#333
    style PROTO fill:#69f,stroke:#333
    style SANDBOX fill:#f66,stroke:#333
```

### 按功能分组的 Crate 地图

```mermaid
graph TD
    subgraph "🖥️ 用户界面 (5)"
        direction LR
        ui1[codex-cli]
        ui2[codex-tui]
        ui3[codex-exec]
        ui4[codex-app-server]
        ui5[codex-mcp-server]
    end

    subgraph "⚙️ 核心引擎 (12)"
        direction LR
        core1[codex-core]
        core2[codex-protocol]
        core3[codex-config]
        core4[codex-tools]
        core5[codex-state]
        core6[codex-rollout]
        core7[codex-hooks]
        core8[codex-skills]
        core9[codex-features]
        core10[codex-thread-store]
        core11[codex-core-api]
        core12[codex-agent-identity]
    end

    subgraph "🔒 安全 (10)"
        direction LR
        sec1[codex-sandboxing]
        sec2[codex-linux-sandbox]
        sec3[codex-bwrap]
        sec4[codex-execpolicy]
        sec5[codex-network-proxy]
        sec6[codex-shell-command]
        sec7[codex-shell-escalation]
        sec8[codex-process-hardening]
        sec9[codex-arg0]
        sec10[guardian]
    end

    subgraph "🤖 模型 (5)"
        direction LR
        model1[codex-api]
        model2[codex-model-provider]
        model3[codex-model-provider-info]
        model4[codex-ollama]
        model5[codex-lmstudio]
    end

    subgraph "🔌 集成 (8)"
        direction LR
        int1[codex-mcp]
        int2[codex-rmcp-client]
        int3[codex-plugin]
        int4[codex-connectors]
        int5[codex-extension-api]
        int6[codex-memories-mcp]
        int7[codex-memories-read]
        int8[codex-memories-write]
    end

    subgraph "🔧 工具库 (20+)"
        direction LR
        util1[codex-utils-*]
        util2[codex-apply-patch]
        util3[codex-file-system]
        util4[codex-git-utils]
        util5[codex-ansi-escape]
    end
```

## 数据流架构

### 用户输入到模型响应

```mermaid
sequenceDiagram
    participant User as 用户
    participant TUI as TUI/App Server
    participant Core as codex-core Session
    participant LLM as OpenAI API
    participant Tool as 工具执行器
    participant SB as 沙箱

    User->>TUI: 输入消息
    TUI->>Core: Op::UserInput
    Core->>Core: 构建 Prompt (历史+工具+指令)
    Core->>LLM: POST /responses (流式)
    LLM-->>Core: ResponseEvent (流式)
    Core-->>TUI: Event::AgentMessage (delta)

    alt 模型请求工具调用
        LLM-->>Core: function_call (shell/patch/mcp)
        Core->>Core: ExecPolicy 评估
        alt 需要审批
            Core-->>TUI: Event::ApprovalRequest
            User->>TUI: 批准/拒绝
            TUI->>Core: Op::ExecApproval
        end
        Core->>SB: 沙箱包装命令
        SB->>Tool: 执行
        Tool-->>Core: 输出
        Core->>LLM: function_call_output
        LLM-->>Core: 继续响应...
    end

    Core-->>TUI: Event::TurnComplete
    TUI-->>User: 显示最终结果
```

### 会话生命周期

```mermaid
stateDiagram-v2
    [*] --> Created: ThreadManager.start_thread()
    Created --> Configured: SessionConfigured 事件
    Configured --> Idle: 等待用户输入

    Idle --> TurnActive: Op::UserInput
    TurnActive --> ToolExec: 模型请求工具
    ToolExec --> Approval: 需要审批
    Approval --> ToolExec: 用户批准
    ToolExec --> TurnActive: 工具输出 → 模型
    TurnActive --> Idle: TurnComplete

    Idle --> Compacting: Op::Compact
    Compacting --> Idle: ContextCompacted

    TurnActive --> Interrupted: Op::Interrupt
    Interrupted --> Idle: TurnAborted

    Idle --> [*]: Op::Shutdown
```

## 关键设计决策

### 1. Queue-Pair 并发模型

```mermaid
graph LR
    subgraph "客户端侧"
        TX[tx_sub<br/>提交通道]
        RX[rx_event<br/>事件通道]
    end

    subgraph "Session 侧"
        SL[submission_loop<br/>单一事件循环]
        SESSION[Session<br/>状态机]
    end

    TX -->|Op| SL
    SL --> SESSION
    SESSION -->|Event| RX
```

**设计理由**：所有状态变更通过单一提交循环序列化，避免 UI 代码中的锁竞争。客户端（TUI/app-server）通过 channel 对与 Session 通信。

### 2. Session vs Turn 分离

| 层次 | 生命周期 | 状态 |
|------|----------|------|
| `SessionConfiguration` | 整个会话 | 模型、权限、CWD、features |
| `TurnContext` | 单次对话轮次 | 当前模型、环境、覆盖设置 |
| `ActiveTurn` | 活跃执行中 | 任务、审批等待器 |

### 3. 双构建系统

```mermaid
graph TD
    subgraph "开发 (Cargo)"
        C1["just test -p codex-tui"]
        C2["just fmt"]
        C3["just fix -p codex-core"]
        C4["cargo build"]
    end

    subgraph "发布 (Bazel)"
        B1["hermetic 构建"]
        B2["跨平台交叉编译"]
        B3["V8 集成"]
        B4["自定义 lint"]
    end

    DEV[开发者] --> C1
    DEV --> C2
    CI[CI/Release] --> B1
    CI --> B2
```

### 4. App Server 中心化

```mermaid
graph TD
    TUI[TUI] -->|"JSON-RPC (内存)"| AS[App Server]
    EXEC[Exec] -->|"JSON-RPC (内存)"| AS
    IDE[VS Code] -->|"JSON-RPC (stdio/WS)"| AS
    SDK[SDK] -->|"JSON-RPC (stdio)"| AS

    AS --> CORE[codex-core]
```

TUI 和 Exec 不直接调用 `codex-core` 的会话 API——它们通过内嵌的 App Server 客户端走标准 JSON-RPC 协议。这确保了：
- 所有客户端的行为一致
- 协议可独立演进
- IDE 集成与终端使用共享相同代码路径

### 5. 沙箱前置

```
默认状态: 沙箱化执行
  ↓ 执行失败 (沙箱拒绝)
  ↓ 检测到权限不足
  ↓ 请求用户审批
  ↓ 批准后: 无沙箱重试
```

安全模型是**先沙箱后升级**，而非先执行后限制。

## 模块职责边界

| 边界 | 左侧 | 右侧 | 接口 |
|------|-------|-------|------|
| 用户 ↔ Agent | TUI/App Server | codex-core | `Op` / `Event` (codex-protocol) |
| Agent ↔ LLM | codex-core | codex-api | `Prompt` / `ResponseStream` |
| Agent ↔ 工具 | codex-core tools | 沙箱/执行器 | `ExecRequest` / `ExecResult` |
| Agent ↔ MCP | codex-core | codex-mcp | MCP JSON-RPC |
| Server ↔ Client | app-server | IDE/SDK | App Server Protocol v2 |
| 安全 ↔ OS | codex-sandboxing | OS syscalls | bwrap/seatbelt/seccomp |

## 目录结构概览

```
codex-rs/
├── cli/                    # 主二进制 (子命令路由)
├── tui/                    # 交互式终端 UI (~318 源文件)
├── exec/                   # 无头执行模式
├── core/                   # 核心引擎 (~358 源文件)
├── protocol/               # 共享类型定义
├── config/                 # 配置加载
├── tools/                  # 工具注册与路由
├── app-server/             # JSON-RPC 服务器
├── app-server-protocol/    # 协议类型 + Schema 生成
├── app-server-transport/   # 传输层 (stdio/WS/Unix)
├── app-server-client/      # 客户端库
├── sandboxing/             # 沙箱平台抽象
├── linux-sandbox/          # Linux 沙箱实现
├── execpolicy/             # 命令策略引擎 (Starlark)
├── network-proxy/          # 网络代理 (SOCKS5/HTTPS MITM)
├── mcp-server/             # MCP 服务端模式
├── codex-mcp/              # MCP 客户端
├── state/                  # SQLite 持久化
├── rollout/                # 会话录制
├── api/                    # OpenAI API 封装
├── model-provider/         # 模型提供商抽象
├── ollama/                 # Ollama 集成
├── lmstudio/               # LM Studio 集成
├── plugin/                 # 插件系统
├── skills/                 # Skills 系统
├── hooks/                  # 生命周期 Hooks
├── memories/               # 记忆管道
├── ext/                    # 扩展 (goal, guardian, memories)
├── utils/                  # 20+ 工具库
├── vendor/                 # 内嵌 bubblewrap
└── docs/                   # 内部文档
```
