# Integración LIS → Healthbook: adapter, orchestrator y backend

Brief para implementar la integración del LIS nuevo con Healthbook. El lado de
Healthbook **ya está hecho**: una API de servicio en
`c:\Projects\LIS\lis-healthbook\lis-healthbook-back` (sección "API de
integración" de su README, leerla entera antes de empezar). Lo que falta vive
en el workspace del LIS y se describe acá.

Resultados e informes **quedan afuera**: el LIS todavía no los emite. Esta
etapa cubre alta y modificación de orden, cambio de paciente y la consulta de
cuenta del portal.

## Qué hay que construir

1. **`lis-adapters/lis-adapter-healthbook`** (NestJS, mismo molde que
   `lis-adapter-labcore`): stateless, recibe del orchestrator la orden canónica
   y la traduce a la API de Healthbook. Más una cara sincrónica para lis-backend.
2. **`lis-orchestrator`**: el bloque de variables del proveedor nuevo, nada más
   (ver "Sumar un proveedor" en su README).
3. **`lis-backend`**: el proveedor `healthbook` en el seeder, su configuración
   de adapter sincrónico, y un cliente + endpoint para que la ficha del
   paciente y la admisión muestren si tiene Healthbook.
4. Reglas de ruteo en el LIS: `order.created`, `order.updated` y
   `order.patient_reassigned` → `healthbook`. **No** `order.samples_registered`
   (Healthbook no modela muestras).

Convenciones: comentarios en castellano, commits en inglés con conventional
commits y scope, README de cada repo actualizado, tests unitarios con jest como
en el adapter de Labcore.

## La API de Healthbook (lo que el adapter consume)

Base: `HEALTHBOOK_API_URL` + `/api/v1/integration`. Header `X-Internal-Token`
con el token de un cliente creado en Healthbook con
`php artisan integration:client-create lis-adapter-healthbook arenco`. El
cliente ya tiene fijado el proveedor `arenco`; el adapter no lo manda.

| | Para qué | Respuesta |
|---|---|---|
| `PUT /orders/{number}` | La orden completa (paciente, cabecera, estudios). Idempotente: la misma foto no cambia nada; una foto nueva agrega y da de baja estudios. | `201` creada / `200` sincronizada, con `order.id`, `patient.id`, `patient.matched_by`, `studies.{created,updated,removed}` |
| `PUT /orders/{number}/patient` | La orden pasa a otro paciente. Cuerpo: `{ "patient": {…} }` | `200` con `previous_patient_id` y `patient` |
| `GET /patients/account?external_id=` o `?document_type=&document_number=&birth_date=` | Si el paciente existe y qué cuenta tiene en el portal | `200` siempre; `found: false` si no está |

Errores con el sobre `{ error: { code, message, details? } }`:

| Status | `code` | Qué hacer en el adapter |
|---|---|---|
| 422 | `ValidationFailed` (details por campo) | `422 VALIDATION_ERROR`, no reintentar |
| 409 | `OrderOwnedByOtherProvider`, `AmbiguousPatient` | `422 REJECTED_BY_PROVIDER`, no reintentar |
| 401 / 403 | `Unauthenticated` / `Forbidden` | `422 PROVIDER_AUTH_ERROR`, no reintentar |
| 429 / 5xx / sin conexión | | `503 SERVICE_UNAVAILABLE`, reintenta |
| timeout | | `504 TIMEOUT`, reintenta |

Cuerpo de `PUT /orders/{number}` (todos los campos, para la traducción de abajo):

```json
{
  "patient": {
    "external_id": "4411", "number": "30463017",
    "first_name": "Ana", "last_name": "Pérez", "birth_date": "1979-02-03", "gender": "female",
    "email": "ana@example.com", "phone": null, "mobile": "5491144440002", "active": true,
    "identifications": [{ "type": "DNI", "number": "30463017" }]
  },
  "order": {
    "origin_code": "B", "origin_name": "BORATTI",
    "service": "Ambulatorio", "diagnostic": "Control",
    "type": "outPatient", "lis_type": "AMBULATORIO",
    "date": "2026-10-08", "promised_for": "2026-10-10",
    "status": "new",
    "physician": { "external_id": 104, "name": "GOMEZ MUÑOZ GUSTAVO JAVIER", "license": "4227" }
  },
  "studies": [
    { "code": "660105", "name": "CULTIVO DE ESPUTO", "ordinal": 1, "status": "pending", "promised_for": "2026-10-10", "priority": "normal" }
  ]
}
```

## Traducción canónico → Healthbook

El payload canónico es el que arma `OrderIntegrationPayloadBuilder` en
lis-backend (tipo en `lis-adapter-labcore/src/domain/canonical-order.ts`,
copiarlo). Los códigos de catálogo **no los decide el adapter**: vienen
resueltos en `payload.mappings.healthbook.codes[entity][lis_id]` con la
política del proveedor aplicada, igual que en Labcore (`CodeMap`). Política
`fail` con registros en `unmapped` → `422 VALIDATION_ERROR` sin llamar.

| Healthbook | Canónico | Notas |
|---|---|---|
| `{number}` | `order.number` | |
| `patient.external_id` | `patient.id` como string | Es lo que Healthbook guarda en `origin_number` con `provider = arenco`. |
| `patient.number` | número de la identificación primaria (`is_primary`), o la de tipo `DNI` | **Es el número de documento**, no `patient.number` del LIS. Healthbook lo guarda en `patients.number` y lo usa para empadronar. Sin documento no se puede cruzar con las fichas viejas: mandar siempre que exista. |
| `patient.first_name` | `first_name` (+ `middle_name` si hay) | |
| `patient.last_name`, `birth_date`, `gender`, `email`, `phone`, `mobile` | ídem | `gender` `male`/`female` tal cual. |
| `patient.active` | `true` | |
| `patient.identifications[]` | `identifications[]` | `type` = `codes.identification_type[type.id]` (passthrough: `DNI`, `CI`, `PAS`). |
| `order.origin_code` | `codes.origin[origin.id]` | Los heredados coinciden (`B`, `C`, `D`…). |
| `order.origin_name` | `origin.name` | Si Healthbook no conoce el código, lo crea con este nombre. |
| `order.service` | `service.name` | Texto para mostrar. |
| `order.type` | `inPatient` si `codes.order_type[order_type.id]` está en el setting `inpatient_order_type_codes`, si no `outPatient` | Setting del proveedor en el LIS (ver seeder). Sin tipo de orden → `outPatient`. |
| `order.lis_type` | `order_type.name` | Healthbook lo muestra tal cual (`AMBULATORIO`, `INTERNADO`…). |
| `order.date` | fecha de `order.ordered_at` | `Y-m-d`. |
| `order.promised_for` | fecha de `order.promised_for` | |
| `order.diagnostic` | `order.diagnosis`, si no `clinical_data` | Máximo 255. |
| `order.status` | `order.status` tal cual | Healthbook acepta snake_case (`in_progress`). No mandar `admin_status` ni `released`: los maneja Healthbook. |
| `order.physician` | `null` si no hay médico o su código resolvió `null` (política `omit`) | Si va: `external_id` = `codes.physician[physician.id]` **como entero** (es el `lis_id` de Healthbook, el id de Labcore que los dos LIS comparten), `name` = `physician.name`, `license` = `state_number` ?? `national_number`. **Nunca** usar la matrícula como clave: se repite entre personas. |
| `studies[]` | `studies[]` por `ordinal`, sin repetir código | `code` = `codes.study_type[study_type.id]` (código `null` → `422 VALIDATION_ERROR`), `name` = `study_type.name`, `ordinal`, `status` tal cual, `promised_for` fecha, `priority`. |

`event_type` del orchestrator:

| `event_type` | Llamada |
|---|---|
| `send-order` (default) y `update-order` | `PUT /orders/{number}` con la foto completa |
| `reassign-patient` | `PUT /orders/{number}/patient` con `patient` traducido igual que arriba. `previous_patient` del payload no se usa. |
| otro | `422 UNSUPPORTED_EVENT_TYPE` |

Respuesta al orchestrator, como Labcore:

```json
{ "data": { "external_id": "9001", "status": "created", "references": { "patient_external_id": "4411" }, "raw_response": {…}, "sent_payload": {…} } }
```

`external_id` = `order.id` de Healthbook; `status` `created` con 201 y
`updated` con 200; `references.patient_external_id` = `patient.id` de la
respuesta, que lis-backend guarda en `patient_external_references` bajo el
proveedor `healthbook` y vuelve en `patient.external_references.healthbook`
en los eventos siguientes (hoy el adapter no lo necesita; queda para la
consulta de cuenta).

## Cara sincrónica del adapter (para lis-backend)

`GET /api/v1/patients/{external_id}/account?document_type=&document_number=&birth_date=`
con `X-Internal-Token`. `external_id` es el id del paciente en el LIS; el
adapter llama a `GET /integration/patients/account` de Healthbook con ese
`external_id` y, como respaldo, documento y nacimiento (Healthbook prueba
primero por id y después por documento). Devuelve la respuesta de Healthbook
tal cual bajo `data`. Errores con el mismo sobre que el envío.

No implementar `GET /api/v1/patients` (la búsqueda de Labcore): Healthbook no
es maestro de pacientes. En lis-backend `patient_lookup` queda en `false`.

## Modos y variables

```
PORT=3022
INTERNAL_TOKEN=lis-adapter-healthbook-local-token
HEALTHBOOK_MODE=stub|http
HEALTHBOOK_API_URL=http://localhost:8001
HEALTHBOOK_API_TOKEN=            # el que da integration:client-create
HEALTHBOOK_API_TIMEOUT_MS=10000
HEALTHBOOK_INPATIENT_ORDER_TYPE_CODES=I,G,N,H,INT   # fallback del setting del LIS
```

`stub` contesta con la misma forma que Healthbook (ids incrementales,
`created` la primera vez por número) y tiene un marcador de falla en las notas
para probar reintentos, como Labcore.

## lis-orchestrator

Bloque `HEALTHBOOK_ENABLED`, `HEALTHBOOK_ADAPTER_URL`,
`HEALTHBOOK_ADAPTER_INTERNAL_TOKEN`, `HEALTHBOOK_ADAPTER_TIMEOUT_MS` en
`.env.example` y `validation.schema.ts`; `healthbook: provider('HEALTHBOOK')`
en `configuration.ts`. Nada más.

## lis-backend

1. **Seeder** `IntegrationProvidersSeeder`: proveedor `code = healthbook`,
   `name = Healthbook`, políticas `passthrough` en todo salvo `physician` en
   `omit` (un médico sin `code` viaja como orden sin médico), `settings`
   `['inpatient_order_type_codes' => 'I,G,N,H,INT']`. Las reglas de ruteo
   (`IntegrationRulesSeeder` o el seeder de CEBAC): los tres eventos de arriba
   → `healthbook`.
2. **Config** `services.integration_adapters.healthbook`: `base_url`
   (`LIS_ADAPTER_HEALTHBOOK_BASE_URL`, default `http://127.0.0.1:3022`),
   `internal_token`, `timeout_seconds` 2, `patient_lookup` false.
3. **Cliente sincrónico** `App\Services\Integrations\Patients\HealthbookAccountClient`
   (misma forma que `AdapterPatientClient`: timeout corto, cualquier falla
   sube como excepción propia) y un servicio que arme la respuesta
   best-effort: si el adapter no está, `available: false` y la pantalla sigue.
   Endpoint `GET /api/v1/patients/{patient}/portal-status` (permiso de ver
   pacientes). Manda `external_id` = id del paciente del LIS, y documento +
   nacimiento de la identificación primaria.
4. **Front** (después, no bloquea): indicador en la ficha del paciente y en la
   admisión con lo que devuelve `portal-status`: tiene Healthbook, cuenta
   habilitada y empadronada, canal verificado, última actividad.

## Datos reales a tener en cuenta

Comprobado contra la copia de producción de CEBAC (2026-10-09):

- `physicians.lis_id` de Healthbook == `physicians.code` del LIS para los
  médicos importados (104, 1266, 4396 son la misma persona en ambas bases).
  Un médico creado en el LIS nuevo nace sin `code`: hasta que un administrador
  le cargue la equivalencia en *Admin > Integraciones > Healthbook*, viaja
  sin médico.
- Las matrículas se repiten entre personas (171 en el LIS, 167 en Healthbook)
  y hay placeholders `-`. No sirven de clave.
- Los códigos de estudio son los de Labcore en los dos sistemas: passthrough.
- Orígenes heredados coinciden por código; `GM`, `GUARDIA` y `CONS_EXT` no
  existen en Healthbook y se crean al llegar con nombre.
- Estados: `orders.status` del LIS es `new | in_progress | cancelled |
  released | posted`; Healthbook acepta esos nombres.

## Para probar punta a punta

En `lis-healthbook-back`: `php artisan serve --port=8001`, crear el cliente
con `php artisan integration:client-create lis-adapter-healthbook arenco` y
poner el token en `HEALTHBOOK_API_TOKEN`. Ojo: el `.env` de ese repo apunta a
una copia de la base productiva (RDS `healthbook-test`); para probar sin
escribir ahí, cambiar `DB_*` a una base local antes.

Después, con Redis, lis-backend (`DOMAIN_EVENTS_ENABLED=true`), el
orchestrator y el adapter levantados, crear una orden desde el LIS y mirar la
trazabilidad en `GET /api/v1/orders/{order}/integrations`.
