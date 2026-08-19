# LIS — Futuras Mejoras

Registro vivo de mejoras identificadas durante el desarrollo, agrupadas por módulo. Cada ítem indica **dónde va el cambio** (repo + archivo/servicio), estimación aproximada y prioridad sugerida.

Prioridades: **P1** crítico antes de producción · **P2** importante · **P3** nice-to-have.

---

## Billing Rules Engine

Contexto: motor de reglas de facturación (`lis-rules-engine`) consumido por el backend Laravel para transformar prácticas antes de valuarlas. Refactor "Option A pura" (2026-08-18/19) consolidó todo el scope de reglas dentro de `trigger_json`.

### BR-1 · Feature flag efectivo en el backend · **P1**
- **Dónde**: `lis-backend/app/Services/Valuation/BillingRuleEngineHttpClient.php`, `lis-backend/app/Services/Valuation/AdmissionValuationService.php`.
- **Qué**: hoy `LIS_RULES_ENGINE_ENABLED` existe en `config/services.php` pero no se consulta en el flujo de `evaluate()`. Solo se lanza excepción si faltan `base_url` o `internal_token`.
- **Cómo**: si el flag es `false`, saltar la llamada HTTP y devolver `['practices' => $basePractices, 'trace' => []]` desde `BillingRuleEngineHttpClient::evaluate()` (o directamente cortocircuitar desde `AdmissionValuationService`). Permite rollback inmediato si el motor tiene un problema en prod.
- **Estimado**: 30 min.

### BR-2 · Modo shadow (dual-run + diff logging) · **P2**
- **Dónde**: `lis-backend/app/Services/Valuation/AdmissionValuationService.php`.
- **Qué**: correr el motor de reglas y a la vez la valuación legacy (sin reglas), comparar prácticas resultantes y loguear diferencias, pero devolver el resultado legacy. Sirve para ganar confianza antes de activar reglas en producción real.
- **Cómo**: nuevo flag `LIS_RULES_ENGINE_MODE=shadow|active`; en shadow se ejecutan ambas ramas dentro de un try/catch y se emite un log estructurado con `rule_ids_applied`, `practices_diff` (added/removed/quantity_changed). Guardar métricas en `App\Services\OperationLog` o dedicado.
- **Estimado**: 2-3 h.

### BR-3 · Nested groups en el trigger builder · **P3**
- **Dónde**: `lis-front-monorepo/apps/lis/src/components/rule-trigger-builder.tsx`.
- **Qué**: hoy el builder solo maneja un nivel plano de `all` o `any`. Reglas del tipo *"osde AND (particular OR convenio X)"* requieren anidar grupos.
- **Cómo**: modelo recursivo `RuleGroup = { all: RuleGroup[] } | { any: RuleGroup[] } | RuleCondition`. Renderizado con indentación por profundidad y botón "Agregar subgrupo" además de "Agregar condición". Validar que el backend `RuleRequest::validateTriggerShape()` acepte estructura recursiva (probablemente ya lo hace porque solo revisa el primer nivel).
- **Estimado**: 4-6 h.

### BR-4 · Dry-run / simulación desde el ABM · **P2**
- **Dónde**: `lis-front-monorepo/apps/lis/src/components/settings-billing-rules-client.tsx` (UI) + `lis-rules-engine/src/modules/billing/billing.controller.ts` (endpoint dedicado si hace falta).
- **Qué**: botón "Probar regla" con un panel de inputs para el contexto de ejemplo (cobertura, plan, prácticas iniciales) y muestra si la regla dispararía, con qué acciones y qué prácticas quedarían.
- **Cómo**: reusar `POST /api/v1/billing/evaluate` limitando el catálogo a la regla que se está editando (o filtrando trace por `ruleId`). Renderizar la salida del `trace` de forma legible.
- **Estimado**: 3-4 h.

### BR-5 · Buscador y filtros en la lista de reglas · **P2**
- **Dónde**: `lis-front-monorepo/apps/lis/src/components/settings-billing-rules-client.tsx`.
- **Qué**: `TextInput` de búsqueda por code/name y `Chip.Group` para filtrar por activa/inactiva. Hoy se listan las 55 reglas sin filtro; va a crecer.
- **Cómo**: filtrado client-side sobre `rules` con `useMemo`; opcional pasar `q` al backend si crece mucho (`rulesApi.list({ q })`).
- **Estimado**: 1 h.

### BR-6 · Persistir orden y prioridad visual · **P3**
- **Dónde**: `lis-front-monorepo/apps/lis/src/components/settings-billing-rules-client.tsx`.
- **Qué**: hoy se muestra la prioridad como número pero no hay manera de reordenar visualmente. Un drag-and-drop en la lista o botones ↑↓ que actualicen `priority` en bloque mejoraría la UX.
- **Cómo**: `@dnd-kit/sortable` sobre la tabla; endpoint bulk `PATCH /api/v1/admin/rules/reorder` con `[{id, priority}]`. Reasignar priorities en pasos de 10 para dejar aire.
- **Estimado**: 3-4 h.

