# 06 - 安全与沙箱

## 安全模型概览

Codex 采用**纵深防御** (Defense-in-Depth) 安全模型：

```mermaid
graph TD
    subgraph "第1层: 声明式权限"
        PP[PermissionProfile]
        FS_POLICY[FileSystemSandboxPolicy]
        NET_POLICY[NetworkSandboxPolicy]
    end

    subgraph "第2层: 策略引擎"
        EXEC_POLICY[ExecPolicy Rules]
        GUARDIAN[Guardian 自动审核]
        HOOKS[Permission Hooks]
    end

    subgraph "第3层: 用户审批"
        APPROVAL[审批流程]
        CACHE[会话级缓存]
    end

    subgraph "第4层: OS 沙箱"
        SEATBELT[macOS Seatbelt]
        BWRAP[Linux bubblewrap]
        SECCOMP[Linux seccomp]
        WIN[Windows Restricted Token]
    end

    subgraph "第5层: 网络控制"
        NET_PROXY[SOCKS5/HTTPS 代理]
        FIREWALL[网络命名空间隔离]
    end

    PP --> EXEC_POLICY
    EXEC_POLICY --> APPROVAL
    APPROVAL --> SEATBELT
    APPROVAL --> BWRAP
    BWRAP --> SECCOMP
    NET_POLICY --> NET_PROXY
    NET_POLICY --> FIREWALL
```

## 权限模型

### PermissionProfile 类型

```mermaid
classDiagram
    class PermissionProfile {
        <<enum>>
    }

    class Managed {
        +file_system: ManagedFileSystemPermissions
        +network: NetworkSandboxPolicy
    }

    class Disabled {
        不应用外部沙箱
    }

    class External {
        +network: NetworkSandboxPolicy
        外部调用者负责文件隔离
    }

    PermissionProfile --> Managed
    PermissionProfile --> Disabled
    PermissionProfile --> External
```

### 内置 Profile 预设

| Profile | 文件系统 | 网络 | 适用场景 |
|---------|----------|------|----------|
| `:read-only` | 只读 | 受限 | 代码分析、审查 |
| `:workspace` | 工作区可写 | 受限 | 日常开发 |
| `:danger-full-access` | 完全访问 | 完全访问 | `--yolo` 模式 |

### 文件系统策略详解

```mermaid
graph TD
    subgraph "FileSystemSandboxPolicy"
        KIND{SandboxKind}
        KIND -->|Restricted| RESTRICTED[受限访问]
        KIND -->|Unrestricted| UNRESTRICTED[不限制]
        KIND -->|ExternalSandbox| EXTERNAL[外部管理]
    end

    subgraph "访问规则 (Restricted)"
        READ[可读路径]
        WRITE[可写路径]
        DENY[拒绝路径]

        READ --> SPECIFICITY[特异性排序]
        WRITE --> SPECIFICITY
        DENY --> SPECIFICITY
        SPECIFICITY --> RESOLVE["解析: deny > write > read"]
    end

    subgraph "特殊路径"
        ROOT[":root → /"]
        PROJECT[":project_roots → 工作区"]
        TMP[":tmpdir → $TMPDIR"]
        SLASH_TMP[":slash_tmp → /tmp"]
    end

    subgraph "保护路径 (始终只读)"
        GIT[".git"]
        CODEX[".codex"]
        AGENTS[".agents"]
    end
```

### 权限冲突解析

```
规则: deny > write > read
特异性: 更具体的路径优先

示例:
  /repo          → write (工作区)
  /repo/.git     → read-only (保护)
  /repo/a        → deny
  /repo/a/b      → write (更具体)

结果:
  /repo/file.rs  → 可写 ✓
  /repo/.git/    → 只读 (强制保护)
  /repo/a/x      → 拒绝 ✗
  /repo/a/b/y    → 可写 ✓ (特异性覆盖)
```

## 沙箱实现

