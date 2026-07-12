---
title: "Neovim Jupyter"
date: 2026-07-11
draft: false
---

`nvim-jupyter` is a Neovim plugin that turns a `.ipynb` file into a live notebook you can run without leaving the editor. Open a notebook, execute cells in place, and see output render inline right below the cell: text, errors, and images. Save with `:w` and it serializes back to real `.ipynb` on disk. No browser, no Jupyter web UI, no context switch.

Source: [github.com/bobcowher/neovim-jupyter](https://github.com/bobcowher/neovim-jupyter)

I built this for the same reason as most of my tools: I live in Neovim, and bouncing out to a browser tab to poke at a notebook was friction I didn't want. I wanted the cell-based, run-and-see-it workflow that makes notebooks useful, but with my own keymaps, my own colorscheme, and my own editor underneath it.

It's a Rust binary and a Lua plugin.

## Requirements

- **A terminal that speaks the Kitty graphics protocol**, for inline images. I build and test against [Ghostty](https://ghostty.org); [Kitty](https://sw.kovidgoyal.net/kitty/) works too. Everything else (text output, errors, execution) works anywhere. You just won't see plots.
- **Linux.**
- **[Neovim](https://neovim.io) 0.9+.**
- **Jupyter and a kernel.** You need `jupyter` and at least one kernel (`ipykernel`) installed. The plugin discovers standard Jupyter kernelspecs *and* your conda environments automatically. If a conda env is missing `ipykernel`, it offers to install it for you when you pick it.
- **A Rust toolchain**, to build the daemon. The easiest path is [rustup](https://rustup.rs), which installs `cargo`:

  ```bash
  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
  ```

## Installation

The Rust daemon has to be compiled before first use. With Lazy.nvim the `build` hook handles that automatically, on install and on every update, so there's nothing to run by hand.

### Lazy.nvim

```lua
{
  "bobcowher/neovim-jupyter",
  build = "bash build.sh",
  config = function()
    require("nvim-jupyter").setup()
  end,
}
```

`build.sh` runs `cargo build --release` and copies the binary to `bin/nvim-jupyter`, where the plugin looks for it. It won't search `$PATH`, which keeps behavior predictable.

### Manual steps (if it doesn't work automatically)

If the build hook didn't fire, or you pulled changes and want to rebuild, run `:JupyterBuild` from inside Neovim. Or do it from a shell:

```bash
git clone https://github.com/bobcowher/neovim-jupyter.git
cd neovim-jupyter
bash build.sh
```

## Usage

Open any `.ipynb` file and the plugin activates automatically. The notebook loads as a virtual cell buffer: each cell separated by a labelled divider, code cells editable as plain Python, markdown cells rendered inline. Save with `:w` and your changes serialize straight back to `.ipynb`.

### Default keymaps

| Key | Action |
|-----|--------|
| `<CR>` or `<C-CR>` | Execute cell in place |
| `<S-CR>` | Execute cell and advance to the next |
| `<M-CR>` | Execute cell and insert a new one below |
| `]c` | Next cell |
| `[c` | Previous cell |
| `<Space>b` | Add cell below |
| `<Space>a` | Add cell above |
| `<Space>d` | Delete current cell |
| `<Space>m` | Toggle cell type (code / markdown) |

*(Ghostty captures `ctrl+enter` by default to toggle its header. Use `shift+enter`, or unbind `ctrl+enter` in Ghostty.)*

### Commands

**Execution:**

- `:JupyterExecute` runs the current cell in place.
- `:JupyterExecuteAndAdvance` runs, then moves to the next cell.
- `:JupyterExecuteAndInsert` runs, then opens a fresh cell below.
- `:JupyterExecuteAll` runs every cell top to bottom.
- `:JupyterExecuteAbove` / `:JupyterExecuteBelow` run everything above, or the current cell and everything below.

**Navigation & editing:**

- `:JupyterNextCell` / `:JupyterPrevCell` move between cells.
- `:JupyterAddCellBelow` / `:JupyterAddCellAbove` add a code cell.
- `:JupyterDeleteCell` deletes the current cell.
- `:JupyterChangeCellType` toggles between code and markdown.
- `:JupyterClearOutput` / `:JupyterClearAllOutputs` clear output for the current cell, or the whole buffer.

**Kernel management:**

- `:JupyterKernel` opens a picker to choose a kernel (Jupyter kernelspecs and conda envs).
- `:JupyterRestartKernel` restarts the current kernel.
- `:JupyterInterrupt` interrupts a running cell (SIGINT).
- `:JupyterKernelStatus` prints the current kernel status.

**Other:**

- `:JupyterInspect` shows hover documentation for the symbol under the cursor.
- `:JupyterShowOutput` dumps the current cell's full output into a scratch split.
- `:JupyterBuild` compiles the Rust daemon.

## Configuration

Everything is overridable through `setup()`:

```lua
require("nvim-jupyter").setup({
  keymaps = true, -- set false to disable the configurable keymaps below
  keymap = {
    execute          = "<C-CR>",
    execute_advance  = "<S-CR>",
    execute_insert   = "<M-CR>",
    next_cell        = "]c",
    prev_cell        = "[c",
    delete_cell      = "<Space>d",
    toggle_cell_type = "<Space>m",
    add_cell_below   = "<Space>b",
    add_cell_above   = "<Space>a",
  },
  max_output_lines = 50,   -- output longer than this is truncated inline
  auto_save        = true, -- write the notebook back to disk when the cursor goes idle
})
```

## Notes

- **Save is real.** `:w` writes valid `.ipynb`, including cells, outputs, and execution counts, so notebooks stay interoperable with Jupyter proper and with anything else that reads the format.
- **Execution counts come from the kernel**, not a local guess, so they match what a normal Jupyter session would show.
- **Kitty graphics protocol only** for images. Built and tested against Ghostty on Linux. Text, errors, and completion work regardless of terminal.
- If the daemon isn't built yet, Neovim notifies you to run `:JupyterBuild`.

## How it works

If you just want to use it, stop here. This is for the curious.

The notebook file is the easy part. The hard part is talking to a kernel. Kernels communicate over [ZeroMQ](https://zeromq.org) across five sockets, with HMAC-signed multipart messages and a handshake you have to get exactly right, or the kernel silently drops everything you send. That's fiddly, timing-sensitive work, and it doesn't belong in Lua. So the plugin splits in two:

1. **A Rust daemon** (`nvim-jupyter`) that owns the entire ZeroMQ Jupyter protocol: launching kernels, signing and sending messages, and draining the shell/iopub/control sockets.
2. **A Lua plugin** that owns everything editor-shaped: the cell buffer, output rendering via extmarks, kernel selection, and keymaps.

The two talk over newline-delimited JSON on stdin/stdout. Lua sends commands down:

```json
{"cmd":"start_kernel","kernel_id":"k1","kernel_name":"python3","cwd":"/home/me/nb"}
{"cmd":"execute","kernel_id":"k1","msg_id":"m1","code":"print('hi')"}
{"cmd":"interrupt_kernel","kernel_id":"k1"}
{"cmd":"restart_kernel","kernel_id":"k1"}
{"cmd":"quit"}
```

The daemon streams events back up as they arrive from the kernel:

```json
{"event":"kernel_ready","kernel_id":"k1"}
{"event":"stream","msg_id":"m1","name":"stdout","text":"hi\n"}
{"event":"execute_result","msg_id":"m1","execution_count":3,"text":"42","image_png":"iVBORw0K..."}
{"event":"execute_done","msg_id":"m1","status":"ok","execution_count":3}
```

Every request carries a `msg_id`, and the Lua side keys a small registry off it so each reply routes back to the cell that asked for it, then cleans itself up when the request finishes. One kernel runs as one async task inside the daemon. That's what lets a long-running cell keep taking control commands: interrupts are serviced concurrently with the cell's output stream, so `:JupyterInterrupt` actually stops a runaway loop instead of waiting for it to end on its own.

A few things I learned the hard way building this:

- **The signing key is deceptive.** The connection file's `key` looks like hex, but Jupyter uses its raw UTF-8 bytes as the HMAC key, *not* the hex-decoded value. Decode it and every message you send fails verification and gets dropped with no error. It cost me an afternoon.
- **Finishing a cell is a two-part signal.** A cell is done only once both the shell `execute_reply` and the iopub `status: idle` have arrived. The idle status is published after all output, so waiting for it guarantees the last line of output makes it to the screen instead of racing the "done" signal.
- **Images bypass the screen renderer.** Inline plots are written to the terminal as Kitty graphics escape sequences straight to the pts device, so they don't get mangled by Neovim's own redraws. They're re-anchored to the cell's last line on every edit, so they stay pinned to the bottom of their cell as you type above them.
