---
title: 'Replace Evernote/Notion: Self-Host Joplin Server in Docker Compose, With Encrypted Sync That Never Leaves Home'
date: 2026-11-18T08:30:00-05:00
draft: false
ShowToc: true
description: >-
  Evernote Personal is $14.99 a month and its free tier caps you at 50 notes.
  Notion's free plan is genuinely usable, but your notes live on someone else's
  servers in a format you can't take with you. Joplin is Markdown notes with
  apps on every platform, and Joplin Server is its official sync backend —
  two containers, end-to-end encryption, a web clipper, and nothing leaving
  your network. Working compose file, first-login security, the encryption
  setup that actually works, and the backup rule.
tags:
  - self-hosted
  - docker
  - homelab
  - privacy
categories: article
keywords:
  - joplin server docker compose
  - self host notes
  - replace evernote notion
---

Your notes are the most personal data you own after your photos. Journal entries, meeting notes, passwords you jotted down "temporarily" three years ago, the novel outline, the medical notes. Evernote Personal lists at $14.99 a month ($129.99 a year), and its free tier has been squeezed to 50 notes and one notebook since December 2023. Notion's free plan is genuinely usable for a solo user — unlimited pages and blocks — but your notes live on Notion's servers, in Notion's format, under Notion's pricing decisions.

Joplin takes the other deal: Markdown notes in an open format, free apps on Windows, macOS, Linux, Android, and iOS with no note or device cap, and a sync backend you can run yourself. This guide stands up Joplin Server — the official sync backend — as two Docker containers on your own hardware, turns on end-to-end encryption, connects the web clipper, and proves the phone syncs. It continues the replace-paid-software run alongside [Stirling-PDF](/posts/stirling-pdf-docker-compose/), [Actual Budget](/posts/replace-ynab-actual-budget-docker/), and [Wallos](/posts/wallos-selfhost-subscription-tracker/).

## What Joplin Server is, honestly

Three things to know before the compose file, because the "open-source Evernote replacement" label hides all three:

- **The apps are open-source; the server is not.** The desktop, mobile, and terminal apps are AGPL-3.0. Joplin Server carries its own license, the Joplin Server Personal Use License v1 (21 December 2021), which covers non-commercial use — personal and professional note-taking, sharing, and syncing for yourself. It does not cover running it as a company service: granting access to others for commercial purposes, or granting access at all when you're engaged in commercial activities, needs the paid Joplin Server Business edition. For one person syncing their own notes, including work notes, the free server license fits. For a team, it doesn't — read the license file in `packages/server` before deciding.
- **The server is sync and accounts, not an editor.** There is no note editor in the browser. The admin page manages users and settings; you read and write notes in the apps. If a web editor matters to you, that's Joplin Cloud's web app, not the self-hosted server. What the server adds over syncing to a plain WebDAV folder is better sync performance and notebook sharing.
- **Joplin is not Notion, and doesn't try to be.** No databases, no kanban boards, no real-time collaboration. Two people editing the same shared note at the same time resolve last-sync-wins. What you get instead is Markdown files in an open format, offline-first apps, and sync you control. And no built-in OCR: if Evernote's searchable scans are your workflow, that's the gap.

This guide pins Joplin Server **3.7.6**, released 16 November 2026. Pinning matters right now: Joplin published a full security review of Joplin Server and Joplin Cloud on 2 November 2026 and told self-hosters to update to 3.7.4 or greater. The 3.7.6 changelog carries further security fixes, including not returning database error messages to clients. Don't run `latest` and don't run an old pin.

## The compose file

