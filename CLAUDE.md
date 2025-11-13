# Codex CLI Project Overview

## What is Codex CLI?

**Codex CLI** is OpenAI's official coding agent that runs locally on your computer. It's a powerful AI-powered development assistant distributed via npm (`@openai/codex`) and Homebrew, designed for ChatGPT Plus, Pro, Team, Edu, and Enterprise users.

### Key Capabilities

- Execute code and commands in a sandboxed environment
- Make file changes and modifications to codebases
- Interact with Git repositories
- Connect to Model Context Protocol (MCP) servers
- Provide interactive TUI (Terminal User Interface) mode
- Support non-interactive exec mode for automation
- Embeddable via TypeScript SDK for programmatic use

## Repository Structure

This is a **monorepo** with a hybrid Rust/TypeScript architecture:

```
/home/user/codex/
├── codex-rs/           # Rust workspace (main implementation)
│   ├── cli/            # Main CLI binary entry point
│   ├── core/           # Business logic and core functionality
│   ├── tui/            # Terminal UI (Ratatui-based)
│   ├── exec/           # Non-interactive/headless execution mode
│   ├── protocol/       # Communication protocol definitions
│   ├── common/         # Shared utilities
│   ├── app-server/     # Local server for IDE integration
│   ├── backend-client/ # Backend API client
│   ├── mcp-server/     # MCP server implementation
│   ├── mcp-types/      # MCP type definitions
│   ├── login/          # Authentication logic
│   ├── sandbox/        # Platform-specific sandboxing
│   │   ├── linux-sandbox/
│   │   └── windows-sandbox-rs/
│   └── utils/          # Various utility crates
├── codex-cli/          # NPM wrapper package
├── sdk/typescript/     # TypeScript SDK for programmatic use
├── docs/               # User documentation
├── scripts/            # Build and release scripts
└── .github/            # CI/CD workflows
```

### Rust Workspace Structure

The `codex-rs/` directory contains 41 workspace member crates:

**Core Components:**
- `cli` - Main CLI binary entry point (`codex-rs/cli/src/main.rs`)
- `core` - Business logic, tool execution, MCP client, config management
- `tui` - Full-screen interactive terminal interface (Ratatui-based)
- `exec` - Headless mode for automation
- `protocol` - Communication protocol definitions

**Security & Sandboxing:**
- `linux-sandbox` - Linux sandboxing (Landlock, seccomp)
- `windows-sandbox-rs` - Windows sandboxing (restricted tokens)
- macOS sandboxing uses Seatbelt (Core Foundation)
- `process-hardening` - Process hardening utilities

**Integration:**
- `mcp-server` - MCP server implementation
- `mcp-types` - MCP type definitions
- `rmcp-client` - MCP client
- `app-server` - Local WebSocket server for IDE integration
- `backend-client` - Backend API client

**Utilities:**
- `utils/git` - Git operations
- `utils/cache` - Caching utilities
- `utils/image` - Image processing
- `utils/pty` - PTY utilities
- `utils/tokenizer` - Tokenization
- `file-search` - File searching
- `apply-patch` - Patch application

