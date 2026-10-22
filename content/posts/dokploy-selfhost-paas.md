---
title: 'Dokploy: Your Own Vercel in One Docker Install'
date: 2026-10-22T09:15:00+05:30
draft: false
ShowToc: true
description: >-
  Dokploy is a self-hosted PaaS — Docker Swarm, Traefik with automatic
  Let's Encrypt, Git push-to-deploy, preview environments, one-click
  databases with S3 backups — installed with one command on a cheap VPS.
  How to deploy your Next.js apps to it, harden the panel, and the honest
  license and cost math.
tags:
  - docker
  - self-hosted
  - homelab
  - devops
categories: article
keywords:
  - dokploy self hosted paas
  - vercel alternative docker
  - dokploy install guide
  - self host nextjs dokploy
---

Vercel's developer experience is genuinely excellent — push to Git, get a URL, previews for every pull request. The bill is the problem: per-seat pricing plus usage, and it scales with your success. Dokploy is the self-hosted answer that's been eating that market: an open-source PaaS that gives you the same workflow — Git deploys, automatic HTTPS, preview environments, one-click databases — on a VPS you control, installed with a single command. Here's the full setup, including the parts the highlight reels skip.

## What Dokploy actually is

Under the dashboard, Dokploy is Docker Swarm plus Traefik. The installer sets up three services: Dokploy itself, its own PostgreSQL, and Redis. Traefik is the reverse proxy — when you attach a domain to an app, Dokploy is writing Traefik routing and Let's Encrypt certificate config for you behind the scenes. If you already run Traefik by hand, Dokploy will feel familiar: it's a polished management layer over the stack you'd otherwise wire yourself, plus the Git integration and database provisioning you'd otherwise build.

What you get:

- **Three build modes:** your Dockerfile, Nixpacks (auto-detects the stack), or a full Docker Compose file deployed as-is.
- **Git integration** with GitHub, GitLab, Bitbucket, and Gitea — push to the configured branch and it redeploys.
- **Preview deployments** per pull request (GitHub), Vercel-style.
- **One-click databases:** PostgreSQL, MySQL, MariaDB, MongoDB, Redis — with scheduled backups to S3-compatible storage (S3, R2, MinIO).
- **Monitoring and notifications:** per-container logs, CPU/RAM/disk, alerts to Slack, Discord, or email.
- **Multi-server:** add remote servers and deploy across a Swarm cluster from one dashboard.
- **Templates** for the usual one-click self-hosted apps, plus an API and CLI.

## Install: one command, three hard requirements

On a fresh server (Ubuntu 22.04/24.04 and Debian 11/12 are the well-tested paths):

```bash
curl -sSL https://dokploy.com/install.sh | sh
```

One caution before you run it: this is a `curl | sh` install on a server that will end up holding your Git deploy credentials and S3 backup keys. Download the script and read it first — `curl -sSL https://dokploy.com/install.sh -o install.sh`, skim it, then run it — or pin to a specific release, rather than piping an unseen script straight into a root shell.

The script installs Docker if it's missing, initializes Swarm, and starts the stack. Then open `http://<your-server-ip>:3000` and create the admin account — immediately, and treat this as a race you're meant to win: until that first account exists, the setup page has no authentication at all, and whoever reaches `:3000` first becomes the admin. Restrict port 3000 with the firewall to your own IP (or reach the panel over an SSH tunnel) until the admin account exists, and read the hardening section below before you expose the box, not after.

The requirements are not suggestions:

- **Ports 80, 443, and 3000 must be free** — the installer fails if anything occupies them. This is the big one: Dokploy runs *its own* Traefik, so it does **not** drop in alongside an existing Traefik/Portainer setup on the same host. Give it a fresh VPS or a dedicated VM.
- **2 GB RAM minimum, 4 GB recommended**, ~30 GB disk. Dokploy itself idles around 250 MB; the rest is headroom for your apps and databases.
- A small VPS in the €4/month class (2 vCPU, 4 GB) runs a meaningful number of small apps — that's the entire hosting bill.

## Deploying a Next.js app

The workflow, start to finish:

1. **Create a project**, add an application, connect your Git repository.
2. **Pick the build method.** For a Next.js storefront or admin dashboard, Nixpacks detects and builds it with zero config; if you already have a Dockerfile, use it verbatim.
3. **Set environment variables** in the dashboard — your Stripe and Clerk keys live here, not in the repo.
4. **Add a domain.** Point DNS at the server; Dokploy provisions the Let's Encrypt certificate through Traefik automatically. HTTPS with no certbot, no renewal cron.
5. **Deploy.** Every push to the configured branch redeploys. Open a pull request and you get a preview environment with its own URL.