The file below is the [official sample compose file](https://github.com/laurent22/joplin/blob/dev/docker-compose.server.yml) from the Joplin repository with one change: the optional Transcribe service is removed. Transcribe does handwriting recognition, wants model downloads and up to 16 GB of RAM, and has nothing to do with syncing notes. The official sample gates it behind a `full` profile; this file is the `server` profile only, so no `--profile` flag is needed.

Create a directory and save this as `docker-compose.yml`:

```yaml
services:
  db:
    image: postgres:16
    container_name: joplin-db
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
    restart: unless-stopped
    environment:
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_DB=${POSTGRES_DATABASE}
  app:
    image: joplin/server:3.7.6
    container_name: joplin-server
    depends_on:
      - db
    ports:
      - "22300:22300"
    restart: unless-stopped
    environment:
      - APP_PORT=22300
      - APP_BASE_URL=${APP_BASE_URL}
      - DB_CLIENT=pg
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DATABASE=${POSTGRES_DATABASE}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PORT=5432
      - POSTGRES_HOST=db
```

And a `.env` file next to it with the three values the official sample says to change:

```env
POSTGRES_USER=joplin
POSTGRES_PASSWORD=generate-a-long-one-and-keep-it-in-your-password-manager
POSTGRES_DATABASE=joplin
APP_BASE_URL=http://192.168.1.10:22300
```

Three details that bite people:

- **PostgreSQL is required for real use.** Joplin Server defaults to SQLite if you don't configure a database, and the official README says that plainly: SQLite is for trying it out; production use should connect a database. The `postgres:16` image matches the official sample — don't bump the major version casually, that's a separate migration.
- **Joplin Server will not create the database or user.** The README says so explicitly. That's why the same `POSTGRES_*` values appear in both services: the `postgres` container creates the database and user on first start from those variables, and the `app` container connects to them. If the values differ between the two services, the server fails to start.
- **`APP_BASE_URL` must be the address your clients can actually open.** LAN-only: your server's LAN IP and port, as above. Internet-facing: the HTTPS domain, behind a reverse proxy. If you change the address later, update `APP_BASE_URL` and restart — client sync, redirects, and attachment links all derive from it.

Then start it and watch the first boot:

```bash
docker compose up -d
docker compose logs -f app
```

The server listens on port 22300. Open your `APP_BASE_URL` in a browser and you should get the login page.

## First login: change the admin password, then stop using admin

Every new Joplin Server starts with the same admin account, as the official README states: email `admin@localhost`, password `admin`. That is a known credential on a service that will hold your notes, so the first two minutes are:

1. Log in as `admin@localhost` / `admin`.
2. Change the admin password immediately.
3. Create a normal user account for yourself (Users in the admin panel), and use that account for syncing from your apps. Don't sync as admin — the admin account is for administering the server.

If you expose the server to the internet, put it behind a reverse proxy with HTTPS. One proxy detail from the official README that people miss: Joplin Server reads the client IP from the `X-Real-IP` header for rate limiting and logging, and the proxy must set it from the real client address, replacing anything the client sent — otherwise anyone can pick their own IP and bypass the rate limits. Nginx: `proxy_set_header X-Real-IP $remote_addr;`. Caddy: `header_up X-Real-IP {remote_host}`. Check your proxy passes the real IP; don't assume.

## Connect the apps and prove the sync

On desktop, open Tools → Options → Synchronization, set the target to **Joplin Server**, and enter your server URL (the same `APP_BASE_URL`), your user email, and your user password. Click Synchronise. On mobile it's the same three fields under Configuration → Synchronization.

The test that proves the whole chain, before you migrate anything real:

1. On desktop, create a note titled `sync-test` with a line of text and an attachment.
2. Synchronise on desktop. It should complete with no errors.
3. On your phone, on the same Wi-Fi, open Joplin and synchronise. The note and its attachment should appear.

If the phone can't reach the server, it's almost always one of two things: the phone isn't on the same network as the LAN address in `APP_BASE_URL`, or the sync target is set to Joplin Cloud or WebDAV instead of Joplin Server. Fix the address or the target, not the server.

Your phone needs the server reachable from wherever the phone is. LAN IP works at home; outside the house you need the HTTPS domain (or your VPN). There is no relay: Joplin has no server in the middle to fall back on, which is the point, and also the thing to plan for.

## End-to-end encryption: enable it on one device, and only one

E2EE in Joplin is client-side: the apps encrypt notes before upload, the server stores ciphertext it cannot read, and the other devices decrypt after download. The server takes no part in the encryption process. It is not enabled by default, and the setup has one rule that causes nearly every forum thread on the subject: **enable it on exactly one device first.**

1. On your desktop app, open Tools → Options → Encryption and enable encryption. Joplin creates a master key and asks you for a password. That password protects the master key — it is separate from your Joplin Server account password, and Joplin cannot recover it for you.
2. Synchronise. The encrypted master key uploads with your notes.
3. On your phone, synchronise. The phone receives the master key and asks for the password. Enter the same password. E2EE enables itself on the phone at that point — do not enable encryption separately on the phone, or you'll create a second master key and spend an evening untangling it.
4. Synchronise again on both devices and confirm the "encrypted items" counts settle to zero.

Three honest caveats:

- **E2EE protects your data on the server, not on your devices.** The local Joplin database on your laptop and phone is not encrypted by E2EE. If someone takes your unlocked laptop, E2EE doesn't help — that's what disk encryption is for.
- **Lose the encryption password and the notes are gone.** There is no reset flow. Joplin doesn't hold the unencrypted files anywhere, so nobody can recover them for you. Put the password in your password manager before you enable anything.
- **Encryption is all-or-nothing at the sync layer.** One master key for the vault. There is no per-notebook password in Joplin today; anything claiming otherwise is describing a different tool.

## The Web Clipper: it talks to your desktop app, not the server

The Joplin Web Clipper is a browser extension for Chrome and Firefox that saves full pages, simplified articles, selections, or screenshots as Markdown notes. The part people get wrong is where it sends the clip: not to Joplin Server, but to a small clipper service inside the desktop app, on localhost port 41184. The desktop app must be running, with the clipper service enabled, or the extension reports it can't connect.

1. In the desktop app, open Tools → Options → Web Clipper and enable the clipper service.
2. Install the official Joplin Web Clipper extension from the link on that settings page and authorize it when it asks.
3. Clip a page. It lands in the notebook you choose, in the desktop app — and from there it syncs to your server and your phone like any other note.

The failure mode is always the same: the extension says it can't reach Joplin because the desktop app is closed or the service got turned off, sometimes by an update. Check the service status and the port shown in the Web Clipper settings before reinstalling anything.

## Migrating from Evernote

Joplin imports Evernote `.enex` exports natively — File → Import → ENEX — including attachments, tags, and note metadata. Export notebook by notebook from Evernote (its export caps make one giant file impractical), import each into Joplin on desktop, sync, and check the phone before deleting anything from Evernote. Keep your Evernote account until the import is verified; the export files are your rollback.

From Notion there is no equivalent one-click path. Notion exports Markdown and CSV per page; Joplin imports Markdown files. Expect to rebuild databases and boards by hand, or accept that Joplin replaces the notes half of Notion, not the workspace half.

## The backup rule

Joplin Server has no backup UI, and your notes are worth more than the container running them. Two layers:

- **Database dump.** The notes, tags, and item contents live in PostgreSQL by default (the `STORAGE_DRIVER` variable can move item contents to the filesystem, but the default keeps them in the database). Back up the database, not just the compose file:
  ```bash
  docker compose exec db pg_dump -U joplin joplin > joplin-backup.sql
  ```
  Put that on a schedule and keep the dumps somewhere other than the same disk. The server migrates its database schema on startup when you upgrade, so going back to an older image means restoring a dump, not just changing the tag.
- **Client copies.** Every synced Joplin app holds a full local copy of your notes. That is a genuine second copy, and the desktop app's File → Export → JEX (Joplin Export File) gives you a portable archive of everything. Export one before every major upgrade and before any migration.

Test a restore before you trust the setup with notes you can't afford to lose. A backup you haven't restored is a hope, not a backup.

## What you give up, stated plainly

- No web editor, no real-time collaboration, no databases or boards. Joplin is notes, done well.
- No built-in OCR for scanned documents. Evernote's searchable scans don't carry over.
- No relay, no push sync, no one to call. The server is yours, including its uptime and its updates — and after the November security review, "keep it updated" is not optional advice.
- The license: free for personal use, including your own professional notes. A shared company service needs the Business edition.

What you get is the thing the paid apps can't sell you: Markdown notes, in an open format, synced through a server you own, encrypted with a key only you hold, for the cost of the hardware already running your homelab. Evernote Personal at $129.99 a year and Notion Plus at $120 a year are both fine products. Joplin's argument isn't that they're bad — it's that your notes shouldn't need anyone's permission to exist.

## Verification note

The compose file and environment variables in this post were checked against the official Joplin Server README and sample compose file in the `laurent22/joplin` repository (dev branch, read 18 November 2026), and the server version against the official Joplin Server changelog (3.7.6, 16 November 2026). The license summary was checked against the actual `packages/server/LICENSE.md` text. Evernote and Notion prices are list prices as published in October 2026 and change — check the vendors' pricing pages before you quote them. The stack itself was not run live for this post: the machine this was written on has no Docker, so the compose file is the verified official sample with the Transcribe service removed and versions pinned, not a booted instance. If you run it and anything differs, the logs will tell you first — `docker compose logs app` is always step one.
