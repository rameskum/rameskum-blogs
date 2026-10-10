---
title: 'Replace Rocket Money: Self-Host Wallos and Audit Every Subscription for $0'
date: 2026-10-10T11:00:00-04:00
draft: false
ShowToc: true
description: >-
  The average household bleeds $200+/year on forgotten subscriptions, and the
  apps that find them want your bank password and $4–12/month. Wallos is a
  self-hosted, open-source subscription tracker — one Docker container,
  SQLite, renewal alerts, and spending stats. Working compose, reverse proxy,
  backup playbook, and the $0 math.
tags:
  - self-hosted
  - docker
  - homelab
  - privacy
categories: article
keywords:
  - wallos self hosted
  - subscription tracker docker
  - rocket money alternative
  - self host wallos compose
---

Quick audit: Netflix, Spotify, iCloud, GitHub Copilot, that domain you meant to cancel, the streaming service you opened twice. The average household loses $200+/year to subscriptions it forgot about — and the apps that promise to find them want your bank login and $4–12/month for the privilege.

Wallos ([ellite/wallos](https://github.com/ellite/wallos), GPL-3.0) is the self-hosted answer: a subscription tracker in one Docker container. Manual entry instead of bank scraping — which is a feature, not a limitation: nothing leaves your hardware, and typing in each subscription is exactly the spending-awareness exercise. Dashboard, statistics, renewal notifications, multi-currency, household members. Let's put it on the NAS.

## The compose file

Wallos is refreshingly boring infrastructure-wise: PHP + SQLite, no separate database container, two volumes.

```yaml
services:
  wallos:
    image: bellamy/wallos:latest
    container_name: wallos
    ports:
      - "8282:80"
    environment:
      TZ: America/Toronto
    volumes:
      - ./db:/var/www/html/db
      - ./logos:/var/www/html/images/uploads/logos
    restart: unless-stopped
```

The two volumes are the entire state: `./db` holds the SQLite database (your subscriptions, settings, history), `./logos` holds uploaded provider logos. Lose the container, keep the folders, lose nothing. `TZ` matters because renewal reminders are date-driven — set it to your actual timezone, not the README's Berlin default.

```bash
mkdir -p db logos && docker compose up -d
```

Open `http://<nas-ip>:8282`. First run asks you to create the admin account — there is no default password, which is the correct design.

## First-run setup: 10 minutes that pay for themselves

1. **Create your admin user**, then go to Settings → Household and add members. Splitting "mine / partner / shared" is what makes the per-person stats useful later.
2. **Currencies.** Set your main currency and add the rest. For automatic exchange rates, grab a free [Fixer](https://fixer.io/) API key and paste it in Settings — Wallos converts everything to your main currency for the totals. (Free tier is rate-limited; for a subscription tracker that updates daily, it's plenty.)
3. **Categories.** The defaults are fine, but add the ones that match your life — "Dev tools" and "Domains" if you're us.
4. **Add every subscription.** Name, price, billing cycle, next payment date, payment method, category, member. Yes, all of them — the 20 minutes of data entry is the audit. You'll find at least one "wait, I'm still paying for *that*?" — everyone does.

## Reverse proxy: put it behind Traefik

Exposing port 8282 directly is fine on the LAN, but if you want it on your domain with TLS, add Traefik labels (or your reverse proxy of choice):

```yaml
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.wallos.rule=Host(`wallos.yourdomain.com`)"
      - "traefik.http.routers.wallos.entrypoints=websecure"
      - "traefik.http.routers.wallos.tls.certresolver=letsencrypt"
      - "traefik.http.services.wallos.loadbalancer.server.port=80"
```

Then drop the `ports:` mapping — the container only needs to be reachable by Traefik's network. Same pattern as every other service on the box.

## Notifications: the actual money-saving feature

A tracker you never look at is a spreadsheet. Wallos sends renewal reminders — configure them under Settings → Notifications:

- **Email** via SMTP (your own mail server or a transactional provider).
- Webhook-style integrations for the notifiers Wallos supports.

Set reminders a few days *before* renewal, not on the day — the point is the cancellation window, and annual renewals are where the real money hides. A $149/year renewal you forgot about dwarfs a $9.99/month one you notice.

## Backup and updates: the two-command playbook

Because state is just two folders, backup is a tarball:

```bash
# backup (run nightly via cron)
tar -czf /backup/wallos-$(date +%F).tgz db logos

# update
docker compose pull && docker compose up -d
```

That's the whole disaster recovery plan: re-create the folders from the tarball on any machine, `docker compose up -d`, done. SQLite means no dump/restore ceremony, no database container to version-match. For a finance-adjacent app, test the restore once — untar into a temp dir, point a second compose at it, confirm your subscriptions are there.

## The math

| | Rocket Money / Truebill | Wallos |
|---|---|---|
| Cost | $4–12/month ($48–144/year) | $0 |
| Bank credentials | Required | Never asked |
| Data location | Their servers | Your NAS |
| Entry | Automatic (scraped) | Manual (intentional) |
| Renewal alerts | Yes | Yes |
| Multi-currency | Yes | Yes (Fixer key) |

The manual-entry "downside" is doing real work. Automatic categorization trains you to ignore the app; typing in $16.99/month for something you use twice a year forces a real decision.

Plenty of people report finding $200+/year in forgotten subscriptions once they list everything — your mileage will vary, but the first forgotten annual renewal usually covers the 20 minutes of setup many times over.

## Cheat sheet

| Task | How |
|---|---|
| Deploy | `bellamy/wallos:latest`, port `8282:80`, volumes `./db` + `./logos` |
| Timezone | `TZ: America/Toronto` — reminders are date-driven |
| First run | Create admin (no default password), add household members |
| Multi-currency | Free Fixer API key in Settings |
| Reverse proxy | Traefik labels, drop `ports:`, TLS via your certresolver |
| Renewal alerts | Settings → Notifications, remind *before* the renewal date |
| Backup | `tar -czf` the `db` + `logos` folders nightly |
| Update | `docker compose pull && docker compose up -d` |
| Restore test | Untar to temp dir, second compose stack, verify subscriptions |

Twenty minutes of data entry, one container, zero dollars, zero bank passwords. The subscriptions you forgot about are the ones costing you the most — go find them.
