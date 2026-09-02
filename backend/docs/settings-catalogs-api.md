# Settings Catalogs API (Front Reference)

Base path: `/api/v1/admin/settings`

Auth/guards:
- Middleware: `throttle:auth`, `tenant`, `auth.cognito:access`
- Permission: `can:users.manage`

## 1) Catalog discovery

### GET `/api/v1/admin/settings/catalogs`
Devuelve los catálogos disponibles con su `key`, `model` y `path`.

## 2) Generic CRUD for catalogs

Para cualquier `{catalog}`:

- GET `/api/v1/admin/settings/catalogs/{catalog}`
- POST `/api/v1/admin/settings/catalogs/{catalog}`
- GET `/api/v1/admin/settings/catalogs/{catalog}/{id}`
- PATCH `/api/v1/admin/settings/catalogs/{catalog}/{id}`
- DELETE `/api/v1/admin/settings/catalogs/{catalog}/{id}`

### Query params (index)
- `search` string
- `is_active` boolean
- `per_page` integer (1..100)
- `order_by` string (`id|code|name|created_at|updated_at|sort_order`)
- `order_dir` string (`asc|desc`)
- `with_trashed` boolean
- `only_trashed` boolean
- `include[]` string (relaciones permitidas por catálogo)

### Query params (show)
- `with_trashed` boolean
- `include[]` string

## 3) Supported catalogs (`{catalog}`)

- `working-areas`
- `study-types`
- `method-types`
- `sample-types`
- `container-types`
- `sites`
- `origins`
- `order-types`
- `identification-types`
- `insurance-types`
- `insurance-plans`
- `insurances`
- `practice-types`
- `services`
- `physicians`
- `test-types`
- `reporting-types`
- `reference-values`
- `stations`

## 4) Includes allowed by catalog

- `study-types`: `workingArea`, `testTypes`, `practiceTypes`
- `sample-types`: `testTypes`, `containerTypes`
- `origins`: `site`
- `insurance-types`: `plans`, `insurances`
- `insurance-plans`: `insuranceType`
- `insurances`: `insuranceType`, `insurancePlan`
- `services`: `site`, `physicians`
- `physicians`: `site`, `service`
- `test-types`: `sampleType`, `workingArea`, `studyTypes`, `reportingTypes`
- `reporting-types`: `testType`, `methodType`, `referenceValues`
- `reference-values`: `reportingTypes`

## 5) Special relation endpoints

### StudyType <-> TestType
- POST `/api/v1/admin/settings/study-types/{studyType}/test-types`
- DELETE `/api/v1/admin/settings/study-types/{studyType}/test-types/{testType}`

POST body:
```json
{
  "test_type_id": 1,
  "sort_order": 10,
  "is_required": true,
  "is_internal": false
}
```

### StudyType <-> PracticeType
- POST `/api/v1/admin/settings/study-types/{studyType}/practice-types`
- DELETE `/api/v1/admin/settings/study-types/{studyType}/practice-types/{practiceType}`

POST body:
```json
{
  "practice_type_id": 1,
  "practice_limit": 1
}
```

### SampleType <-> ContainerType
- POST `/api/v1/admin/settings/sample-types/{sampleType}/container-types`
- DELETE `/api/v1/admin/settings/sample-types/{sampleType}/container-types/{containerType}`

POST body:
```json
{
  "container_type_id": 1,
  "is_default": true
}
```

### ReportingType <-> ReferenceValue
- POST `/api/v1/admin/settings/reporting-types/{reportingType}/reference-values`
- DELETE `/api/v1/admin/settings/reporting-types/{reportingType}/reference-values/{referenceValue}`

POST body:
```json
{
  "reference_value_id": 1
}
```

## 6) Entity payload reference (POST/PATCH)

Nota: en `PATCH` todos los campos son opcionales (`sometimes`), en `POST` los marcados como required son obligatorios.

### working-areas
- `code` (required, unique)
- `name` (required)
- `sort_order` (optional)
- `is_active` (optional)

### study-types
- `code` (required, unique)
- `class` (optional, `study` | `routine`, default `study`)
- `version` (optional, default 1)
- `name` (required)
- `description` (optional)
- `preparation_time` (optional, json/array)
- `component_study_type_ids` (optional, array de ids; los estudios que componen la rutina, en orden. Reemplaza el pivot completo)
- `triggered_study_type_ids` (optional, array de ids; los estudios que este arrastra. Reemplaza el pivot completo)
- `sort_order` (optional)
- `working_area_id` (required para `class=study`; para una rutina lo resuelve el backend)
- `is_active` (optional)
- `is_visible` (optional, default true, solo `class=study`)

`is_active` y `is_visible` son dos ejes distintos. Un estudio inactivo no entra
en una orden por ningun camino: ni buscado, ni como componente de una rutina, ni
disparado por otro estudio. Uno no visible si entra, pero solo tirado por una
rutina o por un disparo: buscarlo y cargarlo suelto no se puede.

Filtro de listado: `?class=routine` devuelve solo rutinas, `?class=study` solo
estudios.

Una rutina no se procesa ni se informa: al guardarla con `class=routine` el
backend limpia metodo, preparacion, dias de proceso, muestras, requerimientos y
disparos, porque de ella solo interesan codigo, nombre, activo y su
composicion. Una rutina nunca llega a ser un `Study` de una orden: se reemplaza
por los estudios que la componen, cada uno con `source=routine` y
`source_study_type_id` apuntando a la rutina.

### method-types
- `code` (required, unique)
- `name` (required)
- `description` (optional)
- `is_active` (optional)

### sample-types
- `code` (required, unique)
- `name` (required)
- `description` (optional)
- `patient_collected` (optional)
- `collection_info` (optional)
- `sample_function` (optional)
- `processing_minutes` (optional)
- `is_active` (optional)

