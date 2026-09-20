# AGENTS.md — rtk (Rust Token Killer)

Instructions for any coding agent (Codex, Claude, Copilot, Cursor, Gemini, …) working in
this repository. `CLAUDE.md` is the Claude-specific twin; keep the two in sync.
**When this file and the source tree disagree, the source tree wins** — then fix the file.

> **Read `.claude/rules/rust-patterns.md` and `.claude/rules/cli-testing.md` before writing
> code.** Claude Code auto-loads them; **every other agent must open them explicitly.** The
> non-negotiable subset is inlined below so a missed load is not a silent loss, but the full
> files carry the worked examples. `.claude/rules/search-strategy.md` is written against
> Claude Code's own Grep/Glob/Read tools — treat its *priority order* as advice and ignore
> its tool names if your harness has different ones.

## Project overview

**rtk (Rust Token Killer)** is a high-performance CLI proxy that minimizes LLM token
consumption by filtering and compressing command output — 60-90% reduction on common
development operations through smart filtering, grouping, truncation and deduplication.

**All percentages in this repo measure bash output, not your bill.** rtk ships no tokenizer
(`src/core/tracking.rs` estimates tokens as `bytes / 4`), so the ratios are reliable but the
absolute token counts are approximate.

This is a fork with fixes for git argument parsing and modern JavaScript stack support
(pnpm, vitest, Next.js, TypeScript, Playwright, Prisma).

### Name collision warning

Two unrelated projects are called "rtk":

- **This one** — Rust Token Killer (`rtk-ai/rtk`)
- `reachingforthejack/rtk` — Rust Type Kit, generates Rust types. Different project.

```bash
rtk --version   # this project reports 0.42.x (see Cargo.toml for the current version)
rtk gain        # token-savings stats — "command not found" means the wrong package
```

## Commands

> If rtk is installed, prefer `rtk <cmd>` over the raw command for token-optimized output.
> Passthrough works even for subcommands rtk doesn't specifically handle.

```bash
# Build & run
cargo build                      # rtk cargo build  (preferred)
cargo build --release
cargo run -- <command>
cargo install --path .           # install locally

# Test
cargo test                       # rtk cargo test  (preferred)
cargo test <name>                # a single test
cargo test <module>::            # a module
cargo test --all                 # everything, incl. top-level tests/*.rs
cargo test --ignored             # real-process integration tests (needs rtk installed)
bash scripts/test-all.sh         # smoke tests (installed binary required)

# Lint & format
cargo check
cargo fmt --all
cargo clippy --all-targets

# Packaging
cargo deb                        # needs cargo-deb
cargo generate-rpm               # needs cargo-generate-rpm, after a release build
```

### The pre-commit gate is mandatory

```bash
cargo fmt --all && cargo clippy --all-targets && cargo test --all
```

After **any** Rust edit, run all three before committing. **Zero tolerance on clippy
warnings** — fix them before moving on. If the build fails, fix it immediately rather than
continuing to the next task.

For filter changes, verify performance did not regress:

```bash
hyperfine 'rtk git log -10' --warmup 3            # before
cargo build --release
hyperfine 'target/release/rtk git log -10' --warmup 3   # after — still <10ms
```

### What CI enforces (`.github/workflows/ci.yml`)

| Job | What it does |
|---|---|
| **test presence** | every filter module must carry tests — a new `src/cmds/**` filter with no `#[cfg(test)]` fails the build |
| **fmt** | `cargo fmt --all -- --check` |
| **clippy** | `cargo clippy --all-targets` |
| **test** | `cargo test --all`, matrixed across operating systems |
| **Security Scan** | `cargo-audit` CVE check, critical-files check, dangerous-pattern scan, new-dependency check, clippy security lints, summary verdict |
| **semgrep** | `semgrep scan --config .semgrep.yml` against the PR base commit |
| **benchmark** | builds release and benchmarks against real tooling (tree, ruff/pytest/mypy, Go) |

The first four are exactly the pre-commit gate plus `fmt --check`. Running the gate locally
is the difference between one push and three.

## Architecture

rtk uses a **command proxy architecture**: `main.rs` routes CLI commands through a Clap
`Commands` enum to specialized filter modules in `src/cmds/*/`, each of which executes the
underlying command and compresses its output. Savings are tracked in SQLite via
`src/core/tracking.rs`.

- [`docs/contributing/ARCHITECTURE.md`](docs/contributing/ARCHITECTURE.md) — system design,
  module organization, filtering strategies, error handling
- [`docs/contributing/TECHNICAL.md`](docs/contributing/TECHNICAL.md) — end-to-end flow,
  folder map, hook system, filter pipeline
