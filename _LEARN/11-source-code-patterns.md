# 11 - 源码模式与实战示例

## 核心设计模式

本文档通过实际源码展示 Codex 中的关键实现模式。

---

## 模式1: 工具执行器 (ToolExecutor Trait)

每个工具实现 `ToolExecutor<ToolInvocation>` trait:

```rust
// codex-rs/core/src/tools/handlers/shell/shell_command.rs

#[async_trait::async_trait]
impl ToolExecutor<ToolInvocation> for ShellCommandHandler {
    fn tool_name(&self) -> ToolName {
        ToolName::plain("shell_command")
    }

    fn spec(&self) -> ToolSpec {
        create_shell_command_tool(CommandToolOptions { /* ... */ })
    }

    fn supports_parallel_tool_calls(&self) -> bool {
        true
    }

    async fn handle(
        &self,
        invocation: ToolInvocation,
    ) -> Result<Box<dyn ToolOutput>, FunctionCallError> {
        let ToolInvocation {
            session, turn, tracker, call_id, payload, ..
        } = invocation;

        // 1. 解析参数
        let params: ShellCommandToolCallParams = parse_arguments_with_base_path(&arguments, &cwd)?;

        // 2. 委托到共享执行管道
        run_exec_like(RunExecLikeArgs {
            tool_name,
            exec_params,
            session,
            turn,
            tracker,
            call_id,
            shell_runtime_backend: self.shell_runtime_backend(),
            // ...
        })
        .await
        .map(boxed_tool_output)
    }
}
```

### 关键要点

- **trait 提供元数据** (`tool_name`, `spec`) 和执行 (`handle`)
- **参数解析与执行分离** — handler 解析，共享管道执行
- **并行支持声明** — `supports_parallel_tool_calls()` 控制并发

---

## 模式2: 共享执行管道 (run_exec_like)

所有类 Shell 工具共享的执行路径:

```rust
// codex-rs/core/src/tools/handlers/shell.rs

async fn run_exec_like(args: RunExecLikeArgs) -> Result<FunctionToolOutput, FunctionCallError> {
    // 1. 环境检查
    let Some(turn_environment) = turn.environments.primary() else {
        return Err(FunctionCallError::RespondToModel("shell is unavailable".into()));
    };

    // 2. 创建事件发射器 (UI 可见)
    let emitter = ToolEmitter::shell(exec_params.command.clone(), exec_params.cwd.clone(), source);

    // 3. 评估执行策略
    let exec_approval_requirement = session.services.exec_policy
        .create_exec_approval_requirement_for_command(ExecApprovalRequest {
            command: &exec_params.command,
            approval_policy: turn.approval_policy.value(),
            permission_profile: turn.permission_profile(),
            file_system_sandbox_policy: &file_system_sandbox_policy,
            // ...
        })
        .await;

    // 4. 构建请求并通过 Orchestrator 执行
    let req = ShellRequest { /* ... */ exec_approval_requirement, /* ... */ };
    let mut orchestrator = ToolOrchestrator::new();
    let mut runtime = ShellRuntime::for_shell_command(shell_runtime_backend);

    let out = orchestrator
        .run(&mut runtime, &req, &tool_ctx, &turn, turn.approval_policy.value())
        .await
        .map(|result| result.output);

    // 5. 完成事件
    emitter.finish(/* ... */);
    out
}
```

### 执行链路

```
Handler → run_exec_like → ExecPolicy评估 → Orchestrator.run → Runtime执行
                                ↓
                        ExecApprovalRequirement
                        (Skip/NeedsApproval/Forbidden)
```

---

## 模式3: Turn 循环 (Agent 采样循环)

