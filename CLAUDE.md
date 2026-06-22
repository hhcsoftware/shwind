# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status: greenfield

The repo is empty (only this file and `README.md`). Nothing below describes existing code; it
records the decisions for the build. When you scaffold something, follow these conventions, and
when a decision here proves wrong in practice, change it here in the same PR so this file stays true.

## Naming and house style (do not get these wrong)

- The project is **`.shwind`** (written lowercase, with the leading dot). The command is `shwind`,
  with `shw` as a shorter alias.
- Repository and Go module: `github.com/hhcsoftware/shwind` (no dot). The leading-dot `.shwind` is
  the brand only; the repo and module path drop the dot because a Go module path element may not
  begin with one (see "Repo name versus the `.shwind` brand" below).
- **Never call it a "time tracker."** It is a **CLI companion**. It does not exist to bill your
  hours; it makes sense of the trail your terminal already leaves.
- **No em-dashes** anywhere in code comments, docs, copy, or commit messages. Use commas,
  conjunctions, parentheses, colons, or a plain hyphen instead. This is a hard style rule.
- Tone: minimal, utilitarian, privacy-first, with a light quirky streak in user-facing copy. Do
  not over-brand: no taglines, no logo art, no marketing voice in code or docs.

## What we're building

`.shwind` is a command-line companion. It reads the logs your shell and tools already keep (shell
history, git logs, and more) and turns them into an **on-demand** dashboard of your terminal life:
busiest repos and branches, most-run commands, an activity heatmap, and a Spotify-Wrapped-style
year(ish)-in-review. The Wrapped is the marquee view, but it is one view among several, not the
whole product.

Two facts shape the whole design:

1. **It works with zero setup**, because it builds from logs that already exist. An optional
   collector (a shell hook) enriches future data with the things history omits (command durations,
   exit codes, and the directory a command ran in).
2. **Generation is on demand.** There is no always-on daemon doing the analysis. Stats and the
   Wrapped are (re)built when the dashboard is opened or when the user runs `shw generate`, from
   whatever sources are available at that moment.

## Tech stack (decided)

- **Core CLI plus local server:** Go, with [Cobra](https://github.com/spf13/cobra) for the command
  tree. The binary parses logs, generates snapshots, and serves the dashboard.
- **Local store:** SQLite via **`modernc.org/sqlite`** (pure-Go, **no CGO**). Deliberate, so the
  binary cross-compiles and ships as a single static file. Do not switch to a CGO driver
  (`mattn/go-sqlite3`) without a strong reason. The store holds two things: collector events, and
  cached generation snapshots. It is not the primary source of truth (existing logs are).
- **Web dashboard (primary surface):** Next.js (App Router) plus TypeScript plus Tailwind. This is
  where the Wrapped renders richly. Build it static-exportable and embed it into the Go binary via
  `embed.FS`, so `shwind` serves the dashboard with no Node at runtime. Charts: prefer hand-rolled
  SVG or a light lib (visx/Recharts) over a heavy charting framework.
- **Optional cloud:** NeonDB (serverless Postgres) for a hosted or shareable Wrapped, reached only
  through an explicit `shw sync`. Never part of generation or any interactive path.
- **Shell support for the collector:** zsh, bash, and fish.

## Architecture, the big picture

Three stages, loosely coupled. Understanding the boundary between "read existing logs" and "the
optional collector" matters more than any single file.

1. **Sources, into a common event model.** Two kinds of input, normalized to the same shape:
   - *Existing logs (always available, zero setup):* per-shell history parsers (zsh extended
     history, bash, fish YAML), and git (commits, branches, repo activity across repos the user has
     worked in). Important limitation to design around: shell history usually lacks duration, exit
     code, and the working directory, so from existing logs alone the honest Wrapped is mostly
     "what you ran and when" plus git's "what you shipped where." Do not invent precision the
     sources do not have.
   - *The optional collector (opt-in, richer):* a sourced shell script (`shell/shwind.zsh`, etc.)
     fires on each command via the shell's hooks and calls the binary (`shwind hook ...`) to append
     an enriched event (duration, exit code, cwd, derived git repo and branch). This path runs on
     every command, so it must return in single-digit milliseconds and never block on the network.
     It is optional, but when present it is the same hot-path discipline as any shell integration.

2. **Generation (on demand).** `shw generate`, or opening the dashboard, reads whatever sources
   exist right now, computes the aggregates (heatmap, per-repo and per-branch breakdowns, top
   commands, the Wrapped narrative), and writes a snapshot to the local store. Always recompute
   from raw sources and the collector event log; never treat a snapshot as the only copy. Generation
   should stay fast over large histories, so favor incremental or cached rollups over rescanning
   everything each time.

3. **Surfaces.** The web dashboard (served and embedded by the Go binary) is the primary surface
   and where Wrapped lives. Terminal subcommands (`shw stats`, `shw top`, `shw heatmap`,
   `shw wrapped`) render the same aggregations for people who would rather not leave the shell.

## Decisions that still need a human call

Flag these in PRs rather than silently picking; they shape the data model and the product:

- **What the zero-setup Wrapped can honestly claim.** Given history's missing fields, decide which
  metrics are "existing-logs only" versus "needs the collector," and label them so the UI never
  implies data it does not have.
- **How repositories are discovered** for the git source (scan configured roots, infer from the
  collector's recorded cwd, walk a path list). History alone often does not tell you where commands
  ran.
- **Command privacy and redaction.** Commands contain secrets. Default to the more private option;
  make it configurable (`SHWIND_REDACT`).
- **Activity attribution.** If any view shows "time on a repo or branch," define it explicitly
  (for example, wall-clock between consecutive events in the same context with an idle cutoff) and
  use one definition everywhere. Frame it as activity, not billed time.
- **Sync granularity** for the optional Neon path (raw events versus rollups).

## Repo name versus the `.shwind` brand (resolved)

A Go module path element **may not begin or end with a dot**, so a repo literally named `.shwind`
cannot back a valid module path and would break `go install`. Resolution: the **brand stays
`.shwind`** (title, dashboard, copy), but the **repo and module are `github.com/hhcsoftware/shwind`**
(no dot). `go install github.com/hhcsoftware/shwind/cmd/shwind@latest` works, and standard tooling
(pkg.go.dev, Go Report Card) is happy. Do not reintroduce the dot into the module path.

## Conventions to adopt when scaffolding

These do not exist yet; establish them with the first code and keep this list current:

- A `Makefile` (or `Taskfile`) with `build` / `test` / `lint` / `run` targets. `make build` writes
  `./bin/shwind` and embeds the built web app. Document the real invocations here once they exist,
  including how to run a single test (`go test ./internal/generate -run TestName`).
- `golangci-lint` for Go; `eslint` plus `tsc --noEmit` for the web app.
- Monorepo layout: Go in `cmd/` and `internal/` at the root, the web app in `web/`, collector shell
  scripts in `shell/`, SQL migrations in `migrations/` (embedded, applied on startup).
- Keep the collector's hook command in its own lean package. It must not import the web, charting,
  or Postgres code, so the hot path stays light.
