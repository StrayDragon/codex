# 03 - 核心引擎 (codex-core)

## 概述

`codex-core` 是整个 Codex 的**组合根** (Composition Root)——它不自行实现所有逻辑，而是将沙箱、配置、API 传输、策略引擎等委托给兄弟 Crate，自己负责编排。

> ⚠️ 项目明确要求：**抵制向 codex-core 添加代码**。新功能应优先放入独立 Crate。

## 核心概念模型

```mermaid
classDiagram
    class ThreadManager {
        +start_thread()
        +resume_thread()
        +fork_thread()
        +list_threads()
    }

    class CodexThread {
        +submit(Op)
        +events() Stream~Event~
        +settings()
    }

    class Codex {
        -tx_sub: Sender~Submission~
        -rx_event: Receiver~Event~
        +spawn() CodexThread
    }

    class Session {
        -configuration: SessionConfiguration
        -state: SessionState
        -services: SessionServices
        -active_turn: Mutex~ActiveTurn~
        +handle_submission()
    }

    class SessionConfiguration {
        +model: String
        +permissions: Permissions
        +cwd: PathBuf
        +features: ManagedFeatures
    }

    class SessionState {
        +history: Vec~HistoryItem~
        +token_count: TokenInfo
        +compaction_window: Range
    }

    class ActiveTurn {
        +task: TurnTask
        +approval_waiters
        +elicitation_waiters
    }

    class TurnContext {
        +model_client: ModelClientSession
        +sandbox_manager: SandboxManager
        +tool_router: ToolRouter
    }

    ThreadManager --> CodexThread
    CodexThread --> Codex
    Codex --> Session
    Session --> SessionConfiguration
    Session --> SessionState
    Session --> ActiveTurn
    ActiveTurn --> TurnContext
```

## Session 生命周期

### 创建流程

```mermaid
sequenceDiagram
    participant TM as ThreadManager
    participant CT as CodexThread
    participant CX as Codex
    participant S as Session
    participant SL as submission_loop

    TM->>CT: start_thread(config)
    CT->>CX: Codex::spawn(config)
    CX->>CX: 创建 channel pair
    CX->>S: Session::new(config, services)
    CX->>SL: 启动 submission_loop task
    S-->>CX: SessionConfigured 事件
    CX-->>CT: Event channel ready
    CT-->>TM: CodexThread handle
```

### 提交循环 (submission_loop)

这是 Session 的心脏——一个无限循环处理所有入站操作：

```mermaid
flowchart TD
    START[等待 Submission] --> RECV[接收 Op]
    RECV --> MATCH{Op 类型}

    MATCH -->|UserInput| TURN[启动/续行 Turn]
    MATCH -->|Interrupt| INT[中断当前 Turn]
    MATCH -->|ExecApproval| APPR[解除审批等待]
    MATCH -->|ThreadSettings| SET[更新配置]
    MATCH -->|Compact| COMPACT[触发压缩]
    MATCH -->|Shutdown| EXIT[退出循环]
    MATCH -->|McpRefresh| MCP[刷新 MCP 连接]

    TURN --> START
    INT --> START
    APPR --> START
    SET --> START
    COMPACT --> START
    MCP --> START
```

### Turn 执行循环 (run_turn)

这是 Agent 的核心采样循环：

```mermaid
flowchart TD
    START[开始 Turn] --> COMPACT{需要预压缩?}
    COMPACT -->|是| DO_COMPACT[执行上下文压缩]
    COMPACT -->|否| BUILD
    DO_COMPACT --> BUILD

    BUILD[构建 Prompt] --> STREAM[流式请求 LLM]
    STREAM --> PROCESS[处理 ResponseEvent]

    PROCESS --> CHECK{事件类型}
    CHECK -->|AssistantMessage| RECORD[记录到历史]
    CHECK -->|FunctionCall| TOOL[路由工具调用]
    CHECK -->|Completed| DONE[Turn 完成]
    CHECK -->|Error| RETRY{可重试?}

    TOOL --> EXEC[并行执行工具]
    EXEC --> OUTPUT[收集输出]
    OUTPUT --> STREAM

    RECORD --> PROCESS
    RETRY -->|是| STREAM
    RETRY -->|否| ERROR[Turn 错误]

    DONE --> END[发送 TurnComplete]
    ERROR --> END
```

