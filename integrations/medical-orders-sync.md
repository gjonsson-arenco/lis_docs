# Órdenes médicas entre el LIS y Healthbook

Cómo se mantienen iguales, en los dos sistemas, las órdenes médicas (la receta
tal como llega, antes de la admisión). Pueden nacer en el LIS (mostrador, mail,
admisión), en Healthbook (el paciente la sube al portal) o en un bot.

Partes del documento:

- **Diseño**: las reglas.
- **Qué está hecho**: lo del lado del LIS.
- **Qué falta**: Healthbook, el front y la puesta en marcha en CEBAC.

Documentos relacionados:

- Lo que Healthbook tiene que construir: [healthbook-medical-orders-brief.md](./healthbook-medical-orders-brief.md).
- El bucket compartido en CEBAC: [../infra/cebac-s3-documents.md](../infra/cebac-s3-documents.md).
- El circuito de eventos sobre el que se apoya el sentido LIS → Healthbook: [event-driven-integrations.md](./event-driven-integrations.md).

## Restricción de partida: el LIS es on-premise

En CEBAC el LIS corre en un servidor propio al que sólo se llega por VPN (sin
dominio público ni TLS). Healthbook corre en la nube. Por eso:

- **Desde el LIS hacia la nube** hay salida HTTPS. Es lo que ya usan el adapter
  de Healthbook, Textract y Bedrock.
- **Desde la nube hacia el LIS** no hay nada, y no se abre: son datos de salud y
  el servidor no tiene TLS.

La regla es que **la conexión siempre la abre el on-premise**. Así queda cada
sentido:

| Sentido | Mecanismo | Latencia |
| --- | --- | --- |
| LIS → Healthbook | Evento de dominio (`medical_order.created` / `.updated`) → Redis Stream → orchestrator → adapter → `PUT /integration/medical-orders/{uuid}`. Es el mismo circuito que las órdenes de laboratorio. | Segundos |
| Healthbook → LIS | El LIS **lee un feed** de cambios de Healthbook cada 15 s, a través del adapter. Healthbook escribe el feed como outbox, en la misma transacción que el cambio. | ≤ 15 s |

Elegimos el feed (*pull*) y no SQS por tres razones:

- **No suma infraestructura** ni credenciales de AWS en el servidor.
- **Se puede reprocesar.** Si el LIS estuvo caído o hubo un bug, se rebobina el
  cursor. Un mensaje de SQS, una vez consumido, se pierde.
- **Admite SQS más adelante.** Si hiciera falta latencia de "al instante",
  bastaría un aviso por SQS que despierte al pull, sin cambiar el modelo: el
  feed sigue siendo la fuente.

## Identidad: el uuid

Cada orden médica tiene un **uuid que pone el sistema donde nace**, y los dos
lados hacen *upsert* por él. Con eso un eco termina en un upsert sin cambios,
no en un duplicado. Un eco es una orden del LIS que Healthbook vuelve a mostrar
en su feed, o una del portal que el LIS le reenvía con su estado.

- **LIS**: `medical_orders.uuid`. Lo saca el modelo al crearse; las órdenes
  existentes se completaron en la migración. `medical_orders.origin` vale
  `lis`, `healthbook` o `bot`.
- **Healthbook**: la misma columna `uuid`, más `origin`. Ver el brief.

## Quién es dueño de qué

No todos los campos se editan en los dos lados. Si se editaran, habría que
resolver conflictos. En cambio, cada parte de la orden tiene un solo dueño:

| Parte | Dueño | Qué hace el otro lado |
| --- | --- | --- |
| **La receta tal como llegó**: archivos, fecha de emisión, notas del paciente | El sistema de origen | La guarda como réplica. Si la corrige localmente (por ejemplo, la fecha que leyó mal el OCR), la corrección se respeta hasta que el origen vuelva a cambiar *ese* campo. |
| **Lo que hizo el laboratorio**: estudios revisados, médico del catálogo, diagnóstico, estado `used` / `cancelled`, orden de laboratorio en la que se usó | El LIS | Healthbook lo muestra (por ejemplo, "tu orden ya se usó en la admisión 20260031"). |
| **El paciente** | El LIS es maestro | Healthbook lo reconoce como en las órdenes de laboratorio. |

Consecuencias concretas:

- Si el paciente **archiva o borra** la orden en el portal, el LIS la **cancela
  sólo si todavía está `new`**. Una que ya se usó en una admisión no cambia.