### BR-7 · Duplicar / clonar regla · **P3**
- **Dónde**: `lis-front-monorepo/apps/lis/src/components/settings-billing-rules-client.tsx`.
- **Qué**: botón "Duplicar" en cada fila que pre-cargue el form con el mismo trigger/actions y `code` en blanco.
- **Cómo**: `handleDuplicate(rule) → setSelectedId(null); setForm({ ...ruleToFormState(rule), code: "" })`.
- **Estimado**: 30 min.

### BR-8 · Timeline / historial de cambios por regla · **P3**
- **Dónde**: backend (`lis-backend/app/Models/Rule.php` + observer) + frontend.
- **Qué**: guardar snapshots de la regla en cada save para auditoría. Ver historial expandible en el ABM.
- **Cómo**: opción A) `spatie/laravel-activitylog` con `LogsActivity` sobre `Rule`. Opción B) tabla propia `rule_revisions` con `rule_id, trigger_json, actions_json, changed_by, changed_at`. Frontend agrega tab "Historial" en el detalle.
- **Estimado**: 4-6 h.

### BR-9 · Actualizar nota de sesión de arquitectura · **P3**
- **Dónde**: `/memories/session/billing-rules-architecture.md` (memoria local del asistente).
- **Qué**: refleja el modelo pre-Option A (pivots + FK). Reescribir con la arquitectura vigente para evitar confusión en futuras sesiones.
- **Estimado**: 15 min.

---

## Valuation / Admission

### VA-1 · Cache de contexto de cobertura por request · **P3**
- **Dónde**: `lis-backend/app/Services/Valuation/AdmissionValuationService.php`.
- **Qué**: `resolveInsuranceContext` y las cargas de `InsuranceType`/`InsurancePlan` con `billingIndications` se ejecutan por cada quote; en flujos multi-quote (ej. re-cotización) hay hits redundantes.
- **Cómo**: memoize por `insurance_type_id + plan_id + agreement_id` dentro del request lifecycle.
- **Estimado**: 1 h.

---

## Frontend — General

### FE-1 · Estandarizar `extractServerError` en un util · **P2**
- **Dónde**: `lis-front-monorepo/packages/api-client/src/` o `apps/lis/src/services/http/`.
- **Qué**: la función `extractServerError` de `settings-billing-rules-client.tsx` que arma un mensaje legible a partir de `HttpError.data.errors` (validación Laravel 422) es útil en cualquier form. Hoy está inlineada en un componente.
- **Cómo**: mover a `@arenco/api-client` como `formatValidationError(err: unknown): string` y adoptarla en las mutaciones existentes.
- **Estimado**: 1 h + revisión de callers.

### FE-2 · Reusar `CatalogAutocomplete` / `CatalogTagsInput` · **P3**
- **Dónde**: `lis-front-monorepo/apps/lis/src/components/rule-trigger-builder.tsx` (donde viven hoy, locales al archivo) → mover a un módulo compartido.
- **Qué**: los componentes que hacen autocomplete/tags con `settingsCatalogsApi.listCatalogItems` son genéricos. También los usa `rule-actions-builder.tsx` con su propio `PracticeCodeInput` duplicado.
- **Cómo**: extraer a `apps/lis/src/components/catalog-inputs.tsx` con props `{ catalog: SettingsCatalogKey, ... }`. Refactorizar `PracticeCodeInput` a `CatalogAutocomplete`.
- **Estimado**: 1-2 h.

---

## Observabilidad / DX

### OBS-1 · Métricas por regla aplicada · **P3**
- **Dónde**: `lis-rules-engine/src/application/billing/billing-service.ts` + observabilidad existente.
- **Qué**: contadores por `ruleId` de cuántas veces disparó, cuántas veces no matcheó y tiempo total. Útil para detectar reglas huérfanas.
- **Cómo**: agregar métricas Prometheus (o el sistema que ya use el engine) desde el trace loop.
- **Estimado**: 2 h.

---

## Convenciones para nuevas entradas

Al agregar un ítem seguir el formato:

```
### <SIGLA>-N · <título corto> · **P1|P2|P3**
- **Dónde**: repo + archivo/servicio específico.
- **Qué**: descripción del problema o gap actual.
- **Cómo**: enfoque propuesto (alto nivel, opcional pero recomendado).
- **Estimado**: rango horas/días.
```

Siglas por módulo:
- `BR` — Billing Rules
- `VA` — Valuation / Admission
- `FE` — Frontend general
- `OBS` — Observabilidad / DX
- (agregar según haga falta: `PAT` pacientes, `Q` queueing, `AUTH`, etc.)
