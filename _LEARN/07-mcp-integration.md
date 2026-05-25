# 07 - MCP 集成

## MCP 在 Codex 中的三种角色

```mermaid
graph TD
    subgraph "角色1: MCP 客户端"
        direction LR
        CORE["codex-core<br/>(Agent Turn)"] --> MCP_MGR["codex-mcp<br/>McpConnectionManager"]
        MCP_MGR --> EXT1["外部 MCP Server 1"]
        MCP_MGR --> EXT2["外部 MCP Server 2"]
        MCP_MGR --> EXT3["外部 MCP Server N"]
    end

    subgraph "角色2: App Server MCP 管理"
        direction LR
        IDE["IDE 客户端"] --> APP["App Server<br/>McpRequestProcessor"]
        APP --> MCP_MGR2["McpConnectionManager"]
    end

    subgraph "角色3: MCP 服务端"
        direction LR
        HOST["外部 MCP Host"] --> MCP_SRV["codex-mcp-server<br/>(独立二进制)"]
        MCP_SRV --> CORE2["codex-core"]
    end
```

| 角色 | Crate | 协议 | 说明 |
|------|-------|------|------|
| MCP 客户端 | `codex-mcp` | MCP JSON-RPC | Agent 调用外部 MCP 工具 |
| MCP 管理 API | `codex-app-server` | App Server Protocol | IDE 管理 MCP 服务器 |
| MCP 服务端 | `codex-mcp-server` | MCP JSON-RPC (stdio) | 外部系统调用 Codex |

## 角色1: MCP 客户端 (codex-mcp)

### 连接管理器架构

```mermaid
classDiagram
    class McpConnectionManager {
        -connections: HashMap~String, McpConnection~
        -tool_index: HashMap~String, ServerName~
        +start_servers(config)
        +aggregate_tools() Vec~ToolSpec~
        +call_tool(server, tool, args)
        +handle_elicitation()
    }

    class McpConnection {
        -client: RmcpClient
        -server_info: ServerInfo
        -tools: Vec~Tool~
        -status: ConnectionStatus
    }

    class RmcpClient {
        +stdio_client(command, args)
        +streamable_http(url)
        +call_tool(name, args)
        +list_tools()
        +read_resource(uri)
    }

    class ElicitationRequestManager {
        +handle_elicitation(request)
        +bridge_to_app_server()
    }

    McpConnectionManager --> McpConnection
    McpConnection --> RmcpClient
    McpConnectionManager --> ElicitationRequestManager
```

### 服务器配置

```toml
# config.toml
[mcp_servers.filesystem]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem", "/path"]
env = { PATH = "/usr/bin" }

[mcp_servers.github]
url = "https://api.github.com/mcp"
auth = "oauth"

[mcp_servers.database]
command = "mcp-server-postgres"
args = ["postgres://localhost/mydb"]
```

### 工具聚合

```mermaid
flowchart TD
    subgraph "配置源"
        USER_CFG["用户 config.toml"]
        PROJ_CFG["项目 config.toml"]
        PLUGIN_CFG["插件提供的 MCP"]
    end

    subgraph "合并"
        EFFECTIVE["effective_mcp_servers()"]
    end

    subgraph "连接"
        CONN1["Server A<br/>tools: read_file, write_file"]
        CONN2["Server B<br/>tools: search, query"]
        CONN3["Server C<br/>tools: deploy, rollback"]
    end

    subgraph "聚合到模型"
        TOOLS["模型可见工具列表"]
        T1["mcp__filesystem__read_file"]
        T2["mcp__filesystem__write_file"]
        T3["mcp__github__search"]
        T4["mcp__database__query"]
    end

    USER_CFG --> EFFECTIVE
    PROJ_CFG --> EFFECTIVE
    PLUGIN_CFG --> EFFECTIVE

    EFFECTIVE --> CONN1
    EFFECTIVE --> CONN2
    EFFECTIVE --> CONN3

    CONN1 --> TOOLS
    CONN2 --> TOOLS
    CONN3 --> TOOLS
    TOOLS --> T1
    TOOLS --> T2
    TOOLS --> T3
    TOOLS --> T4
```

工具命名规则: `mcp__<server_name>__<tool_name>`

### 工具调用流程