- Si el LIS **cancela o usa** una orden que vino del portal, Healthbook recibe
  el estado y no toca los archivos.
- Las **notas del paciente** que se editan en el portal pisan las del LIS sólo
  si cambiaron en el portal. El LIS compara contra la última foto que aplicó
  (`medical_order_imports.applied_payload`).

### El bot

El bot no es un tercer sistema que sincronizar. Escribe en el sistema que lo
aloja, con `origin = bot` y `channel = whatsapp`, y de ahí sigue el flujo de
ese sistema:

- Si vive en la nube (Healthbook), entra por el feed.
- Si vive on-premise (lis-messaging-service), se crea en el LIS y sale por los
  eventos.

Sumar un tercer participante multiplicaría los casos de conflicto sin ganar
nada.

## Archivos: S3 compartido, no viajan

Con los documentos del LIS en S3, en el bucket de CEBAC que comparte con
Healthbook, **los archivos no pasan por la integración**. Viaja
`{bucket, key}` y el otro lado los copia dentro de AWS con `CopyObject`, sin
pasar por el servidor de CEBAC.

- **Prefijos.** Cada sistema escribe en el suyo (`lis/`, `hb/`) y lee el del
  otro. La configuración del LIS es `AWS_ROOT=lis`.
- **LIS → Healthbook.** `documents[].storage` lleva el bucket y la key. Con
  `DOCUMENTS_DISK=local`, `storage` va en `null` y ese archivo no viaja (el
  adapter lo loguea). **Para que viajen archivos, el LIS tiene que estar en
  S3.**
- **Healthbook → LIS.** Cada archivo trae `s3` y además una `url` firmada de
  vida corta. El LIS primero intenta copiar dentro de S3; si no puede (otro
  bucket, sin permiso de lectura o disco local), lo baja por la URL. Cada
  archivo se trae **una sola vez**, por su id en Healthbook
  (`medical_order_imports.files_map`): en Healthbook una página no se edita, se
  sube otra.

## Healthbook → LIS en detalle

1. El contenedor `scheduler` corre `integrations:pull-medical-orders` cada 15 s
   (`MEDICAL_ORDER_PULL_ENABLED`, `config/medical_order_sync.php`).
2. Pide al adapter `GET /api/v1/medical-orders/changes?after={cursor}`. El
   adapter traduce el feed de Healthbook a la forma del LIS (ver su README).
3. Por cada evento, `MedicalOrderImportService` hace upsert en
   `medical_order_imports` por uuid y aplica:
   - **Eco** (la orden es del LIS): queda `ignored` y no se toca nada.
   - **Paciente.** Se busca en este orden:
     1. el id del LIS que Healthbook conoce (`lis_patient_id`);
     2. el enlace que el LIS ya tiene con el paciente de Healthbook
        (`patient_external_references`);
     3. **un único** paciente con ese documento **y** esa fecha de nacimiento.

     Si nada de eso alcanza, queda `pending_patient`: no se adivina.
   - **Alta.** Si es nueva, se crea la orden médica con número `OM-…`, la
     receta y los archivos, y se enlaza el paciente con su id en Healthbook.
     Si ya existe, se aplica sólo lo que cambió en el origen.
   - **Sin rebote.** Nada de esto dispara `medical_order.*`: lo que viene de
     Healthbook no se reenvía a Healthbook.
4. El cursor avanza **después de procesar cada página**. Si algo se corta, la
   página se relee, y como todo es upsert, releer no duplica.
5. **Reintentos.** Lo que quedó `pending_patient` o `failed` se reintenta solo
   cada 10 minutos, hasta 50 veces. Antes de cada reintento se vuelve a pedir
   la foto (`GET /medical-orders/{uuid}`), porque las URLs firmadas vencen.
6. **La bandeja.** Admisión ve lo pendiente en `GET /api/v1/medical-order-imports`
   y lo resuelve de dos formas:
   - `POST .../{id}/assign-patient`: vuelve a pedir la foto, importa y deja el
     paciente enlazado para las siguientes.
   - `POST .../{id}/ignore`: la descarta.

Estados de `medical_order_imports.status`:

| Estado | Qué significa |
| --- | --- |
| `imported` | Es una orden médica del LIS (`medical_order_id`). |
| `pending_patient` | No se sabe de qué paciente es. Está en la bandeja. |
| `failed` | Error al importar (por ejemplo, no se pudo traer un archivo). Se reintenta solo. El motivo está en `error`. |
| `ignored` | Eco de una orden del LIS, borrada en Healthbook antes de importarse, o descartada a mano. |

