# 09 - 插件、Skills 与 Hooks 系统

## 整体架构

```mermaid
graph TD
    subgraph "Marketplace / 安装"
        MKT["marketplace.json<br/>(插件目录)"]
        STORE["PluginStore<br/>$CODEX_HOME/plugins/cache/"]
        INSTALL["install/uninstall"]
    end

    subgraph "加载 (Runtime)"
        PM["PluginsManager<br/>(缓存 + 加载)"]
        SM["SkillsManager<br/>(发现 + 缓存)"]
        HE["ClaudeHooksEngine<br/>(发现 + 派发)"]
    end

    subgraph "集成到 Session"
        PROMPT["Prompt 注入<br/>(Skills 元数据 + 插件能力)"]
        TOOLS["MCP 工具<br/>(插件提供)"]
        HOOKS_EX["Hook 执行<br/>(生命周期事件)"]
    end

    MKT --> STORE
    STORE --> PM
    PM --> SM
    PM --> HE
    PM --> TOOLS

    SM --> PROMPT
    HE --> HOOKS_EX
```

## 插件系统 (Plugin)

### 插件是什么？

插件是一个**文件系统目录**，包含：

```
my-plugin/
├── .codex-plugin/
│   └── plugin.json          # 插件清单 (必须)
├── skills/                   # 可选: 插件提供的 Skills
│   └── my-skill/
│       └── SKILL.md
├── .mcp.json                 # 可选: MCP 服务器配置
├── .app.json                 # 可选: App 连接器
└── hooks/
    └── hooks.json            # 可选: 生命周期 Hooks
```

### 插件生命周期

```mermaid
sequenceDiagram
    participant User as 用户
    participant PM as PluginsManager
    participant Store as PluginStore
    participant Config as config.toml

    Note over PM: 启动时
    PM->>PM: maybe_start_plugin_startup_tasks()
    PM->>PM: 同步 curated marketplace
    PM->>PM: 刷新 Git 插件缓存
    PM->>PM: 远程同步已安装插件

    Note over User: 安装插件
    User->>PM: install("plugin@marketplace")
    PM->>PM: 解析 MarketplacePluginSource
    alt Local
        PM->>Store: 复制本地路径
    else Git
        PM->>Store: git clone + checkout
    end
    Store->>Store: 原子安装到 cache/
    PM->>Config: 写入 plugins.plugin@marketplace.enabled=true

    Note over PM: 加载插件 (每次 Session)
    PM->>PM: load_plugins_from_layer_stack()
    PM->>PM: 解析 plugin.json → PluginManifest
    PM->>PM: load_plugin_skills()
    PM->>PM: load MCP servers
    PM->>PM: load hooks
    PM->>PM: → PluginLoadOutcome
```

### 核心类型

```rust
// 插件标识
pub struct PluginId {
    pub plugin_name: String,
    pub marketplace_name: String,
}
// 格式: "plugin@marketplace"

// 加载结果
pub struct LoadedPlugin<M> {
    pub config_name: String,
    pub root: AbsolutePathBuf,
    pub enabled: bool,
    pub skill_roots: Vec<AbsolutePathBuf>,
    pub disabled_skill_paths: HashSet<AbsolutePathBuf>,
    pub mcp_servers: HashMap<String, M>,
    pub apps: Vec<AppConnectorId>,
    pub hook_sources: Vec<PluginHookSource>,
}

// 聚合结果 (所有活跃插件合并)
pub struct PluginLoadOutcome {
    // 提供:
    //   effective_plugin_skill_roots() → SkillsManager
    //   effective_mcp_servers() → MCP 连接管理器
    //   effective_apps() → App 连接器
    //   effective_plugin_hook_sources() → Hook 引擎
    //   capability_summaries() → Prompt + 遥测
}
```

### 插件来源

```mermaid
graph TD
    subgraph "MarketplacePluginSource"
        LOCAL["Local { path }"]
        GIT["Git { url, path, ref_name, sha }"]
    end

    subgraph "Marketplace 发现"
        PROJ_MKT[".agents/plugins/marketplace.json"]
        CURATED["OpenAI 策展仓库<br/>$CODEX_HOME/.tmp/plugins/"]
        CONFIG_MKT["config 中配置的 marketplace roots"]
    end

    PROJ_MKT --> LOCAL
    PROJ_MKT --> GIT
    CURATED --> GIT
    CONFIG_MKT --> LOCAL
    CONFIG_MKT --> GIT
```

