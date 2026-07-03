# Arenco LIS - Laravel Backend Starter

Starter backend API para Arenco LIS sobre Laravel 12, sin dominio de negocio (sin branches/tickets/pacientes/órdenes).

## Qué incluye

- Laravel 12 API con versionado en `/api/v1/*`
- Docker dev environment (`app`, `nginx`, `mysql`, `redis`)
- Auth Cognito (middleware JWT + JWKS cache + endpoints `/api/auth/me` y `/api/v1/me`)
- Multi-tenancy infraestructura (tenant resolver + middleware + conexión tenant dinámica)
- RBAC con `spatie/laravel-permission` (teams por `tenant_id`)
- Queue infra lista para SQS + `DummyJob`
- Broadcasting/WebSockets con Laravel Reverb + `DummyBroadcastEvent`
- Seguridad/observabilidad base (CORS, rate limiting, correlation ID, JSON error format)
- OpenAPI/Swagger UI con Scramble (`/docs/api`)
- Tests feature mínimos base

## Requisitos

- Docker + Docker Compose
- GNU Make (opcional, también podés usar comandos `docker compose` directos)

## Setup local

1. Copiar variables de entorno:

```bash
cp .env.example .env
```

2. Levantar contenedores:

```bash
make up
```

3. Instalar dependencias dentro del contenedor:

```bash
make composer-install
```

4. Generar app key:

```bash
make artisan cmd="key:generate"
```

5. Ejecutar migraciones y seeders:

```bash
make migrate
make seed
```

La API queda en `http://localhost:8000`.

## Comandos útiles

- `make up`: levanta stack
- `make down`: baja stack
- `make artisan cmd="<comando>"`: ejecuta artisan
- `make migrate`: corre migraciones
- `make test`: ejecuta tests
- `make queue-worker`: inicia worker de colas
- `make reverb`: inicia servidor WebSocket Reverb

## Endpoints base

- `GET /api/v1/health` (público)
- `GET /api/auth/me` (requiere `auth.cognito`)
- `GET /api/v1/me` (requiere `tenant` + `auth.cognito`)
- `POST /api/v1/jobs/dummy` (requiere permiso `jobs.dispatch`)
- `POST /api/v1/broadcast/dummy` (requiere permiso `broadcast.send`)
- `GET /api/v1/admin/users` (requiere permiso `users.manage`)
- `PATCH /api/v1/admin/users/{id}/status` (requiere permiso `users.manage`)
- `PATCH /api/v1/admin/users/{id}/roles` (requiere permiso `users.manage`)
- `GET /api/v1/admin/roles` (requiere permiso `roles.read`)

OpenAPI UI:

- `GET /docs/api`

## Tenancy (infra)

Resolución de tenant:

1. Primary: header `X-Tenant-ID`
2. Fallback: subdominio (`{tenant}.<TENANCY_BASE_DOMAIN>`)

El middleware `tenant`:

- valida tenant activo en tabla central `tenants`
- configura conexión `tenant` dinámicamente en runtime
- inyecta contexto tenant en el request (`tenant_id`, `tenant_slug`)

Conexion central de tenants:

- `tenants` se resuelve usando `TENANCY_CENTRAL_CONNECTION`
- por defecto usa `DB_CONNECTION`
- para separar metadata central, usar `TENANCY_CENTRAL_CONNECTION=central`
- credenciales de `central`: `CENTRAL_DB_*`

## Cognito JWT

Variables mínimas en `.env`:

- `COGNITO_ISSUER`
- `COGNITO_CLIENT_ID`
- `COGNITO_USER_POOL_ID` + `COGNITO_REGION` (o `COGNITO_JWKS_URL`)
- `COGNITO_TOKEN_USE` (`id` por defecto)

Middleware `auth.cognito` valida:

- firma JWT con JWKS cacheado
- `iss`, `aud`/`client_id`, `exp`, `token_use`
- upsert de usuario central (`users`) por `cognito_sub`

## RBAC (Spatie)

- `teams` habilitado con `tenant_id`
- Seeder crea:
	- permisos: `jobs.dispatch`, `broadcast.send`
	- permisos admin: `users.manage`, `roles.read`
	- roles: `tenant-admin`, `tenant-viewer`
	- usuario demo: `demo@arenco.local`

`can:permission-name` está activo en rutas de ejemplo.

## Colas SQS

Config SQS ya disponible en `config/queue.php` + variables:

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_DEFAULT_REGION`
- `SQS_PREFIX`
- `SQS_QUEUE`
- `SQS_SUFFIX`

Para usar SQS:

- setear `QUEUE_CONNECTION=sqs`

`DummyJob` usa:

- retries: `tries=3`
- backoff: `10,30,60`

Recomendación producción: configurar DLQ en AWS SQS (no implementada en este starter).

## Broadcasting / WebSockets

- Driver: Reverb
- Evento dummy: `DummyBroadcastEvent`
- Canal: `tenant.{tenant_id}`

Flujo local:

1. `make reverb`
2. Disparar `POST /api/v1/broadcast/dummy`

## Seguridad y observabilidad

- CORS configurado en `config/cors.php`
- Rate limiting:
	- público por IP (`throttle:public`)
	- autenticado por user (`throttle:auth`)
- Correlation ID middleware:
	- lee/genera `X-Correlation-ID`
	- devuelve header en respuesta
	- agrega contexto a logs
- Errores API unificados:

```json
{
	"error": {
		"code": "...",
		"message": "...",
		"correlation_id": "..."
	}
}
```

## Tests

Correr:

```bash
make test
```

Cobertura mínima incluida:

- `/health` OK
- `/me` con token inválido falla
- tenant middleware: sin tenant falla, con tenant pasa
- encolar `DummyJob` devuelve 202

En tests, la verificación Cognito se mockea (sin llamadas externas a JWKS).
