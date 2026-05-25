# 10 - 模型提供商与记忆系统

## 模型提供商架构

### 分层设计

```mermaid
graph TD
    subgraph "配置层"
        INFO["ModelProviderInfo<br/>(codex-model-provider-info)<br/>序列化配置"]
    end

    subgraph "运行时层"
        PROVIDER["ModelProvider trait<br/>(codex-model-provider)<br/>认证 + 能力 + 工厂"]
    end

    subgraph "HTTP 层"
        API["codex-api<br/>Provider + AuthProvider<br/>Responses API 客户端"]
    end

    subgraph "模型目录"
        MODELS["ModelsManager<br/>(codex-models-manager)<br/>缓存 + 预设"]
    end

    subgraph "本地设置"
        OLLAMA["codex-ollama<br/>健康检查 + 拉取"]
        LMS["codex-lmstudio<br/>健康检查 + 下载"]
    end

    subgraph "核心使用"
        CLIENT["ModelClient<br/>(codex-core)<br/>Turn 请求编排"]
    end

    INFO --> PROVIDER
    PROVIDER --> API
    PROVIDER --> MODELS
    OLLAMA --> PROVIDER
    LMS --> PROVIDER
    API --> CLIENT
    MODELS --> CLIENT
```

### ModelProvider Trait

```rust
pub trait ModelProvider: fmt::Debug + Send + Sync {
    fn info(&self) -> &ModelProviderInfo;
    fn capabilities(&self) -> ProviderCapabilities;
    fn auth_manager(&self) -> Option<Arc<AuthManager>>;
    async fn auth(&self) -> Option<CodexAuth>;
    fn account_state(&self) -> ProviderAccountResult;
    async fn api_provider(&self) -> Result<Provider>;
    async fn runtime_base_url(&self) -> Result<Option<String>>;
    async fn api_auth(&self) -> Result<SharedAuthProvider>;
    fn models_manager(...) -> SharedModelsManager;
}
```

### 内置提供商

```mermaid
graph LR
    subgraph "内置提供商"
        OPENAI["openai<br/>api.openai.com/v1"]
        BEDROCK["amazon-bedrock<br/>bedrock-mantle...api.aws"]
        OLLAMA_P["ollama<br/>localhost:11434/v1"]
        LMS_P["lmstudio<br/>localhost:1234/v1"]
    end

    subgraph "自定义 (config.toml)"
        CUSTOM["[model_providers.my-provider]<br/>base_url = '...'<br/>env_key = 'MY_API_KEY'"]
    end

    OPENAI --> IMPL["ConfiguredModelProvider"]
    OLLAMA_P --> IMPL
    LMS_P --> IMPL
    CUSTOM --> IMPL
    BEDROCK --> BEDROCK_IMPL["AmazonBedrockModelProvider"]
```

### ModelProviderInfo 配置

```rust
pub struct ModelProviderInfo {
    // 身份
    pub name: String,
    pub base_url: String,

    // 认证
    pub env_key: Option<String>,          // 环境变量 API key
    pub auth: Option<AuthCommand>,        // 外部命令获取 bearer
    pub aws: Option<AwsConfig>,           // SigV4 配置
    pub requires_openai_auth: bool,       // 需要 OpenAI 登录

    // 传输
    pub wire_api: WireApi,                // 仅 Responses
    pub supports_websockets: bool,

    // HTTP 调优
    pub query_params: HashMap<String, String>,
    pub http_headers: HashMap<String, String>,
    pub request_max_retries: Option<u32>,
    pub stream_max_retries: Option<u32>,
    pub stream_idle_timeout_ms: Option<u64>,
    pub websocket_connect_timeout_ms: Option<u64>,
}
```

### 认证解析优先级

```mermaid
flowchart TD
    START[api_auth() 调用] --> ENV{env_key 设置?}
    ENV -->|是| BEARER_ENV["BearerAuthProvider(env值)"]
    ENV -->|否| TOKEN{experimental_bearer_token?}
    TOKEN -->|是| BEARER_TOKEN["BearerAuthProvider(token)"]
    TOKEN -->|否| CMD{auth command 配置?}
    CMD -->|是| CMD_AUTH["外部命令 AuthManager"]
    CMD -->|否| LOGIN{CodexAuth 已登录?}
    LOGIN -->|是| CODEX_AUTH["API Key / ChatGPT Auth"]
    LOGIN -->|否| UNAUTH["UnauthenticatedAuthProvider<br/>(本地 OSS)"]
```

### 能力标志

```rust
pub struct ProviderCapabilities {
    pub namespace_tools: bool,      // 工具命名空间
    pub image_generation: bool,     // 图片生成
    pub web_search: bool,           // Web 搜索
}

// OpenAI: 全部 true
// Bedrock: 全部 false
// OSS: 默认 false
```

