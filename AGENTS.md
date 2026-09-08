# AGENTS.md — guía para un asistente de IA que ayuda a instalar Snowstorm Lite (edición español)

Estás ayudando a un usuario a levantar **Snowstorm Lite** (servidor FHIR de terminología
SNOMED CT) localmente con Docker, usando los archivos de este repo (`docker-compose.yml`,
`snowstorm-lite.env.example`, `README.md`). Este repo está orientado a la **edición en
español**. Sigue este playbook. El README tiene los comandos exactos y los bloques para
copiar/pegar — úsalos; este archivo te dice *cómo conducir el proceso*.

**Guía al usuario en español.**

## Principios de trabajo
- **Detecta el sistema operativo** del usuario (macOS / Linux / Windows) y muestra solo los
  comandos de ese SO. No vuelques todas las variantes juntas.
- **Ve paso a paso y verifica cada uno** antes de seguir. Confirma que el comando previo
  funcionó (código de salida, salida esperada) en vez de asumirlo.
- **Asume que el usuario puede no ser técnico.** Explica en una línea qué hace cada comando.
- **Nunca inventes un `version-uri`** de una edición SNOMED. Búscalo (ver los ejemplos de
  Edition URI enlazados en el README) o derívalo de la edición/módulo que nombre el usuario,
  y confírmalo con él.
- **Secretos:** el usuario elige su propio `ADMIN_PASSWORD`. No pidas credenciales MLDS a
  menos que elija la Opción A, y nunca repitas contraseñas completas de vuelta.

## Paso 0 — Verificaciones previas (hazlas primero, antes que nada)
Ejecuta e interpreta esto; resuelve lo que falle antes de continuar.

1. **Docker instalado y corriendo:**
   ```bash
   docker version
   ```
   Si da error, el usuario debe instalar/arrancar Docker Desktop (macOS/Windows) o el
   Docker Engine (Linux) primero.

2. **Memoria suficiente para la JVM (crítico).** La app corre con `-Xmx4g`, así que Docker
   necesita ~5–6 GB disponibles o el import muere por **OOM**. Verifica lo que tiene Docker:
   ```bash
   docker info --format '{{.MemTotal}}'
   ```
   Si es menor a ~5000000000 (5 GB), dile al usuario que suba la memoria de Docker Desktop
   (Settings → Resources → Memory) antes de cargar la terminología. Es la falla real más
   común — verifícala de entrada.

3. **Puerto 8080 libre** (o planifica cambiar el puerto del host en el compose si no lo está).

## Paso 1 — Obtener los archivos y levantar el stack
Guía al usuario por las secciones del README "Obtén los archivos", "Configura tu archivo de
entorno" y "Levanta el servidor". Puntos clave:
- Debe crear `snowstorm-lite.env` a partir del `.example` y poner `ADMIN_PASSWORD`.
- Se arranca con `docker compose up -d`. El servicio `init-permissions` corre una vez y
  corrige el dueño del volumen automáticamente — **no** le digas que corra `docker run` a
  mano o chocará con `AccessDeniedException`.
- Verifica: `docker compose ps` muestra `snowstorm-lite` Up, y los logs terminan con
  *"Snowstorm Lite started. Please load a SNOMED CT package."*

## Paso 2 — Elegir cómo cargar la terminología (pregúntale al usuario)
Pregunta: **"¿Tienes (A) credenciales MLDS, (B) los archivos RF2 ya descargados, o
(C) nada — solo quieres probar?"** Luego sigue la opción correspondiente del README.

