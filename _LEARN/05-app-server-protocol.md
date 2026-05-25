# 05 - App Server 协议

## 协议概述

App Server 使用**精简版 JSON-RPC**协议（省略 `"jsonrpc": "2.0"` 字段），支持双向通信：

```mermaid
graph LR
    subgraph "客户端 (IDE/SDK)"
        CR[ClientRequest<br/>期望响应]
        CN[ClientNotification<br/>仅通知]
    end

    subgraph "服务端 (App Server)"
        SR[ServerRequest<br/>期望客户端响应]
        SN[ServerNotification<br/>流式推送]
    end

    CR -->|"thread/start, turn/start"| SR
    CN -->|"initialized"| SR
    SR -->|"approval, elicitation"| CR
    SN -->|"item/started, delta"| CR
```

## 初始化握手

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as App Server

    C->>S: initialize {clientInfo, capabilities}
    S-->>C: {userAgent, codexHome, platformFamily}
    C->>S: initialized (notification, 无 id)
    Note over C,S: 握手完成，可以发送其他请求

    opt 实验性 API
        Note over C: capabilities.experimentalApi: true
        Note over S: 解锁实验性方法和通知
    end
```

## 传输层

```mermaid
graph TD
    subgraph "传输方式"
        STDIO["Stdio<br/>(默认, JSONL)"]
        WS["WebSocket<br/>(实验性)"]
        UNIX["Unix Socket<br/>(守护进程)"]
        MEMORY["内存通道<br/>(内嵌模式)"]
    end

    subgraph "连接管理"
        ACCEPT["Transport Acceptor"]
        PROCESS["MessageProcessor"]
        OUTBOUND["Outbound Router"]
    end

    STDIO --> ACCEPT
    WS --> ACCEPT
    UNIX --> ACCEPT
    MEMORY --> ACCEPT

    ACCEPT -->|TransportEvent| PROCESS
    PROCESS -->|OutgoingEnvelope| OUTBOUND
    OUTBOUND --> STDIO
    OUTBOUND --> WS
    OUTBOUND --> UNIX
    OUTBOUND --> MEMORY
```

### 传输编码

| 传输 | 编码 | 适用场景 |
|------|------|----------|
| Stdio | 换行分隔 JSON (JSONL) | CLI 嵌入、SDK |
| WebSocket | 每帧一个 JSON 消息 | IDE 远程连接 |
| Unix Socket | HTTP Upgrade → WS | 本地守护进程 |
| 内存通道 | Rust channel (零序列化) | TUI/Exec 内嵌 |

### WebSocket 安全

- 拒绝带 `Origin` 头的请求 (防止浏览器 CSRF)
- 非 loopback 监听需要认证 (`capability-token` 或 `signed-bearer-token`)
- 背压：channel 容量 128，满时返回 `-32001` 错误

## 服务器架构

```mermaid
flowchart TD
    subgraph "入站处理"
        ACCEPT[Transport 接收] --> PARSE[解析 JSONRPCMessage]
        PARSE --> GATE{已初始化?}
        GATE -->|否| INIT_ONLY[仅允许 initialize]
        GATE -->|是| DISPATCH[分发到 RequestProcessor]
    end

    subgraph "请求处理器"
        DISPATCH --> INIT_P[InitializeProcessor]
        DISPATCH --> THREAD_P[ThreadProcessor]
        DISPATCH --> TURN_P[TurnProcessor]
        DISPATCH --> CONFIG_P[ConfigProcessor]
        DISPATCH --> MCP_P[McpProcessor]
        DISPATCH --> FS_P[FsProcessor]
    end

    subgraph "序列化队列"
        SCOPE{序列化范围}
        SCOPE -->|thread_id| TQ[Thread Queue]
        SCOPE -->|global| GQ[Global Queue]
        SCOPE -->|none| FREE[自由并发]
    end

    subgraph "出站路由"
        RESPONSE[响应] --> WRITER
        NOTIFICATION[通知] --> FILTER[过滤实验性/opt-out]
        FILTER --> WRITER[Connection Writer]
    end

    DISPATCH --> SCOPE
    SCOPE --> RESPONSE
    SCOPE --> NOTIFICATION
