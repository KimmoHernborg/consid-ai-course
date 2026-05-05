# Warp Repository — Technical Analysis

## 1. Architecture

### 1.1 Layers

Warp is almost entirely Rust. There is no traditional frontend/backend split or webview; instead the application is layered as a stack of in-process modules communicating via a custom reactive UI framework.

```
+------------------------------------------------------------------------+
| app/ (the warp library) — composed application: terminal, UI panes,   |
|   AI agents, modals, settings, auth flows, root view                   |
+------------------------------------------------------------------------+
| Domain crates: ai, code, code_review, vim, editor, lsp, command,       |
|   warp_terminal (model), warp_completer, warp_files, warp_ripgrep,     |
|   computer_use, persistence, repo_metadata, voice_input, onboarding    |
+------------------------------------------------------------------------+
| Service crates: graphql, http_client, websocket, ipc, jsonrpc,         |
|   warp_server_client, firebase, http_server, remote_server, watcher    |
+------------------------------------------------------------------------+
| UI runtime: warpui (re-exports warpui_core), windowing (winit/cocoa),  |
|   rendering (wgpu/Metal), fonts (font-kit/cosmic-text/dwrote)          |
+------------------------------------------------------------------------+
| Foundations: warp_core (channel/state/features), warp_util,            |
|   warpui_core, warp_features, settings*, channel_versions, sum_tree,   |
|   string-offset, asset_cache, fuzzy_match, markdown_parser, syntax_tree|
+------------------------------------------------------------------------+
```

### 1.2 Communication mechanisms

**In-process messaging** uses a Zed/GPUI-style Entity + View + AppContext pattern (`crates/warpui_core`). Models are `Entity<T>`, views are `ViewHandle<T>`, and events propagate via `cx.emit(...)`, `cx.subscribe(...)`, `cx.observe(...)`. Most cross-component communication flows this way.

**Cross-process IPC** (`crates/ipc`) uses typed RPC over Unix Domain Sockets / Windows named pipes via `interprocess`. Used for three subsystems:
- **Local TTY server** — a child process runs the terminal pty.
- **Plugin host** — `rquickjs` JS engine in a sandboxed process.
- **Minidump crash server** — crash handler isolated from the crashing process.
- **Remote server proxy/daemon** — runs on remote SSH hosts, custom protobuf protocol (`remote_server.proto`).

**Network protocols** connect to Warp's cloud backend:
- HTTPS + GraphQL (`https://app.warp.dev/graphql/v2`)
- WebSocket subscriptions (`wss://rtc.app.warp.dev/graphql/v2`)
- Session sharing (`wss://sessions.app.warp.dev`)
- Ambient agent service (`https://oz.warp.dev`)
- Firebase REST auth

**WASM target**: the same Rust code compiles to `wasm32-unknown-unknown` for "Warp on Web" with alternate/stubbed transports.

### 1.3 Workspace and crate structure

The workspace root (`Cargo.toml`) declares `members = ["crates/*", "app"]` with `resolver = "2"`. There are 60+ supporting crates under `crates/` and a single top-level `app` crate that is the unified application.

**UI / windowing / rendering crates:**
- `crates/warpui/` and `crates/warpui_core/` — the custom in-house UI framework with platform implementations under `src/platform/{mac,linux,windows,wasm,headless}/` and a `wgpu` renderer.
- `crates/warpui_extras/`, `crates/ui_components/` — add-ons and higher-level components.
- `crates/editor/` — code editor; `crates/vim/` — Vim mode.

**Terminal crates:**
- `crates/warp_terminal/` — pure data model: `src/model/` (ansi, escape_sequences, grid, block_id, mode, mouse, indexing), `src/shell/`, `shared_session.rs`.
- `crates/command/`, `crates/command-signatures-v2/`, `crates/warp_completer/` — parsing, signatures, completions.
- `crates/warp_ripgrep/`, `crates/warp_files/`, `crates/repo_metadata/`, `crates/watcher/`.
- `crates/lsp/`, `crates/syntax_tree/`, `crates/markdown_parser/`, `crates/languages/`.

**AI/agent crates:**
- `crates/ai/` — agent orchestration, source code indexing, skills, project context, diff validation. Heavy use of `warp_multi_agent_api` (gRPC/protobuf).
- `crates/computer_use/` — screen capture/automation with per-OS backends.
- `crates/voice_input/` — speech-to-text (gated by `voice_input` feature).
- `crates/managed_secrets/` — encrypted secret storage via Google Tink.

