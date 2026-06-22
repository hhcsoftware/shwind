# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status: greenfield

The repo is empty (only this file and a placeholder `README.md`). Nothing below describes
existing code — it records the stack and architecture decisions for the build, which were
delegated in the project brief. When you scaffold something, follow these conventions; when a
decision here proves wrong in practice, change it here in the same PR so this file stays true.

## What we're building

`shwind` — a terminal companion that quietly tracks how you actually spend time in the shell:
duration per command, per running process, per git branch, per repo; activity heatmaps (hour ×
weekday); and an end-of-period **"Wrapped"** summary. Think *Spotify Wrapped, but for your CLI*.

Tone: minimal, utilitarian, privacy-first, with a light quirky streak in copy and the Wrapped
output. Do **not** over-brand — no taglines, no logo ASCII art, no marketing voice in code or docs.

## Tech stack (decided)

- **Core CLI + daemon:** Go. [Cobra](https://github.com/spf13/cobra) for the command tree.
- **Local store:** SQLite via **`modernc.org/sqlite`** (pure-Go, **no CGO**) — this is deliberate so
  the binary cross-compiles and ships as a single static file. Do not switch to a CGO driver
  (`mattn/go-sqlite3`) without a strong reason; it breaks easy distribution.
- **Local web server:** Go stdlib `net/http` (add `chi` only if routing gets real). Serves a JSON
  API and embeds the built web UI via `embed.FS`, so `shwind serve` is one binary, no Node at runtime.
- **Web dashboard:** Next.js (App Router) + TypeScript + Tailwind. Charts: prefer hand-rolled SVG or
  a light lib (visx/Recharts) over a heavy charting framework. Static-exportable so it can be
  embedded; the same app can point at Neon for a hosted build.
- **Optional cloud:** NeonDB (serverless Postgres) for the hosted / shareable Wrapped. Reached only
  through `shwind sync` — never on the command hot path. See "Two modes" below.
- **Shell integration:** zsh + bash first (`preexec`/`precmd` hooks), fish later.

## Architecture — the big picture

Three pieces talk through the local SQLite DB; understanding their boundary matters more than any
single file:

1. **Shell hooks → ingest (the hot path).** A sourced shell script (`shell/shwind.zsh`,
   `shell/shwind.bash`) fires on every command via `preexec`/`precmd` and shells out to the Go
   binary (e.g. `shwind hook preexec` / `shwind hook precmd`). **This runs on literally every command
   the user types, so it must return in single-digit milliseconds and must never block on the
   network.** It appends an event to local SQLite (WAL mode) and exits. Guard this path obsessively:
   any latency here is felt as shell lag and will get the tool uninstalled.

2. **Store + stats (Go core).** Raw events are the source of truth: one row per command execution
   (command, start/end ts, duration, exit code, cwd, derived git repo + branch, host, shell, session
   id, pid). Aggregations — heatmaps, time-per-branch/repo, "most active hour," Wrapped — are
   **computed from raw events**, via query-time rollups or materialized summary tables. Don't store
   pre-aggregated numbers as the only copy; always keep the raw events so we can recompute.

3. **Read surfaces.** `shwind serve` exposes the stats over a local JSON API and serves the embedded
   Next.js UI; CLI subcommands (`shwind wrapped`, `shwind stats`, …) render the same aggregations in
   the terminal.

### Two modes (keep them separate)
- **Local-first (default):** everything lives in `~/.local/share/shwind/shwind.db` (respect
  `XDG_DATA_HOME`). No account, no network, works offline. This is the privacy promise.
- **Synced (opt-in):** `shwind sync` pushes events/rollups to Neon for the hosted dashboard. It is a
  background, batched, retryable operation — explicitly off the hot path.

## Decisions that still need a human call

Flag these in PRs rather than silently picking — they shape the data model:
- **Time-attribution model.** "Time spent on a branch/repo" is not obvious: sum of command durations
  (undercounts thinking time) vs. wall-clock session time bucketed by repo with an idle cutoff
  (e.g. gaps > N minutes don't count). Pick one, write it down here, and make it the definition the
  whole stats layer uses.
- **Command privacy/redaction.** Shell commands contain secrets (tokens, passwords in flags). Decide
  what's stored: full command, command + first arg only, or redacted-by-pattern — and make it
  configurable. Default to the more private option.
- **Sync granularity.** Raw events vs. aggregated rollups to Neon (volume + privacy tradeoff).

## Conventions to adopt when scaffolding

These don't exist yet — establish them with the first code and keep this list current:
- Go module path and a `Makefile` (or `Taskfile`) with `build` / `test` / `lint` / `run` targets.
  Document the real invocations here once they exist, including how to run a single test
  (`go test ./internal/stats -run TestName`).
- `golangci-lint` for Go; `eslint` + `tsc --noEmit` for the web app.
- Monorepo layout: Go in `cmd/` + `internal/` at the root, web app in `web/`, shell scripts in
  `shell/`, SQL migrations in `migrations/` (embedded, applied on startup).
- Keep the ingest command and the stats/serve commands in separate packages — the hot path should
  not import heavy dependencies (web, charts, Postgres drivers).
