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
- A Linux server with systemd
- One or more GPUs (optional but recommended)

### Installation

Clone the repo and run the setup script. It creates a virtual environment, installs dependencies, and sets up a systemd service.

```bash
git clone https://github.com/bobcowher/beekeeper.git
cd beekeeper
bash setup.sh
```

The setup script will:

1. Detect your Python version (3.12, 3.11, 3.10, or python3)
2. Create a venv and install dependencies
3. Generate and install a systemd service file (requires sudo)
4. Enable and start the service

Once complete, Beekeeper is running on port 5000. Open `http://your-server:5000` in a browser.

### Managing the Service

```bash
# Check status
sudo systemctl status beekeeper

# View logs
journalctl -u beekeeper -f

# Restart
sudo systemctl restart beekeeper

# Stop
sudo systemctl stop beekeeper
```

### Development Mode

If you prefer to run Beekeeper without systemd for development or testing:

```bash
cd beekeeper
source venv/bin/activate
python app.py
```

This runs Flask's development server on port 5000 with auto-reload.

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

Once setup completes, hit **Start Training** on the project page. Beekeeper pulls the latest code from your branch, then launches the training script as a detached subprocess — closing the browser tab has no effect on the running process.

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

- `projects/` — one subdirectory per project
  - `project.json` — config and state
  - `src/` — cloned git repo
  - `venv/` — Python environment
  - `train.log` — training output
- `app.py` — Flask app
- `routes/` — HTTP endpoints
- `services/` — business logic
- `models/` — data models
- `templates/` — Jinja2 templates
- `static/` — CSS and JS
- `setup.sh` — installation script

## Notes

- Beekeeper runs with a single Gunicorn worker (`-w 1`) because training state is tracked in memory. The setup script configures this automatically.
- Training processes are fully detached — they survive browser disconnects, but not server reboots. After a reboot, the systemd service restarts Beekeeper, but any previously running training jobs will need to be restarted from the UI.
- There's no authentication yet. Don't expose Beekeeper to the public internet without putting it behind a reverse proxy with auth.
