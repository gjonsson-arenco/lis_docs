# Presupuestos y turnos entre el LIS y bitra

Cómo el laboratorio presupuesta cada orden médica en el LIS y el paciente ve el
presupuesto en bitra, y cómo la agenda de turnos vive en bitra y el LIS la usa
desde el mostrador. Diseño del 2026-10-30, a partir de estas decisiones:

- **Toda orden médica se presupuesta.** No hace falta que el paciente lo pida:
  cuando entra una orden médica, alguien administrativo la toma en el LIS y
  genera el presupuesto. El pedido de presupuesto que hoy tiene bitra se retira.
- **El presupuesto llega a bitra con detalle por práctica y un PDF.** El
  paciente no acepta ni rechaza: lo ve, y puede sacar turno para esa orden.
- **La agenda de turnos vive en bitra.** El LIS la usa por API: agenda y
  cancela desde el mostrador, muestra los turnos en la ficha y administra los
  horarios y cupos de cada sede desde su Configuración.

Documentos relacionados: [medical-orders-sync.md](./medical-orders-sync.md)
(el circuito de órdenes médicas sobre el que se apoya el presupuesto),
[virtual-queue.md](./virtual-queue.md) (el feed que se copia para los turnos)
y [event-driven-integrations.md](./event-driven-integrations.md).

## Restricción de partida

La de siempre: **la conexión la abre el LIS**. El on-premise de CEBAC no es
alcanzable desde la nube. Por eso:

| Sentido | Mecanismo |
| --- | --- |
| LIS → bitra | Lo que el LIS hace (presupuestar, agendar, cancelar, configurar horarios) sale como evento de dominio por el orchestrator, o como llamada sincrónica a través del adapter cuando el operador necesita la respuesta en el acto. |
| bitra → LIS | El LIS lee un **feed de cambios** cada 15 s, como hace con las órdenes médicas y con la fila virtual. |

## Parte 1: presupuestos

### Qué ve cada uno

**El administrativo, en el LIS.** En Órdenes médicas hay un filtro nuevo, "Sin
presupuesto", que lista las órdenes en estado `new` que todavía no tienen
presupuesto. Abre la orden, revisa los estudios y toca "Presupuestar", igual
que hoy. El presupuesto queda en la orden y, si el paciente usa la app, se va
solo a bitra. Puede volver a presupuestar (otra cobertura, corrección): el
nuevo reemplaza al anterior en bitra y el historial queda en los dos lados.

**El paciente, en bitra.** En la orden médica aparece el presupuesto: la
cobertura con la que se calculó, cada práctica con lo que paga él y lo que
cubre la obra social, el total a su cargo, hasta cuándo vale y el PDF para
descargar o compartir. Recibe un aviso cuando el presupuesto está listo. Desde
ahí mismo puede sacar turno para esa orden. Si el presupuesto tiene pendientes
(autorización o bono), lo ve como aviso, no como monto final.

Lo que se retira de bitra: el botón "Pedir presupuesto" de la app y del portal,
la pantalla de pedidos del paciente y la bandeja "Presupuestos" del admin. El
modelo `QuoteRequest` se deja de usar; se puede borrar en una limpieza
posterior, cuando no quede ninguno abierto en CEBAC.

### Cómo viaja

El presupuesto es **parte de la orden médica**, no una entidad aparte. Viaja
en el mismo `PUT /integration/medical-orders/{uuid}` y por el mismo evento:

- `MedicalOrderQuoteService::quote()` dispara `MedicalOrderUpdated` después del
  commit, como cualquier otra edición de la orden. No hay evento nuevo ni regla
  nueva: `update-medical-order-in-healthbook` ya está activa en CEBAC.
- `MedicalOrderIntegrationPayloadBuilder` agrega `quote`, el último presupuesto
  de la orden, o `null`.
- El adapter lo pasa a bitra tal cual, en el `PUT`.
- Rige la regla de siempre: si el paciente no usa la app, bitra contesta
  `stored: false` y no guarda nada. El presupuesto queda sólo en el LIS, y el
  paciente lo recibe en el mostrador o por el canal que ya use.

### El objeto `quote`

Es una versión **resumida** del `MedicalOrderQuote` del LIS: lo que el paciente
tiene que leer, no la valorización completa (sin reglas aplicadas, sin
nomenclador, sin trazas).

