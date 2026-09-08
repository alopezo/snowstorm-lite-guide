# AGENTS.md — guía para un asistente de IA que ayuda a instalar Snowstorm Lite (edición español)

Estás ayudando a un usuario a levantar **Snowstorm Lite** (servidor FHIR de terminología
SNOMED CT) localmente con Docker, usando los archivos de este repo (`docker-compose.yml`,
`snowstorm-lite.env.example`, `README.md`). Este repo está orientado a la **edición en
español**. Seguí este playbook. El README tiene los comandos exactos y los bloques para
copiar/pegar — usalos; este archivo te dice *cómo conducir el proceso*.

**Guiá al usuario en español.**

## Principios de trabajo
- **Detectá el sistema operativo** del usuario (macOS / Linux / Windows) y mostrá solo los
  comandos de ese SO. No vuelques todas las variantes juntas.
- **Andá de a un paso y verificá cada uno** antes de seguir. Confirmá que el comando previo
  funcionó (código de salida, salida esperada) en vez de asumirlo.
- **Asumí que el usuario puede no ser técnico.** Explicá en una línea qué hace cada comando.
- **Nunca inventes un `version-uri`** de una edición SNOMED. Buscalo (ver los ejemplos de
  Edition URI enlazados en el README) o derivalo de la edición/módulo que nombre el usuario,
  y confirmalo con él.
- **Secretos:** el usuario elige su propio `ADMIN_PASSWORD`. No pidas credenciales MLDS a
  menos que elija la Opción A, y nunca repitas contraseñas completas de vuelta.

## Paso 0 — Chequeos previos (hacelos primero, antes que nada)
Corré e interpretá esto; resolvé lo que falle antes de continuar.

1. **Docker instalado y corriendo:**
   ```bash
   docker version
   ```
   Si da error, el usuario debe instalar/arrancar Docker Desktop (macOS/Windows) o el
   Docker Engine (Linux) primero.

2. **Memoria suficiente para la JVM (crítico).** La app corre con `-Xmx4g`, así que Docker
   necesita ~5–6 GB disponibles o el import muere por **OOM**. Chequeá lo que tiene Docker:
   ```bash
   docker info --format '{{.MemTotal}}'
   ```
   Si es menor a ~5000000000 (5 GB), decile al usuario que suba la memoria de Docker Desktop
   (Settings → Resources → Memory) antes de cargar la terminología. Es la falla real más
   común — chequeala de entrada.

3. **Puerto 8080 libre** (o planificá cambiar el puerto del host en el compose si no lo está).

## Paso 1 — Obtener los archivos y levantar el stack
Guiá al usuario por las secciones del README "Descargá estos archivos", "Configurá tu
archivo de entorno" y "Levantá el servidor". Puntos clave:
- Debe crear `snowstorm-lite.env` a partir del `.example` y poner `ADMIN_PASSWORD`.
- Se arranca con `docker compose up -d`. El servicio `init-permissions` corre una vez y
  corrige el dueño del volumen automáticamente — **no** le digas que corra `docker run` a
  mano o va a chocar con `AccessDeniedException`.
- Verificá: `docker compose ps` muestra `snowstorm-lite` Up, y los logs terminan con
  *"Snowstorm Lite started. Please load a SNOMED CT package."*

## Paso 2 — Elegir cómo cargar la terminología (preguntale al usuario)
Preguntá: **"¿Tenés (A) credenciales MLDS, (B) los archivos RF2 ya descargados, o
(C) nada — solo querés probar?"** Después seguí la opción correspondiente del README.

