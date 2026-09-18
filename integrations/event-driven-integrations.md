# Integraciones con proveedores: orchestrator y adapters

Cómo una orden creada en el LIS llega a un proveedor externo (hoy Labcore),
cómo le siguen sus cambios (estudios que entran y salen) y su toma de
muestra, cómo el usuario ve en qué quedó, y cómo la admisión busca pacientes
en Labcore antes de darlos de alta. Este documento es el modelo tal como quedó
implementado; los detalles de cada pieza están en el README de su repo.

| Pieza | Repo | Rol |
| --- | --- | --- |
| Backend | `lis-backend` | Crea la orden, **emite** `order.created`, `order.updated` y `order.samples_registered`, **guarda la trazabilidad** que le reportan, la expone a la UI. Para la búsqueda de pacientes le pega **directo al adapter**. |
| Orchestrator | `lis-orchestrator` | Lee los eventos, aplica las reglas, llama al adapter con reintentos, reporta cada paso al backend. Sólo para el flujo de eventos. |
| Adapter | `lis-adapters/lis-adapter-labcore` | Traduce la orden canónica al formato del proveedor y llama a su API; traduce los pacientes del proveedor al canónico del LIS. Stateless. Es el **único** que conoce la URL, la clave y el contrato de Labcore. |
| Labcore API | `C:\Projects\Customs\labcore api` (.NET, propia) | `POST /api/v1/orders` con `X-Api-Key`: alta idempotente por número de orden sobre la base del LIS Labcore. `PUT /api/v1/orders/{number}`: la misma sincronización, pero 404 si no existe. `PUT /api/v1/orders/{number}/samples`: estado de los tubos tras la toma. `GET /api/v1/patients`: búsqueda por prefijo en `Historias`. |

Hay dos formas de hablar con un proveedor, y no se mezclan:

- **Event-driven** (órdenes): backend → Redis → orchestrator → adapter →
  proveedor. Asincrónico, con reintentos y trazabilidad. Es el grueso de este
  documento.
