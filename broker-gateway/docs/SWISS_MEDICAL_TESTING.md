# Swiss Medical (SMG) Adapter - Testing Guide

## Overview

El adapter de Swiss Medical es una implementación del `BrokerAdapterPort` que maneja la integración con la API de Swiss Medical. En esta etapa inicial, el adapter devuelve respuestas mokeadas para permitir testing desde el frontend sin necesidad de credenciales reales.

## Endpoints disponibles

```
POST /v1/internal/brokers/swiss-medical/eligibility
POST /v1/internal/brokers/swiss-medical/transaction
POST /v1/internal/brokers/swiss-medical/cancel
POST /v1/internal/brokers/swiss-medical/report-diagnosis
GET  /v1/internal/brokers/swiss-medical/capabilities
```

## Testing desde el Frontend

### 1. Comprobar Elegibilidad

**Request:**
```bash
curl -X POST http://localhost:3000/v1/internal/brokers/swiss-medical/eligibility \
  -H "Content-Type: application/json" \
  -H "x-internal-api-key: your-internal-api-key" \
  -H "x-request-id: req_123" \
  -H "x-trace-id: trace_456" \
  -d '{
    "requestId": "req_123",
    "traceId": "trace_456",
    "memberId": "healthy-patient",
    "documentType": "DNI",
    "documentNumber": "12345678",
    "payerCode": "SMG001",
    "serviceDate": "20260702"
  }'
```

**Response (Aprobado):**
```json
{
  "requestId": "req_123",
  "traceId": "trace_456",
  "broker": "swiss-medical",
  "eligible": true,
  "affiliateStatus": "active",
  "coveragePercent": 95,
  "message": "Swiss Medical eligibility check (mock) - ELIGIBLE",
  "brokerReferenceId": "SMG-ELIG-550e8400-e29b-41d4-a716-446655440000",
  "brokerMetadata": {
    "name": "swiss-medical",
    "version": "mock-v1",
    "environment": "mock",
    "capabilities": ["eligibility", "transaction", "cancellation"]
  }
}
```

**Casos de prueba:**

- `NOELIG` en memberId → elegibilidad rechazada
- `TIMEOUT` en memberId → timeout
- `CREDERR` en memberId → error de credenciales
- `AFF404` en memberId → afiliado no encontrado

### 2. Procesar Transacción

**Request - Aprobación completa:**
```bash
curl -X POST http://localhost:3000/v1/internal/brokers/swiss-medical/transaction \
  -H "Content-Type: application/json" \
  -H "x-internal-api-key: your-internal-api-key" \
  -H "x-request-id: req_123" \
  -H "x-trace-id: trace_456" \
  -d '{
    "requestId": "req_123",
    "traceId": "trace_456",
    "idempotencyKey": "idempo_abc123",
    "memberId": "patient-001",
    "payerCode": "SMG001",
    "items": [
      {
        "practiceCode": "42010100",
        "quantity": 1
      },
      {
        "practiceCode": "35010247",
        "quantity": 2
      }
    ],
    "amount": 500.00,
    "serviceDate": "20260702"
  }'
```

**Response - Aprobación completa:**
```json
{
  "requestId": "req_123",
  "traceId": "trace_456",
  "broker": "swiss-medical",
  "transactionStatus": "approved",
  "authorizationCode": "SMG-AUTH-45623",
  "headerRejection": null,
  "lineItems": [
    {
      "transactionId": 123456789,
      "quantity": 1,
      "status": "approved",
      "rejection": null,
      "copayValue": 15.75
    },
    {
      "transactionId": 234567890,
      "quantity": 2,
      "status": "approved",
      "rejection": null,
      "copayValue": 15.75
    }
  ],
  "brokerReferenceId": "SMG-TXN-550e8400-e29b-41d4-a716-446655440000",
  "brokerMetadata": {
    "name": "swiss-medical",
    "version": "mock-v1",
    "environment": "mock",
    "capabilities": ["eligibility", "transaction", "cancellation"]
  }
}
```