For multi-container apps — say the storefront plus its Postgres — deploy the repo's `docker-compose.yml` directly, or provision the database from Dokploy's Databases section and connect with the provided internal connection string.

## Backups: three layers, one caveat

- **Databases:** each managed database has its own backup schedule to your S3 destination.
- **Volumes:** Dokploy backs up Docker volumes too — but **only named volumes, not bind mounts**. If your compose files bind-mount host directories (many homelab stacks do), those are *not* covered. Check before you trust it.
- **Dokploy itself:** under the web server settings, Dokploy backs up its own database and its `/etc/dokploy` filesystem into one scheduled zip to S3. Configure the S3 destination first.

Two honest warnings: Dokploy doesn't encrypt the backup archives itself, so enable encryption at rest on the bucket and scope the credentials to that bucket only. And the rule that outlives every tool: a backup you haven't restored isn't a backup — schedule a restore drill.

## Hardening: the panel, in the right order

The fresh install serves the dashboard over plain HTTP on port 3000. Dokploy's own docs call fixing this crucial, and the **order matters**:

1. Attach a domain to the Dokploy dashboard itself and let it get a Let's Encrypt certificate.
2. Verify the dashboard loads over HTTPS.
3. **Only then** close direct IP access to port 3000:

```bash
docker service update --publish-rm "published=3000,target=3000,mode=host" dokploy
```

Do step 3 before step 1 and you've locked yourself out of your own panel. After that: keep the Traefik dashboard off, don't enable a database's external port unless you truly need it (and firewall it if you do), set Docker log rotation so one noisy container can't fill the disk, and pin image tags instead of `:latest` for anything you care about.

Updates are manual and take thirty seconds — back up first, then:

```bash
curl -sSL https://dokploy.com/install.sh | sh -s update
```

## The honest license and cost math

Dokploy's core is Apache-2.0 — genuinely open source. There is a caveat worth stating plainly: an Enterprise tier exists (SSO, SCIM, custom roles, audit logs) in a source-available proprietary folder, and there's a managed Dokploy Cloud if you ever want them to run the panel. The free self-hosted edition includes per-member permissions and everything in this guide; for a homelab or a side-project fleet, you won't hit the paywall.

The math against the managed alternative: Vercel Pro is roughly $20 per member per month before usage charges, and bandwidth/build minutes scale from there. Dokploy's cost is the VPS — call it $5/month — flat, whether you deploy two apps or twenty, with bandwidth you already paid for. What you give up is also real: no global edge network, no managed DDoS absorption, and the 3 a.m. uptime problem is yours. For side projects, internal tools, and client staging environments, that trade is easy. For a customer-facing storefront taking real traffic, weigh it honestly — plenty of people run Dokploy in production, but they're accepting the on-call bargain knowingly.

## When not to use it

If you already run a hand-rolled Traefik + Compose homelab and enjoy it, Dokploy replaces a workflow that isn't broken — and it can't share the host with your existing Traefik anyway. Its sweet spot is the gap between "I have side projects rotting on expensive PaaS tiers" and "I want the push-to-deploy experience without becoming my own platform team." A dedicated cheap VPS, your Next.js apps and their databases on it, S3 backups on, panel behind HTTPS — that's the whole play.

## Cheat sheet

| Step | Detail |
|---|---|
| Install | `curl -sSL https://dokploy.com/install.sh \| sh` |
| Requirements | Fresh Ubuntu/Debian, ports 80/443/3000 free, 2 GB RAM min (4 GB rec.) |
| Stack | Docker Swarm + Traefik + Postgres + Redis, panel on :3000 |
| Builds | Dockerfile, Nixpacks, or Docker Compose |
| SSL | Automatic Let's Encrypt via Traefik when you add a domain |
| Databases | Postgres/MySQL/MariaDB/MongoDB/Redis, S3 backups |
| Volume backups | Named volumes only — **bind mounts not covered** |
| Harden | Panel domain + HTTPS **first**, then remove published port 3000 |
| Update | `install.sh \| sh -s update` (back up first) |
| License | Apache-2.0 core; Enterprise features (SSO/SCIM) proprietary |
| Cost | One small VPS (~$5/mo class), flat — vs per-seat + usage PaaS pricing |

One command, one cheap box, and the deploy button is yours again.
