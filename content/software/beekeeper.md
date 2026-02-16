---
title: "Beekeeper"
date: 2026-02-15
draft: false
---

Beekeeper is a lightweight web app for managing long-running AI training runs on a remote server. Point it at a Git repo, and it handles cloning, environment setup, process management, live log streaming, Tensorboard, and file downloads — all from a browser.

https://github.com/bobcowher/beekeeper

## Why Beekeeper

Training runs take hours or days. You SSH into a server, start a script, hope the tmux session survives, and check back later. Beekeeper replaces that workflow with a web UI that lets you start, stop, and monitor training from anywhere — including your phone.

It's a single Python app with no database. Project configs are JSON files. The whole thing runs in one process.

## Getting Started

### Requirements

- Python 3.10+
- Git
- A Linux server with one or more GPUs (optional but recommended)

### Installation

```bash
git clone https://github.com/robertcowher/beekeeper.git
cd beekeeper
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Running

```bash
# Development
python app.py

# Production (single worker required for in-memory state)
gunicorn -w 1 -b 0.0.0.0:5000 "app:create_app()"
```

Beekeeper runs on port 5000 by default. Open `http://your-server:5000` in a browser.

## Creating a Project

From the dashboard, click **+ New Project** and fill in:

| Field | Description | Default |
|-------|-------------|---------|
| Project Name | No spaces, used as the directory name | — |
| Git URL | Public Git repository URL | — |
| Branch | Git branch to clone | `main` |
| Python Version | Detected from system and conda | auto |
| Training Script | Python file to run | `train.py` |
| Tensorboard Log Dir | Where your script writes TB logs | `runs` |
| Requirements File | Pip requirements to install | `requirements.txt` |
| Environment Type | venv or conda | `venv` |

Beekeeper clones the repo, creates an isolated environment, and installs dependencies in the background. The project page shows setup progress in real time.

## Running Training

Once setup completes, hit **Start Training** on the project page. Beekeeper runs your training script as a detached subprocess — closing the browser tab has no effect on the running process.

The project page shows:

- **Status** — running, stopped, crashed, or idle
- **PID and elapsed time** while running
- **Live logs** — expand the Logs section to stream stdout/stderr in real time
- **Tensorboard** — auto-starts alongside training with a dynamic port, embedded as an iframe with an option to open in a new tab

Hit **Stop Training** to send SIGTERM (with a SIGKILL fallback after 5 seconds).

## Environment Variables

Training scripts often need environment variables — API keys, config flags, hyperparameters. Click **Edit** on the project info card to add key-value pairs. These are passed to the training process at startup.

## Editing Project Settings

Click **Edit** on the project page to change:

- Git branch
- Training script path
- Tensorboard log directory
- Requirements file
- Environment variables

Name, Git URL, Python version, and environment type are fixed after creation.

## Downloading Files

Expand the **Files** section on the project page to browse the project's source directory. You can download individual files or entire directories as zip archives.

### Using curl

The same endpoints that power the UI work with curl:

```bash
# List files in the project root
curl http://your-server:5000/projects/my-project/files/

# Download a specific file
curl -O http://your-server:5000/projects/my-project/files/checkpoints/model.pt

# Download a directory as a zip
curl -o checkpoints.zip 'http://your-server:5000/projects/my-project/files/checkpoints/?zip=1'
```

The JSON listing includes file names, sizes, and types — useful for scripting downloads of specific checkpoints or outputs.

## Tensorboard

Tensorboard starts automatically when training starts and stops when training stops. It runs from the project's own environment, so it uses whatever version of Tensorboard is in the project's requirements.

The port is allocated dynamically starting at 6006. You can:

- View it inline in the iframe on the project page
- Expand the iframe to full height
- Open it directly in a new browser tab
- Clear accumulated Tensorboard logs with the **Clear Tensorboard Logs** button

## System Stats

The dashboard shows live GPU, CPU, and memory stats, updated every 2 seconds. GPU monitoring uses nvitop for detailed per-GPU utilization, temperature, VRAM, and power draw.

## Project Structure

Beekeeper organizes everything under its install directory:

```
beekeeper/
├── projects/
│   └── my-project/
│       ├── project.json    # Config and state
│       ├── src/            # Cloned git repo
│       ├── venv/           # Python environment
│       └── train.log       # Training output
├── app.py                  # Flask app
├── routes/                 # HTTP endpoints
├── services/               # Business logic
├── models/                 # Data models
├── templates/              # Jinja2 templates
└── static/                 # CSS & JS
```

## Notes

- Beekeeper must run with a single Gunicorn worker (`-w 1`) because training state is tracked in memory. Multiple workers would each have their own state.
- Training processes are fully detached — they survive browser disconnects, but not server reboots. After a reboot, the project status in `project.json` will still say "running" but the process will be gone. Restart training from the UI.
- There's no authentication yet. Don't expose Beekeeper to the public internet without putting it behind a reverse proxy with auth.
