# Motor de Reglas de Facturación (Billing Rules Engine)

## Propósito

El motor de reglas de facturación permite transformar dinámicamente la lista de prácticas que serán facturadas basándose en:
- Estudios solicitados
- Prácticas mapeadas
- Contexto de cobertura (insurance type/plan)
- Contexto de la orden (origen, servicio, etc.)
- Vigencia de la regla

Las reglas se aplican **antes de valorizar**, transformando la lista de prácticas de forma que al momento de calcular precios, las prácticas sean las correctas según la lógica de negocio.

## Ubicación en el Flujo

```
POST /api/v1/admission-valuations/quote
    ↓
1. Estudios → Prácticas base (via study_practice_type)
    ↓
2. ████ REGLAS DE FACTURACIÓN ████ ← Motor aquí
    ↓
3. Consolidación (merge de repetidas)
    ↓
4. Valorización (manual o API)
    ↓
Respuesta con prácticas finales
```

## Conceptos Básicos

### Regla (BillingRule)

Una regla de facturación define:
- **Cuándo dispararse** (triggers): qué estudios/prácticas deben estar presentes
- **Qué hacer** (action): reemplazar, agregar o suprimir prácticas
- **A qué aplicarse** (scope): contexto de cobertura y vigencia

### Estructura de una Regla

```php
BillingRule {
  id: integer
  code: string (40 chars) — código único identificador
  name: string (180 chars) — descripción legible
  action: enum('replace', 'add', 'suppress')
  priority: integer — orden de ejecución (menor primero)
  
  // Scoping: a qué contextos aplica
  insurance_type_ids: array — cobertura(s) [M:M]
  insurance_plan_ids: array — plan(es) [M:M]
  insurance_pricing_agreement_id: int — convenio específico (opcional)
  
  // Triggers: cuándo dispararse
  trigger_study_type_ids: array — estudios que deben estar presentes
  trigger_practice_type_ids: array — prácticas que deben estar presentes
  requires_all_triggers: bool — AND (true) vs OR (false)
  
  // Acciones
  target_practice_type_id: int — practica destino (para replace/add)
  target_practice_type_ids: array — múltiples prácticas destino
  target_quantity: int — cantidad a agregar/reemplazar (default 1)
  suppress_practice_type_ids: array — prácticas a eliminar (para suppress)
  
  // Vigencia
  valid_from: date — inicio de validez (inclusive)
  valid_through: date — fin de validez (inclusive)
  
  // Administración
  is_active: bool
  metadata: json — información de auditoría/documentación
  created_at, updated_at
  deleted_at — soft delete
}
```

## Tipos de Acciones

### 1. REPLACE — Reemplazar

**Cuándo usar**: Una practica se debe cambiar por otra, o varias se unifican en una.

**Mecánica**:
- Si coinciden los triggers, se **elimina** cada practica que disparó
- Se **agrega** la(s) practica(s) destino con `target_quantity`

**Ejemplo**:
```
Triggers:  ORINA_COMPLETA + SEDIMENTO_URINARIO (ambos presentes)
Action:    replace
Target:    PR_ORINA_COMPLETA (cantidad 1)
Suppress:  PR_ORINA_COMPLETA + PR_SEDIMENTO_URINARIO

Antes:  [ PR_ORINA_COMPLETA, PR_SEDIMENTO_URINARIO ]
Después: [ PR_ORINA_COMPLETA ]
```

### 2. ADD — Agregar

**Cuándo usar**: La presencia de ciertos estudios/prácticas requiere agregar una practica adicional.

**Mecánica**:
- Si coinciden los triggers, se **agrega** la practica destino sin eliminar nada

**Ejemplo**:
```
Triggers:  HEMOGRAMA
Action:    add
Target:    PR_RECUENTO_PLAQUETAS (cantidad 1)

Antes:  [ PR_HEMOGLOBINA, PR_HEMATOCRITO ]
Después: [ PR_HEMOGLOBINA, PR_HEMATOCRITO, PR_RECUENTO_PLAQUETAS ]
```

### 3. SUPPRESS — Suprimir

**Cuándo usar**: Ciertos estudios/prácticas no deben ser facturadas bajo ciertas condiciones.

**Mecánica**:
- Si coinciden los triggers, se **elimina** la(s) practica(s) en `suppress_practice_type_ids`
- Si no hay lista de supresión explícita, usa `trigger_practice_type_ids`

