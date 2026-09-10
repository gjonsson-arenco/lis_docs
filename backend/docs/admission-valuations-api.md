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

Este endpoint soporta un flujo unico de orquestacion:
- Si `order.is_private=true`, la valorizacion se resuelve internamente en LIS.
- Si `order.is_private=false` y el `insurance_type` tiene `valuation_provider_id`, se solicita valorizacion externa por API.
- Si `order.is_private=false` y el `insurance_type` no tiene `valuation_provider_id`, se valoriza internamente en LIS.

### Request body

```json
{
  "order": {
    "id": null,
    "number": null,
    "is_private": false,
    "insurance_type_id": 1,
    "insurance_plan_id": 2,
    "origin_id": null,
    "service_id": null,
    "physician_id": 7
  },
  "patient": {
    "id": null,
    "number": null,
    "last_name": "Perez",
    "first_name": "Ana",
    "gender": "female",
    "birth_date": "1990-04-21"
  },
  "studies": [
    {
      "study_type_id": 3,
      "ordinal": 1
    },
    {
      "study_type_id": 4,
      "ordinal": 2
    }
  ]
}
```

### Validaciones principales

- `order`: opcional, objeto (puede venir `null` en pre-admision)
- `order.is_private`: opcional, boolean (default `false`)
- `order.insurance_type_id`: opcional, `exists:insurance_types,id`
- `order.insurance_plan_id`: opcional, `exists:insurance_plans,id`
- `order.physician_id`: opcional, `exists:physicians,id`
- `patient`: opcional, objeto (puede venir `null` en pre-admision)
- `patient.id`: opcional, entero (no requiere existir en LIS para cotizacion previa)
- `studies`: requerido, array, minimo 1
- `studies[].study_type_id`: requerido, `exists:study_types,id`

### Comportamiento

1. El backend toma cada `study_type_id` y resuelve practicas desde `study_practice_type`.
2. Evalua reglas de facturacion (`billing_rules`) en orden de prioridad:
  - `replace`: reemplaza practicas por una practica objetivo.
  - `add`: agrega una practica adicional.
  - `suppress`: elimina practicas de la lista.
3. Consolida practicas repetidas por `practice_type_id` sumando `quantity`.
4. Enrutamiento:
  - `order.is_private=true` => `manual`.
  - `order.is_private=false` + `insurance_type.valuation_provider_id` informado => `api`.
  - `order.is_private=false` + `insurance_type.valuation_provider_id` nulo => `manual`.
5. En modo `manual`: busca convenio activo por prestador + cobertura del paciente, aplica lista base y lista de excepcion.
6. En modo `api`: llama `api_endpoint` del prestador y normaliza la respuesta.
7. Calcula valores:
  - `value_to_pay`: valor a pagar segun cobertura/plan del convenio activo.
  - `private_value`: valor a pagar si fuera orden particular (sin consulta adicional, pre-calculado en paralelo).
8. Devuelve listado de practicas con estado individual (`status`) y estado global de cotizacion:
  - **Status global de cotización:**
    - `passed`: no hay practicas con `requires_authorization=true` y no requiere voucher.
    - `pending`: existe al menos una practica con autorizacion o la cobertura/plan requiere voucher.
  - **Status individual de cada práctica (en el objeto de la práctica):**
    - `passed`: practica es particular, o no requiere autorización y el financiador/plan no requieren voucher.
    - `pending`: practica requiere autorización, o el financiador/plan requieren voucher.
  - **Voucher por práctica:** cuando la cobertura o el plan requieren voucher, cada práctica valorizada
    hereda `requires_voucher=true`. Un voucher no salda la orden entera: al subirlo, el operador marca
    qué prácticas salda (`documents[].practice_type_ids`). Las prácticas que requieren voucher y no quedan
    saldadas se cobran como no cubiertas (`is_not_covered=true`: el paciente paga el precio convenido y la
    práctica ya no pide voucher ni autorización). Un documento sin `practice_type_ids` salda todas las
    prácticas (compatibilidad con clientes que sólo mandaban `practice_type_id`).

### Response (200)

