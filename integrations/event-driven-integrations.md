# Integraciones con proveedores: orchestrator y adapters

Cómo una orden creada en el LIS llega a un proveedor externo (hoy Labcore),
cómo el usuario ve en qué quedó, y cómo la admisión busca pacientes en
Labcore antes de darlos de alta. Este documento es el modelo tal como quedó
implementado; los detalles de cada pieza están en el README de su repo.

| Pieza | Repo | Rol |
| --- | --- | --- |
| Backend | `lis-backend` | Crea la orden, **emite** `order.created`, **guarda la trazabilidad** que le reportan, la expone a la UI. Para la búsqueda de pacientes le pega **directo al adapter**. |
| Orchestrator | `lis-orchestrator` | Lee los eventos, aplica las reglas, llama al adapter con reintentos, reporta cada paso al backend. Sólo para el flujo de eventos. |
| Adapter | `lis-adapters/lis-adapter-labcore` | Traduce la orden canónica al formato del proveedor y llama a su API; traduce los pacientes del proveedor al canónico del LIS. Stateless. Es el **único** que conoce la URL, la clave y el contrato de Labcore. |
| Labcore API | `C:\Projects\Customs\labcore api` (.NET, propia) | `POST /api/v1/orders` con `X-Api-Key`: alta idempotente por número de orden sobre la base del LIS Labcore. `GET /api/v1/patients`: búsqueda por prefijo en `Historias`. |

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
     │◄──── 201 ───────────────┤                        │                        │                            │                    │
     │                         │                        │◄──XREADGROUP───────────┤                            │                    │
     │                         │                        │   (grupo lis-orchestrator)                          │                    │
     │                         │◄─POST …/events/pending─────────────────────────┤ regla: send-order-to-labcore│                    │
     │                         │  (crea order_integration_logs)                  │                            │                    │
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

## Decisiones

- **El adapter no habla con el backend.** El único que reporta es el
  orchestrator: reintentos, backoff y credenciales del backend en un solo
  lugar. El adapter recibe una orden y contesta un id o un error tipado.
- **La regla "esta orden va a Labcore" vive en el orchestrator**, por
  configuración (`LABCORE_ENABLED`, y filtros opcionales
  `LABCORE_ORIGIN_CODES` / `LABCORE_SERVICE_CODES`). No hay columna
  `orders.destination`: si algún día el destino lo elige el usuario en el
  formulario, se agrega la columna y la regla pasa a leerla del payload.
- **`orders.status` no se toca.** El estado de integración vive en
  `order_integration_logs`; el estado de la orden es del laboratorio.
- **El evento lleva la orden completa.** El orchestrator no vuelve a
  preguntar por ella, y lo que se audita como `payload_to_send` es exactamente
  lo que salió del backend. El contrato es `OrderIntegrationPayloadBuilder`
  (backend) ↔ `CanonicalOrder` (adapter).
- **Sin tabla de alertas por ahora.** `GET /api/v1/integrations/logs?status=error`
  cubre lo mismo hasta que exista un sistema de alertas real.
- **La emisión nunca falla la orden.** Si Redis no está, el backend loguea y
  la orden se crea igual; se reenvía a mano desde el LIS.

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
que llegan. Sin esto, el alta de orden mandaba `patient.externalId = id LIS`,
Labcore no encontraba ninguna Historia con ese `h_external_id` y **creaba una
nueva** para cada paciente histórico: un duplicado por paciente. La búsqueda
y la referencia externa cierran ese circuito.

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
El adapter lo manda como `patient.id` (`h_id`) en el `CreateOrderRequest`.
Labcore resuelve al paciente en este orden: por `patient.id` (y si esa
Historia no tenía `h_external_id`, la enlaza con `patient.externalId` en el
mismo alta); si no viene, por `h_external_id`; y si tampoco, crea. Un
`patient.id` inexistente rechaza la orden con `400` antes de escribir nada.

`GET /patients/{id}` y `POST /patients` devuelven `external_references`.

## Estados de `order_integration_logs`

```
pending_send ──► sent ──► received
                   └────► error
```

| Estado | Quién lo pone | Cuándo |
| --- | --- | --- |
| `pending_send` | orchestrator (`…/pending`) | Detectó el evento y una regla coincidió. Crea la fila con `payload_to_send`. Idempotente por `event_id`. |
| `sent` | orchestrator (`…/sent`) | Está por llamar al adapter. Marca `sent_at`. |
| `received` | orchestrator (`…/ack`) | El proveedor aceptó. `external_id`, `payload_received`, `received_at`. |
| `error` | orchestrator (`…/error`) | Se agotaron los reintentos o el proveedor rechazó. `error_code`, `error_message`, `payload_received`, `failed_at`. |

