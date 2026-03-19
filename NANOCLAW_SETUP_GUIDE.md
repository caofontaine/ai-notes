# Nanoclaw: Secure Docker Setup Guide

A step-by-step guide to running [Nanoclaw](https://github.com/qwibitai/nanoclaw) in an isolated Docker container on macOS, so it has **no access to your host device**.

---

## What is Nanoclaw?

Nanoclaw is a lightweight, open-source personal AI agent that connects to messaging platforms (WhatsApp, Telegram, Discord, Slack, Signal, Gmail). It runs agent sessions inside isolated containers — each group gets its own sandboxed filesystem, process space, and IPC namespace.

---

## Prerequisites

Based on your current system (**macOS 26.0.1, Apple Silicon arm64, 32 GB RAM**):

| Requirement | Status | Action Needed |
|---|---|---|
| macOS on Apple Silicon (M1+) | Installed | None |
| Git | Installed (v2.50.1) | None |
| Node.js 20+ | Installed (v22.21.1) | None |
| npm | Installed (v10.9.4) | None |
| Homebrew | Installed (v5.0.16) | None |
| Claude Code | Installed (v2.1.72) | None |
| Docker Desktop 4.40+ | **Not installed** | See Step 1 below |
| Claude API key or Claude Code subscription | Unknown | You need one — see Step 3 |
| ~5 GB free disk space | 808 GB available | None |

---

## Step 1: Install Docker Desktop

Docker Desktop provides the container runtime that Nanoclaw uses to isolate its agents from your host machine.

### 1.1 Download Docker Desktop

1. Open your browser and go to: **https://www.docker.com/products/docker-desktop/**
2. Click **"Download for Mac — Apple Silicon"** (your Mac has an M-series chip)
3. This downloads a `.dmg` file (around 700 MB)

### 1.2 Install Docker Desktop

1. Open the downloaded `.dmg` file
2. Drag the **Docker** icon into your **Applications** folder
3. Open **Docker** from your Applications folder (or Spotlight: press `Cmd + Space`, type "Docker", press Enter)
4. macOS will ask for permission — click **Open** and enter your password if prompted
5. Docker Desktop will start and you'll see a whale icon in your menu bar (top-right of screen)

### 1.3 Verify Docker is running

Open your terminal and run:

```bash
docker --version
```

You should see something like `Docker version 28.x.x` or newer. If you get "command not found", wait a moment for Docker Desktop to finish starting, or restart your terminal.

Also verify Docker can run containers:

```bash
docker run --rm hello-world
```

You should see a "Hello from Docker!" message. This confirms Docker is working correctly.

### 1.4 Verify Docker Sandbox support

Nanoclaw uses Docker Sandboxes for hypervisor-level isolation. Confirm your Docker Desktop version supports them:

```bash
docker version --format '{{.Client.Version}}'
```

Make sure the version is **4.40 or higher**. If not, update Docker Desktop via its menu bar icon > "Check for Updates".

---

## Step 2: Clone Nanoclaw

Open your terminal and run:

```bash
git clone https://github.com/qwibitai/nanoclaw.git
cd nanoclaw
```

This downloads the Nanoclaw source code to your machine.

---

## Step 3: Ensure you have a Claude API key

Nanoclaw uses Claude as its underlying AI. You need one of:

- **A Claude Code subscription** (which you already have installed), OR
- **An Anthropic API key** from https://console.anthropic.com/

If using an API key, you'll configure it during the setup process in the next step.

---

## Step 4: Run the guided setup

Nanoclaw has an AI-powered setup process. From inside the `nanoclaw` directory:

```bash
claude
```

This opens Claude Code. Then type:

```
/setup
```

The `/setup` command will interactively walk you through:

1. **Dependency verification** — confirms Node.js, Docker, etc. are installed
2. **Container runtime configuration** — sets up Docker as the container backend
3. **Messaging channel authentication** — connects your WhatsApp, Telegram, Discord, etc.
4. **Service startup** — launches Nanoclaw

Follow the prompts. Claude Code will handle the technical details.

---

## Step 5: Verify container isolation

After setup completes, you can verify that Nanoclaw agents are running in isolated containers:

```bash
docker ps
```

This shows running containers. You should see Nanoclaw's agent containers listed. Each group gets its own container with:

- Its own filesystem (cannot see your host files)
- Its own process namespace (cannot see or interact with host processes)
- Its own IPC namespace (cannot communicate with host processes)
- Only explicitly allowed network access (Anthropic API, messaging platforms)

---

## Alternative: One-line Docker Sandbox install

Nanoclaw also provides an automated install script specifically for Docker Sandboxes. If you prefer this over the manual steps above, run:

```bash
curl -fsSL https://nanoclaw.dev/install-docker-sandboxes.sh | bash
```

This script will:
1. Verify you're on Apple Silicon with Docker Desktop 4.40+
2. Clone Nanoclaw into a timestamped workspace (`~/nanoclaw-sandbox-<timestamp>`)
3. Create a Docker sandbox with the correct configuration
4. Set up network proxy bypass rules for Anthropic API and messaging platforms
5. Launch the sandbox and prompt you to run `/setup`

> **Note:** Always review scripts before piping them to bash. You can inspect it first:
> ```bash
> curl -fsSL https://nanoclaw.dev/install-docker-sandboxes.sh | less
> ```

---

## How the security isolation works

When running inside Docker, Nanoclaw's agents:

- **Cannot access your files** — the container has its own filesystem; only explicitly mounted directories are visible
- **Cannot see your processes** — process isolation means the agent cannot list, inspect, or interfere with anything running on your Mac
- **Cannot access your network freely** — only allowed endpoints (Anthropic API, messaging services) are reachable
- **Cannot escape the container** — Docker uses hypervisor-level isolation on macOS, providing strong boundaries

This is significantly more secure than running agents directly on your machine.

---

## Troubleshooting

### Docker Desktop won't start
- Make sure you have macOS 13 (Ventura) or later (you have macOS 26, so this is fine)
- Try restarting your Mac and opening Docker Desktop again

### `docker` command not found
- Docker Desktop may still be starting — wait 30 seconds and try again
- If it persists, open Docker Desktop from Applications manually

### `/setup` fails during container configuration
- Ensure Docker Desktop is running (whale icon in menu bar)
- Run `docker info` to confirm Docker is responsive
- Check Docker Desktop settings: make sure "Use Virtualization Framework" is enabled

### Permission denied errors
- Docker Desktop on macOS should not require `sudo`
- If you see permission errors, restart Docker Desktop

### Network issues in containers
- The setup configures proxy bypass rules for required services
- If messaging platforms can't connect, check Docker Desktop's network settings

---

## Useful commands

| Command | What it does |
|---|---|
| `docker ps` | List running containers |
| `docker ps -a` | List all containers (including stopped) |
| `docker logs <container_id>` | View logs for a specific container |
| `docker stop <container_id>` | Stop a specific container |
| `docker system prune` | Clean up unused containers and images (frees disk space) |

---

## Further reading

- Nanoclaw documentation: https://nanoclaw.dev
- Nanoclaw source code: https://github.com/qwibitai/nanoclaw
- Docker Desktop documentation: https://docs.docker.com/desktop/
