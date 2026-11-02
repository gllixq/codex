# Codex 代码报告

这份文档根据当前工作区源码整理，说明 Codex CLI 仓库怎么组织，以及一次对话从进程启动走到模型、工具和沙箱的路径。它是阅读代码用的地图，不是用户手册。安装、登录和配置的操作说明仍以 `docs/install.md`、`docs/authentication.md`、`docs/config.md` 为准。

仓库是 OpenAI Codex CLI 的单体仓库。用户安装到的产品只有一个二进制 `codex`，源码在 `codex-rs/cli`。终端界面、非交互执行、IDE 用的 app server 都是这个二进制上的入口。代理逻辑在 `codex-core`。npm 包、TypeScript SDK 和 Python SDK 都在二进制外面套一层，自己不实现会话循环。

工作区里约有 172 个 `Cargo.toml`（不含 `target`）。`codex-rs/ext/` 下有 18 个可选扩展。

## 1. 仓库地图

| 路径                                                                       | 角色                                          |
| -------------------------------------------------------------------------- | --------------------------------------------- |
| `codex-rs/cli`                                                             | 产品二进制 `codex`。clap 解析参数并分发子命令 |
| `codex-rs/tui`                                                             | 交互终端界面。不带子命令时进入这里            |
| `codex-rs/exec`                                                            | `codex exec` 与 `codex review`                |
| `codex-rs/app-server` 及 `app-server-*`                                    | IDE / 桌面用的 JSON-RPC                       |
| `codex-rs/exec-server`                                                     | 用 PTY 拉起并控制子进程                       |
| `codex-rs/core`                                                            | 会话、轮次、压缩、工具调度。不直接打印到终端  |
| `codex-rs/protocol`                                                        | `Op` 与 `EventMsg`，界面和 core 之间的合同    |
| `codex-rs/config`、`features`、`codex-home`                                | `config.toml`、功能开关、家目录               |
| `codex-rs/state`、`rollout`、`history`、`thread-store`                     | SQLite 里的线程和可恢复历史                   |
| `codex-rs/sandboxing`、`linux-sandbox`、`bwrap`、`windows-sandbox-service` | 文件系统和进程隔离                            |
| `codex-rs/ext/*`                                                           | MCP、网页搜索、生图、记忆、guardian 等扩展    |
| `codex-rs/code-mode*`、`v8-poc`                                            | 在 V8 里执行工具代码                          |
| `codex-cli`                                                                | npm 包 `@openai/codex`，只负责启动原生二进制  |
| `sdk/typescript`                                                           | 拉起 `codex` 子进程，用 stdin/stdout 交换事件 |
| `sdk/python`、`sdk/python-runtime`                                         | Python 包。运行时二进制在发版时打进 wheel     |
| `justfile`、`MODULE.bazel`                                                 | 日常 Cargo 任务，以及发布 / CI 用的 Bazel     |

依赖方向是固定的：界面和 SDK 依赖 core，core 依赖 protocol、config、工具和沙箱。`codex-core` 用 `clippy::print_stdout` 和 `clippy::print_stderr` 禁止库代码直接打印。用户能看见的输出只从 TUI、exec 或 app server 出去。

## 2. 进程启动

解析子命令之前，程序先看自己是被哪个文件名调用的。`codex-rs/arg0/src/lib.rs` 的 `arg0_dispatch`：

- argv0 是 Linux 沙箱别名 `CODEX_LINUX_SANDBOX_ARG0` 时，调用 `codex_linux_sandbox::run_main`，该函数不再返回。
- argv0 是 `apply_patch` 或拼写变体 `applypatch` 时，进入 `codex_apply_patch::main`。
- Linux 上还会先调用 `codex_utils_pty::init_spawn_helper`。

因此沙箱里的子进程和补丁工具复用同一个二进制，不另带一套可执行文件。`Arg0DispatchPaths` 记下当前可执行文件的稳定路径，供之后再次 exec 使用。测试工具下 `std::env::current_exe()` 可能指向测试驱动，而不是真正的 `codex`。

