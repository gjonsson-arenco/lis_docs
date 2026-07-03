# LIS Clinical Matcher - Proceso Completo de Matching

Este documento describe el flujo actual del matcher de punta a punta, incluyendo:

- entrada por API
- resolucion de lineas de entrada (con y sin selecciones)
- clasificacion de lineas
- retrieval, scoring y ranking
- reglas de estado final
- manejo de bbox

## 1. Entry points

### 1.1 Startup

En startup se carga el catalogo y se indexa en memoria.

Pasos:

1. `app.main` crea `MatchingService`.
2. Evento startup llama `service.load_catalog()`.
3. El catalogo se transforma a `IndexedStudy` con:
	- `search_terms`: nombre normalizado + aliases normalizados
	- `token_set`: union de tokens de todos los search terms

### 1.2 Endpoint `/match`

Recibe `MatchRequest` y ejecuta:

1. validacion pydantic
2. chequeo catalogo cargado
3. `service.match(payload, elapsed_ms=0)`
4. set de `elapsed_ms` real
5. logging de resumen + items

## 2. Flujo principal de `MatchingService.match`

Para cada request:

1. Resolver lineas a procesar (`_resolve_input_lines`).
2. Iterar linea por linea preservando orden.
3. Clasificar linea (`discard`, `study_candidate`, `ambiguous`) o forzar `study_candidate` si viene de seleccion.
4. Si no se descarta:
	- normalizar texto
	- generar shortlist
	- score por candidato
	- rankear
	- filtrar alternativas
	- determinar `match_status`
5. Armar `MatchItem` con bbox, razones y candidatos.
6. Calcular `status` global de response.

## 3. Resolucion de input lines (`_resolve_input_lines`)

El proceso usa exclusivamente `ocr.Blocks` como fuente de verdad.

## 3.1 Modo seleccion (`selection_elements`)

Se usa cuando existen `SELECTION_ELEMENT` con `SelectionStatus=SELECTED` que puedan asociarse a texto.

Pipeline:

1. `extract_selected_lines_from_blocks(blocks)` produce `OcrLine(selected=True, source=...)`.
2. Se procesan solo esas lineas seleccionadas.

Notas:

- Si `source == selection_element_kv`, se respeta como label autoritativo.

## 3.2 Modo lineas Textract (sin seleccion)

Si no hay seleccion util, se procesan los blocks `LINE`:

1. Se toma `Text`, `Id`, `Confidence` y `Geometry.BoundingBox` de cada `LINE` valido.
2. Se normaliza `Confidence` a `[0,1]`.
3. Si no hay `Id`, se genera `line_id` tecnico (`blk_line_{n}`).

No se usa `ocr.lines` como fallback.

## 4. Extraccion de labels seleccionados (`selection_elements.py`)

## 4.1 Prioridad estructural KV

Se intenta primero asociar checks usando estructura `KEY_VALUE_SET`:

1. Encontrar KEY blocks.
2. Resolver VALUE relacionados.
3. Encontrar `SELECTION_ELEMENT` hijos seleccionados.
4. Construir label desde KEY:
	- child `LINE` directo si existe
	- sino concat de child `WORD`
	- si esos WORD pertenecen a una LINE, preferir esa LINE

Salida: `source = selection_element_kv`.

## 4.2 Fallback geometrico

Si no hay KV valido para un check:

1. Buscar mejor `WORD` y mejor `LINE` por distancia ponderada.
2. Expandir WORD a LINE padre por:
	- relaciones CHILD (id-based)
	- geometria (overlap y alineacion) si ids faltan
3. Si no hay LINE block, intentar reconstruir frase con WORD vecinas de la misma fila.

Salida: `source = selection_element`.

## 5. Clasificacion de linea (`LineClassifier`)

Input: texto + tokens + similitud contra catalogo.

Senales negativas (ejemplos):

- datos de paciente (`dni`, `paciente`, etc)
- obra social
- firma/medico
- fechas
- telefono
- probable nombre
- alto ratio numerico

Senales positivas:

- similitud al catalogo
- prefijos clinicos (`hemo`, `gluc`, `tsh`, `t4`, etc)
- abreviaturas clinicas

Decisiones:

