# codex-rs 内置工具盘点

> 盘点对象：`/Users/vector/cattle/codex/codex-rs/`（OpenAI Codex CLI Rust 工作空间）自身实现/注册的工具。
> 盘点方法：纯只读盘点，多轮 `Grep` + `Read` + `Glob` 验证；引用全部 `file:line`；不修改任何业务代码。
> 总规模：**35 个内置 handler** + 6 类 spec 文件 + 1 套派发/沙箱基础设施。

---

## 0. TL 抽查修正（合并自 TL 复核）

原盘点报告中的 3 处偏差，已在本版全部纠正：

1. **`registry.rs` 总行数 = 860**（实测 `wc -l` 验证）。
2. **路由入口函数命名带后缀**：实际签名为
   - `router.rs:300 pub async fn dispatch_tool_call_with_code_mode_result`
   - `router.rs:323 pub(crate) async fn dispatch_tool_call_with_terminal_outcome`
   - `router.rs:346 async fn dispatch_tool_call_with_code_mode_result_inner`（内部实现）
   - 注册表侧：`registry.rs:529 pub(crate) async fn dispatch_any_with_terminal_outcome`
3. **`ZshForkSpawnLifecycle`**：真实存在于 `codex-rs/core/src/tools/runtimes/zsh_fork.rs:42-60`，是 `struct ZshForkSpawnLifecycle` 实现 `SpawnLifecycle` trait，含 `inherited_fds()` 与 `after_spawn()`，用于 unix 平台 unified_exec 的 zsh-fork 后端转发 escalation socket fd，spawn 后清理客户端 socket。

---

## 1. 基础设施（trait / registry / router / runtimes）

| 角色 | 实体 | 文件:行号 |
| --- | --- | --- |
| Handler 实现 | 各 handler 实现 `handle()`（无统一 trait；通过 `CoreToolRuntime` 注册） | `codex-rs/core/src/tools/handlers/*` |
| 注册中心 | `ToolRegistry`（`IndexMap<ToolName, RegisteredTool>`） | `codex-rs/core/src/tools/registry.rs`（总 860 行）；map 在 `:293`，注册 `register_trusted` / `add` 在 `:317-358`，`dispatch_any_with_terminal_outcome` 在 `:529` |
| 路由构造 | `build_tool_router` / `build_core_tool_registry` / `finalize_tool_router` | `codex-rs/core/src/tools/spec_plan.rs:121, 271, 345` |
| Router 入口 | `ToolRouter` + `dispatch_tool_call_with_code_mode_result` 等 | `codex-rs/core/src/tools/router.rs:300`（主入口）, `:309, 346`（内部实现）, `:323`（替代入口）, `:378`（调注册表） |
| Orchestrator | `ToolOrchestrator::run`（仅 patch / unified_exec 走） | `codex-rs/core/src/tools/orchestrator.rs`；桥接点在 `apply_patch.rs:606-611` |
| 沙箱运行时 | `ApplyPatchRuntime` / `UnifiedExecRuntime` / `ZshForkRuntime` + `ZshForkSpawnLifecycle` | `codex-rs/core/src/tools/runtimes/mod.rs:35-37`（声明），`runtimes/zsh_fork.rs:42-60`（`ZshForkSpawnLifecycle` 定义） |
| Schema 描述 | 每个 handler 配套 `*_spec.rs` | `apply_patch_spec.rs:19`、`shell_spec.rs:96/146/180`、`plan_spec.rs:43`、`mcp_resource_spec.rs:24/52/80`、`view_image_spec.rs` 等 |

---

## 2. 全部 35 个内置 handler

### 2.1 文件/Shell（核心 3 个）

| # | 工具名 | 文件 | 用途 | IO 要点 | file:line |
| --- | --- | --- | --- | --- | --- |
| 1 | `apply_patch` | `handlers/apply_patch.rs` | freeform 补丁（Create/Update/Delete/Move） | `ToolPayload::Custom { input: patch_text }` → `codex_apply_patch::parse_patch` + `verify_apply_patch_args_with_mode` | `apply_patch.rs:343-345,376-380,381-404` |
| 2 | `exec_command` | `handlers/unified_exec/exec_command.rs` | 长驻 shell 进程（start/wait/write 三合一） | schema 见 `shell_spec.rs:96` | `exec_command.rs:113-114` |
| 3 | `write_stdin` | `handlers/unified_exec/write_stdin.rs` | 向运行中 shell 进程写 stdin | schema 见 `shell_spec.rs:146` | `write_stdin.rs:37-38` |

