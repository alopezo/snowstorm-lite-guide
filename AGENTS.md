# AGENTS.md — playbook for an AI assistant helping install Snowstorm Lite

You are helping a user stand up **Snowstorm Lite** (a SNOMED CT FHIR terminology server)
locally with Docker, using the files in this repo (`docker-compose.yml`,
`snowstorm-lite.env.example`, `README.md`). It works for **any SNOMED CT edition**
(International, national/language extensions, or self-contained editions). Follow this
playbook. The README has the exact commands and copy/paste blocks — use them; this file tells
you *how to drive the process*.

**Guide the user in the language they write to you in.**

## Operating principles
- **Detect the user's OS** (macOS / Linux / Windows) and show only the commands for that OS.
  Don't dump every variant at once.
- **Go one step at a time and verify each** before moving on. Confirm the previous command
  succeeded (exit code, expected output) instead of assuming.
- **Assume the user may be non-technical.** Explain in one line what each command does.
- **Never invent a `version-uri` or `editionId`** for a SNOMED edition. Look it up (Edition
  URI examples linked in the README, or the syndication listing) or derive it from the
  edition/module the user names, and confirm it with them.
- **Secrets:** the user picks their own `ADMIN_PASSWORD`. Don't ask for MLDS credentials
  unless they choose Option A, and never echo passwords back in full. The user may hand you a
  ready-made `snowstorm-lite.env` (handy for screen-shared demos): the stack reads credentials
  from that file, so **don't ask them to reveal the passwords in chat** — it's enough that the
  file exists in the folder.

## Step 0 — Preflight checks (do these first, before anything else)
Run and interpret these; fix whatever fails before continuing.

1. **Docker installed and running:**
   ```bash
   docker version
   ```
   If it errors, the user must install/start Docker Desktop (macOS/Windows) or the Docker
   Engine (Linux) first.

2. **Enough memory for the JVM (critical).** The app runs with `-Xmx4g`, so Docker needs
   ~5–6 GB available or the import gets **OOM-killed**. Check what Docker has:
   ```bash
   docker info --format '{{.MemTotal}}'
   ```
   If it's below ~5000000000 (5 GB), tell the user to raise Docker Desktop's memory
   (Settings → Resources → Memory) before loading a terminology. It's the most common
   real-world failure — check it up front.

3. **Port 8080 free** (or plan to change the host port in the compose if it isn't).

## Step 1 — Get the files and start the stack
Guide the user through the README sections "Get the files", "Configure your env file" and
"Start the server". Key points:
- They must create `snowstorm-lite.env` from the `.example` and set `ADMIN_PASSWORD`.
- Start it with `docker compose up -d`. The `init-permissions` service runs once and fixes
  volume ownership automatically — **don't** tell them to run `docker run` by hand or they'll
  hit `AccessDeniedException`.
- Verify: `docker compose ps` shows `snowstorm-lite` Up, and the logs end with
  *"Snowstorm Lite started. Please load a SNOMED CT package."*

## Step 2 — Choose how to load the terminology (ask the user)
Ask: **"Do you have (A) MLDS credentials, (B) the RF2 files already, or (C) nothing — just
want to test?"** Then follow the matching option in the README. Also ask **which edition**
they want (International, a national/language edition, etc.) — it drives the `version-uri` and
how many files.

> **Heads up:** Snowstorm Lite holds **one edition at a time** — loading/installing another
> **replaces** the previous one. Warn the user before loading anything if they already have an
> edition they want to keep.
- **A — MLDS syndication:** put `SYNDICATION_*` in the env and `docker compose up -d` again to
  apply. Then **ask the user how they want to trigger the load**: (1) via the **dashboard** —
  recommended and simplest; most people prefer this —, or (2) you trigger it via the admin
  API. **By default steer them to the dashboard and just help them open the app:** have them
  open http://localhost:8080, go to **Syndication**, pick the edition they want and click
  install. Use the API only if they explicitly ask. Multi-package editions (edition +
  dependencies) load automatically. (Remember: installing needs admin auth in the browser —
  see Option C.) If you CAN'T drive a browser (you're a CLI agent), say so and offer either:
  give the exact dashboard clicks for them to do (there are few), or trigger the load via the
  admin API (`POST /syndication/install` with `{editionId, version,
  derivativeContentItemVersions:[]}`) and follow progress with
  `GET /syndication/install/{taskId}`. Don't invent the `editionId`/`version`: first list the
  feed's real editions (`GET /syndication/snomed-editions`) and choose with the user. MLDS
  usually offers many candidates (International, language packages such as the SNOMED CT
  Spanish package `450829007`, and separate national editions like Argentina/Uruguay) —
  confirm which one they want before installing.
