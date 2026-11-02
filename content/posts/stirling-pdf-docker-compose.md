---
title: 'Replace Adobe Acrobat: Stirling-PDF in Docker Compose, With OCR and Signing That Never Leave Home'
date: 2026-10-30T19:10:00+05:30
draft: false
ShowToc: true
description: >-
  Acrobat Pro costs $19.99/month, and every free PDF site asks you to upload your
  tax return to a stranger's server. Stirling-PDF is a self-hosted toolbox with
  50+ tools — merge, split, compress, OCR, sign, redact — running as one Docker
  container on your own hardware. Working compose file, first-login security,
  OCR languages, and the backup rule before you trust it with real documents.
tags:
  - self-hosted
  - docker
  - homelab
  - privacy
categories: article
keywords:
  - stirling pdf docker compose
  - self host pdf tools
  - replace adobe acrobat
---

Think about the last PDF you "just quickly" edited online. A contract, a bank statement, a scanned ID, a tax form. You dragged it onto a website, the site did its thing, and you downloaded the result. That file — with your name, signature, and account numbers in it — passed through a server you don't own, under a privacy policy you didn't read, and you have no way to know whether it was deleted afterwards.

That is the whole pitch for self-hosting your PDF tools. Acrobat Pro is $19.99/month on the annual plan in the US — $239.88 a year, every year — and the free alternatives mostly pay for themselves with your documents. Stirling-PDF does the same everyday jobs on a machine you control, for the cost of the hardware already sitting in your rack.

## What Stirling-PDF is, honestly

Stirling-PDF is a web-based PDF toolbox: merge, split, compress, OCR, sign, redact, rotate, reorder, convert, and more — over 50 tools in one interface, plus a REST API for nearly every tool and no-code pipelines that watch a folder and process whatever lands in it. It is the most-starred PDF application on GitHub (around 95,000 stars as of October 2026) and it runs as a Docker container, a desktop app, or behind Kubernetes.

Two honest caveats before the compose file, because the "open-source" label gets stretched here:

- **It is open-core, not fully open-source.** The repository's LICENSE is MIT for most of the code, but a handful of directories — the processing engine, the SaaS code, and parts of the desktop editor — carry their own proprietary licenses. The everyday tools are free to self-host; enterprise features like SSO and auditing, and the database/S3 file-storage providers, sit behind a paid server plan. The free tier covers up to 5 users, which is plenty for a household.
- **It is not Acrobat feature-for-feature.** You don't get Adobe's cloud, its AI assistant, or its exact rendering engine. What you do get is every tool most people actually open Acrobat for — and the file never leaves your network.

This guide uses version 3.2.2, the latest release as of 24 October 2026, and the official image registry. The older `frooodle/s-pdf` Docker Hub name still gets releases, but the current official image is `docker.stirlingpdf.com/stirlingtools/stirling-pdf`.

## The compose file

Create a directory and save this as `docker-compose.yml`. Every volume here comes from the official docs, and two of them are ones people skip and later regret:

```yaml
services:
  stirling-pdf:
    image: docker.stirlingpdf.com/stirlingtools/stirling-pdf:3.2.2
    container_name: stirling-pdf
    ports:
      - '8080:8080'
    volumes:
      # settings.yml, the built-in database and its backups, generated keys.
      # The docs say it plainly: always mount this.
      - ./stirling-data/configs:/configs
      # Log files.
      - ./stirling-data/logs:/logs
      # Branding overrides and signature files.
      - ./stirling-data/customFiles:/customFiles
      # Watched and finished folders for pipeline folder scanning.
      - ./stirling-data/pipeline:/pipeline
      # Files users save on the server ("My Files").
      # File storage is on by default, so always mount this too.
      - ./stirling-data/storage:/storage
      # Extra OCR language files.
      - ./stirling-data/tessdata:/usr/share/tessdata
    mem_limit: 4g
    restart: unless-stopped
```

Start it:

```bash
docker compose up -d
docker compose logs -f stirling-pdf
```