**Ejemplo**:
```
Triggers:  GLUCOSA + HEMOGLOBINA_GLICADA (ambos presentes)
Action:    suppress
Suppress:  PR_GLUCOSA

Antes:  [ PR_GLUCOSA, PR_HEMOGLOBINA_GLICADA ]
Después: [ PR_HEMOGLOBINA_GLICADA ]
```

## Lógica de Disparos (Triggers)

### Matching de Triggers

Una regla se dispara si se cumplen **ambas** condiciones:
1. **Study match**: Al menos uno (o todos, según `requires_all_triggers`) de `trigger_study_type_ids` está presente en la orden
2. **Practice match**: Al menos uno (o todos) de `trigger_practice_type_ids` está presente en prácticas actuales

### Lógica AND vs OR

#### `requires_all_triggers = true` (AND)

**ALL triggers deben coincidir**

```
triggers: [GLUCOSA_ID, HEMOGLOBINA_GLICADA_ID]
requires_all_triggers: true

Orden con [ GLUCOSA + HEMOGLOBINA_GLICADA ] → DISPARA ✓
Orden con [ GLUCOSA ]                      → NO dispara ✗
Orden con [ HEMOGLOBINA_GLICADA ]          → NO dispara ✗
```

#### `requires_all_triggers = false` (OR)

**AL MENOS UNO de los triggers debe coincidir**

```
triggers: [GLUCOSA_ID, HEMOGLOBINA_GLICADA_ID]
requires_all_triggers: false

Orden con [ GLUCOSA + HEMOGLOBINA_GLICADA ] → DISPARA ✓
Orden con [ GLUCOSA ]                      → DISPARA ✓
Orden con [ HEMOGLOBINA_GLICADA ]          → DISPARA ✓
```

## Scoping — Cuándo se Aplica una Regla

Una regla se aplica solo si se cumplen **todas** estas condiciones:

### 1. Vigencia (fecha)
```php
valid_from ≤ HOY ≤ valid_through
// ambos null = siempre vigente
```

### 2. Activo
```php
is_active = true
```

### 3. Cobertura (insurance_type / insurance_plan)

**Lógica**: Si la regla tiene asignadas coberturas/planes:
- La orden DEBE tener ese `insurance_type_id` O ese `insurance_plan_id`

**Reglas globales**: Si la regla **no tiene** ninguna cobertura asignada:
- Aplica a **todas** las órdenes (sin restricción de cobertura)

**Ejemplo**:
```
Regla A: insurance_type_ids = [OSDE]
         → Aplica solo a órdenes con OSDE

Regla B: insurance_type_ids = [OSDE, SWISS]
         → Aplica a órdenes con OSDE o SWISS

Regla C: insurance_type_ids = [] (vacío)
         → Aplica a todas las órdenes
```

### 4. Convenio (opcional)
```php
insurance_pricing_agreement_id: int|null
// si está informado, regla aplica solo a ese convenio específico
```

## Flujo de Evaluación (Pseudocódigo)

```php
// 1. Resolución de reglas aplicables
rules = BillingRule.query()
  .where('is_active', true)
  .where(vigencia válida)
  .where(cobertura coincide)  ← filtra por insurance_type/plan
  .orderBy('priority', 'asc')
  .orderBy('id', 'asc')
  .get()

currentPractices = basePractices

// 2. Aplicar cada regla en orden
for each rule in rules:
  studyIds = order.studies.map(s => s.study_type_id)
  practiceIds = currentPractices.map(p => p.practice_type_id)
  
  if rule.matches(studyIds, practiceIds):
    currentPractices = rule.apply(currentPractices)
    
    log: {
      rule_id: rule.id,
      code: rule.code,
      action: rule.action,
      applied: true,
      changes: {
        added: N,
        removed: M,
        affected_practice_ids: [...]
      }
    }
  else:
    log: {
      rule_id: rule.id,
      code: rule.code,
      applied: false
    }

// 3. Consolidación (suma cantidades de prácticas repetidas)
consolidatedPractices = merge(currentPractices)

// 4. Devolver resultado
return {
  practices: consolidatedPractices,
  trace: rules_evaluation_log
}
```

## Contexto de Evaluación

El motor recibe un contexto que define el escenario:

```php
$ruleContext = [
  'provider_id' => int|null,          // proveedor de valorización
  'insurance_type_id' => int|null,    // cobertura
  'insurance_plan_id' => int|null,    // plan específico
  // Futuro (no implementado aún):
  // 'order_origin_id' => int|null,
  // 'order_service_id' => int|null,
  // 'order_is_private' => bool,
]
```