**Networking / persistence crates:**
- `crates/graphql/` — `cynic`-based typed GraphQL client.
- `crates/http_client/` — `reqwest` wrapper injecting Warp-specific headers.
- `crates/websocket/` — unified WS API for native and wasm.
- `crates/persistence/` — Diesel + SQLite, 134 migrations, schema at `src/schema.rs`.
- `crates/remote_server/` — daemon for SSH-attached remote terminals.
- `crates/firebase/` — Firebase Auth REST client.
- `crates/ipc/` — typed IPC framework.

**Foundation crates:**
- `crates/warp_core/` — `app_id`, channel, `features`, `host_id`, `platform`, `telemetry`, `user_preferences`, `macos` (cfg-gated).
- `crates/warp_features/` — single-source-of-truth `FeatureFlag` enum (259 variants, backed by atomics).
- `crates/settings/`, `crates/settings_value/`, `crates/settings_value_derive/` — declarative settings with derive macros.
- `crates/warp_cli/` — shared `clap`-based CLI parser dispatching to worker subcommands.
- `crates/integration/` — integration test harness (`Builder`/`TestStep`).

### 1.4 Entry points and boot sequence

Five channel-specific binaries (`app/src/bin/{oss,local,dev,stable,preview}.rs`) are thin wrappers that set a `Channel` and call `warp::run()`. The real entry is `app/src/lib.rs:571`:

1. `platform::init()`
2. `init_feature_flags()` — merges channel defaults + Cargo feature flags + release flags into atomics.
3. Parse `warp_cli::Args::from_env()`.
4. If a subcommand is present, dispatch to a worker (terminal server, plugin host, minidump server, remote server proxy/daemon, ripgrep search) and return.
5. Otherwise, `run_internal()` constructs the entire application graph (AppState, services, views) and starts the `warpui::App` event loop.

---

## 2. External Dependencies

All versions are pinned in `[workspace.dependencies]` of the root `Cargo.toml` (lines 30–379).

### UI / windowing / graphics
| Library | Role |
|---|---|
| `wgpu` 29 | Cross-platform GPU (Vulkan/Metal/DX12/GLES/WebGL) |
| `winit` (forked) | Cross-platform windowing on non-Mac |
| `metal` 0.33 | macOS Metal rendering |
| `core-text`/`core-graphics` | macOS font rendering, graphics |
| `cosmic-text` (forked) | Text shaping on Linux/Windows/wasm |
| `dwrote` (forked) | Windows DirectWrite text |
| `font-kit` (forked) | Font enumeration |
| `resvg` 0.47 | SVG rendering |
| `image` 0.25 | Image decoding |
| `arboard` 3.6 | Clipboard |

### Async / runtime
| Library | Role |
|---|---|
| `tokio` 1.47 | Async runtime |
| `futures`, `async-channel`, `async-broadcast` | Async primitives |
| `parking_lot`, `dashmap` | Concurrent data structures |
| `tikv-jemallocator` (forked, optional) | jemalloc allocator with heap profiling |

### Networking / serialization
| Library | Role |
|---|---|
| `reqwest` 0.12 | HTTP client (rustls, HTTP/2, gzip/brotli, SSE) |
| `rustls` 0.23 | TLS (aws-lc-rs provider) |
| `axum` 0.8 / `hyper` 1.6 / `tower` | Embedded HTTP server |
| `reqwest-eventsource` 0.6 | Server-sent events for AI streaming |
| `cynic` 3 + `cynic-codegen` | Typed GraphQL client |
| `graphql-ws-client` 0.11 | GraphQL subscriptions |
| `prost` 0.14 | Protobuf (agent API, remote server) |
| `rmcp` (forked) | Model Context Protocol client |
| `oauth2` 5.0 | OAuth2 flows |
| `serde` / `serde_json` / `serde_yaml` / `bincode` | Serialization |

### Storage / search
| Library | Role |
|---|---|
| `diesel` 2.3 + `libsqlite3-sys` (bundled) | ORM + SQLite persistence |
| `tantivy` 0.26 | Full-text search index for command history |
| `git2` 0.20 (vendored libgit2) | Git operations for code review |
| `parquet` 55.0 | Columnar data (analytics/export) |