Then open `http://<your-server>:8080`. The container runs as UID/GID 1000; if your host folders belong to another user, set `PUID` and `PGID` to that user's IDs so the container can actually write to them.

The tag is pinned to `3.2.2` deliberately. The docs say `latest` moves with every release, and a PDF tool that silently upgrades itself underneath your documents is not a feature. When you want a newer version, you change the tag, `docker compose pull`, and `docker compose up -d` — that is the whole update runbook.

## First login: change the default password before anything else

Login is on by default, and the first admin account is public knowledge:

- Username: `admin`
- Password: `stirling`

On first sign-in you are forced to set a new password of at least 8 characters, and the first admin is asked whether to send anonymous usage data. Do the password change immediately, before you expose the service anywhere. If you would rather skip the default account entirely, set your own first admin before the first start:

```yaml
    environment:
      - SECURITY_INITIALLOGIN_USERNAME=your-admin
      - SECURITY_INITIALLOGIN_PASSWORD=use-a-real-password-here
```

These are only used while the database has no users, so they do nothing on an existing install — that is by design, not a bug.

There is also a `SECURITY_ENABLELOGIN=false` option that runs the server with no login at all. The docs warn, in so many words, that anyone who can reach the server can then use it. Treat that setting as LAN-only, behind a reverse proxy you control, and never pair it with a public URL. For a household, the right setup is login on, served behind your existing reverse proxy.

If you already run a reverse proxy, stop publishing the port on every interface. A proxy installed on the host (Caddy, nginx) only needs the port bound to localhost (`'127.0.0.1:8080:8080'`). A proxy running in Docker, like Traefik, doesn't use the published port at all: drop the `ports:` mapping, put the service on the proxy's external network, and point a router at `stirling-pdf:8080` (the [Wallos guide](/posts/wallos-selfhost-subscription-tracker/) walks through that exact label set). Either way, your existing auth layer sits in front. The docs also note an optional sub-path setting (`SYSTEM_ROOTURIPATH`, e.g. `/pdf`) if you want it under a path rather than a subdomain.

## The tools you actually came for

Open the dashboard and the search box is the fastest way in. The daily drivers:

- **Merge and split** — combine several PDFs into one, or split by page ranges, by size, or into single pages. The redesigned page editor in 3.2 also lets you reorder, duplicate, and rename pages visually.
- **Compress** — shrink a PDF, and since 3.2 the tool tells you honestly when it could not shrink a file instead of handing back a larger one and staying quiet.
- **OCR** — turn a scanned, image-only PDF into a searchable one (details below).
- **Sign** — add a signature image or certificate-based signature, and since 3.2 the group-signing flow (Stirling Sign) lets you share a document, track it, and collect signatures from several people. Signature files live in the `customFiles` volume, which is why that mount exists.
- **Redact** — black out text permanently, with a match-case option for auto-redaction. A failed redaction fails loudly rather than silently leaving text behind — the 3.2 release made that explicit.
- **Convert and pipelines** — convert between PDF and office formats, and set up a pipeline that watches a folder: drop a scan in, get an OCR'd, compressed PDF out. This is the homelab move — a watched folder on the NAS that does the boring work for you.

Everything above also has a REST API endpoint, so a script can merge or OCR a file without opening the browser at all. The API docs are linked from the project's documentation site.

## OCR: the languages that ship, and the ones that don't

OCR is the feature most likely to disappoint you on first use, so here is the exact situation. The standard and fat images ship five languages: English, German, French, Portuguese, and Simplified Chinese. That is it. If your scan is in anything else, the OCR tool simply won't offer that language.

Two ways to add more:

1. **In the app:** Settings → Server → Advanced → "Download additional tessdata languages", pick the ones you want, and download. The server fetches them from the `tesseract-ocr/tessdata` repository, so it needs internet access at that moment. Warning: languages downloaded this way live inside the container and are lost when the container is recreated — which is exactly why the compose file mounts the tessdata volume.
2. **By hand:** drop `.traineddata` files into the host folder mounted at `/usr/share/tessdata` and restart. At startup the container copies them into Tesseract's own folder (`/usr/share/tesseract-ocr/5/tessdata`); files already there are not overwritten. Download from `tessdata` for accuracy or `tessdata_fast` for speed, and never delete `eng.traineddata` — English is always needed.