### 平台沙箱对比

```mermaid
graph LR
    subgraph "Linux"
        direction TB
        L1[bubblewrap<br/>文件系统隔离]
        L2[seccomp<br/>系统调用过滤]
        L3[命名空间<br/>PID/NET/USER]
        L1 --> L2 --> L3
    end

    subgraph "macOS"
        direction TB
        M1[Seatbelt<br/>sandbox-exec]
        M2[.sbpl 策略文件]
        M1 --> M2
    end

    subgraph "Windows"
        direction TB
        W1[Restricted Token<br/>降权进程]
        W2[Elevated Backend<br/>精确ACL]
        W1 --> W2
    end
```

### Linux 沙箱 (bubblewrap + seccomp)

```mermaid
sequenceDiagram
    participant Core as codex-core
    participant SM as SandboxManager
    participant BWRAP as bubblewrap
    participant SECCOMP as seccomp stage
    participant CMD as 用户命令

    Core->>SM: transform(command, policy)
    SM->>SM: 构建 bwrap 参数

    Note over SM: 文件系统挂载
    SM->>SM: --ro-bind / / (只读根)
    SM->>SM: --bind /workspace /workspace (可写)
    SM->>SM: --ro-bind .git .git (保护)
    SM->>SM: /dev/null → 拒绝路径

    Note over SM: 命名空间
    SM->>SM: --unshare-user --unshare-pid
    SM->>SM: --unshare-net (如果网络受限)

    SM->>BWRAP: 启动外层 (文件系统视图)
    BWRAP->>SECCOMP: --apply-seccomp-then-exec

    Note over SECCOMP: 安全加固
    SECCOMP->>SECCOMP: PR_SET_NO_NEW_PRIVS
    SECCOMP->>SECCOMP: 安装 seccomp 过滤器
    SECCOMP->>CMD: execvp(用户命令)
```

### seccomp 过滤策略

| 模式 | 允许 | 阻止 | 用途 |
|------|------|------|------|
| **Restricted** | AF_UNIX 套接字 | connect, bind, listen (网络) | 默认网络限制 |
| **ProxyRouted** | AF_INET/AF_INET6 | AF_UNIX (除预批) | 强制流量走代理 |
| **通用** | - | ptrace, io_uring, process_vm_* | 所有模式 |

### macOS Seatbelt

```mermaid
flowchart TD
    POLICY_GEN[策略生成] --> BASE["seatbelt_base_policy.sbpl<br/>(deny-default)"]
    POLICY_GEN --> NET["seatbelt_network_policy.sbpl<br/>(可选)"]
    POLICY_GEN --> DYNAMIC["动态规则<br/>(从 FileSystemSandboxPolicy)"]

    BASE --> MERGE[合并策略]
    NET --> MERGE
    DYNAMIC --> MERGE

    MERGE --> EXEC["/usr/bin/sandbox-exec -p 策略"]
    EXEC --> CHILD["子进程<br/>CODEX_SANDBOX=seatbelt"]
```

Seatbelt 基础策略允许：
- 进程 exec/fork
- PTY 操作
- cfprefs 读取
- 基本 sysctl

### 保护路径机制

所有平台统一保护以下路径不被 Agent 写入：

```
.git/          → 防止仓库篡改
.codex/        → 防止配置注入 (权限提升)
.agents/       → 防止规则篡改
gitdir: 引用    → 防止 worktree git 篡改
```

## 审批系统

### 审批流程

