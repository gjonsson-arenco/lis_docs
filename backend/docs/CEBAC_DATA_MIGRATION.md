# Migración de datos legacy de CEBAC

Referencia de todo lo hecho para migrar los datos del LIS legacy de CEBAC
("Labcore", SQL Server) al LIS nuevo, vía exports CSV (no hay bridge en vivo
a la base vieja). Pensado para arrancar un hilo nuevo sin perder contexto —
no repite explicaciones de arquitectura general del LIS, asume que ya se leyó
el código de los seeders si hace falta el detalle exacto.

## Dónde está todo

| Qué | Dónde |
|---|---|
| CSVs legacy (+ sus `*Fields.csv` con el schema real de SQL Server) | `lis-backend/database/seeders/data/cebac-legacy/` |
| Orquestador (tenant + admins + llama a los 3 de abajo) | `database/seeders/CebacSeeder.php` |
| Catálogos generales | `database/seeders/CebacCatalogSeeder.php` |
| Facturación/pricing | `database/seeders/CebacBillingCatalogSeeder.php` |
| Reglas de facturación (motor nuevo) | `database/seeders/CebacRuleCatalogSeeder.php` |
| Requisitos de estudios ("indicaciones") | `database/seeders/CebacRequirementCatalogSeeder.php` |

Se corre todo con:

```bash
php artisan migrate --force
php artisan db:seed --class=CebacSeeder --force
```

Todos los seeders son idempotentes (`updateOrCreate` por `code` resuelto,
ver más abajo) salvo `CebacBillingCatalogSeeder::importInsurancePricingAgreements()`,
que borra (`forceDelete`) y re-crea todo `insurance_pricing_agreements` en
cada corrida porque esa tabla no tiene una key natural para upsert.

## Principios de diseño que se repiten en los 3 seeders

- **El id autoincremental del CSV (primera columna) nunca se guarda tal cual
  como `code`.** Se usa solo como key interna en mapas PHP en memoria, para
  saber si dos filas son "la misma" entidad (ej. una prueba/parámetro
  reusada en varios estudios) y para cruzar entre archivos (`ei_est_id` de
  `EstudiosIndicaciones.csv` contra el `e_id` de `EstudiosPruebas.csv`,
  etc).
- **`resolveCode()`** (repetido igual en los 3 seeders): usa el código de
  negocio real de la fila legacy (`e_codigo` y equivalentes) como `code` de
  destino; si no hay, cae al id autoincremental; si dos ids legacy
  distintos generan el mismo código, el segundo se desambigua con sufijo
  `-{legacyId}`. La comparación de colisión es **case-insensitive**
  (`mb_strtoupper`) porque la columna `code` usa collation
  `utf8mb4_unicode_ci` — sin eso, "BIO" y "bio" (dos altas reales de
  BIOARS en `CoberturaMedica.csv`) se fusionaban solas sin que el chequeo
  en PHP lo detectara.
- **`-1` / "Todos"** es un sentinel de agregado que aparece repetido en
  varios catálogos legacy (`TipoPaciente`, `Centros`, `GrupoCoberturaMedica`,
  `CoberturaMedica`, `FacturacionReglas`...) — siempre se descarta, no es
  una fila real.
- **La mayoría de los CSV NO tienen fila de encabezado** (export crudo de
  SQL Server) — excepciones confirmadas: `TipoPaciente.csv`,
  `Indicaciones.csv`, `EstudiosIndicaciones.csv`. Cada `readCsv()` tiene un
  parámetro `skipHeader` para esto — **si un CSV nuevo trae header y se
  olvida el flag, la fila de encabezado se importa como si fuera un dato
  real** (pasó con `TipoPaciente.csv`, se importó una fila
  `code=tp_codigo, name=tp_nombre`).