Under the hood, the standard and fat images use OCRmyPDF, with a plain-Tesseract fallback; the extra processing options in the OCR tool only work with OCRmyPDF. If you picked the ultra-lite image (see the variants table below), there is no OCR at all.

## Sizing: give it the memory it asks for

PDF processing is memory-hungry — the docs say a single large PDF can take many times its file size in memory, and the image sizes the Java heap from the container limit up to 70% of it. The official guidance:

| Image variant | Memory | What you give up |
| --- | --- | --- |
| `3.2.2` (standard) | 4 GB | Nothing — this is the one to run |
| `3.2.2-fat` | 6 GB, more with AI on | Nothing, but a much bigger image |
| `3.2.2-ultra-lite` | 2 GB | OCR and login/admin settings |

For a household (1–10 users), the docs suggest 2–4 cores, 4 GB of RAM, and 10 GB of disk. Keep `restart: unless-stopped` set: when a server runs out of memory the container stops and every running job fails, and you want it to come back on its own rather than sit dead until you notice. All three images are built for both `amd64` and `arm64`, so a Raspberry Pi host works too.

## The privacy correction: saved files do live on disk

Here is the claim to get right, because the marketing version is sloppy. "Your files never leave your home" is true — nothing is sent to Stirling's servers or any third party. But it does **not** mean your files are never written to disk. File storage is on by default: anything you explicitly save on the server lives in "My Files", stored as a folder on disk under the `/storage` mount. The docs do not promise that tool uploads are processed only in memory and deleted; they promise they stay on your server.

That distinction is actually good news for a homelab: it means your saved PDFs are ordinary files in `./stirling-data/storage` that you can back up, move, or delete with normal tools. It also means the backup rule matters — which brings us to the part to do before trusting this with real documents.

## The backup rule

Back up `./stirling-data` — the whole directory, not just one subfolder:

- `configs/` holds `settings.yml`, the built-in database and its backups, and generated keys. Lose this and you lose users, settings, and signing keys.
- `storage/` holds every file saved on the server.
- `customFiles/` holds branding and signature files.

The safe sequence is the boring one:

```bash
docker compose down
tar -czf stirling-pdf-backup-$(date +%F).tar.gz stirling-data
docker compose up -d
```

Stopping the container first avoids copying the database mid-write. Keep the backup somewhere other than the same disk as the container — a NAS share you already back up, or a second machine. Test a restore once, on a scratch directory, before the day you actually need it. A backup you have never restored is a hope, not a backup.

## The $0 math

Acrobat Pro, US list price on the annual plan billed monthly: $19.99/month, $239.88/year. Stirling-PDF self-hosted: $0 in software, on hardware you already run, with a free tier covering up to 5 users. Over three years that is roughly $720 you didn't pay Adobe — minus nothing, because the container costs you no license, no per-seat fee, and no per-document charge.

The honest remainder of the math: you pay in your own time (an hour to set up, minutes a month to update and back up), and you don't get Adobe's cloud features, its AI assistant, or its exact rendering. If your work depends on Acrobat-specific features — certified PDF workflows, Adobe's e-signature legal framework, or pixel-identical rendering of complex files — keep Acrobat for that and self-host everything else. For merging, splitting, compressing, OCR'ing, signing, and redacting your own documents, the self-hosted tool covers the job.

## Cheat sheet

```bash
# Start
docker compose up -d

# Logs
docker compose logs -f stirling-pdf

# Update (pin the new version first, then)
docker compose pull && docker compose up -d

# Back up (stop first, then tar, then start)
docker compose down
tar -czf stirling-pdf-backup-$(date +%F).tar.gz stirling-data
docker compose up -d
```

One container, one data directory, fifty tools, and the document you just OCR'd never left your house. That is the whole argument — and it is the next entry in the replace-paid-software run, right after [Wallos took over subscription tracking](/posts/wallos-selfhost-subscription-tracker/).
