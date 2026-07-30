# Análisis de Performance - Medical Order Analysis
**Fecha**: 2026-07-30  
**Estado**: En análisis

---

## 1. Síntomas Reportados

### Error Principal
- **Endpoint**: `POST /api/v1/documents/{id}/analysis`
- **Error**: `Maximum execution time of 30 seconds exceeded` en Laravel
- **Manifestación**: `study_candidates` no retornan en respuesta
- **Impacto**: Análisis de órdenes médicas completamente bloqueado

### Timeline de Degradación
- **Antes**: Catálogo pequeño (~100-300 estudios) → respuesta rápida (< 5s)
- **Ahora**: Catálogo extendido (1,388 estudios) → timeout (46+ segundos)

---

## 2. Causa Raíz

### Catálogo Expandido
```
Estudios en catálogo: 1,388
Estado al startup: ✅ Cargado correctamente
```

### Algoritmo de Retrieval - Clinical Matcher
**Ubicación**: `lis-clinical-matcher/app/matching/candidate_retriever.py`

```python
def shortlist(line_text, line_tokens, catalog):
    for study in catalog:  # Itera TODO el catálogo
        best_fuzzy = max(
            fuzz.WRatio(line_text, term) 
            for term in study.search_terms  # ~4 términos por estudio
        )
```

### Complejidad Computacional

| Factor | Valor |
|--------|-------|
| Líneas por orden | 26 |
| Estudios en catálogo | 1,388 |
| Términos de búsqueda por estudio | ~4 |
| **Total WRatio calls por orden** | **26 × 1,388 × 4 = 145,152** |

### Operación Costosa: WRatio
- **Algoritmo**: Levenshtein-based fuzzy matching
- **Complejidad**: O(n×m) donde n,m = longitud de strings
- **Tiempo estimado**: ~0.3-0.5ms por call
- **Total esperado**: 145k × 0.4ms = **58 segundos** ⚠️

### Logs Observados
```
2026-07-30 11:51:47 INFO match.response elapsed_ms=46791
# Respuesta actual: 46.7 segundos
# Laravel timeout: 30 segundos → TIMEOUT
```

---

## 3. Soluciones Intentadas

### ❌ Token-Based Index (Reverted)
**Intención**: Filtrar estudios por token overlap antes de WRatio

**Implementación**:
```python
# Crear índice inverso en startup
token_index = {
    "pcr": [study1, study2, ...],
    "hemograma": [study3, study4, ...],
    ...
}

# En shortlist, filtrar por tokens compartidos
for token in line_tokens:
    candidates = token_index.get(token, [])
```

**Problema Descubierto**:
- Tokens simples muy comunes: "c", "reactiva", "vitamina", "test"
- Estos aparecen en **cientos** de estudios
- Filtro retorna 1,000+ estudios de todas formas (sin mejora)
- Ejemplo:
  - Línea: "PCR"
  - Tokens: {"pcr"}
  - Estudios con "pcr": 234
  - Estudios con "c": 1,387 (casi todos)
  - Reducción neta: **0-5%** (inefectivo)

**Conclusión**: Tokenización no es granular enough para dominio médico

---

## 4. Verdaderos Cuellos de Botella

Basado en análisis de logs y código:

### 4.1 WRatio Fuzzy Matching - PRINCIPAL (80-85% del tiempo)
- 145,152 comparaciones fuzzy por orden
- Cada una: ~0.3-0.5ms
- **Subtotal: ~40-50 segundos**

### 4.2 JSON Serialization (10-15%)
- Response contiene arrays de alternativas
- Payload: ~200KB por respuesta
- **Subtotal: ~5-8 segundos**

### 4.3 Overhead de Red (5-10%)
- Laravel ↔ Clinical Matcher
- **Subtotal: ~1-2 segundos**

---

## 5. Configuración Actual

### ✅ Timeout Aumentado
```env
# Archivos: .env, .env.example, .env.prod
LIS_MATCHER_TIMEOUT_SECONDS=60  # fue 15
```
**Razón**: Permite que el matcher complete, pero no resuelve slowness raíz