- **Los nombres de columna se confirmaron con los `*Fields.csv`** (export
  de metadata de SQL Server: `tabla;columna;tipo;...;ordinal_position`).
  La inferencia por cruce de rangos/unicidad/repetición contra otros
  catálogos acertó la mayoría de las veces, pero tuvo errores reales en
  `FacturacionConvenios` (dos columnas cruzadas), `Medicos` (no se
  encontraba el flag de activo) y `Tarifas`/`TarifasDet`/`FacturacionReglas`
  (varias columnas secundarias mal mapeadas) — ver la lista de bugs abajo.
  Moraleja: pedir los `*Fields.csv` de entrada la próxima vez que aparezca
  un catálogo nuevo, no inferir si se puede evitar.

## CebacSeeder.php

Tenant `cebac` (single-tenant: `db_host`/`db_database`/etc de la fila del
tenant apuntan explícitamente a las mismas env vars `DB_*` que usa la
conexión default, no al default baked-in de la migración que apuntaba al
SQL Server viejo). 3 admins: `gjonsson@arenco-it.com`,
`cebac3@gmail.com` (Marcelo Zanek), `matias.rios@arenco-it.com`.

## CebacCatalogSeeder.php

| Legacy CSV | Modelo destino | Notas |
|---|---|---|
| — (fijo, sin CSV) | `IdentificationType` | DNI, PAS, NN |
| `TipoPaciente.csv` (con header) | `OrderType` | descarta `tp_id=-1` |
| `GruposTrabajo.csv` | `WorkingArea` | |
| `Metodologias.csv` | `MethodType` | |
| `TiposMuestra.csv` | `SampleType` | code = columna 2, no la 1 (esa es el nombre) |
| `Instalaciones.csv` | `Site` | id `0` = CEBAC misma → `is_external=false` |
| `Centros.csv` | `Origin` | descarta `-1` y `13` (CENTRO PRUEBA); linkeado a `Site` por columna 15 |
| `Servicios.csv` | `Service` | |
| `GrupoCoberturaMedica.csv` | `InsuranceGroup` | |
| `CoberturaMedica.csv` | `InsuranceType` | grupo por `cm_gru_id`; `requires_voucher`/`is_active` **NO** vienen de `cm_derivante`/`cm_apb` — esas columnas no significan eso (ver bugs) |
| `Medicos.csv` | `Physician` | **todos** los médicos, activos y no; `m_activo` se conserva en `is_active` (417 activos / 3.607 inactivos) |
| `EstudiosPruebas.csv` | `StudyType` + `TestType` + `ReportingType` | el archivo más grande y complejo; columnas confirmadas 1:1 contra `EstudiosPruebasFields.csv` (`e_id, e_codigo, e_cod_alterno, e_nombre, e_abreviatura, e_proceso_dias, e_proceso_tiempo, g_id/g_codigo/g_nombre, mt_id/mt_codigo/mt_descripcion, p_id/p_codigo/p_nombre, tm_id/tm_codigo/tm_nombre/tm_sufijo`) |
| `Routines with headers.txt` | `StudyType` (bundle) | perfiles/rutinas: cada perfil es un `StudyType` con `parent_study_type_ids` listando los ids de los `StudyType` componentes. Ancla en `e_id` (perfil) y `child_study_id` (componente); el `WorkingArea` sale de `e_gru_id`, con el area "PERFILES" solo como fallback del centinela `-1`. Reemplaza a `Perfiles.csv` (borrado): tenia 31 de estos mismos 45 perfiles, con listas de componentes identicas. Ningun perfil aparece en `EstudiosPruebas.csv` — no tienen parametros, asi que el join los descarta — por eso el archivo no trae `e_activo` y `is_active` se infiere de la convencion de nombres (`_I`/`_IN`, prefijo `INACTIVO_`) |

`InsurancePlan` queda **intencionalmente vacía** — el legacy no tiene un
tercer nivel debajo de `InsuranceType`.

## CebacBillingCatalogSeeder.php

Corre después del de catálogos (necesita `StudyType`/`TestType`/
`InsuranceType` ya creados). Re-deriva mapas legacy-id→modelo repitiendo la
resolución de código de `EstudiosPruebas.csv`/`CoberturaMedica.csv` (no hay
otra forma: `StudyType.code` ya es el `e_codigo`, no el id legacy).