All crate names are prefixed with `codex-` (e.g., the `core` folder's crate is `codex-core`).

## Technology Stack

### Rust Stack

**Language & Edition:**
- Rust 1.90.0 (2024 edition)
- See `codex-rs/rust-toolchain.toml`

**Core Dependencies:**
- **TUI Framework**: Ratatui 0.29.0 (terminal UI rendering)
- **Terminal**: Crossterm 0.28.1 (cross-platform terminal manipulation)
- **Async Runtime**: Tokio (multi-threaded async runtime)
- **HTTP Client**: Reqwest 0.12 (async HTTP)
- **Serialization**: Serde, serde_json
- **Configuration**: TOML (toml 0.9.5, toml_edit 0.23.4)
- **MCP**: rmcp 0.8.5 (Model Context Protocol)
- **Observability**: OpenTelemetry 0.30.0, tracing
- **Sandboxing**:
  - Linux: Landlock 0.4.1, seccompiler 0.5.0
  - macOS: Core Foundation (Seatbelt)
  - Windows: Custom restricted-token implementation

**Testing:**
- cargo-nextest (faster test runner)
- insta 1.43.2 (snapshot testing, especially for TUI)
- wiremock 0.6 (HTTP mocking)
- pretty_assertions 1.4.1 (better assertion diffs)

### TypeScript/Node.js Stack

**Package Manager & Runtime:**
- **pnpm**: 10.8.1 (workspace package manager)
- **Node.js**: >=22
- **Build Tool**: tsup 8.5.0 (TypeScript bundler)
- **Testing**: Jest 29.7.0
- **Linting**: ESLint 9.36.0
- **Formatting**: Prettier 3.5.3
- **MCP SDK**: @modelcontextprotocol/sdk 1.20.2

### Build Tools

- **Just**: Task runner (see `codex-rs/justfile`)
- **cargo-nextest**: Fast parallel test execution
- **cargo-insta**: Snapshot testing tool
- **cargo-shear**: Unused dependency detection
- **sccache**: Build caching for CI
- **GitHub Actions**: CI/CD automation

## Development Workflow

### Building

**Rust:**
```bash
cd codex-rs
just codex          # Run the CLI
just fmt            # Format code (auto-run after changes)
just fix -p <crate> # Fix linter issues for specific crate
just test           # Run tests with nextest
```

**TypeScript:**
```bash
pnpm install        # Install dependencies
pnpm format         # Check formatting
pnpm format:fix     # Fix formatting
```

### Testing

**Rust Testing:**
1. Run tests for specific project: `cargo test -p codex-<crate>`
2. For changes in common/core/protocol: `cargo test --all-features`
3. Use `cargo nextest run` for faster execution
4. Snapshot tests: `cargo insta accept -p codex-tui` (after reviewing changes)

**Test Utilities:**
- Integration tests use `core_test_support::responses`
- TUI tests use snapshot testing (insta)
- Mock SSE responses with `mount_sse*` helpers
- Use `pretty_assertions::assert_eq` for better diffs

**Important Test Notes:**
- Tests check for `CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR` to skip network tests
- Seatbelt tests check `CODEX_SANDBOX=seatbelt` to avoid nested sandboxing
- Never modify these environment variable checks

### Code Style

**Rust Conventions:**
- Collapse if statements (clippy rule)
- Inline format! args when possible
- Use method references over closures
- Never use unsigned integers even for non-negative numbers
- Prefer comparing entire objects over field-by-field comparisons
- All lints configured in `codex-rs/Cargo.toml` under `[workspace.lints.clippy]`

**TUI Styling (Ratatui):**
- Use Stylize trait helpers: `.dim()`, `.bold()`, `.cyan()`, `.italic()`
- Prefer `"text".into()` for simple spans
- Avoid hardcoded `.white()` - use default foreground
- Use `textwrap::wrap` for text wrapping
- See `codex-rs/tui/styles.md` for full conventions

## Configuration

**User Configuration:**
- Location: `~/.codex/config.toml`
- Rich TOML-based configuration system
- See `docs/config.md` for full options
- Example: `docs/example-config.md`

**Build Configuration:**
- `codex-rs/Cargo.toml` - Workspace dependencies and settings
- `codex-rs/clippy.toml` - Clippy linting rules
- `codex-rs/rustfmt.toml` - Formatting rules
- `codex-rs/justfile` - Common build commands

**CI/CD Configuration:**
- `.github/workflows/rust-ci.yml` - Rust CI pipeline
- `.github/workflows/rust-release.yml` - Release automation
- `.github/workflows/sdk.yml` - TypeScript SDK CI

## Distribution & Release

### Build Profiles

**Release Profile** (`[profile.release]`):
- LTO: "fat" (link-time optimization)
- Strip: "symbols" (remove debug symbols)
- codegen-units: 1 (single codegen unit for optimization)

**CI Test Profile** (`[profile.ci-test]`):
- debug: 1 (reduced debug symbols)
- opt-level: 0 (faster compilation)

### Supported Platforms

Codex builds for 8 target platforms:
- **macOS**: aarch64-apple-darwin, x86_64-apple-darwin
- **Linux**: x86_64-unknown-linux-musl, x86_64-unknown-linux-gnu, aarch64-unknown-linux-musl, aarch64-unknown-linux-gnu
- **Windows**: x86_64-pc-windows-msvc, aarch64-pc-windows-msvc

### Distribution Channels

1. **npm**: `@openai/codex` (wraps native binaries)
   - Install: `npm install -g @openai/codex`
   - NPM wrapper in `codex-cli/`

2. **Homebrew**: `codex` (cask)
   - Install: `brew install --cask codex`

3. **GitHub Releases**: Direct binary downloads
   - Tagged with `rust-v*.*.*` format

### Release Process

1. Tag with `rust-v*.*.*` (e.g., `rust-v0.1.0`)
2. CI validates tag matches `Cargo.toml` version
3. Cross-compilation builds for all 8 platforms
4. Binaries uploaded to GitHub Releases
5. NPM package staged and published

## Key Features

### Sandboxing Modes

- **read-only**: Read-only access to workspace
- **workspace-write**: Write access to workspace only
- **full-access**: Full system access

Platform-specific implementations:
- Linux: Landlock LSM + seccomp BPF
- macOS: Seatbelt sandbox profiles
- Windows: Restricted tokens

### Authentication Methods

- ChatGPT SSO (recommended)
- OpenAI API key
- Device code flow
- See `docs/authentication.md`

### MCP Integration

- Acts as MCP client (connects to MCP servers)
- Acts as MCP server (can be used by other MCP clients)
- Configuration in `~/.codex/config.toml`
- See `docs/advanced.md#model-context-protocol-mcp`

### Modes of Operation

1. **Interactive TUI**: Full-screen terminal interface
   - Run: `codex`
   - See: `codex-rs/tui/`

2. **Exec Mode**: Non-interactive automation
   - Run: `codex exec "<prompt>"`
   - See: `docs/exec.md`, `codex-rs/exec/`

3. **TypeScript SDK**: Programmatic API
   - Package: `@openai/codex-sdk`
   - See: `sdk/typescript/`

4. **App Server**: Local WebSocket server for IDE integration
   - See: `codex-rs/app-server/`

## Important Files & Entry Points

### Rust Entry Points

- `codex-rs/cli/src/main.rs` - Main CLI binary entry point
- `codex-rs/tui/src/main.rs` - TUI entry point
- `codex-rs/exec/src/lib.rs` - Exec mode library

### TypeScript Entry Points

- `sdk/typescript/src/index.ts` - SDK public API
- `codex-cli/bin/codex.js` - NPM wrapper script

### Documentation

- `README.md` - Main project README
- `AGENTS.md` - Development guidelines and conventions
- `docs/getting-started.md` - Getting started guide
- `docs/config.md` - Configuration reference
- `docs/sandbox.md` - Sandbox & approvals
- `docs/authentication.md` - Authentication methods
- `docs/exec.md` - Non-interactive mode
- `docs/contributing.md` - Contributing guidelines
- `docs/install.md` - Installation & build instructions
- `docs/faq.md` - Frequently asked questions

### Build Scripts

- `scripts/stage_npm_packages.py` - NPM package staging
- `codex-cli/scripts/build_npm_package.py` - Build NPM distribution

## Development Guidelines Summary

See `AGENTS.md` for full guidelines. Key points:

- All crate names prefixed with `codex-`
- Run `just fmt` after code changes (no approval needed)
- Run `just fix -p <project>` before finalizing changes
- Use `cargo test -p codex-<crate>` for project-specific tests
- Use `cargo insta accept -p codex-<crate>` for snapshot updates
- Never modify sandbox environment variable checks
- Follow Rust 2024 edition conventions
- Use Ratatui Stylize helpers for TUI styling
- Use `textwrap` for text wrapping in TUI

## External Resources

- **Main Website**: https://chatgpt.com/codex
- **IDE Integration**: https://developers.openai.com/codex/ide
- **GitHub Repository**: https://github.com/openai/codex
- **npm Package**: https://www.npmjs.com/package/@openai/codex
- **Documentation**: See `docs/` folder
- **Issues**: https://github.com/openai/codex/issues

## Architecture Notes

This is a production-grade, enterprise-ready tool with:
- Sophisticated security sandboxing per platform
- Comprehensive test coverage (unit, integration, snapshot)
- Multi-platform support (macOS, Linux, Windows, ARM)
- Modern async Rust architecture (Tokio)
- Rich TUI with Ratatui
- Extensible MCP integration
- TypeScript SDK for embedding
- Professional release automation

The codebase follows best practices for Rust development with strict clippy rules, snapshot testing for UI, and comprehensive CI/CD workflows.
