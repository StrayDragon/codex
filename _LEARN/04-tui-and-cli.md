# 04 - TUI 与 CLI

## CLI 架构 (codex-cli)

### 子命令路由

`codex` 二进制是一个多工具路由器 (MultitoolCli)：

```mermaid
graph TD
    CODEX["codex 二进制"] --> CHECK{有子命令?}

    CHECK -->|无| TUI["交互式 TUI<br/>(默认模式)"]
    CHECK -->|有| ROUTE{子命令}

    ROUTE --> EXEC["exec / e<br/>无头执行"]
    ROUTE --> REVIEW["review<br/>代码审查"]
    ROUTE --> LOGIN["login / logout<br/>认证管理"]
    ROUTE --> MCP_CMD["mcp<br/>MCP 服务器管理"]
    ROUTE --> PLUGIN["plugin<br/>插件管理"]
    ROUTE --> APP_SERVER["app-server<br/>JSON-RPC 服务"]
    ROUTE --> MCP_SERVER["mcp-server<br/>MCP 服务端"]
    ROUTE --> SANDBOX["sandbox<br/>调试沙箱"]
    ROUTE --> DOCTOR["doctor<br/>健康诊断"]
    ROUTE --> CLOUD["cloud<br/>云任务"]
    ROUTE --> DEBUG["debug<br/>调试工具"]
    ROUTE --> UPDATE["update<br/>自更新"]
    ROUTE --> RESUME["resume / fork<br/>会话恢复/分叉"]
```

### 共享选项

| 选项 | 作用 | 适用范围 |
|------|------|----------|
| `-m/--model` | 模型覆盖 | TUI + Exec |
| `-s/--sandbox` | 沙箱模式 | TUI + Exec |
| `--yolo` | 跳过审批和沙箱 | TUI + Exec |
| `-p/--profile` | 配置 profile | 全局 |
| `-C/--cd` | 工作目录 | TUI + Exec |
| `-i/--image` | 附加图片 | TUI + Exec |
| `--oss` | 开源本地模型 | TUI + Exec |
| `-c key=value` | 配置覆盖 | 全局 |
| `--enable/--disable` | Feature flags | 全局 |

### Exec 模式特有选项

| 选项 | 作用 |
|------|------|
| `--json` | JSONL 事件流输出 |
| `-o/--output-last-message` | 写最终消息到文件 |
| `--output-schema` | 结构化输出 Schema |
| `--ephemeral` | 不持久化 |
| `--skip-git-repo-check` | 允许在非 git 目录运行 |

## TUI 架构 (codex-tui)

### Widget 层次结构

```mermaid
graph TD
    APP["App<br/>(顶层编排器)"] --> CW["ChatWidget<br/>(主视图)"]
    APP --> OVERLAY["Overlay<br/>(Ctrl+T 转录/Diff)"]
    APP --> ASS["AppServerSession<br/>(JSON-RPC 通信)"]

    CW --> TRANSCRIPT["Transcript 区域<br/>(历史 + 流式)"]
    CW --> BOTTOM["BottomPane<br/>(底部面板)"]

    TRANSCRIPT --> CELLS["HistoryCell[]<br/>(已提交的条目)"]
    TRANSCRIPT --> ACTIVE["active_cell<br/>(流式中)"]

    BOTTOM --> COMPOSER["ChatComposer<br/>(输入框)"]
    BOTTOM --> FOOTER["Footer<br/>(状态栏)"]
    BOTTOM --> VIEWS["BottomPaneView 栈"]

    VIEWS --> APPROVAL["ApprovalOverlay<br/>审批弹窗"]
    VIEWS --> INPUT_REQ["RequestUserInput<br/>用户输入请求"]
    VIEWS --> LIST["ListSelectionView<br/>列表选择器"]
    VIEWS --> ELICIT["McpElicitation<br/>MCP 引出"]
    VIEWS --> FILE_PICK["文件/命令弹窗"]
```

### 渲染系统

TUI 使用自定义 `Renderable` trait 构建可组合的渲染树：

