# CATTLE.md

This file provides guidance to Cattle Code when working with code in this repository.

## What this repo is

This is the **OpenAI Codex CLI** repository. Codex CLI is a coding agent that runs locally. The Rust source lives in `codex-rs/`; the workspace contains a separate TypeScript SDK (`sdk/typescript`) and Python SDK (`sdk/python`).

**External code contributions are not accepted** (see `docs/contributing.md`). Bug reports and root-cause analyses go to the upstream `openai/codex` issue tracker. The Codex team handles code changes.

## Development commands

The repo uses `just` (recipes in `justfile`) and `cargo-nextest` (install with `cargo install --locked cargo-nextest`).

All `just` recipes default to the `codex-rs/` workspace unless marked `[no-cd]`. Run them from the repo root.

| Task | Command |
| --- | --- |
| Format Rust, Bazel/Starlark, Python SDK, Python scripts | `just fmt` (run after any code change) |
| Lint (Clippy; scope to a crate to keep builds fast) | `just fix -p <crate>` |
| Format check only (no write) | `just fmt-check` |
| Run all tests via nextest | `just test` |
| Run tests for one crate | `just test -p <crate>` |
| Run a single test by name | `just test -p <crate> -E 'test_name'` |
| Run `codex` from source | `just codex -- "your prompt"` |
| Run `codex exec` | `just exec -- ...` |
| TUI launched against a local exec-server | `just tui-with-exec-server` (Unix) |
| Build the local Codex package archive | `just assemble-codex-package` |
| Regenerate `config.schema.json` after `ConfigToml` changes | `just write-config-schema` |
| Regenerate app-server protocol schemas + Python SDK | `just write-app-server-schema` (`--experimental` if needed) |
| Regenerate hooks schema | `just write-hooks-schema` |
| Argument-comment lint | `just argument-comment-lint` |
| Update Bazel lockfile after Cargo dep changes | `just bazel-lock-update` |
| Bazel build for the CLI | `just bazel-codex -- ...` |
| Smoke-run all workspace benchmarks once | `just bench-smoke` |
| GitHub-scripts tests (`.github/scripts`) | `just test-github-scripts` |

Direct `cargo test` is **not** used — always go through `just test` so the nextest profile is applied.

For TUI snapshot test maintenance (see `AGENTS.md` Snapshot tests section):
- `cargo install --locked cargo-insta`
- `cargo insta pending-snapshots -p codex-tui`
- `cargo insta show -p codex-tui path/to/file.snap.new`
- `cargo insta accept -p codex-tui`

The repo also builds with **Bazel** (see `BUILD.bazel`, `MODULE.bazel`, `bazel/`, `tools/argument-comment-lint/`). On Linux x86-64 there is a Wine-exec integration target `//codex-rs/core:core-all-wine-exec-test` (see `codex-rs/core/README.md`). The `argument-comment-lint` check is Bazel-only and must be re-run after lockfile changes.

## High-level architecture

`codex-rs/` is a Cargo workspace (~150 crates). Crate directory names are prefixed with `codex-` (so the `core/` directory produces the crate `codex-core`). New crates should generally **not** be added to `codex-core` — it is intentionally being shrunk (see `AGENTS.md` "The `codex-core` crate"). Prefer a new small crate or an existing sibling crate.

Layering:

- `protocol/` — Wire and event types shared between `core` and the UIs / `app-server`. Minimal dependencies; no business logic. See `codex-rs/protocol/README.md`.
- `core/` — Business logic for Codex: agent loop, model I/O, tool execution, sandboxing, session/rollout state, config. See `codex-rs/core/README.md` for the platform sandbox matrix (macOS Seatbelt, Linux bubblewrap via `codex-arg0`, Windows elevated/restricted-token sandboxes).
- `cli/` — Top-level binaries (`codex`, `codex exec`, `codex app-server`, `codex-linux-sandbox`, `codex apply-patch`, `logs_client`, `mcp`, `plugin`, `queue`, `marketplace`, `remote-control`, `daemon-install`, etc.). See `codex-rs/cli/src/`.
- `tui/` — Ratatui-based terminal UI (`chatwidget.rs`, `bottom_pane/`, `app.rs`). TUI style rules live in `codex-rs/tui/styles.md`.
- `app-server/` + `app-server-protocol/` + `app-server-transport/` + `app-server-client/` + `app-server-test-client/` — JSON-RPC API exposed by `codex app-server` (v2 is the active API surface; v1 is frozen). TypeScript bindings are generated into `v2/`. See "App-server API Development Best Practices" in `AGENTS.md`.
- `exec-server/` + `exec-server-protocol/` — Remote exec process that TUI can connect to across machines.
- `ext/` — Optional extension points loaded by `codex-core` (agent, mcp, skills, memories, goal, guardian, etc.). The plugin host lives in `plugin/` and `core-plugins/`.
- `utils/` — Many small shared utilities (absolute path, cargo-bin, image, json-to-toml, oss, plugins, template, etc.).
- `state/`, `thread-store/`, `message-history/`, `rollout/`, `rollout-trace/` — Persistent state backing the SQLite `state` DB and per-thread rollouts.
- `model-provider/` — Provider abstraction (OpenAI Responses API, ChatGPT, Ollama, LM Studio, custom).

App-server is the supported integration surface for IDEs and external clients; TUI is the reference consumer.