1. `discard` si negativa fuerte.
2. `study_candidate` si similitud catalogo alta o senal clinica suficiente.
3. `ambiguous` por defecto para no perder posibles estudios.

Excepcion importante:

- lineas provenientes de seleccion (`selected=True` y `source` selection) se fuerzan a `study_candidate` con razon `selection_element_selected`.

## 6. Candidate retrieval (`CandidateRetriever`)

Para cada estudio indexado:

1. `best_fuzzy = max(WRatio(line, term))`
2. `overlap = token_overlap(line_tokens, study.token_set)`
3. `pre_score = best_fuzzy*0.8 + overlap*0.2`

Entra al shortlist si:

- `best_fuzzy >= min_fuzzy` (default 45), o
- `overlap > 0`

Luego se ordena por `pre_score` y se limita por `shortlist_limit`.

## 7. Scoring (`MatchScorer`)

Para cada candidato shortlisted:

Features:

- exactitud alias/nombre
- `ratio`
- `partial_ratio`
- `token_sort_ratio`
- `token_overlap`
- `ocr_confidence`

Formula:

`score = exact*0.45 + ratio*0.20 + partial*0.10 + token_sort*0.15 + overlap*0.08 + ocr*0.02`

Clampeado a `[0,1]`.

Tambien genera razones (`alias_exact`, `partial_similarity`, etc).

## 8. Ranking y alternativas

`CandidateRanker` ordena desc por score.

Alternativas se incluyen si cumplen ambas:

1. dentro de `max_candidates`
2. `score >= MIN_ALTERNATIVE_SCORE` (0.35)

## 9. Match status por linea

`_resolve_match_status(top_score, has_candidates, gap, options)`:

1. `no_match` si no hay candidatos o `top_score < 0.60`
2. `auto` si `top_score >= auto_threshold` y `gap >= 0.10`
3. `review` si `top_score >= review_threshold`
4. `manual` en otro caso

`selected_candidate` se informa para `auto`, `review` y `manual`.

## 10. Status global de respuesta

`_overall_status(items)`:

1. sin items -> `manual`
2. solo `auto` -> `auto`
3. si hay algun `review` -> `review`
4. mezcla de `auto` con `manual/no_match` -> `review`
5. resto -> `manual`

## 11. BBox: comportamiento esperado

### 11.1 Con seleccion

- bbox sale de label detectado en `textract_blocks` (KV/LINE/WORD) o de reemplazo OCR si aplica.

### 11.2 Sin seleccion

Prioridad:

1. bbox del block `LINE` (fuente primaria)
2. `null` cuando el block no trae geometry util

## 12. Configuracion que impacta matching

Desde `Settings`:

- `shortlist_limit`
- `default_max_candidates`
- `default_auto_threshold`
- `default_review_threshold`
- `catalog_source` y config de catalog API

## 13. Logging operativo

Se loguea:

- request/response completos (si `LOG_MATCH_PAYLOADS=true`)
- modo input (`selection_elements` u `ocr_lines`)
- por linea: proceso, shortlist, ranking, resultado
- resumen final por request

Esto permite auditar por que una linea fue `discard`, `no_match`, etc.

## 14. Riesgos y puntos de mejora

1. OCR muy ruidoso puede mapear bbox a word cercana pero no ideal.
2. En formularios densos, sin `LINE` ni relaciones KV, la heuristica geometrica puede ser ambigua.
3. Podria agregarse trazabilidad explicita de fuente bbox por item (input/line_block/word_block/none).
4. Podria agregarse un modo estricto para nunca tomar WORD si existe evidencia de LINE.

## 15. Procesamiento Especifico: Analisis de Ordenes Medicas

### 15.1 Diferenciacion de ordenes medicas

Una orden medica se diferencia de otros documentos por su **CAPTURA**, no por logica de procesamiento diferenciada:

| Aspecto | Orden Medica (Formulario) | OCR Directo |
|--------|--------------------------|-----------|
| Fuente de lineas | SELECTION_ELEMENT (checkboxes marcados) | LINE blocks (texto OCR directo) |
| Metadata | `context.document_type = "medical_order"` | Puede omitirse |
| Bypass de reglas | Sí: `selection_element_selected` → fuerza `study_candidate` | No: sujeto a clasificacion normal |
| Confianza esperada | Mayor (usuario selecciono activamente) | Variable (depende calidad OCR) |