```mermaid
classDiagram
    class Renderable {
        <<trait>>
        +render(area: Rect, buf: &mut Buffer)
        +desired_height(width: u16) u16
        +cursor_pos() Option~Position~
    }

    class FlexRenderable {
        +items: Vec~(flex, Renderable)~
        +push(flex, item)
    }

    class ColumnRenderable {
        +columns: Vec~Renderable~
    }

    class HistoryCell {
        +display_lines() Vec~Line~
    }

    class ChatComposer {
        +render_input()
        +render_mentions()
    }

    Renderable <|-- FlexRenderable
    Renderable <|-- ColumnRenderable
    Renderable <|-- HistoryCell
    Renderable <|-- ChatComposer
```

**布局模型**：

```
┌──────────────────────────────────┐
│  Transcript (flex: 1)            │  ← 弹性增长
│  - HistoryCell (用户消息)        │
│  - HistoryCell (Agent 回复)      │
│  - HistoryCell (命令执行)        │
│  - active_cell (流式中...)       │
├──────────────────────────────────┤
│  Hook Output (flex: 0)           │  ← 固定
├──────────────────────────────────┤
│  BottomPane (flex: 0)            │  ← 固定
│  ┌──────────────────────────────┐│
│  │ ChatComposer / Approval      ││
│  ├──────────────────────────────┤│
│  │ Footer (状态行)              ││
│  └──────────────────────────────┘│
└──────────────────────────────────┘
```

### 事件处理架构

```mermaid
flowchart TD
    subgraph "事件源 (并发)"
        KBD["键盘/终端事件<br/>(crossterm)"]
        AGENT["Agent 事件<br/>(AppServerSession)"]
        INTERNAL["内部 AppEvent"]
    end

    subgraph "App::run() select!"
        SELECT[tokio::select!]
    end

    subgraph "处理路由"
        KBD_H["handle_tui_event"]
        AGENT_H["handle_app_server_event"]
        INTERNAL_H["handle_app_event"]
    end

    KBD --> SELECT
    AGENT --> SELECT
    INTERNAL --> SELECT

    SELECT --> KBD_H
    SELECT --> AGENT_H
    SELECT --> INTERNAL_H

    KBD_H --> GLOBAL{全局热键?}
    GLOBAL -->|是| APP_ACTION["Overlay/导航"]
    GLOBAL -->|否| CW_KEY["ChatWidget.handle_key"]

    CW_KEY --> BOTTOM_KEY["BottomPane 路由"]
    BOTTOM_KEY --> COMPOSER_KEY["Composer 处理"]
    BOTTOM_KEY --> VIEW_KEY["当前 View 处理"]
```

### 键盘输入流

```mermaid
sequenceDiagram
    participant Term as 终端 (crossterm)
    participant Stream as TuiEventStream
    participant App as App
    participant CW as ChatWidget
    participant BP as BottomPane
    participant Comp as ChatComposer

    Term->>Stream: RawEvent
    Stream->>App: TuiEvent::Key(event)
    App->>App: handle_tui_event()

    alt 全局快捷键 (Ctrl+T, Ctrl+C, etc.)
        App->>App: 处理 overlay/退出
    else 路由到 ChatWidget
        App->>CW: handle_key_event()
        CW->>BP: handle_key_event()
        alt 有活跃的 View (审批等)
            BP->>BP: 路由到 active view
        else 默认 → Composer
            BP->>Comp: handle_key_event()
            alt Enter 键
                Comp->>CW: submit_user_message()
                CW->>App: AppEvent::CodexOp(UserTurn)
                App->>App: submit_active_thread_op()
            end
        end
    end
```

### 提交消息流

```mermaid
sequenceDiagram
    participant User as 用户
    participant Comp as ChatComposer
    participant CW as ChatWidget
    participant App as App
    participant ASS as AppServerSession
    participant Server as App Server

    User->>Comp: 输入文字 + Enter
    Comp->>CW: submit_user_message(text, attachments)
    CW->>App: AppEvent::CodexOp(AppCommand::UserTurn)
    App->>ASS: turn_start / turn_steer

    alt 新 Turn
        ASS->>Server: RPC "turn/start"
    else 进行中的 Turn (steerable)
        ASS->>Server: RPC "turn/steer"
    end

    Server-->>ASS: ServerNotification 流
    ASS-->>App: next_event()
    App->>CW: 更新 HistoryCell / active_cell
    CW->>CW: request_frame() → 重绘
```

### 审批交互流

