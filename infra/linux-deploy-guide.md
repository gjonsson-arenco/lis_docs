# Guía de deploy on-premise en Linux (Docker, sin registry)

Guía práctica para desplegar el stack de LIS (backend, broker-gateway, clinical-matcher,
frontend + MySQL/Redis) en un server Linux on-premise, con build directo en el server
(sin registry de imágenes). Escrita a partir de la experiencia real del primer deploy
(cliente CEBAC) — cada sección de "errores comunes" documenta un bug real que salió en
ese proceso, no hipótesis.

Repo de infra de referencia: `lis-infra` (rama `main` = template genérico, una rama
`deploy/<cliente>` por cada cliente on-premise).

---

## 1. Arquitectura del stack

| Servicio | Tecnología | Puerto | Healthcheck |
|---|---|---|---|
| `backend` | Laravel / PHP-FPM | 8000 | `/api/v1/health` |
| `broker-gateway` | NestJS/TypeScript | 3001 | `/v1/health` (⚠️ versionado, ver §6.4) |
| `frontend` | Next.js (App Router, standalone) | 3000 | `/health` |
| `clinical-matcher` | FastAPI/Python | 8001 | `/health` |
| `mysql` | MySQL 8.0 | 3306 | — |
| `redis` | Redis 7.4 | 6379 | — |

`mysql`/`redis` viven en el compose "base" (infraestructura compartida); los 4
servicios de la app viven en un compose "prod" aparte, que depende del primero.
Ambos se levantan **en una sola invocación** (`-f base.yml -f prod.yml`), no por
separado — si `prod.yml` tiene `depends_on: mysql` pero `mysql` está definido en
el otro archivo, Compose no resuelve la dependencia entre invocaciones separadas.

---

## 2. Prerequisitos del server

- Ubuntu Server con IP estática (vía netplan).
- Docker Engine + Compose plugin instalados desde el **repo oficial de Docker**
  (no el paquete de Ubuntu — suele quedar desactualizado).
- Usuario del deploy agregado al grupo `docker`.
- Deploy key SSH dedicada para clonar los repos (`~/.ssh/id_<algo>`, configurada en
  `~/.ssh/config` para `Host github.com`) — no uses tu key personal.
- Acceso de red: si el server es alcanzable solo por VPN corporativa (sin dominio
  público, sin TLS), documentá la IP privada del server — la vas a necesitar para
  Cognito/CORS/redirect URIs (ver §6.3).

---

## 3. Estructura de directorios y repos

```
/opt/lis/
├── <servicio-1>/   (git@github.com:.../repo1.git)
├── <servicio-2>/   ...
├── ...
└── lis-infra/      (rama deploy/<cliente>)
```

Cloná cada repo con el nombre real del repo en GitHub — **no asumas** que el nombre
del repo coincide con el nombre de la carpeta o con lo que dice algún doc viejo.
Verificá con `git remote -v` en tu clon local antes de armar los comandos de clone
para el server; en este proyecto puntual, tres de los cuatro nombres de repo
diferían de lo que decía la documentación interna (guión vs. guión bajo, nombre
distinto al de la carpeta).

Red Docker compartida, una sola vez:

```bash
docker network create lis-network
```

Todos los servicios se conectan ahí para resolverse por nombre de contenedor
(`http://backend:8000`, `http://mysql:3306`, etc.)

---

## 4. Estrategia multi-cliente en `lis-infra`

Para instalaciones on-premise en varios clientes, conviene decidir esto **antes**
de acumular configuración específica de un cliente en la rama principal:

- **Rama por cliente** (recomendado si ya tenés un pipeline de CI/CD pensado así):
  `main` queda como base/template genérico; `deploy/<cliente>` tiene el
  `docker-compose.prod.yml`, `.env.example` y cualquier ajuste específico de ese
  cliente. Cada server on-premise hace checkout de su propia rama.
