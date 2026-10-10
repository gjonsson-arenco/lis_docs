# Buzón de desarrollo: ver los códigos de verificación en el front

Brief para implementar en el stack del LIS (`lis-messaging-service`,
`lis-adapter-healthbook`, `lis-backend` y `lis-front-monorepo`) lo que ya
existe en Healthbook: que en desarrollo los códigos de verificación de canal
(y cualquier otro mensaje "enviado") se vean en pantalla, sin revolver logs ni
la carpeta `storage/outbox`. El lado de Healthbook **ya está hecho** y se
describe primero como contrato y como modelo a copiar.

Leer antes de empezar: `lis-healthbook-back/README.md`, sección "Probar
códigos y OTP sin Cognito ni logs"; `App\Messaging\DevMailbox`,
`App\Messaging\LogMessenger`, `App\Http\Controllers\Api\V1\DevMessagesController`;
y en el front `apps/web/src/features/dev/components/dev-code-peek.tsx` y
`apps/web/src/app/(public)/dev/mensajes/page.tsx`.

## Por qué

En local nada sale de verdad: Healthbook corre con `MESSAGING_DRIVER=log` y
lis-messaging-service con `EMAIL_DRIVER=log` / `WHATSAPP_DRIVER=log`. Cada vez
que un flujo manda un código (empadronamiento asistido, ingreso por código,
recupero de contraseña, y la verificación de canal que el LIS vaya a hacer
con `whatsapp_consent`), la persona que prueba tiene que ir a buscarlo a un
log o a un `.eml`. Con el buzón, el código aparece al lado del campo donde se
carga, y una página lista todo lo que "salió".

Regla de oro: **nada de esto existe en producción**. Las rutas no se
registran, los componentes no renderizan, y los datos completos (códigos,
links con token) solo viajan en entornos con drivers `log`.

## Lo que ya hace Healthbook (contrato)

### Back (`lis-healthbook-back`)

- `App\Messaging\DevMailbox`: guarda en el cache (`Cache::put`, clave
  `dev:mailbox`, 1 día) los **últimos 50** mensajes, más nuevo primero. Lo
  alimenta `LogMessenger` (el driver `log`) en cada envío, con el `data`
  entero de la plantilla.
- `GET /api/v1/dev/messages[?contact=…]` → `{ "messages": [...] }`. El
  filtro acepta el contacto escrito como sea (`Contact::parse`: mail en
  minúsculas, WhatsApp por los últimos 10 dígitos). Un contacto que no parsea
  devuelve lista vacía.
- `DELETE /api/v1/dev/messages` → `204`, vacía el buzón.
- Las dos rutas solo se registran con `MESSAGING_DRIVER=log` y fuera de
  `production` (`routes/api.php`); el driver `log` ya está prohibido en
  producción por `AppServiceProvider`. Sin auth: es desarrollo. Rate limit
  `throttle:authenticated` (120/min) porque el front hace polling.

Forma de un mensaje (la misma que conviene usar en el LIS, para compartir el
componente del front):

```json
{
  "id": "6b1f…",
  "communication": "healthbook.access_code",
  "channel": "whatsapp",
  "address": "5491155551234",
  "name": "Ana Perez",
  "data": { "code": "482913", "expires_in_minutes": 10 },
  "sent_at": "2026-10-10T03:12:44+00:00"
}
```

- `address` va **sin enmascarar** (solo existe en dev).
- `data` es el payload de la plantilla tal cual: el front busca `data.code`
  y, si no hay, la primera clave que termine en `_url` (links de invitación,
  de informe).

### Front (`lis-healthbook-front/apps/web`)

- `DevCodePeek contact=…`: `useQuery` sobre `/dev/messages?contact=`,
  `refetchInterval: 2500`, `retry: false`. Renderiza un `Alert` amarillo
  "Modo desarrollo" con el último código (o link) enviado a ese contacto y un
  link a `/dev/mensajes`. Si el back no tiene el endpoint (error), no renderiza
  nada. Está puesto al lado del `PinInput` en el flujo de acceso y en el
  ingreso por código.
- `/dev/mensajes`: tabla con los 50 mensajes (cuándo, comunicación + canal,
  destinatario, código o link resaltado y el `data` entero), botones
  "Actualizar" y "Vaciar".
