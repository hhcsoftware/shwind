<!-- markdownlint-configure-file {
  "MD013": {
    "code_blocks": false,
    "tables": false
  },
  "MD033": false,
  "MD041": false
} -->

<div align="center">

# .shwind

[![Release][release-badge]][releases]
[![License][license-badge]][license]

`.shwind` (**sh**ell re**wind**) is a **companion for your command line**.

It reads the logs your shell and tools already leave behind, your shell history, your
git logs, and more, then turns them into an on-demand dashboard of your terminal life,
topped with a year(ish)-in-review reel. Spotify Wrapped, but for your CLI.<br />
Nothing to set up before your first look. Works on all major shells.

[Getting started](#getting-started) •
[Installation](#installation) •
[Configuration](#configuration) •
[What it reads](#what-it-reads)

</div>

## Getting started

<!-- ![Demo][demo] -->

```sh
shwind                # open the dashboard in your browser (built from whatever logs it finds)
shw                   # same thing, fewer keystrokes

shw generate          # (re)build your stats and Wrapped from the logs available right now
shw wrapped           # your CLI Wrapped, the year(ish)-in-review reel
shw stats             # on-demand breakdowns: busiest repos, branches, and commands
shw top               # your most-run commands, ranked
shw heatmap           # when you're at the keyboard, by hour and weekday
shw sources           # show which logs shwind found and is reading

shw init zsh          # optional: turn on the collector for richer data going forward
```

`.shwind` works the moment it's installed because it reads logs you already have. Want sharper
numbers (command durations, exit codes, which repo a command ran in)? Turn on the optional
collector and it enriches everything from that point on.

## Installation

`.shwind` installs in 2 easy steps (3 if you want the good stuff):

1. **Install the binary**

   The command is `shwind`, or `shw` if you prefer fewer keystrokes. It runs on most major
   platforms; if yours isn't listed, please [open an issue][issues].

   <details>
   <summary>Linux / macOS / WSL</summary>

   > The quickest way is the install script:
   >
   > ```sh
   > curl -sSfL https://raw.githubusercontent.com/hhcsoftware/shwind/main/install.sh | sh
   > ```
   >
   > Or, if you have a Go toolchain (1.22+):
   >
   > ```sh
   > go install github.com/hhcsoftware/shwind/cmd/shwind@latest
   > ```
   >
   > Or grab a prebuilt binary from the [releases page][releases] and drop it on your `PATH`.

   </details>

   <details>
   <summary>Windows</summary>

   > `.shwind` works with PowerShell, as well as shells running in Git Bash and MSYS2.
   >
   > With a Go toolchain (1.22+):
   >
   > ```sh
   > go install github.com/hhcsoftware/shwind/cmd/shwind@latest
   > ```
   >
   > Or download a prebuilt binary from the [releases page][releases] and put it on your `PATH`.

   </details>

   <details>
   <summary>From source</summary>

   > Needs a Go toolchain (1.22+). The web dashboard is embedded into the binary at build time,
   > so no Node is required to run it.
   >
   > ```sh
   > git clone https://github.com/hhcsoftware/shwind
   > cd shwind
   > make build        # writes ./bin/shwind
   > ```

   </details>

   > **Note:**
   > More package managers (Homebrew, Nix, the usual suspects) are on the list. PRs welcome.

2. **Run it**

   ```sh
   shwind
   ```

   This opens the dashboard at `http://localhost:7373`, builds your Wrapped from whatever logs
   are available, and gets out of your way. No account, nothing to configure. The terminal
   subcommands above show the same numbers if you'd rather not leave the shell.

3. **Turn on the collector** <sup>(optional)</sup>

   Existing logs are enough to get started, but they're patchy: shell history rarely records how
   long a command took, what it exited with, or which directory it ran in. The collector fills
   those gaps for everything you do from here on.

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
   > The collector is deliberately boring. It appends a row to a local database and gets out of
   > the way in a millisecond or two. It never phones home and never blocks your prompt.

## Configuration

### Flags

When calling `shwind init`, the following flags are available:

- `--cmd`
  - Changes the name of the command the collector wires up.
  - `--cmd shw` would let you run `shw wrapped`, `shw stats`, and friends.
- `--hook <HOOK>`
  - Changes when the collector records a command:

    | Hook                | Description                                                |
    | ------------------- | --------------------------------------------------------- |
    | `none`              | Never (you can still record manually)                     |
    | `command` (default) | Around every command, capturing real duration and exit code |

- `--no-hook`
  - Prints the init script without installing the collector, in case you want to wire it up
    yourself.

### Environment variables

Set these **before** `shwind init` is called.

- `SHWIND_DATA_DIR`
  - Where the local database and generated snapshots live. `.shwind` never writes anywhere else
    by default.
  - Defaults follow each platform's conventions:

    | OS          | Path                                     | Example                                            |
    | ----------- | ---------------------------------------- | -------------------------------------------------- |
    | Linux / BSD | `$XDG_DATA_HOME` or `$HOME/.local/share` | `/home/alice/.local/share/shwind`                  |
    | macOS       | `$HOME/Library/Application Support`       | `/Users/Alice/Library/Application Support/shwind`  |
    | Windows     | `%LOCALAPPDATA%`                          | `C:\Users\Alice\AppData\Local\shwind`              |

- `SHWIND_REDACT`
  - Controls how much of each command is stored. Your shell history is sensitive, and `.shwind`
    treats it that way.

    | Value             | Stored                                                   |
    | ----------------- | -------------------------------------------------------- |
    | `args` (default)  | Command and arguments, with secret-looking tokens scrubbed |
    | `command`         | Just the program name (`git`, `npm`, and so on), no args |
    | `none`            | Nothing but timing, exit code, repo, and branch          |

- `SHWIND_IGNORE`
  - Commands or [globs][glob] to skip entirely, separated by OS path characters (`:` on
    Linux/macOS, `;` on Windows). Example: `clear:ls:* --password*`.
- `SHWIND_SOURCES`
  - Extra history or log files to read, beyond the ones `.shwind` finds automatically. Same
    separators as above.
- `SHWIND_PORT`
  - Port for the dashboard. Defaults to `7373`.

## What it reads

`.shwind` builds everything on demand from logs that already exist on your machine:

- **Shell history**, for zsh, bash, and fish, including timestamps where your shell records them.
- **Git logs**, across the repositories you've worked in, for commit cadence, branches, and
  busiest projects.
- **The optional collector**, once enabled, for the things history leaves out: command durations,
  exit codes, and the directory each command ran in.

Run `shw sources` to see exactly what was found. Every number is recomputed from these sources, so
you can change the rules, or turn the collector on later, and rebuild your history without losing
anything. Nothing leaves your machine.

## Privacy

Everything stays in a local database on your computer. There is no account, no telemetry, and no
network call involved in reading your logs or building your Wrapped. If you ever want it gone:

```sh
shw nuke        # delete the database and forget everything
```

No hard feelings.

[release-badge]: https://img.shields.io/github/v/release/hhcsoftware/shwind?style=flat-square&logo=github
[releases]: https://github.com/hhcsoftware/shwind/releases
[license-badge]: https://img.shields.io/github/license/hhcsoftware/shwind?style=flat-square
[license]: LICENSE
[issues]: https://github.com/hhcsoftware/shwind/issues/new
[glob]: https://man7.org/linux/man-pages/man7/glob.7.html
[demo]: contrib/demo.gif
