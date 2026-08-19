# Cómo construir una Regla de Facturación

Guía práctica para autores de reglas usando el motor nuevo (`lis-rules-engine`). Cubre el ABM en `/settings/rules/billing`, el formato del JSON de la regla y ejemplos listos para pegar.

> El motor viejo (`billing_rules` / `BillingRuleEngine`) fue eliminado: hoy toda la valorización pasa por el `rules-engine`. [BILLING_RULES_ENGINE.md](./BILLING_RULES_ENGINE.md) queda solo como referencia histórica.
> Este documento se enfoca en **cómo escribir una regla**.

## 1. Modelo mental

Una regla se dispara en el flujo de valorización y **transforma la lista de prácticas** que se van a facturar. Solo hay **dos acciones primitivas**:

- `add_practice`: agrega una práctica.
- `remove_practice`: elimina todas las ocurrencias de una práctica.

Un "reemplazo" se expresa como `remove` + `add` en la misma regla.

Cada regla tiene tres componentes:

| Componente | ¿Dónde vive?                          | ¿Para qué?                                                                          |
| ---------- | ------------------------------------- | ----------------------------------------------------------------------------------- |
| **Scope**  | Campos estructurados (multi-select)   | ¿A qué cobertura / plan / convenio aplica?                                          |
| **Trigger**| `trigger_json` (opcional)             | ¿Qué debe estar presente en la orden para que la regla dispare? Puede quedar vacío. |
| **Acciones**| `actions_json` (requerido)           | Qué se agrega o elimina cuando dispara.                                             |

## 2. Cómo se crea desde la UI

1. Abrí **Configuración → Reglas de Facturación** (`/settings/rules/billing`).
2. **Nueva regla** → se abre el panel derecho.
3. Completá los campos estructurados (obligatorios en negrita):
   - **Código**: identificador único (ej: `ORINA_CONSOLIDATE`). Sin espacios, mayúsculas y guiones bajos.
   - **Nombre**: descripción legible.
   - **Prioridad**: número entero. **Menor = ejecuta primero**.
   - **Activa**: switch (default: sí).
   - **Vigente desde / hasta**: fechas opcionales.
   - **Coberturas / Planes / Convenio**: scope (ver §3).
4. Pegá el **`trigger_json`** (o dejalo vacío para que dispare siempre).
5. Pegá el **`actions_json`**.
6. **Crear regla**.

El backend valida:
- Sintaxis JSON.
- Que los operadores del trigger sean válidos.
- Que cada `practiceCode` de las acciones exista en el catálogo de `practice_types`.

Al guardar, el backend notifica al `rules-engine` que invalide su cache de reglas.

## 3. Scope — a quién le aplica

El scope se define con **campos estructurados**, no en JSON:

- **Coberturas** (multi-select): la orden debe tener alguna de estas `insurance_type`.
- **Planes** (multi-select): la orden debe tener alguno de estos `insurance_plan`.
- **Convenio de precios**: si se elige, la regla aplica **solo** a ese convenio.

Reglas de resolución:

| Coberturas | Planes | Comportamiento |
|-----------|--------|----------------|
| vacío     | vacío  | Aplica siempre (regla global) |
| `[OSDE]`  | vacío  | Aplica solo si la orden tiene OSDE |
| vacío     | `[210]`| Aplica solo si la orden tiene el plan 210 |
| `[OSDE]`  | `[210]`| Aplica si tiene OSDE **o** plan 210 (unión) |

Si además se define un **convenio**, la regla exige que el convenio de la orden coincida (AND con lo anterior).

## 4. Trigger — cuándo dispara

El `trigger_json` es un objeto con **una sola clave**: `all` (AND) o `any` (OR). Adentro va un array de **condiciones**.

Cada condición tiene tres campos:

```json
{ "fact": "<nombre>", "op": "<operador>", "value": <valor> }
```

### Facts disponibles