```mermaid
flowchart TD
    TOOL_CALL[工具调用请求] --> POLICY[ExecPolicy 评估]

    POLICY --> DECISION{策略决策}
    DECISION -->|Allow| SKIP_APPROVAL[跳过审批]
    DECISION -->|Prompt| NEED_APPROVAL[需要审批]
    DECISION -->|Forbidden| BLOCK[直接阻止]

    NEED_APPROVAL --> CACHE{会话缓存?}
    CACHE -->|命中| CACHED_APPROVE[使用缓存决策]
    CACHE -->|未命中| HOOKS_CHECK[Permission Hooks]

    HOOKS_CHECK --> HOOK_RESULT{Hook 结果}
    HOOK_RESULT -->|Allow| APPROVE
    HOOK_RESULT -->|Deny| DENY
    HOOK_RESULT -->|Pass| GUARDIAN_CHECK[Guardian 审核]

    GUARDIAN_CHECK --> GUARDIAN_RESULT{Guardian 结果}
    GUARDIAN_RESULT -->|Approve| APPROVE
    GUARDIAN_RESULT -->|Deny| DENY
    GUARDIAN_RESULT -->|Timeout| DENY
    GUARDIAN_RESULT -->|NoGuardian| USER_PROMPT[用户审批]

    USER_PROMPT --> USER_RESULT{用户决策}
    USER_RESULT -->|Approve| APPROVE[批准]
    USER_RESULT -->|ApproveForSession| SESSION_CACHE[批准+缓存]
    USER_RESULT -->|Deny| DENY[拒绝]
    USER_RESULT -->|Abort| ABORT[中止 Turn]

    SKIP_APPROVAL --> EXEC[沙箱内执行]
    APPROVE --> EXEC
    SESSION_CACHE --> EXEC
    CACHED_APPROVE --> EXEC
```

### AskForApproval 策略

```mermaid
graph TD
    subgraph "审批策略 (AskForApproval)"
        UT["UnlessTrusted<br/>除非已知安全，否则询问"]
        OF["OnFailure<br/>(已弃用) 失败时升级"]
        OR["OnRequest<br/>(默认) 模型决定何时询问"]
        GR["Granular<br/>按类别精细控制"]
        NV["Never<br/>从不询问"]
    end

    subgraph "Granular 配置"
        GR --> SB_A[sandbox_approval]
        GR --> RULES_A[rules_approval]
        GR --> SKILLS_A[skills_approval]
        GR --> MCP_A[mcp_approval]
        GR --> NET_A[network_approval]
    end
```

### Guardian 自动审核

Guardian 是一个内置的 LLM 审核器，用于自动判断操作安全性：

```mermaid
flowchart LR
    REQUEST[审批请求] --> GUARDIAN[Guardian LLM]
    GUARDIAN --> POLICY["安全策略检查<br/>(policy.md)"]

    POLICY --> CHECK1[数据泄露?]
    POLICY --> CHECK2[凭据探测?]
    POLICY --> CHECK3[安全削弱?]
    POLICY --> CHECK4[破坏性操作?]

    CHECK1 --> VERDICT{判决}
    CHECK2 --> VERDICT
    CHECK3 --> VERDICT
    CHECK4 --> VERDICT

    VERDICT -->|安全| APPROVE[自动批准]
    VERDICT -->|可疑| ESCALATE[升级到用户]
    VERDICT -->|超时 90s| DENY[拒绝]
```

## ExecPolicy 策略引擎

### 规则文件格式

```starlark
# ~/.codex/rules/default.rules
# Starlark 语法的前缀规则匹配

allow("git status")
allow("git diff")
allow("git log")
allow("ls")
allow("cat")

prompt("rm")
prompt("git push")
prompt("curl")

forbid("rm -rf /")
forbid("sudo")
```

### 评估流程

```mermaid
flowchart TD
    CMD["命令: git push origin main"] --> PARSE[解析 Shell 命令]
    PARSE --> LOWER["bash -c 降级处理"]
    LOWER --> MATCH[前缀匹配规则]

    MATCH --> CHECK{匹配结果}
    CHECK -->|forbid| BLOCK[Forbidden]
    CHECK -->|allow| ALLOW[Allow]
    CHECK -->|prompt| PROMPT[需要审批]
    CHECK -->|无匹配| DEFAULT[默认策略]

    DEFAULT --> HEURISTIC{启发式检查}
    HEURISTIC -->|known_safe| ALLOW
    HEURISTIC -->|might_be_dangerous| PROMPT
    HEURISTIC -->|unknown| PROFILE["根据 PermissionProfile 决定"]
```

