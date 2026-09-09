# Install guide — Snowstorm Lite

A ready-to-use guide and `docker-compose.yml` for running [Snowstorm Lite](https://github.com/IHTSDO/snowstorm-lite)
(a SNOMED CT FHIR terminology server) that **works on the first `docker compose up`** — no
manual `chown`, no first-run crash. It works for **any SNOMED CT edition** (International,
national/language extensions, or self-contained editions).

> 🌐 Spanish-language version of this guide (oriented to the Spanish edition): branch
> [`main`](https://github.com/alopezo/snowstorm-lite-guide/tree/main).

> **Got an AI agent? Start here.** Copy this and hand it to your agent (Claude Code, etc.).
> It will read the instructions and **ask you what it needs** (your OS, whether you have MLDS
> credentials, files, or nothing) and guide you step by step:
>
> ```
> Help me install Snowstorm Lite step by step.
> Clone/read this repo (english branch), follow AGENTS.md, run the preflight
> checks and ask me what you need (OS, MLDS credentials / files / nothing):
> https://github.com/alopezo/snowstorm-lite-guide/tree/english
> ```
>
> Prefer to do it by hand? Follow the rest of this README.

## Files
| File | What it is |
|------|------------|
| `docker-compose.yml` | The stack: an init step that fixes volume permissions + the app. |
| `snowstorm-lite.env.example` | Template for your env file. Copy it to `snowstorm-lite.env`. |
| `AGENTS.md` | Playbook for an AI assistant guiding a user through this setup. |
| `.gitignore` | Keeps your real env and the zips from being committed by accident. |
| `README.md` | This file. |

## Prerequisites
- **Docker** installed and running (Docker Desktop on macOS/Windows, Docker Engine on
  Linux). Check with `docker version`.
- **Memory:** the server runs the JVM with `-Xmx4g`, so give Docker **~5–6 GB** or more
  (Docker Desktop → Settings → Resources → Memory). With less, loading a terminology gets
  **OOM-killed** mid-import. This is the most common failure — set it before you start.
- **Disk:** a few GB free (Lucene index + any downloaded RF2 zips).
- **Port 8080** free (or change the host port in the compose).
- **Internet** to pull the image (~540 MB) and, for Options A/C, to reach the feed.

## 1. Get the files
Clone the repo (this `english` branch):
```bash
git clone -b english https://github.com/alopezo/snowstorm-lite-guide.git
cd snowstorm-lite-guide
```
No `git`? Download the ZIP from the repo page (**Code → Download ZIP**) and unzip it, or grab
individual files by their *raw* URL, e.g.:
```bash
curl -L -O https://raw.githubusercontent.com/alopezo/snowstorm-lite-guide/english/docker-compose.yml
curl -L -O https://raw.githubusercontent.com/alopezo/snowstorm-lite-guide/english/snowstorm-lite.env.example
```
(On Windows PowerShell use `curl.exe`.)

## 2. Configure your env file
```bash
cp snowstorm-lite.env.example snowstorm-lite.env   # then edit it
```

| Key | Required | Purpose |
|-----|----------|---------|
| `ADMIN_USERNAME` | optional (defaults to `admin`) | Dashboard / admin API login |
| `ADMIN_PASSWORD` | **yes** | Admin password — set your own |
| `SYNDICATION_USERNAME` | optional | Only for MLDS auto-download (Option A) |
| `SYNDICATION_PASSWORD` | optional | Only for MLDS auto-download (Option A) |

> **Passwords in demos (screen sharing).** Prepare your `snowstorm-lite.env` yourself,
> ahead of time and privately. Because the stack reads credentials from that file
> (`env_file`), during a Zoom demo you **don't need to type or show the MLDS password on
> screen** — it lives only in the file; keep it closed while sharing your screen. If you're
> using an AI assistant, you can hand it the ready-made `.env`: no need to reveal the
> passwords in chat.
>
> Caveat: if you'll **install via the dashboard** (Option A/C) and your `ADMIN_PASSWORD` is
> not empty, the browser will prompt for it once (Basic Auth). To avoid showing it in the
> demo, authenticate before sharing your screen, or use the API path (it reads the password
> from the file, nothing is typed on screen).

## 3. Start the server
```bash
docker compose up -d
```
Open the dashboard at **http://localhost:8080** and load a terminology (below).
Follow the logs with: `docker compose logs -f snowstorm-lite`

---

## Why the init step exists
The published image runs as a non-root user (uid `1000`, container hardening). A fresh
Docker named volume is created owned by `root`, so the app can't create its Lucene index and
crashes on first start with:

```
java.nio.file.AccessDeniedException: /app/lucene-index/data
```

The `init-permissions` service runs once and `chown`s the volume to `1000:1000` **before**
the app starts, so everything works with no manual steps.

---

## Loading a terminology

There are **three ways** to load content. Pick the one that matches what you have:

| You have… | Use | Credentials? |
|-----------|-----|--------------|
| MLDS credentials | **Option A** — official syndication (real SNOMED CT) | yes |
| the RF2 files | **Option B** — upload the local files directly | no |
| nothing (just testing) | **Option C** — free demo feed (IPS test data) | no |

> **Important:** Snowstorm Lite holds **one SNOMED CT edition at a time**. Loading or
> installing another edition **replaces** whatever was loaded (the index is overwritten).

### Option A — Official syndication with MLDS credentials
For downloading a real SNOMED CT edition. Put your MLDS credentials in the env file:
```
SYNDICATION_USERNAME=your-mlds-username
SYNDICATION_PASSWORD=your-mlds-password
```
Then, in the dashboard choose **Syndication** and select the edition you want. Multi-package
editions (e.g. a national or language edition = International + extension) are downloaded and
loaded **automatically**.

> MLDS lists many editions: the **International Edition**, language packages (e.g. the
> **SNOMED CT Spanish package**, module `450829007`), and national editions (e.g. Argentina,
> Uruguay). Pick the one you need.

> Literal value in the env: if your `SYNDICATION_PASSWORD` contains characters like `*` or
> `$`, write it as-is (no quotes, no `\`). One stray escape causes a `401` when downloading
> the ZIP (see Troubleshooting).

Downloaded RF2 zips are cached **inside the volume** at `/app/lucene-index/rf2-cache` (see
the `--syndication.rf2-download-cache-directory` flag in the compose), so they persist across
restarts and are reused on the next load.

> Without that flag the default cache dir can't be created (`/app` is root-owned, the app
> runs as uid 1000), so syndication downloads would **not** be kept.

To copy the cached zips out of the volume and reuse them:
```bash
docker cp snowstorm-lite:/app/lucene-index/rf2-cache ./rf2-cache
```

**Alternative: trigger the load via API (no dashboard).** Useful for automation. First list
the feed's real editions and pick the right one (don't guess the `editionId`):
```bash
curl -s -u admin:YOUR_ADMIN_PASSWORD http://localhost:8080/syndication/snomed-editions
```
Then install it (example: the International Edition, module `900000000000207008`):
```bash
curl -s -u admin:YOUR_ADMIN_PASSWORD -H 'Content-Type: application/json' \
  -X POST http://localhost:8080/syndication/install \
  -d '{"editionId":"http://snomed.info/sct/900000000000207008","version":"20260701","derivativeContentItemVersions":[]}'
```
The response includes a `taskId`. Monitor progress:
```bash
curl -s -u admin:YOUR_ADMIN_PASSWORD http://localhost:8080/syndication/install/TASK_ID
```

### Option B — Upload local RF2 files directly
If someone gave you the `.zip` file(s), upload them via the admin API — no syndication
needed. Upload **every package the edition needs, in a single call**, with the edition's
`version-uri`.

**How many files?** It depends on the dependency chain — upload the target package plus
everything it depends on, all together:

| Case | Files to upload |
|------|-----------------|
| **International Edition** (self-contained) | **1** — just the International package |
| A **common extension** (depends only on International) | **2** — International + the extension |
| An extension that depends on **another extension** (e.g. a national extension of the Spanish edition) | **3** — International + the intermediate extension + the national extension |
| A self-contained **"Edition" (monolith) package** that already bundles its dependencies | **1** — just that package |

The `version-uri` is always that of the top edition you're loading.

**Example — International Edition (1 file):**
```bash
curl -u admin:YOUR_ADMIN_PASSWORD \
  --form file=@SnomedCT_InternationalRF2_PRODUCTION_20260701T120000Z.zip \
  --form version-uri="http://snomed.info/sct/900000000000207008/version/20260701" \
  http://localhost:8080/fhir-admin/load-package
```

**Example — an extension (2 files), here the Spanish edition:**
```bash
curl -u admin:YOUR_ADMIN_PASSWORD \
  --form file=@SnomedCT_InternationalRF2_PRODUCTION_20260701T120000Z.zip \
  --form file=@SnomedCT_SpanishRelease-es_PRODUCTION_20260810T120000Z.zip \
  --form version-uri="http://snomed.info/sct/450829007/version/20260810" \
  http://localhost:8080/fhir-admin/load-package
```
The International package provides the base content; the extension requires it — that's why
both go in the **same** request. `load-package` is **synchronous**: the command blocks until
the import finishes (a few minutes) and only then returns HTTP 200. Follow progress with
`docker compose logs -f snowstorm-lite`.

- On **Windows (PowerShell)** use `curl.exe` and put the whole call on one line.
- Adjust the `.zip` names to yours and the `version-uri` to the version you're loading.
- `version-uri` for each edition: see the
  [Edition URI examples](https://github.com/IHTSDO/snowstorm-lite/blob/master/docs/snomed-edition-uri-examples.md).

### Option C — Free demo feed (no credentials, just to test the install)
If you have no MLDS account and no files, you can point Snowstorm Lite at a public demo
syndication feed that serves a small, freely-distributable test package: **IPS (International
Patient Summary)**. It's for checking that the install works — just test data.

1. In the dashboard go to **Settings → Syndication Feed**.
2. Replace the feed URL with the demo feed's **base URL**:
   ```
   https://snomed-demo-feed.vercel.app
   ```
   > ⚠️ Enter the base URL **without** `/feed`. Snowstorm Lite appends `/feed` itself (it
   > does `rootUri(base)` then requests `/feed`). If you type `.../feed`, it will request
   > `.../feed/feed` and get a 404.
3. Refresh, open the **Syndication** menu, and install the **IPS Terminology Test** edition.

> **Note on authentication:** the dashboard **discovers** editions without credentials, but
> **installing** is an admin action — the browser will prompt for the admin
> username/password (Basic Auth). Use the `ADMIN_USERNAME`/`ADMIN_PASSWORD` from the env.

Demo feed source: <https://github.com/alopezo/snomed-demo-feed>. For testing/evaluation only
— not production content. Installing the IPS **replaces** whatever edition you had loaded
(see the note above).

## Verify it loaded
List the loaded CodeSystem(s):
```bash
curl -s "http://localhost:8080/fhir/CodeSystem?_format=json"
```
Confirm a known concept resolves — code `195967001` should display **"Asthma"**:
```bash
curl -s "http://localhost:8080/fhir/CodeSystem/\$lookup?system=http://snomed.info/sct&code=195967001&_format=json"
```
For a **language edition**, add `displayLanguage` to get terms in that language, e.g. Spanish
(→ **"asma"**):
```bash
curl -s "http://localhost:8080/fhir/CodeSystem/\$lookup?system=http://snomed.info/sct&code=195967001&displayLanguage=es&_format=json"
```
With `jq` (not installed by default on macOS) you can extract just the term:
```bash
curl -s "http://localhost:8080/fhir/CodeSystem/\$lookup?system=http://snomed.info/sct&code=195967001&_format=json" \
  | jq -r '.parameter[] | select(.name=="display") | .valueString'
```
The FHIR interface is at http://localhost:8080/fhir

## Managing the stack
```bash
docker compose logs -f snowstorm-lite   # follow logs
docker compose stop                      # stop (keeps data)
docker compose start                     # start again
docker compose down                      # remove containers (keeps the volume/data)
docker compose down -v                   # remove containers AND wipe the loaded index
docker compose pull && docker compose up -d   # re-pull the pinned image (:2.5.2)
docker compose down -v --rmi all --remove-orphans   # FULL cleanup: containers, volume (index + zips) and image
```
The last command resets everything (useful for test cycles): the next `up` re-pulls the full
image.

## Troubleshooting
| Symptom | Cause | Fix |
|---------|-------|-----|
| Container exits mid-import; exit code 137 | Not enough memory for `-Xmx4g` | Raise Docker memory to ≥ 5–6 GB and retry. Check: `docker inspect snowstorm-lite --format '{{.State.OOMKilled}} {{.State.ExitCode}}'` |
| `bind: address already in use` (8080) | Port taken | Change the host port in the compose, e.g. `"8081:8080"` |
| `AccessDeniedException: /app/lucene-index/data` | Started with `docker run` by hand, skipping the init step | Use `docker compose up -d` (it fixes volume ownership) |
| Feed 404 / `.../feed/feed` | Feed URL entered **with** `/feed` | Use the base URL **without** `/feed` |
| Option B: import fails or concepts missing | Incomplete file set / wrong `version-uri` | Upload the whole dependency chain (e.g. include International with any extension); check the `version-uri` |
| `401` on `load-package` | Wrong admin credentials | Use the `ADMIN_USERNAME`/`ADMIN_PASSWORD` from your env |
| Option A: `401` when **downloading the ZIP** from MLDS (even though listing editions works) | `SYNDICATION_PASSWORD` mis-written in the `.env` (quotes or `\` escape) | Use the **literal** value, no quotes or backslashes; recreate with `docker compose up -d` |

## Notes
- Keep your real `snowstorm-lite.env` private — it holds credentials. Share only the
  `.example`.
- The image is **pinned to `:2.5.2`** in the compose (reproducible for sharing). To update to
  a newer version, change the tag in both `image:` lines of `docker-compose.yml` and recreate
  with `docker compose up -d`. Available versions are on
  [Docker Hub](https://hub.docker.com/r/snomedinternational/snowstorm-lite/tags) and match
  the [repo releases](https://github.com/IHTSDO/snowstorm-lite/releases).
- **Persistence inside the volume.** Because the image runs as uid 1000 but `/app` is
  root-owned, several things the server would write to `/app` fail with `AccessDenied`. That's
  why the compose redirects two of them to the volume (which is writable), so they **survive
  restarts**:
  - `--syndication.rf2-download-cache-directory=lucene-index/rf2-cache` — cache of RF2 zips
    downloaded via syndication.
  - `--syndication.feed-config-file=lucene-index/syndication-feed-config.properties` — the
    feed URL/credentials saved from Settings.
  Without these flags, those downloads and the feed config would be **lost** on every restart.