### AI / agents
| Library | Role |
|---|---|
| `warp_multi_agent_api` (private fork) | Core typed agent API (protobuf service) |
| `arborium` | Multi-language tree-sitter wrapper (30+ languages) |
| `aws-config` / `aws-sdk-sts` | BYO Bedrock LLM auth |
| `mermaid_to_svg` (forked) | Mermaid diagram rendering |
| `rquickjs` 0.3 | Embedded JS engine for plugin host |

### Telemetry / crash reporting
| Library | Role |
|---|---|
| `sentry` 0.41 + Cocoa framework | Error/crash reporting |
| `crash-handler` / `minidumper` | Linux/Windows crash bridge |
| `pprof` 0.15 | CPU/heap profiling |
| `tracing` 0.1 | Structured tracing |

### Platform bridges
| Library | Role |
|---|---|
| `cocoa` / `objc2` (macOS) | Native macOS APIs and Objective-C interop |
| `windows` 0.62 (Windows) | Win32/WinRT APIs (DWM, DirectWrite, Shell) |
| `nix` 0.26 (Unix) | POSIX APIs |
| `x11rb` / `zbus` / `ashpd` (Linux) | X11, D-Bus, XDG portals |
| `tink-core/hybrid` (forked) | Google Tink hybrid encryption |

### TypeScript sidecars (minimal)
- `crates/command-signatures-v2/js/` — compiles command signature data to JS.
- `crates/warp_graphql_schema/` — GraphQL schema for `cynic-codegen`. Neither is a runtime component.

---

## 3. Integrations

### 3.1 Databases
**SQLite** (bundled, statically linked via `libsqlite3-sys`) is the sole local persistent store.
- Schema: `crates/persistence/src/schema.rs` (Diesel-generated).
- 134 migrations under `crates/persistence/migrations/` (2021-10 → 2026-02).
- Tables include: `agent_conversations`, `agent_tasks`, `ai_document_panes`, `ai_memory_panes`, `ai_queries`, `ambient_agent_panes`, `app`, `blocks`, `active_mcp_servers`, and many more.
- Application-level read/write: `app/src/persistence/sqlite.rs` (3708 lines).

**Tantivy** provides a secondary full-text index for command search (`app/src/search/searcher.rs`), gated by feature `use_tantivy_search`.

### 3.2 Warp backend services
Defined in `crates/warp_core/src/channel/config.rs:43-72`:

| Endpoint | URL | Purpose |
|---|---|---|
| GraphQL API | `https://app.warp.dev/graphql/v2` | REST + typed GraphQL |
| RTC subscriptions | `wss://rtc.app.warp.dev/graphql/v2` | Live drive object updates |
| Session sharing | `wss://sessions.app.warp.dev` | Shared terminal sessions |
| Ambient agent | `https://oz.warp.dev` | "Oz" agent dashboard/API |
| Firebase Auth | `AIzaSy…UygAs` (key per channel) | User authentication |
| Releases | (channel-specific) | Auto-update checks |
| Sentry DSN | (channel-specific) | Crash/error reporting |

Custom HTTP headers injected by `crates/http_client`: `X-Warp-Client-Version`, `X-Warp-OS-Category`, `X-Warp-OS-Name`, `X-Warp-OS-Version`, `X-Warp-Client-ID`.

### 3.3 GraphQL surface
`crates/graphql/src/api/` exposes typed operations: `ai.rs`, `billing.rs`, `experiment.rs`, `folder.rs`, `full_source_code_embedding.rs`, `mcp_gallery_template.rs`, `notebook.rs`, `object.rs`, `object_actions.rs`, `object_permissions.rs`, `mutations/`, `queries/`, `subscriptions/`, `user.rs`, `workflow.rs`, `workspace.rs`.

### 3.4 LLM providers
`app/src/ai/llms.rs` enumerates `LLMProvider::{OpenAI, Anthropic, Google, Xai, Unknown}`. The product supports:
- **Server-side routing** via `warp_multi_agent_api` protobuf service — most agent calls go through Warp's backend for telemetry, redaction, and routing.
- **BYO API keys** via `crates/ai/src/api_keys.rs`.
- **AWS Bedrock** via `crates/ai/src/aws_credentials.rs` and AWS SDK.
- **Sub-agent harnesses** in `app/src/ai/agent_sdk/driver/harness/` for Claude Code (`claude_code.rs`), Codex (`codex.rs`), Gemini (`gemini.rs`).