## 沙箱失败与升级

### 升级流程

```mermaid
sequenceDiagram
    participant Orch as Orchestrator
    participant SB as 沙箱
    participant User as 用户

    Orch->>SB: 沙箱内执行命令
    SB-->>Orch: 失败 (SandboxErr::Denied)

    Orch->>Orch: is_likely_sandbox_denied()?
    Note over Orch: 检测关键词:<br/>permission denied<br/>operation not permitted<br/>read-only file system<br/>SIGSYS (seccomp)

    alt 可升级 & 策略允许
        Orch->>User: 重新请求审批 (含失败原因)
        User-->>Orch: 批准升级
        Orch->>Orch: 无沙箱重试
    else 不可升级
        Orch->>Orch: 返回错误给模型
    end
```

### 沙箱拒绝检测

检测以下信号判断是否为沙箱拒绝：

| 平台 | 信号 |
|------|------|
| 通用 | "operation not permitted", "permission denied" |
| 通用 | "read-only file system" |
| Linux | SIGSYS (exit code 159), "seccomp", "landlock" |
| macOS | "sandbox" in stderr |
| 排除 | exit codes 2, 126, 127 (正常命令错误) |

## 网络安全

### 网络限制层次

```mermaid
graph TD
    subgraph "策略层"
        NP[NetworkSandboxPolicy]
        NP -->|Restricted| R[网络受限]
        NP -->|Enabled| E[网络开放]
    end

    subgraph "强制层 (Linux)"
        R --> NS[网络命名空间隔离<br/>--unshare-net]
        R --> SC[seccomp 过滤<br/>阻止 connect/bind]
    end

    subgraph "强制层 (macOS)"
        R --> SB[Seatbelt 网络规则<br/>deny network*]
    end

    subgraph "代理层"
        PROXY[Managed Network Proxy]
        PROXY --> SOCKS5["SOCKS5 代理<br/>(TCP 转发)"]
        PROXY --> HTTPS["HTTPS MITM<br/>(TLS 检查)"]
        PROXY --> ALLOW["主机白名单"]
        PROXY --> DENY_HOST["主机黑名单"]
    end

    R --> PROXY
```

### 托管网络代理

当 `has_managed_network_requirements = true` 时：

1. 启动内部网络代理
2. 所有沙箱内流量被路由到代理
3. 代理按白名单决定放行/拒绝
4. 即使 `DangerFullAccess` profile 也强制代理
5. 用户可持久化 allow/deny 规则

## 环境标记

| 环境变量 | 含义 | 设置时机 |
|----------|------|----------|
| `CODEX_SANDBOX=seatbelt` | macOS Seatbelt 沙箱中 | 子进程 |
| `CODEX_SANDBOX_NETWORK_DISABLED=1` | 网络已禁用 | 子进程 + 测试 |
| `CODEX_LINUX_SANDBOX_WORKSPACE_ROOT` | Linux 沙箱工作区根 | bwrap 内 |

测试中利用这些标记来跳过无法在沙箱中运行的测试。

## 安全设计原则

1. **Fail-Closed** — 默认拒绝，异常情况（Guardian 超时等）也拒绝
2. **先沙箱后升级** — 总是先在沙箱中尝试，失败后才请求升级
3. **保护不可逆操作** — `.git`、`.codex` 等关键路径永远受保护
4. **分层校验** — 策略引擎 → Hooks → Guardian → 用户，任一层可阻止
5. **会话级缓存** — 批准决策在会话内缓存，避免重复打扰
6. **最小权限** — seccomp 阻止不需要的系统调用 (ptrace, io_uring)
