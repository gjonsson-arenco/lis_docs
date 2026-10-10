# Healthbook: empadronamiento asistido desde la admisión e ingreso por código (OTP)

Brief para implementar, en `c:\Projects\LIS\lis-healthbook\lis-healthbook-back`
(y en su front y su infraestructura de Cognito), lo que hace falta para que la
admisión del LIS deje al paciente con acceso a Healthbook. El lado del LIS
(backend, adapter y front) **ya está hecho** y se describe al final como
contexto: este documento es el contrato que Healthbook tiene que cumplir.

Leer antes de empezar: la sección "Empadronamiento y recupero de contraseña"
del README de Healthbook, `App\Services\Enrollment\EnrollmentService`,
`App\Services\Integration\AccountLookup` y `App\Identity\CognitoIdentityProvider`.

## Por qué

El paso que más traba el empadronamiento hoy no es el canal, es la identidad:
después del código, al paciente que todavía no está en Cognito se le pide DNI
y fecha de nacimiento, y además la ficha tiene que llevar ese mismo contacto o
estar ya vinculada al usuario. Un teléfono nuevo o un mail mal cargado en la
ficha vieja lo dejan afuera.

En la admisión el recepcionista tiene el DNI en la mano y acaba de confirmar
el contacto con el paciente. Puede dar fe de la identidad. Healthbook ya tiene
esa figura para los administradores (`EnrollmentService::invite`: pedido ya
verificado, sin paso de identidad). El **empadronamiento asistido** es esa
misma figura, emitida por la integración, con el código leído cara a cara como
prueba del canal.

Segundo problema: hoy no existe ingreso sin contraseña. El código de 6 dígitos
es el primer paso del empadronamiento o del recupero, no un login. Un paciente
que se va de la admisión con el canal verificado pero sin contraseña no entra.
Por eso este brief trae dos piezas: el empadronamiento asistido (Healthbook,
API de integración) y el **ingreso por código** (Cognito `CUSTOM_AUTH` con sus
Lambdas, más un par de endpoints internos de Healthbook).

## Qué hay que construir

1. **Empadronamiento asistido** en la API de integración: dos endpoints
   (`POST .../enrollments` y `POST .../verify`), un emisor nuevo en
   `EnrollmentRequest`, la variante de `complete()` sin contraseña, un scope
   nuevo para el cliente de integración y un bloque `enrollment` en la
   consulta de cuenta.
2. **Ingreso por código (OTP)**: `ALLOW_CUSTOM_AUTH` en el app client, tres
   Lambdas (`DefineAuthChallenge`, `CreateAuthChallenge`,
   `VerifyAuthChallengeResponse`), dos endpoints internos de Healthbook que las
   Lambdas llaman, y la pantalla en el front.
3. Mensajes: el código asistido reusa `Communication::AccessCode`; hace falta
   una bienvenida nueva que explique las dos formas de entrar.

Las dos piezas son independientes: el empadronamiento asistido sirve aunque el
OTP no exista todavía (el paciente entra por "ingresar con un código", que
para una cuenta ya en Cognito es el recupero de contraseña, sin paso de
identidad). Conviene hacer primero la 1.

## Pieza 1: empadronamiento asistido

### La figura

Un `EnrollmentRequest` con:

| Campo | Valor |
|---|---|
| `issued_by` | `assisted` (caso nuevo de `EnrollmentIssuer`) |
| `mode` | `enrollment` si el usuario resuelto no tiene `cognito_sub`; `password_reset` no aplica: ver "qué pasa al verificar" |
| `identity_required` | `false`: el recepcionista vio el DNI |
| `patient_ids` | `[id de la ficha]`, resuelta por `external_id` del proveedor del cliente |
| `user_id` | el dueño del contacto si ya es de un usuario habilitado; si no, el usuario adoptable del paciente (`adoptableUser`); si no, `null` |
| `code_hash`, `expires_at` | como en `issueCode`: código de 6 dígitos, `code_ttl_minutes` |
| `created_by_user_id` | `null`: el operador no es un usuario de Healthbook. Va en la auditoría |

El código viaja por `Messenger::sendTo(Communication::AccessCode, …)` al
contacto, exactamente como el flujo público. El paciente lo lee en su teléfono
delante del recepcionista, que lo carga en el LIS.

### Qué pasa al verificar

Al validar el código, el pedido **se completa en el acto**, sin paso de
contraseña (`completeAssisted()`, una variante de `complete()`):

