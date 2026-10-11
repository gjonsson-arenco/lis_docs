# Healthbook: órdenes médicas sincronizadas con el LIS

Este brief es para implementar en `c:\Projects\LIS\lis-healthbook\lis-healthbook-back`
lo que falta para que las órdenes médicas (las recetas) queden iguales en
Healthbook y en el LIS, nazcan donde nazcan.

El lado del LIS (backend, adapter e infraestructura) **ya está hecho**. Este
documento es el contrato que Healthbook tiene que cumplir. El diseño completo
y las razones están en [medical-orders-sync.md](./medical-orders-sync.md):
**leerlo primero**, sobre todo "Quién es dueño de qué".

Antes de empezar, leer también:

- en el código de Healthbook: `App\Models\MedicalOrder`,
  `App\Models\MedicalOrderFile`, `App\Services\MedicalOrders\MedicalOrders`,
  `App\Services\Documents\UploadStorage` e `IntegrationOrderController`, que
  es el molde de la API de integración;
- en su README, la sección "API de integración".

## Contexto en dos líneas

- **No hay forma de que Healthbook le hable al LIS de CEBAC.** Corre
  on-premise, detrás de una VPN, sin dominio ni TLS. Es el LIS el que pregunta
  a Healthbook qué cambió, cada 15 segundos.
- **Los archivos no viajan por la API.** El LIS y Healthbook de CEBAC comparten
  un bucket de S3: cada uno escribe en su prefijo y copia del prefijo del otro.

## Qué hay que construir

1. **Identidad y origen** en `medical_orders`: `uuid` y `origin`.
2. **Un outbox de cambios** de órdenes médicas: `integration_outbox`, o el
   nombre que les guste.
3. **Tres endpoints** en la API de integración:

   | Endpoint | Para qué | Scope |
   | --- | --- | --- |
   | `GET /integration/medical-orders/changes` | El feed | `medical_orders.read` |
   | `GET /integration/medical-orders/{uuid}` | La foto vigente | `medical_orders.read` |
   | `PUT /integration/medical-orders/{uuid}` | Lo que manda el LIS | `medical_orders.write` |

4. **Storage en el bucket compartido**, con prefijo propio (`hb/`), y la copia
   de los archivos que manda el LIS.
5. Darle los dos scopes nuevos al cliente de integración del LIS
   (`lis-adapter-healthbook`).

Convenciones, las de siempre:

- comentarios en castellano;
- commits en inglés con conventional commits;
- README actualizado;
- tests de feature como los de `IntegrationOrderController`.

## 1. `uuid` y `origin`

- `medical_orders.uuid`: `char(36)`, único y no nulo. Las órdenes existentes se
  completan en la migración.
- `medical_orders.origin`: `string(20)`, default `healthbook`. Valores:
  - `healthbook`: la subió el paciente;
  - `bot`: la recibió un bot alojado en Healthbook;
  - `lis`: la creó el LIS por la integración.
- Las que sube el paciente generan su uuid al crearse (`Str::uuid()`).
- La API del portal sigue igual. Agregar `origin` a `MedicalOrderResource` sirve
  para que el portal pueda mostrar "cargada en el laboratorio".

Hacen falta además los campos de lo que hizo el laboratorio:

| Columna | Qué guarda |
| --- | --- |
| `lab_status` (`string(20)`, nullable) | `new`, `used` o `cancelled`, como en el LIS |
| `lab_number` (`string(40)`, nullable) | `OM-000123` |
| `order_number` (nullable) | La orden de laboratorio en la que se usó |
| `diagnosis` (nullable) | |
| `physician` (`json`, nullable) | `{external_id, name, license, specialty}` |
| `received_at` (nullable) | |
| `channel` (`string(20)`, default `portal`) | |

Los estudios van en `medical_order_studies`: `medical_order_id`, `code`, `name`
y `ordinal`.

Cómo los muestra el portal es decisión de Healthbook. Lo mínimo es "ya se usó
en la orden X" cuando `lab_status = used`.

## 2. El outbox

Cada cambio **hecho en Healthbook** a una orden médica escribe una fila en la
misma transacción que el cambio. Cuentan:

- el alta por el paciente o por el bot;
- un archivo agregado;
- notas o fecha editadas;
- el archivo de la orden;
- el borrado.

```
integration_outbox
  id (bigint autoincrement)   el `seq` del feed
  stream ('medical_orders')
  subject_uuid                el uuid de la orden
  event ('upserted' | 'deleted')
  occurred_at
```

Tres reglas:

- **Los cambios que entran por `PUT /integration/medical-orders/{uuid}` no van
  al outbox.** Son del LIS; si fueran, el LIS se los leería de vuelta. El LIS
  igual ignora los ecos, pero no hace falta generarlos.