> 沙箱硬规则就位：仅此 3 个 handler 走 `ToolOrchestrator`，触发 Seatbelt/bubblewrap approval。

### 2.2 计划/上下文（5 个）

| # | 工具名 | 文件 | 用途 | file:line |
| --- | --- | --- | --- | --- |
| 4 | `update_plan` | `handlers/plan.rs` | 推送/替换任务计划步骤 | `plan.rs:49-50`；spec `plan_spec.rs:43` |
| 5 | `new_context` | `handlers/new_context_window.rs` | 触发新上下文窗口（带 token 计数） | `:19-20`；常量 `new_context_window_spec.rs:6` |
| 6 | `get_context_remaining` | `handlers/get_context_remaining.rs` | 报告剩余上下文窗口 | `:62-63`；常量 `get_context_remaining_spec.rs:8` |
| 7 | `clock::curr_time` | `handlers/current_time.rs` | 返回 UTC 时间字符串（namespaced） | `:24-25,52-53,73-78` |
| 8 | `sleep` | `handlers/sleep.rs` | agent 线程 sleep N 毫秒 | `:27,66-67` |

### 2.3 用户交互（5 个）

| # | 工具名 | 文件 | 用途 | file:line |
| --- | --- | --- | --- | --- |
| 9 | `request_user_input` | `handlers/request_user_input.rs` | 同步向用户多选/文本提问 | `:31-32`；spec `request_user_input_spec.rs:9` |
| 10 | `request_user_input_async` | `handlers/request_user_input_async.rs` | 异步后台提问（不阻塞 turn） | `:22,35-36` |
| 11 | `request_permissions` | `handlers/request_permissions.rs` | 请求沙箱/网络扩权并写审批理由 | `:31-32`；schema `shell_spec.rs:180` |
| 12 | `send_message_to_user_async` | `handlers/send_message_to_user_async.rs` | 异步推送消息给用户 | `:21,32-33` |
| 13 | `wait_for_environment` | `handlers/wait_for_environment.rs` | 等远端 exec 环境就绪 | `:19,85-86` |

### 2.4 媒体/视觉（1 个）

| # | 工具名 | 文件 | 用途 | IO | file:line |
| --- | --- | --- | --- | --- | --- |
| 14 | `view_image` | `handlers/view_image.rs` | 读本地图片 → base64 data URL 给模型 | args: `path: String, environment_id?: Option<String>, detail?: Option<String>` | `:59-64,73-74` |

### 2.5 MCP（4 个）

| # | 工具名 | 文件 | 用途 | file:line |
| --- | --- | --- | --- | --- |
| 15 | `mcp`（动态 `mcp__<server>__<tool>`） | `handlers/mcp.rs` | MCP 通用 handler（所有 MCP tool 复用同一实现） | `:120-…` |
| 16 | `list_mcp_resources` | `handlers/mcp_resource/list_mcp_resources.rs` | 列出 MCP server 暴露的资源 | `:21-22` |
| 17 | `list_mcp_resource_templates` | `handlers/mcp_resource/list_mcp_resource_templates.rs` | 列出 MCP 资源 URI 模板 | `:21-22` |
| 18 | `read_mcp_resource` | `handlers/mcp_resource/read_mcp_resource.rs` | 按 URI 读取 MCP 资源 | `:24-25` |

### 2.6 动态 / 扩展（3 个）

| # | 工具名 | 文件 | 用途 | file:line |
| --- | --- | --- | --- | --- |
| 19 | `dynamic`（任意名，含可选 namespace） | `handlers/dynamic.rs` | 运行时从 `DynamicToolFunctionSpec` 生成 | `:54-57,86-87` |
| 20 | `tool_search` | `handlers/tool_search.rs` | 客户端 deferred 工具搜索 | `:170-171`（router 在 `router.rs:262-281` 处理） |
| 21 | `ExtensionToolAdapter` | `handlers/extension_tools.rs` | 把扩展 `ToolExecutor<ToolCall<'call>>` 适配进 core registry | `:43-46` |

