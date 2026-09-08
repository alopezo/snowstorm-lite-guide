# Guía de instalación — Snowstorm Lite (edición en español)

Guía y `docker-compose.yml` listos para usar para correr [Snowstorm Lite](https://github.com/IHTSDO/snowstorm-lite)
(servidor FHIR de terminología SNOMED CT) que **funciona al primer `docker compose up`** —
sin `chown` manual ni caída en el primer arranque. Está orientada a cargar la **edición en
español** de SNOMED CT.

## ¿Tenés un agente de IA? Empezá por acá
Copiá esto y pasáselo a tu agente (Claude Code, etc.). Él va a leer las instrucciones y
**hacerte las preguntas necesarias** (tu sistema operativo, si tenés credenciales MLDS,
archivos, o nada) y guiarte paso a paso:

```
Ayudame a instalar Snowstorm Lite (edición español) paso a paso.
Cloná/leé este repo, seguí el AGENTS.md, hacé los chequeos previos
y preguntame lo que necesites (SO, credenciales MLDS / archivos / nada):
https://github.com/alopezo/snowstorm-lite-guide
```

Si preferís hacerlo a mano, seguí el resto de este README.

## Archivos
| Archivo | Qué es |
|---------|--------|
| `docker-compose.yml` | El stack: un paso init que corrige permisos del volumen + la app. |
| `snowstorm-lite.env.example` | Plantilla para tu archivo de entorno. Copiala a `snowstorm-lite.env`. |
| `AGENTS.md` | Guía para que un asistente de IA acompañe al usuario en esta instalación. |
| `.gitignore` | Evita subir tu env real y los zips por accidente. |
| `README.md` | Este archivo. |

## Requisitos previos
- **Docker** instalado y corriendo (Docker Desktop en macOS/Windows, Docker Engine en
  Linux). Verificá con `docker version`.
- **Memoria:** el servidor corre la JVM con `-Xmx4g`, así que asigná a Docker **~5–6 GB** o
  más (Docker Desktop → Settings → Resources → Memory). Con menos, la carga de la
  terminología muere por falta de memoria (**OOM**) a mitad del import. Es la falla más
  común — configuralo antes de empezar.
- **Disco:** unos GB libres (índice Lucene + zips RF2 descargados).
- **Puerto 8080** libre (o cambiá el puerto del host en el compose).
- **Internet** para descargar la imagen (~540 MB) y, en las Opciones A/C, alcanzar el feed.

## 1. Obtené los archivos
Cloná el repo:
```bash
git clone https://github.com/alopezo/snowstorm-lite-guide.git
cd snowstorm-lite-guide
```
¿Sin `git`? Descargá el ZIP desde la página del repo (botón **Code → Download ZIP**) y
descomprimilo, o bajá archivos sueltos por su URL *raw*, p. ej.:
```bash
curl -L -O https://raw.githubusercontent.com/alopezo/snowstorm-lite-guide/main/docker-compose.yml
curl -L -O https://raw.githubusercontent.com/alopezo/snowstorm-lite-guide/main/snowstorm-lite.env.example
```
(En Windows PowerShell usá `curl.exe`.)

## 2. Configurá tu archivo de entorno
```bash
cp snowstorm-lite.env.example snowstorm-lite.env   # después editalo
```

| Clave | ¿Requerida? | Para qué |
|-------|-------------|----------|
| `ADMIN_USERNAME` | opcional (por defecto `admin`) | Login del dashboard / API admin |
| `ADMIN_PASSWORD` | **sí** | Contraseña admin — poné la tuya |
| `SYNDICATION_USERNAME` | opcional | Solo para descarga automática vía MLDS (Opción A) |
| `SYNDICATION_PASSWORD` | opcional | Solo para descarga automática vía MLDS (Opción A) |

## 3. Levantá el servidor
```bash
docker compose up -d
```
Abrí el dashboard en **http://localhost:8080** y cargá una terminología (más abajo).
Seguí los logs con: `docker compose logs -f snowstorm-lite`

---

## ¿Por qué existe el paso init?
La imagen publicada corre como usuario no-root (uid `1000`, hardening del contenedor). Un
volumen Docker nuevo se crea con dueño `root`, así que la app no puede crear su índice
Lucene y se cae en el primer arranque con:

```
java.nio.file.AccessDeniedException: /app/lucene-index/data
```

El servicio `init-permissions` corre una sola vez y hace `chown` del volumen a `1000:1000`
**antes** de que arranque la app, así todo funciona sin pasos manuales.

---

## Cargar la terminología

Hay **tres formas** de cargar contenido. Elegí la que corresponda a lo que tenés:

| Tenés… | Usá | ¿Credenciales? |
|--------|-----|----------------|
| credenciales MLDS | **Opción A** — sindicación oficial (SNOMED CT real) | sí |
| los archivos RF2 | **Opción B** — subir los archivos locales directo | no |
| nada (solo probar) | **Opción C** — feed demo libre (datos IPS de prueba) | no |

> **Importante:** Snowstorm Lite mantiene **una sola edición SNOMED CT a la vez**. Cargar o
> instalar otra edición **reemplaza** la que estuviera cargada (el índice se sobrescribe).

### Opción A — Sindicación oficial con credenciales MLDS
Para descargar la edición en español real. Poné tus credenciales MLDS en el archivo de
entorno:
```
SYNDICATION_USERNAME=tu-usuario-mlds
SYNDICATION_PASSWORD=tu-password-mlds
```
Después, en el dashboard elegí **Syndication** y seleccioná la **edición en español**. Las
ediciones multi-paquete (español = International + extensión en español) se descargan y
cargan **automáticamente**.

> MLDS suele listar varias ediciones relacionadas: el **SNOMED CT Spanish package** genérico
> (módulo `450829007`) y además ediciones nacionales separadas (p. ej. Argentina, Uruguay).
> Elegí la que corresponda a tu caso — no son la misma.

> Valor literal en el env: si tu `SYNDICATION_PASSWORD` tiene caracteres como `*` o `$`,
> escribilo tal cual (sin comillas ni `\`). Un escape de más provoca un `401` al descargar
> el ZIP (ver Solución de problemas).

Los zips RF2 descargados se cachean **dentro del volumen** en `/app/lucene-index/rf2-cache`
(ver el flag `--syndication.rf2-download-cache-directory` en el compose), así persisten
entre reinicios y se reutilizan en la próxima carga.

> Sin ese flag, el directorio de cache por defecto no se puede crear (`/app` es de root, la
> app corre como uid 1000), así que las descargas de sindicación **no** se guardarían.

Para copiar los zips cacheados fuera del volumen y reutilizarlos:
```bash
docker cp snowstorm-lite:/app/lucene-index/rf2-cache ./rf2-cache
```

**Alternativa: disparar la carga por API (sin dashboard).** Útil para automatizar. Primero
listá las ediciones reales del feed y elegí la correcta (no inventes el `editionId`):
```bash
curl -s -u admin:TU_ADMIN_PASSWORD http://localhost:8080/syndication/snomed-editions
```
Luego instalá (el español genérico es el módulo `450829007`):
```bash
curl -s -u admin:TU_ADMIN_PASSWORD -H 'Content-Type: application/json' \
  -X POST http://localhost:8080/syndication/install \
  -d '{"editionId":"http://snomed.info/sct/450829007","version":"20260810","derivativeContentItemVersions":[]}'
```
La respuesta trae un `taskId`. Monitoreá el progreso:
```bash
curl -s -u admin:TU_ADMIN_PASSWORD http://localhost:8080/syndication/install/TASK_ID
```

### Opción B — Subir los archivos RF2 locales directo
Si te pasaron los `.zip`, subilos por la API admin — sin sindicación. Como la edición en
español está hecha de **dos paquetes, se suben LOS DOS en una sola llamada** con el
`version-uri` de la edición.

**Edición en español (dos archivos):**
```bash
curl -u admin:TU_ADMIN_PASSWORD \
  --form file=@SnomedCT_InternationalRF2_PRODUCTION_20260701T120000Z.zip \
  --form file=@SnomedCT_SpanishRelease-es_PRODUCTION_20260810T120000Z.zip \
  --form version-uri="http://snomed.info/sct/450829007/version/20260810" \
  http://localhost:8080/fhir-admin/load-package
```
El paquete International aporta el contenido base; el paquete español es una extensión de
idioma que lo necesita — por eso van los dos en la **misma** petición. `load-package` es
**síncrono**: el comando queda bloqueado hasta que termina el import (unos minutos) y recién
ahí devuelve HTTP 200. Seguí el progreso con `docker compose logs -f snowstorm-lite`.

**Windows (PowerShell)** — la misma llamada en una línea con `curl.exe`:
```powershell
curl.exe -u admin:TU_ADMIN_PASSWORD --form file=@SnomedCT_InternationalRF2_PRODUCTION_20260701T120000Z.zip --form file=@SnomedCT_SpanishRelease-es_PRODUCTION_20260810T120000Z.zip --form version-uri="http://snomed.info/sct/450829007/version/20260810" http://localhost:8080/fhir-admin/load-package
```

- Ajustá los nombres de los `.zip` a los tuyos y el `version-uri` a la versión que cargás.
- `version-uri` de otras ediciones: ver los
  [ejemplos de Edition URI](https://github.com/IHTSDO/snowstorm-lite/blob/master/docs/snomed-edition-uri-examples.md).

### Opción C — Feed demo libre (sin credenciales, solo para probar la instalación)
Si no tenés cuenta MLDS ni archivos, podés apuntar Snowstorm Lite a un feed de sindicación
demo público que sirve un paquete chico de prueba, de distribución libre: **IPS
(International Patient Summary)**. Sirve para verificar que la instalación funciona; **no es
contenido en español**, es sólo de prueba.

1. En el dashboard andá a **Settings → Syndication Feed**.
2. Reemplazá la URL del feed por la **URL base** del feed demo:
   ```
   https://snomed-demo-feed.vercel.app
   ```
   > ⚠️ Poné la URL base **sin** `/feed`. Snowstorm Lite le agrega `/feed` solo (hace
   > `rootUri(base)` y después pide `/feed`). Si escribís `.../feed`, va a pedir
   > `.../feed/feed` y da 404.
3. Refrescá, abrí el menú **Syndication** e instalá la edición **IPS Terminology Test**.

> **Nota sobre autenticación:** el dashboard **descubre** ediciones sin credenciales, pero
> **instalar** es una acción admin — el navegador pedirá usuario/contraseña admin (Basic
> Auth). Usá el `ADMIN_USERNAME`/`ADMIN_PASSWORD` del env.

Fuente del feed demo: <https://github.com/alopezo/snomed-demo-feed>. Solo para
pruebas/evaluación — no es contenido de producción. Instalar el IPS **reemplaza** la
edición que tuvieras cargada (ver la nota de arriba).

## Verificar que cargó
Listá los CodeSystem cargados:
```bash
curl -s "http://localhost:8080/fhir/CodeSystem?_format=json"
```
Al ser edición en español, confirmá que un término resuelve en español — el código
`195967001` debería mostrar **"asma"**:
```bash
curl -s "http://localhost:8080/fhir/CodeSystem/\$lookup?system=http://snomed.info/sct&code=195967001&displayLanguage=es&_format=json"
```
Si tenés `jq` (no viene por defecto en macOS), extraé solo el término y evitá el ruido:
```bash
curl -s "http://localhost:8080/fhir/CodeSystem/\$lookup?system=http://snomed.info/sct&code=195967001&displayLanguage=es&_format=json" \
  | jq -r '.parameter[] | select(.name=="display") | .valueString'
```
La interfaz FHIR está en http://localhost:8080/fhir

## Gestionar el stack
```bash
docker compose logs -f snowstorm-lite   # seguir logs
docker compose stop                      # parar (conserva los datos)
docker compose start                     # arrancar de nuevo
docker compose down                      # borrar contenedores (conserva el volumen/datos)
docker compose down -v                   # borrar contenedores Y vaciar el índice cargado
docker compose pull && docker compose up -d   # re-descargar la imagen fijada (:2.5.2)
docker compose down -v --rmi all --remove-orphans   # limpieza TOTAL: contenedores, volumen (índice + zips) e imagen
```
El último comando deja todo de cero (útil para ciclos de prueba): el próximo `up` vuelve a
hacer el `pull` completo de la imagen.

## Solución de problemas
| Síntoma | Causa | Solución |
|---------|-------|----------|
| El contenedor sale a mitad del import; exit code 137 | Falta memoria para `-Xmx4g` | Subí la memoria de Docker a ≥ 5–6 GB y reintentá. Chequeo: `docker inspect snowstorm-lite --format '{{.State.OOMKilled}} {{.State.ExitCode}}'` |
| `bind: address already in use` (8080) | Puerto ocupado | Cambiá el puerto del host en el compose, ej. `"8081:8080"` |
| `AccessDeniedException: /app/lucene-index/data` | Arrancaste con `docker run` a mano, salteando el paso init | Usá `docker compose up -d` (corrige el dueño del volumen) |
| Feed 404 / `.../feed/feed` | Pusiste la URL del feed **con** `/feed` | Usá la URL base **sin** `/feed` |
| Opción B: el import falla o faltan conceptos | Set de archivos incompleto / `version-uri` incorrecto | Incluí el paquete International junto con la extensión; revisá el `version-uri` |
| `401` en `load-package` | Credenciales admin incorrectas | Usá el `ADMIN_USERNAME`/`ADMIN_PASSWORD` de tu env |
| Opción A: `401` al **descargar el ZIP** de MLDS (aunque el listado de ediciones funcione) | `SYNDICATION_PASSWORD` mal escrita en el `.env` (comillas o `\` de escape) | Poné el valor **literal**, sin comillas ni barras invertidas; recreá con `docker compose up -d` |

## Notas
- Mantené privado tu `snowstorm-lite.env` real — tiene credenciales. Compartí solo el
  `.example`.
- La imagen está **fijada a `:2.5.2`** en el compose (reproducible para compartir). Para
  actualizar a una versión nueva, cambiá el tag en las dos líneas `image:` del
  `docker-compose.yml` y recreá con `docker compose up -d`. Las versiones disponibles están
  en [Docker Hub](https://hub.docker.com/r/snomedinternational/snowstorm-lite/tags) y se
  corresponden con los [releases del repo](https://github.com/IHTSDO/snowstorm-lite/releases).
- **Persistencia dentro del volumen.** Como la imagen corre como uid 1000 pero `/app` es de
  root, varias cosas que el servidor querría escribir en `/app` fallan con `AccessDenied`.
  Por eso el compose redirige dos de ellas al volumen (que sí es escribible), así
  **sobreviven a reinicios**:
  - `--syndication.rf2-download-cache-directory=lucene-index/rf2-cache` — cache de zips RF2
    descargados por sindicación.
  - `--syndication.feed-config-file=lucene-index/syndication-feed-config.properties` — la
    URL/credenciales del feed guardadas desde Settings.
  Sin estos flags, esas descargas y la config del feed **se perderían** en cada reinicio.