- [`src/cmds/README.md`](src/cmds/README.md#adding-a-new-command-filter) — step-by-step
  checklist for adding a filter
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — contribution workflow and design philosophy

Per-module responsibilities live in each folder's `README.md` and each file's `//!` doc
header. Browse `src/cmds/*/` to discover the available filters.

**Supported ecosystems:** git/gh/gt, cargo, go/golangci-lint, npm/pnpm/npx,
ruff/pytest/pip/mypy, rspec/rubocop/rake, dotnet, playwright/vitest/jest,
docker/kubectl/aws, gradlew/mvn, php/artisan/phpunit/phpstan/pest.

### Proxy mode

`rtk proxy <command> [args...]` executes a command **without** filtering while still tracking
usage. Use it to bypass a filter bug, get genuinely full output, or measure which commands an
agent reaches for. Proxy calls appear in `rtk gain --history` at 0% reduction (input = output).

```bash
rtk proxy git log --oneline -20     # full git log, no truncation
rtk proxy npm install express       # raw npm output
```

## Non-negotiable Rust rules

Inlined from `.claude/rules/rust-patterns.md`. These **override general Rust conventions**;
read that file for the worked examples behind each.

1. **No async.** Zero `tokio`, `async-std`, `futures`. Single-threaded by design — async
   costs 5-10 ms of startup against a <10 ms budget.
2. **No `unwrap()` in production.** Use `.context("description")?`. In tests, `expect("reason")`.
   The one sanctioned exception is `lazy_static!` regex initialization, where a bad literal is
   a programming error caught at first use.
3. **Lazy regex.** `Regex::new()` inside a function recompiles on every call. Always
   `lazy_static!`.
4. **Fallback pattern — mandatory for every filter.** If filtering fails, print the raw
   command output unchanged and warn on stderr. **Never block the user**, and never swallow
   an error into `Err(_) => {}` — in rtk that means the user gets *no output at all*.
5. **Propagate exit codes.** `std::process::exit(code)` when the underlying command fails,
   or CI thinks a failed command succeeded.
6. **`anyhow::Result` everywhere**, always with `.context()`.
7. **Borrow over clone**, iterators over manual loops, `&str` over `&String` in signatures.
8. **No `println!` in the filter path** — a debug artifact lands in the user's output. Use
   `eprintln!`.

Every `*_cmd.rs` follows one shape: imports → args struct → `lazy_static!` regexes → public
`run()` → private filter fns → `#[cfg(test)] mod tests` (always present).

## Testing rules

Inlined from `.claude/rules/cli-testing.md`; read it for the full patterns.

- **Unit tests are colocated** in the filter's own `#[cfg(test)] mod tests`, asserting
  directly with `assert_eq!` / `assert!`. Cover the common case plus at least one edge case
  (empty input, error output, malformed input, unicode, ANSI codes) — edge cases must **not
  panic**; best-effort output or unchanged passthrough are both acceptable.
- **Two fixture styles, both valid.** Inline literal strings for small cases (most of
  `src/cmds/**`); real captured output via `include_str!` from `tests/fixtures/` once the case
  is large or format-sensitive — `src/cmds/jvm/mvn_cmd.rs` is the reference (23+ fixtures).
  Capture fixtures from **real command output**, never synthetic data.
- **Token-accuracy tests are mandatory** on every filter. There is a single enforced floor —
  **≥60% savings is a release blocker** — not a per-filter table. Do **not** assert an invented
  per-command percentage; doc tables of made-up numbers rot immediately.
- `count_tokens` is defined per test module. There is no shared `tests/common/mod.rs` — don't
  assume one.
- **Integration tests are top-level `tests/*.rs`**, not colocated (`grep_context_test.rs`,
  `guard_integration_test.rs`, `search_compress_test.rs`, …). Real-process ones are
  `#[ignore]`d; run with `cargo test --ignored` after `cargo install --path .`.
- **Cross-platform.** rtk must work on macOS (zsh), Linux (bash) and Windows (PowerShell), and
  shell escaping differs. Gate platform assertions with `#[cfg(target_os = …)]`. Test Linux and
  macOS locally; trust CI for Windows.
- **Performance targets:** <10 ms startup, <5 MB memory, <5 MB binary. Verify with `hyperfine`
  and `/usr/bin/time -l` (macOS) / `-v` (Linux). Investigate any startup increase over 2 ms.

## Working style

**Confirm the working directory before you start.** Never assume which project you are in:

```bash
pwd          # verify the rtk project root
git branch   # verify the branch
```

**Don't go down rabbit holes.** If verifying something external takes more than 3-4
exploratory commands, stop and ask whether to continue or trust what you have. Specifically:
don't hand-verify 20 regex edge cases (trust the tests), don't research git/cargo internals
(use fixtures), don't test beyond two platforms (trust CI), don't clone crates to check an
API signature (docs.rs is enough).

**When given a numbered plan** (QW1-QW4, Phase 1-5, sprint tasks):

1. Execute sequentially unless told otherwise.
2. Commit once per completed phase or task.
3. Never skip or reorder — if a step is blocked, report it and ask before proceeding.
4. Track progress in whatever task list your harness provides, for any plan of 3+ steps.
5. Before starting, verify every referenced path exists and the working directory is right.