**Importante**: El matcher es agnóstico al tipo de documento. La diferencia está en cómo captura las líneas, no en lógica diferenciada.

### 15.2 Flujo completo: Orden medica paso a paso

#### **Fase 1: Startup (Carga del Catálogo)**

```
1. Aplicacion inicia
2. Evento "startup" → service.load_catalog()
3. Catalogo se indexa en memoria:
   ├─ Cada estudio normaliza nombre + aliases
   ├─ Se generan token_set (union de tokens normalizados)
   └─ Se crea indice full-text en memoria
```

Resultado: Catálogo listo para búsquedas rápidas.

#### **Fase 2: Recepcion de MatchRequest (Orden Medica)**

```
Input JSON:
{
  "analysis_id": "order_20260310_001",
  "tenant_id": "tenant_xyz",
  "context": {
    "document_type": "medical_order",
    "source": "form_scan",
    "specialty": "cardiology"
  },
  "ocr": {
    "blocks": [
      SELECTION_ELEMENT(SelectionStatus=SELECTED),
      KEY_VALUE_SET(...),
      LINE(...),
      WORD(...)
    ]
  },
  "options": {
    "auto_threshold": 0.92,
    "review_threshold": 0.70
  }
}
```

La estructura es agnóstica - el matcher trata todas las ordenes igual.

#### **Fase 3: Resolucion de Lineas de Entrada**

**3a. Deteccion de SELECTION_ELEMENT marcados**

```python
# En: app/ocr/selection_elements.py
extract_selected_lines_from_blocks(blocks):
    
    1. Buscar bloques SELECTION_ELEMENT con SelectionStatus="SELECTED"
    
    2. Para cada SELECTED encontrado:
       └─ Asociar LABEL usando:
          a) KEY_VALUE_SET (relacion CHILD) - preferencia 1
             └─ source = "selection_element_kv"
          b) Fallback geometrico (WORD cercano)
             └─ source = "selection_element"
    
    3. Crear OcrLine(selected=True, ...)
       └─ MARCA como seleccionada (importante para Fase 4)
```

**Ejemplo - Entrada de bloques Textract:**

```json
{
  "blocks": [
    {
      "Id": "ce_checkbox_001",
      "BlockType": "SELECTION_ELEMENT",
      "SelectionStatus": "SELECTED",    // ← CHECKBOX MARCADO
      "Geometry": {"BoundingBox": {...}}
    },
    {
      "Id": "kv_set_001",
      "BlockType": "KEY_VALUE_SET",
      "Type": "VALUE",
      "Relationships": [{
        "Type": "CHILD",
        "Ids": ["ce_checkbox_001"]   // ← El checkbox es hijo del KV
      }]
    },
    {
      "Id": "line_hemograma",
      "BlockType": "LINE",
      "Text": "Hemograma Completo",
      "Confidence": 0.98,
      "Geometry": {"BoundingBox": {
        "Width": 0.4,
        "Height": 0.02,
        "Left": 0.1,
        "Top": 0.15
      }}
    }
  ]
}
```

**Resultado - Lineas resueltas:**

```python
[
  OcrLine(
    line_id="selection_001",
    text="Hemograma Completo",
    normalized_text="hemograma completo",
    selected=True,              # ← MARCADA (crucial)
    source="selection_element_kv",
    ocr_confidence=0.98,
    bbox=BoundingBox(...)
  )
]
```

**3b. Lineas sin seleccion (fallback)**

Si la orden no tiene SELECTION_ELEMENT validos, se usan LINE blocks directos:

```python
# Prioridad si no hay selecciones:
for line_block in blocks:
    if line_block.BlockType == "LINE":
        yield OcrLine(
            line_id=f"blk_line_{n}",
            text=line_block.Text,
            selected=False,         # ← NO MARCADA
            source="textract_line",
            ocr_confidence=line_block.Confidence
        )
```

#### **Fase 4: Clasificacion de Linea**

**4a. Linea seleccionada (BYPASS de reglas)**

