# CEBAC: bitra en AWS

Estrategia de deploy de **bitra cebac** (el portal de pacientes, ex Healthbook)
en la cuenta de AWS de CEBAC, con el dominio `cebac.bitra-care.com`, y del APK de
Android para probarlo. Los pasos concretos están en
[cebac-bitra-aws-todo.md](./cebac-bitra-aws-todo.md).

Ya está resuelto y no se toca acá:

- **La base** está en RDS.
- **Los PDF** están en S3. Ver [cebac-s3-documents.md](./cebac-s3-documents.md).

## La cuenta, tal como está

Relevado el 2026-10-13 con el perfil `bitra-cebac`. La cuenta es la
340482984767, en `sa-east-1`.

- **La cuenta es compartida** con healthbook v1, metrocuba, deliverybook,
  ankra-lis y arenco-it. Todo lo de este deploy lleva el prefijo
  `bitra-cebac-*` y los tags de abajo. No se toca nada más.
- **RDS.** bitra usa `healthbook-test`: es una copia de producción del
  2026-10-07, con MySQL 8.0, la base `ebdb` y el usuario `healthbook`. Ya
  tiene el esquema de v2, porque es la misma base que usa el back en
  desarrollo local. La contraseña está en `/bitra/cebac/api/db-password`.
  - **`healthbook-cebac-pr` es la producción de v1** (Elastic Beanstalk
    `cebac-pr`). No se toca.
  - La copia está en la VPC default (172.31.0.0/16, tres subnets públicas)
    con el security group `default`, que acepta 3306 desde 0.0.0.0/0. Así se
    conecta el desarrollo local. Las tasks entran sin cambios. Cerrarlo queda
    para después, junto con las IPs de desarrollo.
- **El bucket de documentos lo usa v1 en producción.** No se toca: ni su
  configuración, ni su política, ni sus objetos. bitra-api lo lee con su task
  role.
- **DNS.** La zona `bitra-care.com` está en Route 53 y delegada. Los registros
  de `cebac.bitra-care.com` van ahí.
- **Cognito.** Un pool nuevo, `cebac.bitra-care.com`, en su propio stack
  (`bitra-cebac-cognito`), con protección contra borrado. Los pools que ya
  existen (`healthbook-cebac-test`, los de dev) no se usan.

**Tags**, en todos los stacks (CloudFormation los propaga, y ECS con
`propagateTags`):

```
Project=bitra   Customer=cebac   Environment=test   ManagedBy=cloudformation
```

Lo que se crea a mano lleva `ManagedBy=manual`. Para ver los costos por tag
hay que activarlos como *cost allocation tags* en Billing.

Este documento cubre el front, el back, las Lambdas de Cognito y
lis-messaging-service. Messaging deja de correr en el server de CEBAC y pasa a
AWS: **una sola instancia** para el LIS y para bitra.

## Arquitectura

Todo va en `sa-east-1`, en la VPC del RDS.

```
cebac.bitra-care.com      ─► CloudFront (cache de estáticos) ─┐
api.cebac.bitra-care.com  ─► CloudFront (sin cache)          ─┼─► ALB, ruteo por host
msg.cebac.bitra-care.com  ─► (directo, sólo la IP de CEBAC)  ─┘     │
                                                                     ├─► bitra-web      Next, :3000
                                                                     ├─► bitra-api      Laravel, :8080 ─► RDS, S3 (PDF)
                                                                     └─► lis-messaging  NestJS, :3040  ─► RDS, SES, Meta

ECS Fargate, cluster bitra-cebac:
  bitra-api ──Service Connect──► lis-messaging
  lis-messaging ◄── SQS (eventos de SES, de Meta y del LIS)

Cognito user pool ─► 3 Lambdas (SAM) ─► https://api.cebac.bitra-care.com/api/v1/internal/...

CEBAC on-prem:
  LIS backend         ─HTTPS─► msg.cebac.bitra-care.com  (facturas, indicaciones, Comunicaciones)
  adapter-healthbook  ─HTTPS─► api.cebac.bitra-care.com  (pacientes, órdenes, fila; push y pull)
  Redis lis:events:messaging ─► relay ─► SQS FIFO ─► lis-messaging
```

Todo el tráfico entre CEBAC y AWS **sale** de CEBAC. No se abre nada hacia el
server on-prem.

## Dominios

| Host | Entrada | Destino | Acceso |
| --- | --- | --- | --- |
| `cebac.bitra-care.com` | CloudFront | bitra-web | Público |
| `api.cebac.bitra-care.com` | CloudFront | bitra-api | Público. Lo usan el web, la app, las Lambdas y el LIS |
| `msg.cebac.bitra-care.com` | ALB directo | lis-messaging | Sólo la IP pública de salida de CEBAC |