- Alternativas (carpetas por cliente en una sola rama, o un repo de infra por
  cliente) existen, pero traen más complejidad de mantenimiento si ya tenés el
  patrón de branch-per-environment establecido en el resto del proyecto.

---

## 5. Build y levantamiento

```bash
cd /opt/lis/lis-infra
git checkout deploy/<cliente>

docker network create lis-network   # una sola vez

# variables de entorno — ver sección 6 antes de este paso
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```

Migraciones (si aplica, ej. Laravel):

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml exec backend \
  php artisan migrate --force
```

El `--force` es necesario porque el framework suele pedir confirmación interactiva
en `APP_ENV=production`, y en un `exec` no hay TTY para confirmar.

Ver estado y logs:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml ps
docker compose -f docker-compose.yml -f docker-compose.prod.yml logs -f <servicio>
```

---

## 6. Manejo de variables de entorno

### 6.1 Regla de secretos

Los `.env` reales **no se versionan** en ningún repo (gitignorados, `.env.example`
sí se versiona). Se crean a mano en el server con `chmod 600`.

### 6.2 Runtime vs. build-time — el gotcha más caro de esta guía

`env_file:` y `environment:` en `docker-compose.prod.yml` solo llegan al
**contenedor ya corriendo**. Cualquier variable que la app necesite **durante el
build** (`docker build` / `docker compose build`) nunca las va a ver por esa vía.

El caso más común: apps Next.js con variables `NEXT_PUBLIC_*` — Next.js las
inlinea en el bundle en tiempo de build, no las lee en runtime. Si tu Dockerfile
no declara `ARG`/`ENV` para ellas y las pasás como build args, la app arranca con
esas variables `undefined` (o, si el código valida con un `throw`, el build
directamente falla con "Missing environment variable").

**Fix**: declarar `ARG`/`ENV` en el Dockerfile para cada variable que se necesite
en build-time, y pasarlas como `build.args` en el compose:

```yaml
# Dockerfile
ARG NEXT_PUBLIC_COGNITO_DOMAIN
ENV NEXT_PUBLIC_COGNITO_DOMAIN=$NEXT_PUBLIC_COGNITO_DOMAIN
```

```yaml
# docker-compose.prod.yml
build:
  args:
    - NEXT_PUBLIC_COGNITO_DOMAIN   # forma "bare": toma el valor del shell/.env del proyecto
```

Como esas variables suelen vivir en el `.env.prod` de OTRO repo (no en el
directorio donde corrés `docker compose`), hay que exportarlas al shell antes de
buildear:

```bash
set -a
source /opt/lis/<frontend-repo>/apps/<app>/.env.prod
set +a
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```

### 6.3 URLs cuando no hay dominio ni TLS

Si el acceso es solo por VPN sin dominio público, las URLs que le importan al
navegador (redirect/logout URIs de un proveedor OAuth, CORS) tienen que usar la
**IP privada del server + puerto**, con `http://` (no `https://`):

```
NEXT_PUBLIC_COGNITO_REDIRECT_URI=http://<ip-server>:3000/auth/callback
CORS_ALLOWED_ORIGINS=http://<ip-server>:3000
```

Si usás un proveedor de identidad externo (Cognito, Auth0, etc.), **estas mismas
URLs también hay que registrarlas del lado del proveedor** (allowed
callback/sign-out URLs) — cambiar solo el `.env` no alcanza, el proveedor va a
rechazar el redirect igual si no coincide con lo que tiene configurado.

### 6.4 Variables compartidas entre servicios

Algunos valores tienen que **coincidir exactamente** entre dos `.env` de
servicios distintos (ej.: un token interno que un servicio genera y otro
valida). Mantenerlos sincronizados a mano en varios archivos es una fuente
constante de bugs silenciosos. Preferí centralizar lo compartido en el `.env`
de `lis-infra` e inyectarlo a cada contenedor vía `environment:` en el compose
(mismo mecanismo que en §6.2), en vez de duplicar el valor en cada `.env.prod`
de cada repo.