```json
"quote": {
  "external_id": "812",
  "status": "passed",
  "coverage_label": "OSDE 310",
  "is_private": false,
  "currency": "ARS",
  "totals": { "patient": 12500.00, "insurance": 38200.00, "total": 50700.00 },
  "requires_voucher": false,
  "pending": [
    { "code": "authorization", "practice_code": "660474", "message": "Requiere autorización de la cobertura." }
  ],
  "practices": [
    {
      "code": "660474", "name": "Hemograma completo", "quantity": 1,
      "coverage_percent": 100, "patient_amount": 0, "insurance_amount": 3200.00,
      "requires_authorization": true, "requires_voucher": false, "status": "resolved"
    }
  ],
  "valid_until": "2026-11-14",
  "generated_at": "2026-10-30T14:02:11-03:00",
  "pdf": { "s3": { "bucket": "cebac-documents", "key": "lis/medical-orders/quotes/812.pdf" } }
}
```

- `external_id` es el id del `MedicalOrderQuote` en el LIS. bitra hace *upsert*
  por `(medical_order_id, external_id)` y guarda todos: el vigente es el de
  `generated_at` más nuevo.
- `status` sigue al del LIS: `passed` (monto final), `pending` (hay
  autorización o bono por resolver), `blocked` (hay prácticas sin precio: no
  se manda; el LIS no genera `MedicalOrderUpdated` para un presupuesto
  `blocked`, porque no es un presupuesto que el paciente pueda leer).
- `pending[]` son los avisos que bitra muestra arriba del detalle. Se derivan
  de las prácticas: `authorization`, `voucher`, `not_billable`.
- `pdf` es el documento. Viaja como los archivos de la receta: **el LIS lo
  escribe en el bucket compartido, en su prefijo, y bitra lo copia al suyo**.
  Sin S3 (local, o mientras no se cierre el bucket en CEBAC), `pdf` es `null`
  y bitra muestra el detalle sin archivo. Depende del ítem abierto del S3
  compartido: ver "Qué falta".

### El PDF

Lo genera el LIS con el servicio de documentos, plantilla nueva
`customers/cebac/medical-order-quote/v1` (A4, nunjucks, como las otras). Se
genera **al presupuestar**, se guarda como documento del LIS (sirve para
imprimirlo en el mostrador o mandarlo por mensajería) y su ubicación viaja en
`quote.pdf`. El contenido: datos del laboratorio, paciente, orden médica y
médico, cobertura, tabla de prácticas con las tres columnas (cobertura,
paciente, total), totales, pendientes, vigencia y la leyenda legal del
presupuesto.

### El aviso

bitra manda `healthbook.quote_ready` al paciente (usuario de la ficha o tutor)
cuando llega un presupuesto nuevo en `passed` o `pending`, con `patient_name`,
`patient_total`, `currency`, `valid_until`, `coverage_label` y la `url` de la
orden en el portal. Hay que darlo de alta en el catálogo de
lis-messaging-service, con plantilla de email. `quote_requested` y
`quote_answered` quedan sin uso.

## Parte 2: turnos

### Qué ve cada uno

**El operador, en el LIS.**

- En la ficha del paciente hay una pestaña **Turnos**: los vigentes y los
  últimos pasados, con sede, fecha y hora, estado, la orden médica si la hay, y
  los botones Agendar y Cancelar.
- En la orden médica, la tarjeta "Turno" abre el mismo diálogo de agenda:
  elige la sede, ve el calendario con los días habilitados y las franjas con
  cupo, elige una y confirma. Ese diálogo reemplaza al actual, que inventa
  franjas de media hora sin consultar a nadie.
- En Configuración > Sistema hay una tarjeta **Turnos**: por sede, si recibe
  turnos, la duración de la franja, los horarios por día de semana con su cupo,
  y los cierres (feriados, días sin atención). Es lo mismo que hoy edita el
  admin de bitra en "Sedes", pero desde el LIS.
- En Admisión no cambia nada por ahora. Cuando el paciente con turno llega y se
  lo admite, el LIS marca el turno como cumplido; los que pasan de fecha sin
  admisión quedan como ausente (ver "Qué falta").

**El paciente, en bitra.** Lo que ya tiene: saca turno desde la orden médica o
desde el presupuesto, lo ve en la agenda, lo cancela con la ventana de 24 h, y
se anuncia con "Ya llegué" cuando está en la sede. No cambia la pantalla; sólo
ve además los turnos que le agendaron en el mostrador.

### Dónde vive cada cosa