### 3.5 Model Context Protocol (MCP)
`app/src/ai/mcp/` (10+ files) handles the MCP gallery, file-based MCP servers, templatable MCP, OAuth discovery, and reconnecting peers. Uses the forked `rmcp` crate with transport layers: streamable-HTTP, SSE, child-process.

### 3.6 Other integrations
- **Sentry** — crash reporting + error capture; native crate on all platforms, Cocoa framework on macOS, `minidumper` server on Linux/Windows.
- **RudderStack** — non-PII telemetry via vendored SDK (`app/src/server/telemetry/`).
- **Firebase** — user authentication (`crates/firebase`).
- **Linear** — `app/src/linear.rs` deep-linking for issue workflows.
- **GitHub** — feature-flagged PR integration (`github_pr_prompt_chip`, `pr_comments_*`).
- **OAuth2** — generic flows used by MCP and agent integrations.

---

## 4. Business Logic

### 4.1 Application orchestration
- `app/src/lib.rs` (2889 lines) — boot, feature gating, `run()`/`run_internal()`, `AppState` wiring, `init_feature_flags()`.
- `app/src/root_view.rs` (3605 lines) — top-level window view: `RootView`, quake mode, opening drive objects, joining shared sessions, cloud conversations, deep-linking.
- `app/src/voltron.rs` — composite UI element unifying workflows, AI commands, and history search.
- `app/src/menu.rs` (2734 lines), `app/src/app_menus.rs` (1187 lines) — menus and macOS menu bar.
- `app/src/tab.rs` (1732 lines) — tab management.

### 4.2 Terminal
`app/src/terminal/` (~80+ files):
- `terminal_manager.rs`, `view.rs`, `input.rs`, `bootstrap.rs` — terminal lifecycle and I/O.
- `local_tty/`, `remote_tty/`, `ssh/`, `wsl/`, `writeable_pty/` — pty backends.
- Block model: `block_filter.rs`, `block_list_element.rs`, `blockgrid_renderer.rs`.
- Shell integration: `available_shells.rs`, `warpify/`, `prompt/`.
- History: `history.rs`, `rich_history.rs`, `recorder.rs`.
- Shared sessions: `shared_session/`, `share_block_modal.rs`.
- CLI agent integration: `cli_agent.rs`, `cli_agent_sessions/`.

Underlying terminal data model in `crates/warp_terminal/src/model/` (ANSI parser, grid, escape sequences, mouse/mode).

### 4.3 AI / agent mode (headline product feature)
`app/src/ai/` (50+ subdirs) is the largest single domain:
- `agent/` — action, action_result, citation, convert, orchestration_config.
- `agent_sdk/driver/harness/` — sub-agent drivers for Claude Code, Codex, Gemini.
- `agent_conversations_model.rs` (1954 lines) — agent conversation state.
- `conversation_details_panel.rs` (2010 lines) — chat UI.
- `ambient_agents/` — scheduled agents, GitHub auth notifier.
- `blocklist/` — tool action models (execute/grep, response_stream).
- `mcp/` — MCP gallery, templatable servers, OAuth.
- `context_chips/` — @-mention context system for AI input.
- `skills/`, `facts/`, `artifacts/`, `predict/`, `loading/`.

`crates/ai/src/` handles infrastructure: source-code indexing (`index/`), project context, diff validation, API keys, AWS credentials.

### 4.4 Code editing / review
- `app/src/code/` (22 modules) — file editor model, opened files.
- `app/src/code_review/code_review_view.rs` (**7881 lines** — largest file in the repo) — the full code review surface.
- `app/src/code_review/diff_state.rs` (3016 lines) — diff state management.
- `crates/editor/`, `crates/vim/`, `crates/lsp/` — editor, Vim mode, LSP client.

### 4.5 Cloud / drive / notebooks / workflows
- `app/src/drive/index.rs` (5646 lines) — server-side drive object model.
- `app/src/cloud_object/mod.rs` (1692 lines) — abstraction over drive objects.
- `app/src/notebooks/` (16 files) — cloud notebooks and offline editor.
- `app/src/workflows/` (20 files) — workflow library.
- `app/src/server/cloud_objects/{listener.rs, update_manager.rs}` — RTC subscription pipeline.