- **El outbox no guarda la foto.** El feed la arma al responder, con la orden
  como esté en ese momento. Así una orden que cambió tres veces se lee
  completa y actualizada, y las URLs firmadas salen frescas.
- **Retención.** Basta con unos días: el LIS lee cada 15 segundos. Una tarea
  que borre lo de más de 30 días alcanza.

## 3. Endpoints

Todos van bajo `/api/v1/integration`, con `X-Internal-Token` y los middlewares
de siempre (`throttle:integration`, `integration.log`, `integration.auth:<scope>`).
Errores con el sobre `{ error: { code, message, details? } }`.

### `GET /medical-orders/changes?after={seq}&limit={n}`

Lo que cambió después de `after`, en orden de `seq`. Sin `after`, desde el
principio. `limit` va de 1 a 200; el default es 50.

```json
{
  "data": [
    {
      "seq": 101,
      "event": "upserted",
      "occurred_at": "2026-10-10T12:00:00-03:00",
      "medical_order": { "...": "la foto, ver abajo" }
    }
  ],
  "next_after": 101,
  "has_more": false
}
```

- **Un evento por orden.** Si la misma orden tiene varias filas en la página,
  basta con una: la foto es la vigente. `next_after` es el último `seq` cubierto.
- **Un `deleted`** lleva la última foto conocida (con `withTrashed`), sin
  archivos.
- **Huecos.** Un autoincremental puede confirmarse fuera de orden: la fila 102
  se commitea antes que la 101. Para no saltear la 101 conviene una de dos:
  - devolver sólo filas con `occurred_at` de hace más de 2 segundos;
  - cortar la página en el primer hueco que tenga menos de unos segundos.

  Cualquiera sirve. Lo que no puede pasar es que `next_after` pase por encima
  de una fila que todavía no se veía.

**La foto** es la misma forma en el feed y en `GET /medical-orders/{uuid}`:

```json
{
  "id": 77,
  "uuid": "6b1f9a52-3c1e-4e5a-9d64-0c2f8e1a7b01",
  "origin": "healthbook",
  "channel": "portal",
  "status": "active",
  "notes": "Me la dio el clínico",
  "issued_at": "2026-10-08",
  "created_at": "2026-10-10T12:00:00-03:00",
  "updated_at": "2026-10-10T12:05:00-03:00",
  "patient": {
    "id": 4411,
    "external_id": "123",
    "first_name": "Ana", "last_name": "Pérez", "birth_date": "1979-02-03", "gender": "female",
    "identifications": [{ "type": "DNI", "number": "30463017" }]
  },
  "files": [
    {
      "id": 9001,
      "original_name": "receta.jpg",
      "mime_type": "image/jpeg",
      "size_bytes": 482113,
      "ordinal": 1,
      "url": "https://<bucket>.s3.sa-east-1.amazonaws.com/hb/medical-orders/…?X-Amz-…",
      "s3": { "bucket": "<bucket>", "key": "hb/medical-orders/0b6c….jpg" }
    }
  ]
}
```

- **`patient.external_id`** es el id del paciente **en el LIS**: el
  `origin_number` cuando el `provider` del paciente es el del cliente de
  integración (`arenco`). Si la ficha la cargó otro proveedor, va `null`; el LIS
  cruza entonces por documento y nacimiento.
- **`identifications`**: el tipo con el código que conoce el LIS (`DNI`, `CI`,
  `PAS`), como llega en las órdenes.
- **`url`**: firmada, de 15 minutos. El LIS la usa sólo si no puede copiar
  dentro de S3.
- **`s3`**: la key **completa**, con el prefijo.
- **`origin` y `channel`**: los que tenga la orden. Para una del bot,
  `origin: "bot"` y `channel: "whatsapp"`.

### `GET /medical-orders/{uuid}`

`200 { "medical_order": {…la foto…} }`, o `404` si no existe (incluidas las
borradas). El LIS lo usa:

- para reintentar una importación;
- cuando admisión asigna a mano el paciente de una orden que quedó pendiente,
  quizás horas después, con las URLs ya vencidas.

### `PUT /medical-orders/{uuid}`

La orden médica del LIS, entera. Es idempotente.