## Configuration surface

`config.toml` (see `docs/config.md`) and the `ConfigToml` Rust type drive everything. The JSON Schema at `codex-rs/core/config.schema.json` is generated — never hand-edit it.

## Skills and specialized workflows

`.codex/skills/` contains workflow skills consumed by Codex itself. Relevant ones when working in this repo:

- `code-review`, `code-review-breaking-changes`, `code-review-change-size`, `code-review-context`, `code-review-testing`
- `codex-pr-body` (PR body template, even though external PRs aren't accepted)
- `babysit-pr`
- `remote-tests` (cross-OS app-server + exec-server testing)
- `test-tui`
- `update-v8-version`
- `path-types`

`.codex/environments/environment.toml` defines the development environment.

## Hard rules worth re-surfacing (from `AGENTS.md`)

These come up on every task; the full list is in `AGENTS.md`.

- **Sandbox env vars** (`CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR`, `CODEX_SANDBOX_ENV_VAR`) — never add or modify related code. In this sandbox `CODEX_SANDBOX_NETWORK_DISABLED=1` is set whenever `shell` is used, and child Seatbelt processes see `CODEX_SANDBOX=seatbelt` — existing checks for these are early-exit guards the authors added for exactly this reason.
- **`codex-core` is bloated on purpose being shrunk** — resist adding code there; introduce or reuse a focused crate.
- **Module size**: target under 500 LoC, avoid files over ~800 LoC. Especially avoid growing `codex-rs/tui/src/app.rs`, `chatwidget.rs`, `bottom_pane/chat_composer.rs`, `bottom_pane/footer.rs`, `bottom_pane/mod.rs`.
- **`argument_comment_lint`**: prefix opaque positional literals (`None`, booleans, numbers) with `/*param_name*/` matching the callee signature. Run `just argument-comment-lint` (Bazel, can be slow cold).
- **Rust style**: inline `format!` args; collapse `if` per `collapsible_if`; method refs over closures; exhaustive `match` over wildcard arms; native RPITIT (`fn foo(...) -> impl Future<Output = T> + Send`) over `#[async_trait]`/`#[allow(async_fn_in_trait)]`; `#[tracing::instrument]` at the function definition rather than `.instrument(...)` at call sites.
- **Tests**: prefer integration tests over unit tests for agent work; use `core_test_support::responses` and `TestCodexBuilder::build_with_auto_env()`. Hold the `ResponseMock` returned by `mount_sse*` helpers. Prefer `wait_for_event` over `wait_for_event_with_timeout`, and `mount_sse_once` over `mount_sse_once_match`/`mount_sse_sequence`. Use `pretty_assertions::assert_eq` and deep equality. Put unit tests in `*_tests.rs` files via `#[path = "..."] mod tests;`.
- **Snapshot tests** (TUI): any user-visible UI change needs `insta` snapshot coverage (added or updated, and accepted as part of the PR).
- **Build-time file reads**: `include_str!`, `include_bytes!`, `sqlx::migrate!` etc. need the crate's `BUILD.bazel` `compile_data`/`build_script_data` updated or Bazel will fail.
- **Model context is append-only**: never rewrite history; bounded sizes on injected fragments; hard cap per fragment; >1k tokens highlighted as P0; define all fragments as structs in `core/context` implementing `ContextualUserFragment`.
- **Breaking-change surfaces**: app-server APIs, raw response item events, CLI args, config loading, rollout resume — all considered breaking.
- **Change size**: prefer under 800 lines, under 500 for complex logic; split larger work into reviewable stages.
- **Bazel lockfile drift**: CI verifies — any `Cargo.toml`/`Cargo.lock` change must include `just bazel-lock-update` output.
- **After making code changes**: run `just fmt` (no approval needed), then `just fix -p <crate>` scoped to the changed crate, then `just test -p <crate>`.
- **Doc policy**: no general product/user docs in `docs/` (exception: app-server API docs).

## SDKs

- TypeScript SDK: `sdk/typescript/` — built from generated bindings off the app-server protocol schemas (`write-app-server-schema`).
- Python SDK: `sdk/python/` — same generation pipeline.

## Build helpers worth knowing

- `scripts/format.py` — backing script for `just fmt`/`fmt-check`.
- `scripts/build_codex_package.py` — backing script for `just assemble-codex-package`.
- `scripts/run_tui_with_exec_server.sh` — backing script for `just tui-with-exec-server`.
- `tools/argument-comment-lint/` — Dylint linter (Bazel-driven).
- `.codex/skills/remote-tests/SKILL.md` — required reading for cross-OS app-server/exec-server integration tests.

## Conventions reminder

- Repo language for explanations is Chinese, but code, identifiers, and existing doc comments stay in English.
- TUI style rules: see `codex-rs/tui/styles.md`. Avoid `Style::default().fg(Color::White)`; prefer Stylize helpers (`"text".dim()`, `.bold()`, `.cyan()`, `.italic()`, `.underlined()`) and `style::secondary_text_style()`, `style::accent_style()`, `style::readable_color_on(...)`.
- Text wrapping: use `textwrap::wrap`; for `Line` use `tui/src/wrapping.rs` helpers (`word_wrap_lines`/`word_wrap_line`); prefer `initial_indent`/`subsequent_indent` over custom logic; use `prefix_lines` for list prefixing.