| Legacy CSV | Modelo destino | Notas |
|---|---|---|
| — (mirror de `StudyType`) | `PracticeType` | 1:1 con **todos** los `StudyType` (estudios reales + perfiles), vía pivot `study_practice_type` — "misma cantidad de estudios que prácticas al inicio" |
| `Tarifas.csv` | `Nomenclator` | `t_fnt_id` → `type` (1=NBU→`UNITS`, 2=INOS→`FEES`, 3=LISTA→`LIST`); `t_activo` → `is_active`. **Solo se importan los referenciados por algún convenio importado** (93 de 497) |
| `TarifasDet.csv` | `NomenclatorPracticePrice` | **forma de fila distinta según el tipo del nomenclador padre**: LIST usa `td_valor`/`td_cod_est`; UNITS usa `td_ub`; FEES usa `td_ug`→`expense_value` y `td_uh`→`fee_value`. `td_convenido`=1 → `is_agreed` + `coverage=100`; `td_observacion` con nota de autorización → `requires_authorization` |
| `FacturacionConvenios.csv` | `InsurancePricingAgreement` | `fc_cm_id`→cobertura, `fc_tar_id`→`nomenclator_id`, `fc_tar_dif_id`→`exception_nomenclator_id` (ya existía, pensado para esto), `fc_ftn_id` decide qué factor leer: 1→`fc_nbu_factor` como `unit_value`, 2→`fc_inos_ug`/`fc_inos_uh` como `expense_value`/`fee_value`, 3→ninguno. `fc_allow_hasta` (100% NULL) y `fc_suc_id` (constante) se descartan, no aportan info. **Solo vigentes + fallback** (ver abajo) |

### Vigencia y cascada de facturación

La tabla legacy guarda el historial completo de cada renegociación (8.488
filas, ~12 por cobertura), así que importarla entera cargaría una década de
pricing pisado junto al actual. El criterio:

1. **Vigentes**: `fc_hasta` abierto o no pasado, y `fc_desde` ya empezado.
   En este export son exactamente los de vigencia abierta — el `fc_hasta`
   más nuevo de todo el archivo es `2026-07-31`, ya vencido. Son 568.
2. **Fallback**: las coberturas que quedarían sin ningún convenio se llevan
   su **último convenio vencido** (el de `fc_hasta` más reciente, desempate
   por `fc_id`). Son 102, y se importan con **`valid_through` en NULL**:
   `PracticeValuationService` filtra por `valid_through >= hoy`, así que
   conservar la fecha original importaría una fila que el motor nunca puede
   usar — lo mismo que no importarla. La fecha real queda registrada en la
   `description` (`"(último convenio, vencido 2025-12-31)"`).
3. **Cascada**: los nomencladores se importan solo si algún convenio
   importado los referencia (`fc_tar_id` o `fc_tar_dif_id`), y los precios de
   `TarifasDet` caen solos al no existir su nomenclador padre.

Quedan **2 coberturas sin precio** (legacy ids 140 y 461): todos sus
convenios apuntan a tarifas que no existen en `Tarifas.csv`. No se pueden
recuperar desde este export.

`FacturacionReglas.csv` ya **no** se importa acá — pasó a
`CebacRuleCatalogSeeder` (ver más abajo), que apunta a la tabla `rules` del
motor nuevo en vez de a `billing_rules`.

## CebacRuleCatalogSeeder.php

Migra `FacturacionReglas.csv` a la tabla `rules` (el formato del motor nuevo,
documentado en [RULES_AUTHORING_BILLING.md](./RULES_AUTHORING_BILLING.md)).
Reemplaza al import a `BillingRule` que vivía en el seeder de facturación.

Corré `CEBAC_RULES_DRY_RUN=1 php artisan db:seed --class=CebacRuleCatalogSeeder`
para ver el reporte completo sin escribir nada. **Vale la pena correrlo cada
vez que llegue un export nuevo**: el reporte dice, regla por regla, qué se
descartó y por qué, y agrupa los códigos sin resolver por causa.