### 4.6 Auth
`app/src/auth/` (23 files): `auth_manager/`, `auth_state.rs`, `login_slide.rs` (1351 lines), `web_handoff.rs`, `credentials.rs`, `user.rs`. Authentication flows through Firebase REST with Warp session tokens on top.

### 4.7 Settings / experiments
- `app/src/settings/` (41 files), `app/src/settings_view/` (48 files).
- `app/src/experiments/`, `app/src/server/experiments/` — server-driven A/B experiments.
- `crates/settings/`, `crates/settings_value/`, `crates/settings_value_derive/` — declarative settings with proc macros.

### 4.8 Telemetry and crash reporting
- `app/src/server/telemetry/{collector.rs, events.rs, rudder_message.rs, secret_redaction.rs}` — RudderStack pipeline.
- `send_telemetry_from_ctx!` / `send_telemetry_from_app_ctx!` macros re-exported from `warp_core`.
- `app/src/crash_reporting/mac.rs` (Sentry Cocoa via FFI), `sentry_minidump.rs` (Linux/Windows).

### 4.9 Other product surfaces
- `app/src/search/` (30 files) — palette, command search, file search, project search.
- `app/src/billing/`, `app/src/pricing/`, `app/src/usage/` — monetization.
- `app/src/onboarding/`, `app/src/tips/`, `app/src/banner/` — new-user flows.
- `app/src/themes/` — theme system.
- `app/src/persistence/sqlite.rs` (3708 lines) — all Diesel read/write logic.

---

## 5. Technical Debt

### 5.1 Very large files
The following files are strong decomposition candidates:

| File | Lines |
|---|---|
| `app/src/code_review/code_review_view.rs` | 7881 |
| `app/src/drive/index.rs` | 5646 |
| `app/src/persistence/sqlite.rs` | 3708 |
| `app/src/root_view.rs` | 3605 |
| `app/src/code_review/diff_state.rs` | 3016 |
| `app/src/lib.rs` | 2889 |
| `app/src/menu.rs` | 2734 |
| `app/src/ai/conversation_details_panel.rs` | 2010 |
| `app/src/ai/agent_conversations_model.rs` | 1954 |
| `app/src/settings/ai.rs` | 1984 |

`code_review_view.rs` and `drive/index.rs` are especially problematic — nearly 8000 and 5600 lines of UI+logic in a single file.

### 5.2 God function: `run_internal()`
`app/src/lib.rs:run_internal()` constructs the entire application graph by hand — modals, models, views, services — in a single function body. This is an initialization-order minefield and is nearly impossible to test in isolation. `init_common()` has an explicit comment warning contributors to be careful adding to it.

### 5.3 Feature-flag explosion
- `app/Cargo.toml` defines **350+** Cargo features.
- `crates/warp_features/src/lib.rs` enumerates **259** runtime `FeatureFlag` variants (1204 lines).
- The `enabled_features()` function in `app/src/lib.rs:2418-2889` is ~470 lines of repetitive `#[cfg(feature = "...")]` arms mapping Cargo features to runtime flags.

While there is a lifecycle for flags (`promote-feature`, `remove-feature-flag` skills), the sheer number creates significant maintenance burden.

### 5.4 Pinned forks and git-revision dependencies
The `[patch.crates-io]` section pins forked versions of: `core-foundation`, `core-graphics`, `core-text`, `objc`, `pathfinder_simd`, `yaml-rust`, `tink-{core,proto,hybrid}`, `tikv-jemallocator`. Many top-level deps use git revisions without versions: `cosmic-text`, `vte`, `font-kit`, `winit`, `dwrote`, `notify`, `rmcp`, `warp_multi_agent_api`, `session-sharing-protocol`. Reproducibility depends entirely on `Cargo.lock`; each fork represents upstream maintenance debt.

### 5.5 Dead and deprecated code
- 149 `#[allow(dead_code)]` annotations in `app/src/` (65 in `crates/`) indicate large feature-gated branches not exercised on every build.
- Deprecated internal APIs: `app/src/auth/auth_manager/user_persistence.rs:28` (`use auth_tokens.refresh_token instead`), `app/src/menu.rs:1270` (submenus not ready), deprecated fields in `crates/ai/src/agent/`.