## 模型目录管理

### ModelInfo 元数据

```rust
pub struct ModelInfo {
    pub slug: String,                    // 模型标识
    pub display_name: String,            // 显示名
    pub context_window: Option<i64>,     // 上下文窗口
    pub max_context_window: Option<i64>, // 最大上下文
    pub auto_compact_token_limit: Option<i64>,

    // 能力
    pub supports_reasoning_summaries: bool,
    pub supports_parallel_tool_calls: bool,
    pub supports_search_tool: bool,
    pub input_modalities: Vec<InputModality>,

    // 推理
    pub default_reasoning_level: Option<ReasoningEffort>,
    pub supported_reasoning_levels: Vec<ReasoningEffortPreset>,

    // 工具
    pub shell_type: ConfigShellToolType,
    pub experimental_supported_tools: Vec<String>,

    // 其他
    pub truncation_policy: TruncationPolicyConfig,
    pub priority: i32,
    pub visibility: ModelVisibility,
}
```

### 目录获取策略

```mermaid
graph TD
    PROVIDER[ModelProvider] --> CHECK{有 config_model_catalog?}

    CHECK -->|是| STATIC["StaticModelsManager<br/>(配置中硬编码)"]
    CHECK -->|否| REMOTE["OpenAiModelsManager"]

    REMOTE --> CACHE{磁盘缓存有效?<br/>(TTL 5min)}
    CACHE -->|是| USE_CACHE["使用缓存"]
    CACHE -->|否| FETCH["GET /models"]
    FETCH --> SAVE["保存到<br/>~/.codex/models_cache.json"]

    STATIC --> RESULT[ModelInfo 列表]
    USE_CACHE --> RESULT
    SAVE --> RESULT
```

## Ollama 集成

### 工作原理

Ollama **不是**独立的 `ModelProvider` 实现——它是一个内置的 `ModelProviderInfo` 条目加上启动检查客户端。

```mermaid
sequenceDiagram
    participant CLI as codex --oss
    participant OSS as codex-utils-oss
    participant Ollama as OllamaClient
    participant Server as Ollama Server

    CLI->>OSS: ensure_oss_ready(config)
    OSS->>Ollama: 创建客户端 (localhost:11434)

    Ollama->>Server: GET /api/version
    Server-->>Ollama: {"version": "0.13.5"}
    Ollama->>Ollama: 检查 >= 0.13.4

    Ollama->>Server: GET /api/tags
    Server-->>Ollama: 模型列表

    alt 默认模型不存在
        Ollama->>Server: POST /api/pull {"name": "gpt-oss:20b"}
        Server-->>Ollama: 流式进度
    end

    Note over CLI: 推理时通过标准 Responses API
    CLI->>Server: POST /v1/responses (OpenAI 兼容)
```

### 版本要求

- **最低版本**: 0.13.4 (Responses API 支持)
- **默认模型**: `gpt-oss:20b`
- **兼容检测**: 检查 base_url 是否以 `/v1` 结尾

## LM Studio 集成

### 工作原理

与 Ollama 类似的模式：

```mermaid
sequenceDiagram
    participant CLI as codex --oss
    participant OSS as codex-utils-oss
    participant LMS as LMStudioClient
    participant Server as LM Studio Server

    CLI->>OSS: ensure_oss_ready(config)
    OSS->>LMS: 创建客户端 (localhost:1234)

    LMS->>Server: GET /models
    Server-->>LMS: 模型列表

    alt 默认模型不存在
        LMS->>LMS: 查找 lms CLI
        LMS->>LMS: 执行 "lms get --yes openai/gpt-oss-20b"
    end

    Note over LMS: 预热模型加载
    LMS->>Server: POST /responses (max_output_tokens=1)

    Note over CLI: 推理时通过标准 Responses API
    CLI->>Server: POST /v1/responses
```

### 区别

| 特性 | Ollama | LM Studio |
|------|--------|-----------|
| 默认端口 | 11434 | 1234 |
| 版本检查 | 是 (≥0.13.4) | 否 |
| 模型拉取 | HTTP API | CLI 命令 (`lms`) |
| 预热 | 否 | 是 (POST /responses) |
| 默认模型 | gpt-oss:20b | openai/gpt-oss-20b |

## 统一设计理念

```mermaid
graph TD
    ALL[所有提供商] --> RESPONSES["统一协议: Responses API"]
    RESPONSES --> STREAM["SSE 流式响应"]

    subgraph "差异点 (封装在 ModelProviderInfo)"
        URL[Base URL]
        AUTH[认证方式]
        RETRY[重试策略]
        CAPS[能力标志]
        CATALOG[模型目录来源]
    end

    URL --> ALL
    AUTH --> ALL
    RETRY --> ALL
    CAPS --> ALL
    CATALOG --> ALL
```