### Mapeo de las 4 acciones legacy a las 2 primitivas nuevas

El motor nuevo solo tiene `add_practice` y `remove_practice`:

| `fr_accion` | `actions_json` |
|---|---|
| 1 Reemplazar | `remove_practice` de cada `fr_est_cod` + `add_practice` de cada `fr_est_cod_new` (removes primero) |
| 2 Anular | `remove_practice` de cada **`fr_est_cod_new`** |
| 3 Agregar / 4 Sumar | `add_practice` de cada `fr_est_cod_new`, con `quantity = fr_est_cant_new` |

Detalles que no son obvios:

- **El trigger se arma sobre el fact `practiceCodes`, no `studyCodes`**. El
  motor recalcula `practiceCodes` por regla sobre la lista ya mutada, que es
  lo que hacía el motor viejo y de lo que dependen las cascadas reales de
  estos datos (FR_267 convierte 660911B en 660911, y FR_293/FR_297 recién
  entonces convierten 660911 en 660035+660105+660176). Como consecuencia el
  **orden importa**: `priority` se asigna 100, 110, 120... por `fr_id`
  ascendente, lo más parecido a un orden que tiene el legacy.
- `fr_est_cant_new` está en `1` en las 114 filas, pero ahora se lee de verdad
  en vez de asumirse.
- Alcance por cobertura vía `fr_cm_id` (`-1` y `NULL` = todas). El legacy no
  tiene planes ni convenios, así que `insurance_pricing_agreement_id` y el
  pivot de planes quedan vacíos.
- Idempotente por `code = FR_{fr_id}`; `metadata` guarda `legacy_id`,
  `legacy_action`, `legacy_name` y `legacy_insurance_id` para poder auditar
  después contra el CSV.

### Los códigos de las reglas son códigos de facturación, no `e_codigo`

Esto es lo que más cambia respecto del import viejo. `fr_est_cod` /
`fr_est_cod_new` son **`td_cod_est`** (la columna de código de facturación de
`TarifasDet`), no `e_codigo` de `EstudiosPruebas`. Los nombres de las reglas
lo dicen: "2- CAMBIA INFLUENZA X **COD FAC**". Coinciden en 1276 de 1604
casos, porque el código de facturación de un estudio arranca siendo su propio
`e_codigo` — pero el objetivo de la mayoría de estas reglas es justamente el
caso en que difieren.

Cada código se resuelve en dos pasos: primero contra `PracticeType.code`
directo, después vía `td_cod_est` → `td_est_id` → estudio → práctica. Si un
código de facturación mapea a **más de un** estudio (118 de 1604 lo hacen) la
regla se descarta en vez de adivinar.

### Estado de la última corrida: 55 de 114

| Motivo | Reglas |
|---|---|
| **Importadas** | **55** |
| Target sin resolver | 30 |
| Trigger sin resolver | 19 |
| Duplicado exacto de otra regla | 6 |
| No-op (`660001` → `660001`) | 3 |
| Trigger vacío (FR_329) | 1 |

Las 49 que se caen por códigos sin resolver son 36 códigos distintos, de dos
tipos bien distintos (el reporte los separa):

- **14 códigos cuyo estudio falta en el export** — `660001`, `660035`,
  `660105`, `660176`, `660193`, `660463`, `660598`, `660762`, `660948`,
  `662745`, `664775`, `668298`, `800006`, `ADM015`. Existen en `TarifasDet`
  con un `td_est_id`, pero ese estudio no está en `EstudiosPruebas.csv`. Son
  24 reglas recuperables sin tocar código en cuanto llegue un export completo
  (ver "Pendiente"); el techo pasaría de 55 a 79.
