---
title: "Neovim Graphics Viewer"
date: 2026-05-28
draft: true
---

`nvim-gfx` is a Neovim plugin that displays images and video directly inside your terminal. Open a PNG, JPEG, WebP, or a video file in Neovim and it renders right where the buffer is — same position, same size. Zoom, pan, scrub video, close with standard Neovim keymaps. No context switch.

Source: [github.com/bobcowher/neovim-graphics-viewer](https://github.com/bobcowher/neovim-graphics-viewer)

I built this because I kept needing to peek at images while working — training curves, screenshots, diagrams — and alt-tabbing out of the terminal was friction I didn't want. Later I wanted the same thing for short video clips.

It's a Rust binary and a Lua plugin. Built with plenty of Claude help.

## How it works

Two components:

1. **A Rust binary** (`nvim-gfx`) that decodes the image or video and writes Kitty graphics protocol escape sequences straight to the terminal device.
2. **A Lua plugin** that figures out where your Neovim buffer sits on screen (row, column, width, height in cells), launches the binary, and drives it over stdin/stdout JSON.

The binary reads newline-delimited commands from stdin:

```json
{"cmd":"show","path":"/home/me/image.png","row":0,"col":0,"width":120,"height":40}
{"cmd":"zoom","factor":1.25}
{"cmd":"pan","dx":-1,"dy":0}
{"cmd":"play_pause"}
{"cmd":"seek","delta":10}
{"cmd":"quit"}
```

Neovim sends `show` when you open a file, re-sends it when the window geometry changes (resize, scroll, layout), and sends `quit` when the viewer closes. The binary never dismisses itself — Neovim stays in control.

One wrinkle worth mentioning: a binary spawned by `jobstart` has no controlling terminal, so it can't just open `/dev/tty` to talk to the screen. The Lua side resolves the real pts device (`/proc/self/fd/1`) and hands it to the binary as the `NVIM_GFX_TTY` environment variable; the binary opens that and writes its escape sequences there. Video frames are decoded with `ffmpeg` and pushed on a frame-timed loop.

## Requirements

- **A terminal that speaks the Kitty graphics protocol.** I build and test against [Ghostty](https://ghostty.org); [Kitty](https://sw.kovidgoyal.net/kitty/) should work too. Terminals without it — GNOME Terminal, Alacritty — render nothing. That's the cost of dropping the overlay window.
- **Linux.**
- **[Neovim](https://neovim.io) 0.9+.**
- **A Rust toolchain**, to build the binary. The easiest path is [rustup](https://rustup.rs), which installs `cargo`:

  ```bash
  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
  ```

- **FFmpeg development libraries**, for video. The binary links the system FFmpeg libraries at compile time through the [`ffmpeg-next`](https://crates.io/crates/ffmpeg-next) crate — so you need the `-dev` packages plus `pkg-config` and `clang`, not just the `ffmpeg` command. On Debian/Ubuntu:

  ```bash
  sudo apt install pkg-config clang \
    libavformat-dev libavfilter-dev libavdevice-dev \
    libavutil-dev libavcodec-dev libswscale-dev libswresample-dev
  ```

  For other platforms, see the [FFmpeg download page](https://ffmpeg.org/download.html).

## Installation

### Lazy.nvim

```lua
{
  "bobcowher/neovim-graphics-viewer",
  tag = "v1.0.3",
  config = function()
    require("nvim-gfx").setup()
  end,
}
```

Then build the binary once:

```
:NvimGfxBuild
```

`:NvimGfxBuild` runs `cargo build --release` in the plugin directory and copies the binary to `bin/nvim-gfx`. The plugin looks for it there — it won't search `$PATH`, which keeps behavior predictable.

### Manual

```bash
git clone https://github.com/bobcowher/neovim-graphics-viewer.git
cd neovim-graphics-viewer
cargo build --release
cp target/release/nvim-gfx bin/nvim-gfx
```

## Usage

### Auto-preview

Opening any `.png`, `.jpg`, `.jpeg`, `.webp`, `.mp4`, `.mkv`, `.webm`, `.avi`, `.mov`, or `.m4v` file in Neovim triggers the viewer automatically. The media fills the current buffer area.

### Explicit command

```
:ViewImage /path/to/image.png
```

Takes any readable file path. Shell expansion works (it calls `expand()` internally).

### Image keymaps

Set buffer-local while the viewer is active:

| Key | Action |
|-----|--------|
| `q` | Close viewer |
| `+`, `=` | Zoom in (1.25×) |
| `-` | Zoom out (0.8×) |
| `r` | Reset — fit to window |
| `h`, `<Left>` | Pan left |
| `l`, `<Right>` | Pan right |
| `k`, `<Up>` | Pan up |
| `j`, `<Down>` | Pan down |

Zoom is clamped between 5% and 3200%. Pan is in terminal-cell increments.

### Video keymaps

| Key | Action |
|-----|--------|
| `q` | Close viewer |
| `p`, `<Space>` | Play / pause |
| `l`, `<Right>` | Seek +1s |
| `h`, `<Left>` | Seek −1s |
| `k`, `<Up>` | Seek +10s |
| `j`, `<Down>` | Seek −10s |
| `r` | Rewind to start |
| `+`, `=` | Zoom in |
| `-` | Zoom out |

## Behavior

The viewer is **dedicated**: the file you open owns its window.

- **Focusing another window leaves it up.** Tab over to your file tree and the image stays put — it's only a focus change, not a navigation.
- **Opening another file closes it.** As soon as you open a different real file, the viewer tears down and clears the image. Press `q` to close it explicitly; if a file-tree sidebar is open, `q` bounces your cursor back to it.
- **Video pauses when you look away.** A playing video repaints every frame, and each frame nudges the terminal cursor — with focus elsewhere that flickers against Neovim's own cursor. So the video auto-pauses when its window loses focus and resumes when you come back. Images draw once, so they're unaffected.

## Notes

- **Image formats:** PNG, JPEG, WebP. **Video:** MP4, MKV, WebM, AVI, MOV, M4V. No animated GIF yet.
- **Kitty graphics protocol only.** Built and tested against Ghostty on Linux. Terminals without the protocol show nothing.
- If the binary isn't built yet, Neovim notifies you with `nvim-gfx: binary not found, run :NvimGfxBuild`.