## Ubicación en el Código

### Core Engine
- **Clase**: `App\Services\Valuation\BillingRuleEngine`
- **Método principal**: `evaluate(array $practices, array $context): array`
- **Archivo**: `lis-backend/app/Services/Valuation/BillingRuleEngine.php`

### Invocación
- **Servicio**: `App\Services\Valuation\AdmissionValuationService`
- **Línea**: 71 (aprox)
- **Archivo**: `lis-backend/app/Services/Valuation/AdmissionValuationService.php`

```php
$ruleContext = [
  'provider_id' => $insuranceType?->valuation_provider_id,
  'insurance_type_id' => $insuranceContext['insurance_type_id'],
  'insurance_plan_id' => $insuranceContext['insurance_plan_id'],
];

$ruleResult = $this->billingRuleEngine->evaluate($basePractices, $ruleContext);
$practicesAfterRules = $ruleResult['practices'];
$ruleTrace = $ruleResult['trace'];
```

### Modelo
- **Clase**: `App\Models\BillingRule`
- **Archivo**: `lis-backend/app/Models/BillingRule.php`
- **Tabla**: `billing_rules`

### API Admin
- **Controller**: `App\Http\Controllers\Api\V1\Admin\SettingsCatalogController`
- **Endpoints**: CRUD vía catálogos genéricos
  - GET `/api/v1/admin/settings/catalogs/billing-rules`
  - POST `/api/v1/admin/settings/catalogs/billing-rules`
  - PATCH `/api/v1/admin/settings/catalogs/billing-rules/{id}`
  - DELETE `/api/v1/admin/settings/catalogs/billing-rules/{id}`

### UI
- **Componente**: `SettingsCatalogsClient`
- **Archivo**: `lis-front-monorepo/apps/lis/src/components/settings-catalogs-client.tsx`
- **Ruta**: `/settings/catalogs/facturacion`

## Ejemplos de Uso

### Ejemplo 1: Consolidar Orinas

**Requisito**: Si una orden tiene tanto ORINA_COMPLETA como SEDIMENTO_URINARIO, facturar solo ORINA_COMPLETA (que incluye ambos).

**Regla**:
```
Code: ORINA_CONSOLIDATE
Name: Unificar orina completa y sedimento
Action: replace
Priority: 30
Triggers (Study): [ORINA_COMPLETA, SEDIMENTO_URINARIO]
Requires All: true
Target: PR_ORINA_COMPLETA (qty 1)
Suppress: [PR_ORINA_COMPLETA, PR_SEDIMENTO_URINARIO]
Is Active: true
Metadata: {
  "test_case": "ORINA_COMPLETA + SEDIMENTO_URINARIO => deja PR_ORINA_COMPLETA"
}
```

### Ejemplo 2: Suprimir Glucosa cuando hay HbA1c

**Requisito**: Con OSDE/Swiss Medical, si se solicitan GLUCOSA y HEMOGLOBINA_GLICADA, facturar solo HEMOGLOBINA_GLICADA (que es más precisa).

**Regla**:
```
Code: GLUCOSE_SUPPRESS_HBA1C
Name: Suprimir glucosa puntual si hay hemoglobina glicada
Action: suppress
Priority: 20
Insurance Types: [OSDE, SWISS_MEDICAL]
Triggers (Study): [GLUCOSA, HEMOGLOBINA_GLICADA]
Requires All: true
Suppress: [PR_GLUCOSA]
Valid From: 2026-01-01
Valid Through: 2026-12-31
Is Active: true
Metadata: {
  "seed": "AdmissionValuationSeeder",
  "test_case": "GLUCOSA + HEMOGLOBINA_GLICADA => suprime PR_GLUCOSA"
}
```

### Ejemplo 3: Agregar Estudio Automatizado

**Requisito**: Cuando se pide HEMOGRAMA, siempre agregar RECUENTO_PLAQUETAS (es parte estándar).

**Regla**:
```
Code: HEMOGRAM_ADD_PLATELETS
Name: Agregar recuento de plaquetas al hemograma
Action: add
Priority: 10
Triggers (Study): [HEMOGRAMA]
Requires All: true
Target: PR_RECUENTO_PLAQUETAS (qty 1)
Is Active: true
Metadata: {
  "test_case": "HEMOGRAMA => agrega PR_RECUENTO_PLAQUETAS"
}
```

## Métodos Principales del Engine

