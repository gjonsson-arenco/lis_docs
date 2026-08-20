# API - Pacientes y Ordenes (Front)

Base URL: `http://localhost:8000/api`

Headers requeridos:
- `Authorization: Bearer <cognito_access_token>`
- `X-Tenant-ID: <tenant_id>`
- `Content-Type: application/json`

Middleware de estas rutas:
- `throttle:auth`
- `tenant`
- `auth.cognito:access`

## 1) Alta de paciente

- Metodo: `POST`
- Endpoint: `/api/v1/patients`

### Request body

```json
{
  "number": "P-000123",
  "last_name": "Perez",
  "first_name": "Ana",
  "middle_name": "Maria",
  "birth_date": "1990-04-21",
  "gender": "female",
  "email": "ana.perez@mail.com",
  "phone": "1144556677",
  "mobile": "11999888777",
  "address": "Calle 123",
  "identifications": [
    {
      "identification_type_id": 1,
      "number": "30111222",
      "is_primary": true,
      "issued_country": "AR",
      "valid_through": null
    }
  ],
  "insurances": [
    {
      "insurance_type_id": 1,
      "insurance_plan_id": 2,
      "affiliate_number": "SOC-778899",
      "holder_name": "Ana Perez",
      "valid_through": "2027-12-31",
      "is_primary": true,
      "is_active": true
    }
  ]
}
```

### Campos y validaciones

- `number`: opcional, `string`, max 40, unico.
- `last_name`: requerido, `string`, max 120.
- `first_name`: requerido, `string`, max 120.
- `middle_name`: opcional, `string`, max 120.
- `birth_date`: opcional, `date`.
- `gender`: opcional, `male | female | other`.
- `email`: opcional, `email`, max 150.
- `phone`: opcional, `string`, max 40.
- `mobile`: opcional, `string`, max 40.
- `address`: opcional, `string`, max 255.
- `identifications[]`: opcional.
- `identifications[].identification_type_id`: requerido si existe item, `exists:identification_types,id`.
- `identifications[].number`: requerido si existe item, `string`, max 40.
- `identifications[].is_primary`: opcional, `boolean`.
- `identifications[].issued_country`: opcional, `string`, size 2.
- `identifications[].valid_through`: opcional, `date`.
- `insurances[]`: opcional.
- `insurances[].insurance_type_id`: requerido si existe item, `exists:insurance_types,id`.
- `insurances[].insurance_plan_id`: opcional, `exists:insurance_plans,id`.
- `insurances[].affiliate_number`: opcional, `string`, max 50.
- `insurances[].holder_name`: opcional, `string`, max 140.
- `insurances[].valid_through`: opcional, `date`.
- `insurances[].is_primary`: opcional, `boolean`.
- `insurances[].is_active`: opcional, `boolean`.

### Response (201)

```json
{
  "data": {
    "id": 15,
    "number": "P-000123",
    "last_name": "Perez",
    "first_name": "Ana",
    "middle_name": "Maria",
    "birth_date": "1990-04-21",
    "gender": "female",
    "email": "ana.perez@mail.com",
    "phone": "1144556677",
    "mobile": "11999888777",
    "address": "Calle 123",
    "identifications": [
      {
        "id": 9,
        "identification_type_id": 1,
        "number": "30111222",
        "is_primary": true,
        "issued_country": "AR",
        "valid_through": null
      }
    ],
    "insurances": [
      {
        "id": 4,
        "insurance_type_id": 1,
        "insurance_plan_id": 2,
        "insurance_type_name": "OSDE",
        "insurance_plan_name": "Plan 210",
        "affiliate_number": "SOC-778899",
        "holder_name": "Ana Perez",
        "valid_through": "2027-12-31",
        "is_primary": true,
        "is_active": true
      }
    ]
  }
}
```

## 2) Busqueda/listado de pacientes

- Metodo: `GET`
- Endpoint: `/api/v1/patients`