1. Se crea o se reusa la fila de `users` con las mismas reglas que
   `complete()`: la existente siempre se reusa; sin usuario, uno nuevo con rol
   paciente, `enabled = true`, nombre de la ficha.
2. El canal probado pasa a ser el contacto del usuario y queda verificado
   (`email`/`whatsapp` + `*_verified_at`). La contraseña del sistema anterior
   se borra (`password = null`), como hoy.
3. La ficha se vincula al usuario (`user_patient`, `syncWithoutDetaching`).
4. **Se provisiona la cuenta en Cognito** aunque no haya contraseña:
   `provision()` y, si el usuario no la tenía, una contraseña **al azar y
   permanente** (`adminSetUserPassword` con `Permanent = true`), que no se
   muestra ni se guarda. Sin esto la cuenta queda en `FORCE_CHANGE_PASSWORD` y
   el `CUSTOM_AUTH` no la deja pasar. Si el usuario ya estaba en Cognito, no se
   toca la contraseña: sólo `updateContact()`.
5. `completed_at = now()`, auditoría `enrollment.assisted_completed`, y la
   bienvenida (ver "Mensajes").

Resultado: `enrolled = true`, `registered = true`. El paciente entra por OTP
cuando exista, y mientras tanto por "ingresar con un código", que al tener
`cognito_sub` entra en modo `password_reset` y le deja elegir contraseña sin
pedirle identidad. No hay estado intermedio "le falta la contraseña".

Todo o nada, como `complete()`: si Cognito falla, la base queda como estaba y
el LIS puede reintentar la verificación con el mismo código (el pedido sigue
abierto: no marcar `verified_at` hasta que la transacción cierre).

### El contacto es de otra persona

`ContactResolver::resolve()` ya dice de quién es el contacto. Reglas:

- **De un usuario habilitado ya vinculado a esta ficha**: es la cuenta del
  paciente. Caso normal del paciente de v1 que cambió de teléfono o nunca
  verificó WhatsApp.
- **De un usuario habilitado no vinculado a la ficha**: es el caso de la
  madre que empadrona al hijo con su número. La verificación vincula la ficha
  a esa cuenta (el dueño del teléfono es quien va a entrar). El `POST` lo
  anticipa en `account.linked_to_patient = false` para que el LIS lo muestre
  antes de mandar el código. **No** se crea el vínculo `patient_guardians`:
  eso viene del LIS.
- **De un usuario suspendido**: `409 ContactUnavailable`. No se manda código.
- **De nadie**: usuario nuevo, o el adoptable del paciente si hay uno solo.

`users.whatsapp` es único: si el usuario resuelto ya tiene otro WhatsApp
verificado, el nuevo lo reemplaza (es lo que hace `complete()` hoy). Si el
contacto nuevo choca con el de **otro** usuario, ese otro es el dueño y aplica
la segunda regla.

### Endpoints

Bajo `/api/v1/integration`, con el `X-Internal-Token` del cliente y los
middlewares `throttle:integration` e `integration.log`. Scope nuevo
`patients.enroll` (sumarlo a `integration:client-create`; el cliente
`lis-adapter-healthbook` de producción hay que recrearlo o agregárselo).

#### `POST /integration/patients/{externalId}/enrollments` (`patients.enroll`)

El paciente tiene que existir para este proveedor: el adapter hace antes
`PUT /integration/patients/{externalId}` con la foto. Si no está, `404 PatientNotFound`.

Cuerpo:

```json
{
  "channel": "whatsapp",
  "contact": "5493764123456",
  "operator": { "id": 12, "name": "María López" }
}
```

- `channel`: `email` o `whatsapp`. `contact` se normaliza con `Contact::parse`;
  si no parsea, `422 ValidationFailed`.
- `operator`: el usuario del LIS que asiste. Va a `audit_logs.metadata`
  (`operator_id`, `operator_name`, `provider`). Obligatorio.

Si hay un pedido asistido abierto para el mismo contacto y paciente dentro
de `resend_cooldown_seconds`, no se manda otro código: se devuelve ese pedido
con `delivery = "cooldown"`. Fuera del cooldown, los pedidos asistidos
abiertos del mismo contacto se vencen y se emite uno nuevo (como `issueCode`).

Respuesta `201` (`200` en el caso cooldown):