```python
# En: app/classification/line_classifier.py

if ocr_line.selected == True and "selection_element" in ocr_line.source:
    # BYPASS TOTAL de reglas negativas/positivas
    return LineClassification(
        classification="study_candidate",
        reason="selection_element_selected",
        confidence=1.0
    )
    # ↓ Va directamente a Fase 5 (Retrieval)
```

**4b. Linea OCR sin seleccion (evaluacion de senales)**

```
Para cada linea OCR sin seleccion:

1. EVALUAR SENALES NEGATIVAS (reglas administrativas):
   ├─ Datos de paciente: "DNI", "paciente", "nombre"
   ├─ Obra social: "OSDE", "os", "seguros"
   ├─ Firma/medico: "Dr.", "MN", "MP"
   ├─ Fechas: "2026/03/10"
   ├─ Telefonos: "+54 11 4123..."
   ├─ Alto ratio numerico: >45% de caracteres
   └─ si negative_score ≥ 0.75 → "discard" (descartada)

2. EVALUAR SENALES POSITIVAS (senales clinicas):
   ├─ Prefijos clinicos: "hemo", "gluc", "tsh", "t4", "t3"
   ├─ Abreviaturas: "TSH", "PCR", "VSG", "HbA1c"
   ├─ Similitud con catalogo: WRatio ≥ 0.80 → señal fuerte
   └─ si similitud ≥ 0.80 O senales_positivas ≥ 0.65 → "study_candidate"

3. DECISION FINAL:
   ├─ Si negativa fuerte (≥0.75) → "discard"
   ├─ Si similitud catalogo ≥0.80 → "study_candidate"
   ├─ Si senales positivas ≥0.65 → "study_candidate"
   ├─ Si negativa moderada (≥0.50) Y sin senales positivas → "discard"
   └─ En otros casos → "ambiguous" (no pierde estudios)
```

**Ejemplo - Clasificacion de lineas de orden:**

| Texto | Tokens | Senales | Clasificacion | Razon |
|------|--------|---------|--------------|--------|
| "HEMOGRAMA COMPLETO" (seleccionada) | hemo, completo | selected=True | `study_candidate` | `selection_element_selected` |
| "Glucosa" | gluc | Prefijo "gluc" ✓ | `study_candidate` | `clinical_signal` |
| "DNI 30.555.666" | dni, numeros | Match DNI (0.96) | `discard` | `patient_data` |
| "Obra Social: OSDE" | obra, social | Match OSDE (0.92) | `discard` | `insurance_data` |
| "Dr. Juan Pérez" | dr, juan | Match Dr (0.90) | `discard` | `medical_signature` |
| "Estudio especial" | estudio | Catalogo=0.32 | `ambiguous` | `insufficient_signal` |

#### **Fase 5: Normalizacion de Texto**

```python
# En: app/normalization/normalizer.py

def normalize(text: str) -> str:
    # 1. Convertir a ASCII + minusculas
    result = unidecode(text.lower())  # "HEMOGRAMA" → "hemograma"
    
    # 2. Remover caracteres especiales (excepto letras, numeros, espacios)
    result = regex.sub(r"[^\p{L}\p{N}\s]+", " ", result)
    # "Hemog. Cmp." → "hemog cmp"
    
    # 3. Colapsar espacios multiples
    result = regex.sub(r"\s+", " ", result).strip()
    
    # 4. Aplicar correcciones OCR conocidas
    for erroneo, correcto in ocr_corrections.items():
        result = regex.sub(rf"\b{erroneo}\b", correcto, result)
    
    return result
```

**Ejemplos:**
- `"HEMOGRAMA COMPLETO"` → `"hemograma completo"`
- `"Hemògramma"` → `"hemograma"` (diacríticos removidos)
- `"Hemog. Comp."` → `"hemog comp"` (puntos removidos)

#### **Fase 6: Retrieval de Candidatos (Shortlist)**

```python
# En: app/matching/candidate_retriever.py

Para linea normalizada "hemograma completo":

1. TOKENIZAR:
   tokens_line = {"hemograma", "completo"}

2. PARA CADA ESTUDIO EN CATALOGO:
   ├─ best_fuzzy = max(WRatio("hemograma completo", alias) for alias in aliases)
   ├─ overlap = |tokens_line ∩ study.token_set| / |study.token_set|
   └─ pre_score = best_fuzzy * 0.8 + overlap * 0.2

3. FILTRAR SHORTLIST (inclusion criteria):
   └─ Entra si: best_fuzzy ≥ 45 (min_fuzzy default) O overlap > 0

4. ORDENAR Y LIMITAR:
   └─ Ordenar desc por pre_score
   └─ Limitar a shortlist_limit (default: 25)
```