| Dato | Dueño | El otro lado |
| --- | --- | --- |
| Horarios, cupos, cierres, duración de franja, si la sede recibe turnos | **bitra** (son sus tablas `site_schedules`, `site_closures`, `sites`) | El LIS los edita a través de la API; no los guarda. |
| Disponibilidad | **bitra** la calcula | El LIS la pide cuando el operador abre el diálogo. |
| El turno | **bitra** lo crea, sea desde la app o desde el mostrador | El LIS guarda una **réplica** en su tabla `appointments`, con el id de bitra, para la ficha, las listas y la admisión. |
| Cumplido / ausente | **el LIS** lo decide (admisión) | bitra lo recibe por la API y lo muestra. |

La réplica en el LIS existe por dos razones: las listas del LIS (ficha, orden
médica, admisión) no pueden depender de una llamada a la nube por cada fila, y
CEBAC tiene cortes de internet. La fuente de verdad sigue siendo bitra.

### Los cruces

- **Sede**: `sites.lis_site_code` en bitra = `sites.code` en el LIS. Ya existe
  por la fila virtual, y el sync de la fila crea la sede en bitra si no está
  (deshabilitada para turnos hasta que se le carguen horarios).
- **Paciente**: `external_id` del paciente = id del paciente en el LIS, con el
  `provider` del cliente de integración. Si el paciente no existe en bitra, el
  LIS lo da de alta antes de agendar con el `PUT /integration/patients/{id}`
  que ya usa para los tutores. Un paciente puede tener turno sin tener cuenta
  en la app.
- **Orden médica**: por `uuid`, opcional.
- **Turno**: `appointments.id` de bitra = `appointments.external_id` en el LIS.

### La API de integración de bitra

Todo bajo `/api/v1/integration`, con `X-Internal-Token`, `integration.log` y
`integration.auth:<scope>`. Tres scopes nuevos: `appointments.read`,
`appointments.write`, `sites.manage`.

| Endpoint | Para qué | Scope |
| --- | --- | --- |
| `GET sites/{code}/availability?from&to` | Las franjas con cupo. Misma respuesta que `GET /sites/{site}/availability` del paciente. | `appointments.read` |
| `GET patients/{externalId}/appointments` | Los turnos del paciente (vigentes primero), para rehidratar la réplica. | `appointments.read` |
| `GET appointments/changes?after&limit` | El feed de cambios de turnos. | `appointments.read` |
| `POST patients/{externalId}/appointments` | Agendar desde el mostrador: `{ site_code, scheduled_at, medical_order_uuid?, notes?, booked_by }`. | `appointments.write` |
| `POST appointments/{id}/cancel` | `{ reason?, cancelled_by }`. Sin ventana de 24 h: el laboratorio puede cancelar siempre. | `appointments.write` |
| `PATCH appointments/{id}` | `{ status: completed \| no_show }`. | `appointments.write` |
| `GET sites/{code}` | La sede con horarios y cierres. | `sites.manage` |
| `PATCH sites/{code}` | `{ enabled, slot_minutes, address?, phone?, instructions? }`. | `sites.manage` |
| `PUT sites/{code}/schedules` | `{ schedules: [{ weekday, starts_at, ends_at, capacity }] }`, reemplaza todo. | `sites.manage` |
| `PUT sites/{code}/closures` | `{ closures: [{ date, reason? }] }`, reemplaza los futuros. | `sites.manage` |

`{code}` es siempre el `lis_site_code`. `scheduled_at` va en ISO 8601 con
offset, exactamente como lo devuelve la disponibilidad; bitra exige que
coincida con el inicio de una franja.

**El actor.** Hoy `AppointmentBooking` y los servicios de sedes exigen un
`User`. Pasan a aceptar un actor nulo con un nombre libre (`booked_by`,
`cancelled_by`: el operador del LIS), que queda en la auditoría. El aviso
`appointment_confirmed` deja de ir al actor y va al **paciente** (su usuario o
el del tutor), tanto si agendó él como si le agendaron en el mostrador.

**El feed.** Stream `appointments` en `integration_outbox`, eventos `booked`,
`cancelled`, `updated` (cambio de estado). A diferencia de las órdenes
médicas, **los cambios que entran por la integración también van al outbox**:
el LIS hace *upsert* por `external_id` y un eco es inocuo, mientras que
perderse el turno que agendó el mostrador porque se cortó la respuesta no lo
es. Cada entrada trae el turno completo:

```json
{ "seq": 118, "event": "booked", "occurred_at": "…",
  "appointment": {
    "id": 57, "status": "scheduled", "scheduled_at": "2026-11-03T10:30:00+00:00",
    "timezone": "America/Argentina/Buenos_Aires", "slot_minutes": 15,
    "site": { "id": 3, "lis_site_code": "0", "name": "CEBAC" },
    "patient": { "id": 41, "external_id": "660475", "first_name": "…", "last_name": "…", "identifications": [ … ] },
    "medical_order": { "id": 9, "uuid": "…" },
    "notes": null, "source": "app" | "lis", "booked_by": null,
    "cancelled_at": null, "cancel_reason": null, "created_at": "…" } }
```