El backend arma los facts automáticamente al invocar el motor. Podés referenciarlos desde el trigger:

| Fact                              | Tipo       | Descripción                                              |
| --------------------------------- | ---------- | -------------------------------------------------------- |
| `studyCodes`                      | `string[]` | Códigos de los estudios de la orden.                     |
| `practiceCodes`                   | `string[]` | Códigos de las prácticas actuales (post reglas previas). |
| `insuranceTypeCode`               | `string`   | Cobertura de la orden.                                   |
| `insurancePlanCode`               | `string`   | Plan de la orden.                                        |
| `insurancePricingAgreementCode`   | `string`   | Convenio de precios.                                     |
| `orderOriginCode`                 | `string`   | Origen de la orden.                                      |
| `orderServiceCode`                | `string`   | Servicio de la orden.                                    |
| `isPrivate`                       | `boolean`  | `true` si es privada.                                    |
| `date`                            | `string`   | Fecha de la valorización (`YYYY-MM-DD`).                 |

### `studyCodes` vs `practiceCodes`

Los dos son arrays de códigos y en la mayoría de las órdenes traen lo mismo,
pero no son intercambiables:

- `studyCodes` es **fijo**: los estudios que pidió la orden, sin importar qué
  hicieron las reglas anteriores.
- `practiceCodes` se **recalcula antes de cada regla**, sobre la lista ya
  modificada por las reglas de prioridad menor.

Usá `practiceCodes` cuando una regla tenga que ver el resultado de otra
(cadenas de reemplazo), y `studyCodes` cuando quieras decidir sobre lo que
pidió el médico independientemente de lo que hayan tocado las demás. Ojo: con
`practiceCodes` el `priority` deja de ser cosmético — una regla que quita un
código impide que dispare cualquier regla posterior que lo necesite.

Las reglas migradas del LIS viejo (código `FR_*`) usan `practiceCodes`, porque
el motor legacy funcionaba así.

### Operadores soportados

| Operador                | Uso típico                       | Ejemplo de `value` |
| ----------------------- | -------------------------------- | ------------------ |
| `equal`, `notEqual`     | igualdad exacta                  | `"OSDE"`, `true`   |
| `in`, `notIn`           | pertenencia a un conjunto        | `["OSDE","SWISS"]` |
| `lessThan`, `lessThanInclusive` | comparación numérica     | `100`              |
| `greaterThan`, `greaterThanInclusive` | comparación numérica | `1000`           |
| `contains`, `doesNotContain` | substring o item en array   | `"URGENTE"`        |
| `includesAll`           | array del fact contiene TODOS    | `["ORINA","SED"]`  |
| `includesAny`           | array del fact contiene ALGUNO   | `["A","B"]`        |

### Trigger vacío

Si dejás `trigger_json` en blanco, la regla dispara **siempre que el scope aplique**. Útil para reglas incondicionales de un plan/cobertura.

## 5. Acciones — qué hace

El `actions_json` es un **array** con al menos una acción. Se ejecutan en orden.

### `add_practice`

```json
{ "type": "add_practice", "practiceCode": "PR_ORINA_COMPLETA", "quantity": 1 }
```

- `practiceCode` (requerido): debe existir en el catálogo.
- `quantity` (opcional, default `1`): cantidad a agregar.

Si la práctica ya existe en la lista, se agrega igual como una entrada nueva; el motor **consolida sumando cantidades** al final de todas las reglas.

### `remove_practice`

```json
{ "type": "remove_practice", "practiceCode": "PR_SEDIMENTO_URINARIO" }
```

Elimina **todas las ocurrencias** de esa práctica de la lista actual.

### Orden de ejecución

1. Reglas se ordenan por `priority` ascendente (menor dispara primero), y por `id` como desempate.
2. Cada regla que dispara aplica sus acciones **secuencialmente** sobre la lista actual.
3. Al finalizar todas las reglas, la lista se **consolida**: prácticas repetidas se suman por `practiceCode`.