- **B — Local RF2 files:** upload them with the `load-package` curl. Upload the **whole
  dependency chain in a single call**, with the top edition's `version-uri`. How many files:
  **1** for the International Edition alone; **2** for a common extension (International + the
  extension); **3** when the extension depends on another extension (International +
  intermediate extension + the national extension); or **1** again if it's a self-contained
  **"Edition" (monolith)** package that already bundles its dependencies. `load-package` is
  **synchronous** — the curl blocks until done (a few minutes) and returns HTTP 200.
- **C — Demo feed:** in the dashboard, Settings → Syndication Feed, set the base URL
  `https://snomed-demo-feed.vercel.app` **without** `/feed` (the client appends it), then
  install the IPS Terminology Test edition. Make clear this is only to test the install (IPS
  test data). **The dashboard discovers editions without credentials, but the Install button
  is an admin action**: the browser will prompt for the admin Basic Auth (use the
  `ADMIN_USERNAME`/`ADMIN_PASSWORD` from the env). The dashboard has no login of its own.

## Step 3 — Confirm success
- That a CodeSystem is loaded:
  ```bash
  curl -s "http://localhost:8080/fhir/CodeSystem?_format=json"
  ```
- That a known concept resolves (code 195967001 → "Asthma"):
  ```bash
  curl -s "http://localhost:8080/fhir/CodeSystem/\$lookup?system=http://snomed.info/sct&code=195967001&_format=json"
  ```
  For a language edition, add `&displayLanguage=<lang>` to check terms in that language (e.g.
  `es` → "asma"). Then hand off the FHIR base http://localhost:8080/fhir.

## Diagnostic playbook (symptom → likely cause → action)
| Symptom | Likely cause | Action |
|---------|--------------|--------|
| Container exits during import; exit code 137; `OOMKilled=true` | Not enough memory for `-Xmx4g` | Raise Docker Desktop memory to ≥ 5–6 GB and retry. Check: `docker inspect snowstorm-lite --format '{{.State.OOMKilled}} {{.State.ExitCode}}'` |
| `bind: address already in use` on 8080 | Port taken | Change the host port in the compose (e.g. `"8081:8080"`) and use that port |
| `java.nio.file.AccessDeniedException: /app/lucene-index/data` | Ran `docker run` by hand, skipping the init | Use `docker compose up -d` (fixes volume ownership) |
| Feed 404 / `.../feed/feed` | Entered the feed URL **with** `/feed` | Use the base URL without `/feed` |
| Option B: import fails / concepts missing | Incomplete file set or wrong `version-uri` | Upload the whole dependency chain (e.g. include International with the extension); verify the `version-uri` |
| `401` on `load-package` | Wrong admin credentials | Use the `ADMIN_USERNAME`/`ADMIN_PASSWORD` from the env |
| Option A: listing MLDS editions works but the **ZIP download** returns `401` | `SYNDICATION_PASSWORD` with quotes or a `\` escape in the `.env` (taken literally) | Fix to the literal value (e.g. `*` not `\*`), recreate the container and retry |
| Image pull fails | No internet / proxy | Check connectivity; configure Docker's proxy if there is one |

When diagnosing, always read the logs: `docker compose logs -f snowstorm-lite`.