### El lado del LIS

- **`appointments` pasa a ser la réplica.** Columnas nuevas: `external_id`
  (único, nullable para lo viejo), `site_id`, `slot_minutes`, `source`
  (`app` | `lis`), `cancelled_at`, `cancel_reason`, `booked_by`. `service_id`
  y `duration_minutes` quedan por compatibilidad; dejan de usarse. Los estados
  pasan a los de bitra: `scheduled`, `cancelled`, `completed`, `no_show`
  (`confirmed` se migra a `scheduled`).
- **Pull.** `integrations:sync-appointments` cada 15 s, flag
  `APPOINTMENT_SYNC_ENABLED`, cursor `integration_inbound_cursors` stream
  `appointments`, importador que hace *upsert* por `external_id` y resuelve el
  paciente con `MedicalOrderImportService::resolvePatient`. Copia del runner de
  la fila virtual, sin la parte de publicar.
- **Escritura.** `POST /appointments` y `POST /appointments/{id}/cancel` del
  LIS llaman al adapter en forma sincrónica (el operador espera la respuesta),
  aplican la respuesta a la réplica y devuelven el turno. `PATCH` y `DELETE`
  actuales se retiran. Cliente nuevo `AppointmentClient`, hermano de
  `VirtualQueueFeedClient`.
- **Disponibilidad y configuración.** `GET /appointments/availability?site_id&from&to`
  y `GET/PUT /admin/appointments/sites/{site}` (horarios, cierres, habilitada,
  franja) son *proxies* al adapter; el LIS no guarda nada. Permiso
  `users.manage`, como la fila virtual.
- **Turnos de hoy.** `GET /appointments?date&site_id` sobre la réplica, para
  la futura vista de agenda del día.
- **Fila virtual.** El importador de pedidos ya guarda
  `appointment_external_id`; ahora puede enlazar la fila local.

### El adapter

Módulo `appointments`: controlador con los mismos verbos hacia el LIS
(`/api/v1/appointments/...`, `/api/v1/sites/...`), puerto y cliente HTTP con
los diez métodos nuevos, stub con agenda en memoria (una sede con horario de
lunes a viernes) y tests. El feed se traduce a la forma canónica
`{ cursor, type, occurred_at, appointment }` como los otros dos.

## Qué hay que construir, por repo

**bitra back (`lis-healthbook-back`)**

1. Tabla `medical_order_quotes` + modelo + `quote` opcional en el
   `PUT /integration/medical-orders/{uuid}` (upsert, copia del PDF al prefijo
   propio, aviso `quote_ready`). `MedicalOrderResource` suma `quote` (vigente)
   y `quotes`.
2. Scopes nuevos, rutas de turnos y sedes de la tabla de arriba, outbox
   `appointments`, actor nulo en `AppointmentBooking` y en los servicios de
   sedes, aviso al paciente y no al actor.
3. `quotesEnabled` deja de tener sentido: se quita de `features` y del app
   config. Las rutas de `QuoteRequest` se dejan hasta la limpieza.
4. README (API de integración) y tests de feature como los de
   `IntegrationMedicalOrderController`.

**bitra front (app y web)**

5. Pantalla de **detalle de la orden médica** en la app (hoy no existe): archivos,
   estado en el laboratorio, presupuesto con detalle y PDF, botón "Sacar
   turno con esta orden". En el portal web, lo mismo en la orden médica.
6. Se quitan "Pedir presupuesto", `/quotes/*` y la bandeja admin de
   presupuestos. La agenda y las sedes del admin web quedan: siguen sirviendo
   como vista.

**LIS backend**

7. Evento `MedicalOrderUpdated` al presupuestar, `quote` en el payload, PDF al
   presupuestar (config `documents.medical_order_quote`), filtro
   `without_quote` en `GET /medical-orders`.
8. Réplica de turnos, pull, `AppointmentClient`, endpoints nuevos y retiro de
   los viejos, proxies de disponibilidad y configuración, config
   `appointment_sync`, `.env.example`.
9. Tests unitarios del importador y del payload.

**Adapter**

10. `quote` en el mapper de órdenes médicas (passthrough con validación).
11. Módulo `appointments` completo, stub y tests.

**Servicio de documentos**

12. Plantilla `customers/cebac/medical-order-quote/v1` (A4) con su schema.