```mermaid
sequenceDiagram
    participant Server as App Server
    participant ASS as AppServerSession
    participant App as App
    participant CW as ChatWidget
    participant BP as BottomPane
    participant User as 用户

    Server->>ASS: ServerRequest(CommandExecutionApproval)
    ASS->>App: server_request
    App->>CW: show_approval_overlay(command_info)
    CW->>BP: push ApprovalOverlay
    BP-->>User: 显示: "允许执行 `rm -rf build/`?"

    User->>BP: 选择: 批准 / 拒绝 / 本次全批
    BP->>CW: approval_decision
    CW->>App: AppEvent::CodexOp(ExecApproval)
    App->>ASS: respond_to_server_request(decision)
    ASS->>Server: RPC response
```

## HistoryCell 类型

转录区域由不同类型的 HistoryCell 组成：

```mermaid
classDiagram
    class HistoryCell {
        <<enum>>
    }

    class UserMessage {
        +text: String
        +attachments: Vec~Attachment~
    }

    class AgentMessage {
        +content: StreamingMarkdown
        +citations: Vec~Citation~
    }

    class ExecCommand {
        +command: String
        +exit_code: i32
        +output: String
        +duration: Duration
    }

    class McpToolCall {
        +server: String
        +tool: String
        +args: Value
        +result: String
    }

    class ApplyPatch {
        +files: Vec~FileChange~
        +diff: String
    }

    class Reasoning {
        +content: String
    }

    HistoryCell --> UserMessage
    HistoryCell --> AgentMessage
    HistoryCell --> ExecCommand
    HistoryCell --> McpToolCall
    HistoryCell --> ApplyPatch
    HistoryCell --> Reasoning
```

## Exec 模式架构

### 执行流程

```mermaid
flowchart TD
    START[启动] --> PARSE[解析 CLI / stdin]
    PARSE --> CONFIG[加载配置]
    CONFIG --> CLIENT[创建 InProcessAppServerClient]
    CLIENT --> THREAD[thread/start 或 thread/resume]
    THREAD --> TURN[turn/start (UserInput)]

    TURN --> LOOP[事件循环]
    LOOP --> EVENT{事件类型}

    EVENT -->|ItemStarted| PROC[EventProcessor]
    EVENT -->|TurnCompleted| DONE[完成]
    EVENT -->|ServerRequest| REJECT[自动拒绝]
    EVENT -->|Error| FAIL[失败退出]

    PROC --> FORMAT{输出格式}
    FORMAT -->|human| STDERR[stderr 进度 + stdout 结果]
    FORMAT -->|json| JSONL[stdout JSONL 流]

    DONE --> EXIT[退出]
```

### 输出模式对比

| 特性 | Human 模式 | JSON 模式 |
|------|-----------|-----------|
| 进度信息 | stderr (spinner) | 无 |
| Agent 消息 | stdout (最终) | stdout JSONL |
| 工具执行 | stderr (简要) | stdout JSONL (详细) |
| 退出码 | 0=成功, 1=错误 | 0=成功, 1=错误 |
| 适用场景 | 人工使用 | 管道/CI |

### JSONL 事件 Schema

```json
{"type": "message", "role": "assistant", "content": "..."}
{"type": "function_call", "name": "shell", "arguments": "..."}
{"type": "function_call_output", "output": "..."}
{"type": "error", "message": "..."}
```

## TUI 样式系统

### Ratatui Stylize 约定

```rust
// 推荐：使用 Stylize trait helpers
"text".dim()
"text".red()
"text".cyan().underlined()
vec!["M".red(), " ".dim(), "file.rs".dim()].into()

// 避免：手动构造 Style
Span::styled("text", Style::default().fg(Color::Red))
```

### 文本换行

```rust
// 纯文本：使用 textwrap
textwrap::wrap(text, width);

// ratatui Line：使用 wrapping.rs helpers
word_wrap_lines(&lines, width);
word_wrap_line(&line, width);
```

## 快照测试

TUI 使用 `insta` 进行快照测试，验证渲染输出：

```bash
# 运行测试，生成新快照
just test -p codex-tui

# 查看待审查快照
cargo insta pending-snapshots -p codex-tui

# 接受所有新快照
cargo insta accept -p codex-tui
```

每个影响 UI 的变更都必须包含对应的 insta 快照覆盖。