bitra-api le habla a messaging por la red interna (ECS Service Connect), no
por `msg.`.

**Certificados de ACM.** Los dos son para `cebac.bitra-care.com` y
`*.cebac.bitra-care.com`, validados por DNS:

- uno en `us-east-1`, porque CloudFront sólo acepta certificados de esa región;
- uno en `sa-east-1`, para el ALB.

## CloudFront

Dos distribuciones, con el mismo ALB como origen:

- **`cebac.bitra-care.com` (web).**
  - `/_next/static/*` y los archivos de `public/` se cachean en el borde (hay
    uno en Buenos Aires). Next les pone un hash en el nombre, así que no hay
    que invalidar nada en cada deploy.
  - El resto pasa sin cache, con la política de pedido al origen `AllViewer`.
- **`api.cebac.bitra-care.com` (api).**
  - Sin cache (`CachingDisabled`), con todos los headers, cookies y query
    strings, incluido `Authorization`. La política de pedido al origen es
    `AllViewerAndCloudFrontHeaders-2022-06`, que además agrega
    `CloudFront-Viewer-Address` (ver la IP del cliente, abajo).
  - Sirve para dejar el ALB cerrado a internet y para tener un lugar donde
    poner WAF.

Las dos políticas de pedido al origen le pasan al ALB el `Host` del pedido,
que es lo que usa para rutear.

**Messaging no pasa por CloudFront.** Detrás de CloudFront, el ALB vería IPs
de CloudFront y no podría filtrar por la de CEBAC.

**El ALB queda cerrado a internet.**

- Su security group acepta 443 sólo de la prefix list administrada de
  CloudFront (`com.amazonaws.global.cloudfront.origin-facing`) y de la IP de
  CEBAC.
- Las dos distribuciones le mandan un header secreto (`X-Origin-Verify`,
  guardado en SSM). Las reglas del ALB para `cebac.` y `api.` lo exigen, así
  nadie llega al ALB desde otra distribución de CloudFront.
- CloudFront le habla al ALB por HTTPS. El certificado del ALB cubre los dos
  hosts.

**La IP del cliente.** Detrás de CloudFront y del ALB, Laravel ve la IP del
balanceador, y hoy el back no tiene configurados los proxies de confianza. Así,
los límites por IP (por ejemplo, 5 pedidos de código por minuto) se
compartirían entre todos los pacientes. Hay que hacer dos cosas en el back:

- confiar en la red del ALB;
- tomar la IP del cliente del header `CloudFront-Viewer-Address`, que la
  distribución del api agrega en el pedido al origen.

**WAF** queda opcional, por unos 10 USD por mes: una regla de rate limit sobre
los endpoints de códigos, en el borde.

## Fargate: una imagen por servicio

| Imagen (ECR) | Qué es | Estado |
| --- | --- | --- |
| `bitra-web` | Next 16 con `output: "standalone"`, un proceso Node | Falta el Dockerfile (monorepo pnpm: `turbo prune`) |
| `bitra-api` | Laravel 12 sobre PHP 8.3, con nginx y php-fpm en la misma imagen (`serversideup/php:8.3-fpm-nginx`) | Falta el Dockerfile |
| `lis-messaging` | NestJS | El Dockerfile existe; pasar a `node:22` |

- **El web es por cliente.** Los `NEXT_PUBLIC_*` se fijan al construir
  (`NEXT_PUBLIC_BITRA_CLIENT=cebac`, las URLs, el app client de Cognito), así
  que el tag lo dice: `bitra-web:cebac-<sha>`.
- **El back no tiene estado.** No usa colas (`QUEUE_CONNECTION=sync`), ni
  scheduler, y la sesión es `array`. Una task alcanza y escala horizontal.
- **Las migraciones** van como una task de una sola corrida con la imagen de
  `bitra-api` y el comando `php artisan migrate --force`. No corren al
  arrancar el contenedor.
- **Messaging** corre en un solo proceso: la API, el outbox, los pollers de
  SQS y el consumidor de eventos del LIS. Con dos tasks no hay mensajes
  repetidos, porque el outbox reparte el trabajo con leases
  (`OUTBOX_LEASE_MS`).
- **Para arrancar:** 1 task por servicio, de 0,25 vCPU y 0,5 GB. Pasar el back
  y messaging a 2 tasks cuando haya tráfico real, para que un deploy no corte
  el servicio.