> **Atención:** Snowstorm Lite mantiene **una sola edición a la vez** — cargar/instalar otra
> **reemplaza** la anterior. Avisa al usuario antes de cargar algo si ya tiene una edición
> que quiere conservar.
- **A — Sindicación MLDS:** pon `SYNDICATION_*` en el env y `docker compose up -d` de nuevo
  para aplicar. Luego **pregúntale al usuario cómo quiere disparar la carga**:
  (1) por el **dashboard** — recomendado y lo más simple; la mayoría va a preferir esto —, o
  (2) que tú la dispares por la API admin. **Por defecto llévalo al dashboard y limítate a
  acompañarlo a abrir la app:** que abra http://localhost:8080, entre a **Syndication**,
  elija la edición en español y le dé instalar. Usa la API solo si te lo pide expresamente.
  Las ediciones multi-paquete (español = International + extensión) se cargan
  automáticamente. (Recuerda: instalar necesita auth admin en el navegador — ver Opción C.)
  Si NO puedes manejar el navegador (eres un agente de CLI), díselo al usuario y ofrécele:
  o le pasas los clics exactos del dashboard para que los haga él (son pocos), o disparas la
  carga por la API admin (`POST /syndication/install` con `{editionId, version,
  derivativeContentItemVersions:[]}`) y sigues el progreso con
  `GET /syndication/install/{taskId}`. No inventes el `editionId`/`version`: primero lista
  las ediciones reales del feed (`GET /syndication/snomed-editions`) y elige con el usuario.
  MLDS suele ofrecer varias candidatas: para "edición español" genérica la correcta es
  **SNOMED CT Spanish package** (módulo `450829007`); Argentina y Uruguay son ediciones
  nacionales distintas — confirma cuál quiere antes de instalar.
- **B — Archivos RF2 locales:** súbelos con el curl a `load-package`. Como el español son
  DOS paquetes (International + extensión español), sube LOS DOS en una sola llamada con el
  `version-uri` de la edición (`http://snomed.info/sct/450829007/version/AAAAMMDD`). Confirma
  que el set esté completo (una extensión nacional necesita también el paquete International).
  `load-package` es **síncrono** — el curl queda bloqueado hasta terminar (unos minutos) y
  devuelve HTTP 200.
- **C — Feed demo:** en el dashboard, Settings → Syndication Feed, pon la URL base
  `https://snomed-demo-feed.vercel.app` **sin** `/feed` (el cliente lo agrega), y luego
  instala la edición IPS Terminology Test. Aclárale que esto es solo para probar la
  instalación; no es contenido en español. **El dashboard descubre ediciones sin
  credenciales, pero el botón Install es una acción admin**: el navegador pedirá el Basic
  Auth admin (usa el `ADMIN_USERNAME`/`ADMIN_PASSWORD` del env). El dashboard no tiene login
  propio.

## Paso 3 — Confirmar el éxito
- Que haya un CodeSystem cargado:
  ```bash
  curl -s "http://localhost:8080/fhir/CodeSystem?_format=json"
  ```
- Al ser edición de idioma, confirma que un término resuelve en español:
  ```bash
  curl -s "http://localhost:8080/fhir/CodeSystem/\$lookup?system=http://snomed.info/sct&code=195967001&displayLanguage=es&_format=json"
  ```
  (código 195967001 → display "asma"). Luego entrega la base FHIR http://localhost:8080/fhir.

## Playbook de diagnóstico (síntoma → causa probable → acción)
| Síntoma | Causa probable | Acción |
|---------|----------------|--------|
| El contenedor sale durante el import; exit code 137; `OOMKilled=true` | Falta memoria para `-Xmx4g` | Sube la memoria de Docker Desktop a ≥ 5–6 GB y reintenta. Verificación: `docker inspect snowstorm-lite --format '{{.State.OOMKilled}} {{.State.ExitCode}}'` |
| `bind: address already in use` en 8080 | Puerto ocupado | Cambia el puerto del host en el compose (ej. `"8081:8080"`) y usa ese puerto |
| `java.nio.file.AccessDeniedException: /app/lucene-index/data` | Corrió `docker run` a mano, saltándose el init | Usa `docker compose up -d` (corrige el dueño del volumen) |
| Feed 404 / `.../feed/feed` | Puso la URL del feed **con** `/feed` | Usar la URL base sin `/feed` |
| Opción B: el import falla / faltan conceptos | Set de archivos incompleto o `version-uri` incorrecto | Incluir el paquete International con la extensión; verificar el `version-uri` |
| `401` en `load-package` | Credenciales admin incorrectas | Usar el `ADMIN_USERNAME`/`ADMIN_PASSWORD` del env |
| Opción A: el **listado** de ediciones MLDS funciona pero la **descarga del ZIP** da `401` | `SYNDICATION_PASSWORD` con comillas o `\` de escape en el `.env` (se toma literal) | Corregir a valor literal (ej. `*` no `\*`), recrear el contenedor y reintentar |
| Falla la descarga de la imagen | Sin internet / proxy | Verificar conectividad; configurar el proxy de Docker si hay uno |

Cuando diagnostiques, lee siempre los logs: `docker compose logs -f snowstorm-lite`.
