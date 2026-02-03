# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

rustypaste-cli is a command-line tool (`rpaste`) for uploading files and URLs to a [rustypaste](https://github.com/orhun/rustypaste) server. It supports file uploads, URL shortening, remote URL pasting, stdin input, file deletion, and server file listing.

## Build Commands

```bash
cargo build                                        # Debug build
cargo build --release                              # Release build (LTO enabled)
cargo build --release --features use-native-certs  # With OS trust store for TLS
cargo test                                         # Run all tests
cargo test test_parse_token                        # Run a specific test
cargo fmt --check                                  # Check formatting (CI enforced)
cargo clippy --tests -- -D warnings                # Check lints (CI enforced, warnings are errors)
```

## Architecture

Six modules in `src/`:

- **main.rs** - Entry point: parses `Args`, calls `rustypaste_cli::run()`
- **lib.rs** - `run()` orchestrates the entire flow: load config → parse token files → merge CLI args → dispatch action (upload/delete/list/version)
- **args.rs** - CLI argument parsing with `getopts`; `Args::parse()` handles flags, options, and `RPASTE_CONFIG` env var
- **config.rs** - TOML config via `serde`; `Config` has `ServerConfig`, `PasteConfig`, `StyleConfig`; `PasteConfig::filename` is `#[serde(skip_deserializing)]` (set only from CLI args)
- **error.rs** - `thiserror`-based `Error` enum and `Result<T>` type alias
- **upload.rs** - `Uploader` struct wraps `ureq` HTTP client; all operations return `UploadResult<'a, T>(&'a str, Result<T>)` pairing the input label with its result

**Key patterns:**
- CLI args override config file values (`Config::update_from_args`)
- Config lookup order: `RPASTE_CONFIG` env var → `-c` flag → `~/.rustypaste/config.toml` → platform config dir
- On macOS, `XDG_CONFIG_HOME` is honored explicitly since `dirs-next` ignores it (see `retrieve_xdg_config_on_macos`)
- Token files support tilde expansion via `shellexpand` and have whitespace trimmed
- Reference config at repo root: `config.toml`

## Code Quality

- `#![warn(missing_docs)]` is enabled in lib.rs
- `#![warn(clippy::unwrap_used)]` — avoid unwrap, use proper error handling
- All clippy warnings treated as errors in CI
- CI also runs `cargo-audit` for security vulnerability checks