正常的 CLI 入口是 `codex-rs/cli/src/main.rs` 的 `main`：

1. `codex_build_info::initialize!()` 写入构建信息。
2. `codex_app_server::take_remote_control_disabled_env()` 读远程控制是否被环境变量关掉。
3. `arg0_dispatch_or_else` 确认这不是沙箱或补丁别名之后，调用 `dispatch_cli`。
4. `dispatch_cli` 若返回 future，那就是交互 TUI，在子命令处理完成之后才开始轮询界面事件。

`dispatch_cli` 用 clap 解析 `MultitoolCli`。根上的 `--enable` / `--disable` 被折成配置覆盖，再追加到每个子命令，所以功能开关不用在每个子命令里重写一遍。

## 3. 子命令分发

`Subcommand` 定义在 `cli/src/main.rs`。和运行时代码对应的关系如下。

| 用户命令                                                                                      | 代码去向                                                                                        |
| --------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `codex`、`codex agents`                                                                       | `run_interactive_tui` → `codex_tui::run_main`。`agents` 把 `interactive.agents_overview` 设为真 |
| `codex exec`                                                                                  | `codex_exec::run_main`                                                                          |
| `codex review`                                                                                | 组装成 `ExecCommand::Review`，仍调用 `codex_exec::run_main`                                     |
| `codex app-server`                                                                            | `codex_app_server::run_main_with_transport_options`，会话来源为 `SessionSource::VSCode`         |
| `codex exec-server`                                                                           | 独立的 exec server 子命令                                                                       |
| `codex mcp`、`codex plugin`                                                                   | 各自的命令模块，不进入一轮采样                                                                  |
| `codex login` / `logout`、`features`、`sandbox`、`cloud`、`resume`、`fork`、`doctor`、`debug` | 同一分发器里的其他分支                                                                          |

`exec` 和 `review` 会拒绝同时指定远程 app server。根上的 `-c key=value` 通过 `prepend_config_flags` 放进子命令，根覆盖的优先级低于子命令自己带的覆盖。

app server 的传输：

- `--stdio` 使用标准输入输出。
- 否则使用 `listen` 给出的套接字。
- 远程控制有三种启动：命令行强制打开、环境变量强制关闭、或读取已经保存的设置（`RemoteControlStartupMode`）。

## 4. 三种前端和 core 的边界

TUI、exec、app server 的差别在传输和展示，不在采样循环。

- **TUI**（`codex-rs/tui`）是默认界面。`codex [PROMPT]` 不写子命令时走这里。它也可以连远程 app server，由 `RemoteAppServerEndpoint` 描述。
- **exec**（`codex-rs/exec`）的入口是 `run_main`。它支持 `resume`、`fork`、`review`，以及 `--json`、`--output-schema`、`--ephemeral`、`--ignore-user-config`。脚本和两个 SDK 都走这条路径。originator 默认设成 `codex_exec`。
- **app server**（`codex-rs/app-server`）按 JSON-RPC 方法分发。`MessageProcessor` 把请求分到线程、账号、配置、MCP、文件系统、命令执行等处理器。`thread/start`、`thread/resume`、`thread/fork`、`turn/start` 已不再接受旧字段 `permissionProfile`，调用方要改用带名字的 `permissions`。

core 对外的句柄是 `CodexThread`（`codex-rs/core/src/codex_thread.rs`）。它保存：

- `Session`：真正的会话状态和服务。
- `SessionIo`：提交队列和事件通道。
- `session_source`：谁打开了这条线程。
- `rollout_path`：可恢复历史的路径。
- `residency_gate`：代理邮件一类的读锁，驱逐时需要写锁。

对外提交只有 `CodexThread::submit(Op)`，内部转到 `SessionIo::submit`。

## 5. 一轮对话