## 6. Ejemplos completos

### Ejemplo 1 — Consolidar Orinas

**Objetivo**: si la orden tiene `ORINA_COMPLETA` y `SEDIMENTO_URINARIO`, facturar solo `PR_ORINA_COMPLETA`.

| Campo | Valor |
| --- | --- |
| Código | `ORINA_CONSOLIDATE` |
| Nombre | Unificar orina completa y sedimento |
| Prioridad | `30` |
| Scope | (vacío = todas las coberturas) |

`trigger_json`:
```json
{
  "all": [
    { "fact": "studyCodes", "op": "includesAll", "value": ["ORINA_COMPLETA", "SEDIMENTO_URINARIO"] }
  ]
}
```

`actions_json`:
```json
[
  { "type": "remove_practice", "practiceCode": "PR_SEDIMENTO_URINARIO" },
  { "type": "add_practice",    "practiceCode": "PR_ORINA_COMPLETA", "quantity": 1 }
]
```

### Ejemplo 2 — Suprimir Glucosa cuando hay HbA1c (OSDE / Swiss)

**Objetivo**: con OSDE o Swiss Medical, si se piden `GLUCOSA` y `HEMOGLOBINA_GLICADA`, facturar solo la HbA1c.

| Campo | Valor |
| --- | --- |
| Código | `GLUCOSE_SUPPRESS_HBA1C` |
| Nombre | Suprimir glucosa cuando hay HbA1c |
| Prioridad | `20` |
| Coberturas | `OSDE`, `SWISS_MEDICAL` |
| Vigencia | 2026-01-01 → 2026-12-31 |

`trigger_json`:
```json
{
  "all": [
    { "fact": "studyCodes", "op": "includesAll", "value": ["GLUCOSA", "HEMOGLOBINA_GLICADA"] }
  ]
}
```

`actions_json`:
```json
[
  { "type": "remove_practice", "practiceCode": "PR_GLUCOSA" }
]
```

### Ejemplo 3 — Agregar plaquetas al hemograma

**Objetivo**: cuando se pide un hemograma, sumar automáticamente el recuento de plaquetas.

| Campo | Valor |
| --- | --- |
| Código | `HEMOGRAM_ADD_PLATELETS` |
| Nombre | Agregar recuento de plaquetas al hemograma |
| Prioridad | `10` |
| Scope | (vacío) |

`trigger_json`:
```json
{
  "all": [
    { "fact": "studyCodes", "op": "includesAny", "value": ["HEMOGRAMA"] }
  ]
}
```

`actions_json`:
```json
[
  { "type": "add_practice", "practiceCode": "PR_RECUENTO_PLAQUETAS", "quantity": 1 }
]
```

### Ejemplo 4 — Regla global sin trigger (siempre dispara)

**Objetivo**: para el convenio "CONV_PROMO_2026", siempre agregar una práctica administrativa fija.

| Campo | Valor |
| --- | --- |
| Código | `PROMO_2026_ADD_FEE` |
| Nombre | Cargo administrativo promoción 2026 |
| Prioridad | `500` |
| Convenio | `CONV_PROMO_2026` |

`trigger_json`: **vacío**

`actions_json`:
```json
[
  { "type": "add_practice", "practiceCode": "PR_CARGO_ADMIN", "quantity": 1 }
]
```

### Ejemplo 5 — Trigger con OR (`any`)

**Objetivo**: si viene `URGENCIA` **o** `GUARDIA`, agregar recargo.

`trigger_json`:
```json
{
  "any": [
    { "fact": "studyCodes",       "op": "includesAny", "value": ["URGENCIA_LAB"] },
    { "fact": "orderOriginCode",  "op": "equal",       "value": "GUARDIA" }
  ]
}
```

`actions_json`:
```json
[
  { "type": "add_practice", "practiceCode": "PR_RECARGO_URGENCIA", "quantity": 1 }
]
```

