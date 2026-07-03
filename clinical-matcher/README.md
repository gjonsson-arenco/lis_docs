# LIS Clinical Matcher (FastAPI)

Servicio sincrono de matching de estudios medicos para integracion con Laravel.

## Caracteristicas de V1

- Python 3.11+
- FastAPI + Pydantic + Uvicorn
- Catalogo cargado en memoria al iniciar
- Sin Redis
- Endpoints:
  - `GET /health`
  - `POST /match`
  - `POST /internal/reload-catalog`
- Matching con aliases, normalizacion y fuzzy matching (rapidfuzz)
- Respuesta con maximo 3 candidatos por linea
- Estados: `auto`, `review`, `manual`, `no_match`
- Preparado para evolucionar a otro origen de catalogo via `CatalogRepository`

## Estructura

- `app/main.py`: API y wiring
- `app/config.py`: configuracion por entorno
- `app/schemas.py`: modelos request/response
- `app/catalog/repository.py`: interfaz de repositorio
- `app/catalog/memory_repository.py`: implementacion en memoria
- `app/normalization/normalizer.py`: normalizacion OCR
- `app/matching/candidate_retriever.py`: shortlist de candidatos
- `app/matching/scorer.py`: score combinado
- `app/matching/ranker.py`: ranking de candidatos
- `app/matching/service.py`: orquestacion de matching
- `app/utils/timing.py`: medicion de tiempo

## Instalacion

```bash
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```

## Ejecutar servicio

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

Variables opcionales:

- `APP_NAME` (default: `lis-clinical-matcher`)
- `APP_VERSION` (default: `1.0.0`)
- `CATALOG_SOURCE` (default: `local_json`, opciones: `local_json`, `http`)
- `CATALOG_PATH` (default: `app/catalog/data/catalog.json`)
- `CATALOG_API_BASE_URL` (default: `http://localhost`)
- `CATALOG_API_PATH` (default: `/api/v1/admin/settings/catalogs/study-types`)
- `CATALOG_API_TIMEOUT_SECONDS` (default: `10`)
- `CATALOG_API_TOKEN` (default: vacio)
- `CATALOG_API_PER_PAGE` (default: `500`)
- `DEFAULT_MAX_CANDIDATES` (default: `3`)
- `DEFAULT_AUTO_THRESHOLD` (default: `0.92`)
- `DEFAULT_REVIEW_THRESHOLD` (default: `0.70`)
- `SHORTLIST_LIMIT` (default: `25`)

Para usar catalogo remoto desde Laravel (admin settings), configurar:

```bash
set CATALOG_SOURCE=http
set CATALOG_API_BASE_URL=http://tu-laravel-interno
set CATALOG_API_PATH=/api/v1/admin/settings/catalogs/study-types
set CATALOG_API_TOKEN=tu_token_si_aplica
```

El matcher consume:

`GET /api/v1/admin/settings/catalogs/study-types?is_active=true&per_page=500&order_by=name&order_dir=asc`

y mantiene ese catalogo cargado en memoria. El endpoint interno `POST /internal/reload-catalog` vuelve a consultar Laravel y refresca el catalogo sin reiniciar proceso.

## Ejecutar tests

```bash
pytest -q
```

## Ejemplos curl

Health:

```bash
curl -X GET "http://localhost:8000/health"
```

Match:

```bash
curl -X POST "http://localhost:8000/match" \
  -H "Content-Type: application/json" \
  -d '{
    "analysis_id": "pa_123",
    "tenant_id": "tenant_1",
    "context": {
      "patient_id": "P_123",
      "doctor_id": "D_456",
      "specialty": "clinica_medica"
    },
    "ocr": {
      "DocumentMetadata": {
        "Pages": 1
      },
      "Blocks": [
        {
          "BlockType": "LINE",
          "Id": "ocr_1",
          "Page": 1,
          "Confidence": 94,
          "Text": "Hemograma completo",
          "Geometry": {
            "BoundingBox": {
              "Width": 0.42,
              "Height": 0.05,
              "Left": 0.12,
              "Top": 0.34
            }
          }
        },
        {
          "BlockType": "LINE",
          "Id": "ocr_2",
          "Page": 1,
          "Confidence": 91,
          "Text": "Glucemia",
          "Geometry": {
            "BoundingBox": {
              "Width": 0.24,
              "Height": 0.04,
              "Left": 0.12,
              "Top": 0.41
            }
          }
        }
      ]
    },
    "options": {
      "max_candidates": 3,
      "auto_threshold": 0.92,
      "review_threshold": 0.7
    }
  }'
```

Reload catalogo:

```bash
curl -X POST "http://localhost:8000/internal/reload-catalog"
```

## Integracion Laravel

Laravel debe enviar el payload OCR al endpoint `/match` y consumir la respuesta para persistir:

- `match_status` por linea
- candidato seleccionado
- alternativas
- `status` global del analisis

## Notas de evolucion

- La interfaz `CatalogRepository` permite reemplazar el origen por Redis, DB u otro servicio sin romper contratos.
- El pipeline de score se puede reemplazar por un modelo entrenado (Logistic Regression, XGBoost) manteniendo el contrato de API.