开一轮不直接调用 `run_turn`。调用方走 `SessionIo::submit_turn_input`（`core/src/session/mod.rs`）：

1. 分配 submission id。
2. 构造 `Op::TurnInput { request, mode, reply }`。
3. `submit_with_id` 把 `Submission` 送进 `tx_sub`。发送失败表示会话循环已经死了，返回 `InternalAgentDied`。
4. oneshot 只等待 core 的入队决定。调用方丢掉等待方，已经入队的输入也不会撤回。

会话循环随后调用 `run_turn`（`core/src/session/turn.rs`）。函数注释写明循环规则：

- 每次采样，模型要么请求函数调用，要么给出助手消息。
- 函数调用执行完，输出成为下一次采样的输入。
- 只有助手消息时，记入历史，本轮结束。

采样之前的固定准备：

- 若线程数据里没有 `AutoCompactionDisabled`，并且没有打开 `Feature::TokenBudget`，则允许自动摘要。压缩在记录新用户消息之前运行（`run_pre_sampling_compact`）。
- 基本会话来源会做 guardian 的待处理输入检查。
- `drain_async_hook_results` 把上一轮滞后完成的 hook 结果写进历史。
- 从用户输入收集必须先启动的 MCP 服务，以及被显式点名的插件 id。

压缩失败时，新输入仍通过 `run_hooks_and_record_inputs` 落盘，再发 `EventMsg::Error`。注释说明这样做是为了避免客户端在提示词还没保存时就把下一句插进来。`TurnAborted` 和 `ToolCollision` 会把错误原样返回。

`TurnContext` 携带这一轮的配置、模型、权限和会话来源。`CodexThreadSettingsOverrides` 是 app server 在开轮前可以改的稀疏设置，包括模型、推理力度、沙箱策略、审批策略、插件禁用列表和协作模式。`ThreadSettings` 这个 `Op` 与开轮共用同一条队列，所以设置变更和用户输入的顺序与调用顺序一致。

## 6. 模型客户端

`ModelClient`（`core/src/client.rs`）持有会话级状态：认证、供应商、线程 id、HTTP 客户端工厂、扩展贡献的 `ModelRequestContributor`，以及传输是否已经退回 HTTP。WebSocket 一旦在某一轮退回 HTTP，这个 `ModelClient` 上之后的轮次都走 HTTP。

每一轮必须新建 `ModelClientSession`。它懒连接 Responses WebSocket，并在本轮的多次请求之间复用连接。它还缓存：

- 上一份完整请求，便于后续增量 WebSocket 载荷只在「当前请求是上一请求的延伸」时使用。
- 响应头 `x-codex-turn-state`。客户端在本轮的重试、增量追加和续写里原样回放，不得带到下一轮。认证归属变化时清空，让新的所有者拿到新的路由状态。

`ModelClientSession::stream` 只匹配 `WireApi::Responses`：

1. 供应商允许 WebSocket 时调用 `stream_responses_websocket`。
2. 结果为 `WebsocketStreamOutcome::Stream` 时直接返回流。
3. 结果为 `FallbackToHttp` 时调用 `try_switch_fallback_transport`，再由 `ResponsesHttpExecutor::stream_responses_http` 完成同一次请求。

模型、推理力度、服务等级和本轮元数据由调用方按轮传入，不藏在客户端里面。这样轮次的生命周期在调用点是可见的。

## 7. 工具分发

模型返回的函数调用进入 `core/src/tools/parallel.rs` 的 `ToolCallRuntime::handle_tool_call`：

1. 用 `executed_tool_calls.prepare_direct_call` 登记。
2. 问 `ToolRouter::tool_supports_parallel`。不能并行的调用共用 `parallel_execution` 这把锁。
3. 通知每个扩展的 `on_tool_dispatch`。
4. 调用 `ToolRouter::dispatch_tool_call_with_state`。

来源分两类，计时和沙箱都认这个字段：

