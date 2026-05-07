# wezterm-nvim-opener

Double-click a text or code file in Finder and have it open in Neovim inside a new WezTerm window.

## Requirements

- macOS
- [WezTerm](https://wezfurlong.org/wezterm/)
- [Neovim](https://neovim.io/)
- [Homebrew](https://brew.sh/) — used to install `duti` and `jq` during setup

## Installation

**1. Put the scripts somewhere on your PATH**, e.g.:

```bash
cp wezterm-nvim-open setup-nvim-opener ~/bin/
chmod +x ~/bin/wezterm-nvim-open ~/bin/setup-nvim-opener
```

Or clone and symlink:

```bash
git clone https://github.com/woodm1979/wezterm-nvim-opener
ln -s "$PWD/wezterm-nvim-opener/wezterm-nvim-open" ~/bin/
ln -s "$PWD/wezterm-nvim-opener/setup-nvim-opener" ~/bin/
```

`wezterm-nvim-open` finds `wezterm` and `nvim` automatically by searching `/usr/local/bin` and `/opt/homebrew/bin` (covering both Intel and Apple Silicon Macs). If your binaries are elsewhere, prepend their directories to PATH before running.

**2. Run the setup script once:**

```bash
setup-nvim-opener
```

Re-run it any time you move the scripts or want to rebuild the app.

## What `setup-nvim-opener` does

1. Installs `jq` and `duti` via Homebrew if not already present
2. Writes an AppleScript source file to `/tmp` and compiles it into `~/Applications/NvimOpen.app` via `osacompile`
3. Injects `CFBundleIdentifier = com.woodnt.nvimopen` into the app's `Info.plist` so `duti` can reference it
4. Registers `NvimOpen.app` with Launch Services (`lsregister`) so macOS knows the app exists
5. Calls `duti` to register `NvimOpen.app` as the default handler for these UTIs:

| UTI | Covers |
|-----|--------|
| `public.plain-text` | `.txt` and similar |
| `public.source-code` | most code files |
| `net.daringfireball.markdown` | `.md` |
| `public.python-script` | `.py` |
| `public.shell-script` | `.sh` |
| `public.json` | `.json` |

## How it works

### Why an `.app` bundle?

macOS file associations require a proper `.app` bundle as the handler — a bare script or binary won't work. When you double-click a file in Finder, macOS sends an **Apple Event** (`kAEOpenDocuments`) to the registered handler app. The handler must implement an `on open` handler in AppleScript (or the Cocoa equivalent) to receive the file list.

`osacompile` builds a minimal "droplet" app from an AppleScript source file, which is the lightest-weight way to get a working `.app` without Xcode or Automator.

### `wezterm cli spawn` vs `wezterm start`

`wezterm start -- nvim <file>` ignores the `PROG` argument on macOS — WezTerm opens with a default shell regardless of what program is passed. This affects both the "already running" and "not running" cases.

`wezterm cli spawn --new-window -- nvim <file>` correctly passes the command to the running WezTerm GUI via IPC, but requires WezTerm to already be running and requires `$WEZTERM_PANE` or an explicit `--pane-id` when called from outside WezTerm.

`wezterm-nvim-open` uses `wezterm cli list` to detect whether WezTerm is running, then picks the appropriate path:

- **Running** → `wezterm cli spawn --new-window -- nvim <file>` (WezTerm is already up, IPC works)
- **Not running** → start WezTerm in the background, poll until its IPC socket is ready, then `cli spawn` using the initial pane's ID, and kill the initial shell pane so only the nvim tab remains

### Execution path on double-click

```
Finder double-click
  → Apple Event to NvimOpen.app
    → AppleScript "on open" handler
      → do shell script "wezterm-nvim-open <path> &"
        → wezterm cli list (check if WezTerm is running)
          → [running]     wezterm cli spawn --new-window -- nvim <path>
          → [not running] wezterm start & (background, opens default shell)
                          poll wezterm cli list until socket ready (15s timeout)
                          wezterm cli spawn --pane-id <initial-pane> -- nvim <path>
                          wezterm cli kill-pane --pane-id <initial-pane>
```

## Shell notes

Both scripts use `#!/bin/bash`. This is intentional:

- `/bin/bash` is available on every macOS system (bash 3.2, the GPL-2 version Apple ships)
- AppleScript's `do shell script` runs commands via `/bin/sh`, not your login shell, so your interactive shell preference (zsh, fish, etc.) has no effect on how the scripts are invoked
- `wezterm-nvim-open` explicitly prepends `/usr/local/bin` and `/opt/homebrew/bin` to `PATH` so Homebrew-installed binaries are found even in the restricted `do shell script` environment

## Caveats

**Single file per window.** Each double-clicked file gets its own WezTerm window. Selecting multiple files in Finder and opening them will produce multiple windows.

**UTI coverage is broad but not exhaustive.** `public.source-code` covers most compiled languages and interpreted scripts, but some file types have more specific UTIs that may not be caught. If a file type isn't opening in nvim, check its UTI with `mdls -name kMDItemContentType <file>` and add a `duti` line to `setup-nvim-opener`.