```json
{
  "data": {
    "status": "pending",
    "requires_voucher": true,
    "billing_indications": {
      "insurance_type": [
        { "id": 1, "code": "AUTH", "name": "Requiere autorizacion administrativa", "description": "Validar autorizacion previa con financiador.", "category": "warning" }
      ],
      "insurance_plan": [
        { "id": 2, "code": "VOUCHER", "name": "Presentar voucher", "description": "Solicitar voucher o token vigente al paciente.", "category": "warning" }
      ]
    },
    "provider": {
      "id": 1,
      "code": "PREST-001",
      "name": "Prestador Central",
      "valuation_mode": "manual"
    },
    "valuation_mode": "manual",
    "valuation_reason": "provider_routing",
    "agreement": {
      "id": 12,
      "description": "Convenio PAMI 2026",
      "type": "LIST",
      "unit_value": null,
      "nomenclator_id": 3,
      "exception_nomenclator_id": 5,
      "valid_from": "2026-01-01",
      "valid_through": "2026-12-31"
    },
    "practices": [
      {
        "practice_type": {
          "id": 2,
          "code": "PR-HEM",
          "name": "Practica Hemograma"
        },
        "study_type_ids": [3],
        "quantity": 1,
        "value_to_pay": 4160,
        "private_value": 5200,
        "coverage": 80,
        "is_private": false,
        "requires_authorization": false,
        "requires_voucher": true,
        "status": "pending"
      },
      {
        "practice_type": {
          "id": 3,
          "code": "PR-GLU",
          "name": "Practica Glucosa"
        },
        "study_type_ids": [4],
        "quantity": 1,
        "value_to_pay": 2500,
        "private_value": 3200,
        "coverage": 80,
        "is_private": false,
        "requires_authorization": true,
        "requires_voucher": false,
        "status": "pending"
      }
    ],
    "rule_evaluation": {
      "base_practices_count": 3,
      "post_rules_count": 2,
      "post_consolidation_count": 2,
      "applied_rules": [
        {
          "rule_id": 20,
          "code": "HEMO_GLU_REPLACE",
          "action": "replace",
          "priority": 10,
          "applied": true,
          "changes": {
            "added": 1,
            "removed": 2,
            "affected_practice_type_ids": [2, 3, 8]
          }
        }
      ]
    },
    "totals": {
      "studies_count": 2,
      "practices_count": 2,
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
    { "key": "pricing-agreements", "path": "/api/v1/admission-valuations/catalogs/pricing-agreements" },
    { "key": "billing-indications", "path": "/api/v1/admission-valuations/catalogs/billing-indications" }
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
- Indicaciones de facturacion: `/api/v1/admission-valuations/catalogs/billing-indications`

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
- `billing-indications`: `insurance_type_id`, `insurance_plan_id`

### 2.5 Includes permitidos

- `providers`: `pricingAgreements`
- `insurance-types`: `plans`, `billingIndications`
- `insurance-plans`: `insuranceType`, `billingIndications`
- `nomenclators`: `practicePrices`
- `practice-types`: (sin include)
- `nomenclator-practice-prices`: `nomenclator`, `practiceType`
- `pricing-agreements`: `provider`, `insuranceType`, `insurancePlan`, `nomenclator`, `exceptionNomenclator`
- `billing-indications`: `insuranceTypes`, `insurancePlans`

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

- `/api/v1/admin/settings/catalogs/valuation-providers`
- `/api/v1/admin/settings/catalogs/nomenclators`
- `/api/v1/admin/settings/catalogs/nomenclator-practice-prices`
- `/api/v1/admin/settings/catalogs/insurance-pricing-agreements`
- `/api/v1/admin/settings/catalogs/billing-rules`
- `/api/v1/admin/settings/catalogs/billing-indications`

### Flujo recomendado de parametrizacion

1. Crear prestador (`valuation-providers`) con `valuation_mode=manual` o `api`.
2. Crear nomenclador base (`nomenclators`).
3. Cargar precios por practica (`nomenclator-practice-prices`).
4. Crear convenio (`insurance-pricing-agreements`) para combinar prestador + cobertura + nomenclador.
5. Configurar reglas (`billing-rules`) para `replace`, `add` y `suppress` con prioridad.
6. Opcional: agregar `exception_nomenclator_id` para lista diferencial de excepcion.

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
        "insurance_amount": 4160,
        "patient_amount": 1040,
        "requires_authorization": false
      }
    ]
  }
}
```

Si el endpoint remoto falla o no responde el formato esperado, la API local devuelve practicas con `status=unresolved` y `reason` explicando el motivo.