- `ToolCallSource::Direct` 或 `DirectPlaintextMessage`：模型直接点名。
- `ToolCallSource::CodeMode`：V8 代码模式里的回调。这类调用不走直接工具的计时门闩。

`core/src/tools/router.rs` 的 `dispatch_tool_call_with_code_mode_result_inner` 把参数装成 `ToolInvocation`（session、turn、step context、cancellation token、call id、payload、来源），再交给 `registry.dispatch_any_with_state`。

失败策略：普通 `FunctionCallError` 收成一条失败的 `ResponseItem` 送回模型。只有 `FunctionCallError::Fatal` 变成 `CodexErr::Fatal`。

工具可见性在 `record_registration_metrics` 里分成 Direct、DirectModelOnly、Deferred、DeferredModelOnly、CodeModeOnly、Hidden。注册了不等于模型看得到，托管工具不进这组计数。

内置工具在 `core/src/tools/handlers/`。可选能力在 `codex-rs/ext/`：

`agent`、`agent-message-board`、`connectors`、`extension-api`、`git-attribution`、`goal`、`guardian-reviewer`、`guardian-v2`、`guardian-v3`、`history-notes`、`image-generation`、`items`、`mcp`、`memories`、`multi-env-copy-file`、`queue`、`skills`、`web-search`。

代码模式相关 crate 是 `code-mode`、`code-mode-host`、`code-mode-protocol`、`code-mode-runtime` 和 `v8-poc`。JavaScript 引擎来自 `third_party/v8`。

## 8. 沙箱

Linux 上，受限命令重新 exec 同一个二进制，argv0 换成沙箱别名，于是回到第 2 节的 `arg0_dispatch`。

`codex-rs/linux-sandbox/src/linux_run_main.rs` 的 `run_bwrap_with_proc_fallback`：

1. `initialize_bwrap_launcher` 选择 bwrap。
2. 若需要挂 `/proc`，先做 preflight。失败则把 `mount_proc` 关掉再重试，并且不把这次失败写进子进程 stderr。
3. `build_bwrap_argv` 按文件系统策略、沙箱 cwd 和命令 cwd 拼参数。构造失败则 `exit_with_bwrap_build_error`。

`linux-sandbox/src/launcher.rs` 优先使用 PATH 上、且不在当前工作目录中的系统 `bwrap`。`--argv0` 从 bubblewrap 0.9.0 起存在。帮助文本里没有这个参数时，走无 `--argv0` 的兼容路径。系统里没有可用 bwrap 时，core 的说明（`codex-rs/core/README.md`）写的是回退到安装包内的 `codex-resources/bwrap`，并通过正常通知路径给一条启动警告。WSL1 不能创建 user namespace，会在进入 bwrap 之前拒绝沙箱命令。WSL2 走普通 Linux 路径。

策略类型在 `codex-protocol`：`FileSystemSandboxPolicy` 与 `NetworkSandboxPolicy`。`codex-rs/sandboxing` 把旧的 `SandboxPolicy`（只读、工作区可写）翻译成拆开的文件系统策略和网络策略。需要精确拒绝或只读嵌套目录时走 bubblewrap；能在旧模型里往返且语义不变的策略才继续用 Landlock。

其他平台：

- macOS 使用 `/usr/bin/sandbox-exec`。工作区可写策略下，`.git` 和 `.codex` 保持只读。
- Windows 使用 `windows-sandbox-service`。旧的只读和工作区可写仍被接受；精确的可读根要走拆开的文件系统策略。

网络域名和 Unix 套接字权限由 `codex-network-proxy` 表达。执行前的人工决定走 `Op::ExecApproval` 和 `Op::PatchApproval`。app server 还可以依据托管的 `requirements.toml` 发出 `item/commandExecution/requestApproval`（`kind: humanCommand`）。这种检查点不能被会话级或持久批准满足。

## 9. 协议

