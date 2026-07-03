# API - Administracion de Usuarios (Front)

Base URL: `http://localhost:8000/api`

Headers requeridos:
- `Authorization: Bearer <cognito_jwt>`
- `X-Tenant-ID: <tenant_id>`

## Tipo de token por ruta

- `/api/auth/me`: usar `ID token`
- `/api/v1/me`: usar `ID token`
- Rutas operativas `/api/v1/admin/*`, `/api/v1/jobs/*`, `/api/v1/broadcast/*`: usar `Access token`

## 1) Listar usuarios

- Metodo: `GET`
- Endpoint: `/api/v1/admin/users`
- Permiso: `users.manage`

Query params opcionales:
- `search` (nombre/email/sub)
- `status` (`pending | authorized | blocked`)
- `role` (nombre de rol)
- `per_page` (1..100)

Ejemplo:
```bash
curl -X GET "http://localhost:8000/api/v1/admin/users?search=camila&status=pending&per_page=10" \
  -H "Authorization: Bearer <token>" \
  -H "X-Tenant-ID: 1"
```

Respuesta ejemplo:
```json
{
  "data": [
    {
      "id": 12,
      "sub": "cognito-sub-123",
      "name": "Camila Acosta",
      "email": "camila.acosta@clinic-demo.com",
      "status": "pending",
      "login_provider": "google",
      "roles": [],
      "first_login_at": "2026-03-08T12:00:00+00:00",
      "last_login_at": "2026-03-09T09:44:00+00:00",
      "authorized_at": null,
      "blocked_at": null,
      "created_at": "2026-03-08T12:00:00+00:00"
    }
  ],
  "meta": {
    "current_page": 1,
    "per_page": 10,
    "last_page": 1,
    "total": 1
  }
}
```

## 2) Cambiar estado del usuario

- Metodo: `PATCH`
- Endpoint: `/api/v1/admin/users/{userId}/status`
- Permiso: `users.manage`

Body DTO:
```json
{
  "status": "authorized"
}
```

Valores validos para `status`:
- `pending`
- `authorized`
- `blocked`

Ejemplo:
```bash
curl -X PATCH "http://localhost:8000/api/v1/admin/users/12/status" \
  -H "Authorization: Bearer <token>" \
  -H "X-Tenant-ID: 1" \
  -H "Content-Type: application/json" \
  -d "{\"status\":\"blocked\"}"
```

Respuesta ejemplo:
```json
{
  "data": {
    "id": 12,
    "sub": "cognito-sub-123",
    "name": "Camila Acosta",
    "email": "camila.acosta@clinic-demo.com",
    "status": "blocked",
    "login_provider": "google",
    "roles": [],
    "first_login_at": "2026-03-08T12:00:00+00:00",
    "last_login_at": "2026-03-09T09:44:00+00:00",
    "authorized_at": "2026-03-09T10:00:00+00:00",
    "blocked_at": "2026-03-09T10:30:00+00:00",
    "created_at": "2026-03-08T12:00:00+00:00"
  }
}
```

## 3) Actualizar roles del usuario (reemplaza roles actuales)

- Metodo: `PATCH`
- Endpoint: `/api/v1/admin/users/{userId}/roles`
- Permiso: `users.manage`

Body DTO:
```json
{
  "roles": ["recepcion", "bioquimica"]
}
```

Ejemplo:
```bash
curl -X PATCH "http://localhost:8000/api/v1/admin/users/12/roles" \
  -H "Authorization: Bearer <token>" \
  -H "X-Tenant-ID: 1" \
  -H "Content-Type: application/json" \
  -d "{\"roles\":[\"recepcion\",\"bioquimica\"]}"
```

Respuesta ejemplo:
```json
{
  "data": {
    "id": 12,
    "sub": "cognito-sub-123",
    "name": "Camila Acosta",
    "email": "camila.acosta@clinic-demo.com",
    "status": "authorized",
    "login_provider": "google",
    "roles": ["recepcion", "bioquimica"],
    "first_login_at": "2026-03-08T12:00:00+00:00",
    "last_login_at": "2026-03-09T09:44:00+00:00",
    "authorized_at": "2026-03-09T10:00:00+00:00",
    "blocked_at": null,
    "created_at": "2026-03-08T12:00:00+00:00"
  }
}
```

## 4) Listar roles del tenant

- Metodo: `GET`
- Endpoint: `/api/v1/admin/roles`
- Permiso: `roles.read`

Ejemplo:
```bash
curl -X GET "http://localhost:8000/api/v1/admin/roles" \
  -H "Authorization: Bearer <token>" \
  -H "X-Tenant-ID: 1"
```

Respuesta ejemplo:
```json
{
  "data": [
    { "id": 1, "name": "admin", "guard_name": "web" },
    { "id": 2, "name": "recepcion", "guard_name": "web" },
    { "id": 3, "name": "bioquimica", "guard_name": "web" }
  ]
}
```

## 5) Endpoint de sesion para Front (actualizado)

- Metodo: `GET`
- Endpoint: `/api/v1/me`

Ahora incluye tambien:
- `user.status`
- `user.login_provider`

Respuesta ejemplo:
```json
{
  "user": {
    "id": 99,
    "sub": "admin-sub",
    "email": "admin@clinic-demo.com",
    "name": "Admin User",
    "status": "authorized",
    "login_provider": "google"
  },
  "tenant": {
    "id": 1,
    "slug": "demo"
  },
  "roles": ["admin"],
  "permissions": ["users.manage", "roles.read"]
}
```

## Errores esperables para Front

Formato base:
```json
{
  "error": {
    "code": "ValidationException",
    "message": "Validation failed",
    "correlation_id": "..."
  }
}
```

Casos comunes:
- `401 InvalidToken` (token invalido / usuario no autorizado)
- `400 TenantNotResolved` (falta o invalido `X-Tenant-ID`)
- `422 ValidationException` (payload invalido, rol inexistente, status invalido)
- `403 Forbidden` (sin permisos `users.manage` o `roles.read`)