## 配置系统深度解析

### 配置加载流水线

```mermaid
flowchart LR
    subgraph "配置源 (按优先级降序)"
        CLI["CLI 参数"]
        ENV["环境变量"]
        PROJ["项目 config.toml"]
        USER["用户 config.toml"]
        MANAGED["托管需求"]
        DEFAULT["默认值"]
    end

    subgraph "处理"
        LOADER["ConfigBuilder"]
        MERGE["层叠合并"]
        VALIDATE["约束验证"]
    end

    subgraph "输出"
        CONFIG["Config 结构体"]
        SESSION_CFG["SessionConfiguration"]
        TURN_CTX["TurnContext"]
    end

    CLI --> LOADER
    ENV --> LOADER
    PROJ --> LOADER
    USER --> LOADER
    MANAGED --> LOADER
    DEFAULT --> LOADER

    LOADER --> MERGE
    MERGE --> VALIDATE
    VALIDATE --> CONFIG
    CONFIG --> SESSION_CFG
    SESSION_CFG -->|"每次 Turn"| TURN_CTX
```

### 关键配置字段

```toml
# ~/.codex/config.toml 示例

model = "o4-mini"
approval_policy = "unless-trusted"

[sandbox]
permissions_profile = ":workspace"

[permissions]
read_file_on_approve = true

[mcp_servers.filesystem]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem", "/path"]

[hooks.on_turn_complete]
command = "notify-send"
args = ["Codex", "Turn complete!"]
```

## 工具系统

### 工具注册与路由

```mermaid
flowchart TD
    subgraph "工具规划 (spec_plan.rs)"
        FEATURES[ManagedFeatures] --> PLAN[ToolSpecPlan]
        CONFIG[Config] --> PLAN
        PLAN --> REG[ToolRegistry]
    end

    subgraph "工具注册表 (registry.rs)"
        REG --> SHELL[Shell Handler]
        REG --> PATCH[ApplyPatch Handler]
        REG --> MCP_T[MCP Tool Handler]
        REG --> UNIFIED[UnifiedExec Handler]
        REG --> CUSTOM[Custom Tools]
    end

    subgraph "模型可见"
        ROUTER[ToolRouter] --> SPECS["tools JSON<br/>(给 Responses API)"]
        ROUTER --> DISPATCH[dispatch 方法]
    end

    REG --> ROUTER
```

### 工具执行编排 (Orchestrator)

```mermaid
flowchart TD
    CALL[模型返回 function_call] --> ROUTER[ToolRouter.dispatch]
    ROUTER --> PRE_HOOK[执行 pre-tool hooks]
    PRE_HOOK --> HANDLER[Tool Handler]

    HANDLER --> ORCH[ToolOrchestrator]
    ORCH --> POLICY[ExecPolicy 评估]

    POLICY --> DECISION{决策}
    DECISION -->|Allow| SANDBOX_SELECT[选择沙箱]
    DECISION -->|Prompt| APPROVAL[请求审批]
    DECISION -->|Forbidden| DENY[拒绝]

    APPROVAL --> HOOKS[permission hooks]
    HOOKS --> GUARDIAN[Guardian 自动审核]
    GUARDIAN --> USER_PROMPT[用户审批]
    USER_PROMPT --> SANDBOX_SELECT

    SANDBOX_SELECT --> ATTEMPT[沙箱内执行]
    ATTEMPT --> CHECK{成功?}
    CHECK -->|是| POST_HOOK[post-tool hooks]
    CHECK -->|沙箱拒绝| ESCALATE{可升级?}
    ESCALATE -->|是| RE_APPROVE[重新审批]
    RE_APPROVE --> RETRY[无沙箱重试]
    ESCALATE -->|否| FAIL[返回错误给模型]

    POST_HOOK --> OUTPUT[FunctionCallOutput]
    RETRY --> OUTPUT
    DENY --> OUTPUT
    FAIL --> OUTPUT
```

### Shell 工具执行路径