**Ejemplo - Generacion de shortlist:**

```
Linea: "hemograma completo" (tokens: {hemograma, completo})

ST_001: "Hemograma Completo" (aliases: [hemograma, hemograma completo])
├─ best_fuzzy = WRatio("hemograma completo", "hemograma completo") = 100
├─ overlap = 2/2 = 1.0
└─ pre_score = 100*0.8 + 1.0*0.2 = 80.2

ST_002: "Hemograma + Plaquetas" (aliases: [hemograma plaquetas])
├─ best_fuzzy = WRatio("hemograma completo", "hemograma plaquetas") = 85
├─ overlap = 1/2 = 0.5
└─ pre_score = 85*0.8 + 0.5*0.2 = 68.1

ST_005: "Glucemia" (aliases: [glucemia, glucosa])
├─ best_fuzzy = WRatio("hemograma completo", "glucemia") = 12
├─ overlap = 0
└─ pre_score = 12*0.8 + 0*0.2 = 9.6 → NO ENTRA (< 45)

SHORTLIST FINAL (ordenado por pre_score):
1. ST_001 (80.2)
2. ST_002 (68.1)
```

#### **Fase 7: Scoring de Candidatos Shortlisted**

```python
# En: app/matching/scorer.py

Para cada candidato en shortlist:

score = (
    exact_match * 0.45 +           # ¿alias es exacto?
    fuzzy_ratio * 0.20 +           # WRatio fuzzy
    partial_ratio * 0.10 +         # partial_ratio fuzzy
    token_sort_ratio * 0.15 +      # token-level fuzzy
    token_overlap * 0.08 +         # overlap tokens
    ocr_confidence * 0.02          # confianza OCR
)

reasons = [lista de razone de scoring]
```

**Ejemplo - Scoring detallado:**

```
Linea: "hemograma completo" (ocr_confidence=0.98)

ST_001: "Hemograma Completo" (alias exacto disponible)
├─ exact_match = 1.0 (alias "hemograma completo" coincide exactamente)
├─ fuzzy_ratio = 0.92
├─ partial_ratio = 1.0
├─ token_sort_ratio = 1.0
├─ token_overlap = 1.0 (2/2 tokens comun)
├─ ocr_confidence = 0.98
└─ score = 1.0*0.45 + 0.92*0.20 + 1.0*0.10 + 1.0*0.15 + 1.0*0.08 + 0.98*0.02
   score = 0.45 + 0.184 + 0.10 + 0.15 + 0.08 + 0.0196 = 0.9796

ST_002: "Hemograma + Plaquetas"
├─ exact_match = 0.0 (no hay alias exacto)
├─ fuzzy_ratio = 0.72
├─ partial_ratio = 0.95
├─ token_sort_ratio = 0.92
├─ token_overlap = 0.5 (1/2 tokens)
├─ ocr_confidence = 0.98
└─ score = 0.0*0.45 + 0.72*0.20 + 0.95*0.10 + 0.92*0.15 + 0.5*0.08 + 0.98*0.02
   score = 0.0 + 0.144 + 0.095 + 0.138 + 0.04 + 0.0196 = 0.4366

RANKING FINAL:
1. ST_001: 0.9796 ← TOP CANDIDATE
2. ST_002: 0.4366 ← ALTERNATIVA (si score ≥ 0.35)
```

#### **Fase 8: Resolucion del Match Status por Linea**

```python
# En: app/matching/service.py - _resolve_match_status()

gap = top_score - second_score

if not candidatos or top_score < 0.60:
    status = "no_match"

elif top_score ≥ auto_threshold (0.92) and gap ≥ 0.10:
    status = "auto"
    # → Front puede aplicar automáticamente

elif top_score ≥ review_threshold (0.70):
    status = "review"
    # → Front debe mostrar para revision

else:
    status = "manual"
    # → Front requiere seleccion manual del usuario
```

**Ejemplo - Determinacion de status:**