```rust
// codex-rs/core/src/session/turn.rs

pub(crate) async fn run_turn(
    sess: Arc<Session>,
    turn_context: Arc<TurnContext>,
    input: Vec<TurnInput>,
    cancellation_token: CancellationToken,
) -> Option<String> {
    let mut client_session = sess.services.model_client.new_session();

    loop {
        // 1. 排空待处理输入 (steerable turns)
        let pending_input = if can_drain_pending_input {
            sess.input_queue.get_pending_input(&sess.active_turn).await
        } else {
            Vec::new()
        };

        // 2. 执行 hooks 并记录输入
        if run_hooks_and_record_inputs(&sess, &turn_context, &pending_input).await {
            break;
        }

        // 3. 构建 prompt (完整历史)
        let sampling_request_input: Vec<ResponseItem> = {
            sess.clone_history().await
                .for_prompt(&turn_context.model_info.input_modalities)
        };

        // 4. 发送采样请求 (包含工具执行)
        match run_sampling_request(
            Arc::clone(&sess),
            Arc::clone(&turn_context),
            sampling_request_input.clone(),
            cancellation_token.child_token(),
        ).await {
            Ok(output) => {
                let needs_follow_up = model_needs_follow_up || has_pending_input;
                if !needs_follow_up {
                    break; // Turn 完成
                }
                continue; // 继续循环
            }
            Err(CodexErr::TurnAborted) => break,
            Err(e) => { /* 发送错误事件, break */ }
        }
    }

    last_agent_message
}
```

### 核心循环不变式

1. **每次迭代** = 一次 LLM 调用 + 可能的工具执行
2. **Steerable** — 循环中可接收新用户输入
3. **Break 条件** — 模型完成 / 用户中断 / 错误
4. **Continue 条件** — 模型需要后续调用 (工具输出) 或有待处理输入

---

## 模式4: ExecPolicy 评估

```rust
// codex-rs/core/src/exec_policy.rs

pub(crate) async fn create_exec_approval_requirement_for_command(
    &self,
    req: ExecApprovalRequest<'_>,
) -> ExecApprovalRequirement {
    // 1. 解析命令为多段
    let ExecPolicyCommands { commands, used_complex_parsing, command_origin }
        = commands_for_exec_policy(command);

    // 2. 定义未匹配时的回退策略
    let exec_policy_fallback = |cmd: &[String]| {
        render_decision_for_unmatched_command(cmd, UnmatchedCommandContext {
            approval_policy,
            permission_profile,
            file_system_sandbox_policy,
            // ...
        })
    };

    // 3. 针对规则集匹配所有命令段
    let evaluation = exec_policy.check_multiple_with_options(
        commands.iter(),
        &exec_policy_fallback,
        &MatchOptions { resolve_host_executables: true },
    );

    // 4. 映射决策到审批需求
    match evaluation.decision {
        Decision::Forbidden => ExecApprovalRequirement::Forbidden { reason },
        Decision::Prompt => ExecApprovalRequirement::NeedsApproval { reason, amendment },
        Decision::Allow => ExecApprovalRequirement::Skip {
            bypass_sandbox: /* 所有段都是 Allow 规则时 */,
            proposed_execpolicy_amendment,
        },
    }
}
```

### 决策映射

```
命令 → 解析为段 → 每段匹配规则
  ↓
forbid 匹配 → Forbidden (直接拒绝)
prompt 匹配 → NeedsApproval (请求审批)
allow 匹配  → Skip (可能绕过沙箱)
无匹配      → 回退策略 (基于 profile 和启发式)
```

---

## 模式5: 沙箱变换 (Platform Dispatch)

```rust
// codex-rs/sandboxing/src/manager.rs

pub fn transform(
    &self,
    request: SandboxTransformRequest<'_>,
) -> Result<SandboxExecRequest, SandboxTransformError> {
    // 1. 计算有效权限 (合并 additional_permissions)
    let effective_permission_profile =
        effective_permission_profile(permissions, additional_permissions.as_ref());
    let (effective_file_system_policy, effective_network_policy) =
        effective_permission_profile.to_runtime_permissions();

    // 2. 构建命令 argv
    let mut argv = Vec::with_capacity(1 + command.args.len());
    argv.push(command.program);
    argv.extend(command.args.into_iter().map(OsString::from));

    // 3. 平台特定包装
    let (argv, arg0_override) = match sandbox {
        SandboxType::None => (os_argv_to_strings(argv), None),

        #[cfg(target_os = "macos")]
        SandboxType::MacosSeatbelt => {
            // sandbox-exec -p <policy> -- <command>
            (full_command, None)
        }

        SandboxType::LinuxSeccomp => {
            // codex-linux-sandbox -s <policies> -- <command>
            let exe = codex_linux_sandbox_exe
                .ok_or(SandboxTransformError::MissingLinuxSandboxExecutable)?;
            (full_command, Some(linux_sandbox_arg0_override(exe)))
        }

        SandboxType::WindowsRestrictedToken => (os_argv_to_strings(argv), None),
    };

    // 4. 返回变换后的请求
    Ok(SandboxExecRequest {
        command: argv,
        cwd: command.cwd,
        env: command.env,
        sandbox,
        permission_profile: effective_permission_profile,
        file_system_sandbox_policy: effective_file_system_policy,
        network_sandbox_policy: effective_network_policy,
        arg0: arg0_override,
    })
}
```

