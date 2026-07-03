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
      "insurance_id": 1,
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
- `insurances[].insurance_id`: requerido si existe item, `exists:insurances,id`.
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
        "insurance_id": 1,
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

## 2) Alta de orden

- Metodo: `POST`
- Endpoint: `/api/v1/orders`

Nota importante:
- Si `order.number` no se envia o viene vacio, se autogenera con reglas de numeracion.

### Request body

```json
{
  "patient": {
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
        "identification_type_id": 1,
        "number": "30111222",
        "is_primary": true,
        "issued_country": "AR",
        "valid_through": null
      }
    ]
  },
  "order": {
    "number": null,
    "origin_id": 1,
    "service_id": 1,
    "order_type_id": 1,
    "physician_id": 1,
    "insurance_id": 1,
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
      "ordinal": 1,
      "status": "pending",
      "priority": "normal",
      "promised_for": "2026-03-11T08:00:00Z",
      "indications": [{ "text": "Muestra en frio" }],
      "valuations": [{ "code": "URG", "value": false }],
      "tests": [
        {
          "test_type_id": 1,
          "site_id": 1,
          "reporting_type_id": 1,
          "reference_value_id": 1,
          "status": "pending",
          "ordinal": 1,
          "sample": {
            "sample_type_id": 1,
            "container_type_id": 1,
            "class": "primary",
            "status": "pending",
            "sampled_at": null,
            "sampled_by": null,
            "received_at": null,
            "received_by": null,
            "locator": "A-01",
            "volume": 3.5,
            "temperature": 21.0
          }
        }
      ]
    }
  ]
}
```

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

## 3) Modificar datos de una orden

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
- `origin_id`, `service_id`, `order_type_id`, `physician_id`, `insurance_id`
- `status`: `new | in_progress | cancelled | released | posted`
- `admin_status`: `pending | controlled | closed`
- `priority`: `normal | high`
- `ordered_at`, `promised_for`
- `clinical_data`, `indications`, `valuations`, `notes`

### Response (200)

Mismo formato que `POST /api/v1/orders`.

## 4) Agregar estudio a una orden

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

## 5) Quitar estudio de una orden

- Metodo: `DELETE`
- Endpoint: `/api/v1/orders/{orderId}/studies/{studyId}`

Comportamiento:
- Si el estudio no pertenece a la orden indicada, devuelve `422`.
- Si los tests del estudio no quedan asociados a otro estudio, se eliminan logicamente.

### Response (200)

Mismo formato que `POST /api/v1/orders`.

## 6) Errores esperables

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