```
Caso 1: ST_001 score=0.9796, ST_002 score=0.4366
├─ gap = 0.9796 - 0.4366 = 0.5430 ≥ 0.10 ✓
├─ top_score = 0.9796 ≥ 0.92 ✓
└─ status = "AUTO" ← Aplicable automáticamente

Caso 2: ST_005 score=0.88, no hay segunda opcion
├─ gap = 0.88 - 0.0 = 0.88 ≥ 0.10 ✓
├─ top_score = 0.88 ≥ 0.92? NO
├─ top_score = 0.88 ≥ 0.70? SÍ ✓
└─ status = "REVIEW" ← Requiere revision humana

Caso 3: ST_103 score=0.65
├─ gap = 0.65 - 0.0 = 0.65 ≥ 0.10 ✓
├─ top_score = 0.65 ≥ 0.92? NO
├─ top_score = 0.65 ≥ 0.70? NO
└─ status = "MANUAL" ← Seleccion manual del usuario
```

#### **Fase 9: Armar MatchItem**

```python
# En: app/matching/service.py

MatchItem(
    line_id="selection_001",
    source_text="HEMOGRAMA COMPLETO",
    normalized_text="hemograma completo",
    bbox=BoundingBox(width=0.4, height=0.02, left=0.1, top=0.15),
    line_type="study_candidate",
    classification_reason="selection_element_selected",
    match_status="auto",
    confidence_score=0.9796,
    selected_candidate=SelectedCandidate(
        study_id="ST_001",
        code="HEMO001",
        name="Hemograma Completo",
        score=0.9796
    ),
    alternatives=[
        StudyCandidate(
            study_id="ST_002",
            code="HEMO002",
            name="Hemograma + Plaquetas",
            score=0.4366
        )
    ],
    reasons=[
        "line_classified_as_study_candidate",
        "line_reason_selection_element_selected",
        "alias_exact",
        "match_auto_con_diferencia_suficiente"
    ]
)
```

#### **Fase 10: Determinacion del Status Global**

```python
# En: app/matching/service.py - _overall_status()

Algoritmo:
1. Si no hay items → "manual"
2. Si TODOS los items son "auto" → "auto"
3. Si HAY ALGUNO "review" → "review" (independiente del resto)
4. Si hay "manual" o "no_match" pero sin "review" → "manual"
5. Si fallo alguna excepcion → "failed"
```

**Ejemplo - Determinacion de status global:**

```
Items procesados:
├─ "HEMOGRAMA" → match_status="auto" (0.9796)
├─ "Glucosa" → match_status="review" (0.88)
└─ "TSH" → match_status="auto" (0.95)

Status global = "REVIEW" ← Porque hay alguno "review"
                           (independiente de que hay 2 "auto")
```

#### **Fase 11: MatchResponse Final**

```json
{
  "analysis_id": "order_20260310_001",
  "status": "review",
  "elapsed_ms": 234,
  "processing_mode": "selection_elements",
  "items": [
    {
      "line_id": "selection_001",
      "source_text": "HEMOGRAMA COMPLETO",
      "normalized_text": "hemograma completo",
      "line_type": "study_candidate",
      "classification_reason": "selection_element_selected",
      "match_status": "auto",
      "confidence_score": 0.9796,
      "selected_candidate": {
        "study_id": "ST_001",
        "code": "HEMO001",
        "name": "Hemograma Completo",
        "score": 0.9796
      },
      "alternatives": [
        {
          "study_id": "ST_002",
          "code": "HEMO002",
          "name": "Hemograma + Plaquetas",
          "score": 0.4366
        }
      ],
      "reasons": [
        "selection_element_selected",
        "alias_exact",
        "match_auto_con_diferencia_suficiente"
      ],
      "bbox": {
        "width": 0.4,
        "height": 0.02,
        "left": 0.1,
        "top": 0.15
      }
    },
    {
      "line_id": "ocr_002",
      "source_text": "Glucosa",
      "normalized_text": "glucosa",
      "line_type": "study_candidate",
      "classification_reason": "clinical_signal",
      "match_status": "review",
      "confidence_score": 0.88,
      "selected_candidate": {
        "study_id": "ST_005",
        "code": "GLUC001",
        "name": "Glucemia",
        "score": 0.88
      },
      "reasons": [
        "clinical_signal_detected",
        "similitud_parcial",
        "requiere_revision"
      ]
    }
  ],
  "discarded_lines": [
    {
      "line_id": "ocr_003",
      "source_text": "DNI 30.555.666",
      "normalized_text": "dni 30555666",
      "classification": "discard",
      "reason": "patient_data",
      "confidence": 0.96
    },
    {
      "line_id": "ocr_004",
      "source_text": "Obra Social: OSDE",
      "normalized_text": "obra social osde",
      "classification": "discard",
      "reason": "insurance_data",
      "confidence": 0.92
    }
  ]
}
```