```json
{
  "token": "5f0c1c2e-…",
  "status": "pending_code",
  "channel": "whatsapp",
  "contact": "***3456",
  "expires_at": "2026-10-09T15:42:00-03:00",
  "delivery": "sent",
  "patient": { "id": 4411, "external_id": "77" },
  "account": {
    "id": 120,
    "name": "M*** L***",
    "enrolled": true,
    "linked_to_patient": false
  }
}
```

- `delivery`: `sent`, `failed` (el mensaje no salió: el LIS lo muestra y
  ofrece reintentar) o `cooldown`.
- `account`: el usuario al que va a quedar vinculada la ficha, si ya existe;
  `null` si se va a crear uno. Nombre enmascarado, sin contactos: el LIS ya
  tiene los suyos y éstos pueden ser de otra persona.

Errores, con el sobre `{ error: { code, message, details? } }`:

| Status | `code` |
|---|---|
| 422 | `ValidationFailed` (`details` por campo; el contacto que no parsea va en `contact`) |
| 404 | `PatientNotFound` (no hay ficha con ese `external_id` para el proveedor) |
| 409 | `ContactUnavailable` (el contacto es de un usuario suspendido) |
| 409 | `PatientInactive` (`patients.active = false`) |

#### `POST /integration/enrollments/{token}/verify` (`patients.enroll`)

```json
{ "code": "482913", "operator": { "id": 12, "name": "María López" } }
```

Sólo pedidos con `issued_by = assisted`: un token de otro emisor es `404`.
Mismas reglas de intentos que `verifyCode`: cada código incorrecto cuenta, al
quinto el pedido vence.

Respuesta `200`:

```json
{
  "token": "5f0c1c2e-…",
  "status": "completed",
  "channel": "whatsapp",
  "contact": "***3456",
  "account": {
    "id": 120, "enabled": true, "enrolled": true, "registered": true,
    "channels": { "email": { "address": null, "verified": false }, "whatsapp": { "number": "***3456", "verified": true } },
    "pending_enrollment": false, "last_activity_at": null, "created_at": "2026-10-09T15:40:12-03:00"
  },
  "welcome_sent": true
}
```

`account` tiene la misma forma que en `GET /integration/patients/account`,
para que el LIS refresque el indicador sin volver a consultar.

| Status | `code` | `details` |
|---|---|---|
| 422 | `InvalidCode` | `{ "attempts_left": 3 }` |
| 422 | `EnrollmentExpired` | se venció o agotó los intentos: pedir otro código |
| 422 | `EnrollmentNotPending` | ya estaba completado |
| 404 | `NotFound` | token desconocido o de otro emisor |
| 503 | `IdentityProviderUnavailable` | Cognito no respondió; el pedido sigue abierto, se puede reintentar |

#### `GET /integration/patients/account`: bloque `enrollment`

Agregar, al nivel de `found`, el pedido abierto más reciente del paciente
(el que lo tiene en `patient_ids` o el de su usuario), o `null`:

```json
"enrollment": {
  "token": "5f0c1c2e-…",
  "status": "pending_code",
  "issued_by": "assisted",
  "channel": "whatsapp",
  "contact": "***3456",
  "expires_at": "2026-10-09T15:42:00-03:00"
}
```

Es lo que le permite al LIS retomar la carga del código si la pantalla se
recargó. `accounts[].pending_enrollment` sigue siendo el booleano de hoy.

#### `GET /integration/patients/account`: fechas en `accounts[]`

La pestaña Healthbook de la ficha del LIS muestra cuándo se verificó cada
canal y cuándo entró el paciente por última vez. Agregar a cada cuenta:

- `channels.email.verified_at` y `channels.whatsapp.verified_at`: el
  `*_verified_at` de `users`, ISO 8601, o `null`.
- `last_login_at`: el último ingreso al portal. Healthbook no ve el login
  (va directo a Cognito), así que es el primer request autenticado de cada
  sesión: un middleware que marque `users.last_login_at` cuando el token que
  llega es de una sesión que todavía no vio (o, más simple, una vez por hora
  por usuario). `last_activity_at` queda como está: es el último registro
  de auditoría, que puede ser una acción de la integración y no del paciente.

El LIS ya los lee y los muestra cuando vienen; si faltan, muestra sólo lo de hoy.

### Mensajes

- El código asistido reusa `Communication::AccessCode` (`code`,
  `expires_in_minutes`).