### container-types
- `code` (required, unique)
- `name` (required)
- `style` (optional, json/array)
- `volume` (optional)
- `is_active` (optional)

### sites
- `code` (required, unique)
- `name` (required)
- `description` (optional)
- `is_external` (optional)
- `is_active` (optional)

### origins
- `code` (required, unique)
- `name` (required)
- `description` (optional)
- `site_id` (optional, nullable)
- `is_active` (optional)

### order-types
- `code` (required, unique)
- `name` (required)
- `is_active` (optional)

### identification-types
- `code` (required, unique)
- `name` (required)
- `is_active` (optional)

### insurance-types
- `code` (required, unique)
- `name` (required)
- `description` (optional)
- `is_active` (optional)

### insurance-plans
- `insurance_type_id` (required)
- `code` (required)
- `name` (required)
- `description` (optional)
- `is_active` (optional)

### insurances
- `insurance_type_id` (required)
- `insurance_plan_id` (optional, nullable)
- `number` (required)
- `valid_through` (optional, date)
- `is_active` (optional)

### practice-types
- `code` (required, unique)
- `name` (required)
- `is_active` (optional)

### services
- `site_id` (optional, nullable)
- `code` (required, unique)
- `name` (required)
- `description` (optional)
- `is_active` (optional)

### physicians
- `site_id` (optional, nullable)
- `service_id` (optional, nullable)
- `state_number` (optional, nullable)
- `national_number` (optional, nullable)
- `name` (required)
- `is_active` (optional)

### test-types
- `code` (required, unique)
- `version` (optional, default 1)
- `name` (required)
- `description` (optional)
- `result_class` (optional: `text|number|calc|list|multi|isolate`)
- `options` (optional, json/array)
- `sample_type_id` (optional, nullable)
- `working_area_id` (optional, nullable)
- `min_volume_required` (optional)
- `is_active` (optional)

### reporting-types
- `test_type_id` (required)
- `method_type_id` (required)
- `unit_of_measure` (required)
- `is_default` (optional)
- `is_active` (optional)

### reference-values
- `gender` (optional: `male|female|other`)
- `min_age` (optional)
- `min_age_unit` (optional: `d|w|m|y`)
- `max_age` (optional)
- `max_age_unit` (optional: `d|w|m|y`)
- `min_reference` (optional)
- `max_reference` (optional)
- `min_alert` (optional)
- `max_alert` (optional)
- `min_repeat` (optional)
- `max_repeat` (optional)
- `valid_from` (optional, date)
- `valid_through` (optional, date)
- `note` (optional)
- `normal_options` (optional, json/array)
- `abnormal_options` (optional, json/array)
- `alert_options` (optional, json/array)
- `repeat_options` (optional, json/array)
- `is_active` (optional)

## 7) Response shape

### Index
```json
{
  "data": [],
  "meta": {
    "current_page": 1,
    "per_page": 20,
    "last_page": 1,
    "total": 0
  }
}
```

### Show / Store / Update
```json
{
  "data": {}
}
```

### Delete
- HTTP `204 No Content`

## Stations API

### Endpoints
```
GET    /api/stations
POST   /api/stations
GET    /api/stations/{id}
PATCH  /api/stations/{id}
DELETE /api/stations/{id}
```

### Payloads
#### Crear/Actualizar Station
```json
{
    "name": "Nombre de la estación",
    "description": "Descripción opcional",
    "site_id": 1,
    "type": "admin", // Valores posibles: admin, technique, sample-management, sample-collection
    "parameters": { "key": "value" } // Opcional
}
```

## Servicios por origen y tipo de paciente

Un servicio es una sala o sector de una institución concreta ("UTI Adultos",
"2do Piso", "Guardia"), así que no aplica a cualquier admisión: depende de
dónde está el paciente (origen) y de cómo se lo atiende (tipo de paciente /
`order_types`). La tabla `service_availabilities` guarda las combinaciones
permitidas.

**Sin configuración no hay servicio.** Un par (origen, tipo de paciente) sin
filas no ofrece ningún servicio: admisión deja el campo desactivado y la orden
se graba con `service_id` en `null`. No existe un fallback permisivo.

### Consulta desde admisión
```
GET /api/v1/services/available?origin_id={id}&order_type_id={id}
```
```json
{
  "data": [
    { "id": 5, "code": "UTI", "name": "UTI Adultos" }
  ]
}
```
Una lista vacía es una respuesta válida y significa "esta combinación no lleva
servicio".

### Configuración (ABM)
```
GET /api/v1/admin/service-availability
PUT /api/v1/admin/service-availability
```
`GET` devuelve `origins`, `order_types`, `services` y `matrix` — esta última
sólo con las celdas que ofrecen al menos un servicio.

`PUT` reemplaza el conjunto completo de una celda; un `service_ids` vacío la
vacía:
```json
{ "origin_id": 2, "order_type_id": 1, "service_ids": [5, 12] }
```

### Validación de órdenes
`POST /api/v1/orders` y `PATCH /api/v1/orders/{order}` rechazan un
`service_id` que no esté disponible para el par efectivo de la orden. En el
`PATCH` el par se resuelve contra la orden ya guardada, de modo que cambiar
sólo el origen puede invalidar el servicio que la orden ya tenía.

### Carga inicial
`CebacServiceAvailabilitySeeder` propone un punto de partida clasificando los
servicios del legacy por tipo (internación, guardia, ambulatorio, domicilio,
derivación, hemoterapia, neuma) y ofreciéndolos en los orígenes que tienen
sede propia. El legacy nunca registró esta relación: la propuesta se corrige
desde el ABM.
