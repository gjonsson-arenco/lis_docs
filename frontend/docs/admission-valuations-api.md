# API - Valorizaciones de Admision

Base URL: `http://localhost:8000/api`

Headers requeridos:
- `Authorization: Bearer <cognito_access_token>`
- `X-Tenant-ID: <tenant_id>`
- `Content-Type: application/json`

Middleware de estas rutas:
- `throttle:auth`
- `tenant`
- `auth.cognito:access`

## 1) Cotizar valorizacion de admision

- Metodo: `POST`
- Endpoint: `/api/v1/admission-valuations/quote`

Este endpoint soporta los 2 modos:
- Prestador `manual`: backend valoriza con convenios + nomenclador + listas diferenciales (excepcion).
- Prestador `api`: backend envia payload al endpoint configurado del prestador y normaliza respuesta.

### Request body

```json
{
  "provider_id": 1,
  "order": {
    "id": 88,
    "number": "A030088",
    "insurance_id": 10,
    "origin_id": 2,
    "service_id": 5
  },
  "patient": {
    "id": 15,
    "number": "P-000123",
    "last_name": "Perez",
    "first_name": "Ana",
    "gender": "female",
    "birth_date": "1990-04-21"
  },
  "physician": {
    "id": 7,
    "name": "Dra. Maria Rossi"
  },
  "studies": [
    {
      "study_type_id": 3,
      "ordinal": 1
    },
    {
      "study_type_id": 4,
      "ordinal": 2,
      "practice_type_ids": [2, 3]
    }
  ]
}
```

### Validaciones principales

- `provider_id`: requerido, `exists:admission_valuation_providers,id`
- `order`: requerido, objeto
- `order.insurance_id`: opcional, `exists:insurances,id`
- `patient`: requerido, objeto
- `physician`: opcional, objeto
- `studies`: requerido, array, minimo 1
- `studies[].study_type_id`: requerido, `exists:study_types,id`
- `studies[].practice_type_ids[]`: opcional, `exists:practice_types,id`

### Comportamiento

1. El backend toma cada `study_type_id` y resuelve practicas desde `study_practice_type`.
2. Si en un estudio viene `practice_type_ids`, filtra solo esas practicas.
3. Segun el prestador:
   - `manual`: busca convenio activo por prestador + cobertura del paciente, aplica lista base y lista de excepcion (si existe).
   - `api`: llama `api_endpoint` del prestador y traduce la respuesta al formato comun.
4. Devuelve listado de practicas facturables con estado `valued` o `unresolved`.

### Response (200)

```json
{
  "data": {
    "provider": {
      "id": 1,
      "code": "PREST-001",
      "name": "Prestador Central",
      "valuation_mode": "manual"
    },
    "valuation_mode": "manual",
    "agreement": {
      "id": 12,
      "description": "Convenio PAMI 2026",
      "nomenclator_id": 3,
      "exception_nomenclator_id": 5,
      "value_multiplier": 1.08,
      "valid_from": "2026-01-01",
      "valid_through": "2026-12-31"
    },
    "practices": [
      {
        "study_index": 1,
        "study_type_id": 3,
        "study_type_code": "HEMOGR",
        "study_type_name": "Hemograma",
        "practice_type_id": 2,
        "practice_code": "PR-HEM",
        "practice_name": "Practica Hemograma",
        "quantity": 1,
        "status": "valued",
        "pricing_source": "manual",
        "unit_price": 5200,
        "total_price": 5200,
        "insurance_coverage_pct": 80,
        "insurance_amount": 4160,
        "patient_amount": 1040,
        "requires_authorization": false
      },
      {
        "study_index": 2,
        "study_type_id": 4,
        "study_type_code": "GLUC",
        "study_type_name": "Glucosa",
        "practice_type_id": 3,
        "practice_code": "PR-GLU",
        "practice_name": "Practica Glucosa",
        "quantity": 1,
        "status": "unresolved",
        "reason": "price_not_found_for_practice",
        "pricing_source": "manual"
      }
    ],
    "totals": {
      "valued_count": 1,
      "unresolved_count": 1,
      "total_price": 5200,
      "insurance_amount": 4160,
      "patient_amount": 1040
    }
  }
}
```

## 2) Catalogos para front (solo lectura)

Estos endpoints son para levantar combos y grillas en UI de admision/valorizacion.

- Metodo: `GET`
- Base path: `/api/v1/admission-valuations/catalogs`

### 2.1 Indice de catalogos

- Endpoint: `/api/v1/admission-valuations/catalogs`

Response:

```json
{
  "data": [
    { "key": "providers", "path": "/api/v1/admission-valuations/catalogs/providers" },
    { "key": "insurance-types", "path": "/api/v1/admission-valuations/catalogs/insurance-types" },
    { "key": "insurance-plans", "path": "/api/v1/admission-valuations/catalogs/insurance-plans" },
    { "key": "nomenclators", "path": "/api/v1/admission-valuations/catalogs/nomenclators" },
    { "key": "practice-types", "path": "/api/v1/admission-valuations/catalogs/practice-types" },
    { "key": "nomenclator-practice-prices", "path": "/api/v1/admission-valuations/catalogs/nomenclator-practice-prices" },
    { "key": "pricing-agreements", "path": "/api/v1/admission-valuations/catalogs/pricing-agreements" }
  ]
}
```

### 2.2 Endpoints de catalogo

- Prestadores: `/api/v1/admission-valuations/catalogs/providers`
- Tipos de cobertura: `/api/v1/admission-valuations/catalogs/insurance-types`
- Planes: `/api/v1/admission-valuations/catalogs/insurance-plans`
- Nomencladores: `/api/v1/admission-valuations/catalogs/nomenclators`
- Tipos de practica: `/api/v1/admission-valuations/catalogs/practice-types`
- Lista de precios: `/api/v1/admission-valuations/catalogs/nomenclator-practice-prices`
- Convenios: `/api/v1/admission-valuations/catalogs/pricing-agreements`

### 2.3 Query params comunes

- `search`: opcional.
- `is_active`: opcional, `true|false`.
- `per_page`: opcional (default 50, max 500).
- `order_by`: opcional.
- `order_dir`: opcional, `asc|desc`.
- `include`: opcional (CSV o array).

Ejemplos:
- `/api/v1/admission-valuations/catalogs/providers?is_active=true&order_by=name&order_dir=asc`
- `/api/v1/admission-valuations/catalogs/insurance-types?include=plans`
- `/api/v1/admission-valuations/catalogs/nomenclator-practice-prices?nomenclator_id=1&include=nomenclator,practiceType`
- `/api/v1/admission-valuations/catalogs/pricing-agreements?provider_id=1&active_on=2026-03-12&include=provider,insuranceType,insurancePlan,nomenclator,exceptionNomenclator`

### 2.4 Filtros especificos

- `insurance-plans`: `insurance_type_id`
- `nomenclator-practice-prices`: `nomenclator_id`, `practice_type_id`
- `pricing-agreements`: `provider_id`, `insurance_type_id`, `insurance_plan_id`, `active_on`

### 2.5 Includes permitidos

- `providers`: `pricingAgreements`
- `insurance-types`: `plans`
- `insurance-plans`: `insuranceType`
- `nomenclators`: `practicePrices`
- `practice-types`: (sin include)
- `nomenclator-practice-prices`: `nomenclator`, `practiceType`
- `pricing-agreements`: `provider`, `insuranceType`, `insurancePlan`, `nomenclator`, `exceptionNomenclator`

### 2.6 Formato de respuesta de listados

```json
{
  "data": [
    {
      "id": 1,
      "code": "PREST_MAN",
      "name": "Prestador Manual Demo"
    }
  ],
  "meta": {
    "current_page": 1,
    "per_page": 50,
    "last_page": 1,
    "total": 1
  }
}
```

## 3) Carga de configuracion (admin)

Se reutiliza el endpoint de catalogos de settings para ABM de listas y convenios.

### Catalogos nuevos

- `/api/v1/admin/settings/catalogs/admission-valuation-providers`
- `/api/v1/admin/settings/catalogs/nomenclators`
- `/api/v1/admin/settings/catalogs/nomenclator-practice-prices`
- `/api/v1/admin/settings/catalogs/insurance-pricing-agreements`

### Flujo recomendado de parametrizacion

1. Crear prestador (`admission-valuation-providers`) con `valuation_mode=manual` o `api`.
2. Crear nomenclador base (`nomenclators`).
3. Cargar precios por practica (`nomenclator-practice-prices`).
4. Crear convenio (`insurance-pricing-agreements`) para combinar prestador + cobertura + nomenclador.
5. Opcional: agregar `exception_nomenclator_id` para lista diferencial de excepcion.

## 4) Formato esperado para prestadores API externos

Cuando el prestador tiene `valuation_mode=api`, el backend enviara `POST` al `api_endpoint` configurado y espera una respuesta con:

```json
{
  "data": {
    "practices": [
      {
        "study_index": 1,
        "practice_type_id": 2,
        "unit_price": 5200,
        "total_price": 5200,
        "insurance_coverage_pct": 80,
        "insurance_amount": 4160,
        "patient_amount": 1040,
        "requires_authorization": false
      }
    ]
  }
}
```

Si el endpoint remoto falla o no responde el formato esperado, la API local devuelve practicas con `status=unresolved` y `reason` explicando el motivo.