**Request - Rechazo a nivel de cabecera:**
```bash
# Usar memberId que contenga "REJECT-HEADER"
curl -X POST http://localhost:3000/v1/internal/brokers/swiss-medical/transaction \
  -H "Content-Type: application/json" \
  -H "x-internal-api-key: your-internal-api-key" \
  -d '{
    "requestId": "req_124",
    "traceId": "trace_457",
    "idempotencyKey": "idempo_abc124",
    "memberId": "patient-REJECT-HEADER",
    ...
  }'
```

**Response - Rechazo a nivel de cabecera (código 12: Requiere autorización):**
```json
{
  "requestId": "req_124",
  "traceId": "trace_457",
  "broker": "swiss-medical",
  "transactionStatus": "rejected",
  "authorizationCode": null,
  "headerRejection": {
    "code": "REQUIRES_AUTHORIZATION",
    "message": "Swiss Medical rejection (code 12): Requiere autorizacion",
    "brokerCode": 12,
    "brokerMessage": "Requiere autorizacion",
    "level": "header",
    "brokerMetadata": {
      "smgCode": 12,
      "smgDescription": "Requiere autorizacion",
      "adapter": "swiss-medical"
    }
  },
  "lineItems": [
    {
      "transactionId": 123456789,
      "quantity": 0,
      "status": "rejected",
      "copayValue": 0
    }
  ],
  "brokerReferenceId": "SMG-TXN-550e8400-e29b-41d4-a716-446655440000",
  "brokerMetadata": {...}
}
```

**Request - Aprobación parcial:**
```bash
# Usar memberId que contenga "PARTIAL"
curl -X POST http://localhost:3000/v1/internal/brokers/swiss-medical/transaction \
  -H "Content-Type: application/json" \
  -H "x-internal-api-key: your-internal-api-key" \
  -d '{
    "requestId": "req_125",
    "traceId": "trace_458",
    "idempotencyKey": "idempo_abc125",
    "memberId": "patient-PARTIAL",
    "items": [
      {
        "practiceCode": "42010100",
        "quantity": 1
      },
      {
        "practiceCode": "35010247",
        "quantity": 2
      }
    ],
    ...
  }'
```

**Response - Aprobación parcial (1ª línea OK, 2ª rechazada con código 6: Prestación dada de baja):**
```json
{
  "requestId": "req_125",
  "traceId": "trace_458",
  "broker": "swiss-medical",
  "transactionStatus": "partial",
  "authorizationCode": "SMG-AUTH-87654",
  "headerRejection": null,
  "lineItems": [
    {
      "transactionId": 123456789,
      "quantity": 1,
      "status": "approved",
      "rejection": null,
      "copayValue": 15.75
    },
    {
      "transactionId": 234567890,
      "quantity": 2,
      "status": "rejected",
      "rejection": {
        "code": "PRACTICE_INACTIVE",
        "message": "Swiss Medical rejection (code 6): Prestación dada de baja",
        "brokerCode": 6,
        "brokerMessage": "Prestación dada de baja",
        "level": "line",
        "brokerMetadata": {
          "smgCode": 6,
          "smgDescription": "Prestación dada de baja",
          "adapter": "swiss-medical"
        }
      },
      "copayValue": 0
    }
  ],
  "brokerReferenceId": "SMG-TXN-550e8400-e29b-41d4-a716-446655440000",
  "brokerMetadata": {...}
}
```

### 3. Cancelar Transacción

**Request - Cancelación exitosa:**
```bash
curl -X POST http://localhost:3000/v1/internal/brokers/swiss-medical/cancel \
  -H "Content-Type: application/json" \
  -H "x-internal-api-key: your-internal-api-key" \
  -d '{
    "requestId": "req_126",
    "traceId": "trace_459",
    "memberId": "patient-001",
    "payerCode": "SMG001",
    "transactionId": 123456789,
    "serviceDate": "20260702"
  }'
```