### 变换结果

| 平台 | 输入命令 | 输出命令 |
|------|----------|----------|
| None | `ls -la` | `ls -la` |
| macOS | `ls -la` | `/usr/bin/sandbox-exec -p <policy> -- ls -la` |
| Linux | `ls -la` | `codex-linux-sandbox -s <opts> -- ls -la` |

---

## 模式6: 配置层叠加载

```rust
// codex-rs/config/src/loader/mod.rs

pub async fn load_config_layers_state(
    fs: &dyn ExecutorFileSystem,
    codex_home: &Path,
    cwd: Option<AbsolutePathBuf>,
    cli_overrides: &[(String, TomlValue)],
    options: impl Into<ConfigLoadOptions>,
) -> io::Result<ConfigLayerStack> {

    // 1. 托管需求 (最低优先级但有约束力)
    if !ignore_managed_requirements {
        if let Some(requirements) = cloud_requirements.get().await? {
            merge_requirements_with_remote_sandbox_config(
                &mut config_requirements_toml,
                RequirementSource::CloudRequirements,
                requirements,
            );
        }
    }

    // 2. 系统层 (/etc/codex/config.toml)
    let system_layer = load_config_toml_for_required_layer(system_path).await?;
    layers.push(system_layer);

    // 3. 用户层 ($CODEX_HOME/config.toml)
    let base_user_layer = load_user_config_layer(fs, &base_user_file).await?;
    layers.push(base_user_layer);

    // 4. Profile 层 ($CODEX_HOME/<name>.config.toml)
    if active_user_file != base_user_file {
        layers.push(load_user_config_layer(fs, &active_user_file).await?);
    }

    // 5. 项目层 (信任门控)
    if let Some(cwd) = cwd {
        let project_layers = load_project_layers(
            fs, &cwd, &project_root, &trust_context, codex_home,
        ).await?;
        layers.extend(project_layers.layers);
    }

    // 6. CLI 覆盖 (最高优先级)
    if !cli_overrides.is_empty() {
        layers.push(build_cli_overrides_layer(cli_overrides));
    }

    Ok(ConfigLayerStack::new(layers, requirements))
}
```

### 层优先级 (从高到低)

```
CLI 参数         ← 最高优先级
Profile 层
项目 config      ← 需要信任验证
用户 config
系统 config
托管需求         ← 最低优先级 (但可约束上层)
```

---

## 模式7: TUI 渲染 (HistoryCell)

```rust
// codex-rs/tui/src/history_cell/messages.rs

impl HistoryCell for UserHistoryCell {
    fn display_lines(&self, width: u16) -> Vec<Line<'static>> {
        let wrap_width = width
            .saturating_sub(LIVE_PREFIX_COLS + 1)
            .max(1);

        let style = user_message_style();

        // 1. 自适应换行
        let wrapped = adaptive_wrap_lines(
            message.split('\n').map(|line| Line::from(line).style(style)),
            RtOptions::new(usize::from(wrap_width))
                .wrap_algorithm(textwrap::WrapAlgorithm::FirstFit),
        );

        // 2. 前缀装饰
        let mut lines: Vec<Line<'static>> = vec![Line::from("").style(style)];
        lines.extend(prefix_lines(
            wrapped,
            "› ".bold().dim(),       // 首行前缀
            "  ".into(),             // 续行前缀
        ));
        lines.push(Line::from("").style(style));
        lines
    }
}
```

