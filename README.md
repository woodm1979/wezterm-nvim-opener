# wezterm-nvim-opener

Double-click a text or code file in Finder and have it open in Neovim inside a new WezTerm window.

## Requirements

- macOS
- [WezTerm](https://wezfurlong.org/wezterm/)
- [Neovim](https://neovim.io/)
- [Homebrew](https://brew.sh/) — used to install `duti` during setup

## Installation

**1. Verify the hardcoded paths in `wezterm-nvim-open`:**

```bash
# wezterm — usually a symlink into the .app bundle
ls /usr/local/bin/wezterm

# nvim — Homebrew on Apple Silicon
ls /opt/homebrew/bin/nvim
# Intel Mac: /usr/local/bin/nvim
```

If your paths differ, edit `wezterm-nvim-open` before continuing.

**2. Put the scripts somewhere on your PATH**, e.g.:

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

**3. Run the setup script once:**

```bash
setup-nvim-opener
```

Re-run it any time you move the scripts or want to rebuild the app.

## What `setup-nvim-opener` does

1. Writes an AppleScript source file to `/tmp` and compiles it into `~/Applications/NvimOpen.app` via `osacompile`
2. Injects `CFBundleIdentifier = com.woodnt.nvimopen` into the app's `Info.plist` so `duti` can reference it
3. Registers `NvimOpen.app` with Launch Services (`lsregister`) so macOS knows the app exists
4. Installs `duti` via Homebrew if not already present
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

`wezterm start -- nvim <file>` has a bug in WezTerm ≤ 20240203: when a WezTerm instance is already running, the CLI hands off to the existing GUI process via a Unix socket, but the GUI ignores the `PROG` argument and opens the default shell instead.

`wezterm cli spawn --new-window -- nvim <file>` uses a different IPC code path that correctly passes the command to the running GUI, but requires WezTerm to already be running.

`wezterm-nvim-open` uses `wezterm cli list` to detect whether WezTerm is running, then picks the appropriate command:

- **Running** → `wezterm cli spawn --new-window` (avoids the `start` bug)
- **Not running** → `wezterm start` (no existing instance to trigger the bug)

### Execution path on double-click

```
Finder double-click
  → Apple Event to NvimOpen.app
    → AppleScript "on open" handler
      → do shell script "wezterm-nvim-open <path> &"
        → wezterm cli list (check if WezTerm is running)
          → [running]     wezterm cli spawn --new-window -- nvim <path>
          → [not running] wezterm start -- nvim <path>
            → new WezTerm window running nvim
```

## Shell notes

Both scripts use `#!/bin/bash`. This is intentional:

- `/bin/bash` is available on every macOS system (bash 3.2, the GPL-2 version Apple ships)
- AppleScript's `do shell script` runs commands via `/bin/sh`, not your login shell, so your interactive shell preference (zsh, fish, etc.) has no effect on how the scripts are invoked
- If you use **zsh** as your login shell and Homebrew is only initialized in your `.zshrc` (not in `/etc/paths` or `/etc/profile`), running `setup-nvim-opener` from a plain bash context might not find `brew`. Fix: either ensure `/opt/homebrew/bin` is in your system `PATH`, or run the script explicitly as `zsh setup-nvim-opener`
- If you want to use **fish** or another shell for the runtime script, change the shebang in `wezterm-nvim-open` — just make sure the `exec` equivalent works in that shell

## Caveats

**Single file per window.** Each double-clicked file gets its own WezTerm window. Selecting multiple files in Finder and opening them will produce multiple windows.

**UTI coverage is broad but not exhaustive.** `public.source-code` covers most compiled languages and interpreted scripts, but some file types have more specific UTIs that may not be caught. If a file type isn't opening in nvim, check its UTI with `mdls -name kMDItemContentType <file>` and add a `duti` line to `setup-nvim-opener`.