**Response - Cancelación exitosa:**
```json
{
  "requestId": "req_126",
  "traceId": "trace_459",
  "broker": "swiss-medical",
  "cancelStatus": "approved",
  "rejection": null,
  "brokerReferenceId": "SMG-CANCEL-550e8400-e29b-41d4-a716-446655440000",
  "brokerMetadata": {...}
}
```

**Request - Cancelación rechazada (transactionId = 999):**
```bash
curl -X POST http://localhost:3000/v1/internal/brokers/swiss-medical/cancel \
  -H "Content-Type: application/json" \
  -H "x-internal-api-key: your-internal-api-key" \
  -d '{
    "requestId": "req_127",
    "traceId": "trace_460",
    "memberId": "patient-001",
    "payerCode": "SMG001",
    "transactionId": 999,
    "serviceDate": "20260702"
  }'
```

**Response - Cancelación rechazada (código 277: Transacción inexistente o de baja):**
```json
{
  "requestId": "req_127",
  "traceId": "trace_460",
  "broker": "swiss-medical",
  "cancelStatus": "rejected",
  "rejection": {
    "code": "GENERIC_ERROR",
    "message": "Swiss Medical rejection (code 277): Tran.Inex.o de Baja",
    "brokerCode": 277,
    "brokerMessage": "Tran.Inex.o de Baja",
    "level": "header",
    "brokerMetadata": {
      "smgCode": 277,
      "smgDescription": "Tran.Inex.o de Baja",
      "adapter": "swiss-medical"
    }
  },
  "brokerReferenceId": "SMG-CANCEL-550e8400-e29b-41d4-a716-446655440000",
  "brokerMetadata": {...}
}
```

### 4. Informar Diagnóstico

**Request - Diagnóstico exitoso:**
```bash
curl -X POST http://localhost:3000/v1/internal/brokers/swiss-medical/report-diagnosis \
  -H "Content-Type: application/json" \
  -H "x-internal-api-key: your-internal-api-key" \
  -d '{
    "requestId": "req_128",
    "traceId": "trace_461",
    "memberId": "patient-001",
    "payerCode": "SMG001",
    "transactionId": 123456789,
    "diagnosisCode": "J06.9",
    "diagnosisDescription": "Infección aguda de las vías respiratorias superiores"
  }'
```

**Response - Diagnóstico exitoso:**
```json
{
  "requestId": "req_128",
  "traceId": "trace_461",
  "broker": "swiss-medical",
  "reportStatus": "approved",
  "rejection": null,
  "brokerReferenceId": "SMG-DIAG-550e8400-e29b-41d4-a716-446655440000",
  "brokerMetadata": {...}
}
```

**Request - Diagnóstico rechazado (diagnosisCode = INVALID):**
```bash
curl -X POST http://localhost:3000/v1/internal/brokers/swiss-medical/report-diagnosis \
  -H "Content-Type: application/json" \
  -H "x-internal-api-key: your-internal-api-key" \
  -d '{
    "requestId": "req_129",
    "traceId": "trace_462",
    "memberId": "patient-001",
    "payerCode": "SMG001",
    "transactionId": 123456789,
    "diagnosisCode": "INVALID",
    "diagnosisDescription": "Código inválido"
  }'
```

**Response - Diagnóstico rechazado (código 632: Diagnóstico inexistente):**
```json
{
  "requestId": "req_129",
  "traceId": "trace_462",
  "broker": "swiss-medical",
  "reportStatus": "rejected",
  "rejection": {
    "code": "DIAGNOSIS_INEXISTENT",
    "message": "Swiss Medical rejection (code 632): Diagnostico Inexistente",
    "brokerCode": 632,
    "brokerMessage": "Diagnostico Inexistente",
    "level": "line",
    "brokerMetadata": {
      "smgCode": 632,
      "smgDescription": "Diagnostico Inexistente",
      "adapter": "swiss-medical"
    }
  },
  "brokerReferenceId": "SMG-DIAG-550e8400-e29b-41d4-a716-446655440000",
  "brokerMetadata": {...}
}
```