### TUI 渲染约定

```rust
// ✓ Stylize trait helpers
"text".dim()
"text".red()
"text".bold().cyan()
vec!["M".red(), " ".dim(), "file.rs".dim()].into()

// ✓ 自适应换行
adaptive_wrap_lines(lines, RtOptions::new(width));

// ✓ 前缀装饰
prefix_lines(lines, first_prefix, subsequent_prefix);

// ✗ 避免
Span::styled("text", Style::default().fg(Color::Red))
"text".white()  // 不硬编码白色
```

---

## 模式8: Queue-Pair 并发

```rust
// codex-rs/core/src/session/mod.rs (概念性)

pub struct Codex {
    tx_sub: mpsc::Sender<Submission>,
    rx_event: mpsc::Receiver<Event>,
}

impl Codex {
    pub fn spawn(config: Config) -> Self {
        let (tx_sub, rx_sub) = mpsc::channel(32);
        let (tx_event, rx_event) = mpsc::channel(128);

        // 单一事件循环 — 所有状态变更在此序列化
        tokio::spawn(async move {
            let mut session = Session::new(config, tx_event.clone());
            while let Some(submission) = rx_sub.recv().await {
                session.handle_submission(submission).await;
            }
        });

        Codex { tx_sub, rx_event }
    }
}
```

### 并发保证

- **单一写入者** — `submission_loop` 是唯一修改 Session 状态的地方
- **无锁读取** — 客户端通过 event channel 获取状态更新
- **背压** — channel 容量限制防止内存无限增长
- **取消传播** — `CancellationToken` 跨 turn/工具传播

---

## 模式9: 事件发射器 (ToolEmitter)

```rust
// 工具执行时的事件通知模式

// 1. 创建发射器 (开始事件)
let emitter = ToolEmitter::shell(command.clone(), cwd.clone(), source);
emitter.begin(&session).await;

// 2. 执行过程中发送增量
emitter.output_delta(&session, OutputDelta::Stdout(chunk)).await;

// 3. 完成时发送结束事件
emitter.finish(&session, exit_code, duration).await;
```

UI 端接收:
- `ExecCommandBegin` → 开始显示命令
- `ExecCommandOutputDelta` → 流式显示输出
- `ExecCommandEnd` → 标记完成 + 显示退出码

---

## 模式10: 错误处理分层

```rust
// 工具层: 错误分为 "告知模型" vs "硬失败"
enum FunctionCallError {
    // 模型可以从这个错误中恢复 (作为 tool output 返回)
    RespondToModel(String),
    // 严重错误，Turn 可能中止
    Hard(anyhow::Error),
}

// 核心层: 控制 Turn 循环行为
enum CodexErr {
    Stream(StreamError),          // 可重试 → 循环继续
    TurnAborted,                  // 中断 → 循环退出
    ContextWindowExceeded,        // 触发压缩
    Sandbox(SandboxErr),          // 沙箱错误
    Fatal(String),                // 不可恢复
}

// 使用模式:
match run_sampling_request(...).await {
    Ok(output) => { /* 正常处理 */ }
    Err(CodexErr::Stream(_)) => { /* 重试 */ }
    Err(CodexErr::TurnAborted) => { break; }
    Err(e) => { send_error_event(e); break; }
}
```

---

## 模式总结

| 模式 | 位置 | 目的 |
|------|------|------|
| ToolExecutor trait | `tools/handlers/` | 统一工具接口 |
| 共享执行管道 | `tools/handlers/shell.rs` | 策略+沙箱+编排复用 |
| Turn 循环 | `session/turn.rs` | Agent 采样不变式 |
| ExecPolicy 评估 | `exec_policy.rs` | 命令安全决策 |
| 沙箱变换 | `sandboxing/manager.rs` | 平台无关包装 |
| 配置层叠 | `config/loader/mod.rs` | 多源合并+信任门控 |
| HistoryCell 渲染 | `tui/history_cell/` | 可组合终端UI |
| Queue-Pair | `session/mod.rs` | 无锁并发模型 |
| ToolEmitter | `tools/` | 增量UI通知 |
| 分层错误 | 跨模块 | 可恢复 vs 终止 |
