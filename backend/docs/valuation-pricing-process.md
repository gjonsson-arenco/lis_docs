# Proceso de valorizacion de practicas (detalle tecnico)

## Objetivo
Documentar como el backend obtiene precios de practicas, donde se registran convenios, como decide la lista para particular, y como aplica listas diferenciales por excepcion.

## Flujo end-to-end actual
1. Entrada: `POST /api/v1/admission-valuations/quote`.
2. Mapeo: estudios -> practicas base (tabla pivote `study_practice_type`).
3. Reglas de facturacion: `replace`, `add`, `suppress` por prioridad (tabla `billing_rules`).
4. Consolidacion: merge de practicas repetidas (suma cantidad).
5. Valorizacion:
   - `manual`: busca convenio activo y calcula precios por nomenclador.
   - `api`: delega al proveedor remoto.

## De donde salen los precios
Los precios unitarios base salen de `nomenclator_practice_prices`:
- clave: `(nomenclator_id, practice_type_id)`
- campos principales:
  - `base_value`
   - `coverage`
  - `patient_value`
  - `requires_authorization`

En valorizacion manual:
1. Se resuelve un convenio (`insurance_pricing_agreements`).
2. Se toma `nomenclator_id` (lista base).
3. Si el convenio define `exception_nomenclator_id`, esa lista se consulta tambien.
4. Para cada practica:
   - primero intenta precio en la lista de excepcion,
   - si no existe, usa la lista base.
5. `unit_price` depende del tipo de convenio:
   - `LIST`: `unit_price = base_value`
   - `UNITS`: `unit_price = base_value * unit_value`
6. `total_price = unit_price * quantity`.
7. Cobertura y paciente:
   - `coverage` representa el porcentaje de cobertura del financiador.
   - `patient_amount` usa `patient_value` si existe; si no, diferencia contra cobertura.

## Donde se registran los convenios
Catalogo admin:
- `/api/v1/admin/settings/catalogs/insurance-pricing-agreements`

Relaciones del convenio:
- `provider_id`: prestador de valorizacion
- `is_private_agreement`: convenio para orden particular
- `insurance_type_id` (opcional)
- `insurance_plan_id` (opcional)
- `nomenclator_id` (obligatorio)
- `exception_nomenclator_id` (opcional, lista diferencial)
- `type` (`LIST` | `UNITS`)
- `unit_value` (obligatorio en `UNITS`)
- `valid_from`, `valid_through`
- `is_active`

## Como decide que convenio usar
Busqueda por orden de especificidad y vigencia:
1. mismo `provider_id`
2. `is_active=true`
3. fecha actual dentro de vigencia
4. matching por cobertura:
   - plan+tipo especificos tienen mayor prioridad,
   - luego tipo,
   - luego convenio global (`NULL/NULL`).

Si no encuentra convenio activo, la practica queda `unresolved` con razon `no_active_agreement`.

## Como decide la lista si es particular
Comportamiento actual:
1. Si `order.is_private=true` siempre entra por `manual`.
2. Provider usado en manual:
   - si no hay provider por insurance_type (o no aplica), toma el primer provider manual activo.
3. Si `order.insurance_type_id` y `order.insurance_plan_id` vienen `null`, la resolucion de convenio cae en alcance global (`insurance_type_id` y `insurance_plan_id` nulos).

Implicancia:
- Para particular sin insurance, la lista depende del convenio global configurado para ese provider manual.

## Listas diferenciales por excepcion
Soporte implementado via `exception_nomenclator_id` en convenio.

Regla aplicada por practica:
- usar precio de excepcion si existe para esa practica,
- sino usar precio de nomenclador base.

Esto permite excepciones puntuales sin duplicar toda la lista base.

## Donde configurar cada cosa
1. Providers valorizacion:
- `/api/v1/admin/settings/catalogs/valuation-providers`

2. Nomencladores:
- `/api/v1/admin/settings/catalogs/nomenclators`

3. Precios por practica:
- `/api/v1/admin/settings/catalogs/nomenclator-practice-prices`

4. Convenios:
- `/api/v1/admin/settings/catalogs/insurance-pricing-agreements`

5. Reglas de facturacion:
- `/api/v1/admin/settings/catalogs/billing-rules`

6. Asignacion provider por insurance type (para API routing):
- `/api/v1/admin/settings/catalogs/insurance-types`
- campo: `valuation_provider_id`

## Ajustes recomendados
1. Provider manual explicito para particular:
- evitar fallback al "primer provider manual".
- sugerido: flag dedicado o regla de routing explicita para `is_private=true`.

2. Convenio particular dedicado:
- crear convenio global (sin tipo/plan) para particular y documentarlo como obligatorio.

3. Validaciones de integridad:
- si una insurance tiene provider API, validar que provider tenga `api_endpoint`.
- impedir convenios superpuestos con misma especificidad y vigencia.

4. Observabilidad:
- registrar en logs el convenio seleccionado (`agreement_id`) y nomenclador usado por practica (base vs excepcion).

## Referencias de implementacion
- Orquestacion general: `app/Services/Valuation/AdmissionValuationService.php`
- Pricing manual/API y resolucion de convenio: `app/Services/Valuation/PracticeValuationService.php`
- Modelo convenio: `app/Models/InsurancePricingAgreement.php`
- Precios por practica: `app/Models/NomenclatorPracticePrice.php`
- Catalogos de consulta: `app/Http/Controllers/Api/V1/AdmissionValuationCatalogController.php`
- Doc API funcional: `docs/admission-valuations-api.md`