**核心设计**: 一个线上协议 (Responses API) 适配所有后端。提供商差异限定在 URL、认证、重试调优、能力标志和模型目录来源。

---

## 记忆系统 (Memories)

### 概述

记忆系统让 Codex 能**跨会话积累知识**，通过两阶段管道将会话经验提炼为持久化的结构化记忆。

```mermaid
graph TD
    subgraph "记忆管道"
        SESSION["完成的会话<br/>(Rollout JSONL)"]
        PHASE1["Phase 1: 提取<br/>(每个 rollout)"]
        PHASE2["Phase 2: 整合<br/>(全局)"]
        STORAGE["记忆存储<br/>~/.codex/memories/"]
    end

    subgraph "记忆使用"
        READ["记忆读取<br/>(Prompt 注入)"]
        MCP_M["记忆 MCP<br/>(工具接口)"]
    end

    SESSION --> PHASE1
    PHASE1 --> PHASE2
    PHASE2 --> STORAGE
    STORAGE --> READ
    STORAGE --> MCP_M
```

### 存储结构

```
~/.codex/memories/
├── .git/                        # Git 基线 (跟踪变更)
├── MEMORY.md                    # 主记忆文档 (Agent 整理)
├── memory_summary.md            # 摘要 (注入 Prompt)
├── raw_memories.md              # 原始提取的记忆
├── rollout_summaries/
│   ├── fix-auth-bug.md          # 每个 rollout 的摘要
│   └── add-dark-mode.md
└── skills/                      # Agent 自动生成的 Skills
```

### Phase 1: 每 Rollout 提取

```mermaid
sequenceDiagram
    participant Start as 启动任务
    participant DB as State DB
    participant Rollout as Rollout 文件
    participant Model as 提取模型
    participant Store as 存储

    Start->>DB: claim_stage1_jobs_for_startup()
    Note over Start: 过滤: 交互式来源, 时间窗口,<br/>空闲时间, 并发上限 8

    loop 每个 Job
        Start->>Rollout: 加载 JSONL
        Start->>Start: 过滤记忆相关 ResponseItems
        Start->>Model: 流式请求 (JSON Schema 输出)
        Note over Model: 模型: gpt-5.4-mini (默认)
        Model-->>Start: {raw_memory, rollout_summary, rollout_slug}
        Start->>Start: 脱敏 (redact secrets)
        Start->>DB: 存储 stage-1 输出
    end
```

### Phase 2: 全局整合

```mermaid
sequenceDiagram
    participant P2 as Phase 2 任务
    participant DB as State DB
    participant FS as 文件系统
    participant Git as Git
    participant Agent as 整合 Agent

    P2->>DB: claim_global_phase2_lock()
    P2->>Git: 初始化 ~/.codex/memories/.git

    P2->>DB: 加载 top-N stage-1 输出
    Note over P2: 按使用频率 + 最近度排序

    P2->>FS: 同步 raw_memories.md
    P2->>FS: 同步 rollout_summaries/
    P2->>FS: 清理过期摘要 (>7天)

    P2->>Git: 计算 diff
    P2->>P2: 生成 phase2_workspace_diff.md

    alt 有变更
        P2->>Agent: 启动整合子Agent
        Note over Agent: 模型: gpt-5.4<br/>无审批, 无网络, 仅本地写入
        Agent->>Agent: 读取 consolidation.md prompt
        Agent->>FS: 更新 MEMORY.md
        Agent->>FS: 更新 memory_summary.md
        Agent->>FS: 可选: 生成 skills/
        P2->>Git: 重置基线
    end
```

### 记忆读取

```mermaid
flowchart TD
    subgraph "读取路径 (codex-memories-read)"
        PROMPT_INJ["prompts.rs<br/>注入 memory_summary.md<br/>到 developer instructions"]
        CITATIONS["citations.rs<br/>解析 Agent 输出中的引用"]
        USAGE["usage.rs<br/>分类记忆相关命令"]
    end

    subgraph "Prompt 注入"
        SUMMARY["memory_summary.md<br/>(截断到 2500 tokens)"]
        DEV_INST["Developer Instructions"]
    end

    SUMMARY --> PROMPT_INJ
    PROMPT_INJ --> DEV_INST
```

### 记忆 MCP 服务器

`codex-memories-mcp` 提供只读 MCP 工具：

| 工具 | 说明 |
|------|------|
| `list` | 列出记忆文件 |
| `read` | 读取特定记忆 |
| `search` | 搜索记忆内容 |

