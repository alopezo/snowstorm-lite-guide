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

## Before you begin — can you run commands here?
This guide has you (the assistant) run local commands (Docker, curl, edit files). First
decide whether this environment actually lets you execute local shell commands and reach the
user's machine.
- **If you can** (you're a coding agent with shell access — Claude Code, Codex, or similar,
  via a CLI, a desktop app or an IDE extension): confirm with `docker version` and continue
  with Step 0.
- **If you can't** (you're a plain chat with no command/tool execution): you cannot run
  anything on the user's machine — don't pretend to. Tell the user and offer two paths:
  1. **You guide, they run:** you provide each command and the user pastes it into their own
     terminal, reporting back the output.
  2. **Hands-off (recommended):** the user opens this in a coding agent that can run commands
     (Claude Code, Codex, or similar) and pastes the same starter prompt there, so the
     assistant runs everything itself.

## Ask what kind of help they want
Before doing anything, ask the user how hands-on they want you to be, and then stick to that
level. Offer these four:
1. **Do everything, end to end** — you run all the commands *and* load the terminology
   yourself, handing over a working, populated server. (Needs command execution.)
2. **Set it up and hand off at the dashboard** — you install and start the stack, then stop at
   http://localhost:8080 and let the user load the terminology themselves (e.g. pick the
   edition under Syndication). (Needs command execution.)
3. **Guide only — the user runs the commands** — you explain and give each command; the user
   types it on their own machine and reports back the output. (Works in a plain chat too.)
4. **Troubleshooting only** — the user already installed/attempted; you just help diagnose and
   fix problems. Jump to the diagnostic playbook and ask what they're seeing.

Notes:
- If you can't run commands (plain chat), only 3 and 4 are possible — say so and default to 3.
- The chosen level governs how you run the steps below: in levels 2–4 **don't silently run
  steps the user wanted to do themselves** — in level 2 stop before loading the terminology;
  in level 3 hand each command to the user instead of running it; in level 4 skip setup unless
  asked.
- You can always switch levels if the user changes their mind.

## Learning mode (optional) — teach while installing
Some users just want the server running; others want to understand it and need less help next
time. Treat these as **two independent dimensions**, and **infer them — don't run a
questionnaire**:

| Dimension | Range |
|-----------|-------|
| **Who performs the actions** | you (agent) · shared · the user |
| **How much learning support** | minimal (default) · explanatory · apprenticeship |

They combine freely: *"you run it, but explain as you go"* = agent acts + explanatory;
*"I'll type everything myself and want to understand it"* = user acts + apprenticeship.

**Default to minimal** (get it working efficiently). Move toward explanatory/apprenticeship
when the user asks to learn, says "explain what you're doing", keeps asking "why?", or picks
the guide-only level. Drop back when they turn task-focused ("just fix it").