```mermaid
sequenceDiagram
    participant Model as LLM
    participant Router as ToolRouter
    participant Shell as Shell Handler
    participant Orch as Orchestrator
    participant SB as SandboxManager
    participant OS as 操作系统

    Model->>Router: function_call("shell", {command: "ls -la"})
    Router->>Shell: handle(args)
    Shell->>Orch: orchestrate(command, sandbox_type)

    Orch->>Orch: ExecPolicy 检查
    Orch->>SB: transform(command)

    alt Linux
        SB->>OS: bubblewrap + seccomp
    else macOS
        SB->>OS: sandbox-exec (Seatbelt)
    else Windows
        SB->>OS: Restricted Token
    end

    OS-->>SB: 退出码 + stdout/stderr
    SB-->>Orch: ExecResult
    Orch-->>Shell: 输出 (可能截断)
    Shell-->>Router: FunctionCallOutput
    Router-->>Model: tool response
```

## 事件系统

### 事件类型层次

```mermaid
graph TD
    subgraph "入站 Op (用户 → Agent)"
        OP[Op enum]
        OP --> UI[UserInput]
        OP --> INT[Interrupt]
        OP --> EA[ExecApproval]
        OP --> PA[PatchApproval]
        OP --> TS[ThreadSettings]
        OP --> CMP[Compact]
        OP --> REV[Review]
        OP --> SD[Shutdown]
        OP --> MR[McpRefresh]
    end

    subgraph "出站 EventMsg (Agent → 用户)"
        EV[EventMsg enum]
        EV --> SC[SessionConfigured]
        EV --> TS2[TurnStarted]
        EV --> TC[TurnComplete]
        EV --> TA[TurnAborted]
        EV --> AM[AgentMessage]
        EV --> AR[AgentReasoning]
        EV --> ECB[ExecCommandBegin]
        EV --> ECE[ExecCommandEnd]
        EV --> ECD[ExecCommandOutputDelta]
        EV --> EAR[ExecApprovalRequest]
        EV --> TK[TokenCount]
        EV --> ERR[Error]
    end
```

### 事件流转

```mermaid
sequenceDiagram
    participant Client as TUI/App Server
    participant Channel as Channel Pair
    participant SL as submission_loop
    participant Session as Session
    participant Rollout as RolloutRecorder

    Client->>Channel: tx_sub.send(Submission{id, op})
    Channel->>SL: recv()
    SL->>Session: handle_op(op)
    Session->>Session: 执行逻辑...
    Session->>Channel: send_event_raw(EventMsg)
    Channel->>Client: rx_event.recv()
    Session->>Rollout: 持久化选定事件
```

## LLM 交互

### ModelClient 架构

```mermaid
classDiagram
    class ModelClient {
        -auth: AuthState
        -provider: ModelProvider
        -transport: Transport
        +new_session() ModelClientSession
    }

    class ModelClientSession {
        -client: Arc~ModelClient~
        -turn_state: TurnState
        +stream_response(prompt) ResponseStream
        +compact(history) CompactResult
    }

    class Prompt {
        +input: Vec~HistoryItem~
        +tools: Vec~ToolSpec~
        +base_instructions: String
        +personality: Option~String~
        +output_schema: Option~Schema~
    }

    class ResponseStream {
        +next() Option~ResponseEvent~
    }

    class ResponseEvent {
        <<enum>>
        Created
        OutputItemAdded
        OutputTextDelta
        FunctionCallArgumentsDelta
        Completed
        Failed
    }

    ModelClient --> ModelClientSession
    ModelClientSession --> Prompt : 使用
    ModelClientSession --> ResponseStream : 返回
    ResponseStream --> ResponseEvent : 产出
```

### 请求组装

```mermaid
flowchart LR
    subgraph "输入构建"
        HISTORY[对话历史] --> INPUT
        SYSTEM[系统指令] --> INPUT[Prompt.input]
        CONTEXT[上下文片段] --> INPUT
    end

    subgraph "工具声明"
        TOOLS[ToolRouter.specs()] --> TOOLS_JSON[tools JSON]
    end

    subgraph "元数据"
        MODEL[模型名] --> HEADERS
        TURN_ID[Turn ID] --> HEADERS[请求头]
        THREAD_ID[Thread ID] --> HEADERS
    end

    INPUT --> REQUEST[POST /responses]
    TOOLS_JSON --> REQUEST
    HEADERS --> REQUEST
```

## 上下文管理

### 上下文压缩 (Compaction)

