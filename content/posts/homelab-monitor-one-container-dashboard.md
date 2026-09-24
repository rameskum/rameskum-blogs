---
title: 'One Container to Watch Them All: homelab-monitor Replaces Your Uptime + Grafana Tabs'
date: 2026-09-24T08:53:00-04:00
draft: false
ShowToc: true
description: >-
  Prometheus + Grafana is five containers and a weekend of YAML; UptimeRobot is
  a subscription. homelab-monitor is one container: per-container GPU/VRAM
  truth, power-cost tracking, SSH fleet monitoring with no agents, built-in
  uptime checks, and push alerts. Full setup guide.
tags:
  - homelab
  - docker
  - self-hosted
  - monitoring
  - gpu
categories: article
keywords:
  - homelab monitoring dashboard
  - docker monitoring single container
  - uptimerobot alternative self-hosted
  - grafana alternative homelab
  - gpu monitoring homelab
---

My monitoring stack used to be five containers: Prometheus, Grafana, node-exporter, cadvisor, and an uptime checker — plus a paid UptimeRobot plan for the "is the house on fire" pings. Total maintenance burden: a weekend to set up, and a Sunday every few months when something in the pipeline silently died.

[homelab-monitor](https://github.com/SikamikanikoBG/homelab-monitor) replaces all of it with **one container**. Pure Python + Flask, no agents, no Prometheus, no cloud. It does the thing most dashboards skip: per-container GPU/VRAM attribution — which container is actually squatting on your VRAM — plus power-cost tracking, multi-machine monitoring over SSH, and built-in uptime checks with push alerts.

## The 30-second deploy

```bash
mkdir -p homelab-monitor/data && cd homelab-monitor
curl -fsSLO https://raw.githubusercontent.com/SikamikanikoBG/homelab-monitor/main/docker-compose.yml
docker compose up -d
# → http://<host>:9800
```

That upstream compose file is the real thing (179 lines, well-commented). If you prefer to own the config, here's the trimmed working version with the essentials:

```yaml
services:
  homelab-monitor:
    image: sikamikaniko123/homelab-monitor:latest
    container_name: homelab-monitor
    restart: unless-stopped
    network_mode: host     # reaches model APIs, serves the dashboard directly
    pid: host              # maps GPU PIDs back to containers
    cap_add:
      - SYS_PTRACE
    security_opt:
      - apparmor=unconfined
    environment:
      - PORT=9800
      - SAMPLE_INTERVAL=10      # seconds between samples
      - RETENTION_DAYS=180      # SQLite history retention
      - PRESSURE_FREE_MB=2048   # free VRAM below this = "pressure"
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=utility  # injects nvidia-smi; inert without a GPU
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /:/rootfs:ro                              # disk stats
      - /sys/class/powercap:/sys/class/powercap:ro # RAPL CPU/DRAM power counters
      - ./data:/data                               # SQLite history persists here
      - /run/dbus/system_bus_socket:/run/dbus/system_bus_socket  # systemd status
```

No GPU? It starts anyway — the GPU panels just stay dark until one's present. There's deliberately **no** hard `deploy.devices: nvidia` reservation, because that would refuse to start on a GPU-less host.

### The NVIDIA runtime gotcha

The `NVIDIA_*` env vars only take effect when nvidia is Docker's **default** runtime. On stock Ubuntu/Debian the default is `runc`, so the GPU goes undetected until you do one of:

```bash
# Option A: bring it up with the project's GPU override
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up -d

# Option B: make nvidia the default runtime (then RECREATE, not restart)
sudo nvidia-ctk runtime configure --runtime=docker --set-as-default
sudo systemctl restart docker
docker compose up -d --force-recreate
```

## What it replaces, tab by tab

| Instead of | You get |
|---|---|
| Grafana dashboards you maintain | Overview, GPU, Costs, Containers, Services, Disks tabs — live, no queries to write |
| UptimeRobot / BetterStack ($/mo) | Built-in uptime checks: any HTTP endpoint or TCP port, heartbeat strip, 24h/7d uptime %, latency |
| `watch nvidia-smi` | **GPU truth**: throttle reasons (red banner the moment it's power-capped or too hot), memory-bandwidth util, clocks, power-vs-limit, p-state |
| Guessing which container ate VRAM | Per-container GPU/VRAM attribution — RAM and VRAM in **separate columns** (real resident RAM, not page cache); click a container to tail its logs |
| htop over SSH | Mini-htop, per-container network top talkers |
| WinDirStat / `ncdu` | WizTree-style disk treemaps — on the hub *or any SSH host* |
| A power meter and a spreadsheet | Costs page: watts → kWh → money, per machine, per component, per process/container; day/night tariffs; 7×24 busy-hours cost heatmap |

The uptime checks deserve a callout because they're the paid-subscription killer: smart per-check alerts with **anti-flap confirmation** (no alert storms when a service bounces), recovery notifications with downtime duration, and optional slow-response warnings. Push alerts go to **Discord, ntfy.sh, or Telegram**, edge-triggered so they don't spam. Point ntfy at a topic and your phone buzzes when something actually dies — that's the whole UptimeRobot replacement.

## The SSH fleet: no agents, anywhere

This is the part that sold me. In the dashboard you paste **one SSH key per box** — a Proxmox node, a Pi, a Windows machine, your Unraid NAS — and it's monitored. Nothing to install on the remote end. The GPU tab works per host too: a remote multi-GPU rig shows every card's VRAM, utilization, power, temperature, and which processes hold the memory. Disk treemaps run over the same SSH connection.

```bash
# On each remote box: nothing. Literally nothing.
# In the dashboard: Hosts → Add → paste IP + key. Done.
```

For an Unraid box (SMB shares, Docker containers, the usual), it shows up like any other Linux host — container health, disk usage, the lot.

## The Costs page: your power bill, itemized

In Settings, set your electricity price (`kwh_price`, optional night tariff with `night_start`/`night_end`, currency). The monitor integrates GPU power draw — plus CPU via RAPL — into kWh and then money, per machine, per component, and per process/container/model you click into. The busy-hours heatmap shows *when* your lab actually costs you money across the week.

This is genuinely useful for the "should I leave the GPU rig on overnight" question: run the training job, check the Costs page in the morning, and you know what the run cost — the Experiments feature even pairs a pushed run (from Jupyter/Colab/Kaggle, or mirrored from MLflow) with the loss curve **and** the real GPU energy it burned, on one timeline.

## The AI-lab extras

If you run local models, the Benchmark Lab is a sortable leaderboard of your models: tokens/sec, VRAM fit, recommended context, plus a context-sweep chart. Live serving stats — real tokens/sec, queue depth, KV-cache, TTFT — come straight from vLLM/TGI, and Ollama shows param size, quant, and context at a glance. And there's a **read-only MCP server** on port 9810, so an AI agent can query your lab state:

```bash
claude mcp add --transport http homelab http://<host>:9810/mcp
```

Read-only by design — the agent can look, not touch.

## Hardening notes (read before exposing anything)

This container is deliberately privileged — host networking, `pid: host`, the Docker socket, D-Bus. That's what makes one container able to see everything. Treat it accordingly:

- **Do not expose port 9800 to the internet.** Put it behind your VPN/Tailscale or reverse proxy with auth. (There's an optional `PUBLIC_STATUS=1` read-only status page if you want a shareable view — that's the safe way to publish.)
- The Docker socket is mounted **read-write** by default, which powers the one-click self-update button and the Containers tab's start/stop/restart actions. If that makes you twitch, set `ALLOW_SELF_UPDATE=0` and `ENABLE_CONTROLS=0`, or use the provided `docker-compose.readonly.yml` to lock the socket back to `:ro`.
- The MCP server is read-only and safe on your LAN; you can pin it with `MCP_ALLOWED_HOSTS` or disable it with `ENABLE_MCP=0`.

## The verdict

| | Prometheus + Grafana + UptimeRobot | homelab-monitor |
|---|---|---|
| Containers | 5+ | 1 |
| Setup time | a weekend of YAML | `docker compose up -d` |
| Uptime checks | paid plan | built in, with anti-flap alerts |
| Per-container GPU/VRAM | custom exporter plumbing | a column in the UI |
| Power cost | spreadsheet | Costs tab with tariffs |
| Remote hosts | exporters everywhere | one SSH key per box, no agents |
| Maintenance | yours, forever | one-click self-update |

It's not a replacement for Prometheus if you're running a fleet with SLOs and alert routing — that's a different job. But for a homelab, it deletes an entire category of maintenance: no exporters, no dashboard JSON, no subscription for "is it down" pings. One container, one page, and finally an answer to "what is the GPU actually doing."