- **Sincrónico** (búsqueda de pacientes): backend → adapter → proveedor, con
  un usuario esperando. Sin Redis ni orchestrator: no hay evento, no hay nada
  que reintentar ni auditar. Ver [Búsqueda de pacientes](#búsqueda-de-pacientes-en-el-proveedor-sincrónico).

## Flujo

```
  Admisión                lis-backend                Redis               lis-orchestrator            lis-adapter-labcore        Labcore
     │  POST /orders           │                        │                        │                            │                    │
     ├────────────────────────►│ INSERT orders           │                        │                            │                    │
     │                         │ commit                  │                        │                            │                    │
     │                         ├─XADD lis:events:order──►│                        │                            │                    │
     │◄──── 201 ───────────────┤ (integration_logs:     │                        │                            │                    │
     │                         │  una fila por proveedor)│                        │                            │                    │
     │                         │                        │◄──XREADGROUP───────────┤                            │                    │
     │                         │                        │   (grupo lis-orchestrator)                          │                    │
     │                         │◄─POST …/events/routed──────────────────────────┤ regla: send-order-to-labcore│                    │
     │                         │◄─POST …/events/pending─────────────────────────┤ (held si el proveedor está  │                    │
     │                         │  (confirma la fila)     │                        │  en pausa: no envía)        │                    │
     │                         │◄─POST …/events/sent────────────────────────────┤                            │                    │
     │                         │                        │                        ├─POST /api/v1/orders───────►│                    │
     │                         │                        │                        │                            ├─(traduce, llama)──►│
     │                         │                        │                        │                            │◄── orderId ────────┤
     │                         │                        │                        │◄─ 200 {external_id} ───────┤                    │
     │                         │◄─POST …/events/ack─────────────────────────────┤                            │                    │
     │                         │                        │◄──XACK─────────────────┤                            │                    │
     │  GET /orders/77/integrations                     │                        │                            │                    │
     ├────────────────────────►│                        │                        │                            │                    │
     │◄─ [{status: received, external_id: orderId, sent_at, received_at, payload_to_send, payload_received}]      │                    │
```

El usuario no espera a nada de esto: el `201` vuelve apenas commitea la
orden. Con el stub, la trazabilidad queda en `received` unos segundos después
(el consumer bloquea hasta 5 s esperando eventos, no hace polling).

Los otros dos eventos recorren exactamente el mismo camino; sólo cambia lo
que el adapter hace al final:

| Evento del LIS | Cuándo lo emite el backend | `event_type` de la regla | Adapter → Labcore |
| --- | --- | --- | --- |
| `order.created` | `POST /orders`, tras el commit | `send-order` | `POST /api/v1/orders` |
| `order.updated` | `POST /orders/{id}/studies`, `DELETE /orders/{id}/studies/{study}`, `PATCH /orders/{id}` | `update-order` | `PUT /api/v1/orders/{number}` (si Labcore no la tiene, `POST`) |
| `order.samples_registered` | `POST /orders/{id}/samples/register` con al menos una decisión | `update-samples` | `PUT /api/v1/orders/{number}/samples` (si Labcore no la tiene, primero `POST`) |

Cada evento necesita su propia regla en el ABM (el seeder crea las tres para
Labcore). Sin regla para `order.updated`, por ejemplo, los cambios de
estudios no salen y el orchestrator los marca `skipped`.

## Decisiones

- **El adapter no habla con el backend.** El único que reporta es el
  orchestrator: reintentos, backoff y credenciales del backend en un solo
  lugar. El adapter recibe una orden y contesta un id o un error tipado.
- **La regla "esta orden va a Labcore" se administra en el LIS**
  (`integration_rules`, en *Sistema → Integraciones → Orquestador*: evento
  → proveedor, filtros por origen y servicio, activa). El orchestrator la
  aplica: la lee de `GET /internal/integrations/config` y la relee cada
  `orchestrator.config_refresh_seconds`; el `.env` sólo dice dónde está cada
  adapter. Los valores de despacho (intentos, backoff) viajan en la misma
  respuesta desde `application_settings` (`orchestrator.*`). Qué eventos
  existen y qué llevan sigue siendo código (`IntegrationEvents`): es el
  contrato con los adapters. No hay columna `orders.destination`: si algún
  día el destino lo elige el usuario en el formulario, se agrega la columna
  y la regla pasa a leerla del payload.
- **`orders.status` no se toca.** El estado de integración vive en
  `integration_logs`; el estado de la orden es del laboratorio.
- **El sujeto es polimórfico.** `integration_logs.subject_type/subject_id`
  (hoy `order`); el día que se informen resultados entran por la misma tabla
  con otro `event_name`, sin tocar el orchestrator.
- **El LIS registra antes de publicar.** Una fila por proveedor con una
  regla activa para el evento, así lo que no llegó al stream o quedó
  retenido se ve y se reemite. El orchestrator confirma (`pending`) y avisa
  a quién ruteó de verdad (`routed`).
- **El evento lleva la orden completa.** El orchestrator no vuelve a
  preguntar por ella, y lo que se audita como `payload_to_send` es exactamente
  lo que salió del backend. El contrato es `OrderIntegrationPayloadBuilder`
  (backend) ↔ `CanonicalOrder` (adapter).
- **Los cambios viajan como foto, no como delta.** `order.updated` no dice
  "se agregó el estudio X": lleva la orden entera tal como quedó, y la Labcore
  API calcula qué entra y qué sale (sin borrar nunca una prueba con resultado
  validado). Lo mismo `order.samples_registered`: todas las muestras con su
  estado, no sólo las que se tocaron. Así un evento que se perdió o llegó
  desordenado se corrige solo con el siguiente, y un reenvío manual siempre
  manda el estado actual. Decisión del usuario (2026-09-17) frente a eventos
  granulares por estudio y por tubo.
- **El adapter completa lo que falte.** Un `update-*` sobre una orden que
  Labcore nunca recibió (el alta falló y nadie la reenvió) no es un error: el
  adapter la da de alta y sigue. El `PUT` de la Labcore API sí devuelve 404,
  para que un consumidor que no es el adapter se entere.
- **Sin tabla de alertas por ahora.** `GET /api/v1/integrations/logs?status=error`
  cubre lo mismo hasta que exista un sistema de alertas real.
- **La emisión nunca falla la orden.** Si Redis no está, la orden se crea
  igual y la fila queda `held` por `publish_failed`: sale al reanudar el
  proveedor o al reenviar.

## Mapeo de códigos de catálogo

Cada proveedor habla con sus propios códigos (`Estudios.e_codigo`,
`Medicos.m_codigo`…). La equivalencia se **administra desde el LIS** y el
backend la resuelve al emitir el evento, así lo que salió queda en
`payload_to_send` y corregir un mapeo + **Reintentar** alcanza.

```
integration_providers       code, name, is_active, policies {entity: política}, settings {clave: valor}
integration_code_mappings   provider_id, entity, lis_id, external_code, is_active, notes
```

Entidades mapeables: `study_type`, `test_type`, `sample_type`, `physician`
(su "código LIS" es la matrícula nacional), `service`, `origin`,
`insurance_type`, `insurance_plan`, `identification_type`, `order_type`.

Política por entidad para lo que **no** tiene equivalencia:

| Política | Qué viaja | Para qué |
| --- | --- | --- |
| `passthrough` | el código del LIS | catálogos que coinciden (Labcore heredó los de CEBAC) |
| `omit` | nada | datos opcionales para el proveedor (el médico, hoy) |
| `fail` | nada, y el registro va en `unmapped` | el adapter rechaza la orden nombrando lo que falta |

El bloque que viaja en el evento, por proveedor activo:

```json
"mappings": {
  "labcore": {
    "policies": { "study_type": "passthrough", "physician": "omit", … },
    "settings": { "patient_type_code": null, "priority_code_urgent": null, … },
    "codes": { "study_type": { "551": "660015" }, "physician": { "2": null }, … },
    "unmapped": { "physician": [ { "id": 2, "code": "3894", "name": "…" } ] }
  }
}
```

Administración (`integrations.manage`), bajo `/api/v1/admin/integrations`:

| Método y ruta | Para qué |
| --- | --- |
| `GET /providers` | Proveedores con política por entidad, settings y cuántos mapeos tiene cada entidad. `meta.entities` lista las entidades. |
| `POST /providers`, `PATCH /providers/{id}` | Alta y edición (`policies`, `settings`, `is_active`). |
| `GET /providers/{id}/mappings/{entity}?search=&unmapped=1&include_inactive=` | El catálogo del LIS con su equivalencia al lado, paginado. `unmapped=1` es la lista de lo que falta. |
| `PUT /providers/{id}/mappings/{entity}` | Upsert en lote `items: [{lis_id, external_code, is_active?, notes?}]`; `external_code` vacío borra. |
| `DELETE /providers/{id}/mappings/{entity}/{lisId}` | Borra una. |
| `POST /providers/{id}/mappings/{entity}/copy-lis-codes` | Deja el código del LIS como equivalencia en todo lo no mapeado. |
| `POST /providers/{id}/mappings/{entity}/import` (`file`) | CSV `lis_code,external_code[,notes]` (o `lis_id`). Devuelve cargados y rechazados con línea y motivo. |
| `GET /providers/{id}/mappings/{entity}/export` | CSV con el catálogo y su equivalencia. |

`IntegrationProvidersSeeder` deja a Labcore con `passthrough` en todo y
`omit` en médico.

## Búsqueda de pacientes en el proveedor (sincrónico)

**Labcore es el maestro de pacientes.** El LIS los va dando de alta a medida
que llegan. **Labcore no guarda ningún id del LIS** (su esquema no tiene
`h_external_id` ni nada parecido): el enlace vive sólo acá, en
`patient_external_references`, con el `h_id` que el LIS aprende de la búsqueda
o del `ack` de la primera orden. La búsqueda evita crear una Historia nueva por
cada paciente histórico.

```
  Admisión                 lis-backend                          lis-adapter-labcore        Labcore API
     │ GET /patients/lookup?search=per │                               │                       │
     ├────────────────────────────────►│ SELECT patients (LIKE %per%)   │                       │
     │                                 ├─GET /api/v1/patients?last_name=per───────────────────►│                       │
     │                                 │                               ├─GET /api/v1/patients─►│ Historias LIKE 'per%'
     │                                 │                               │◄─ items (h_id, doc…) ─┤
     │                                 │◄─ {items: [ExternalPatient]}  (códigos de Labcore) ───┤
     │                                 │ códigos → ids LIS (mapeos inversos)                    │
     │                                 │ descarta los que ya están en el LIS (referencia o documento)
     │◄─ {local, external}, meta.external_unavailable ┤                                        │
     │                                                                                          │
     │ elige uno externo → alta del LIS prellenada (+ ID fotográfico, etc.)                     │
     │ POST /patients {…, external_references: [{provider: labcore, external_id: h_id}]}        │
     ├────────────────────────────────►│ INSERT patients + patient_external_references           │
```

- **`GET /api/v1/patients/lookup?search=&limit=`** (mismo auth que
  `/patients`). Devuelve `data.local` (pacientes del LIS, misma forma que
  `GET /patients`) y `data.external` (candidatos del proveedor, ya
  traducidos). `meta.external_unavailable` avisa que el adapter o Labcore no
  contestaron: la búsqueda local **nunca** depende de la externa (timeout de
  2 s, `LIS_ADAPTER_LABCORE_TIMEOUT_SECONDS`). `meta.external_truncated`
  avisa que Labcore devolvió el máximo pedido.
- **Criterios**: sólo dígitos → documento; texto → `Apellido[,] [Nombre]`.
  Menos de 3 caracteres no viaja. En Labcore todo es **prefijo**
  (`Historias` es toda la historia del laboratorio; un `%x%` la recorre
  entera); en el LIS sigue siendo "contiene".
- **Deduplicación**: un candidato cuyo `h_id` ya está en
  `patient_external_references`, o cuyo documento (tipo traducido + número)
  ya existe en `identifications`, no se ofrece como externo: si la búsqueda
  local no lo trajo, se agrega a `local` igual.
- **Traducción inversa de códigos**: el adapter devuelve los códigos de
  Labcore tal cual (`document.type_code`, `coverage.code`); el backend los
  resuelve con `integration_code_mappings` (`external_code → lis_id`) y, si no
  hay equivalencia, por `passthrough` sobre el código del LIS
  (`IntegrationCodeReverseResolver`). Lo que no se pudo traducir viaja como
  `null` y el formulario lo pide.
- Cada externo trae `prefill`: el cuerpo listo para `POST /patients`
  (`identifications`, `insurances`, `external_references`).
- **Flag**: `LIS_ADAPTER_LABCORE_PATIENT_LOOKUP` en el backend. Apagado, el
  lookup devuelve sólo lo local y `meta.external_providers = []`.
- **UI**: en *Admisión por pasos*, *Admisión rápida* y el alta de *Órdenes
  médicas*, el modal de coincidencias muestra los del LIS como siempre y,
  debajo, "En Labcore, sin alta en el LIS" con el botón *Dar de alta desde
  Labcore*, que abre el alta prellenada. El ABM de *Pacientes* busca sólo en
  el LIS, a propósito: ahí se administra lo que ya está dado de alta.

### `patient_external_references`

```
patient_id → patients, provider_id → integration_providers,
external_id (h_id), external_number (historia clínica, para mostrar),
source ('lookup' | 'order_ack'), linked_at
unique (patient_id, provider_id), unique (provider_id, external_id)
```

Cómo nace una referencia:

| Origen | Cuándo |
| --- | --- |
| `lookup` | Admisión eligió al paciente de la búsqueda en Labcore y lo dio de alta con `external_references` en el `POST /patients`. El request rechaza (`422`) un `external_id` que ya sea de otro paciente. |
| `order_ack` | El paciente nació en el LIS; en la primera orden Labcore creó su Historia y devolvió `patientId`. El adapter lo normaliza como `references.patient_external_id`, el orchestrator lo reenvía en el `ack` y el backend lo guarda si el paciente no tenía referencia. |

Nunca se pisa: si el paciente ya tiene otra referencia en ese proveedor, o el
id externo es de otro paciente, se loguea un warning y no se toca.

### Cómo viaja en el alta de orden

El payload canónico lleva `patient.external_references = {labcore: {external_id, external_number}}`.
El adapter lo manda como `patient.id` (`h_id`) en el `CreateOrderRequest`; el
id del LIS no viaja. Labcore resuelve al paciente en este orden: una orden que
ya existe conserva el suyo; con `patient.id`, esa Historia (si no existe,
`400` antes de escribir nada); sin `patient.id`, busca por tipo y número de
documento — una coincidencia la reusa, más de una rechaza con `400` en
`patient.documentNumber` para que admisión elija con la búsqueda —; y si no
hay ninguna, la crea. En todos los casos el `h_id` vuelve en el `ack`.

`GET /patients/{id}` y `POST /patients` devuelven `external_references`.

## Estados de `integration_logs`

```
            ┌─► held ──(reanudar/reenviar)──┐
pending_send┤                                ├──► sent ──► received
            └─► skipped                      │       └────► error
                                             ▼
                                        pending_send
```

| Estado | Quién lo pone | Cuándo |
| --- | --- | --- |
| `pending_send` | backend (al emitir) | Registrado para ese proveedor; `published_at` y `stream_entry_id` cuando llegó al stream. El `…/pending` del orchestrator lo confirma (idempotente por `event_id` + `provider`). |
| `held` | backend | No salió: `held_reason` = `provider_paused` (el proveedor está en pausa) o `publish_failed` (Redis no estaba). Se reemite al reanudar el proveedor. |
| `skipped` | orchestrator (`…/routed`) | Ninguna regla lo ruteó a ese proveedor. |
| `sent` | orchestrator (`…/sent`) | Está por llamar al adapter. Marca `sent_at`. |
| `received` | orchestrator (`…/ack`) | El proveedor aceptó. `external_id`, `payload_received`, `received_at`. |
| `error` | orchestrator (`…/error`) | Se agotaron los reintentos o el proveedor rechazó. `error_code`, `error_message`, `payload_received`, `failed_at`. |

Cada fila es (evento, proveedor). Un **reenvío manual**
(`POST /api/v1/orders/{id}/integrations/{provider}/retry`, permiso
`integrations.manage`) publica un evento nuevo con `attempt_number + 1`,
`retry_of_event_id` y `provider` fijado, y termina en una fila nueva: la
historia completa queda a la vista. Reenvía **el último evento** que se le
mandó a ese proveedor (si lo último fue la toma de muestra, sale
`order.samples_registered`), con el payload armado de nuevo sobre el estado
actual de la orden. Si el proveedor está en pausa, la fila nueva queda `held`.

## Pausas

El tablero *Instrumentos y conexiones* tiene dos palancas (permiso
`integrations.manage`, quedan en `operation_logs` con quién las accionó):

| Palanca | Endpoint del LIS | Dónde vive | Efecto |
| --- | --- | --- | --- |
| Pausar la **lectura** del orquestador | `POST /api/v1/integrations/orchestrator/consumer/pause` / `resume` | Redis (`lis:events:order:paused`), vía el orchestrator | Deja de leer; lo en vuelo termina. Los eventos se acumulan en el stream (lag) y se drenan en orden al reanudar. Sobrevive a un reinicio y vale para todas las instancias. |
| Pausar los **envíos a un proveedor** | `POST /api/v1/integrations/providers/{code}/pause` / `resume` | `integration_providers.paused_at/paused_by` | Lo que se emite queda `held`; lo que el orchestrator ya tenía en la cola también (el `pending` contesta `held: true`). Al reanudar, el LIS reemite lo retenido en orden, dirigido a ese proveedor, con el payload armado de nuevo. La búsqueda de pacientes no se ve afectada. |

La cola no tiene pausa propia: frenar la escritura perdería órdenes; lo que
se quiere es frenar la lectura. Apagar el proceso del adapter no es una
pausa: produce `error` tras los reintentos.

## Errores y reintentos

| Qué falló | Qué pasa |
| --- | --- |
| El adapter/proveedor no responde o devuelve 5xx/408/429 | El orchestrator reintenta (`DISPATCH_MAX_ATTEMPTS`, backoff exponencial desde `DISPATCH_BACKOFF_BASE_MS`) y cierra en `error` con `SERVICE_UNAVAILABLE` / `TIMEOUT`. |
| El adapter devuelve 422 (`VALIDATION_ERROR`, `REJECTED_BY_PROVIDER`, `PROVIDER_AUTH_ERROR`) | `error` sin reintentar: hay que corregir la orden o el mapeo y reenviar. |
| El backend no acepta un webhook | El evento va a `lis:events:order:dead` con el motivo y se confirma; no traba la cola. |
| El orchestrator se cae a mitad de un evento | Al reiniciar retoma lo que dejó sin confirmar; lo de instancias muertas se reclama por autoclaim. |
| Redis no está cuando se crea la orden | La orden se crea y la fila queda `held` (`publish_failed`). Sale al reanudar el proveedor o al reenviar. |

Un evento reprocesado tras una caída puede llegar dos veces al proveedor. El
`pending` del backend es idempotente por `event_id` y el alta en la Labcore
API lo es por número de orden (la segunda vez responde `200` y actualiza), así
que no se duplican órdenes. Un adapter de otro proveedor sin esa garantía
tendría que deduplicar por `event_id`.

## Endpoints

### lis-backend

| Método y ruta | Auth | Para qué |
| --- | --- | --- |
| `GET /api/v1/integrations/status` | `integrations.view` | Estado consolidado para el tablero: orchestrator (`/api/v1/status`: consumer, contadores, health de adapters), stream en Redis (largo, lag, pendientes, cola muerta) y envíos de las últimas 24 h. |
| `GET /api/v1/integrations/logs/{log}` | `integrations.view` | Una fila con sus payloads. |
| `GET /api/v1/internal/integrations/config` | `X-Internal-Token` (`lis_orchestrator`) | Reglas vigentes, valores de despacho y `version`; el orchestrator lo relee cada `refresh_seconds`. |
| `POST /api/v1/internal/integrations/events/routed` | ídem | A qué proveedores se ruteó el evento; el resto queda `skipped`. |
| `POST /api/v1/internal/integrations/events/pending` | ídem | Confirma/crea la fila en `pending_send`. Responde `held: true` si el proveedor está en pausa. |
| `POST /api/v1/internal/integrations/events/sent` | ídem | Marca `sent`. |
| `POST /api/v1/internal/integrations/events/ack` | ídem | Marca `received`. Si trae `references.patient_external_id`, enlaza al paciente (`order_ack`). |
| `POST /api/v1/internal/integrations/events/error` | ídem | Marca `error`. |
| `GET /api/v1/patients/lookup` | usuario | La búsqueda de admisión: `local` + `external` (Labcore) + `meta.external_unavailable`. |
| `GET /api/v1/orders/{order}/integrations` | usuario | La trazabilidad de una orden, con payloads. |
| `POST /api/v1/orders/{order}/integrations/{provider}/retry` | `integrations.manage` | Reenvía. Responde `202` con la fila nueva; si es `held`, no salió todavía. |
| `GET /api/v1/integrations/logs` | `integrations.view` | Listado global, filtros `status`, `provider`, `subject_type`, `subject_id`, `external_id`, `from`, `to`. Sin payloads. |
| `POST /api/v1/integrations/orchestrator/consumer/pause` / `resume` | `integrations.manage` | Frena / reanuda la lectura del orchestrator. |
| `POST /api/v1/integrations/providers/{code}/pause` / `resume` | `integrations.manage` | Frena / reanuda los envíos a un proveedor; `resume` devuelve `released` y `still_held`. |
| `GET /api/v1/admin/integrations/orchestrator` | `integrations.manage` | El orquestador para el ABM: estado, valores de despacho, `config_version`, reglas, y en `meta` el catálogo de eventos, los proveedores y los códigos de origen/servicio para filtrar. |
| `PATCH /api/v1/admin/integrations/orchestrator/settings` | ídem | `dispatch_max_attempts`, `dispatch_backoff_base_ms`, `dispatch_backoff_max_ms`, `config_refresh_seconds`. |
| `POST` / `PATCH` / `DELETE /api/v1/admin/integrations/rules[/{rule}]` | ídem | Las reglas. `event_type` sale del evento; `name` en minúsculas y guiones. |

### lis-orchestrator

`GET /health`, `GET /ready`, `GET /api/v1/rules` (reglas y proveedores cargados),
`GET /api/v1/status`, `POST /api/v1/consumer/pause` / `resume` (con
`X-Internal-Token` = `BACKEND_INTERNAL_TOKEN`).

### lis-adapter-labcore

`GET /health`, `GET /ready`, `POST /api/v1/orders`, `GET /api/v1/patients`,
`GET /api/v1/patients/{external_id}` (todos con `X-Internal-Token`).

## Evento en el stream

`XADD lis:events:order` con campos planos:

| Campo | Ejemplo |
| --- | --- |
| `event_id` | `evt_01m270yjmbqj79k796805ag8bm` (ULID) |
| `event_name` | `order.created`, `order.updated` u `order.samples_registered` |
| `aggregate_type` / `aggregate_id` | `order` / `26` |
| `occurred_at` | ISO-8601 |
| `attempt_number` | `1` (`n+1` en reenvíos) |
| `retry_of_event_id` | vacío, o el `event_id` anterior |
| `provider` | vacío (a quien diga la regla) o el código del proveedor al que va dirigido |
| `source` | `lis-backend` |
| `payload` | JSON con `order`, `patient`, `insurance`, `physician`, `studies[].tests[]`, `samples[]`, `mappings` |

El stream va **sin el prefijo** que Laravel le pone a sus claves (conexión
`events` en `config/database.php`), así el nombre es el mismo de los dos lados.

## Configuración que tiene que coincidir

| lis-backend | lis-orchestrator | lis-adapter-labcore |
| --- | --- | --- |
| `DOMAIN_EVENTS_ORDER_STREAM` | `DOMAIN_EVENTS_ORDER_STREAM` | — |
| reglas y `orchestrator.*` (ABM) | `CONFIG_REFRESH_MS` sólo hasta la primera lectura | — |
| `LIS_ORCHESTRATOR_INTERNAL_TOKEN` | `BACKEND_INTERNAL_TOKEN` | — |
| `LIS_ADAPTER_LABCORE_INTERNAL_TOKEN` | `LABCORE_ADAPTER_INTERNAL_TOKEN` | `INTERNAL_TOKEN` |
| `LIS_ADAPTER_LABCORE_BASE_URL`, `LIS_ADAPTER_LABCORE_PATIENT_LOOKUP`, `LIS_ADAPTER_LABCORE_TIMEOUT_SECONDS` | `LABCORE_ADAPTER_URL` | `PORT` (3021) |
| — | — | `LABCORE_MODE` (`stub` \| `http`), `LABCORE_API_URL`, `LABCORE_API_KEY` |

En producción todo sale del `.env` de `lis-infra`
(`LIS_ORCHESTRATOR_INTERNAL_TOKEN`, `LIS_ADAPTER_LABCORE_INTERNAL_TOKEN`,
`LABCORE_*`). El backend necesita además `REDIS_CLIENT=predis` y
`DOMAIN_EVENTS_ENABLED=true`.

## Pendiente

- **Probar contra una Labcore API real.** El cliente está hecho contra el
  contrato (`CreateOrderRequest`) y probado contra una API de juguete; falta
  correrlo contra una instancia con base Labcore y confirmar que los códigos
  de catálogo (estudios, tipos de muestra, centros, servicios, coberturas,
  tipos de documento) coinciden. Lo que no coincida se resuelve desde el LIS
  con las equivalencias y políticas por entidad (arriba), sin tocar el
  adapter. El adapter arranca en `LABCORE_MODE=stub`; contra la API real va
  `LABCORE_MODE=http` con `LABCORE_API_URL` y `LABCORE_API_KEY`.
- **El alta de orden contra la base real de Labcore.** La búsqueda ya corre
  contra la base real (SQL Server 2008 R2, 644 k historias, ~200 ms). El alta
  de orden todavía no: `Historias` **no tiene `h_external_id`** (la API lo
  asumía en `InsertPatient` y `GetPatientByExternalId`) y `h_ti_id` está
  vacío (el tipo de documento es `h_tipo_identificacion`, el `ti_id` como
  texto). Decisión tomada: el enlace vive sólo del lado del LIS
  (`patient_external_references`); Labcore no guarda nuestro id. Falta
  adaptar `ResolvePatientAsync`/`InsertPatient` a eso.
- **Documentos placeholder.** Labcore no exige documento: tipo `NN` y número
  `_159145`. El adapter los manda como `document: null`; el LIS no deduplica
  ni prellena con ellos. `h_numero` real viene sin puntos.
- **Resultados de vuelta** (`send-result` / Labcore → LIS): no está diseñado.
- **Codificación de `mo_estado` en Labcore.** La Labcore API traduce
  `pending` / `collected` / `cancelled` con `Lis:Defaults:SampleStates`
  (0 / 1 / 2 por defecto). El 0 es el del alta; los otros dos son una
  suposición: confirmarlos contra la base real antes de prender la regla
  `update-samples-in-labcore` en producción. Cancelar un tubo no cancela sus
  pruebas en `Laboratorios`.
- **Muestras nuevas después del alta.** Agregar un estudio en el LIS no crea
  muestras nuevas hoy; si algún día lo hace, `order.updated` ya las lleva y
  el `PUT` de Labcore las crea. Un tubo que Labcore no conoce en el
  `PUT …/samples` es un 400 (`REJECTED_BY_PROVIDER`): se resuelve reenviando
  la orden y después la toma.
- **UI, hecho**: *Instrumentos y conexiones → Integraciones* (tarjetas de
  orquestador, cola, adapters y envíos 24 h + tabla de eventos con detalle y
  reenvío), pestaña *Integraciones* en la ficha de la orden, y
  *Administración → Integraciones* (proveedores, políticas, valores fijos y
  grilla de equivalencias con import/export CSV). Falta que un `error` con
  *"Faltan equivalencias"* linkee a la grilla filtrada.
- **Alertas**: cuando haya un sistema de alertas, `error` debería generar una.
- **Sumar proveedores** (PACS, HIS, AMS): un bloque de config en el
  orchestrator y un adapter con el mismo contrato HTTP.