- **22 códigos externos, sin estudio detrás** — `230120`, `230122`, `230153`,
  `774607`, `775238`, `667626`, etc. Son códigos del nomenclador del
  financiador (PAMI, OSDE, NBU 2024), no prácticas nuestras. Reglas como
  "COOMBS DIR PAMI INTER" (`660184` → `230120`) no son sustituciones de
  práctica sino **el mismo estudio facturado bajo otro código para ese
  pagador**, que en el modelo nuevo es `nomenclator_practice_prices.billing_code`
  por convenio, no una regla. No son migrables tal cual y no deberían serlo.
  (`99999`/`1111`, las 7 filas placeholder, caen también acá.)

### Conflictos de orden

El reporte lista 16 casos donde una regla quita un código que otra posterior
necesita para disparar, o sea que la segunda nunca va a correr (ej. FR_453
"LACTICO Y EAB ART" y FR_454 "LACTICO Y EAB VEN", ambas sobre la cobertura
145 y ambas disparando con `660592`). El motor viejo se comportaba igual, así
que la migración es fiel — pero son reglas que en la práctica están muertas y
convendría que negocio las revise.

## CebacRequirementCatalogSeeder.php

Corre después del de catálogos (necesita `StudyType`/`TestType`).

| Legacy CSV | Modelo destino | Notas |
|---|---|---|
| `Indicaciones.csv` (con header) | `RequirementType` | `ind_encuesta` (XML) → `type=data`; `<tipo>` selecciona el layout: T=text, B=bool, L=list (de `<opciones>`), D/DH=date. `ind_archivo` (sin XML) → `type=printable`, bajo una carpeta nueva `requirements/` en el disk de documentos (paralela a `documents/` de `DocumentService`) — los PDF en sí se suben aparte. `<pruebas><prueba>` → `linked_test_type_ids` dentro del `layout` json (el requisito sigue atado al `StudyType` entero; esto es solo para saber a qué `TestType` corresponde pegar la respuesta más adelante) |
| `EstudiosIndicaciones.csv` (con header) | pivot `study_requirement_type` | **¡Ojo con exports parciales!** — el primer export vino con "Select Top 1000 Rows" de SSMS (1000 de 1667 filas reales), y varios estudios se veían sin requisitos por eso, no por un bug de mapeo. Reexportar con "Export Data" completo si un estudio conocido aparece con menos indicaciones de las esperadas |

3 filas de `Indicaciones.csv` tenían las opciones de respuesta escritas
adentro del label de texto libre en vez de en `<opciones>` (cada una con un
delimitador distinto) — corregidas a mano vía
`LABEL_OPTIONS_OVERRIDES` en el seeder, no con regex.

`ind_hijas` (indicaciones que dependen de otras) es una columna real del
schema legacy pero está **vacía en el 100% de las filas de este export** —
no hay nada que migrar ahí todavía; si en algún momento aparece con datos,
hay que diseñar cómo se modela (hoy `RequirementType` no tiene ningún
concepto de dependencia entre requisitos).

## Cambios de schema agregados para soportar todo esto

No son datos de CEBAC — son capacidades nuevas en el modelo base:

- `InsuranceGroup` (modelo + migración) + `insurance_types.insurance_group_id`
- `nomenclators.type` enum: se agregó `FEES` (antes solo `LIST`/`UNITS`)
- `nomenclator_practice_prices`: +`billing_code`, +`fee_value`, +`expense_value`
- `insurance_pricing_agreements`: +`fee_value`, +`expense_value`
- `rules` + `rule_insurance_type` + `rule_insurance_plan` (motor nuevo). La
  tabla `billing_rules` y sus pivots **fueron eliminados** junto con
  `BillingRule`, `BillingRuleEngine` y `ValuationRuleTestingSeeder`;
  `AdmissionValuationService` ahora llama al `rules-engine` vía
  `BillingRuleEngineHttpClient`
- `RequirementType::LAYOUT_TEXT()` / `LAYOUT_LIST()`: nuevo parámetro
  opcional `bool $required = false`

## Bugs reales encontrados (y corregidos) durante la migración

Vale la pena releer esta lista si algo migrado "no cierra" — varios de
estos fueron sutiles y no tiraban error, solo datos mal cargados:

- **`FacturacionReglas` acción "Anular" estaba invertida**: el import a
  `BillingRule` hacía `suppress_practice_type_ids = triggerIds`, o sea borraba
  los códigos *disparadores*; lo que el legacy borra es `fr_est_cod_new`. En
  FR_268 ("QUITA ESTUDIOS CALCULADOS", trigger `660193`, target = 5 códigos
  derivados) el efecto era exactamente el opuesto al buscado. Son 6 reglas.
  Corregido en `CebacRuleCatalogSeeder`; la tabla `billing_rules` ya no
  existe, así que no quedaron filas con el bug.
- **Los códigos de `FacturacionReglas` se resolvían contra el catálogo
  equivocado**: son `td_cod_est` (código de facturación), no `e_codigo` — ver
  la sección de `CebacRuleCatalogSeeder`. Explica buena parte del "60 de 114"
  del import viejo, que nunca se había desglosado.
- **Reglas "Reemplazar" con target vacío** (FR_393, FR_394): son borrados
  puros y el import viejo las descartaba por no tener target. En el formato
  nuevo son `remove_practice` y se recuperan.
- **`TarifasDet`**: `td_ug`/`td_uh` (gastos/honorarios, tipo INOS) estaban
  invertidos — se corrigió el mapeo (`ug`→`expense_value`, `uh`→`fee_value`).
- **`Tarifas.t_activo`** nunca se leía — las 129 tarifas inactivas (de 497)
  se importaban todas como activas.
- **`FacturacionReglas`**: se usaba `fr_suc_id` (sucursal, casi constante
  en todas las filas) en vez de `fr_cm_id` (cobertura real) para el
  alcance de la regla.
- **`CoberturaMedica.cm_derivante`/`cm_apb`**: se usaban como
  `requires_voucher`/`is_active` por posición, pero no significan eso
  (`cm_derivante` marca si es un derivante/médico de facturación directa;
  `cm_apb` no tiene un significado de "activo" confirmado — no existe
  `cm_activo` en el schema).
- **`Medicos.m_activo`**: es el flag de activo real (confirmado con
  `*Fields.csv` y con un caso conocido: CARBALLO GRACIELA, referente real
  de CEBAC, tiene `m_activo=1`). Hasta encontrarlo se importaban ~4024
  médicos sin filtrar; con el filtro correcto quedan 417.
- **Pivot `billing_rule_insurance_type`** usaba `syncWithoutDetaching` en
  vez de `sync` — una asociación incorrecta de una corrida vieja (buggeada
  por el error de `fr_suc_id` de arriba) sobrevivía indefinidamente a
  reseeds posteriores porque nunca se "destegaba". Con `sync()` completo
  se limpia sola en cada corrida.
- **Export truncado de `EstudiosIndicaciones.csv`** (1000 de 1667 filas,
  "Select Top 1000 Rows" de SSMS) — ver tabla de arriba.
- **`TipoPaciente.csv` con header sin `skipHeader`** — la fila de
  encabezado se coló como un `OrderType` falso (`code=tp_codigo`).

## Estado final (última corrida completa validada)

| Tabla | Filas |
|---|---|
| `identification_types` | 3 |
| `order_types` | 9 |
| `working_areas` | 26 |
| `insurance_groups` | 21 |
| `insurance_types` | 909 |
| `insurance_plans` | 0 (a propósito) |
| `physicians` | 4.024 (417 activos / 3.607 inactivos) |
| `study_types` | 1388 (1357 estudios reales + 31 perfiles) |
| `practice_types` | 1388 (espejo 1:1) |
| `nomenclators` | 93 (de 497 — solo los usados por algún convenio importado) |
| `nomenclator_practice_prices` | 33.160 (29.002 convenidos al 100%, 242 con autorización previa) |
| `insurance_pricing_agreements` | 670 (568 vigentes + 102 fallback vencidos) |
| `rules` | 55 (de 114 filas legacy — desglose completo arriba) |
| `rule_insurance_type` (pivot) | 4 |
| `requirement_types` | 240 (152 preguntas de encuesta + 88 documentos imprimibles) |
| `study_requirement_type` (pivot) | 1.071 |

