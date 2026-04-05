# Claude Code — Clean-Room Rust Reimplementation

[![Rust](https://img.shields.io/badge/Rust-1.75%2B-orange?logo=rust)](https://www.rust-lang.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

A clean-room Rust reimplementation of Claude Code's behavior — Anthropic's AI-powered coding CLI. This project reproduces the functionality of the original TypeScript tool as idiomatic, high-performance Rust, built entirely from behavioral specifications without referencing the original source code.

## Overview

This project was developed using a strict two-phase clean-room process:

1. **Specification** ([`spec/`](spec/)) — An AI agent analyzed publicly available behavior and produced exhaustive behavioral specifications covering architecture, data flows, tool contracts, and system designs. No source code was carried forward.

2. **Implementation** ([`src-rust/`](src-rust/)) — A separate AI agent implemented the system from the spec alone, never referencing the original TypeScript. The result is idiomatic Rust that reproduces the *behavior*, not the *expression*.

## Features

- **Interactive Terminal UI** — Full TUI built with [ratatui](https://github.com/ratatui/ratatui) and [crossterm](https://github.com/crossterm-rs/crossterm), featuring syntax highlighting, streaming responses, permission dialogs, and cost tracking
- **25+ Tools** — Shell execution (Bash/PowerShell), file I/O, glob/grep search, web fetch, MCP resources, task management, cron scheduling, and more
- **Agentic Query Loop** — Streams Claude API responses, detects tool-use requests, dispatches tools, and feeds results back automatically
- **Auto-Compaction** — Automatically compacts conversation context when the token window fills up
- **Multi-Agent Support** — Coordinator mode for parallel task delegation across worker agents
- **Memory Consolidation** — Background "dream" system that consolidates session memories into durable storage
- **MCP Integration** — Full [Model Context Protocol](https://modelcontextprotocol.io/) client with JSON-RPC 2.0, supporting stdio and HTTP/SSE transports
- **Bridge Protocol** — Remote control integration with claude.ai via JWT-authenticated long-polling
- **Buddy Companion** — Tamagotchi-style companion pet system with deterministic gacha mechanics
- **100+ Slash Commands** — `/help`, `/compact`, `/clear`, `/model`, `/config`, `/cost`, `/rewind`, `/oauth`, and many more
- **OAuth Authentication** — Built-in OAuth flow for Anthropic API access

## Architecture

The project is organized as a Rust workspace with 10 crates:

```
src-rust/
├── Cargo.toml              # Workspace root
└── crates/
    ├── cli/                # Binary entry point — arg parsing, REPL loop, OAuth
    ├── core/               # Types, configuration, errors, message models
    ├── api/                # Anthropic API client with SSE streaming
    ├── tools/              # 25+ tool implementations (bash, file ops, search, web, etc.)
    ├── query/              # Agentic conversation loop, tool dispatch, auto-compaction
    ├── tui/                # Terminal UI — ratatui rendering, input handling, themes
    ├── commands/           # Slash command framework and implementations
    ├── mcp/                # Model Context Protocol client (JSON-RPC 2.0)
    ├── bridge/             # Remote control bridge (JWT auth, long-polling)
    └── buddy/              # Companion pet system (deterministic PRNG-based)
```

## Specifications

The [`spec/`](spec/) directory contains ~990 KB of detailed behavioral specifications across 14 markdown files:

| File | Contents |
|------|----------|
| `00_overview.md` | Master architecture, repo structure, data flow |
| `01_core_entry_query.md` | Entry points, query engine, token budget |
| `02_commands.md` | All 100+ slash commands |
| `03_tools.md` | All 40+ tools with schemas and permissions |
| `04_components_core_messages.md` | UI components and message rendering |
| `05_components_agents_permissions_design.md` | Agent creation, permission dialogs |
| `06_services_context_state.md` | Analytics, API client, memory, state |
| `07_hooks.md` | All 104 React hooks (reference) |
| `08_ink_terminal.md` | Terminal framework internals |
| `09_bridge_cli_remote.md` | Bridge protocol, JWT auth, transports |
| `10_utils.md` | ~564 utility functions |
| `11_special_systems.md` | Buddy, memory, keybindings, skills, voice |
| `12_constants_types.md` | Constants, types, system prompts, betas |
| `13_rust_codebase.md` | Rust implementation details |

## Getting Started

### Prerequisites

- [Rust](https://www.rust-lang.org/tools/install) 1.75 or later
- An Anthropic API key or OAuth credentials

### Build

```bash
cd src-rust
cargo build --release
```

### Run

```bash
# Interactive mode
cargo run --release

# Headless mode (single query)
cargo run --release -- --print "Explain this codebase"

# With a specific model
cargo run --release -- --model claude-sonnet-4-20250514
```

### Configuration

Settings are loaded from `~/.config/claude/settings.json`. Key options include:

- `apiKey` — Anthropic API key
- `model` — Default model (e.g., `claude-sonnet-4-20250514`)
- `permissionMode` — `default`, `auto`, or `bypass`
- `mcpServers` — MCP server configurations

## Key Dependencies

| Crate | Purpose |
|-------|---------|
| `tokio` | Async runtime |
| `reqwest` | HTTP client with SSE streaming |
| `ratatui` / `crossterm` | Terminal UI framework |
| `clap` | CLI argument parsing |
| `serde` / `serde_json` | Serialization |
| `syntect` | Syntax highlighting |
| `similar` | Diff computation for file edits |
| `tracing` | Structured logging |

## Legal

This project follows the clean-room engineering methodology established by *Phoenix Technologies v. IBM* (1984) and the principle from *Baker v. Selden* (1879) that copyright protects expression, not ideas or behavior. The specification phase produced behavioral descriptions; the implementation phase produced original Rust code from those descriptions alone.

## License

MIT

---

*Originally created by [Kuber Mehta](https://kuber.studio/)*