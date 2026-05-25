# 01 - 产品概览

## 产品定位

Codex CLI 是 OpenAI 推出的**本地 AI 编程代理**（Local Coding Agent）。它运行在开发者的机器上，能够：

- 阅读和理解代码库
- 编写和修改代码
- 执行 Shell 命令
- 通过 MCP 协议连接外部工具
- 与 IDE 深度集成

### 核心价值主张

```mermaid
mindmap
  root((Codex CLI))
    本地优先
      数据不离开本机
      离线可用(本地模型)
      低延迟响应
    安全可控
      OS级沙箱隔离
      显式审批机制
      可审计操作记录
    多模式交互
      交互式TUI
      无头批处理
      IDE内嵌
      MCP工具接口
    开放集成
      MCP协议
      TypeScript SDK
      Python SDK
      插件系统
```

## 用户画像

| 用户类型 | 使用场景 | 首选模式 |
|----------|----------|----------|
| **终端开发者** | 日常编码、代码审查、重构 | 交互式 TUI |
| **CI/CD 工程师** | 自动化代码生成、测试、审查 | `codex exec` 无头模式 |
| **IDE 用户** | VS Code / Cursor 内使用 | App Server + SDK |
| **工具开发者** | 构建 AI 驱动的开发工具 | MCP Server / SDK |

## 核心功能矩阵

### 三种运行模式

```mermaid
graph LR
    subgraph "交互式 TUI"
        T1[实时对话]
        T2[流式输出]
        T3[审批交互]
        T4[会话恢复]
    end

    subgraph "无头 Exec"
        E1[单次执行]
        E2[JSON输出]
        E3[管道集成]
        E4[代码审查]
    end

    subgraph "App Server"
        A1[JSON-RPC]
        A2[多线程管理]
        A3[WebSocket]
        A4[SDK绑定]
    end
```

### 功能详表

| 功能 | 描述 | 实现位置 |
|------|------|----------|
| **代码生成** | 根据自然语言生成代码 | `codex-core` → LLM |
| **代码编辑** | Apply Patch 工具修改文件 | `codex-apply-patch` |
| **Shell 执行** | 在沙箱内运行命令 | `codex-core/tools/shell` |
| **MCP 工具** | 连接外部 MCP 服务器 | `codex-mcp` |
| **代码审查** | `codex review` / `codex exec review` | `codex-exec` |
| **会话管理** | 持久化、恢复、分叉 | `codex-state`, `codex-rollout` |
| **上下文压缩** | 自动/手动上下文窗口管理 | `codex-core/compact` |
| **多模型支持** | OpenAI / Ollama / LM Studio | `codex-model-provider` |
| **插件系统** | 可扩展的工具和行为 | `codex-plugin` |
| **Skills** | 可复用的知识和能力 | `codex-skills` |
| **记忆系统** | 跨会话知识积累 | `codex-memories-*` |

## 产品价值分析

### 与传统开发工具对比

```mermaid
quadrantChart
    title 开发工具定位象限
    x-axis "手动操作" --> "自动化"
    y-axis "云端" --> "本地"
    quadrant-1 "本地自动化 ⭐"
    quadrant-2 "本地手动"
    quadrant-3 "云端手动"
    quadrant-4 "云端自动化"
    "Codex CLI": [0.85, 0.9]
    "GitHub Copilot": [0.6, 0.3]
    "ChatGPT": [0.5, 0.1]
    "传统IDE": [0.2, 0.8]
    "CI/CD": [0.9, 0.2]
    "Cursor": [0.7, 0.6]
```

### 核心差异化优势

1. **本地优先 (Local-First)**
   - 代码不上传到云端（使用本地模型时）
   - 完整的文件系统访问（受沙箱控制）
   - 低延迟的命令执行

2. **安全沙箱 (Sandboxed Execution)**
   - OS 级隔离（Seatbelt/bubblewrap/seccomp）
   - 精细的文件系统权限
   - 网络访问控制
   - 操作审批流程

3. **协议开放 (Protocol-First)**
   - MCP 协议双向支持
   - App Server JSON-RPC 协议
   - TypeScript / Python SDK

4. **模式灵活 (Multi-Modal)**
   - TUI 交互 → 日常开发
   - 无头执行 → CI/CD 自动化
   - IDE 集成 → 无缝体验

## 技术栈选择

| 层次 | 技术 | 理由 |
|------|------|------|
| **核心语言** | Rust | 性能、安全、跨平台 |
| **TUI 框架** | Ratatui + Crossterm | 成熟的终端 UI 生态 |
| **异步运行时** | Tokio | Rust 异步标准 |
| **持久化** | SQLite | 轻量、嵌入式、零配置 |
| **配置格式** | TOML | Rust 生态标准 |
| **构建系统** | Cargo + Bazel | 开发效率 + 发布可靠性 |
| **分发** | npm + Homebrew | 覆盖主流安装方式 |
| **协议** | JSON-RPC (lite) | 简单、双向、IDE 友好 |

## 安装与分发

```mermaid
graph TD
    subgraph "安装方式"
        A["npm i -g @openai/codex"]
        B["brew install codex"]
        C["curl 安装脚本"]
        D["源码编译"]
    end

    subgraph "npm 包结构"
        E["@openai/codex<br/>(JS Launcher)"]
        F["@openai/codex-linux-x64"]
        G["@openai/codex-darwin-arm64"]
        H["@openai/codex-win32-x64"]
    end

    A --> E
    E -->|"按平台选择"| F
    E -->|"按平台选择"| G
    E -->|"按平台选择"| H

    F --> I["原生 Rust 二进制"]
    G --> I
    H --> I
```

npm 包 `@openai/codex` 是一个**薄启动器**——真正的实现是平台特定的 Rust 编译二进制文件。

## 配置系统

```
~/.codex/
├── config.toml          # 用户级配置
├── instructions.md      # 全局系统指令
├── sessions/            # 会话持久化
├── skills/              # 用户 Skills
├── rules/               # 执行策略规则
└── state.db             # SQLite 状态数据库

<项目>/.codex/
├── config.toml          # 项目级配置
├── instructions.md      # 项目级指令
└── skills/              # 项目 Skills
```

### 配置层叠模型

```mermaid
graph BT
    A["CLI 参数 (最高优先级)"] --> FINAL[最终配置]
    B["环境变量"] --> FINAL
    C["项目 config.toml"] --> FINAL
    D["用户 config.toml"] --> FINAL
    E["托管需求 (Managed)"] --> FINAL
    F["默认值 (最低优先级)"] --> FINAL
```

## 关键指标

| 指标 | 数据 |
|------|------|
| Rust Crate 数量 | 113 |
| 源码文件 (Rust) | ~2000+ |
| 支持平台 | Linux (x64/arm64), macOS (arm64/x64), Windows |
| 沙箱后端 | 3 (Seatbelt, bubblewrap+seccomp, Windows Restricted Token) |
| 模型提供商 | OpenAI, Ollama, LM Studio, 自定义 |
| 协议版本 | App Server v2 (活跃开发) |