### `evaluate(array $practices, array $context): array`

**Entrada**:
- `$practices`: Array de prácticas mapeadas con estructura:
  ```php
  [
    [
      'study_type_id' => int,
      'practice_type_id' => int,
      'quantity' => int,
      'study_index' => int,
      // ... otros campos
    ],
    // ...
  ]
  ```
- `$context`: Array de contexto (ver "Contexto de Evaluación")

**Salida**:
```php
[
  'practices' => [ /* prácticas transformadas */ ],
  'trace' => [
    [
      'rule_id' => int,
      'code' => string,
      'action' => string,
      'priority' => int,
      'applied' => bool,
      'changes' => [
        'added' => int,
        'removed' => int,
        'affected_practice_type_ids' => [int, ...]
      ]
    ],
    // ... para cada regla evaluada
  ]
]
```

### `resolveApplicableRules(array $context): Collection`

**Resolución de reglas aplicables** filtrando por:
- `is_active = true`
- Vigencia (fechas)
- Cobertura (insurance_type/plan)
- Ordenadas por prioridad

### `ruleMatches($rule, $availableStudyIds, $availablePracticeIds): bool`

**Determina si los triggers de una regla coinciden** con:
- Estudios presentes en la orden
- Prácticas presentes actualmente

Usa `requires_all_triggers` para decidir lógica AND/OR.

### `applyRule($practices, $rule, $targetPracticeMeta): array`

**Aplica la acción de la regla** a la lista actual de prácticas.

Devuelve:
```php
[
  'practices' => [ /* prácticas post-acción */ ],
  'added' => int,
  'removed' => int,
  'affected_practice_type_ids' => [int, ...]
]
```

## Tracer/Auditoria

Cada evaluación devuelve un `trace` con detalles de todas las reglas procesadas:

```php
[
  [
    'rule_id' => 42,
    'code' => 'GLUCOSE_SUPPRESS_HBA1C',
    'action' => 'suppress',
    'priority' => 20,
    'applied' => true,  // Se disparó y se aplicó
    'changes' => [
      'added' => 0,
      'removed' => 1,
      'affected_practice_type_ids' => [15]  // Eliminó PR_GLUCOSA (id 15)
    ]
  ],
  [
    'rule_id' => 10,
    'code' => 'HEMOGRAM_ADD_PLATELETS',
    'action' => 'add',
    'priority' => 10,
    'applied' => false,  // Triggers no coincidieron
    'changes' => []
  ],
  // ... más reglas
]
```

**Nota**: El trace se devuelve en la respuesta de API pero **no se persiste actualmente en base de datos** (future enhancement).

## Limitaciones Actuales (2026-08-13)

1. ❌ **Sin preview**: No hay endpoint para testear una regla sin guardarla
2. ❌ **Sin validación de loops**: Reglas que se auto-replican no se detectan
3. ❌ **Sin análisis de cobertura**: No hay reportes de "qué reglas aplican a qué orden"
4. ❌ **Metadata no interpretada**: Campo JSON sin lógica de negocio
5. ❌ **Sin versioning**: No se registra historial de cambios
6. ❌ **Trace no persistente**: Log de ejecución se devuelve pero no guarda en BD
7. ❌ **Scoping limitado**: Solo por cobertura/plan; no por origen/servicio/etc.
8. ❌ **UI muy básica**: Solo tabla CRUD; falta visualización de impacto

## Mejoras Planeadas

### Próximo Sprint
- Remover relación `provider_id` (no debe asociarse a prestador)
- Extender contexto: `order_origin_id`, `order_service_id`, `order_is_private`
- Agregar campos a rule: `trigger_origin_ids`, `trigger_service_ids`, `trigger_is_private`

### Roadmap
- Endpoint de preview: `POST /api/v1/admin/billing-rules/{id}/preview`
- Validación de circularidad antes de guardar
- Persistencia de trace en tabla `billing_rule_execution_logs`
- UI avanzada: visualización de dependencias y impacto
- Reportes: cobertura de reglas por study_type

## Referencias

- **API Funcional**: [admission-valuations-api.md](admission-valuations-api.md)
- **Proceso de Valorización**: [valuation-pricing-process.md](valuation-pricing-process.md)
- **Código del Engine**: `lis-backend/app/Services/Valuation/BillingRuleEngine.php`
- **Modelo**: `lis-backend/app/Models/BillingRule.php`
- **Tests**: `lis-backend/tests/Feature/ValuationRuleTestingSeeder.php`