- **Deploy:** build, push a ECR, nueva revisión de la task definition y
  `aws ecs update-service`. ECS hace un rolling update.

**La red.** Las tasks necesitan salida a internet: ECR, Cognito, SES, SQS,
Meta. Para el test van en subnets públicas, con IP pública y un security
group que sólo acepta tráfico del ALB, sin NAT Gateway (que cuesta unos
35 USD por mes). Lo que acepta el ALB está en [CloudFront](#cloudfront). El
RDS sigue privado y acepta conexiones sólo desde el security group de las
tasks. Más adelante se puede pasar a subnets privadas con NAT o con VPC
endpoints.

**Secretos.** Van en SSM Parameter Store (`SecureString`) bajo
`/bitra/cebac/...`, y la task definition los inyecta como variables. No hay
access keys en las tasks: cada servicio tiene su task role.

## bitra-api

Variables que cambian respecto del desarrollo:

```
APP_ENV=production
APP_URL=https://api.cebac.bitra-care.com
FRONT_URL=https://cebac.bitra-care.com
CORS_ALLOWED_ORIGINS=https://cebac.bitra-care.com
DB_HOST=healthbook-test.ccf1edaxmims.sa-east-1.rds.amazonaws.com
DB_DATABASE=ebdb                  DB_USERNAME=healthbook
DB_PASSWORD=…                     (SSM: /bitra/cebac/api/db-password)
AUTH_TOKEN_DRIVER=cognito
IDENTITY_DRIVER=cognito
COGNITO_REGION=sa-east-1
COGNITO_USER_POOL_ID=…            COGNITO_CLIENT_ID=…
DOCUMENTS_DISK=s3                 AWS_BUCKET=…   (el de v1: sólo lectura; sin access keys, task role)
UPLOADS_DISK=s3-uploads           UPLOADS_AWS_BUCKET=bitra-cebac-uploads
MESSAGING_DRIVER=http
MESSAGING_URL=http://lis-messaging:3040   (Service Connect)
MESSAGING_INTERNAL_TOKEN=…        (SSM)
SIGN_IN_LAMBDA_TOKEN=…            (SSM, el mismo parámetro que leen las Lambdas)
```

Con `APP_ENV=production` el back no arranca con `MESSAGING_DRIVER=log`,
`IDENTITY_DRIVER=log` ni `AUTH_TOKEN_DRIVER=local`. Tampoco existe el buzón de
desarrollo: **los códigos salen de verdad**. Ver
[Mensajería en la etapa de test](#mensajería-en-la-etapa-de-test).

Task role:

- **S3:** lectura y escritura sobre `hb/*`, lectura sobre `lis/*`,
  `ListBucket` con los dos prefijos y la clave KMS del bucket. Ver
  [cebac-s3-documents.md](./cebac-s3-documents.md#2-permisos).
- **Cognito sobre el pool:** `AdminCreateUser`, `AdminGetUser`,
  `AdminSetUserPassword`, `AdminUpdateUserAttributes`,
  `AdminDeleteUserAttributes`, `AdminEnableUser`, `AdminDisableUser`,
  `AdminUserGlobalSignOut` y `AdminDeleteUser`.

## bitra-web

Variables de build:

```
NEXT_PUBLIC_API_URL=https://api.cebac.bitra-care.com/api/v1
NEXT_PUBLIC_AUTH_MODE=cognito
NEXT_PUBLIC_COGNITO_REGION=sa-east-1
NEXT_PUBLIC_COGNITO_CLIENT_ID=…
NEXT_PUBLIC_BITRA_CLIENT=cebac
```

La CSP de `next.config.ts` arma el `connect-src` con el origen de la API y el
de Cognito, así que tampoco hay nada que tocar a mano en runtime.

## Lambdas de Cognito

Ya existen en `lis-healthbook-back/aws/cognito-auth` (SAM, Node 22, arm64), y
el procedimiento completo está en su README. Para CEBAC:

- `HealthbookUrl=https://api.cebac.bitra-care.com`.
- El token va en `/healthbook/cebac/sign-in-lambda-token` (el default del
  template).

Van **después** del back, porque necesitan su URL pública. Al enganchar los
triggers al pool, `update-user-pool` pisa lo que no se le pasa, así que hay que
correr `describe-user-pool` antes.

## lis-messaging-service, centralizado en AWS

Hoy corre en el compose de CEBAC (`lis-infra`, rama `deploy/cebac`), con la
base `lis_messaging` en el MySQL local. Pasa a ser un servicio de ECS que
atiende a los dos:

- **bitra-api:** por la red interna (los OTP y los avisos del portal).
- **El LIS on-prem:** por `https://msg.cebac.bitra-care.com` (facturas,
  indicaciones y la pestaña Comunicaciones). Se protege con el
  `X-Internal-Token` y con una condición `source-ip` (la IP de CEBAC) en la
  regla del ALB para `msg.`. No alcanza con el security group: ese también
  deja entrar a CloudFront.

Con una sola instancia también se resuelve el webhook de WhatsApp: hay un
único consumidor de los eventos de Meta.

### Los avisos automáticos pasan a SQS

Los avisos de "orden creada" y "muestras tomadas" hoy salen del stream
`lis:events:messaging` del Redis on-prem, que messaging lee con un consumer
group. Desde AWS no llega a ese Redis, así que quedan así:

```
LIS backend ─XADD─► Redis lis:events:messaging ─► relay (scheduler del LIS) ─SendMessage─► SQS FIFO ─► lis-messaging
                    (outbox local)                  XACK después del envío                   + DLQ
```

- **El emisor del LIS no cambia.** `PatientMessagingEventEmitter` sigue
  escribiendo en Redis después del commit. Redis hace de outbox local: si se
  corta internet en CEBAC, los eventos esperan ahí y no se pierden.
  - Publicar directo a SQS desde el listener los perdería: el emisor loguea el
    error y sigue, y on-prem no hay worker de colas que reintente.
- **El relay** es un comando nuevo de lis-backend que corre en el contenedor
  `scheduler`, detrás de un flag. Lee el stream con su propio consumer group,
  manda a SQS y recién ahí hace el XACK.
- **La cola es FIFO.**
  - `MessageGroupId` = la orden: "orden creada" llega antes que "muestras
    tomadas".
  - `MessageDeduplicationId` = `event_id`: si el relay se corta entre el envío
    y el XACK, el reenvío no duplica el aviso. Igual el servicio tiene que
    ignorar un `event_id` repetido (verificarlo).
  - Una DLQ con redrive después de N intentos reemplaza a `<stream>:dead`.
- **Del lado de messaging**, un consumidor SQS con el `SqsQueuePoller` que ya
  usan SES y Meta reemplaza al de Redis. El evento se arma con el mismo
  `parseMessagingEvent`, así que la lógica de los avisos no cambia. La cola va
  en `lis-messaging-service/aws/template.yaml`.
- **Credenciales del relay:** el usuario IAM que el LIS on-prem ya usa para S3,
  Textract y Bedrock (`cebac-lis-onprem`), sumándole `sqs:SendMessage` sobre
  esa cola.

### Mensajería en la etapa de test

Con `APP_ENV=production` y `NODE_ENV=production` no hay buzón de desarrollo ni
códigos en pantalla. Para probar el ingreso por código:

- **Email por SES real.** Hay que verificar la identidad del remitente (el
  dominio `cebac.bitra-care.com`, con DKIM). Mientras la cuenta esté en el sandbox
  de SES, cada tester tiene que tener su dirección verificada. Para abrirlo a
  pacientes hay que pedir el acceso de producción.
- **WhatsApp** se prende cuando estén el número y las plantillas aprobadas en
  Meta. Falta decidir qué app y qué número de Meta usa el LIS: hoy bitra ya
  tiene su app con webhook.

### El corte en CEBAC

1. Messaging en AWS, primero sólo para bitra, con `EVENTS_ENABLED=false`.
2. El relay y la cola FIFO andando. Probar con `MESSAGING_EVENTS_ENABLED`
   apagado en CEBAC y eventos de prueba.
3. **Ventana de corte:**
   - parar el `messaging-service` de CEBAC;
   - `mysqldump` de `lis_messaging` al RDS (así la pestaña Comunicaciones
     conserva la historia);
   - `LIS_MESSAGING_SERVICE_BASE_URL=https://msg.cebac.bitra-care.com` en el
     backend;
   - prender el relay y `EVENTS_ENABLED` en AWS.
4. Sacar `messaging-service` del compose de `deploy/cebac` y escribir la nota
   de release.

**Vuelta atrás:** volver a levantar el contenedor de CEBAC con la URL vieja.
Lo que se mandó desde AWS en el medio queda sólo en el RDS.

## El LIS hacia bitra

Todo lo que el LIS le manda a bitra o le pide pasa por el contenedor
`adapter-healthbook`. Hoy está en `stub` (`push-cebac-env.sh`). Cuando bitra
esté andando, en `lis-infra/.env` (rama `deploy/cebac`):

```
HEALTHBOOK_MODE=http
HEALTHBOOK_API_URL=https://api.cebac.bitra-care.com     (sin /api/v1)
HEALTHBOOK_API_TOKEN=…                             (el del cliente de integración)
MEDICAL_ORDER_PULL_ENABLED=true
VIRTUAL_QUEUE_SYNC_ENABLED=true
```

El cliente de integración se crea en bitra con una task de una sola corrida:
`php artisan integration:client-create lis-adapter-healthbook arenco`. Sin
`--scope` recibe todos los scopes, incluidos los de órdenes y los de la fila.

## APK de Android

La app es `lis-healthbook-front/apps/mobile` (Expo). Las `EXPO_PUBLIC_*`
quedan dentro del bundle, así que se fijan al construir:

```
EXPO_PUBLIC_API_URL=https://api.cebac.bitra-care.com/api/v1
EXPO_PUBLIC_AUTH_MODE=cognito
EXPO_PUBLIC_COGNITO_REGION=sa-east-1
EXPO_PUBLIC_COGNITO_CLIENT_ID=…
EXPO_PUBLIC_BITRA_CLIENT=cebac
EXPO_PUBLIC_WEB_URL=https://cebac.bitra-care.com
```

Hay dos caminos:

- **Local (para arrancar).** `expo prebuild` y `gradlew assembleRelease`, con
  esas variables en la shell; pisan el `.env` de desarrollo, que apunta a
  `10.0.2.2`. Usa el toolchain que ya está instalado (JDK 17, CMake 3.31,
  rutas largas). Sale firmado con el keystore de debug, que alcanza para
  instalarlo en celulares de prueba.
- **EAS Build.** Sumar `"android": { "buildType": "apk" }` al perfil `preview`
  de `eas.json` y correr `eas build -p android --profile preview`. Necesita una
  cuenta de Expo y da un link o QR para pasarles a los testers.

**La firma.** El dev build de desarrollo usa el mismo package
(`com.arenco.healthbook`) con otra firma, así que el APK no se instala encima.
Hay que desinstalar el de desarrollo o, para el test, compilar con un sufijo en
el package para tener las dos apps a la vez.

## Cómo se deploya

Todo se hace por CLI con un usuario IAM propio, `bitra-ceabc` (sic), y el
perfil `bitra-cebac` (región `sa-east-1`). Nada se crea a mano en la consola:

- **La infraestructura** va en plantillas de CloudFormation/SAM versionadas en
  los repos. Cada cambio pasa por un change set que se revisa antes de
  aplicarlo.
- **Las imágenes:** build, push a ECR y `aws ecs update-service`.

Permisos del usuario:

- **`PowerUserAccess`** (administrada por AWS): todo menos IAM y
  Organizations.
- **[cebac-bitra-aws-deployer-policy.json](./cebac-bitra-aws-deployer-policy.json)**,
  como inline. Le permite crear roles y políticas sólo con los prefijos de los
  stacks (`bitra-cebac-*`, `healthbook-cebac-*`, `lis-messaging-cebac-*`) y los
  service-linked roles de ECS y del ALB.
  - No puede tocar usuarios. Por eso el `sqs:SendMessage` del usuario
    on-prem `cebac-lis-onprem` lo carga una persona, una vez.
- La cuenta se comparte con otros proyectos, y `PowerUserAccess` también los
  alcanza. Si hace falta un candado, se le puede poner un permissions
  boundary al usuario.

Las claves del usuario no se comparten: quien lo crea configura el perfil con
`aws configure --profile bitra-cebac`.

## Lo que falta decidir

- **Si las subidas vuelven al bucket de documentos** cuando v1 se apague. Hoy
  van a `bitra-cebac-uploads` (ver bitra-api). Mientras tanto, para la
  sincronización de órdenes médicas el LIS va a necesitar lectura sobre ese
  bucket.
- **La base compartida con el desarrollo local:** las migraciones que se
  corran en un lado afectan al otro.
- **La app y el número de Meta** que usa el LIS para WhatsApp.
- **La región:** son datos de salud en `sa-east-1`. Ver la nota de legales en
  [cebac-s3-documents.md](./cebac-s3-documents.md#lo-que-hay-que-decidir-antes).

## Costo aproximado del test

| Ítem | USD por mes |
| --- | --- |
| ALB | ~20 |
| 3 tasks de Fargate de 0,25 vCPU y 0,5 GB | ~30 |
| CloudFront (el free tier cubre 1 TB y 10 millones de requests) | ~0 |
| Lambdas, SQS, SSM, ECR | Centavos |
| WAF (opcional) | ~10 |
| NAT Gateway (si se pasa a subnets privadas) | ~35 |

El RDS y el S3 ya existen y no están en la cuenta.