### 6.5 Gotchas de shell al crear/editar `.env` a mano

- **CRLF**: si el archivo se creó o editó desde Windows (pegado en un editor, o
  transferido sin cuidado), las líneas terminan en `\r\n`. Al hacer `source` de
  ese archivo en bash, cada línea arrastra un `\r` invisible que rompe el
  parseo — típicamente como `command not found` con nombres de comando raros.
  Verificar con `cat -A archivo | head` (buscar `^M$` al final de cada línea) y
  arreglar con `sed -i 's/\r$//' archivo`.
- **Valores con espacios sin comillas**: `VAR=valor con espacios` en un archivo
  que se va a `source`-ar rompe — bash interpreta las palabras después de la
  primera como comandos separados. Necesita comillas: `VAR="valor con espacios"`.
- Si un `.env.example` no existe para algún servicio, es buena señal de que hay
  que crearlo — no reconstruir el archivo real de memoria cada vez que hace
  falta.

---

## 7. Errores comunes en Dockerfiles multi-stage (y cómo evitarlos)

Todos estos bugs comparten un patrón: **el build (`docker build`) pasa sin
error, pero el contenedor crashea o queda "unhealthy" recién al correrlo**.
Verificar solo que el build termine no alcanza — hay que correr el contenedor
y probar una request real antes de dar un fix por bueno (ver §8).

### 7.1 `COPY --from=<stage>` resuelve paths desde la raíz del stage, no desde su WORKDIR

```dockerfile
WORKDIR /app
# ...
COPY --from=builder /app/package.json pnpm-lock.yaml ./
```

A diferencia de un `COPY` normal del build context, con `--from=<stage>` las
rutas (incluso las relativas, sin `/` inicial) se resuelven contra la **raíz del
filesystem** del stage de origen, no contra su `WORKDIR`. En el ejemplo,
`pnpm-lock.yaml` se busca en `/pnpm-lock.yaml` (raíz), no en
`/app/pnpm-lock.yaml` (donde realmente está) — y el build falla con un error de
BuildKit tipo `failed to compute cache key: ... "/pnpm-lock.yaml": not found`,
que fácilmente se confunde con un problema de cache corrupta. Fix: usar la ruta
absoluta completa en todas las fuentes: `COPY --from=builder /app/package.json /app/pnpm-lock.yaml ./`.

### 7.2 Output "standalone" de Next.js en un monorepo duplica el path

Con `output: "standalone"` en `next.config`, Next genera el bundle de servidor en
`.next/standalone/`. En un monorepo (Turborepo/pnpm workspaces), ese directorio
**ya reproduce internamente la ruta relativa al workspace** (ej.
`.next/standalone/apps/<app>/server.js`), porque Next traza el grafo de
dependencias completo desde la raíz del monorepo detectada.

Si el Dockerfile copia ese contenido a `./apps/<app>/` (en vez de a `./`), el
resultado queda duplicado: `/app/apps/<app>/apps/<app>/server.js`, mientras el
`CMD` busca `/app/apps/<app>/server.js` → `Error: Cannot find module`, contenedor
en restart loop. El build en sí **no falla**, así que este bug solo aparece al
correr el contenedor. Fix: copiar el standalone a la raíz del WORKDIR (`./`), no
a la subcarpeta del app; los `COPY` de `.next/static` y `public/` sí van a la
subcarpeta (esos no vienen duplicados).

También verificar que `turbo prune --docker` reciba en su contexto los archivos
de configuración de raíz que el app necesita por fuera del grafo de dependencias
de paquetes (ej. un `tsconfig.base.json` referenciado por `extends` con path
relativo) — `turbo prune` solo entiende package.json/lockfile/turbo.json, no
copia archivos sueltos de la raíz automáticamente.

### 7.3 Usuario no-root + volumen con nombre montado en un directorio que no existe en la imagen