## Deliberadamente no migrado / no modelado

- **`Tarifas.t_id_tar_base` / `t_base_factor`** (cascada de tarifa base ×
  factor, ~40% de las tarifas la tienen poblada): no se implementó.
  Verificado que las 11 tarifas sin `TarifasDet` propio tampoco se
  recuperarían con esto (su `t_id_tar_base` es NULL o el sentinel `0`), así
  que no hay pérdida de datos real — parece metadata histórica/administrativa,
  no necesaria para el pricing actual (que ya viene "aplanado" en
  `TarifasDet`).
- **`FacturacionReglas.fr_revisar`** (2 de 114 filas marcadas "a revisar"
  en el sistema legacy): no se filtran, se importan igual que el resto.
- **`ind_hijas`** (indicaciones dependientes): sin datos en este export,
  sin modelar.

## Pendiente / para el próximo hilo

- **Pedirle a CEBAC un export completo de `EstudiosPruebas.csv`**: el actual
  trae 1357 estudios, pero `TarifasDet` referencia 1693 `td_est_id` distintos
  — faltan **364 estudios** (~21%), y son estudios reales, no basura
  (HEMOSIDERINURIA, PROTEINA S TOTAL, CARIOTIPO, PROTEINOGRAMA LCR...). 211 de
  los 364 traen el nombre en la propia `TarifasDet`, así que se puede
  verificar sin acceso a la base vieja. Impacto conocido: 24 reglas de
  facturación bloqueadas (el techo pasaría de 55 a 79). Impacto no medido
  todavía: cualquier otra cosa que cruce contra estudios. No es un
  truncamiento de "Select Top 1000" como pasó con `EstudiosIndicaciones.csv`
  (1357 no es un número redondo) — más probable que el export tenga un filtro
  puesto.
- **Reglas de "código de facturación externo"** (PAMI/OSDE/NBU 2024): 22
  códigos que no son prácticas nuestras. Modelarlas como
  `nomenclator_practice_prices.billing_code` por convenio en vez de como
  reglas.
- **Seguridad, sin resolver**: `lis-broker-gateway/.env.prod` está
  trackeado en git (no en `.gitignore`, a diferencia de los otros 4 repos)
  — tiene credenciales reales de Swiss Medical (API key, password, CUIT)
  en el historial. Sacarlo del tracking es fácil; si el repo se pusheó
  alguna vez a un remoto con eso adentro, hay que rotar esas credenciales
  con Swiss Medical independientemente.
- **Deploy CEBAC**: durante el primer deploy real con estos datos aparecieron
  varios problemas de `.env.prod` en producción (no específicos de la
  migración de datos en sí, pero directamente downstream):
  - `CENTRAL_DB_*`/`TENANT_DB_*` tenían baked-in la conexión al SQL Server
    legacy ("Labcore", 192.168.5.10) — colgaba `migrate`. Fix: deben
    espejar `DB_*` (vía interpolación `${DB_HOST}` etc, que `phpdotenv`
    soporta).
  - Faltaba toda la sección `BROKER_GATEWAY_*` en el `.env.prod` del
    backend.
  - `AWS_TEXTRACT_ENABLED=false` con `DOCUMENT_ANALYSIS_PROVIDER=textract`
    (inconsistente) — sospechoso de causar 422 en clinical-matcher
    `/analyze/person-id`; se corrigió a `true` pero falta confirmar que
    resolvió el 422 en el server real.
  - nginx/backend-proxy resuelven el hostname de sus upstreams una sola
    vez al arrancar — hay que reiniciarlos (no solo "recrear si cambió
    config") cada vez que se recrea cualquier contenedor que proxean, o
    quedan devolviendo 502 contra una IP vieja. `lis-infra/scripts/redeploy.sh`
    ya lo hace automático.
  - Login de Cognito dio "Invalid Cognito token" en un momento del deploy
    — quedó sin confirmar si se resolvió solo con los fixes de arriba o
    si sigue pendiente; revisar de nuevo si vuelve a aparecer.