```mermaid
sequenceDiagram
    participant LLM as 模型
    participant Core as codex-core
    participant MCM as McpConnectionManager
    participant Server as MCP Server

    LLM->>Core: function_call("mcp__github__search", args)
    Core->>Core: ToolRouter 解析 mcp__ 前缀
    Core->>MCM: call_tool("github", "search", args)
    MCM->>MCM: 查找连接 "github"
    MCM->>Server: MCP tools/call {name: "search", arguments: args}
    Server-->>MCM: MCP result {content: [...]}
    MCM-->>Core: ToolCallResult
    Core-->>LLM: function_call_output
```

### 引出 (Elicitation) 桥接

当 MCP 服务器需要用户输入时：

```mermaid
sequenceDiagram
    participant Server as MCP Server
    participant MCM as McpConnectionManager
    participant Core as codex-core
    participant App as App Server
    participant Client as IDE 客户端
    participant User as 用户

    Server->>MCM: elicitation request (form schema)
    MCM->>Core: ElicitationRequestEvent
    Core->>App: Event → ServerRequest
    App->>Client: mcpServer/elicitation/request
    Client->>User: 显示表单 UI
    User->>Client: 填写并提交
    Client->>App: 响应 (accept + content)
    App->>Core: ElicitationResponse
    Core->>MCM: 转发响应
    MCM->>Server: elicitation response
```

### 连接状态管理

```mermaid
stateDiagram-v2
    [*] --> Starting: start_servers()
    Starting --> Connected: 连接成功
    Starting --> Failed: 连接失败
    Connected --> Refreshing: tools/list 刷新
    Refreshing --> Connected: 刷新完成
    Connected --> Disconnected: 连接断开
    Disconnected --> Reconnecting: 自动重连
    Reconnecting --> Connected: 重连成功
    Reconnecting --> Failed: 重连失败
    Failed --> [*]

    Connected --> OAuthNeeded: 需要认证
    OAuthNeeded --> OAuthFlow: 开始 OAuth
    OAuthFlow --> Connected: 认证成功
```

## 角色2: App Server MCP 管理

### 管理 API

```mermaid
graph TD
    subgraph "IDE 客户端操作"
        LIST["mcpServerStatus/list<br/>列出所有 MCP 服务器状态"]
        OAUTH["mcpServer/oauth/login<br/>启动 OAuth 认证"]
        RELOAD["config/mcpServer/reload<br/>重载配置"]
        CALL["mcpServer/tool/call<br/>直接工具调用"]
        READ["mcpServer/resource/read<br/>读取资源"]
    end

    subgraph "App Server 处理"
        MRP["McpRequestProcessor"]
    end

    subgraph "底层"
        MCM["McpConnectionManager"]
        RMCP["codex-rmcp-client"]
    end

    LIST --> MRP
    OAUTH --> MRP
    RELOAD --> MRP
    CALL --> MRP
    READ --> MRP

    MRP --> MCM
    MRP --> RMCP
```

### 状态列表返回

```json
{
  "data": [
    {
      "name": "filesystem",
      "status": "connected",
      "tools": [
        {"name": "read_file", "description": "Read a file"},
        {"name": "write_file", "description": "Write to a file"}
      ],
      "resources": [],
      "authStatus": null
    },
    {
      "name": "github",
      "status": "auth_required",
      "tools": [],
      "resources": [],
      "authStatus": {
        "type": "oauth",
        "loginUrl": "https://github.com/login/oauth/..."
      }
    }
  ],
  "nextCursor": null
}
```

### OAuth 流程

```mermaid
sequenceDiagram
    participant Client as IDE
    participant App as App Server
    participant RMCP as codex-rmcp-client
    participant MCP as MCP Server
    participant Browser as 浏览器

    Client->>App: mcpServer/oauth/login {serverName: "github"}
    App->>RMCP: start_oauth("github")
    RMCP->>MCP: 获取 OAuth URL
    MCP-->>RMCP: authorization_url
    RMCP-->>App: URL
    App-->>Client: {loginUrl: "https://..."}
    Client->>Browser: 打开 URL
    Browser->>MCP: OAuth callback
    MCP-->>RMCP: tokens
    RMCP-->>App: OAuth 完成
    App-->>Client: notification: mcpServer/oauthLogin/completed
```

### 热重载

