---
title: 'Ditching Plex Pass: Jellyfin 12 with Hardware Transcoding in Docker'
date: 2026-09-22T09:14:00-04:00
draft: false
ShowToc: true
description: >-
  Jellyfin 12 just shipped (.NET 10, FFmpeg 8.1, a rewritten database) and
  it's the moment to cancel Plex Pass. Full working Docker Compose with
  Intel QuickSync and NVIDIA transcoding, the feature-parity math, and
  the 10.x → 12 upgrade checklist that prevents a broken library.
tags:
  - homelab
  - docker
  - self-hosted
  - jellyfin
  - plex
categories: article
keywords:
  - jellyfin 12 docker
  - plex pass alternative
  - hardware transcoding jellyfin
  - self-hosted media server
---

Plex keeps moving features behind Plex Pass, and this month it finally got personal: hardware transcoding — the thing that lets a $150 mini-PC serve 4K to five devices at once — is paywalled. Meanwhile Jellyfin 12 shipped on September 7th (.NET 10, FFmpeg 8.1, a rewritten database, 12.1 bugfix follow-up on the 14th), and hardware transcoding in Jellyfin is free. It always was.

This is the full migration: the cost math, a working Docker Compose file with hardware transcoding for both Intel and NVIDIA, and the 10.x → 12 upgrade checklist — because 12 is a major migration and doing it wrong eats your library.

## The math