### Query params

- `search`: opcional, `string` max 120. Busca por `number`, `last_name`, `first_name`, `middle_name`, `email`, `phone`, `mobile`, numero de identificacion y datos de cobertura (`affiliate_number`, `holder_name`).
- `per_page`: opcional, `integer` min 1 max 100. Default `20`.
- `include`: opcional. Relaciones permitidas: `identifications`, `insurances`.

Formas validas para `include`:
- CSV: `?include=identifications,insurances`
- Array: `?include[]=identifications&include[]=insurances`

### Ejemplo

`GET /api/v1/patients?search=jonsson&per_page=20&include=identifications,insurances`

### Response (200)

```json
{
  "data": [
    {
      "id": 15,
      "number": "P-000123",
      "last_name": "Jonsson",
      "first_name": "Ana",
      "middle_name": null,
      "birth_date": "1990-04-21",
      "gender": "female",
      "email": "ana.jonsson@mail.com",
      "phone": "1144556677",
      "mobile": "11999888777",
      "address": "Calle 123",
      "identifications": [
        {
          "id": 9,
          "identification_type_id": 1,
          "number": "30111222",
          "is_primary": true,
          "issued_country": "AR",
          "valid_through": null
        }
      ],
      "insurances": [
        {
          "id": 4,
          "insurance_type_id": 1,
          "insurance_plan_id": 2,
          "insurance_type_name": "OSDE",
          "insurance_plan_name": "Plan 210",
          "affiliate_number": "SOC-778899",
          "holder_name": "Ana Jonsson",
          "valid_through": "2027-12-31",
          "is_primary": true,
          "is_active": true
        }
      ]
    }
  ],
  "meta": {
    "current_page": 1,
    "per_page": 20,
    "last_page": 1,
    "total": 1
  }
}
```

## 3) Modificar datos de un paciente

- Metodo: `PATCH`
- Endpoint: `/api/v1/patients/{patientId}`

### Request body (parcial, solo campos a cambiar)

```json
{
  "last_name": "Jonsson",
  "email": "ana.jonsson@mail.com",
  "phone": "1144001122",
  "is_active": true
}
```

Campos permitidos:
- `number` (unico)
- `last_name`, `first_name`, `middle_name`
- `birth_date`, `gender`
- `email`, `phone`, `mobile`, `address`
- `is_active`
- `identifications[]` (opcional; si se envia, reemplaza la lista actual completa)
- `insurances[]` (opcional; si se envia, reemplaza la lista actual completa)

### Response (200)

Mismo formato que `POST /api/v1/patients`.

## 4) Alta de orden

- Metodo: `POST`
- Endpoint: `/api/v1/orders`

Nota importante:
- Si `order.number` no se envia o viene vacio, se autogenera con reglas de numeracion.
- `order.patient_id` es obligatorio y debe referenciar un paciente existente.

### Request body

```json
{
  "requirements": [
    {
      "requirement_type_id": 10,
      "value": "AUT-99881"
    },
    {
      "requirement_type_id": 11,
      "value": null
    }
  ],
  "practices": [
    {
      "practice_type_id": 5,
      "quantity": 1,
      "patient_price": 1200.0,
      "insurance_price": 3800.0,
      "billable": true,
      "payload": {
        "status": "valued"
      }
    }
  ],
  "bill": {
    "amount_paid": 500.0,
    "amount_balance": 700.0
  },
  "order": {
    "patient_id": 15,
    "number": null,
    "origin_id": 1,
    "service_id": 1,
    "order_type_id": 1,
    "physician_id": 1,
    "status": "new",
    "admin_status": "pending",
    "priority": "normal",
    "ordered_at": "2026-03-10T15:22:00Z",
    "promised_for": "2026-03-11T08:00:00Z",
    "clinical_data": "Paciente en ayunas.",
    "indications": [{ "text": "No fumar 8h" }],
    "valuations": [{ "code": "RISK", "value": "LOW" }],
    "notes": "Orden creada por recepcion"
  },
  "studies": [
    {
      "study_type_id": 1,
      "status": "pending",
      "priority": "normal",
      "promised_for": "2026-03-11T08:00:00Z"
    }
  ]
}
```