- Bienvenida nueva, `Communication::AssistedWelcome`
  (`healthbook.assisted_welcome`): "ya tenés acceso a Healthbook con este
  WhatsApp/mail; entrá con un código, o elegí una contraseña desde
  <login>/ingresar con código". Sin links con token: el canal ya está
  verificado y la persona entra por el camino normal. `Welcome` queda para el
  empadronamiento propio.

### Auditoría

Casos nuevos de `AuditAction`: `enrollment.assisted_started`,
`enrollment.assisted_completed`, `enrollment.assisted_code_failed`. Los tres
con `metadata.operator = { id, name }` y `metadata.provider`. El `actor` es el
usuario del pedido cuando existe; si no, sin actor.

### Lo que no cambia

- Los endpoints públicos (`/public/access-codes`, `/public/enrollments`) no
  se tocan.
- La integración sigue sin crear usuarios **al recibir órdenes**. Los crea sólo
  acá, con el canal probado y a pedido del paciente.
- `patient_guardians` sigue viniendo del LIS.

### Tests que se esperan

- Flujo completo para los cuatro casos de contacto (propio, de otro usuario,
  suspendido, de nadie) y para el usuario adoptable (cambio de teléfono).
- Cooldown, intentos, expiración, token de otro emisor.
- Cognito que falla en el verify deja el pedido abierto y la base intacta.
- `AccountLookup` con y sin pedido abierto.

## Pieza 2: ingreso por código (OTP)

### Cómo funciona

El login sigue yendo del front directo a Cognito. Se agrega el flujo
`CUSTOM_AUTH` del pool: el front manda el alias (email o WhatsApp), Cognito
llama a las Lambdas, la Lambda de crear desafío le pide a Healthbook que mande
el código, la persona lo escribe, la Lambda de verificar le pregunta a
Healthbook si es correcto, y Cognito emite los tokens de siempre. El backend
no cambia nada de cómo valida tokens.

**Healthbook es dueño de los códigos.** Las Lambdas son finas: no generan ni
guardan nada, llaman a dos endpoints internos. Así las reglas (TTL, intentos,
auditoría, canal) viven en un solo lugar y el código sale por
lis-messaging-service, que es el único que sabe mandar WhatsApp.

### Cognito

- App client: sumar `ALLOW_CUSTOM_AUTH` a los flujos permitidos. Se mantienen
  `USER_PASSWORD_AUTH`/`USER_SRP_AUTH`.
- El pool sigue sin mandar mails ni SMS, sin MFA.
- *Prevent user existence errors* queda activado. Con `CUSTOM_AUTH` y un alias
  que no existe, Cognito igual invoca `DefineAuthChallenge` con
  `userNotFound = true`: la Lambda tiene que devolver un desafío igual al real
  (y Healthbook un `delivery` falso) para que el tiempo y la forma de la
  respuesta no revelen si el contacto existe. Es el mismo principio que
  `requestCode()`.
- Las cuentas tienen que estar en estado `CONFIRMED`: por eso la pieza 1 pone
  una contraseña permanente al azar al provisionar sin contraseña. Las cuentas
  que ya se empadronaron por el flujo público ya lo están.

### Las tres Lambdas (Node, en lis-infra)

| Trigger | Qué hace |
|---|---|
| `DefineAuthChallenge` | Si `session` está vacía: `challengeName = CUSTOM_CHALLENGE`. Si el último desafío fue respondido bien: `issueTokens = true`. Si hay 3 fallidos: `failAuthentication = true`. Con `userNotFound`: mismo desafío. |
| `CreateAuthChallenge` | `POST {HEALTHBOOK}/api/v1/internal/sign-in-codes` con `{ "cognito_sub": …, "username": …, "user_not_found": bool }`. Guarda `challenge_id` en `privateChallengeParameters` y `{ channel, contact }` (enmascarado) en `publicChallengeParameters` para que el front diga "te mandamos un código al ***3456". |
| `VerifyAuthChallengeResponse` | `POST {HEALTHBOOK}/api/v1/internal/sign-in-codes/{challenge_id}/verify` con `{ "code": answer }`. `answerCorrect` según la respuesta. |

Las Lambdas se autentican contra Healthbook con un token interno
(`X-Internal-Token`, en Secrets Manager) distinto del de los clientes de
integración. Timeout corto (5 s) y, si Healthbook no contesta, el desafío
falla cerrado.