定义在 `codex-rs/protocol/src/protocol.rs`。

`Op` 是客户端提交给 core 的操作。和一轮对话直接相关的变体：

| 变体                            | 行为                                                                 |
| ------------------------------- | -------------------------------------------------------------------- |
| `TurnInput`                     | 携带 `TurnInputRequest`、`TurnInputMode`，oneshot 回传入队结果       |
| `RecoverTurn`                   | 恢复被打断的普通轮次                                                 |
| `Interrupt`                     | 中止当前任务，不结束后台终端。core 回应 `EventMsg::TurnAborted`      |
| `CleanBackgroundTerminals`      | 显式结束这条线程上的后台终端                                         |
| `ThreadSettings`                | 不开新轮，只应用稀疏的线程设置。成功时仍广播 `ThreadSettingsApplied` |
| `ExecApproval`、`PatchApproval` | 用户对命令或补丁的 `ReviewDecision`                                  |
| `UserInputAnswer`               | 回答 `request_user_input`                                            |
| `RequestPermissionsResponse`    | 回答权限请求                                                         |
| `ResolveElicitation`            | 回答 MCP elicitation                                                 |
| `SuspendTurnAndShutdown`        | 停下根轮次，并且不记录终态轮次事件                                   |

`EventMsg` 用 serde 内部标签 `type`，名字为 snake_case，并生成 JSON Schema 和 TypeScript 类型。历史兼容：

- 一轮开始在线上叫 `task_started`，别名 `turn_started`。
- 一轮结束在线上叫 `task_complete`，别名 `turn_complete`。

界面常用的事件包括 `Error`、`Warning`、`AgentMessage`、`AgentReasoning`、`TokenCount`、`ContextCompacted`、`ExecCommandBegin`、`ExecCommandOutputDelta`、`ExecCommandEnd`、`ExecApprovalRequest`、`McpToolCallBegin` / `End`、`WebSearchBegin` / `End`、`ImageGenerationBegin` / `End`。`ThreadRolledBack` 只为重放旧 rollout 保留，注释写明实时回滚已经不支持。

枚举注释要求 `EventMsg` 的载荷不要使用可选类型，否则扩展代码生成会坏。

## 10. 配置、功能开关和本地状态

`codex-rs/config/src/config_layer_source.rs` 的 `ConfigLayerSource::precedence` 决定谁覆盖谁。数字越大越优先。

| 层                                | 优先级 |
| --------------------------------- | ------ |
| `PackagedDefaults` 安装包默认值   | -10    |
| `Mdm`                             | 0      |
| `System` 主机范围文件             | 10     |
| `EnterpriseManaged` 企业云配置    | 15     |
| `User` 的 `config.toml`           | 20     |
| `User` 且选中了 profile           | 21     |
| `Project` 项目 `.codex` 目录      | 25     |
| `SessionFlags` 本次进程的 `-c`    | 30     |
| `LegacyManagedConfigTomlFromFile` | 40     |
| `LegacyManagedConfigTomlFromMdm`  | 50     |

家目录由 `CODEX_HOME` 决定，解析函数在 `codex_utils_home_dir::find_codex_home`。登录态在 `codex-rs/login`。密钥可以进 `keyring-store`，或在选中密钥环存储时进入 `secrets` 描述的加密后端（功能开关 `SecretAuthStorage`）。

线程正文在 `codex-rs/state` 的 SQLite。`rollout` 和 `history` 负责可恢复历史，`thread-store` 负责本地队列一类的存储。app server 的 `thread/list` 支持 `excludedThreadIds`（最多 100 个），排除发生在分页限制之前。

功能开关是 `codex-rs/features/src/lib.rs` 里的 `Feature` 枚举。稳定项包括 `ShellTool`、`ViewImage`、`SleepTool`、`CodexHooks`、`DaemonAutoStart`。实验项和已删除项也留在枚举里，因此 `codex features list` 和旧 rollout 仍能认出它们。CLI 的 `--enable` 只是向 `SessionFlags` 层写入 `features.<name>=true`。