Notas sobre facturacion:
- `practices` es opcional y guarda las practicas valorizadas enviadas por frontend.
- `bill` es opcional y permite registrar el pago/factura en el alta.
- Si una practica tiene `billable=true` y se envia `bill`, se vincula a esa factura.

### Response (201)

```json
{
  "data": {
    "id": 77,
    "number": "A030001",
    "patient_id": 15,
    "status": "new",
    "admin_status": "pending",
    "priority": "normal",
    "ordered_at": "2026-03-10T15:22:00+00:00",
    "promised_for": "2026-03-11T08:00:00+00:00",
    "requirements": [
      {
        "id": 501,
        "requirement_type_id": 10,
        "value": "AUT-99881"
      },
      {
        "id": 502,
        "requirement_type_id": 11,
        "value": null
      }
    ],
    "practices": [
      {
        "id": 330,
        "practice_type_id": 5,
        "bill_id": 210,
        "quantity": 1,
        "patient_price": 1200,
        "insurance_price": 3800,
        "payload": {
          "status": "valued"
        }
      }
    ],
    "bill": {
      "id": 210,
      "amount_balance": 700,
      "amount_paid": 500,
      "status": "partial",
      "paid_at": null
    },
    "studies": [
      {
        "id": 120,
        "study_type_id": 1,
        "ordinal": 1,
        "status": "pending",
        "priority": "normal",
        "tests": [
          {
            "id": 881,
            "test_type_id": 1,
            "sample_id": 550,
            "status": "pending",
            "ordinal": 1
          }
        ]
      }
    ]
  }
}
```

## 5) Estimar fecha de entrega (sin guardar orden)

- Metodo: `POST`
- Endpoint: `/api/v1/orders/estimate-delivery-dates`

Objetivo:
- Permite al frontend calcular fecha de entrega por estudio y fecha final de la orden sin persistir datos.

### Request body

```json
{
  "order": {
    "ordered_at": "2026-03-10T15:22:00Z"
  },
  "studies": [
    {
      "study_type_id": 1
    },
    {
      "study_type_id": 2,
      "promised_for": "2026-03-12T08:00:00Z"
    }
  ]
}
```

Reglas:
- `studies` es requerido, minimo 1 item.
- `study_type_id` es requerido y debe existir.
- `order.ordered_at` es opcional; si no se envia, backend usa `now()`.
- Si un estudio trae `promised_for`, se respeta.
- Si no trae `promised_for`, backend lo calcula con:
  - calendario laboral,
  - feriados,
  - dias de procesamiento del estudio,
  - `turnaround_days` configurado en `study_type`.
- `promised_for` de la orden es la mayor fecha entre los estudios.

### Response (200)

```json
{
  "data": {
    "ordered_at": "2026-03-10T15:22:00+00:00",
    "promised_for": "2026-03-12T08:00:00+00:00",
    "studies": [
      {
        "index": 1,
        "study_type_id": 1,
        "promised_for": "2026-03-11T08:00:00+00:00"
      },
      {
        "index": 2,
        "study_type_id": 2,
        "promised_for": "2026-03-12T08:00:00+00:00"
      }
    ]
  }
}
```

## 6) Modificar datos de una orden

- Metodo: `PATCH`
- Endpoint: `/api/v1/orders/{orderId}`

### Request body (parcial, solo campos a cambiar)

```json
{
  "status": "in_progress",
  "admin_status": "controlled",
  "priority": "high",
  "promised_for": "2026-03-11T10:00:00Z",
  "notes": "Se actualiza prioridad"
}
```