- Prendido por `devCodesEnabled` (`src/lib/dev.ts`): si existe
  `NEXT_PUBLIC_DEV_CODES` manda esa variable; si no, prendido en modo `dev` y
  apagado con Cognito. En `vitest.config.ts` está en `"false"` para que los
  tests no le peguen al back.

### Ingreso por código sin Cognito (no aplica al LIS, se menciona por completitud)

`POST /api/v1/dev/sign-in-codes` y `…/{challenge}/verify` hacen lo que harían
las Lambdas del pool y emiten el token local. El LIS no tiene OTP propio: lo
que necesita es solo el buzón.

## Qué hay que construir en el LIS

Hay **dos fuentes** de mensajes que un recepcionista quiere ver desde el
front del LIS:

1. Los que manda **Healthbook** durante el empadronamiento asistido (el
   código que el paciente "leería en su teléfono"). Ya están en el buzón de
   Healthbook; falta que el LIS los traiga.
2. Los que manda **el propio LIS** por lis-messaging-service (hoy
   `order.created`, `indications.sent`, etc.; mañana el código de
   verificación de canal con el que el paciente confirma su WhatsApp o mail).
   Hoy terminan en `storage/outbox/<id>.eml` del servicio: hace falta un buzón
   equivalente ahí.

### 1. `lis-messaging-service`: buzón de los drivers `log`

- Cuando `EMAIL_DRIVER=log` o `WHATSAPP_DRIVER=log`, además del `.eml` en
  `storage/outbox`, dejar el mensaje en un buzón en memoria (o en la tabla de
  historial, que ya existe) consultable por:
  - `GET /api/v1/dev/messages?address=…&limit=50` → `{ "messages": [...] }`
    con la **misma forma** de arriba. `communication` es el código del
    catálogo (`order.created`, `patient.channel_verification`…), `data` las
    variables que pidió el llamador, y conviene sumar `rendered_text` (el
    cuerpo de texto ya armado) para los mensajes que no traen un `code`
    explícito.
  - `DELETE /api/v1/dev/messages` → `204`.
- Con `X-Internal-Token`, como todo el servicio. La ruta **no existe** si los
  dos drivers son reales (`ses` / `cloud`) o si `NODE_ENV=production`.
- Filtro por `address` con el mismo criterio que Healthbook: mail en
  minúsculas; WhatsApp por los últimos 10 dígitos, para que `11 5555 1234` y
  `5491155551234` sean el mismo.
- Tests: jest, igual que el resto del servicio: un envío con driver `log` queda
  en el buzón; con driver real la ruta responde 404; el filtro por dirección.

### 2. `lis-adapter-healthbook`: passthrough del buzón de Healthbook

- `GET /api/v1/dev/messages?contact=…` (con el `X-Internal-Token` del
  adapter):
  - `HEALTHBOOK_MODE=http`: llama a `GET {HEALTHBOOK_API_URL}/api/v1/dev/messages?contact=…`
    y devuelve el cuerpo tal cual. Si Healthbook responde 404 (producción, o
    sin `MESSAGING_DRIVER=log`), devolver `404` con el sobre de error de
    siempre: el front lo toma como "buzón apagado".
  - `HEALTHBOOK_MODE=stub`: devolver lo que el stub "mandó": por cada pedido
    asistido abierto, un mensaje `healthbook.access_code` con
    `data.code = STUB_ENROLLMENT_CODE` (`123456`) al contacto del pedido. Así
    el front se comporta igual en los dos modos.
- La ruta solo se registra con `HEALTHBOOK_DEV_MAILBOX=true` (default `true`
  en `.env.example`, `false` en el compose de producción).
- Tests: el spec del cliente http (pasa la query y el 404) y el del stub.

### 3. `lis-backend`: un solo endpoint para el front

- `GET /api/v1/dev/messages?contact=…` (autenticado como el resto de la API
  del LIS, con `tenant`): junta las dos fuentes y devuelve
  `{ "messages": [...] }` ordenado por `sent_at` descendente, cada mensaje con
  un campo extra `source`: `"healthbook"` o `"lis"`. Si una fuente no está
  (404, timeout de 2 s, sin configurar), se omite y se informa en
  `"unavailable": ["healthbook"]`; nunca falla por eso.