## 7. Buenas prácticas

- **Códigos claros y estables**: `ORINA_CONSOLIDATE`, `HEMOGRAM_ADD_PLATELETS`. Evitá códigos numéricos.
- **Prioridad en incrementos de 10**: dejá espacio para insertar reglas en el medio (`10`, `20`, `30`, ...). Sirve cuando el orden de ejecución importa (una regla puede depender de que otra ya haya corrido).
- **Un solo cambio por regla**: preferí varias reglas chicas sobre una gigante. Facilita auditar y desactivar puntualmente.
- **Nombrá el propósito**: el campo Nombre lo lee el equipo de negocio; que sea autoexplicativo.
- **Scope explícito**: si una regla es específica de una cobertura, no la dejes global "por si acaso". Reduce falsos positivos.
- **Vigencia**: usá `valid_from` / `valid_through` para reglas estacionales o promocionales. Al vencer, no hace falta desactivarlas manualmente.
- **Referenciá códigos existentes**: si un `practiceCode` no existe en el catálogo, el backend rechaza el guardado.
- **Test antes de activar**: mientras se implementa el preview, creá la regla con `is_active = false`, verificá manualmente y activá.

## 8. Errores comunes

| Síntoma                                                     | Causa probable                                                            |
| ----------------------------------------------------------- | ------------------------------------------------------------------------- |
| "practiceCode 'PR_X' does not exist"                        | El código de la práctica no existe. Chequealo en catálogo de prácticas.   |
| "trigger_json must contain exactly one of 'all' or 'any'"   | El trigger tiene ambas claves o ninguna. Elegí una.                       |
| "condition #0 uses unsupported operator 'X'"                | Operador mal escrito o no listado en §4.                                  |
| "actions_json must be an array with at least one action"    | El array quedó vacío o no es array.                                       |
| La regla se guarda pero no dispara                           | Revisar (1) `is_active`, (2) vigencia vs fecha de la orden, (3) scope, (4) trigger. |
| Los cambios no se ven inmediatos                             | El `rules-engine` cachea. Se invalida por push del backend al guardar, o por TTL (default 5 min). |

## 9. Cheatsheet — plantilla mínima

Copiá esto como punto de partida:

**trigger_json**:
```json
{
  "all": [
    { "fact": "studyCodes", "op": "includesAll", "value": [""] }
  ]
}
```

**actions_json**:
```json
[
  { "type": "add_practice",    "practiceCode": "", "quantity": 1 },
  { "type": "remove_practice", "practiceCode": "" }
]
```

## 10. Ruta técnica de una evaluación

Para quien quiera entender el flujo end-to-end:

1. Backend Laravel llama `POST http://rules-engine/api/v1/billing/evaluate` con:
   ```json
   { "tenantId": "lis-default", "context": {...}, "studies": [...], "practices": [...] }
   ```
2. El rules-engine:
   - Levanta las reglas del cache (o pull HTTP a `GET /api/v1/internal/rules-catalog?domain=billing` en el backend si expiró).
   - Filtra por vigencia + scope.
   - Ordena por prioridad ascendente.
   - Por cada regla: arma `facts`, corre el trigger con `json-rules-engine`, y si matchea aplica las acciones.
   - Consolida prácticas repetidas.
3. Devuelve `{ practices, trace, evaluationMeta }`. El `trace` indica qué regla se aplicó, cuántas prácticas agregó/quitó y cuáles.

Referencias de código:
- Applicator: `lis-rules-engine/src/application/billing/billing-service.ts`
- Cliente HTTP al backend: `lis-rules-engine/src/infrastructure/billing/backend-rule-catalog-client.ts`
- CRUD backend: `lis-backend/app/Http/Controllers/Api/V1/Admin/RulesController.php`
- Serialización catálogo: `lis-backend/app/Http/Controllers/Api/V1/Internal/InternalRulesCatalogController.php`