### 2.7 插件（2 个）

| # | 工具名 | 文件 | 用途 | file:line |
| --- | --- | --- | --- | --- |
| 22 | `request_plugin_install` | `handlers/request_plugin_install.rs` | 请求安装插件 | `:73-74`；spec `:102,161` |
| 23 | `list_available_plugins_to_install` | `handlers/list_available_plugins_to_install.rs` | 列出可安装插件 | `:58-59`；spec `:32` |

### 2.8 多 Agent v1（5 个，namespace `multi_agent`）

| # | 工具名 | 文件 | file:line |
| --- | --- | --- | --- |
| 24 | `spawn_agent`（v1） | `handlers/multi_agents/spawn.rs` | `:27` |
| 25 | `send_input`（v1） | `handlers/multi_agents/send_input.rs` | `:10` |
| 26 | `resume_agent`（v1） | `handlers/multi_agents/resume_agent.rs` | `:13` |
| 27 | `wait_agent`（v1） | `handlers/multi_agents/wait.rs` | `:31` |
| 28 | `close_agent`（v1） | `handlers/multi_agents/close_agent.rs` | `:9` |

### 2.9 多 Agent v2（6 个）

| # | 工具名 | 文件 | file:line |
| --- | --- | --- | --- |
| 29 | `spawn_agent`（v2） | `handlers/multi_agents_v2/spawn.rs` | `:44-45` |
| 30 | `send_message`（v2） | `handlers/multi_agents_v2/send_message.rs` | `:12-13` |
| 31 | `followup_task`（v2） | `handlers/multi_agents_v2/followup_task.rs` | `:12-13` |
| 32 | `interrupt_agent`（v2） | `handlers/multi_agents_v2/interrupt_agent.rs` | `:11-12` |
| 33 | `wait_agent`（v2） | `handlers/multi_agents_v2/wait.rs` | `:23-24` |
| 34 | `list_agents`（v2） | `handlers/multi_agents_v2/list_agents.rs` | `:9-10` |

### 2.10 测试（1 个）

| # | 工具名 | 文件 | 用途 | file:line |
| --- | --- | --- | --- | --- |
| 35 | `test_sync_tool` | `handlers/test_sync.rs` | 仅测试用同步工具 | `:64-65`；spec `:59` |

---

## 3. 派发机制（链路含命名后缀的精确版）

| 步 | 动作 | 关键函数 / 文件:行 |
| --- | --- | --- |
| 1 | LLM 返回 tool_call JSON | 经 model-provider 解析为 `ResponseItem::FunctionCall` |
| 2 | Codex 解析为 Rust 类型 | `router.rs:244 pub fn build_tool_call(item: ResponseItem) -> Result<Option<ToolCall>, FunctionCallError>` |
| 3 | Router 主入口 | `router.rs:300 pub async fn dispatch_tool_call_with_code_mode_result(...)` → 内部 `:309 dispatch_tool_call_with_code_mode_result_inner` |
| 4 | 替代入口（带 terminal_outcome） | `router.rs:323 pub(crate) async fn dispatch_tool_call_with_terminal_outcome(...)` → `:333` 内部实现 → `:346` |
| 5 | 注册表查 handler | `router.rs:378 .dispatch_any_with_terminal_outcome(...)` → `registry.rs:529 pub(crate) async fn dispatch_any_with_terminal_outcome`（注册表主分发） |
| 6 | Orchestrator 桥接（仅 patch / unified_exec） | `apply_patch.rs:606-611` 调 `ToolOrchestrator::new().run(...)` |

> **关键事实**：**没有 `ToolRouter::new()`**。Handler 注册统一通过 `ToolRegistry::register_trusted` / `add`，每 turn 由 `spec_plan::finalize_tool_router`（`spec_plan.rs:345`）重新组装 router——支持 host-driven 动态开关。

---