Patrón: la app escribe logs en un path relativo (`logs/algo.log`) que resuelve a
`/app/logs/...`, corriendo como usuario no-root, y el compose monta un volumen
nombrado ahí (`- app-logs:/app/logs`).

Si `/app/logs` no existe **en la imagen** en el momento en que Docker crea el
volumen por primera vez, Docker lo inicializa como `root:root` — sin importar que
el Dockerfile haya hecho `chown -R` sobre `/app` en un paso anterior (no hay nada
que "heredar" si el directorio nunca existió). El proceso no-root falla con
`EACCES: permission denied` al intentar escribir ahí. Si ese log se abre en el
constructor/bootstrap de la app (común en frameworks con inyección de
dependencias), **tira abajo toda la app**, no solo el logging.

Fix: crear el directorio explícitamente con el owner correcto *antes* de que el
volumen lo monte:

```dockerfile
RUN mkdir -p /app/logs && chown app:app /app/logs
```

### 7.4 Healthcheck apuntando a una ruta que no existe o está mal versionada

Dos variantes vistas en el mismo deploy:

- Una API con versionado de rutas (`/v1/health` en vez de `/health`) tenía el
  `HEALTHCHECK` del Dockerfile pegándole a `/health` → 404 constante. El
  `healthcheck:` del compose ya estaba bien y lo pisaba en producción, pero el
  `HEALTHCHECK` de la imagen quedaba mal para cualquiera que corriera el
  contenedor suelto (que es como se detectó).
- Un frontend Next.js con `healthcheck: curl .../health` pero sin ningún route
  handler `/health` definido en el App Router → 404 constante, contenedor
  perpetuamente `unhealthy` **aunque la app funcionara perfecto** para tráfico
  real. `unhealthy` no significa "la app no sirve" — significa "el comando del
  healthcheck no dio el resultado esperado"; son cosas relacionadas pero no
  idénticas, y vale la pena chequear la app real (`curl /` o la ruta que
  corresponda) antes de asumir que está caída.

### 7.5 Gestor de paquetes: plataforma pineada vs. versión real de quien corre el comando

Un `composer.lock` (o cualquier lockfile equivalente) puede terminar con
paquetes que requieren una versión de runtime más nueva que la que declara el
proyecto (`composer.json: "php": "^8.2"`) y que usan los Dockerfiles (PHP 8.3),
simplemente porque alguien corrió `composer update`/`require` en una máquina con
una versión de PHP más nueva instalada localmente — sin un override explícito,
el resolver de dependencias resuelve contra el intérprete que lo está
ejecutando, no contra lo que declara el proyecto.

Fix estructural (no solo puntual): fijar la plataforma objetivo en la config del
gestor de paquetes, para que esto no vuelva a pasar sin importar qué versión
tenga instalada quien corra el comando:

```json
"config": { "platform": { "php": "8.3.7" } }
```

Y regenerar el lock contra esa plataforma pineada (usando un container con la
versión real de destino, no el intérprete local, para evitar el mismo problema
al revés).

---

## 8. Metodología: validar local antes de tocar el server

Con Docker Desktop (o cualquier Docker local) se puede reproducir el build del
server sin usar SSH, ahorrando vueltas completas de "buildear en el server,
falla, diagnosticar a ciegas con logs pegados por chat":

```bash
# build
docker build -f Dockerfile.prod -t <imagen>:test . [--build-arg ...]

# correrlo de verdad, no solo buildear
docker run -d --name test-run -p <puerto-local>:<puerto-app> <imagen>:test
sleep 4
docker logs test-run
curl -sv http://localhost:<puerto-local>/<ruta-de-health>
docker rm -f test-run
```

El paso de `docker run` + `curl` es el que importa — **un build exitoso no
garantiza que el contenedor arranque bien** (§7.2 y §7.3 son bugs que pasan el
build limpio y solo se ven corriendo el contenedor). Para diagnosticar un bug de
paths específicamente, correr un container efímero del stage intermedio y listar
el filesystem real suele ser más rápido que adivinar leyendo el Dockerfile:

