# CEBAC: to-do de bitra en AWS

Los pasos en orden para [cebac-bitra-aws.md](./cebac-bitra-aws.md). Todo en
`sa-east-1` (salvo el certificado de CloudFront), en la cuenta de CEBAC. Lo
marcado con 💻 es código en los repos; lo marcado con 👤 lo hace una persona
en la consola; el resto se corre por CLI con el perfil `bitra-cebac`.

## 0. Antes de empezar

- [x] 👤 Crear el usuario IAM de deploy (quedó `bitra-ceabc`), sólo con acceso por CLI,
      con `PowerUserAccess` y
      [cebac-bitra-aws-deployer-policy.json](./cebac-bitra-aws-deployer-policy.json)
      como inline. Ver [Cómo se deploya](./cebac-bitra-aws.md#cómo-se-deploya).
- [x] 👤 `aws configure --profile bitra-cebac` (región `sa-east-1`).
- [x] `aws sts get-caller-identity --profile bitra-cebac`: es la cuenta del RDS
      y del bucket (340482984767).

- [x] El RDS: `healthbook-test`, la copia de producción (base `ebdb`). La
      contraseña está en `/bitra/cebac/api/db-password`.
- [x] Cognito: un pool nuevo, `cebac.bitra-care.com` (se crea en la fase 1).
- [x] DNS: la zona `bitra-care.com` está en Route 53 y delegada.
- [x] Las subidas van a un bucket propio, `bitra-cebac-uploads` (stack
      `base`).
- [x] 💻 lis-healthbook-back: `UPLOADS_DISK` y el disco `s3-uploads`.
- [ ] 👤 Activar los tags `Project`, `Customer` y `Environment` como cost
      allocation tags en Billing.
- [ ] Anotar la IP pública de salida de CEBAC (para filtrar `msg.`).

## 1. Base común

- [x] **ACM:** dos certificados para `cebac.bitra-care.com` y
      `*.cebac.bitra-care.com`, validados por DNS: uno en `sa-east-1` (ALB) y
      uno en `us-east-1` (CloudFront).
- [x] **ECR:** repositorios `bitra-web`, `bitra-api` y `lis-messaging`, con una
      lifecycle policy que conserve las últimas 10 imágenes.
- [x] **Security groups** en la VPC del RDS:
  - [x] `alb`: entrada 443 sólo desde la prefix list
        `com.amazonaws.global.cloudfront.origin-facing` y desde la IP de CEBAC;
  - [x] `tasks`: entrada sólo desde `alb` y desde sí mismo (Service Connect);
  - [x] `rds`: nada por ahora. El `default` de `healthbook-test` ya acepta
        3306 desde cualquier lado. Cerrarlo queda para "Después".
- [x] **ALB** con listener 443 (el certificado de `sa-east-1`). La acción por
      defecto es un 403 fijo: sólo pasa lo que coincide con una regla de host.
- [x] **Cluster de ECS** `bitra-cebac` (Fargate), con un namespace de Service
      Connect.
- [x] **CloudWatch Logs:** un log group por servicio, con retención de 30 días.
- [x] **Cognito:** stack `bitra-cebac-cognito` con el pool
      `cebac.bitra-care.com` (ver el README del back: username UUID con email y
      teléfono como alias, `hb_user_id`, sin auto-registro, *prevent user
      existence errors*) y un app client público con `CUSTOM_AUTH`.
- [x] **SSM:** los parámetros `SecureString` bajo `/bitra/cebac/`:
  - [x] `api/app-key` (`php artisan key:generate --show`);
  - [x] `api/db-password`;
  - [x] `messaging/internal-token` (el mismo que va a usar el LIS on-prem);
  - [x] `messaging/db-password`;
  - [x] `cloudfront/origin-verify` (el header secreto entre CloudFront y el
        ALB);
  - [x] `/healthbook/cebac/sign-in-lambda-token` (lo leen las Lambdas y el back).
- [x] **Execution role** de ECS: pull de ECR, escritura de logs y lectura de
      `/bitra/cebac/*` y `/healthbook/cebac/*` en SSM.

Hecho con las plantillas de `lis-infra/aws/bitra-cebac/` (rama `deploy/cebac`):
`./deploy.sh <stack> review|apply` y `./push-image.sh api|web`.

## 2. lis-messaging en AWS (sólo para bitra)

- [x] 💻 Dockerfile en `node:22`.
- [x] **RDS:** base `lis_messaging` y su usuario.
- [x] **SES:** verificar el dominio `cebac.bitra-care.com` (DKIM en Route 53) y
      definir el remitente (`EMAIL_FROM`).
- [x] **SES:** la cuenta ya tiene acceso de producción en `sa-east-1` (sin
      sandbox).
- [x] 💻 lis-messaging-service: `healthbook.sign_in_code` y
      `healthbook.access_code` en el catálogo, con sus mails (sólo email).
- [ ] 💻 lis-messaging-service: el resto de las comunicaciones de bitra
      (bienvenida, invitación, cuenta, turnos, presupuestos).
- [x] **Stack SAM** `lis-messaging-service/aws/template.yaml` (colas de eventos
      de SES y Meta, webhook de WhatsApp, configuration set). Revisar si ya
      está deployado.
- [x] **Task role:** `ses:SendEmail` y `ses:SendRawEmail`, más leer y borrar
      mensajes de las colas del stack.
- [x] **Task definition y service** `lis-messaging`: puerto 3040, publicado en
      Service Connect como `lis-messaging`, con `NODE_ENV=production`,
      `EVENTS_ENABLED=false` y `TEMPLATES_CUSTOMER=cebac`.
- [x] **Health check:** `GET /health` responde 200 (fuera del prefijo `/api/v1`).

## 3. bitra-api

- [x] 💻 Dockerfile (`serversideup/php:8.3-fpm-nginx`).
- [x] **Task role:** `s3:GetObject` sobre el bucket de documentos de v1 (sólo
      lectura: el bucket no se toca), lectura y escritura sobre
      `bitra-cebac-uploads`, y los
      9 permisos `cognito-idp:Admin*` sobre el pool nuevo.
- [x] **Task definition** con las variables de
      [bitra-api](./cebac-bitra-aws.md#bitra-api).
- [x] **Migraciones:** `migrate:status` en una task de una sola corrida: no
      hay nada pendiente (la copia ya tiene el esquema de v2).
- [ ] **Cuentas de prueba en el pool nuevo:**
      `php artisan identity:provision-legacy --user=…` con unos pocos
      pacientes, no con todos.
- [ ] **Limpieza de usuarios no verificados (corte):**
      `php artisan users:purge-unverified --dry-run` y después sin `--dry-run`,
      sobre la copia definitiva. Borra los pacientes sin canal verificado con
      sus vínculos a fichas; criterio `User::hasVerified` (cuenta el registro
      por WhatsApp de v1). Antes, borrarles el `cognito_sub` del pool viejo a
      los 11 usuarios de prueba.
- [ ] **Alta silenciosa de los registrados (corte):**
      `php artisan identity:provision-legacy --dry-run` y después completo, en
      una task de ECS (decenas de miles de llamadas a Cognito). Resolver a
      mano los `AliasExistsException` (contacto repetido entre usuarios) que
      liste al final.
- [ ] **Plantilla de autenticación de Meta** para `healthbook.sign_in_code` y
      `healthbook.access_code`: sin ella los códigos salen sólo por email
      (ver `catalog.ts` del messaging).
- [x] 💻 La IP real del cliente: confiar en la red del ALB y tomar la IP del
      header `CloudFront-Viewer-Address`. Sin esto, los límites por IP se
      comparten entre todos los pacientes.
- [x] **Service** `bitra-api` con un target group en el puerto 8080 y health
      check (`/up`), más la regla del ALB por host `api.cebac.bitra-care.com`
      que exige `X-Origin-Verify`.
- [x] **CloudFront** del api: origen el ALB por HTTPS, `CachingDisabled`,
      `AllViewerAndCloudFrontHeaders-2022-06`, el header `X-Origin-Verify` y
      el certificado de `us-east-1`.
- [x] **DNS:** alias `api.cebac.bitra-care.com` → la distribución.
- [x] **Probar:** `curl https://api.cebac.bitra-care.com/up` responde 200.

## 4. Lambdas de Cognito

Ver `lis-healthbook-back/aws/cognito-auth/README.md`.

- [x] `sam build` y `sam deploy` con el stack `healthbook-cebac-cognito-auth`,
      `UserPoolId=…` y `HealthbookUrl=https://api.cebac.bitra-care.com`.
- [x] `describe-user-pool` y guardar la salida.
- [x] `update-user-pool --lambda-config …`, repitiendo lo que haya que
      conservar.
- [x] `describe-user-pool-client` y, después, `update-user-pool-client` con
      `ALLOW_CUSTOM_AUTH` además de los flujos que ya tiene.
- [ ] **Probar:** ingreso por código con un email verificado en SES.
- [ ] **Antes de probar con datos reales:** prender la setting `messagingSandbox`
      en Configuración de bitra (y cargar nuestros contactos en
      `messagingSandboxAllowlist`), para que nada le llegue a los pacientes:
      los mensajes quedan en Administración > Mensajes retenidos. Apagarla
      antes de abrir a pacientes.

## 5. bitra-web

- [x] 💻 Dockerfile (`turbo prune` + Next standalone).
- [x] Build con las variables de
      [bitra-web](./cebac-bitra-aws.md#bitra-web) y push como
      `bitra-web:cebac-<sha>`.
- [x] **Service** `bitra-web` con un target group en el puerto 3000, más la
      regla del ALB por host `cebac.bitra-care.com` que exige `X-Origin-Verify`.
- [x] **CloudFront** del web: origen el ALB por HTTPS, `AllViewer`,
      `/_next/static/*` con `CachingOptimized` y el resto con
      `CachingDisabled`, el header `X-Origin-Verify` y el certificado de
      `us-east-1`.
- [x] **DNS:** alias `cebac.bitra-care.com` → la distribución.
- [ ] **Probar:** la landing de CEBAC, el login con contraseña y con código, y
      ver un PDF de resultados.

## 6. APK de Android

- [ ] 💻 Elegir local o EAS. Para EAS: `buildType: apk` en el perfil
      `preview`.
- [ ] Build con las variables de
      [APK de Android](./cebac-bitra-aws.md#apk-de-android).
- [ ] Desinstalar el dev build (o compilar con un sufijo en el package) e
      instalar el APK.
- [ ] **Probar:** login, resultados, PDF, órdenes médicas y fila virtual.

## 7. El LIS de CEBAC hacia bitra

- [ ] Crear el cliente de integración:
      `php artisan integration:client-create lis-adapter-healthbook arenco`
      (como task de una sola corrida).
- [ ] En `lis-infra/.env` de CEBAC: `HEALTHBOOK_MODE=http`,
      `HEALTHBOOK_API_URL=https://api.cebac.bitra-care.com` y
      `HEALTHBOOK_API_TOKEN=…`. Después, `up -d adapter-healthbook`.
- [ ] Prender `MEDICAL_ORDER_PULL_ENABLED` y `VIRTUAL_QUEUE_SYNC_ENABLED`, y
      hacer `up -d scheduler`.
- [ ] **Probar:** una orden del LIS aparece en bitra, y un pedido de la fila
      desde la app emite el ticket.

## 8. Messaging: corte de CEBAC a AWS

- [ ] 💻 lis-messaging-service: el consumidor SQS de eventos del LIS, en lugar
      del de Redis.
- [ ] 💻 `aws/template.yaml`: la cola FIFO `lis-events` con su DLQ.
- [ ] 💻 lis-backend: el comando del relay (Redis → SQS), en el scheduler y
      detrás de un flag.
- [ ] Deployar el stack con la cola FIFO.
- [ ] 👤 **IAM:** sumarle a `cebac-lis-onprem` `sqs:SendMessage` sobre la cola
      FIFO. El deployer no puede tocar usuarios.
- [ ] **Task role de messaging:** leer y borrar mensajes de la cola FIFO.
- [ ] **Exponer messaging:** target group en el 3040 y regla del ALB por host
      `msg.cebac.bitra-care.com` con la condición `source-ip` = la IP de CEBAC
      (el security group solo no alcanza, porque también deja pasar a
      CloudFront). Alias de DNS directo al ALB, sin CloudFront.
- [ ] **Probar desde el server de CEBAC:**
      `curl https://msg.cebac.bitra-care.com/health`, y que desde otra IP no
      conteste.
- [ ] **Ventana de corte:**
  - [ ] parar `messaging-service` en CEBAC;
  - [ ] `mysqldump lis_messaging` → RDS;
  - [ ] `LIS_MESSAGING_SERVICE_BASE_URL=https://msg.cebac.bitra-care.com` en el
        backend y `up -d backend scheduler`;
  - [ ] prender el relay y `EVENTS_ENABLED=true` en la task de messaging.
- [ ] **Probar:** enviar una factura desde caja, ver la pestaña Comunicaciones
      y el aviso de una orden nueva.
- [ ] 💻 Sacar `messaging-service` del compose de `deploy/cebac` y escribir la
      nota de release.

## Después

- [ ] 👤 WhatsApp: el número y las plantillas en Meta, y decidir qué app usa
      el LIS.
- [ ] WAF en la distribución del api, con rate limit sobre los endpoints de
      códigos.
- [ ] Pasar `bitra-api` y `lis-messaging` a 2 tasks.
- [ ] Subnets privadas con NAT o VPC endpoints.
- [ ] Cerrar el 3306 de `healthbook-test` a las tasks y a las IPs de
      desarrollo.
- [ ] Un pipeline de deploy (GitHub Actions: build, push a ECR y
      `update-service`).