> **Ojo:** Snowstorm Lite mantiene **una sola edición a la vez** — cargar/instalar otra
> **reemplaza** la anterior. Avisá al usuario antes de cargar algo si ya tiene una edición
> que quiere conservar.
- **A — Sindicación MLDS:** poné `SYNDICATION_*` en el env y `docker compose up -d` de nuevo
  para aplicar. Después **preguntale al usuario cómo quiere disparar la carga**:
  (1) por el **dashboard** — recomendado y lo más simple; la mayoría va a preferir esto —, o
  (2) que vos la dispares por la API admin. **Por defecto llevalo al dashboard y limitate a
  acompañarlo a abrir la app:** que abra http://localhost:8080, entre a **Syndication**,
  elija la edición en español y le dé instalar. Usá la API solo si te lo pide expresamente.
  Las ediciones multi-paquete (español = International + extensión) se cargan
  automáticamente. (Recordá: instalar necesita auth admin en el navegador — ver Opción C.)
  Si NO podés manejar el navegador (sos un agente de CLI), decíselo al usuario y ofrecele:
  o le pasás los clics exactos del dashboard para que los haga él (son pocos), o disparás la
  carga por la API admin (`POST /syndication/install` con `{editionId, version,
  derivativeContentItemVersions:[]}`) y seguís el progreso con
  `GET /syndication/install/{taskId}`. No inventes el `editionId`/`version`: primero listá
  las ediciones reales del feed (`GET /syndication/snomed-editions`) y elegí con el usuario.
  MLDS suele ofrecer varias candidatas: para "edición español" genérica la correcta es
  **SNOMED CT Spanish package** (módulo `450829007`); Argentina y Uruguay son ediciones
  nacionales distintas — confirmá cuál quiere antes de instalar.
- **B — Archivos RF2 locales:** subilos con el curl a `load-package`. Como el español son
  DOS paquetes (International + extensión español), subí LOS DOS en una sola llamada con el
  `version-uri` de la edición (`http://snomed.info/sct/450829007/version/AAAAMMDD`). Confirmá
  que el set esté completo (una extensión nacional necesita también el paquete International).
  `load-package` es **síncrono** — el curl queda bloqueado hasta terminar (unos minutos) y
  devuelve HTTP 200.
- **C — Feed demo:** en el dashboard, Settings → Syndication Feed, poné la URL base
  `https://snomed-demo-feed.vercel.app` **sin** `/feed` (el cliente lo agrega), y luego
  instalá la edición IPS Terminology Test. Aclarale que esto es solo para probar la
  instalación; no es contenido en español. **El dashboard descubre ediciones sin
  credenciales, pero el botón Install es una acción admin**: el navegador pedirá el Basic
  Auth admin (usa el `ADMIN_USERNAME`/`ADMIN_PASSWORD` del env). El dashboard no tiene login
  propio.

## Paso 3 — Confirmar el éxito
- Que haya un CodeSystem cargado:
  ```bash
  curl -s "http://localhost:8080/fhir/CodeSystem?_format=json"
  ```
- Al ser edición de idioma, confirmá que un término resuelve en español:
  ```bash
  curl -s "http://localhost:8080/fhir/CodeSystem/\$lookup?system=http://snomed.info/sct&code=195967001&displayLanguage=es&_format=json"
  ```
  (código 195967001 → display "asma"). Después entregá la base FHIR http://localhost:8080/fhir.

## Playbook de diagnóstico (síntoma → causa probable → acción)
| Síntoma | Causa probable | Acción |
|---------|----------------|--------|
| El contenedor sale durante el import; exit code 137; `OOMKilled=true` | Falta memoria para `-Xmx4g` | Subí la memoria de Docker Desktop a ≥ 5–6 GB y reintentá. Chequeo: `docker inspect snowstorm-lite --format '{{.State.OOMKilled}} {{.State.ExitCode}}'` |
| `bind: address already in use` en 8080 | Puerto ocupado | Cambiá el puerto del host en el compose (ej. `"8081:8080"`) y usá ese puerto |
| `java.nio.file.AccessDeniedException: /app/lucene-index/data` | Corrió `docker run` a mano, salteando el init | Usá `docker compose up -d` (corrige el dueño del volumen) |
| Feed 404 / `.../feed/feed` | Puso la URL del feed **con** `/feed` | Usar la URL base sin `/feed` |
| Opción B: el import falla / faltan conceptos | Set de archivos incompleto o `version-uri` incorrecto | Incluir el paquete International con la extensión; verificar el `version-uri` |
| `401` en `load-package` | Credenciales admin incorrectas | Usar el `ADMIN_USERNAME`/`ADMIN_PASSWORD` del env |
| Opción A: el **listado** de ediciones MLDS funciona pero la **descarga del ZIP** da `401` | `SYNDICATION_PASSWORD` con comillas o `\` de escape en el `.env` (se toma literal) | Corregir a valor literal (ej. `*` no `\*`), recrear el contenedor y reintentar |
| Falla la descarga de la imagen | Sin internet / proxy | Verificar conectividad; configurar el proxy de Docker si hay uno |

Cuando diagnostiques, leé siempre los logs: `docker compose logs -f snowstorm-lite`.