```bash
docker build -f Dockerfile.prod --target <stage-intermedio> -t debug-image .
docker run --rm debug-image sh -c "find /app -iname '<archivo-que-buscas>'"
```

En Windows con Git Bash + Docker, si `docker run -v ...` o `-w ...` con paths
tipo `/app` fallan con errores de "invalid path" (`C:/Program Files/Git/app`),
es MSYS reescribiendo el path automáticamente — anteponer `MSYS_NO_PATHCONV=1`
al comando lo evita.

---

## 9. Acceso a la base de datos desde fuera del server

`mysql` (y en general, cualquier servicio de infraestructura que no necesite
exponerse) puede no tener `ports:` en el compose — en ese caso solo es
alcanzable desde otros contenedores en la misma red Docker, ni siquiera por VPN
al host. Dos formas de habilitar acceso externo con un cliente de escritorio:

- **Túnel SSH** (no expone ningún puerto nuevo del server): `ssh -L
  3306:127.0.0.1:3306 usuario@ip-server`, y conectar el cliente a
  `127.0.0.1:3306` en tu propia máquina — o usar el modo "connect over SSH"
  nativo del cliente (MySQL Workbench, DBeaver, etc. lo soportan), que evita
  mantener una terminal aparte abierta. Ojo con confundir el host del salto SSH
  (la IP del server) con el host de la base **después** del salto (`127.0.0.1`,
  desde el punto de vista del server).
- **Publicar el puerto directo** (`ports: "3306:3306"` en el compose): más
  simple, mismo modelo de exposición que el resto de los servicios de la app si
  ya están publicados igual y el único control de acceso real es la VPN — pero
  es una superficie más si después se refuerza el firewall.

---

## 10. Checklist de troubleshooting rápido

| Síntoma | Causas más probables | Dónde mirar |
|---|---|---|
| Build falla en un paso de `pecl`/`apt`/`apk install` | Falta una lib de sistema (headers `-dev`) que la extensión necesita para compilar | Mensaje de `configure: error` suele nombrar el paquete exacto |
| Build falla con "Missing environment variable" en medio del build de un frontend | Variable `NEXT_PUBLIC_*` no llegó como build arg | §6.2 |
| `failed to compute cache key: ... not found` | Casi siempre un path mal resuelto en `COPY --from=`, no cache corrupta — confirmar corriendo el stage intermedio antes de asumir que hay que limpiar cache | §7.1, §8 |
| Contenedor en `Restarting (1)` loop | Revisar `docker logs <contenedor>` — buscar `EACCES`/`Permission denied` (§7.3) o error de arranque de la app | — |
| Contenedor `Up` pero `(unhealthy)` | El proceso puede estar sirviendo bien — probar la ruta real de la app, no asumir que está caída. Si la ruta de health no existe o está mal versionada, es el healthcheck el que está mal, no la app | §7.4 |
| `command not found` raro al hacer `source` de un `.env` | CRLF o valor sin comillas con espacios | §6.5 |
| Dos servicios no logran autenticarse entre sí con un token/secreto compartido | Verificar que el valor coincida exactamente en ambos `.env` — típicamente diverge por edición manual en momentos distintos | §6.4 |

---

## 11. Pendientes de seguridad antes de considerar un deploy productivo

- Reactivar el firewall del host (`ufw` u equivalente) con reglas explícitas —
  no depender únicamente de que el server sea alcanzable solo por VPN.
- TLS: si en algún momento se pasa de IP+puerto a dominio, hay que revisar cada
  URL hardcodeada a `http://ip:puerto` (redirect URIs, CORS, `NEXT_PUBLIC_API_*`).
- Backup cifrado de los `.env` reales del server (no se versionan, así que su
  única copia vive ahí).
- Rotar cualquier secreto que haya quedado en valores de placeholder/test
  durante el primer deploy (tokens internos, API keys) antes de ir a producción
  real.