Cada fila es un evento. Un **reenvío manual**
(`POST /api/v1/orders/{id}/integrations/{provider}/retry`, permiso
`integrations.manage`) publica un evento nuevo con `attempt_number + 1` y
`retry_of_event_id`, y termina en una fila nueva: la historia completa queda
a la vista.

## Errores y reintentos

| Qué falló | Qué pasa |
| --- | --- |
| El adapter/proveedor no responde o devuelve 5xx/408/429 | El orchestrator reintenta (`DISPATCH_MAX_ATTEMPTS`, backoff exponencial desde `DISPATCH_BACKOFF_BASE_MS`) y cierra en `error` con `SERVICE_UNAVAILABLE` / `TIMEOUT`. |
| El adapter devuelve 422 (`VALIDATION_ERROR`, `REJECTED_BY_PROVIDER`, `PROVIDER_AUTH_ERROR`) | `error` sin reintentar: hay que corregir la orden o el mapeo y reenviar. |
| El backend no acepta un webhook | El evento va a `lis:events:order:dead` con el motivo y se confirma; no traba la cola. |
| El orchestrator se cae a mitad de un evento | Al reiniciar retoma lo que dejó sin confirmar; lo de instancias muertas se reclama por autoclaim. |
| Redis no está cuando se crea la orden | La orden se crea, el evento se pierde, queda el log de error del backend. Reenvío manual. |

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
| `POST /api/v1/internal/integrations/events/pending` | `X-Internal-Token` (`lis_orchestrator`) | Crea/actualiza la fila en `pending_send`. |
| `POST /api/v1/internal/integrations/events/sent` | ídem | Marca `sent`. |
| `POST /api/v1/internal/integrations/events/ack` | ídem | Marca `received`. Si trae `references.patient_external_id`, enlaza al paciente (`order_ack`). |
| `POST /api/v1/internal/integrations/events/error` | ídem | Marca `error`. |
| `GET /api/v1/patients/lookup` | usuario | La búsqueda de admisión: `local` + `external` (Labcore) + `meta.external_unavailable`. |
| `GET /api/v1/orders/{order}/integrations` | usuario | La trazabilidad de una orden, con payloads. |
| `POST /api/v1/orders/{order}/integrations/{provider}/retry` | `integrations.manage` | Reenvía. Responde `202` con el `event_id` nuevo; `503` si el bus no está. |
| `GET /api/v1/integrations/logs` | `integrations.view` | Listado global, filtros `status`, `provider`, `order_id`, `external_id`, `from`, `to`. Sin payloads. |

### lis-orchestrator

`GET /health`, `GET /ready`, `GET /api/v1/rules` (reglas y proveedores cargados).

### lis-adapter-labcore

`GET /health`, `GET /ready`, `POST /api/v1/orders`, `GET /api/v1/patients`,
`GET /api/v1/patients/{external_id}` (todos con `X-Internal-Token`).

## Evento en el stream

`XADD lis:events:order` con campos planos:

| Campo | Ejemplo |
| --- | --- |
| `event_id` | `evt_01m270yjmbqj79k796805ag8bm` (ULID) |
| `event_name` | `order.created` |
| `aggregate_type` / `aggregate_id` | `order` / `26` |
| `occurred_at` | ISO-8601 |
| `attempt_number` | `1` (`n+1` en reenvíos) |
| `retry_of_event_id` | vacío, o el `event_id` anterior |
| `source` | `lis-backend` |
| `payload` | JSON con `order`, `patient`, `insurance`, `physician`, `studies[].tests[]`, `samples[]`, `mappings` |

El stream va **sin el prefijo** que Laravel le pone a sus claves (conexión
`events` en `config/database.php`), así el nombre es el mismo de los dos lados.

## Configuración que tiene que coincidir

| lis-backend | lis-orchestrator | lis-adapter-labcore |
| --- | --- | --- |
| `DOMAIN_EVENTS_ORDER_STREAM` | `DOMAIN_EVENTS_ORDER_STREAM` | — |
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
- **UI, hecho**: *Instrumentos y conexiones → Integraciones* (tarjetas de
  orquestador, cola, adapters y envíos 24 h + tabla de eventos con detalle y
  reenvío), pestaña *Integraciones* en la ficha de la orden, y
  *Administración → Integraciones* (proveedores, políticas, valores fijos y
  grilla de equivalencias con import/export CSV). Falta que un `error` con
  *"Faltan equivalencias"* linkee a la grilla filtrada.
- **Alertas**: cuando haya un sistema de alertas, `error` debería generar una.
- **Sumar proveedores** (PACS, HIS, AMS): un bloque de config en el
  orchestrator y un adapter con el mismo contrato HTTP.