```

## RPC 方法目录

### 线程管理

| 方法 | 方向 | 说明 |
|------|------|------|
| `thread/start` | C→S | 创建新线程 |
| `thread/resume` | C→S | 恢复已有线程 |
| `thread/fork` | C→S | 分叉线程 |
| `thread/list` | C→S | 列出线程 |
| `thread/read` | C→S | 读取线程详情 |
| `thread/archive` | C→S | 归档线程 |
| `thread/unsubscribe` | C→S | 取消订阅事件 |
| `thread/compact/start` | C→S | 触发上下文压缩 |
| `thread/rollback` | C→S | 回滚到某个点 |

### Turn 操作

| 方法 | 方向 | 说明 |
|------|------|------|
| `turn/start` | C→S | 开始新的对话轮次 |
| `turn/steer` | C→S | 在活跃 turn 中追加输入 |
| `turn/interrupt` | C→S | 中断当前 turn |
| `review/start` | C→S | 开始代码审查 |

### 配置与账户

| 方法 | 方向 | 说明 |
|------|------|------|
| `config/read` | C→S | 读取配置 |
| `config/value/write` | C→S | 写入单个配置值 |
| `config/batchWrite` | C→S | 批量写入配置 |
| `account/login/start` | C→S | 开始登录流程 |
| `account/read` | C→S | 读取账户信息 |
| `account/rateLimits/read` | C→S | 读取速率限制 |

### MCP 管理

| 方法 | 方向 | 说明 |
|------|------|------|
| `mcpServer/oauth/login` | C→S | MCP 服务器 OAuth |
| `config/mcpServer/reload` | C→S | 重载 MCP 配置 |
| `mcpServerStatus/list` | C→S | MCP 服务器状态列表 |
| `mcpServer/resource/read` | C→S | 读取 MCP 资源 |
| `mcpServer/tool/call` | C→S | 直接调用 MCP 工具 |

### 文件系统

| 方法 | 方向 | 说明 |
|------|------|------|
| `fs/read` | C→S | 读取文件 |
| `fs/write` | C→S | 写入文件 |
| `fs/list` | C→S | 列出目录 |

## 服务器推送

### 通知 (ServerNotification)

```mermaid
graph TD
    subgraph "线程事件"
        TS[thread/started]
        TSC[thread/status/changed]
        TC[thread/closed]
    end

    subgraph "Turn 事件"
        TRS[turn/started]
        TRC[turn/completed]
        TRD[turn/diff/updated]
    end

    subgraph "Item 事件"
        IS[item/started]
        IC[item/completed]
        AMD[item/agentMessage/delta]
        MTP[item/mcpToolCall/progress]
    end

    subgraph "MCP 事件"
        MOC[mcpServer/oauthLogin/completed]
        MSU[mcpServer/startupStatus/updated]
    end
```

### 服务器请求 (需要客户端响应)

| 请求方法 | 说明 | 典型响应 |
|----------|------|----------|
| `item/commandExecution/requestApproval` | 命令执行审批 | approve/deny |
| `item/fileChange/requestApproval` | 文件变更审批 | approve/deny |
| `item/tool/requestUserInput` | 请求用户输入 | text response |
| `mcpServer/elicitation/request` | MCP 引出请求 | form data |
| `item/permissions/requestApproval` | 权限请求审批 | approve/deny |
| `item/tool/call` | 动态工具调用 | tool result |
| `account/chatgptAuthTokens/refresh` | Auth token 刷新 | new tokens |

## 线程事件管道

```mermaid
sequenceDiagram
    participant Core as codex-core
    participant BEH as bespoke_event_handling
    participant Filter as NotificationFilter
    participant Router as Outbound Router
    participant Client as 客户端

    Core->>BEH: Event / EventMsg
    BEH->>BEH: 映射到 ServerNotification/ServerRequest

    alt 是通知
        BEH->>Filter: ServerNotification
        Filter->>Filter: 检查 experimental / opt-out
        Filter->>Router: 通过过滤
        Router->>Client: JSON message
    else 是请求 (审批等)
        BEH->>Router: ServerRequest + 等待响应
        Router->>Client: JSON request
        Client->>Router: JSON response
        Router->>BEH: 客户端决策
    end