```json
{
  "origin": "lis",
  "number": "OM-000123",
  "channel": "mostrador",
  "status": "used",
  "received_at": "2026-10-10T12:00:00+00:00",
  "issued_at": "2026-10-08",
  "notes": "Traer en ayunas",
  "diagnosis": "Control",
  "physician": { "external_id": 104, "name": "GOMEZ MUÑOZ GUSTAVO JAVIER", "license": "4227", "specialty": null },
  "order_number": "20260031",
  "patient": { "...": "la misma foto que en PUT /orders/{number}, con guardians" },
  "studies": [{ "code": "660475", "name": "HEMATOCRITO", "ordinal": 1 }],
  "files": [
    {
      "external_id": "55",
      "original_name": "receta.jpg",
      "mime_type": "image/jpeg",
      "ordinal": 1,
      "s3": { "bucket": "<bucket>", "key": "lis/documents/1f0c….jpg" }
    }
  ]
}
```

| Campo | Notas |
| --- | --- |
| `status` | El del LIS (`new`, `used`, `cancelled`). Va a `lab_status`. |
| `physician.external_id` | `physicians.lis_id`, como en las órdenes. Puede venir `null` con el nombre del papel. |
| `studies` | Lista completa: reemplaza. |
| `files` | Sólo los que el LIS tiene en S3. `external_id` es el id del documento en el LIS. |

Qué hace Healthbook según el uuid:

| Caso | Qué hacer |
| --- | --- |
| **uuid nuevo** | Crear la orden con `origin = lis`. Resolver el paciente igual que en `PUT /orders/{number}`: proveedor + `external_id`, después documento y nacimiento, si no crear; mismos `409 AmbiguousPatient`. Copiar cada archivo de `s3` a `hb/medical-orders/<uuid>.<ext>` con `CopyObject`, guardando el `external_id` para no copiarlo dos veces. `201`. |
| **uuid existente con `origin = lis`** | La receta también es del LIS: actualizar todo. Archivos: agregar los `external_id` nuevos y borrar los que ya no vienen. `200`. |
| **uuid existente con `origin` ≠ `lis`** (la subió el paciente o el bot) | Actualizar **sólo** lo del laboratorio: `lab_status`, `lab_number`, `order_number`, `diagnosis`, `physician`, `received_at` y estudios. **No tocar** archivos, notas, fecha de emisión, `status` ni paciente: son del paciente. `200`. |

Ninguno de los tres casos escribe en el outbox.

Respuesta:

```json
{
  "medical_order": { "id": 77, "uuid": "…", "origin": "lis" },
  "created": true,
  "patient": { "id": 4411, "matched_by": "external_id" },
  "files": { "created": 1, "removed": 0, "skipped": 0 }
}
```

- **`skipped`** cuenta los archivos que no se pudieron copiar: key inexistente
  o sin permiso. No es un error. Loguearlo y seguir, porque la orden igual
  tiene que quedar.
- **Errores**:
  - `422 ValidationFailed` con `details` por campo;
  - `409 AmbiguousPatient`;
  - `401` y `403` como siempre.

  El adapter los traduce igual que en las órdenes.

## 4. Storage en el bucket compartido

En CEBAC, `DOCUMENTS_DISK=s3` y el disco S3 con `root` igual a `hb`. Si el
`config/filesystems.php` de Healthbook todavía no lo lee, agregar
`'root' => env('AWS_ROOT', '')`. Con eso:

- **Sin cambios de código.** `UploadStorage` sigue escribiendo
  `medical-orders/<uuid>.<ext>` y queda en `hb/medical-orders/…`.
- **La foto lleva la key completa**: el `root` más el `path`.
- **Copia desde el LIS.** Para copiar se usa el cliente de S3 del disco:
  `Storage::disk('s3')->getClient()->copyObject([...])`, con `Bucket` el propio,
  `Key` igual a `hb/medical-orders/…` y `CopySource` igual a
  `<bucket>/<key del LIS>`.

El rol de Healthbook necesita lectura sobre `lis/*`. Bucket, permisos y KMS
están en [../infra/cebac-s3-documents.md](../infra/cebac-s3-documents.md).

## 5. Cómo se prueba contra el LIS

- **Lo que espera el adapter.** Su README tiene la sección "Órdenes médicas":
  qué manda en el PUT y qué espera del feed. Los tipos están en
  `src/domain/healthbook-medical-order.ts`, en
  `lis-adapters/lis-adapter-healthbook`.
- **El LIS contra un Healthbook local:**
  1. el adapter en `HEALTHBOOK_MODE=http` apuntando al Healthbook local;
  2. en el backend, `php artisan integrations:pull-medical-orders --force -v`.
- **Del LIS a Healthbook.** Activar las reglas
  `send-medical-order-to-healthbook` y `update-medical-order-in-healthbook`
  (Admin > Integraciones) y editar una orden médica del LIS.
- **Archivos.** Sin S3 los archivos no viajan del LIS a Healthbook. En sentido
  contrario sí: el LIS baja por `url`.