### 5.6 TODO / FIXME density
~672 TODO/FIXME/HACK occurrences in `app/src/`, ~343 in `crates/`. Highest concentrations:
- `app/src/terminal/view.rs` (29 TODOs)
- `app/src/terminal/input.rs` (21 TODOs)
- `app/src/workspace/view.rs` (17 TODOs)
- `app/src/code/editor/model.rs` (12 TODOs)

### 5.7 Channel binary duplication
Five channel binaries (`oss.rs`, `local.rs`, `dev.rs`, `stable.rs`, `preview.rs`) each embed a nearly-identical ~30-line macOS Info.plist literal. Differences are only Bundle ID, URL scheme, and copyright year — trivially deduplicated via a macro.

---

## 6. Platform Support

### 6.1 Supported targets
- **macOS** — primary platform, most native code.
- **Linux** (including FreeBSD-style freedesktop).
- **Windows**.
- **WebAssembly (`wasm32-unknown-unknown`)** — "Warp on Web".

### 6.2 Conditional compilation — three styles

**a) Cargo target selectors in `app/Cargo.toml`:**
```toml
[target.'cfg(target_os = "macos")'.dependencies]
# cocoa, mach2, objc, objc2-foundation, permissions, security-framework-sys

[target.'cfg(not(target_os = "macos"))'.dependencies]
# winit, wgpu

[target.'cfg(not(target_family = "wasm"))'.dependencies]
# axum, tokio, git2, aws-config, tantivy, rmcp, syntect, sysinfo

[target.'cfg(target_family = "wasm")'.dependencies]
# console_error_panic_hook, js-sys, wasm-bindgen, web-sys, gloo

[target.'cfg(unix)'.dependencies]
# cvt, signal-hook, nix

[target.'cfg(any(target_os = "linux", target_os = "freebsd", target_os = "windows"))'.dependencies]
# crash-handler, minidumper

[target.'cfg(target_os = "windows")'.dependencies]
# windows 0.62, windows-registry, winreg, bytemuck
```

**b) Build script feature injection (`app/build.rs:227-240`):**
```rust
fn add_features(target_family: &str, target_os: &str) {
    if target_family != "wasm" {
        println!("cargo:rustc-cfg=feature=\"local_fs\"");
        println!("cargo:rustc-cfg=feature=\"local_tty\"");
    }
    if target_os != "windows" {
        println!("cargo:rustc-cfg=feature=\"iterm_images\"");
    }
    // ...
}
```
`cfg_aliases::cfg_aliases!` in `build.rs:20-23` defines aliases like `linux_or_windows` and `enable_crash_recovery`.

**c) Inline `#[cfg(...)]` throughout source** (`app/src/lib.rs` examples):
```rust
#[cfg(target_os = "macos")] mod app_menus;
#[cfg(target_family = "wasm")] mod wasm_nux_dialog;
#[cfg(any(target_os = "macos", target_os = "windows"))] mod login_item;
#[cfg(windows)] mod dynamic_libraries;
#[cfg(feature = "plugin_host")] pub use plugin::run_plugin_host;
```

### 6.3 Platform-specific source trees

**UI runtime** (`crates/warpui/src/platform/`):
- `mac/` — largest platform dir: `app.rs`, `clipboard.rs`, `delegate.rs`, `event.rs`, `fonts.rs`, `keycode.rs`, `menus.rs`, `objc/`, `rendering/`, `window.rs`.
- `linux/` — single `mod.rs` (logic shared via winit/wgpu).
- `windows/` — `mod.rs` (winit + dwrote).
- `wasm/` — `hidden_input.rs`, `mobile_detection/`, `soft_keyboard.rs`.
- `headless/` — for integration tests and CI.

**Rendering:**
- macOS: Metal + `core-graphics` + `core-text`, with custom Objective-C glue compiled by `app/build.rs:44-47` via `cc` (`app_bundle.m`, `services.m` → `libwarp_objc.a`).
- Linux/Windows/wasm: `wgpu` + `winit` + `cosmic-text` + `fontdb`.
- Windows additionally: `dwrote` (DirectWrite).
- Linux additionally: `fontconfig`, `x11rb`, `zbus`, `ashpd` (XDG portals).

**Crash reporting:**
- macOS: Sentry Cocoa framework via FFI (`app/src/crash_reporting/mac.rs`).
- Linux/Windows: `minidumper` + `crash-handler` (`app/src/crash_reporting/sentry_minidump.rs`).