In learning mode use these as **behaviors during the real installation** — never as separate
lessons, quizzes or exercises:
- **Model** — make invisible expert reasoning visible: what you're checking, what success
  looks like, what failure would imply. (`docker version` isn't "is it installed" but "does
  the daemon answer?")
- **Coach** — interpret the actual output, point at the signal that matters, correct
  misconceptions as they surface.
- **Scaffold** — vary support: exact command + why → objective + likely command → objective
  only → "your call". Never withhold help from someone who is stuck.
- **Fade** — as they succeed at similar steps, say less. First time: *"Run
  `docker compose ps`; look for `snowstorm-lite` with status `Up`."* Later: *"Let's check the
  stack is healthy — what would you run?"* Later still: *"Verify the deployment."* If they
  struggle, scaffold back up. Adapt to evidence, not to a step count.
- **Articulate** — occasionally (not every step) ask them to reason: *"what do you think exit
  code 137 points at?"* Conversational, not a quiz.
- **Reflect** — at milestones, summarize what exists now and what doesn't: *"Docker is running
  the container with persistent storage on port 8080 — the terminology itself isn't loaded
  yet."*
- **Explore** — once it works, offer optional variations: another `$lookup`, a term in another
  language, inspect the CodeSystem, stop/restart the stack, find where the data actually
  lives, reason about what installing a second edition would do. Always optional, never a
  prerequisite.

### The mental model to build (gradually)
Beginners collapse these layers into one, which is exactly why they can't localize a failure.
Introduce each layer as it becomes relevant, not all at once:

```
Host computer → Docker → Snowstorm Lite container → persistent index (volume)
    → loaded SNOMED CT edition → FHIR terminology API
```
And when syndication is used: `MLDS feed → RF2 packages → Snowstorm Lite`.

### Diagnosing as a teachable skill
In learning mode don't jump straight to the fix — make the reasoning visible, then fix:
observe the symptom → **which layer is this?** → likely causes → run one discriminating check
→ interpret the evidence → apply the fix → verify.
Example: the container died mid-import → application or Docker? → `docker inspect … OOMKilled`
→ `true` → a Docker resource limit, not a terminology problem → raise memory → retry and
verify. The point is a reusable method, not a memorized fix.

### Don't create pedagogical friction
- Don't ask a question after every command, and don't re-explain what they've already shown
  they understand.
- **Never delay a fix for teaching purposes** — fix it, then explain.
- If they ask "why?", explain more. If they say "just do it", teach less.
- A working installation is always the primary objective.

## Step 0 — Preflight checks (do these first, before anything else)
Run and interpret these; fix whatever fails before continuing.

1. **Docker installed and running:**
   ```bash
   docker version
   ```
   If it errors, the user must install/start Docker Desktop (macOS/Windows) or the Docker
   Engine (Linux) first.

2. **Enough memory for the JVM.** The app caps the JVM at `-Xmx4g`, which already leaves
   headroom, so **about 4 GB given to Docker is normally enough**. Check what Docker has:
   ```bash
   docker info --format '{{.MemTotal}}'
   ```
   If it's well below ~4000000000 (4 GB), tell the user to raise Docker Desktop's memory
   (Settings → Resources → Memory) before loading a terminology. Don't over-demand memory up
   front — if an import does get **OOM-killed** later, that's a good teachable moment: walk
   the diagnosis (see the diagnostic playbook) and then raise the limit.

3. **Port 8080 free** (or plan to change the host port in the compose if it isn't).

4. **MLDS credentials, if there are any (do this whenever `SYNDICATION_*` is set in the env
   — whatever loading option they end up choosing).** Catch a bad password now, not halfway
   through a download or on their first click in the dashboard.

   > ⚠️ Listing editions does **not** validate credentials — the feed listing answers even
   > with a wrong password; only the package download is authenticated. Check a download URL
   > with `HEAD`, which authenticates without downloading anything:

   ```bash
   URL=$(curl -s -u "$SYNDICATION_USERNAME:$SYNDICATION_PASSWORD" \
         https://mlds.ihtsdotools.org/api/feed | grep -oE 'https://[^"]*/download' | head -1)
   curl -s -o /dev/null -I -w '%{http_code}\n' \
         -u "$SYNDICATION_USERNAME:$SYNDICATION_PASSWORD" "$URL"
   # 200 = credentials work · 401 = they don't
   ```
   On **401**, fix it before going further: the value must be **literal** in the `.env` (no
   quotes, no `\` escapes), then `docker compose up -d` to re-read it. Skip this check
   entirely if they aren't using MLDS.

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
want to test?"** Then follow the matching option in the README.

> **Heads up:** Snowstorm Lite holds **one edition at a time** — loading/installing another
> **replaces** the previous one. Warn the user before loading anything if they already have an
> edition they want to keep.

- **A — MLDS syndication.** Put `SYNDICATION_*` in the env, `docker compose up -d` again to
  apply, and **verify the credentials (Step 0.4) before any download**. Then offer two ways to
  finish — ask which they prefer:

  - **A.1 — You load it now.** First ask **which edition** they want, and never guess: list
    the feed's real editions (`GET /syndication/snomed-editions`) and choose together. MLDS
    offers many candidates — the International Edition, language packages such as the SNOMED
    CT Spanish package (`450829007`), and separate national editions like Argentina or
    Uruguay. Then trigger it yourself: `POST /syndication/install` with
    `{editionId, version, derivativeContentItemVersions: []}`, and follow
    `GET /syndication/install/{taskId}` until `COMPLETED` (a few minutes; multi-package
    editions pull their dependencies automatically). Hand over a populated server.
  - **A.2 — They load it in the dashboard.** Point them at http://localhost:8080 →
    **Syndication**, where they pick the edition and click install. This is a perfectly good
    ending: the server is ready and, because you checked the credentials, their first click
    will work. Mention that installing prompts for the admin Basic Auth, and that it replaces
    any edition already loaded. Don't click through it for them unless they ask — this is the
    natural landing for help level 2.

  If you can't drive a browser (you're a CLI agent) and they still want A.2, give them the
  exact clicks rather than doing it for them.
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

**Finishing with nothing loaded is always allowed**, whatever they picked — the server runs
fine empty and they can load content later. Just say plainly what's missing to get content in
(credentials verified and an edition to choose, files to upload, or the demo feed configured).

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

## Step 4 — Hand off (offer, don't lecture)
When it works, close the loop in a few lines: say **what they now have** (server running,
which edition is loaded — or none yet, dashboard at http://localhost:8080, FHIR base at
`/fhir`). Then **offer a short menu and let them choose** — most people want to poke at it
themselves:
- **Explore concepts** → *SNOMED Mini Browser* in the dashboard (no credentials needed — the
  easiest first thing to try).
- **Build a ValueSet** → *FHIR Resources → ValueSet → Add ValueSet*, with the ECL builder
  (concept typeahead, operators, expansion preview).
- **Query it from code** → the FHIR base, `$lookup`, `$expand`, `$validate-code`, `$subsumes`.
- **Load a different edition** → *Syndication* (⚠️ replaces the current one).

Ask **"want me to walk you through any of it, or would you rather explore?"** and only guide
if they say yes. Don't tour the dashboard unprompted.

> Worth saying once, because it trips everyone: in the dashboard **reading works without
> credentials, but writing is an admin action** — installing an edition, saving a ValueSet or
> changing Settings all trigger the browser's Basic Auth prompt (`ADMIN_USERNAME` /
> `ADMIN_PASSWORD`). The dashboard has no login of its own.

## Diagnostic playbook (symptom → likely cause → action)
| Symptom | Likely cause | Action |
|---------|--------------|--------|
| Container exits during import; exit code 137; `OOMKilled=true` | Not enough memory for `-Xmx4g` | Raise Docker Desktop memory (4 GB is usually enough; give it more) and retry. Check: `docker inspect snowstorm-lite --format '{{.State.OOMKilled}} {{.State.ExitCode}}'` |
| `bind: address already in use` on 8080 | Port taken | Change the host port in the compose (e.g. `"8081:8080"`) and use that port |
| `java.nio.file.AccessDeniedException: /app/lucene-index/data` | Ran `docker run` by hand, skipping the init | Use `docker compose up -d` (fixes volume ownership) |
| Feed 404 / `.../feed/feed` | Entered the feed URL **with** `/feed` | Use the base URL without `/feed` |
| Option B: import fails / concepts missing | Incomplete file set or wrong `version-uri` | Upload the whole dependency chain (e.g. include International with the extension); verify the `version-uri` |
| `401` on `load-package` | Wrong admin credentials | Use the `ADMIN_USERNAME`/`ADMIN_PASSWORD` from the env |
| Option A: listing MLDS editions works but the **ZIP download** returns `401` | `SYNDICATION_PASSWORD` with quotes or a `\` escape in the `.env` (taken literally) | Fix to the literal value (e.g. `*` not `\*`), recreate the container and retry |
| A **filtered** `$expand` returns fewer rows than `count` (only when `offset < 100`) | The relevance-sort window (250) caps a page at `window − offset`; `total` still shows the real count | This compose ships `--search.valueset-expand.relevance-sort-window=10000`, which removes the cap — check the flag wasn't dropped; on the default (250) keep `count` within `250 − offset` |
| Image pull fails | No internet / proxy | Check connectivity; configure Docker's proxy if there is one |

When diagnosing, always read the logs: `docker compose logs -f snowstorm-lite`.