## 11. SDK 和 npm 包装

这些包装不链接 Rust 库。

- `codex-cli/package.json` 的 bin 是 `bin/codex.js`，描述为启动本地 Codex CLI。
- `sdk/typescript/src/codex.ts` 构造 `CodexExec`。`sdk/typescript/src/exec.ts` 用 `spawn` 启动 `executablePath`。API key 写入子进程环境变量 `CODEX_API_KEY`。继续已有线程时，参数是子命令 `resume` 加上线程 id。审批策略、网页搜索和网络访问通过 `--config` 传给 CLI。
- `sdk/python` 的公开 API 调用已发布的 CLI。`sdk/python-runtime` 的说明写明：平台二进制在发版时打进 wheel，源码仓库不存放该二进制。包名 `openai-codex-cli-bin` 只发 wheel，不发 sdist。

## 12. 构建

两条构建路径并存。

- 日常开发：`docs/install.md` 与根目录 `justfile`。在 `codex-rs` 里使用 Rust 1.95.0（`codex-rs/rust-toolchain.toml`，组件为 clippy、rustfmt、rust-src）。`just test` 调用 cargo-nextest。`just fmt` 同时格式化 Rust、Python 和 Bazel。常用命令是 `cargo build -p codex-cli --bin codex`。
- 发布和 CI 里的大测试：Bazel（`MODULE.bazel`、各目录 `BUILD.bazel`）。`just bazel-codex` 用 Bazel 构建并运行 CLI。
- Node：根 `package.json` 要求 Node `>=24.22.1`，`.nvmrc` 为 `24.22.1`，包管理器为 pnpm `10.34.5`。`pnpm-workspace.yaml` 只包含 `codex-cli`、`codex-rs/responses-api-proxy/npm` 和 `sdk/typescript`。
- `tools/argument-comment-lint` 使用单独的 nightly 工具链，不属于主 CLI 构建。
- `third_party/` 还包含 voice、wezterm、wine 和 powershell。Wine 用于在 Linux 上跑 Windows exec 测试。

Linux 上编译原生依赖至少需要 `pkg-config`、`libssl-dev`、`libcap-dev`，沙箱和音频相关测试还需要 `bubblewrap` 与 `libasound2-dev`。`openssl-sys` 在未找到系统 OpenSSL 开发包时会让构建失败。

## 13. 建议的阅读顺序

| 目的                         | 从这里开始                                                    |
| ---------------------------- | ------------------------------------------------------------- |
| 子命令一览                   | `codex-rs/cli/src/main.rs` 的 `Subcommand` 与 `dispatch_cli`  |
| 沙箱别名为何能进同一个二进制 | `codex-rs/arg0/src/lib.rs` 的 `arg0_dispatch`                 |
| 一轮采样循环                 | `codex-rs/core/src/session/turn.rs` 的 `run_turn`             |
| 提交队列                     | `codex-rs/core/src/session/mod.rs` 的 `submit_turn_input`     |
| 模型传输                     | `codex-rs/core/src/client.rs` 的 `ModelClientSession::stream` |
| 工具实现                     | `codex-rs/core/src/tools/handlers/`，或 `codex-rs/ext/<名字>` |
| 工具如何被调用               | `codex-rs/core/src/tools/router.rs`                           |
| 事件和审批的名字             | `codex-rs/protocol/src/protocol.rs` 的 `Op`、`EventMsg`       |
| IDE 方法名                   | `codex-rs/app-server/src/message_processor.rs`                |
| Linux 隔离                   | `codex-rs/linux-sandbox/src/linux_run_main.rs`                |
| 配置谁覆盖谁                 | `codex-rs/config/src/config_layer_source.rs`                  |
| 功能开关名单                 | `codex-rs/features/src/lib.rs` 的 `Feature`                   |