### 存储结构

```
$CODEX_HOME/plugins/
├── cache/
│   └── <marketplace>/
│       └── <plugin>/
│           └── <version>/      # 安装的插件内容
└── data/
    └── <plugin>-<marketplace>/ # 插件运行时数据
```

## Skills 系统

### Skill 是什么？

Skill 是一个包含 `SKILL.md` 的目录，为 Agent 提供特定知识或能力：

```
my-skill/
├── SKILL.md              # 必须: YAML frontmatter + markdown 正文
├── agents/
│   └── openai.yaml       # 可选: 接口、依赖、策略
├── scripts/              # 可选: 脚本资源
├── references/           # 可选: 参考文档
└── assets/               # 可选: 资产文件
```

### SKILL.md 格式

```markdown
---
name: my-awesome-skill
description: 一个示例 Skill 的描述
metadata:
  short-description: 简短描述 (用于列表显示)
---

# My Awesome Skill

这里是完整的 Skill 指导内容...
当 Agent 需要使用此 Skill 时，整个正文会被注入到上下文中。
```

### Skill 发现与加载

```mermaid
flowchart TD
    subgraph "Skill 根目录 (按优先级)"
        PROJ["项目 skills/<br/>(Repo scope)"]
        AGENTS_DIR[".agents/skills<br/>(项目根→CWD)"]
        USER["$CODEX_HOME/skills<br/>(User scope)"]
        USER_AGENTS["$HOME/.agents/skills<br/>(User scope)"]
        SYSTEM["$CODEX_HOME/skills/.system<br/>(System scope)"]
        ADMIN["/etc/codex/skills<br/>(Admin scope)"]
        PLUGIN["插件 skill_roots<br/>(User scope)"]
    end

    subgraph "加载过程"
        SCAN["BFS 扫描<br/>max depth=6, max dirs=2000"]
        PARSE["解析 SKILL.md<br/>YAML frontmatter"]
        RULES["应用 SkillConfigRules<br/>(启用/禁用)"]
        CACHE["SkillsManager 缓存"]
    end

    PROJ --> SCAN
    AGENTS_DIR --> SCAN
    USER --> SCAN
    USER_AGENTS --> SCAN
    SYSTEM --> SCAN
    ADMIN --> SCAN
    PLUGIN --> SCAN

    SCAN --> PARSE
    PARSE --> RULES
    RULES --> CACHE
```

### 两阶段 Prompt 注入

```mermaid
sequenceDiagram
    participant SM as SkillsManager
    participant Session as Session
    participant Model as 模型
    participant User as 用户

    Note over Session: 阶段1: Thread 启动 (元数据目录)
    Session->>SM: build_available_skills()
    SM-->>Session: Skills 列表 (名称+描述+路径)
    Session->>Session: 注入 "## Skills" developer section
    Note over Session: 预算: ~2% 上下文窗口

    Note over Session: 阶段2: 每次 Turn (完整正文)
    User->>Session: 消息中提到 $skill-name
    Session->>SM: collect_explicit_skill_mentions()
    SM-->>Session: 匹配的 Skill 列表
    Session->>SM: build_skill_injections()
    SM-->>Session: 完整 SKILL.md 内容
    Session->>Model: 注入 SkillInjection 到上下文

    Note over Model: 模型也可以主动读取 Skill 路径
    Model->>Model: 渐进式发现 (读取路径)
```

### Skill 作用域优先级

```
Repo > User > System > Admin
(相同路径时，高优先级作用域胜出)
```

### 配置控制

```toml
# config.toml
[[skills.config]]
name = "dangerous-skill"
enabled = false

[[skills.config]]
path = "/path/to/specific/skill"
enabled = false
```

## Hook 系统

### Hook 是什么？

Hook 是在 Agent 生命周期特定事件触发的 Shell 命令：

```json
// hooks/hooks.json
{
  "hooks": [
    {
      "event": "PreToolUse",
      "matcher": { "tool_name": "shell.*" },
      "command": "./scripts/validate-command.sh",
      "timeout_ms": 5000
    },
    {
      "event": "Stop",
      "command": "notify-send 'Codex' 'Session ended'"
    }
  ]
}
```

### 生命周期事件

