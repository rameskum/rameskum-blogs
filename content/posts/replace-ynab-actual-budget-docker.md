---
title: 'Replace YNAB: Self-Host Actual Budget in One Docker Container'
date: 2026-09-26T08:41:00-04:00
draft: false
ShowToc: true
description: >-
  YNAB charges $14.99/month to tell you where your money went. Actual Budget is
  the envelope-budgeting clone — local-first, self-hosted, free. One-container
  Docker setup, reverse proxy, bank-sync reality check, and the backup playbook
  for data you can't afford to lose.
tags:
  - homelab
  - docker
  - self-hosted
  - finance
  - ynab
categories: article
keywords:
  - actual budget docker
  - self-hosted ynab alternative
  - envelope budgeting self-hosted
  - actual budget bank sync
---

YNAB is $14.99 a month — $180 a year — for envelope budgeting. That's a subscription to be told where your own money went. [Actual Budget](https://actualbudget.org) is the open-source clone: same "give every dollar a job" philosophy, local-first so it works offline, and a tiny server that syncs your budget between devices. One container, zero dollars, your data never leaves your network.

## The math

| | YNAB | Actual Budget (self-hosted) |
|---|---|---|
| Cost | $14.99/mo ($180/yr) | $0 |
| Budgeting method | Envelope | Envelope |
| Your data lives | YNAB's cloud | Your `/data` folder |
| Works offline | Mostly | Yes — local-first |
| Bank sync | Built-in, polished | US/EU only (see below) |
| Mobile apps | Yes | Yes (point at your server) |

The honest trade-off is bank sync — more on that below. Everything else is at parity or better.

## The Compose file

```yaml
services:
  actual:
    image: actualbudget/actual-server:latest   # pin a version (e.g. :26.3.0) once you're happy
    container_name: actual
    restart: unless-stopped
    ports:
      - "5006:5006"
    volumes:
      - ./actual-data:/data      # <-- your money lives here. Back it up.
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:5006/"]
      interval: 30s
      timeout: 10s
      retries: 3
```

```bash
mkdir -p actual/actual-data && cd actual
docker compose up -d
# → http://<host>:5006
```

That's the whole deploy — 256 MB RAM, ~500 MB disk. First visit: set a **server password** (this guards your financial data), then create your first budget file. The desktop and mobile apps connect to the same URL with the same password, and everything syncs through your server.

## Envelope budgeting in 60 seconds

If you've never YNAB'd: every dollar of income gets assigned to a category *before* you spend it — rent, groceries, the homelab fund. Spending draws down the envelope. The software's job is to make the envelopes visible and keep them honest. In Actual: **Budget → create categories → assign this month's income → import transactions → categorize → reconcile.** Do it for one month and the method clicks.

## Bank sync: the honest part

This is where I won't oversell it. Actual has built-in bank sync via **SimpleFIN** (US banks) and **GoCardless** (EU/UK banks). If you're in the US or Europe, it mostly just works. **If you're in Canada — like me — neither covers Canadian banks well**, and this is the one place YNAB's paid integrations are genuinely better.

The dependable path, everywhere: **CSV import.** Every bank on the planet exports CSV or OFX — the format is universal even where API sync isn't. In Actual: account → Import → map the columns once → it remembers the mapping. It takes 90 seconds a month, and frankly, the manual review is *why* envelope budgeting works — auto-import is how mystery spending hides.

Third-party syncers exist for specific regions (EU folks: `enable-actual` bridges Enable Banking; there's also `bankingsync` in Docker), but start with CSV and only automate if the chore actually bothers you.

## Reverse proxy (Nginx)

Don't serve 5006 raw. Behind your existing Nginx:

```nginx
server {
    listen 443 ssl;
    server_name budget.example.com;

    location / {
        proxy_pass http://127.0.0.1:5006;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        # Actual uses websockets for sync
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

The websocket headers matter — sync between devices runs over websockets, and without them the apps silently fall back to misery.

## The backup playbook (non-negotiable)

This is financial data. The entire budget is SQLite files under `./actual-data`. Three layers:

```bash
# 1. The folder is the backup unit — include it in whatever you already run
#    (restic, Unraid appdata backup, rsync to the NAS, all fine):
tar -czf actual-backup-$(date +%F).tar.gz ./actual-data

# 2. In-app export: Settings → Export → downloads a zip of the budget.
#    Do this monthly; it's the format you'd use to migrate anywhere.

# 3. Automate it — a cron line next to the compose file:
0 3 * * * tar -czf /mnt/backups/actual-$(date +\%F).tar.gz -C /opt/actual ./actual-data
```

Test the restore once: stop the container, move `actual-data` aside, untar, start. If you've never restored a backup, you don't have a backup.

## Verdict

| | YNAB | Actual Budget |
|---|---|---|
| Cost | $180/yr forever | $0 |
| Envelope budgeting | The original | Faithful clone |
| Bank sync (Canada) | Works | CSV import (honest) |
| Data custody | Their cloud | Your `/data` folder |
| Setup | Sign up | One compose file |

$180 a year is a lot to pay for a prettier bank importer. If you're in the US/EU with supported banks, Actual is a clean win. If you're in Canada, you're trading 90 seconds of monthly CSV import for $180/year and full custody of your financial data — and the manual import makes you look at your spending, which was the point all along.