```mermaid
flowchart TD
    CHECK[检查 token 用量] --> THRESHOLD{超过阈值?}
    THRESHOLD -->|否| CONTINUE[继续]
    THRESHOLD -->|是| STRATEGY{压缩策略}

    STRATEGY -->|自动| AUTO[自动压缩]
    STRATEGY -->|手动| MANUAL[用户触发]

    AUTO --> REMOTE[远程压缩 API]
    MANUAL --> REMOTE

    REMOTE --> RESULT[压缩后的历史]
    RESULT --> UPDATE[更新 SessionState.history]
    UPDATE --> EVENT[发送 ContextCompacted 事件]
```

### ContextManager 职责

| 功能 | 说明 |
|------|------|
| 历史记录 | 维护完整对话历史 |
| Token 计数 | 跟踪当前 token 用量 |
| 压缩边界 | 确定可压缩范围 |
| 记忆注入 | 在适当位置插入记忆内容 |
| 引用解析 | 解析记忆引用标记 |

## 多 Agent 系统

```mermaid
graph TD
    MAIN[主 Agent] --> SPAWN[agent/spawn]
    SPAWN --> SUB1[子 Agent 1]
    SPAWN --> SUB2[子 Agent 2]

    SUB1 --> REG[Agent Registry]
    SUB2 --> REG

    REG --> LIMITS[Agent 数量限制]
    REG --> COMM[Agent 间通信]

    MAIN --> DELEGATE[codex_delegate]
    DELEGATE --> THREAD[新 Thread]
```

## 错误处理层次

```mermaid
graph TD
    subgraph "公共 API"
        CE[CodexErr]
        CE --> STREAM[Stream - 可重试]
        CE --> ABORT[TurnAborted - 用户中断]
        CE --> CTX[ContextWindowExceeded]
        CE --> SB_ERR[Sandbox - 沙箱错误]
        CE --> FATAL[Fatal - 不可恢复]
    end

    subgraph "工具层"
        FCE[FunctionCallError]
        FCE --> RESPOND[RespondToModel - 告知 LLM]
        FCE --> HARD[Hard Failure]
    end

    subgraph "策略层"
        EPE[ExecPolicyError]
        EPE --> FORBIDDEN[Forbidden]
        EPE --> PARSE[ParseError]
    end

    subgraph "配置层"
        CONST[ConstraintError]
        CONST --> INVALID[Invalid Value]
        CONST --> CONFLICT[Conflict]
    end
```

## 关键源文件索引

| 关注点 | 文件路径 | 说明 |
|--------|----------|------|
| 公共 API | `core/src/lib.rs` | re-exports |
| Thread 管理 | `core/src/thread_manager.rs` | 创建/恢复/分叉 |
| Thread 句柄 | `core/src/codex_thread.rs` | 提交/订阅 |
| Session 生成 | `core/src/session/mod.rs` | Queue-pair, spawn |
| Op 分发 | `core/src/session/handlers.rs` | submission_loop 处理 |
| Turn 循环 | `core/src/session/turn.rs` | run_turn, 采样循环 |
| Turn 上下文 | `core/src/session/turn_context.rs` | 每次 turn 的快照 |
| Session 状态 | `core/src/state/session.rs` | 可变历史/token |
| 配置结构 | `core/src/config/mod.rs` | ~3800 LoC |
| LLM 客户端 | `core/src/client.rs` | ModelClient |
| 工具注册 | `core/src/tools/spec_plan.rs` | 条件注册 |
| 工具路由 | `core/src/tools/router.rs` | dispatch |
| 工具编排 | `core/src/tools/orchestrator.rs` | 审批→沙箱→执行 |
| Shell 执行 | `core/src/exec.rs` | 进程执行+输出捕获 |
| 沙箱桥接 | `core/src/sandboxing/mod.rs` | ExecRequest 适配 |
| 策略引擎 | `core/src/exec_policy.rs` | .rules 文件评估 |
| Guardian | `core/src/guardian/` | 自动审批审核器 |
| 事件映射 | `core/src/event_mapping.rs` | ResponseItem → TurnItem |
| 流处理 | `core/src/stream_events_utils.rs` | 流式delta处理 |
| 压缩 | `core/src/compact.rs` | 上下文压缩 |
| MCP | `core/src/mcp.rs` | MCP 服务器解析 |
| 子 Agent | `core/src/agent/` | 多 agent 控制 |
| Hooks | `core/src/hook_runtime.rs` | 用户 hooks 执行 |