### Endpoints internos de Healthbook

Bajo `/api/v1/internal`, middleware nuevo `internal.lambda` (token fijo de
`.env`, `SIGN_IN_LAMBDA_TOKEN`), `throttle:integration`. No pasan por
`integration.log` (son de autenticación, no de datos).

`POST /internal/sign-in-codes`: busca el usuario por `cognito_sub`; si
`user_not_found` o no lo encuentra, responde igual que si existiera, con un
`challenge_id` al azar que nunca va a verificar (sin mandar nada, sin
auditoría de usuario). Si existe, genera el código, lo manda al canal
verificado que la persona usó como alias (si el alias es el mail, al mail; si
es el WhatsApp, al WhatsApp; si no se puede saber, al último verificado) con
`Communication::SignInCode` (`healthbook.sign_in_code`, nuevo), y guarda una
fila en `sign_in_challenges` (`id` uuid, `user_id`, `channel`, `code_hash`,
`attempts`, `expires_at` 5 min, `verified_at`). Auditoría `auth.otp_sent`.

```json
{ "challenge_id": "…", "channel": "whatsapp", "contact": "***3456", "expires_in_minutes": 5 }
```

`POST /internal/sign-in-codes/{challenge_id}/verify`: compara con
`hash_equals`; cada fallo suma `attempts`; al tercero o al vencer, `expired`.
Al acertar marca `verified_at`, auditoría `auth.otp_login`, y responde
`{ "correct": true }`. Un `challenge_id` desconocido responde `{ "correct": false }`.

### Front de Healthbook

En la pantalla de login, debajo de la contraseña: "Ingresar con un código".
Pide el email o WhatsApp, llama `initiateAuth` con `AuthFlow = CUSTOM_AUTH`,
muestra el canal que vino en `publicChallengeParameters`, y manda el código con
`respondToAuthChallenge`. Si Cognito corta (3 fallos o vencido), vuelve al
principio.

La opción actual "ingresar con un código" (empadronamiento / recupero) se
renombra a "Crear o cambiar mi contraseña" para que no se confundan. Es la
que sigue usando quien todavía no está en Cognito.

### Lo que hay que decidir

- Si el código de ingreso también vale para **profesionales** o sólo para el
  rol paciente. Recomendación: para todos; la Lambda no distingue y el riesgo
  es el mismo que el recupero por código que ya existe.
- Vigencia de la sesión de Cognito tras un login por OTP: la misma que por
  contraseña (refresh token de 30 días) o más corta. Recomendación: la misma.

### Tests que se esperan

- Endpoints internos: usuario existente, inexistente (misma forma y tiempo),
  intentos, vencimiento, canal elegido según el alias.
- Lambdas: pruebas unitarias con eventos de Cognito grabados (los tres
  triggers, incluido `userNotFound`).
- Un paciente empadronado por la pieza 1 (contraseña al azar) entra por OTP de
  punta a punta contra el pool de staging.

## Contexto: lo que ya hace el LIS

Para probar contra algo real. Está en `lis-adapters/lis-adapter-healthbook`,
`lis-backend` y `lis-front-monorepo`.

**Adapter** (`X-Internal-Token` del LIS):

- `POST /api/v1/patients/{externalId}/enrollments` con
  `{ patient: <canónico>, channel, contact, operator }`: hace
  `PUT /integration/patients/{externalId}` con la foto y después
  `POST /integration/patients/{externalId}/enrollments`. Devuelve la respuesta
  de Healthbook bajo `data`.
- `POST /api/v1/enrollments/{token}/verify` con `{ code, operator }`:
  passthrough.
- En `HEALTHBOOK_MODE=stub` imita todo: el código es siempre `123456`; con
  `[healthbook:fail]` en el contacto no responde.

**Backend**: `POST /patients/{patient}/portal-enrollment` (`channel`,
`contact` opcional: sin él usa el mail o el celular de la ficha; WhatsApp exige
`whatsapp_consent`) y `POST /patients/{patient}/portal-enrollment/verify`
(`token`, `code`). El operador es el usuario logueado.
`GET /patients/{patient}/portal-status` ya pasa `enrollment` si viene.

**Front**: en la ficha, la admisión y la admisión rápida, al lado del
indicador de Healthbook, el botón "Dar acceso a Healthbook": elegir canal,
mandar el código, cargarlo, ver el resultado. Nunca bloquea la admisión.