```mermaid
graph TD
    subgraph "会话级"
        SS[SessionStart]
        STOP[Stop]
    end

    subgraph "用户输入"
        UPS[UserPromptSubmit]
    end

    subgraph "工具调用"
        PTU[PreToolUse]
        PR[PermissionRequest]
        POST[PostToolUse]
    end

    subgraph "上下文压缩"
        PRE_C[PreCompact]
        POST_C[PostCompact]
    end

    subgraph "子 Agent"
        SAS[SubagentStart]
        SAST[SubagentStop]
    end

    SS --> UPS
    UPS --> PTU
    PTU --> PR
    PR --> POST
    POST --> PRE_C
    PRE_C --> POST_C
    POST_C --> STOP
```

### 10 种事件类型

| 事件 | 触发时机 | 可阻止? |
|------|----------|---------|
| `SessionStart` | 会话/子Agent启动 | 否 |
| `UserPromptSubmit` | 用户消息提交前 | 否 |
| `PreToolUse` | 工具执行前 | **是** |
| `PermissionRequest` | 审批流程中 | **是** (Allow/Deny) |
| `PostToolUse` | 工具执行后 | 否 |
| `PreCompact` | 上下文压缩前 | 否 |
| `PostCompact` | 上下文压缩后 | 否 |
| `SubagentStart` | 子Agent启动 | 否 |
| `SubagentStop` | 子Agent停止 | 否 |
| `Stop` | Turn 结束 | 否 |

### Hook 发现

```mermaid
flowchart TD
    subgraph "来源 (按优先级)"
        MANAGED["托管需求 Hooks (MDM)"]
        CONFIG["配置层 hooks<br/>(hooks/hooks.json 或 TOML)"]
        PLUGIN_H["插件 Hook Sources"]
    end

    subgraph "合并"
        DISCOVER["discover_handlers()"]
        TRUST["信任检查"]
        ENABLE["启用状态 (hook_states)"]
    end

    subgraph "执行"
        SELECT["select_handlers(event)"]
        MATCH["Matcher 过滤"]
        RUN["并发执行"]
        AGGREGATE["有序聚合结果"]
    end

    MANAGED --> DISCOVER
    CONFIG --> DISCOVER
    PLUGIN_H --> DISCOVER

    DISCOVER --> TRUST
    TRUST --> ENABLE
    ENABLE --> SELECT
    SELECT --> MATCH
    MATCH --> RUN
    RUN --> AGGREGATE
```

### Hook 执行机制

```mermaid
sequenceDiagram
    participant Core as codex-core
    participant Engine as ClaudeHooksEngine
    participant Handler as ConfiguredHandler
    participant Shell as Shell 进程

    Core->>Engine: dispatch_event(PreToolUse, context)
    Engine->>Engine: select_handlers(event, matcher)

    par 并发执行所有匹配 handler
        Engine->>Handler: run_command()
        Handler->>Shell: 启动进程 (command)
        Handler->>Shell: stdin ← JSON 上下文
        Shell-->>Handler: stdout → JSON 结果
        Shell-->>Handler: exit code
    end

    Engine->>Engine: 有序聚合结果
    Engine-->>Core: HookOutcome

    alt 结果可阻止工具
        Core->>Core: 阻止执行 / 修改输入
    else 结果注入上下文
        Core->>Core: additional_contexts → Prompt
    end
```

### Hook 结果能力

| 能力 | 适用事件 | 说明 |
|------|----------|------|
| 阻止工具执行 | PreToolUse | 返回 block 决策 |
| 修改工具输入 | PreToolUse | 返回修改后的参数 |
| Allow/Deny 审批 | PermissionRequest | 提前决定审批结果 |
| 注入上下文 | 任意 | `additional_contexts` 字段 |
| 停止会话 | Stop | 清理操作 |

### 插件 Hooks 环境变量

插件提供的 Hook 会获得额外环境变量：

```
PLUGIN_ROOT      → 插件安装根目录
PLUGIN_DATA_ROOT → 插件数据目录
```

## Extension API (编译时扩展)

### 与插件的区别

| 特性 | 插件 (Plugin) | 扩展 (Extension) |
|------|---------------|------------------|
| 形式 | 文件系统目录 | Rust crate |
| 发现 | 运行时 marketplace | 编译时链接 |
| 能力 | Skills + MCP + Hooks | 完整贡献者 trait |
| 用户安装 | 是 | 否 (内置) |
| 示例 | 社区工具 | goal, memories, guardian |

### 贡献者 Traits