### 15.3 Resumen: Diagrama del flujo completo

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. STARTUP: Cargar y indexar catálogo en memoria               │
└─────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. POST /match: Recibir MatchRequest (orden médica)            │
└─────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. RESOLVER LINEAS (bloques Textract):                         │
│    • Si hay SELECTION_ELEMENT.SELECTED                         │
│      └─ extraer líneas seleccionadas (con bbox)                │
│    • Si no, usar LINE blocks (OCR directo)                     │
└─────────────────────────────────────────────────────────────────┘
                               ↓
┌──────────────────────────────────────────────────────────────────┐
│ 4. CLASIFICAR CADA LINEA:                                       │
│    • Si selected=True → BYPASS: "study_candidate"              │
│    • Si selected=False → evaluar señales:                       │
│      - Negativas (DNI, obra social, firma) → "discard"         │
│      - Positivas (clínicas) + similitud → "study_candidate"    │
│      - Ambiguo → "ambiguous"                                    │
└──────────────────────────────────────────────────────────────────┘
                               ↓
┌──────────────────────────────────────────────────────────────────┐
│ 5. SOLO SI study_candidate/ambiguous:                           │
│    a) NORMALIZAR texto (ASCII, minúsculas, limpiar)            │
│    b) RETRIEVAL: Generar shortlist fuzzy                        │
│       └─ WRatio ≥45 O overlap>0                                │
│    c) SCORING: Puntuar cada candidato (formula 7 features)     │
│    d) RANKING: Ordenar por score, filtrar alternativas         │
└──────────────────────────────────────────────────────────────────┘
                               ↓
┌──────────────────────────────────────────────────────────────────┐
│ 6. MATCH STATUS (por línea):                                    │
│    • AUTO: top_score≥0.92 Y gap≥0.10                           │
│    • REVIEW: top_score≥0.70                                    │
│    • MANUAL: top_score<0.70                                    │
│    • NO_MATCH: sin candidatos                                  │
└──────────────────────────────────────────────────────────────────┘
                               ↓
┌──────────────────────────────────────────────────────────────────┐
│ 7. ARMAR MatchItems (una por línea procesada)                   │
│    + Lineas descartadas en discarded_lines[]                    │
└──────────────────────────────────────────────────────────────────┘
                               ↓
┌──────────────────────────────────────────────────────────────────┐
│ 8. STATUS GLOBAL:                                               │
│    • Si alguno "review" → "review"                              │
│    • Si todos "auto" → "auto"                                  │
│    • Si hay "manual"/"no_match" → "manual"                     │
└──────────────────────────────────────────────────────────────────┘
                               ↓
┌──────────────────────────────────────────────────────────────────┐
│ 9. RESPONSE: MatchResponse(status, items[], elapsed_ms)        │
└──────────────────────────────────────────────────────────────────┘
```

## 16. Resumen corto

El matcher hoy combina:

- estructura Textract (KV + SELECTION_ELEMENT)
- heuristicas geometricas para asociar labels a checkboxes
- normalizacion de OCR
- clasificacion por senales (clinicas + administrativas)
- retrieval + score + ranking

Para ordenes medicas:
1. **Prioriza líneas seleccionadas** (checkboxes marcados)
2. **Bypassa reglas** si `selection_element_selected=True`
3. **Mantiene bbox** desde el bloque detectado
4. Devuelve `status` por línea (auto/review/manual/no_match)
5. Calcula `status` global (review > auto > manual)

y devuelve un resultado robusto para front, incluyendo bbox cuando es posible reconstruirlo desde blocks aunque `ocr.lines` no lo traiga.