```

## 序列化与类型生成

### 双格式输出

```mermaid
flowchart LR
    RUST["Rust 类型定义<br/>(schemars + ts-rs)"] --> JSON_SCHEMA["JSON Schema<br/>schema/json/"]
    RUST --> TS["TypeScript<br/>schema/typescript/v2/"]

    JSON_SCHEMA --> VALIDATE["运行时验证"]
    TS --> SDK_TS["TypeScript SDK"]
    TS --> IDE["IDE 扩展"]
```

### 类型命名约定

| 类型 | 命名规则 | 示例 |
|------|----------|------|
| 请求参数 | `*Params` | `ThreadStartParams` |
| 响应 | `*Response` | `ThreadStartResponse` |
| 通知 | `*Notification` | `TurnCompletedNotification` |
| 线上字段 | camelCase | `threadId`, `clientInfo` |
| 配置字段 | snake_case | `config_key` |

### Serde 注解约定

```rust
// 标准类型
#[derive(Serialize, Deserialize, JsonSchema, TS)]
#[serde(rename_all = "camelCase")]
#[ts(export_to = "v2/")]
pub struct ExampleParams {
    pub thread_id: String,
    #[ts(optional = nullable)]  // 仅用于 C→S 请求参数
    pub cursor: Option<String>,
}

// 区分联合体 (Discriminated Union)
#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
#[ts(tag = "type")]
pub enum ItemKind {
    Message { content: String },
    Command { command: String },
}
```

## v1 vs v2

| 特性 | v1 (遗留) | v2 (活跃) |
|------|-----------|-----------|
| 状态 | 仅维护 | 活跃开发 |
| 方法命名 | PascalCase enum | `resource/method` 字符串 |
| 类型导出 | 无 | JSON Schema + TypeScript |
| 新增 API | 禁止 | 首选位置 |
| 分页 | 无标准 | cursor 分页 |

## 实验性 API 门控

```mermaid
flowchart TD
    REQ[客户端请求] --> CHECK{experimentalApi?}
    CHECK -->|未启用| NORMAL[正常方法处理]
    CHECK -->|已启用| ALL[所有方法可用]

    NORMAL --> EXP_CHECK{方法标记 experimental?}
    EXP_CHECK -->|是| REJECT["返回错误<br/>'experimental API not enabled'"]
    EXP_CHECK -->|否| PROCESS[正常处理]
```

标记方式：

```rust
#[experimental("thread/realtime/start")]
ThreadRealtimeStart => "thread/realtime/start" { ... }
```

## 连接会话状态

每个连接维护：

- `ConnectionRpcGate` — 追踪进行中的 ServerRequest
- `experimental_api_enabled` — 是否启用实验性 API
- `opted_out_notification_methods` — 客户端不想接收的通知
- `client_name` / `client_version` — 客户端标识

## 典型交互序列

### 完整对话流

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as App Server

    C->>S: initialize
    S-->>C: response (server info)
    C->>S: initialized (notification)

    C->>S: thread/start
    S-->>C: response {threadId}
    S-->>C: notification: thread/started

    C->>S: turn/start {items: [{type: "message", content: "Fix the bug"}]}
    S-->>C: notification: turn/started
    S-->>C: notification: item/started (reasoning)
    S-->>C: notification: item/agentMessage/delta
    S-->>C: notification: item/started (shell command)

    S->>C: request: item/commandExecution/requestApproval
    C-->>S: response: {decision: "approve"}

    S-->>C: notification: item/completed (shell)
    S-->>C: notification: item/agentMessage/delta (更多内容)
    S-->>C: notification: item/completed (message)
    S-->>C: notification: turn/completed

    C->>S: thread/unsubscribe
```
