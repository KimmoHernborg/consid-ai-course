# Warp Repository Analysis

## Purpose

Warp is an open-source, AI-powered terminal and agentic development environment built in Rust. It evolved from a cloud-backed terminal into a full development platform that combines modern terminal emulation with AI orchestration, agent-to-agent workflows, and cloud synchronization. The project is licensed under AGPL v3 (application) and MIT (UI framework).

## Main Features

### Terminal & Shell

- Terminal emulation with multi-shell support and session management
- Advanced command completion (v2 system), autosuggestion, and correction
- Vim-mode editor, undo/redo for closed panes, shell selection UI
- Collaborative sessions with ACL-controlled sharing

### AI & Agent Mode

- Built-in Agent Mode with context-aware codebase indexing
- Computer use capabilities for agents (vision-based UI interaction)
- Agent-to-agent orchestration: local-to-cloud and cloud-to-cloud handoff
- Support for external agents: Claude Code, Codex, Gemini CLI
- Conversation persistence, agent management views, multi-agent workflows

### Code Development

- Integrated code editor with diff, code review UI, and save/revert support
- LSP support, symbol navigation, global search with full-text indexing
- File tree, project explorer, command palette, snippet/workflow support
- Linked code blocks and inline AI-assisted code review

### Cloud & Collaboration

- Warp Drive: cloud synchronization of objects across devices and teams
- API key management, billing/usage tracking, workspace sync
- Cloud mode for fully cloud-based development sessions

### Customization & Extensibility

- Theme system, keybinding customization, multi-profile management
- Plugin system with JavaScript runtime (QuickJS)
- MCP (Model Context Protocol) server integration
- BYOK (Bring Your Own Key) for LLM providers

## Notable Technologies

| Category              | Technologies                                                  |
| --------------------- | ------------------------------------------------------------- |
| Primary language      | Rust (dominant), TypeScript/JS (plugins, web), WASM           |
| UI framework          | WarpUI — custom Entity-Component-Handle system (MIT licensed) |
| Async runtime         | Tokio                                                         |
| Web/HTTP              | Axum                                                          |
| Database              | SQLite via Diesel ORM                                         |
| API layer             | GraphQL with schema codegen                                   |
| Terminal emulation    | VTE (escape sequence parsing)                                 |
| Syntax highlighting   | Syntect                                                       |
| Tree-sitter languages | Arborium (25+ languages)                                      |
| Full-text search      | Tantivy                                                       |
| Graphics              | wgpu (non-macOS), native rendering (macOS)                    |
| Agent protocol        | RMCP (multi-agent communication)                              |
| Error tracking        | Sentry                                                        |
| Serialization         | Serde, Bincode                                                |
| Testing               | Nextest, custom integration test framework, Mockall           |
| Memory allocator      | jemalloc                                                      |
| Platform FFI          | CoreFoundation (macOS), Win32 (Windows), X11rb/ZBus (Linux)   |

## Project Structure

```
warp/
├── app/src/          # Main application (~50+ modules: ai, terminal, editor, auth, drive, settings, ...)
├── crates/           # 63 specialized library crates
│   ├── warpui_core/  # Core UI framework (MIT)
│   ├── warpui/       # UI component library (MIT)
│   ├── warp_core/    # Utilities and feature flags
│   ├── warp_terminal/# Terminal emulation
│   ├── warp_editor/  # Text editor
│   ├── ai/           # AI/agent integration
│   ├── lsp/          # Language Server Protocol
│   ├── persistence/  # Database layer
│   ├── computer_use/ # Vision-based computer interaction
│   └── integration/  # Integration test framework
├── specs/            # Feature specs (PRODUCT.md + TECH.md per feature)
├── .agents/          # Agent skills and configurations
└── script/           # Build, test, and platform setup scripts
```

## Architectural Highlights

**Entity-Component-Handle (ECH) pattern**: Views use `ViewHandle<T>` references rather than direct ownership. A global `App` object owns all entities; `AppContext` provides temporary access during render/event handling. Inspired by Flutter's widget model.

**Feature flag system**: Runtime flags in `warp_core/src/features.rs` gate features by channel (Stable, Preview, Dogfood, Dev, OSS). Runtime checks are preferred over `#[cfg]` for rollout flexibility.

**Monorepo with 63 crates**: Each crate has a single focused responsibility. Workspace-level `Cargo.toml` manages shared dependencies. Clear split between MIT-licensed framework crates and AGPL application code.

**Spec-driven development**: Significant features start with `PRODUCT.md` + `TECH.md` specs before implementation. AI agents (Oz) assist with review; human SME review follows.

**Cross-platform**: Native macOS, Linux, and Windows implementations with shared core logic. WASM target for web-based Warp.

**Telemetry**: Named telemetry events system for product metrics, Sentry crash reporting, structured logging, optional CPU/heap profiling.