## LIS → Healthbook en detalle

Los eventos los dispara `MedicalOrderService` después del commit (listeners
`ShouldHandleEventsAfterCommit`):

| Evento | Cuándo | `event_type` |
| --- | --- | --- |
| `medical_order.created` | Alta en el LIS, sea la carga manual o la que crea la admisión con la foto de la receta. | `send-medical-order` |
| `medical_order.updated` | Edición, cancelación, estudios revisados o aplicados del análisis, páginas agregadas o quitadas, uso en una admisión. | `send-medical-order` |

- **Payload.** Es `MedicalOrderIntegrationPayloadBuilder`: la orden médica, el
  paciente canónico (el mismo que en las órdenes de laboratorio), el médico,
  los estudios, los documentos con su `storage` y `mappings`. Las entidades
  son `study_type`, `physician` e `identification_type`.
- **Transporte.** Viaja por el **mismo stream** que las órdenes de laboratorio
  (`lis:events:order`). El orchestrator rutea por nombre de evento y no cambió.
  En `integration_logs` el sujeto es `medical_order`.
- **Reglas.** No tienen filtro de origen ni de servicio: la orden médica no los
  tiene. En el seed de CEBAC quedan **inactivas** hasta que Healthbook tenga el
  endpoint (Admin > Integraciones).

## Qué está hecho (2026-10-10)

**lis-backend**

- Migración `2026_10_10_130000_add_healthbook_sync_to_medical_orders`:
  `medical_orders.uuid` y `.origin`, más las tablas `medical_order_imports` e
  `integration_inbound_cursors`.
- Eventos `medical_order.created` / `.updated` con su payload builder, y reglas
  en los seeders.
- `MedicalOrderFeedClient`, `MedicalOrderImportService` y
  `MedicalOrderFeedPuller`, más el comando `integrations:pull-medical-orders`
  programado cada 15 s.
- La bandeja: `GET /api/v1/medical-order-imports`, `.../assign-patient` y
  `.../ignore`.
- `DocumentObjectStore`, que da la ubicación en S3 y trae archivos de otro
  sistema; `AWS_ROOT` en el disco S3. Textract ahora respeta el prefijo.
- `documents:copy-to-disk`, para pasar los documentos de local a S3.
- El listado de órdenes médicas filtra por `origin` y devuelve `origin` y
  `uuid`.

**lis-adapter-healthbook**

- `send-medical-order` por el mismo `POST /api/v1/orders`.
- `GET /api/v1/medical-orders/changes` y `GET /api/v1/medical-orders/{uuid}`.
- El stub tiene una orden del portal en el feed y acepta el upsert.

**lis-infra (deploy/cebac)**

- Contenedor `scheduler` (`schedule:work`) y `MEDICAL_ORDER_PULL_ENABLED`
  (apagado).

**Probado**

- Tests unitarios del backend y jest del adapter (76).
- Smoke contra la base de dev con el adapter en stub, en los dos sentidos:
  - importación;
  - eco ignorado;
  - paciente pendiente y asignado a mano;
  - cambio en el portal sin pisar lo corregido en el LIS;
  - archivado → cancelada;
  - el payload hacia Healthbook.

## Qué falta

- **Healthbook**: todo el brief. Hasta entonces, contra el Healthbook real el
  adapter da 404 y sólo funciona el stub.
- **Front del LIS**:
  - la bandeja de "órdenes del portal sin paciente" (listar, buscar el
    paciente, asignar, descartar);
  - una marca de origen en el listado y en el detalle de órdenes médicas;
  - un filtro por origen.
- **Análisis de lo que llega del portal.** La orden importada no corre el
  OCR/IA sola: hay que analizarla desde el detalle, como una que se sube a
  mano. Si se quiere automático, va después de `importFiles`.
- **CEBAC**:
  - bucket, IAM y pasaje a S3 (ver el runbook);
  - activar las dos reglas y `MEDICAL_ORDER_PULL_ENABLED` cuando Healthbook
    esté.
- **A validar con legales.** Los documentos clínicos de CEBAC pasan a un bucket
  en `sa-east-1` (São Paulo): es transferencia internacional de datos de salud
  (Ley 25.326).
