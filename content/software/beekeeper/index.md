---
title: "Beekeeper"
date: 2026-02-15
draft: false
---

Beekeeper is a lightweight web app designed to allow you to do AI training on a remote server as part of your home lab. At its core, it's designed to handle -

1. Cloning a repository. 
2. Setting up the python environment(based on your requirements.txt)
3. Remote log streaming
4. Tensorboard Display
5. File downloads

![Beekeeper](beekeeper.png)

Critical missing features...mostly security stuff.

1. Authentication - Beekeeper has no authentication, and it **does** allow access to files you've cloned or generated in your training run. For now, I would strongly recommend running Beekeeper only in a home lab scenario, where the server is sitting safely on your local network, and avoiding any sensitive data. 
2. GitHub auth - Beekeeper has no method of authenticating with your remote repo. It only works on repos you've made public. 
3. Https - For https, you'll need to put Beekeeper behind a proxy and, again, it's not ready to do anything secure anyway. 
4. Multi-server support - Eventually, I'd like to have a central Hive server managing multiple workers, and farming jobs out. Today is not that day. This is a single server product. 


## Getting Started

### Requirements

- Python 3.10+
- Git
- A Linux server with systemd(Currently tested on Ubuntu)
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
| Branch | Git branch to clone and pull before each run | `main` |
| Python Version | Detected from system and conda | auto |
| Environment Type | venv or conda | `venv` |
| Training Script | Python file to execute when training starts | `train.py` |
| Tensorboard Log Dir | Where your script writes TB event files | `runs` |
| Requirements File | Pip requirements file installed at setup and before each run | `requirements.txt` |
| Setup Script | Optional shell script run at setup and before each training run | — |
| Data Dir (local) | Local path in the repo to symlink to your data volume | `data` |
| Data Dir (system) | Absolute path on the server to a persistent data volume | — |

Every field has a tooltip — hover the **?** icon for a description.

Once you submit, Beekeeper runs the following in the background:

1. **Git clone** — clones the repository at the specified branch into a `workspace/` directory
2. **Create environment** — creates a venv or conda env with the selected Python version
3. **Data dir symlink** — if enabled, creates a symlink from `workspace/<local path>` to the system data directory
4. **Setup script** — if configured and the file exists, runs it from the workspace root
5. **Pip install** — installs packages from the requirements file

The project page refreshes automatically and shows the current step. If any step fails, the error is displayed and a **Retry Setup** button appears. Retry is smart — it skips the clone and environment creation if they already completed successfully, and picks up from the failed step.

## Running Training

Once setup completes, hit **Start Training** on the project page. Beekeeper runs the following sequence before launching your script:

1. **Git pull** — pulls the latest code from your configured branch
2. **Data dir symlink** — verifies or creates the symlink if a data directory is configured
3. **Setup script** — runs your setup script if configured and present
4. **Pip install** — installs/updates packages from the requirements file
5. **Launch** — starts the training script as a detached subprocess

If any step fails, training is aborted and the error is shown on the project page.

Closing the browser tab has no effect on the running process.

The project page shows:

- **Status** — running, stopped, crashed, or idle
- **PID and elapsed time** while running
- **Live logs** — expand the Logs section to stream stdout/stderr in real time. Each run starts with a header showing the timestamp, hostname, git commit SHA, branch, Python version, training script, and GPU info, and ends with a footer showing elapsed time and exit status.
- **Tensorboard** — auto-starts alongside training with a dynamic port, embedded as an iframe with an option to open in a new tab

Hit **Stop Training** to send SIGTERM (with a SIGKILL fallback after 5 seconds).

## Environment Variables

Training scripts often need environment variables — API keys, config flags, hyperparameters. Click **Edit** on the project info card to add key-value pairs. These are passed to the training process at startup.

## Setup Script

If your project needs system-level setup beyond pip — downloading a dataset, linking shared weights, generating config files — you can point Beekeeper at a shell script.

```bash
# example setup.sh (place this in your repo root)
#!/bin/bash
set -e

mkdir -p data

if [ ! -f data/iris.csv ]; then
    echo "Downloading dataset..."
    curl -fsSL https://raw.githubusercontent.com/mwaskom/seaborn-data/master/iris.csv \
         -o data/iris.csv
fi
```

Set **Setup Script** to `setup.sh` (or whatever your script is named) when creating or editing a project. Beekeeper will run it from the repository root:

- Once during initial project setup (after the environment is created, before pip install)
- Again before every training run (after git pull, before pip install)

The script is silently skipped if the file doesn't exist, so you can set it once and it won't cause errors on environments that don't have it yet.

## Data Directory

For projects that need access to a large persistent dataset stored elsewhere on the server — a mounted NAS share, a shared `/data` volume, or any local path — use the Data Directory fields.

| Field | Purpose |
|-------|---------|
| Data Dir (local) | Path within the repo to create as a symlink (default: `data`) |
| Data Dir (system) | Absolute path on the server to link to |

Beekeeper creates a symlink at `src/<local>` → `<system path>` during project setup, and ensures it exists again before each training run. Your training script just reads from `data/` as if the dataset lived inside the repo.