```mermaid
sequenceDiagram
    participant Client as IDE
    participant App as App Server
    participant Thread as 加载的 Thread

    Client->>App: config/mcpServer/reload
    App->>App: 从磁盘重载 MCP 配置
    App->>Thread: 队列 McpServerRefreshConfig
    Thread->>Thread: 断开旧连接，建立新连接
    App-->>Client: 成功
    App-->>Client: notification: mcpServer/startupStatus/updated
```

## 角色3: MCP 服务端 (codex-mcp-server)

### 作为 MCP 工具暴露

```mermaid
graph LR
    subgraph "外部 MCP Host"
        HOST[Claude Desktop / 其他 Agent]
    end

    subgraph "codex-mcp-server"
        STDIO["stdin/stdout<br/>MCP JSON-RPC"]
        HANDLER["请求处理"]
        CORE_API["codex-core API"]
    end

    HOST -->|"tools/list"| STDIO
    HOST -->|"tools/call"| STDIO
    STDIO --> HANDLER
    HANDLER --> CORE_API
```

这是一个**独立二进制**，暴露 Codex 为 MCP 工具供其他系统调用。

注意：这与 App Server 协议完全不同——这里说的是真正的 MCP 协议。

### 启动方式

```bash
# 作为 MCP stdio 服务器运行
codex mcp-server

# 在其他工具的 MCP 配置中引用
# claude_desktop_config.json:
{
  "mcpServers": {
    "codex": {
      "command": "codex",
      "args": ["mcp-server"]
    }
  }
}
```

## RMCP 客户端 (codex-rmcp-client)

底层 MCP 通信实现：

```mermaid
classDiagram
    class RmcpClient {
        +stdio(command, args, env)
        +streamable_http(url, headers)
        +call_tool(name, args)
        +list_tools()
        +list_resources()
        +read_resource(uri)
    }

    class Transport {
        <<enum>>
        StdioChild
        StreamableHttp
    }

    class OAuthManager {
        +start_flow(server_url)
        +exchange_code(code)
        +refresh_token()
    }

    RmcpClient --> Transport
    RmcpClient --> OAuthManager
```

支持的传输方式：
- **Stdio** — 启动子进程，通过 stdin/stdout 通信
- **Streamable HTTP** — HTTP SSE 流式通信
- **OAuth** — 自动 token 管理和刷新

## MCP 在 Turn 中的集成点

```mermaid
flowchart TD
    subgraph "Turn 开始"
        START[Turn 启动] --> REFRESH[检查 MCP 刷新请求]
        REFRESH --> TOOLS[聚合所有 MCP 工具]
        TOOLS --> PROMPT[构建 Prompt (含 MCP 工具)]
    end

    subgraph "Turn 执行中"
        CALL[模型调用 MCP 工具] --> ROUTE[ToolRouter 路由]
        ROUTE --> MCP_CALL[McpConnectionManager.call_tool]
        MCP_CALL --> PROGRESS[发送进度事件]
        PROGRESS --> RESULT[返回结果给模型]
    end

    subgraph "事件通知"
        STARTUP["McpStartupUpdate<br/>(服务器启动状态)"]
        TOOL_BEGIN["McpToolCallBegin<br/>(开始调用)"]
        TOOL_PROGRESS["McpToolCallProgress<br/>(执行中)"]
        TOOL_END["McpToolCallEnd<br/>(调用完成)"]
    end

    START --> CALL
    CALL --> STARTUP
    MCP_CALL --> TOOL_BEGIN
    MCP_CALL --> TOOL_PROGRESS
    MCP_CALL --> TOOL_END
```

## 关键源文件

| 文件 | 说明 |
|------|------|
| `codex-mcp/src/connection_manager.rs` | MCP 连接管理器核心 |
| `codex-mcp/src/mcp_connection_manager.rs` | 工具变更管理 |
| `codex-rmcp-client/src/lib.rs` | RMCP SDK 客户端封装 |
| `codex-mcp-server/src/main.rs` | MCP 服务端入口 |
| `core/src/mcp.rs` | Core 中的 MCP 服务器解析 |
| `core/src/mcp_tool_call.rs` | MCP 工具调用处理 |
| `app-server-protocol/src/protocol/v2/mcp.rs` | MCP 管理 API 类型 |
| `protocol/src/mcp.rs` | 共享 MCP 类型 |
