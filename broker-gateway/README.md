# lis-broker-gateway

Integration Gateway para desacoplar el dominio de LIS (Laravel) de brokers/prestadores/transaccionadores de salud en Argentina.

## Decision arquitectonica (Etapa 1)

Se analizaron dos alternativas:

- Alternativa A: monolito modular (un solo servicio, adapters internos).
- Alternativa B: orquestador + microservicios por adapter.

Recomendacion Etapa 1: Alternativa A (implementada en este scaffold).

Justificacion:

- Complejidad operativa: A es menor (1 deploy, 1 observabilidad central, 1 pipeline).
- Mantenibilidad: A mantiene alta cohesión del contrato canónico y reduce friccion inicial.
- Escalabilidad: A escala vertical/horizontal en bloque y deja fronteras para extraer adapters.
- Costo de deploy: A significativamente menor en infraestructura y DevOps.
- Velocidad de implementacion: A acelera salida a produccion.
- Aislamiento de fallos: B gana en aislamiento fino, pero A cubre fase inicial con timeout/retries y diseño resiliente.
- Agregar brokers nuevos: A es rapido para stage 1; B se habilita luego al extraer adapters sin romper contrato.

Conclusion: se implementa A con puertos e interfaces para evolucionar a B sin romper el contrato interno consumido por Laravel.

## Arquitectura

Hexagonal / Ports and Adapters:

- domain: modelos canonicos, errores, puertos.
- application: casos de uso y factory/strategy.
- infrastructure: adapters de brokers (stubs), mensajeria SQS preparada.
- interfaces/http: adapters de entrada (API interna).
- config: settings por entorno y por broker.
- observability: health/ready, logs estructurados, base para auditoria.

## Estructura principal

- src/domain/models
- src/domain/ports
- src/domain/errors
- src/application/use-cases
- src/application/services
- src/infrastructure/brokers/adapters
- src/infrastructure/messaging/sqs
- src/interfaces/http
- src/config
- src/observability
- docs/endpoints.http

## Contratos canonicos internos

Modelos definidos para:

- PatientEligibilityRequest
- PatientEligibilityResponse
- TransactionRequest
- TransactionResponse
- BrokerError
- BrokerCapability
- BrokerMetadata

Los contratos internos son independientes de Traditum/IMED y sirven como anti-corruption layer para Laravel.

## Endpoints internos

Con versionado URI:

- POST /v1/internal/brokers/:broker/eligibility
- POST /v1/internal/brokers/:broker/transaction
- GET /v1/internal/brokers/:broker/capabilities
- GET /v1/health
- GET /v1/ready

Swagger:

- GET /docs

## Seguridad interna Laravel -> Gateway

- Header requerido: x-internal-api-key
- Validacion por guard (`InternalApiKeyGuard`)
- Configurable con `INTERNAL_API_KEY`

## Correlacion y trazabilidad

- requestId y traceId via headers `x-request-id`, `x-trace-id`.
- Si no llegan, se generan automaticamente.
- Se devuelven en headers de respuesta.
- Logging estructurado request/response con redaccion de campos sensibles.

## Errores normalizados

Se modelan codigos canonicos:

- BROKER_UNAVAILABLE
- TIMEOUT
- INVALID_CREDENTIALS
- INVALID_REQUEST
- AFFILIATE_NOT_FOUND
- PRACTICE_NOT_COVERED
- FUNCTIONAL_REJECTION
- UNSUPPORTED_CAPABILITY
- UNSUPPORTED_BROKER

El filtro global serializa errores de forma consistente.

## Stubs de brokers (sin integracion real)

- TraditumAdapterStub
- ImedAdapterStub

Transforman contratos internos y devuelven respuestas simuladas coherentes.

No hay credenciales ni llamadas reales a terceros.

## Capabilities variables por broker

Cada adapter declara sus capabilities:

- eligibility
- transaction
- authorization
- cancellation
- statusQuery

El caso de uso valida capability antes de ejecutar.

## Async preparado (SQS)

- Puerto: `BrokerCommandQueuePort`
- Adapter scaffold: `SqsBrokerCommandQueueAdapter`
- Variables: `AWS_SQS_ENABLED`, `AWS_SQS_BROKER_COMMAND_QUEUE_URL`, `AWS_REGION`

Etapa 1 opera en modo sync, pero deja base para colas.

## Resiliencia preparada

Configuracion por broker:

- timeout
- retryAttempts
- retryDelay
- circuitBreakerEnabled (feature flag de diseno para evolucion)

Variables en `.env.example`.

## Como correr

1. Instalar dependencias:

```bash
npm install
```

2. Configurar entorno:

```bash
cp .env.example .env
```

3. Desarrollo:

```bash
npm run start:dev
```

4. Tests:

```bash
npm test
npm run test:e2e
```

## Docker

Build y run:

```bash
docker compose up --build
```

## Como agregar un nuevo broker

1. Crear adapter en `src/infrastructure/brokers/adapters/<nuevo>-adapter.stub.ts` implementando `BrokerAdapterPort`.
2. Declarar capabilities y metadata.
3. Implementar mapping canonico interno <-> contrato broker.
4. Registrar provider en `BrokersModule`.
5. Agregar variables `BROKER_<NUEVO>_*` en `.env.example` y schema.
6. Agregar tests unitarios del adapter.

## Integracion con Laravel

1. Laravel invoca endpoints internos versionados (`/v1/internal/...`).
2. Laravel envia `x-internal-api-key`, `x-request-id`, `x-trace-id`.
3. Laravel consume contratos canonicos estables, sin acoplarse al broker.
4. Laravel decide broker por routing interno (parametro `:broker`) segun negocio LIS.

## Evolucion a microservicios de adapters (Alternativa B)

Estrategia incremental sin romper contrato:

1. Mantener este gateway como facade estable hacia Laravel.
2. Extraer adapter X a microservicio dedicado conservando contrato canonico interno.
3. Reemplazar implementacion local del adapter por cliente HTTP/gRPC/queue al microservicio.
4. Mantener `BrokerAdapterFactory` como punto de resolucion unificado.
5. Aplicar despliegues independientes por broker cuando el volumen o complejidad lo requiera.

## Ejemplos request/response

Request elegibilidad:

```json
{
  "requestId": "REQ-ELIG-001",
  "traceId": "TRC-ELIG-001",
  "memberId": "123456",
  "documentType": "DNI",
  "documentNumber": "30123456",
  "payerCode": "OSDE",
  "planCode": "210",
  "serviceDate": "2026-03-13"
}
```

Response elegibilidad (stub):

```json
{
  "requestId": "req-http-001",
  "traceId": "trc-http-001",
  "broker": "traditum",
  "eligible": true,
  "affiliateStatus": "active",
  "coveragePercent": 80,
  "message": "Stub response with timeout 5000ms",
  "brokerReferenceId": "TRA-ELIG-...",
  "brokerMetadata": {
    "name": "traditum",
    "version": "stub-v1",
    "environment": "stub",
    "capabilities": ["eligibility", "transaction", "statusQuery"]
  }
}
```

Response error normalizado:

```json
{
  "requestId": "req-http-001",
  "traceId": "trc-http-001",
  "error": {
    "code": "TIMEOUT",
    "message": "Traditum timeout (stub)",
    "details": {
      "broker": "traditum"
    }
  }
}
```