### 5. Obtener Capabilities

**Request:**
```bash
curl -X GET http://localhost:3000/v1/internal/brokers/swiss-medical/capabilities \
  -H "x-internal-api-key: your-internal-api-key"
```

**Response:**
```json
{
  "broker": "swiss-medical",
  "capabilities": ["eligibility", "transaction", "cancellation"],
  "metadata": {
    "name": "swiss-medical",
    "version": "mock-v1",
    "environment": "mock",
    "capabilities": ["eligibility", "transaction", "cancellation"]
  }
}
```

## Tabla de Rechazos Mapeados

El adapter mapea códigos de rechazo SMG a códigos canónicos de negocio (BrokerBusinessRejectCode):

| SMG Code | Descripción                          | Código Canónico              | Nivel  |
|----------|--------------------------------------|------------------------------|--------|
| 2-4      | Afiliado inactivo/inhabilitado       | AFFILIATE_INACTIVE           | H      |
| 5        | Prestación inexistente               | PRACTICE_INEXISTENT          | L      |
| 6        | Prestación dada de baja              | PRACTICE_INACTIVE            | L      |
| 9        | Cobertura inexistente                | NOT_IN_PLAN                  | H      |
| 10       | Cobertura dada de baja               | COVERAGE_INACTIVE            | H      |
| 12       | Requiere autorización                | REQUIRES_AUTHORIZATION       | H      |
| 30-31    | Tope diario/mensual                  | DAILY/MONTHLY_LIMIT_EXCEEDED | H      |
| 35       | Credencial inexistente               | AFFILIATE_CREDENTIAL_INEXISTENT | H |
| 37       | Credencial no vigente                | AFFILIATE_CREDENTIAL_EXPIRED | H      |
| 54       | Prestador no en cartilla             | NOT_IN_PLAN                  | H      |
| 141      | Prestación duplicada                 | PRACTICE_DUPLICATED          | L      |
| 148      | Requiere token de afiliado           | REQUIRES_TOKEN               | H      |
| 149      | Token inválido                       | TOKEN_INVALID                | H      |
| 271      | Prestador no habilitado para terminal| PROVIDER_NOT_ENABLED_FOR_TERMINAL | H |
| 632      | Diagnóstico inexistente              | DIAGNOSIS_INEXISTENT         | L      |
| 633      | Diagnóstico ya ingresado             | DIAGNOSIS_ALREADY_ENTERED    | L      |

*H = Header (rechazo a nivel de request), L = Line (rechazo a nivel de línea)*

## Notas de Implementación

1. **Token Manager (Mock)**: El adapter usa un TokenManager mockeado en memoria. En producción, será implementado con Redis para manejo compartido de tokens entre instancias.

2. **Deduplicación**: El campo `idempotencyKey` está presente en `TransactionRequest` para permitir dedupe por idempotencia. En esta fase mock, se ignora. En producción, se implementará cache en Redis.

3. **Validación de campos**: El adapter valida que `items` no esté vacío en transacciones. En producción, se agregará validación de longitud de campos (ej: creden 19 chars, fechas yyyymmdd).

4. **Respuestas multi-línea**: El adapter soporta transacciones con múltiples prácticas (`items[]`), devolviendo un estado `partial` si algunas líneas son aprobadas y otras rechazadas.

## Próximas fases

1. **Integración real con API de SMG**: Reemplazar respuestas mock con llamadas HTTP reales.
2. **Token Manager con Redis**: Implementar sesión compartida entre instancias del gateway.
3. **Deduplicación con Redis**: Prevenir duplicaciones por retries fallidos.
4. **Validación de campos**: Validar formato y longitud antes de enviar a SMG.
5. **Manejo de credencial digital**: Implementar lógica de token de 3 dígitos (rechazo 148/149).
