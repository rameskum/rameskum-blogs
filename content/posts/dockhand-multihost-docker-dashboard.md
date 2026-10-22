---
title: 'Dockhand 1.0.54: One Dashboard to Run Every Docker Host I Own'
date: 2026-10-11T11:30:00-04:00
draft: false
ShowToc: true
description: >-
  Five Docker hosts, one browser tab. Dockhand 1.0.54 plus the Hawser agent
  gives you Portainer-style multi-host management — stacks, terminals,
  GitOps auto-sync, OIDC and vulnerability scans — free for homelab use.
  Working compose, Standard vs Edge agent modes, and the honest Portainer math.
tags:
  - self-hosted
  - docker
  - homelab
categories: article
keywords:
  - dockhand docker management
  - hawser agent multi host
  - portainer alternative
  - docker gui self hosted
---

The old routine: SSH into the NAS, `docker ps`. SSH into the app box, `docker ps`. SSH into the VPS, `docker ps`. Somewhere in there a container has been crash-looping for three days and you find out from a user, not from your own tooling.

Dockhand ([Finsys/dockhand](https://github.com/Finsys/dockhand), v1.0.54) is the fix: a self-hosted Docker management UI that runs every host you own from one dashboard — containers, Compose stacks, terminals, logs, metrics, GitOps deploys, backups, and vulnerability scans. The multi-host part comes from its companion, the Hawser agent. Here's the full setup.

## The honest license note first

Dockhand is **not** MIT/Apache open source — it's published under the Business Source License 1.1. (GitHub reports the repo license as `NOASSERTION` because the terms ship as `LICENSE.md`/`LICENSE.txt` rather than a plain `LICENSE` file — but the text itself is explicit, so here it is from the LICENSE's own Parameters.) The **Additional Use Grant** permits production use only while at least one of these applies: **(a)** you are "an individual using the Licensed Work for personal, non-commercial purposes (for example a home lab or a hobby project), and not for the benefit of any business or other organization"; **(b)** a non-profit or charitable organization; **(c)** a school, university or non-commercial research institution; or **(d)** a sole operator running infrastructure you operate yourself, with no employee using it alongside you. Beyond that, the LICENSE is blunt: "Any other production use — including production use by or on behalf of a company, public body or any other organization that has employees — requires a Commercial License." Non-production use — development, testing, staging, CI/CD, evaluation, demos, training — is free for everyone, companies included, with no license key and no user limits. The **Change Date** is "four years from the date on which a given version of the Licensed Work is first publicly distributed", and the **Change License** is the Apache License, Version 2.0 — so each release becomes Apache 2.0 four years after it ships. One more hard line: offering Dockhand, or a fork of it, as a hosted, managed or SaaS service to third parties is never permitted without a separate written agreement. If "source-available, free for homelab" works for you — and for most of us it does — read on.

## The hub: one container

Dockhand itself is a single container. It manages its own host through the Docker socket, and everything else through agents:

```yaml
services:
  dockhand:
    image: fnsys/dockhand:1.0.54
    container_name: dockhand
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      # manage the host Dockhand runs on
      - /var/run/docker.sock:/var/run/docker.sock
      # persist Dockhand's own data (database, stacks, settings)
      - dockhand_data:/app/data

volumes:
  dockhand_data:
```

The `dockhand_data` volume is not decoration: Dockhand keeps its database, stacks, and settings in `/app/data` — that's the `DATA_DIR` default, and it's the exact mount in the project's own compose file. Skip it and a container re-create wipes your entire Dockhand state. (Prefer a host directory you can back up directly? Bind-mount it and set `DATA_DIR` to the container path, as the manual describes.)

Pin the version tag rather than floating on `latest` — you want to read the changelog *before* the dashboard upgrades itself, not after. Open `http://<host>:3000`, create the admin user, and you've got host number one.

## Hawser: the agent that connects every other host

Hawser ([Finsys/hawser](https://github.com/Finsys/hawser)) is Dockhand's lightweight Go agent. One per remote host, and it has two modes — picking the right one is the whole multi-host design:

**Standard mode** — the agent listens, Dockhand connects to it. Right for your LAN, where hosts have stable addresses:

```bash
docker run -d \
  --name hawser \
  --restart unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -p 2376:2376 \
  -e TOKEN=your-secret-token \
  ghcr.io/finsys/hawser:latest
```

Note the `TOKEN`: Hawser **refuses to start** on a non-loopback bind without one. That's not bureaucracy — it's the agent refusing to expose your Docker API to the network unauthenticated. Generate something long, store it once, move on. Add TLS certs in front if the agent traffic crosses anything you don't fully trust.

> ⚠️ One warning that deserves its own line: the agent mounts `/var/run/docker.sock` and serves it on the network behind that single shared token — **whoever holds the token is effectively root on that host**. Put TLS in front of it and firewall port 2376 so only the Dockhand host can reach it (or front the socket with a socket proxy that limits the exposed API surface).

**Edge mode** — the agent dials *out* to Dockhand over a WebSocket. Right for the VPS behind NAT, the box on a dynamic IP, anything Dockhand can't reach inbound:

```bash
docker run -d \
  --name hawser \
  --restart unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  ghcr.io/finsys/hawser:latest \
  --server wss://dockhand.yourdomain.com/api/hawser/connect \
  --token your-secret-token
```

No inbound firewall rules, no port forwards — the agent holds the connection open and Dockhand talks back through it. This is the mode that makes "my homelab plus a $6 VPS" actually manageable from one place.

In the Dockhand UI: Environments → Add → agent, paste the address or accept the Edge connection, done. Each host gets its own dashboard tile with live CPU, memory, and disk.

## What you actually get

- **Environments overview** — every host, container counts, resource pressure, one click to switch.
- **Compose stacks with a visual editor** — edit YAML side-by-side with env vars, deploy, redeploy, roll back. Import your existing `docker-compose.yml` files and stop keeping them only in your head.
- **GitOps** — point a stack at a Git repo with webhook triggers and scheduled auto-sync. Your compose files live in Git; the dashboard just executes.
- **Terminal, logs, file browser** — exec into any container, stream logs, browse and edit files inside containers *and* volumes, without SSH.
- **Metrics and image management** — per-host dashboards, image sizes, update badges, prune the cruft.
- **Backups** — restic destinations with scheduled backup, restore, check, and prune. Your stacks' volumes, actually backed up, on a schedule, instead of "I'll tar it someday."
- **Vulnerability scans** — Grype and Trivy CVE results per image, right next to the update badge. The scan that tells you your three-year-old `nginx:alpine` has opinions.

## Hardening: OIDC, MFA, and the agent token

Don't expose the dashboard to the internet behind just a password. Dockhand ships **OIDC/SSO and MFA** — wire it to your identity provider (Authelia, Authentik, whatever you run) and put it behind your reverse proxy with TLS, same as every other service. Local users exist for the simple case.

On the agent side: the `TOKEN` is a shared secret, so treat it like one — unique per host, stored in your secrets management, rotated if it ever leaks. Standard-mode agents should bind TLS when the traffic leaves the trusted LAN; Edge mode rides your existing TLS termination at the Dockhand endpoint.

## The Portainer math, honestly

Portainer CE is genuinely open source (zlib license) and fine for one host. To be precise about the tiers: CE does include external authentication — OAuth and LDAP are right there in Portainer's settings docs — so SSO alone isn't the paywall. What lives in the paid Business tier is RBAC (and registry management), and the agent model comes with Portainer's own complexity. Dockhand gives you OIDC/SSO, MFA, multi-host via Hawser, GitOps auto-sync, restic backups, and Grype/Trivy scans **free for personal/homelab use** — with the BSL caveat above. The trade: no Kubernetes (Docker-only — if you need k8s, Portainer still wins), a younger project, and a smaller community. For a Docker-only homelab, it's the better fit today.

## Update playbook

```bash
# hub
docker compose pull && docker compose up -d
# agents — there is no compose file on these hosts; they were started with
# docker run, so update them the same way: pull, remove, re-run
docker pull ghcr.io/finsys/hawser:latest
docker rm -f hawser
# then re-run the exact docker run command from the Hawser section above
# (Standard or Edge variant) — --restart unless-stopped brings it back up
```

Read the release notes before pulling the hub — Dockhand ships fast (1.0.48 → 1.0.54 in about a month), and stack-management software is exactly the thing you don't auto-upgrade blind. Pin, read, then pull. Same discipline for the agent: Hawser publishes versioned releases, so if you'd rather pin than float on `:latest`, put the version tag in both the install `docker run` and the update pull above — just keep the two in sync.

## Cheat sheet

| Task | How |
|---|---|
| Hub install | `fnsys/dockhand:1.0.54`, port `3000`, socket mount, `dockhand_data:/app/data`, pinned tag |
| LAN host | Hawser Standard mode: listens on `2376`, `TOKEN` required |
| VPS / NAT host | Hawser Edge mode: dials out via `wss://…/api/hawser/connect` |
| Agent image | `ghcr.io/finsys/hawser:latest` + `/var/run/docker.sock` mount |
| Auth | OIDC/SSO + MFA; local users for the simple case |
| GitOps | Stack from Git repo, webhook + scheduled auto-sync |
| Backups | Restic destinations, scheduled backup/restore/prune |
| Vuln scans | Grype + Trivy per image, next to the update badge |
| License | BSL 1.1 — free for homelab/personal, commercial for org production |
| Updates | Pin, read the changelog, then pull |

Five hosts, one tab, zero SSH sessions to check on containers. The Hawser Edge mode is the quiet hero — it's what turns "a VPS I occasionally remember" into infrastructure you actually see.