| Feature | Plex (free) | Plex Pass | Jellyfin 12 |
|---|---|---|---|
| Hardware transcoding | ❌ | ✅ ($5/mo or $120 lifetime... when it's on sale) | ✅ free |
| Mobile app downloads / offline sync | ❌ | ✅ | ✅ free |
| Skip intros / credits | ❌ | ✅ | ✅ free (Intro Skipper plugin) |
| Live TV & DVR | ❌ | ✅ | ✅ free |
| Multi-user with parental controls | limited | ✅ | ✅ free |
| Your data on someone else's servers | ✅ | ✅ | ❌ never |

Plex Pass lifetime is $120 on a good day and they've raised it before. Jellyfin is $0 forever, GPL-licensed, no phone-home, no account required. The one honest trade-off: Plex has better native apps on some smart TV platforms. If your household lives on a weird TV OS, check the Jellyfin client for it before you commit — for Android TV, Fire TV, Roku, iOS, and Android, you're covered.

## The Compose file

One file, everything included. This uses the official image; the LinuxServer image works too if you prefer `PUID`/`PGID` env vars.

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:12.1
    container_name: jellyfin
    user: "1000:1000"          # match your media owner's UID:GID
    restart: unless-stopped
    ports:
      - "8096:8096"            # web UI
      # - "8920:8920"          # HTTPS (if not behind a reverse proxy)
      - "7359:7359/udp"        # client auto-discovery
      - "1900:1900/udp"        # DLNA
    volumes:
      - ./config:/config       # server config + database — BACK THIS UP
      - ./cache:/cache         # transcoding cache (disposable)
      - /mnt/media/movies:/media/movies:ro
      - /mnt/media/tv:/media/tv:ro
      - /mnt/media/music:/media/music:ro
    devices:
      - /dev/dri:/dev/dri      # Intel QuickSync / VA-API
    group_add:
      - "109"                  # render group — verify with: getent group render
    environment:
      - JELLYFIN_PublishedServerUrl=http://192.168.1.10:8096
```

```bash
mkdir -p jellyfin/config jellyfin/cache && cd jellyfin
docker compose up -d
# → http://<host>:8096, run the setup wizard, add /media/movies etc. as libraries
```

### NVIDIA instead of Intel

Swap the `devices`/`group_add` block for the NVIDIA container toolkit (install `nvidia-container-toolkit` on the host first):

```yaml
    # devices:            # remove the Intel block
    #   - /dev/dri:/dev/dri
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

Then in both cases: **Dashboard → Playback → Transcoding → Hardware acceleration** → pick **Intel QuickSync (QSV)** or **Nvidia NVENC**, tick **Enable hardware encoding**, save. Play something, drop the quality to force a transcode, and verify the GPU is doing the work:

```bash
# Intel: should show Video/0 busy
intel_gpu_top
# NVIDIA:
nvidia-smi dmon -s u
# If neither moves and your CPU is pegged, transcoding fell back to software —
# recheck the device mapping and the render group.
```

The render-group permission issue is the #1 reason hardware transcoding silently doesn't work in Docker. If `/dev/dri` exists on the host but the container can't use it, confirm the GID: `getent group render` — it's not always 109.

## Migrating from Plex

The honest part: **watch state doesn't transfer.** Played/unplayed status, watch history, and playlists stay in Plex. Everything else — the media files themselves — just works, because Jellyfin reads the same folders.

```bash
# Point Jellyfin at the exact same directories Plex used. Standard naming
# (which Plex also wanted) makes the scan clean:
/mnt/media/movies/Dune Part Two (2024)/Dune Part Two (2024).mkv
/mnt/media/tv/Severance/Season 02/Severance S02E01.mkv
```

After the first scan, install the plugins that close the remaining Plex Pass gaps: **Intro Skipper** (skip intros), **Playback Reporting** (watch stats), **TMDb** (better metadata). Dashboard → Plugins → Catalog.

## Already on Jellyfin 10.x? Read this before touching 12.

Jellyfin 12 is not a routine update. The database schema changes and the migration rewrites data on first boot — **there is no downgrade without a full backup.** The project put the warning in bold and so will I. Checklist, in order:

1. **Stop Jellyfin and take a full manual backup** of your config *and* data directories. This is the only way back.
2. **You must be on 10.10.7 or any 10.11.x** before upgrading to 12.0. Older than that? Upgrade to 10.10.7 first, then 12.
3. **Check your usernames.** Usernames are now case-insensitive — if you have `jellyfin` and `Jellyfin` as two accounts, the migration *fails*. Rename one first.
4. **Uninstall all third-party plugins** before upgrading. Anything compiled for 10.11 will not load on the .NET 10 backend. Re-add them after, from the stable repo.
5. **Reset the plugin repository URL** to `https://repo.jellyfin.org/files/plugin/manifest.json` (Dashboard → Plugins → Repositories) if you ever switched it.
6. **Upgrade, start, and do not interrupt it.** The first boot runs the migrations; the first library scan after that takes significantly longer than normal and some items will appear as newly added. This is expected — 12 clears auto-grouped alternate versions and rebuilds them from disk.
7. **Hard-refresh your browser** (Ctrl+Shift+R) after upgrading — stale cached web assets are the #1 cause of weird UI glitches post-upgrade.

Also note: legacy `/emby/*` and `/mediabrowser/*` routes are gone, and old third-party clients that relied on them will break. If you have scripts hitting the API, the server now reports version `12.0.0` and the OpenAPI output changed — regenerate clients.

## Remote access (2 minutes)

Don't expose 8096 to the internet directly. Behind your existing reverse proxy:

```nginx
server {
    listen 443 ssl;
    server_name jellyfin.example.com;

    location / {
        proxy_pass http://127.0.0.1:8096;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        # websocket support (used by the web client)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Or Tailscale if you'd rather not open ports at all — Jellyfin works fine over a tailnet, and it's the lowest-effort secure remote access for a homelab.

## The verdict

| | Plex + Pass | Jellyfin 12 |
|---|---|---|
| Cost | $120 lifetime (and rising) | $0 |
| Hardware transcoding | paywalled | free, QSV/NVENC/VA-API |
| Privacy | account + telemetry | none, GPL |
| Upgrade risk this month | n/a | real — follow the checklist |
| TV app coverage | best in class | good, verify your TV OS |

Jellyfin 12's upgrade is genuinely disruptive — that's the price of the database rewrite that makes everything faster. But it's a one-time migration for a permanent $0 bill. Cancel the Pass, keep the mini-PC, and put the $120 toward more storage. You'll need it.