**Mensajería**

13. Catálogo: `healthbook.quote_ready`, `healthbook.appointment_confirmed`,
    `healthbook.appointment_cancelled`, con plantillas de email. Los demás
    códigos de bitra que faltan (invitación, bienvenida, cuenta) quedan para el
    ítem 3 del backlog.

**LIS front**

14. Filtro "Sin presupuesto" y columna en Órdenes médicas.
15. Diálogo de agenda nuevo (sede, calendario, franjas) en la orden médica;
    pestaña Turnos en la ficha con Agendar y Cancelar.
16. Tarjeta Turnos en Configuración > Sistema (habilitada, franja, horarios por
    día con cupo, cierres).

## Qué está hecho (2026-10-30): la fase 1, presupuestos de punta a punta

Commiteado en `main` de cada repo el mismo día del diseño:

- **LIS backend** (`09bea07`): `MedicalOrderQuoteService::quote()` genera el
  PDF con el servicio de documentos (documento del paciente de tipo `quote`,
  colgado del presupuesto por `medical_order_quotes.document_id`) y dispara
  `MedicalOrderUpdated`; `MedicalOrderQuoteSummary` arma el objeto `quote`
  del payload (null si el último está `blocked`); `GET /medical-orders`
  acepta `has_quote`; `GET /medical-orders/{id}/quotes/{quote}/pdf` baja el
  PDF guardado o lo dibuja en el momento. Config en
  `documents.medical_order_quote`.
- **Servicio de documentos** (`3bb3cf3`): plantilla
  `customers/cebac/medical-order-quote/v1`, A4.
- **Adapter** (`7ef66a5`): `quote` pasa en el `PUT /medical-orders/{uuid}`
  con la forma verificada; el PDF sólo si está en S3.
- **bitra back** (`52f6ff7`, `61a3478`): tabla `medical_order_quotes`
  (upsert por `external_id`, historial), copia del PDF a `quotes/`, aviso
  `healthbook.quote_ready` al paciente, `quote`/`quotes` en
  `MedicalOrderResource`, `GET /medical-orders/{id}/quotes/{quote}/pdf`.
  `features.quotes` dejó de anunciarse. Las rutas y el modelo `QuoteRequest`
  siguen en el código hasta la limpieza.
- **Mensajería** (`24ab513`): `healthbook.quote_ready` en el catálogo, con
  su mail. Los avisos de turnos quedan para la fase 2.
- **bitra front** (`6c13cca`): detalle de la orden médica en la app y en el
  portal con la tarjeta del presupuesto y el PDF, "Pedir turno con esta
  orden"; retirados el pedido de presupuesto, sus pantallas y la bandeja del
  admin.
- **LIS front** (`baeae82`): filtro "Sin presupuesto" en Órdenes médicas y
  botón de PDF en la tarjeta y en el diálogo.

Para probarlo de punta a punta hace falta el LIS con el matcher y el motor
de reglas (la valorización), el servicio de documentos en 3030 y el adapter
en `http` contra un bitra con el cliente de integración; sin S3, el
presupuesto llega sin PDF. Falta ese smoke y el APK nuevo.

## Orden de implementación

1. **Presupuestos de punta a punta** (ítems 1, 5, 7, 10, 12, 13-parcial, 14).
   Es el circuito más corto porque se apoya en el de órdenes médicas, que ya
   funciona en CEBAC.
2. **Turnos** (2, 6, 8, 9, 11, 15, 16).
3. **Puesta en marcha en CEBAC:** scopes al cliente `lis-cebac-qa`, flag del
   pull, horarios de la sede cargados desde el LIS, `appointmentsEnabled` en
   bitra, build nuevo de la app.

## Qué falta decidir o queda para después

- **El bucket compartido.** El PDF del presupuesto depende de la misma
  decisión pendiente que los archivos de la receta (ítem 5 del backlog del
  repaso): prefijos `lis/*` y `hb/*` con lectura cruzada. Hasta entonces,
  `pdf: null`.
- **Cumplido y ausente.** Marcar el turno como cumplido al admitir al paciente
  y como ausente al cierre del día es trabajo de Admisión; queda para después
  de la fase 2, con la API ya lista (`PATCH appointments/{id}`).
- **Turnos viejos del LIS.** Si en CEBAC hay turnos agendados con el módulo
  actual, quedan visibles como históricos sin `external_id`; no se migran a
  bitra.
- **Un turno, varias órdenes médicas.** bitra admite una sola orden por turno.
  Si hace falta más, se cambia a una tabla pivote; no es el caso de arranque.