- `DELETE /api/v1/dev/messages`: vacía las dos.
- Se registra solo con `APP_ENV != production` **y** `DEV_MAILBOX_ENABLED=true`
  (`.env.example`: `true`). En producción la ruta no existe.
- Dónde: `App\Services\Dev\DevMailboxAggregator` con dos clientes finos
  (`HealthbookAccountClient` ya tiene la base URL y el token del adapter;
  `MessagingClient` tiene los del servicio de mensajería).
- Tests de feature: con las dos fuentes falsas (Http::fake), con una caída, y
  que la ruta no existe en producción.

### 4. `lis-front-monorepo` (`apps/lis`)

- `services/apis/dev-mailbox.api.ts`: `getDevMessages(contact?)`,
  `clearDevMessages()`.
- `components/dev-code-peek.tsx`, copia del de Healthbook: polling cada
  2,5 s, `retry: false`, no renderiza si el endpoint da error o si está
  apagado. Mostrar `source` ("Healthbook" / "LIS") junto a la comunicación.
- Ponerlo en `patient-portal-enrollment.tsx`, en el paso `code`, al lado del
  campo donde el recepcionista carga el código, con `contact` = el contacto al
  que se mandó (el que devolvió `start`, antes de enmascarar: el componente ya
  lo tiene porque lo eligió el operador). Cuando exista la verificación de
  canal del LIS, el mismo componente al lado de ese campo.
- Página `/dev/mensajes` dentro de la app (con sesión), misma tabla que la
  de Healthbook más la columna `source`, con "Actualizar" y "Vaciar".
- Flag `NEXT_PUBLIC_DEV_CODES`: si está definida manda; si no, prendido
  cuando `NODE_ENV !== "production"`. Apagado en la config de vitest.
- Tests: el peek muestra el código que devuelve la API y no renderiza nada con
  error; la página lista y vacía.

## Cómo se prueba de punta a punta hoy, antes de hacer nada

Ya se puede, mirando el buzón de Healthbook directo:

1. Healthbook back con `MESSAGING_DRIVER=log`, front web en modo `dev`.
2. En la admisión del LIS, "Dar acceso a Healthbook" con el adapter en
   `HEALTHBOOK_MODE=http` apuntando a `http://localhost:8100`.
3. Abrir `http://localhost:3100/dev/mensajes`: ahí está el código asistido.
   Cargarlo en el LIS. Después, en Healthbook, "Ingresar con un código" con el
   mismo contacto: el código también aparece ahí.

Con `HEALTHBOOK_MODE=stub` el código es siempre `123456` y no hace falta nada.

## Lo que no cambia

- Los mensajes reales siguen saliendo solo por lis-messaging-service; el
  buzón es una copia de lo que el driver `log` no mandó.
- No se agregan campos a las respuestas de los endpoints de negocio
  (`portal-enrollment`, `messages`): el código nunca viaja en ellas, ni en dev.
- Los `.eml` de `storage/outbox` siguen generándose: son la prueba de cómo se
  ve el mail; el buzón es para leer rápido el dato.

## Estado (2026-10-10)

Hecho en los cuatro repos. Diferencias con lo de arriba, para quien lo lea
después:

- Los tres servicios del LIS devuelven `{ "data": { "messages": [...] } }`
  (y el backend suma `unavailable` adentro de `data`), no el `{ "messages" }`
  pelado de Healthbook: es el sobre de toda la API del LIS. La forma de cada
  mensaje sí es la misma, más `source` en el backend.
- lis-messaging-service: la respuesta automática por WhatsApp (`sendText`)
  también queda en el buzón, como `whatsapp.text`.
- El adapter en modo stub guarda un `healthbook.access_code` por pedido
  iniciado (no por pedido abierto): "Vaciar" lo borra sin tocar el pedido.
- La página `/dev/mensajes` del LIS no tiene entrada en el menú: se entra
  por URL o desde el link del peek.
- Pendiente: probar de punta a punta contra Healthbook con
  `MESSAGING_DRIVER=log` (se hace junto con el empadronamiento asistido).
