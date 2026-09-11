# Integraciones event-driven: orchestrator y adapters

Cómo una orden creada en el LIS llega a un proveedor externo (hoy Labcore) y
cómo el usuario ve en qué quedó. Este documento es el modelo tal como quedó
implementado; los detalles de cada pieza están en el README de su repo.

| Pieza | Repo | Rol |
| --- | --- | --- |
| Backend | `lis-backend` | Crea la orden, **emite** `order.created`, **guarda la trazabilidad** que le reportan, la expone a la UI. |
| Orchestrator | `lis-orchestrator` | Lee los eventos, aplica las reglas, llama al adapter con reintentos, reporta cada paso al backend. |
| Adapter | `lis-adapters/lis-adapter-labcore` | Traduce la orden canónica al formato del proveedor y llama a su API. Stateless. |
| Labcore API | `C:\Projects\Customs\labcore api` (.NET, propia) | `POST /api/v1/orders` con `X-Api-Key`: alta idempotente por número de orden sobre la base del LIS Labcore. |

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
| `POST /api/v1/internal/integrations/events/pending` | `X-Internal-Token` (`lis_orchestrator`) | Crea/actualiza la fila en `pending_send`. |
| `POST /api/v1/internal/integrations/events/sent` | ídem | Marca `sent`. |
| `POST /api/v1/internal/integrations/events/ack` | ídem | Marca `received`. |
| `POST /api/v1/internal/integrations/events/error` | ídem | Marca `error`. |
| `GET /api/v1/orders/{order}/integrations` | usuario | La trazabilidad de una orden, con payloads. |
| `POST /api/v1/orders/{order}/integrations/{provider}/retry` | `integrations.manage` | Reenvía. Responde `202` con el `event_id` nuevo; `503` si el bus no está. |
| `GET /api/v1/integrations/logs` | `integrations.view` | Listado global, filtros `status`, `provider`, `order_id`, `external_id`, `from`, `to`. Sin payloads. |

### lis-orchestrator

`GET /health`, `GET /ready`, `GET /api/v1/rules` (reglas y proveedores cargados).

### lis-adapter-labcore

`GET /health`, `GET /ready`, `POST /api/v1/orders` (`X-Internal-Token`).

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
| — | `LABCORE_ADAPTER_INTERNAL_TOKEN` | `INTERNAL_TOKEN` |

En producción todo sale del `.env` de `lis-infra`
(`LIS_ORCHESTRATOR_INTERNAL_TOKEN`, `LIS_ADAPTER_LABCORE_INTERNAL_TOKEN`,
`LABCORE_*`). El backend necesita además `REDIS_CLIENT=predis` y
`DOMAIN_EVENTS_ENABLED=true`.

## Pendiente

- **Probar contra una Labcore API real.** El cliente está hecho contra el
  contrato (`CreateOrderRequest`) y probado contra una API de juguete; falta
  correrlo contra una instancia con base Labcore y confirmar que los códigos
  de catálogo (estudios, tipos de muestra, centros, servicios, coberturas,
  tipos de documento) coinciden. Si alguno no, se apaga con `LABCORE_SEND_*`
  en el adapter.
- **Resultados de vuelta** (`send-result` / Labcore → LIS): no está diseñado.
- **UI**: la línea de tiempo en la orden, el listado de integraciones con el
  botón de reenvío y el workspace Admin > Integraciones (proveedor → entidad →
  grilla con filtro "sin mapear", copiar códigos, import CSV) consumen los
  endpoints de arriba; no están hechos. Un `error` con *"Faltan
  equivalencias"* debería linkear a la grilla filtrada.
- **Alertas**: cuando haya un sistema de alertas, `error` debería generar una.
- **Sumar proveedores** (PACS, HIS, AMS): un bloque de config en el
  orchestrator y un adapter con el mismo contrato HTTP.
