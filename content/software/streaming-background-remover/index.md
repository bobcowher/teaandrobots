---
title: "Streaming Background Remover for Linux"
date: 2026-02-22
draft: false
---

Streaming Background Remover (`sbr`) is a Linux CLI tool that removes your webcam background in real time and exposes the result as a virtual V4L2 camera device. The virtual camera appears in OBS Studio (and any other app that reads V4L2 devices) as a standard webcam source. The default output is a chroma green background, which OBS's built-in Chroma Key filter can turn into a transparent background.

The end result is the transparent "just hanging out on the screen" look commonly used in streaming & recording. 

Source: [github.com/bobcowher/streaming-background-remover](https://github.com/bobcowher/streaming-background-remover)

**Important:** For now, this is an application I built(with lots of Claude help) to solve my personal problems with OBS's background removal capabilities. As of today(02/22/2026) it has been tested on precisely one machine, with one user, one version of OBS, one version of Python...you get the idea. 

I'm releasing it, free and open source, in the event it solves someone else's problems. I make no claims as to its portability or suitability across applications. 

## Getting Started

### Requirements

- Linux (V4L2)
- NVIDIA GPU (recommended — falls back to CPU)
- Conda
- `v4l2loopback-dkms` package (virtual camera kernel module — `sbr` will offer to load it for you on first run)

### Installation

Clone the repo, activate the conda environment, and run the setup script.

```bash
git clone https://github.com/bobcowher/streaming-background-remover.git
cd streaming-background-remover
conda activate streaming-background-remover
pip install -e .
bash setup.sh
```

The setup script installs `sbr` as a system-wide command via a symlink in `/usr/local/bin`. CUDA libraries (`libcublasLt`, `libcudnn`, etc.) are bundled as pip wheels and loaded automatically at startup — no system-level CUDA installation required. Model weights are downloaded on first run to `~/.cache/sbr/models/`.

## First Run

Just run `sbr`. It handles the setup steps that would otherwise require manual intervention:

1. If one webcam is detected, it starts immediately. If more than one is found, it presents a numbered list and prompts you to choose.
2. If the `v4l2loopback` kernel module is not loaded, it asks whether to load it via `sudo modprobe` and names the virtual device after your webcam — for example, `Full HD webcam (Background Removed)`.
3. The RVM model is downloaded on first run if not already cached.

```bash
sbr
```

## Usage

### List available webcams

```bash
sbr --list-cameras
```

```
[sbr] Available webcams:
  Logitech HD Pro Webcam C920  (/dev/video0)
  Built-in Camera              (/dev/video2)
```

### Specify a webcam

```bash
sbr --source /dev/video0
```

### Background options

The default background is chroma green, intended for use with OBS's Chroma Key filter. You can switch to any solid color for a more traditional virtual background:

```bash
sbr                             # chroma green (default)
sbr --background grey           # solid grey
sbr --background '#1a1a2e'      # custom hex color
```

#### Named colors

| Name | Hex | Notes |
|------|-----|-------|
| `chroma` | `#00FF00` | Default — use with OBS Chroma Key filter |
| `green` | `#00FF00` | Alias for chroma |
| `black` | `#000000` | |
| `white` | `#FFFFFF` | |
| `grey` / `gray` | `#808080` | |
| `red` | `#FF0000` | |
| `blue` | `#0000FF` | |
| `cyan` | `#00FFFF` | |
| `magenta` | `#FF00FF` | |
| `yellow` | `#FFFF00` | |

#### Custom hex colors

Any six-digit hex color works, with or without the `#` prefix:

```bash
sbr --background '#1a1a2e'      # dark navy
sbr --background '2d6a4f'       # forest green
sbr --background '#ffffff'      # white
```

### Model quality

`sbr` selects fp16 or fp32 automatically based on detected GPU VRAM, and chooses between the MobileNetV3 (lite) and ResNet-50 (full) backbone with `--quality`:

```bash
sbr --quality auto    # default: fp16 on GPU with ≥4 GB VRAM, fp32 otherwise
sbr --quality lite    # MobileNetV3 — fastest
sbr --quality full    # ResNet-50 — highest quality
```

### Preview window

```bash
sbr --preview
```

Opens an OpenCV window showing the composited output alongside the virtual camera stream. Press `q` to quit.

### Offline test

Run the pipeline against a video file to verify the model and GPU are working before setting up v4l2loopback:

```bash
sbr --test test_capture.mp4
```

## Using in OBS

1. Start `sbr`. It loads `v4l2loopback` automatically if needed.
2. In OBS → **Sources** → `+` → **Video Capture Device (V4L2)**.
3. Select the device named `<Your Webcam> (Background Removed)`.
4. Right-click the source → **Filters** → `+` → **Chroma Key**. Set key color type to **Green**.

The background becomes transparent. Because the alpha mask from RVM is per-pixel and smooth, edges (hair, hands) blend cleanly without the color-spill artifacts you get from a physical green screen.

## Option Reference

| Flag | Default | Description |
|---|---|---|
| `--source DEVICE` | auto | Input webcam path (e.g. `/dev/video0`) |
| `--background COLOR` | `chroma` | Background fill — named color or `#RRGGBB` |
| `--camera` | off | Always prompt for camera selection |
| `--quality auto\|lite\|full` | `auto` | Model size and precision |
| `--preview` | off | Show local OpenCV preview window |
| `--output-device DEVICE` | auto | V4L2 loopback device to write to |
| `--test VIDEO_FILE` | — | Offline test on a video file |
| `--list-cameras` | — | Print available webcams and exit |
| `--version` | — | Print version and exit |

## Notes

- `sbr` filters V4L2 device nodes by capability, so cameras that expose multiple `/dev/video*` entries (metadata nodes, output nodes) are collapsed to a single entry in the selection list.
- On CPU fallback, the model is automatically reloaded as fp32. Running fp16 on CPU has no native hardware support and produces worse results at slower speeds.
- The `v4l2loopback` `card_label` is set at module load time. The label `sbr` suggests is based on your source webcam's name. If you want to change it after the fact, reload the module with `sudo modprobe -r v4l2loopback` followed by the new `modprobe` command.
- v4l2loopback can be configured to load automatically on boot — see the output of `sbr` when it offers to load the module for the exact config file commands.