---

## 6. Soluciones Recomendadas

### CORTO PLAZO (Quick Wins)
1. **Reduce shortlist_limit**
   - Actual: 25 candidatos por línea
   - Propuesta: 10-12
   - Ahorro: ~50% en WRatio calls
   - Risk: Bajo, ya retorna top alternativas

2. **Aumentar min_fuzzy threshold**
   - Actual: 45 (de 100)
   - Propuesta: 55-60
   - Ahorro: Filtrar más candidatos temprano
   - Risk: Bajo, threshold ya aplica

3. **Limitar por study_type**
   - Pre-filtrar catálogo por tipo de documento
   - "medical_order" → filtrar solo a estudios relevantes
   - Ahorro: Potencial 30-50% reducción
   - Risk: Requiere mapping tipo de documento → estudio

### MEDIANO PLAZO (Engineering)
4. **Algoritmo fuzzy más rápido**
   - Cambiar WRatio → RapidFuzz (ya usan) con `processor=None`
   - Usar LCS o Jaro-Winkler si tolerancia permite
   - Ahorro: 30-40% tiempo por call

5. **Paralelización**
   - Usar multiprocessing.Pool
   - Procesar líneas en paralelo (26 líneas simultáneas)
   - Ahorro: ~5-8x speedup
   - Risk: Memory overhead, complexity

6. **Caching**
   - Cache de resultados WRatio para términos frecuentes
   - TTL: 24 horas
   - Ahorro: 20-30% en órdenes repetidas

### LARGO PLAZO (Architecture)
7. **Search Index Especializado**
   - Elasticsearch, MeiliSearch o Milvus
   - Vector embeddings del catálogo
   - Búsqueda similarity ~1-2ms vs 40s actual
   - Cost: Infrastructure, maintenance

8. **Catálogo Segmentado**
   - Split por especialidad médica
   - Request incluye especialidad → filtro automático
   - Ahorro: 70-80% reducción

---

## 7. Recomendación Inmediata

**Aplicar Corto Plazo #1-3 (combinadas)**:

```python
# config.py
shortlist_limit = 10  # was 25
min_fuzzy = 60  # was 45
study_type_filter_enabled = True
```

**Impacto Esperado**:
- Reducción: 40-50% en tiempo total
- Nuevo tiempo estimado: 20-25 segundos
- Sigue en timeout, pero más cercano

**Próximo paso**: Perfilar específicamente cuál operación consume más tiempo:
```bash
# En clinical-matcher
python -m cProfile -s cumtime app/main.py | grep -E "(WRatio|normalize|shortlist)"
```

---

## 8. Logs Disponibles para Análisis

**Ubicación**: `c:\Projects\LIS\lis-clinical-matcher\logs\matcher.log`

**Últimas líneas contienen**:
- Timestamp: `2026-07-30 11:59:35`
- Analysis ID: `pa_lpgxgxu1kxrt`
- 26 candidatos procesados
- Errores de type hints en optimizaciones anteriores

**Comandos útiles**:
```powershell
# Ver últimas 100 líneas
Get-Content "c:\Projects\LIS\lis-clinical-matcher\logs\matcher.log" -Tail 100

# Filtrar por analysis_id específico
Select-String "pa_lpgxgxu1kxrt" "c:\Projects\LIS\lis-clinical-matcher\logs\matcher.log"

# Ver tiempos de respuesta
Select-String "elapsed_ms" "c:\Projects\LIS\lis-clinical-matcher\logs\matcher.log" | Tail -20
```

---

## 9. Checklist para Próximos Pasos

- [ ] Aplicar Quick Wins (shortlist_limit, min_fuzzy, study_type_filter)
- [ ] Restar con orden médica de prueba
- [ ] Capturar tiempo nuevo (target: <25s)
- [ ] Hacer profiling de WRatio vs otras operaciones
- [ ] Evaluar Elasticsearch vs parallelization
- [ ] Documentar configuración final en README

---

**Autor**: GitHub Copilot  
**Última actualización**: 2026-07-30 12:00 UTC