## 4. 沙箱接入点

- **不在 router 包裹**——`runtimes/mod.rs:35-37` 声明三个 runtime：`ApplyPatchRuntime` / `UnifiedExecRuntime` / `ZshForkRuntime`。
- 仅 patch / unified_exec 启 runtime；其他 handler 走 in-process（无沙箱）。
- macOS Seatbelt / Linux bubblewrap / Windows restricted-token 由 core 平台层按 OS 选择，触发点是 `ApplyPatchRuntime` 的 approval 流（`runtimes/apply_patch.rs:145`）。

### 4.1 ZshForkSpawnLifecycle（unified_exec 的 unix 后端）

| 项 | 内容 |
| --- | --- |
| 位置 | `codex-rs/core/src/tools/runtimes/zsh_fork.rs:42-60` |
| 类型 | `struct ZshForkSpawnLifecycle { escalation_session: EscalationSession }` |
| 实现 trait | `impl SpawnLifecycle for ZshForkSpawnLifecycle` |
| 方法 `inherited_fds()`（`:48-55`） | 从 `ESCALATE_SOCKET_ENV_VAR` 解析 fd 返回 `Vec<i32>` |
| 方法 `after_spawn()`（`:57-59`） | 调 `escalation_session.close_client_socket()` |
| 生产路径 | `zsh_fork.rs:84 Box::new(ZshForkSpawnLifecycle { escalation_session: prepared.escalation_session })` 装入 `PreparedUnifiedExecSpawn.spawn_lifecycle` |
| 用途 | 为 unix 平台 unified_exec 的 zsh-fork 后端转发 escalation socket fd，spawn 后清理客户端 socket |

---

## 5. 明确不在 codex-rs 内置清单（Grep 验证）

| 工具 | 验证 | 结论 |
| --- | --- | --- |
| `web_fetch` | `Grep "ToolName::plain(\"web_fetch\")\|ToolSpec::WebFetch"` 在 `core/src` | **No matches**——非内置；由扩展/上游注入 |
| `notebook_edit` | 同上 | **No matches**——非内置 |
| `read_file` / `list_dir` / `grep_files` | 同上（除 guardian/tests.rs:1044 fixture 写死） | **No matches**——LLM 通过 `apply_patch` + `exec_command`/`write_stdin` 自助组合 |
| `web_search` | 实际存在于 `tools/hosted_spec.rs`（按 config/model 能力受控暴露） | 特殊形式：hosted tool，组装点 `spec_plan.rs:607-618` |

---

## 6. 对 agent 运行时的意义

| 维度 | 状态 |
| --- | --- |
| 模型实际可调用的 LLM 工具 | 受 config 注册；`apply_patch` / `exec_command` / `write_stdin` / `update_plan` / `request_user_input` 等是 base 集合 |
| CATTLE.md 提示的 `apply_patch` / `shell` / `web_fetch` 等 | `apply_patch` ✅；`shell` = `exec_command`+`write_stdin` ✅；`web_fetch` ❌（非内置）——模型若调用会走 MCP 扩展或上游注入 |
| 派发链路上代码可控点 | 通过 `config.toml` 的 `[tools]` / `disable_*` 控制启用与禁用 |
| 工具数完整度 | 35 个 handler 全覆盖（v1/v2 双轨 + 测试 1 + 扩展/mcp 4 + 动态 3 + 核心 3 + 计划 5 + 交互 5 + 媒体 1 + 插件 2） |

---

## 7. 覆盖自查

- ✅ handlers 目录全部 `.rs` 已列举（35 + 6 个 spec 配套）
- ✅ apply_patch 实现位置精确到 `apply_patch.rs:343-345,376-380`
- ✅ router / registry / orchestrator / runtimes / spec_plan 链路闭环
- ✅ web_fetch / notebook_edit / read_file / list_dir / grep_files 全部确认非内置
- ✅ TL 抽查 3 处修正已全部合并（registry.rs 860 行、router 入口命名后缀、ZshForkSpawnLifecycle）
- ⚠️ `hosted_spec.rs` 中的 `web_search` 仅按 spec_plan 控制暴露，未在 35 个 handler 内