**Agent computer use** (`crates/computer_use/src/`): dedicated `mac/`, `linux/`, `windows/` implementations, with `noop.rs` fallback.

### 6.4 Channel system

The `Channel` enum (`crates/warp_core/src/channel/mod.rs`) has six values: `Stable`, `Preview`, `Dev`, `Local`, `Oss`, `Integration`. Key behavioral differences:
- `is_dogfood()` — only `Dev`/`Local`.
- `allows_server_url_overrides()` — only `Dev`/`Local`/`Integration`; release channels ignore env vars so shipped builds cannot be redirected.
- `is_release_bundle()` — gates whether `RELEASE_FLAGS` are added to the runtime feature set.
- CLI command name: `oz` (Stable), `oz-dev`, `oz-preview`, `oz-local`, `oz-integration`, `warp-oss`.

### 6.5 Runtime feature flag system

`crates/warp_features/src/lib.rs` defines 259 `FeatureFlag` variants backed by atomics, indexed by the `enum-iterator`-derived `Sequence`. Flags can be toggled at runtime via `RUNTIME_FEATURE_FLAGS`. Channel defaults flow through `crates/warp_core/src/channel/state.rs` with named sets: `RELEASE_FLAGS`, `DEBUG_FLAGS`, `DOGFOOD_FLAGS`, `PREVIEW_FLAGS`.

---

## Conclusion

### What the project does and for whom
Warp is a **GPU-accelerated, AI-native terminal emulator** targeting professional developers on macOS, Linux, Windows, and the web. Its distinguishing features are a block-based command history (each command/output is an addressable object), deep AI agent integration (autonomous shell agents, code review, multi-LLM support), cloud sync of terminal sessions and notebooks, and collaborative session sharing. The product targets individual developers and engineering teams, with a freemium + team subscription monetization model.

### Architecture in broad strokes
The application is a **monolithic Rust GUI client** (~60 crates in a single Cargo workspace) built on a custom reactive UI framework (`warpui`/`warpui_core`) that follows the Zed/GPUI Entity+View+AppContext pattern. There is no webview and no Electron. On macOS the renderer uses Metal; everywhere else it uses `wgpu`. Platform-specific code is isolated in `crates/warpui/src/platform/{mac,linux,windows,wasm}/`. The client speaks GraphQL+WebSocket to Warp's cloud backend, stores all local state in a bundled SQLite database via Diesel, and spawns sub-processes for the terminal pty, JS plugin host, and crash handler via a typed IPC layer. Feature gating uses 259 runtime feature flags backed by atomics, layered on top of ~350 Cargo features.

### Three most important technical dependencies

1. **`wgpu` 29** — the cross-platform GPU rendering foundation that makes it possible to build a single high-performance renderer targeting Metal, Vulkan, DX12, and WebGL without maintaining four separate rendering backends. Without it, the non-Mac platforms would require entirely separate rendering implementations.

2. **`diesel` 2.3 + bundled SQLite** — provides the entire local persistence layer with type-safe schema evolution (134 migrations). All agent conversations, terminal history, blocks, MCP server configs, and user preferences live here. Using bundled SQLite means zero external database dependency for users.

3. **`warp_multi_agent_api` (private protobuf service)** — the typed contract between the client and Warp's AI routing backend. It abstracts over all LLM providers (Anthropic, OpenAI, Google, Bedrock) and provides the orchestration primitives (`AutonomyLevel`, `IsolationLevel`, `Task`, `ResponseEvent`, `ToolType`) that power the entire agent system. Every AI feature in the product is built on this API.

### One surprising thing about the structure
The repository contains a `specs/` directory with **146 product/tech spec pairs** (`PRODUCT.md` + `TECH.md` per ticket, e.g. `APP-1234/`), alongside a `.claude/`, `.warp/`, and `.agents/` directory of repository-specific AI agent skills (`promote-feature`, `remove-feature-flag`, `add-telemetry`, `warp-integration-test`, `review-pr-local`, etc.). This reveals that the entire development workflow — from spec writing to implementation to PR review to feature flag promotion — is deeply integrated with AI agents as first-class contributors, not just productivity tools. The codebase is actively being developed *with* the AI systems it is building *for*.
