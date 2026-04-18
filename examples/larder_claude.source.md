# CLAUDE.md

This file is guidance for Claude Code when contributing to **Larder**, a tiny CLI for tracking what's in your pantry.

## Project overview

Larder is a single-binary Rust CLI (`larder`) with a SQLite backing store at `~/.larder.db`. It runs offline only — no network, no telemetry.

## Build & test

```bash
cargo build --release      # Build release binary
cargo test                 # Run unit tests
cargo run -- list          # Show pantry contents
```

## Rules

- Every changed line must trace back to an issue or explicit user request. Don't refactor adjacent code, reformat, or tidy up things you weren't asked to touch.
- Don't add dependencies without justification in the PR description. Default to the Rust std library first.
- Run `cargo fmt && cargo clippy --all-targets -- -D warnings` before every commit.
- Write a failing test before fixing any bug. Landing a fix without a reproduction is rejected in review.
- Don't log the contents of `~/.larder.db` or its path at any log level. Users' pantry contents are private.

## Golden rules

- Offline-only. If you find yourself reaching for `reqwest` or `ureq`, stop — Larder has no network features.
- SQLite migrations are append-only. Never edit a past migration; add a new one.
- Treat all CLI input as untrusted. Pass values through `rusqlite` parameters, never via string concatenation.

## Thresholds

- Keep individual functions under 40 lines.
- Target 80% line coverage on new code (run `cargo tarpaulin --lib`).
- CLI commands should return in under 200ms on a 10k-row pantry.

## Workflow

- Branch off `main`; PRs go into `main` via squash-merge.
- Before opening a PR, run the full test matrix: `cargo test && cargo clippy --all-targets`.
- Tag releases as `v0.X.Y` and attach binaries built via `./scripts/release.sh`.

## What NOT to do

- Don't introduce `unsafe` blocks without team review.
- Don't commit generated files.
- Don't push to `main` directly — all changes land via PR.
