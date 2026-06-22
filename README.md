<!-- markdownlint-configure-file {
  "MD013": {
    "code_blocks": false,
    "tables": false
  },
  "MD033": false,
  "MD041": false
} -->

<div align="center">

# shwind

[![Release][release-badge]][releases]
[![Go Report Card][goreport-badge]][goreport]
[![License][license-badge]][license]

shwind is a **time tracker for your terminal**.

It quietly clocks every command, process, branch, and repo — so once a year it can
hand you a *Spotify Wrapped, but for your CLI.*<br />
Local-first, no account required. shwind works on all major shells.

[Getting started](#getting-started) •
[Installation](#installation) •
[Configuration](#configuration) •
[Web dashboard](#web-dashboard)

</div>

## Getting started

<!-- ![Demo][demo] -->

```sh
shwind                # today at a glance: time tracked, top commands, current streak
shwind wrapped        # your CLI Wrapped — the headline reel (try `shwind wrapped --year`)

shwind stats          # where your time actually went
shwind stats --repo   # ranked by time spent per repo
shwind stats --branch # ranked by time spent per branch
shwind top            # your most-run commands, ranked
shwind heatmap        # when you're at the keyboard (hour × weekday)

shwind serve          # open the web dashboard → http://localhost:7373
shwind pause          # stop tracking for a bit; `shwind resume` picks it back up
```

shwind records one row per command — duration, exit code, the directory, and the git repo +
branch it ran in — then does the arithmetic so you don't have to. Read more about how time is
attributed [here](#how-time-is-counted).

## Installation

shwind installs in 3 easy steps:

1. **Install the binary**

   shwind runs on most major platforms. If yours isn't listed, please
   [open an issue][issues].

   <details>
   <summary>Linux / macOS / WSL</summary>

   > The quickest way is the install script:
   >
   > ```sh
   > curl -sSfL https://raw.githubusercontent.com/axo/shwind/main/install.sh | sh
   > ```
   >
   > Or, if you have a Go toolchain (1.22+):
   >
   > ```sh
   > go install github.com/axo/shwind/cmd/shwind@latest
   > ```

   </details>

   <details>
   <summary>Windows</summary>

   > shwind works with PowerShell, as well as shells running in Git Bash and MSYS2.
   >
   > With a Go toolchain (1.22+):
   >
   > ```sh
   > go install github.com/axo/shwind/cmd/shwind@latest
   > ```
   >
   > Make sure `%GOPATH%\bin` (usually `%USERPROFILE%\go\bin`) is on your `PATH`.

   </details>

   <details>
   <summary>From source</summary>

   > ```sh
   > git clone https://github.com/axo/shwind
   > cd shwind
   > make build        # writes ./bin/shwind
   > ```
   >
   > The web dashboard is embedded into the binary at build time — no Node required to run it.

   </details>

   > **Note:**
   > More package managers (Homebrew, Nix, the usual suspects) are on the list. PRs welcome.

2. **Hook it into your shell**

   This is the part that does the tracking. Add the line for your shell, open a new terminal,
   and shwind starts quietly taking notes.

   <details>
   <summary>Bash</summary>

   > Add this to the <ins>**end**</ins> of your config file (usually `~/.bashrc`):
   >
   > ```sh
   > eval "$(shwind init bash)"
   > ```

   </details>

   <details>
   <summary>Fish</summary>

   > Add this to the <ins>**end**</ins> of your config file (usually
   > `~/.config/fish/config.fish`):
   >
   > ```sh
   > shwind init fish | source
   > ```

   </details>

   <details>
   <summary>PowerShell</summary>

   > Add this to the <ins>**end**</ins> of your config file (find it by running
   > `echo $profile` in PowerShell):
   >
   > ```powershell
   > Invoke-Expression (& { (shwind init powershell | Out-String) })
   > ```

   </details>

   <details>
   <summary>Zsh</summary>

   > Add this to the <ins>**end**</ins> of your config file (usually `~/.zshrc`):
   >
   > ```sh
   > eval "$(shwind init zsh)"
   > ```

   </details>

   > **Note:**
   > The hook is intentionally boring — it appends a row to a local database and gets out of the
   > way in a millisecond or two. It never phones home and never blocks your prompt.

3. **Look at your data** <sup>(optional)</sup>

   ```sh
   shwind serve
   ```

   Opens the dashboard at `http://localhost:7373` — heatmaps, per-repo breakdowns, and the
   Wrapped view. The terminal subcommands above show the same numbers if you'd rather not leave.

## Configuration

### Flags

When calling `shwind init`, the following flags are available:

- `--cmd`
  - Changes the name of the `shwind` command.
  - `--cmd sh` would let you run `sh wrapped`, `sh stats`, etc.
- `--hook <HOOK>`
  - Changes when shwind records a command:

    | Hook              | Description                                              |
    | ----------------- | ------------------------------------------------------- |
    | `none`            | Never (you can still record manually)                   |
    | `command` (default) | Around every command — captures real duration & exit code |

- `--no-hook`
  - Prints the init script without installing the shell hook, in case you want to wire it up
    yourself.

### Environment variables

Set these **before** `shwind init` is called.

- `SHWIND_DATA_DIR`
  - Where the local database lives. shwind never writes anywhere else by default.
  - Defaults follow each platform's conventions:

    | OS          | Path                                     | Example                                    |
    | ----------- | ---------------------------------------- | ------------------------------------------ |
    | Linux / BSD | `$XDG_DATA_HOME` or `$HOME/.local/share` | `/home/alice/.local/share/shwind`          |
    | macOS       | `$HOME/Library/Application Support`       | `/Users/Alice/Library/Application Support/shwind` |
    | Windows     | `%LOCALAPPDATA%`                          | `C:\Users\Alice\AppData\Local\shwind`      |

- `SHWIND_REDACT`
  - Controls how much of each command is stored. Your shell history is sensitive; shwind treats
    it that way.

    | Value             | Stored                                            |
    | ----------------- | ------------------------------------------------- |
    | `args` (default)  | Command + arguments, with secret-looking tokens scrubbed |
    | `command`         | Just the program name (`git`, `npm`, …) — no args |
    | `none`            | Nothing but timing, exit code, repo, and branch   |

- `SHWIND_IGNORE`
  - Commands or [globs][glob] to skip entirely, separated by OS path characters (`:` on
    Linux/macOS, `;` on Windows). Example: `clear:ls:* --password*`.
- `SHWIND_IDLE_TIMEOUT`
  - Gaps longer than this don't count as "time spent" (you went to lunch; shwind shouldn't bill
    you for it). Defaults to `5m`. See [How time is counted](#how-time-is-counted).
- `SHWIND_PORT`
  - Port for `shwind serve`. Defaults to `7373`.

## Web dashboard

`shwind serve` starts a small local server (no Node, no cloud) that renders your stats: an
hour-by-weekday activity heatmap, time-per-repo and time-per-branch breakdowns, your most-run
commands, and the year-in-review **Wrapped** view. It reads the same local database the CLI does,
so there's nothing to sync and nothing to log into.

## How time is counted

shwind stores raw command events and computes everything from them. "Time spent on a repo or
branch" is wall-clock time between consecutive commands in the same context, with idle gaps longer
than `SHWIND_IDLE_TIMEOUT` thrown out — so a long-running build counts, but stepping away for
coffee doesn't. Every number you see is recomputed from the raw events, which means you can change
the rules and re-run history without losing anything.

## Privacy

Everything stays in a local database on your machine. There is no account, no telemetry, and no
network call on the hot path. If you ever want it gone:

```sh
shwind nuke        # delete the database and forget everything
```

No hard feelings.

[release-badge]: https://img.shields.io/github/v/release/axo/shwind?style=flat-square&logo=github
[releases]: https://github.com/axo/shwind/releases
[goreport-badge]: https://goreportcard.com/badge/github.com/axo/shwind?style=flat-square
[goreport]: https://goreportcard.com/report/github.com/axo/shwind
[license-badge]: https://img.shields.io/github/license/axo/shwind?style=flat-square
[license]: LICENSE
[issues]: https://github.com/axo/shwind/issues/new
[glob]: https://man7.org/linux/man-pages/man7/glob.7.html
[demo]: contrib/demo.gif