```mermaid
classDiagram
    class ContextContributor {
        +contribute() Vec~PromptFragment~
    }

    class ThreadLifecycleContributor {
        +on_thread_start()
        +on_thread_resume()
        +on_thread_stop()
    }

    class TurnLifecycleContributor {
        +on_turn_start()
        +on_turn_stop()
        +on_turn_abort()
    }

    class ToolContributor {
        +tools() Vec~ToolExecutor~
    }

    class ToolLifecycleContributor {
        +on_tool_start()
        +on_tool_finish()
    }

    class ConfigContributor {
        +on_config_changed()
    }

    class TokenUsageContributor {
        +on_token_usage()
    }

    class ApprovalReviewContributor {
        +claim_review_prompt()
    }
```

### 注册模式

```rust
// 在 codex-core 启动时
let mut builder = ExtensionRegistryBuilder::<Config>::new();

// 注册各扩展
memories::install(&mut builder);
goal::install(&mut builder, goal_config);
guardian::install(&mut builder);

let registry = builder.build();
// registry 存储在 SessionServices 中
```

### 内置扩展

| 扩展 | Crate | 功能 |
|------|-------|------|
| **Goal** | `ext/goal` | 线程目标追踪、预算核算、`update_goal` 工具 |
| **Memories** | `ext/memories` | 记忆读取工具、developer prompt 注入 |
| **Guardian** | `ext/guardian` | 通过 AgentSpawner 产生子Agent 审核 |

## 端到端数据流

```mermaid
flowchart TD
    subgraph "启动"
        START[Session 启动]
        START --> PM_LOAD["PluginsManager<br/>加载活跃插件"]
        PM_LOAD --> OUTCOME["PluginLoadOutcome"]

        OUTCOME --> SK_ROOTS["effective_skill_roots()"]
        OUTCOME --> MCP_SERVERS["effective_mcp_servers()"]
        OUTCOME --> HOOK_SRC["effective_plugin_hook_sources()"]
        OUTCOME --> CAPS["capability_summaries()"]

        SK_ROOTS --> SM_LOAD["SkillsManager<br/>发现 + 缓存"]
        HOOK_SRC --> HOOKS["build_hooks_for_config()"]
        MCP_SERVERS --> MCP["MCP 连接管理器"]
    end

    subgraph "Thread 启动"
        SM_LOAD --> AVAIL["build_available_skills()<br/>(元数据目录)"]
        CAPS --> PLUGIN_INST["AvailablePluginsInstructions"]
        AVAIL --> DEV_PROMPT["Developer Prompt<br/>## Skills 部分"]
        PLUGIN_INST --> DEV_PROMPT
    end

    subgraph "每次 Turn"
        USER_MSG["用户消息"] --> MENTIONS["collect_explicit_skill_mentions()"]
        MENTIONS --> INJECT["build_skill_injections()<br/>(完整 SKILL.md)"]
        INJECT --> CONTEXT["Turn 上下文"]

        HOOKS --> PRE["PreToolUse hooks"]
        PRE --> TOOL_EXEC["工具执行"]
        TOOL_EXEC --> POST["PostToolUse hooks"]
    end
```

## 关键源文件

| 模块 | 文件 |
|------|------|
| 插件标识 | `plugin/src/plugin_id.rs` |
| 插件加载结果 | `plugin/src/load_outcome.rs` |
| 插件管理器 | `core-plugins/src/manager.rs` |
| 插件存储 | `core-plugins/src/store.rs` |
| 插件清单 | `core-plugins/src/manifest.rs` |
| Marketplace | `core-plugins/src/marketplace.rs` |
| Skill 模型 | `core-skills/src/model.rs` |
| Skill 加载器 | `core-skills/src/loader.rs` |
| Skill 管理器 | `core-skills/src/manager.rs` |
| Skill 注入 | `core-skills/src/injection.rs` |
| Skill 渲染 | `core-skills/src/render.rs` |
| Hook 引擎 | `hooks/src/engine/mod.rs` |
| Hook 发现 | `hooks/src/engine/discovery.rs` |
| Hook 派发 | `hooks/src/engine/dispatcher.rs` |
| Extension API | `ext/extension-api/src/contributors.rs` |
| Extension 注册 | `ext/extension-api/src/registry.rs` |
| Core Hook 集成 | `core/src/hook_runtime.rs` |
| Core Skill 集成 | `core/src/skills.rs` |