### 触发条件

记忆管道在以下条件下启动：

1. 非临时会话 (`--ephemeral` 时不启动)
2. `Feature::MemoryTool` 启用
3. 是根 Agent (非子 Agent)
4. State DB 可用

### 关键配置

```toml
# config.toml
[memories]
extract_model = "gpt-5.4-mini"  # Phase 1 提取模型
```

## Responses API 交互

### 请求结构

```rust
pub struct ResponsesApiRequest {
    pub model: String,
    pub instructions: Option<String>,
    pub input: Vec<ResponseItem>,       // 对话历史
    pub tools: Vec<ToolSpec>,           // 工具声明
    pub reasoning: Option<ReasoningConfig>,
    pub text: Option<TextConfig>,       // 输出格式
    pub store: bool,                    // 是否存储
    pub stream: bool,                   // 流式
    // ...
}
```

### 流式响应事件

```mermaid
sequenceDiagram
    participant Client as ModelClient
    participant API as Responses API
    participant Stream as ResponseStream

    Client->>API: POST /responses (stream=true)
    API-->>Stream: SSE: response.created
    API-->>Stream: SSE: output_item.added (reasoning)
    API-->>Stream: SSE: reasoning_summary.delta
    API-->>Stream: SSE: output_item.added (message)
    API-->>Stream: SSE: output_text.delta (多次)
    API-->>Stream: SSE: output_item.done
    API-->>Stream: SSE: output_item.added (function_call)
    API-->>Stream: SSE: function_call_arguments.delta (多次)
    API-->>Stream: SSE: output_item.done
    API-->>Stream: SSE: response.completed
```

### ResponseEvent 枚举

```rust
pub enum ResponseEvent {
    Created { response_id: String },
    OutputItemAdded { item: ResponseItem },
    OutputTextDelta { text: String },
    ReasoningSummaryDelta { text: String },
    FunctionCallArgumentsDelta { text: String },
    OutputItemDone { item: ResponseItem },
    Completed { response_id, token_usage, end_turn },
    RateLimits { ... },
    ServerModel { model: String },
    Failed { error: ApiError },
}
```

### 传输选择

```mermaid
flowchart TD
    CHECK{Provider supports_websockets?}
    CHECK -->|是| WS["WebSocket 传输<br/>(双向, 低延迟)"]
    CHECK -->|否| HTTP["HTTP SSE 传输<br/>(单向流)"]

    WS --> FALLBACK{连接失败?}
    FALLBACK -->|是| HTTP
    FALLBACK -->|否| USE_WS[使用 WebSocket]
```

### 核心 ModelClient 编排

```mermaid
sequenceDiagram
    participant Turn as run_turn
    participant MC as ModelClient
    participant Session as ModelClientSession
    participant API as ApiResponsesClient

    Turn->>MC: new_session()
    MC-->>Turn: ModelClientSession

    Turn->>Session: stream_response(prompt)
    Session->>Session: 解析 Provider + Auth
    Session->>Session: 构建 ResponsesApiRequest
    Session->>API: stream_request(request, options)
    API->>API: POST /responses + SSE 解析
    API-->>Session: ResponseStream

    loop 消费事件
        Session-->>Turn: ResponseEvent
        alt 401 Unauthorized
            Session->>Session: 刷新 Auth
            Session->>API: 重试请求
        end
    end
```

## 关键源文件

| 模块 | 文件 |
|------|------|
| Provider trait | `model-provider/src/provider.rs` |
| Provider 配置 | `model-provider-info/src/lib.rs` |
| 认证解析 | `model-provider/src/auth.rs` |
| Bedrock 实现 | `model-provider/src/amazon_bedrock/mod.rs` |
| Models 端点 | `model-provider/src/models_endpoint.rs` |
| 模型元数据 | `protocol/src/openai_models.rs` |
| 模型缓存 | `models-manager/src/manager.rs` |
| Ollama 客户端 | `ollama/src/client.rs` |
| LM Studio 客户端 | `lmstudio/src/client.rs` |
| OSS 编排 | `utils/oss/src/lib.rs` |
| Responses 客户端 | `codex-api/src/endpoint/responses.rs` |
| SSE 解析 | `codex-api/src/sse/responses.rs` |
| Core ModelClient | `core/src/client.rs` |
| 记忆启动 | `memories/write/src/start.rs` |
| Phase 1 提取 | `memories/write/src/phase1.rs` |
| Phase 2 整合 | `memories/write/src/phase2.rs` |
| 记忆读取 | `memories/read/src/lib.rs` |
| 记忆 MCP | `memories/mcp/src/server.rs` |