Leave the system path blank if you don't need this feature.

## Editing Project Settings

Click **Edit** on the project page to change:

- Git branch
- Training script path
- Tensorboard log directory
- Requirements file
- Setup script
- Data directory (local and system paths)
- Environment variables

Name, Git URL, Python version, and environment type are fixed after creation.

## Viewing and Downloading Files

Expand the **Files** section on the project page to browse the project's workspace directory. You can preview files inline, download individual files, or download entire directories as zip archives.

### Inline Viewer

Click any viewable filename or the **view** button to open it in a modal without leaving the page.

| File type | Extensions | Behavior |
|-----------|------------|----------|
| Images | png, jpg, jpeg, gif, webp, svg, bmp, ico | Rendered inline. **Auto-refreshes every 2 seconds** — useful for monitoring debug images written during training. |
| Text / code | py, log, json, yaml, md, sh, csv, toml, js, ts, html, xml, and more | Displayed in a monospace viewer. Files over 1 MB fall back to download. |

Close the viewer with the **×** button, by clicking the backdrop, or by pressing Escape.

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

## REST API

Beekeeper exposes a REST API for programmatic control of projects. All endpoints return JSON with a consistent format:

```json
{"success": true, "data": {...}}
{"success": false, "error": {"code": "...", "message": "..."}}
```

### Key Endpoints

| Action | Method | Endpoint |
|--------|--------|----------|
| List projects | GET | `/api/v1/projects` |
| Start training | POST | `/api/v1/projects/<name>/training/start` |
| Stop training | POST | `/api/v1/projects/<name>/training/stop` |
| Check status | GET | `/api/v1/projects/<name>/training/status` |
| Get logs | GET | `/api/v1/projects/<name>/logs?tail=100` |
| Get metrics | GET | `/api/v1/projects/<name>/tensorboard/latest` |
| List files | GET | `/api/v1/projects/<name>/files` |
| Download file | GET | `/api/v1/projects/<name>/files/<path>` |
| System stats | GET | `/api/v1/stats` |

### TensorBoard Metrics Analysis

The `/tensorboard/latest` endpoint analyzes your training metrics and returns insights:

```bash
curl http://your-server:5000/api/v1/projects/my-project/tensorboard/latest?detail=medium
```

**Which run is analyzed?**
- If training is running, it analyzes the **current active run**
- If training is idle, it analyzes the **most recent completed run**
- The response includes `is_active: true/false` to indicate which

**To compare with past runs:**
```bash
# List all runs
curl http://your-server:5000/api/v1/projects/my-project/runs

# Get metrics for a specific past run
curl http://your-server:5000/api/v1/projects/my-project/runs/3/metrics
```

The response includes trend analysis, convergence detection, and anomaly detection for each metric:

- **trend**: `improving`, `stable`, `worsening`, or `unstable`
- **converged**: boolean indicating if the metric has stabilized
- **anomalies**: array of unusual spikes or drops
- **summary**: human-readable interpretation

## Agent Integration

> **Beta Feature** — Available in the `develop` branch. Not yet in stable release.

Beekeeper can be controlled by AI agents (like Claude Code). Each project page has an **API** section with two subsections:

- **Human**: curl examples for command-line use
- **Agent**: downloadable instructions file

### Setting Up an Agent

1. Open your project in Beekeeper
2. Expand **API** → **Agent**
3. Click **Download BEEKEEPER_\<project\>.md** (or use the curl command shown)
4. Add the file to your project's root directory or `~/.claude/`

The downloaded file contains:
- Quick reference table of all endpoints
- Pre-flight check guidance (always check status before start/stop)
- Terminology mapping ("check logs" → which endpoint to use)
- Detailed metrics interpretation guide
- Common workflows

Agents can then control training runs, monitor progress, analyze metrics, and download results via HTTP requests

## Run History

Each project tracks its training runs. Expand the **Run History** section on the project page to see:

- Start time and duration
- Status (completed, crashed, canceled)
- Git commit at the time of the run
- Download link for archived logs

Run logs are automatically archived when training completes or is stopped. The history is pruned to keep the last 20 runs.

## Organizing Projects

As your project list grows, the dashboard gives you two tools to stay organized.

### Sort Order

A toggle in the Projects header switches between:

- **Last Run** (default) — projects you've trained most recently float to the top. Projects that have never been run sink to the bottom.
- **A–Z** — alphabetical order.

Your preference is saved in the browser and remembered across sessions.

### Pinning

Click the 📌 icon on any project row to pin it. Pinned projects always appear above the sorted list, regardless of sort order. Click again to unpin. Pin state is saved in `project.json` on the server.

## System Stats

The dashboard shows live GPU, CPU, and memory stats, updated every 2 seconds. GPU monitoring uses nvitop for detailed per-GPU utilization, temperature, VRAM, and power draw.

## Project Structure

Beekeeper organizes everything under its install directory:

- `projects/` — one subdirectory per project
  - `project.json` — config and state
  - `workspace/` — cloned git repo
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