Campos permitidos:
- `origin_id`, `service_id`, `order_type_id`, `physician_id`
- `status`: `new | in_progress | cancelled | released | posted`
- `admin_status`: `pending | controlled | closed`
- `priority`: `normal | high`
- `ordered_at`, `promised_for`
- `clinical_data`, `indications`, `valuations`, `notes`
- `requirements`: arreglo opcional de `{ requirement_type_id, value }`

Notas sobre `requirements`:
- Si `requirements` se envia en `PATCH`, backend reemplaza la lista actual completa.
- `value` es opcional y se usa para requerimientos de tipo `data`.

### Resolucion de requerimientos

- `POST /api/v1/admission-requirements/resolve`

Devuelve los requerimientos de los estudios enviados, deduplicados y con la
lista de estudios que los originan. Cada item trae `category`
(`indication|administrative|technical`), `type` (`info|data|printable`),
`severity` (solo en `indication`) y `layout`.

Body opcional `categories: []` para acotar el alcance; si se omite devuelve
todas las categorias. Admision pide `["administrative"]` y toma de muestras
`["technical"]`.

### Documento de un requerimiento imprimible

- `GET /api/v1/admission-requirements/{requirementType}/document`

Devuelve el PDF **inline** (`Content-Type` del archivo, `Content-Disposition:
inline`), autenticado igual que el resto de la API — no es una URL firmada, asi
que hay que consumirlo como blob y no linkearlo directo.

El archivo se resuelve desde `layout.source.disk` + `layout.source.path`, con lo
cual pasar `DOCUMENTS_DISK=s3` en produccion no requiere cambios de codigo. Los
PDFs de indicaciones viven en la carpeta `requirements/` del disco de
documentos (`storage/app/private/requirements` con el disco `local`).

Errores:
- `422` si el requerimiento no es `printable`, o si su layout es `report`
  (apunta a un servicio externo) en lugar de `document`.
- `404` si el requerimiento es imprimible pero el archivo no esta en el disco.

### Response (200)

Mismo formato que `POST /api/v1/orders`.

## 7) Agregar estudio a una orden

- Metodo: `POST`
- Endpoint: `/api/v1/orders/{orderId}/studies`

### Request body

```json
{
  "study_type_id": 2,
  "ordinal": 2,
  "status": "pending",
  "priority": "normal",
  "promised_for": "2026-03-11T08:00:00Z",
  "indications": [{ "text": "Ayuno 8h" }],
  "valuations": [{ "code": "URG", "value": false }],
  "tests": [
    {
      "test_type_id": 3,
      "site_id": 1,
      "reporting_type_id": 3,
      "reference_value_id": 3,
      "status": "pending",
      "ordinal": 1
    }
  ]
}
```

Notas:
- `tests` es opcional.
- Si no se envia `tests`, backend arma los tests segun `study_type -> test_types`.
- Si no se envia `ordinal`, backend usa el siguiente ordinal libre de la orden.

### Response (200)

Mismo formato que `POST /api/v1/orders`.

## 8) Quitar estudio de una orden

- Metodo: `DELETE`
- Endpoint: `/api/v1/orders/{orderId}/studies/{studyId}`

Comportamiento:
- Si el estudio no pertenece a la orden indicada, devuelve `422`.
- Si los tests del estudio no quedan asociados a otro estudio, se eliminan logicamente.

### Response (200)

Mismo formato que `POST /api/v1/orders`.

## 9) Errores esperables

Formato base:

```json
{
  "error": {
    "code": "ValidationException",
    "message": "Validation failed",
    "correlation_id": "..."
  }
}
```

Casos comunes:
- `401` token invalido o ausente.
- `400` tenant no resuelto (`X-Tenant-ID`).
- `403` sin permisos.
- `404` recurso no encontrado (`orderId`, `studyId`).
- `422` validacion de payload o estudio fuera de la orden.
