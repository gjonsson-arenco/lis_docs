# LIS Web App

Aplicacion principal LIS en Next.js (migrada desde la raiz del repo).

## Scripts

```bash
pnpm --filter @arenco/lis dev
pnpm --filter @arenco/lis build
pnpm --filter @arenco/lis test
```

## Variables de entorno

Mantiene el mismo contrato de variables que tenia la app original (`NEXT_PUBLIC_API_*`, Cognito y auth flags).

1. Crear `apps/lis/.env.local` a partir de `apps/lis/.env.example`.
2. Completar los valores reales de Cognito y de las APIs.

Ejemplo en PowerShell:

```powershell
Copy-Item .\apps\lis\.env.example .\apps\lis\.env.local
```

Variables de Cognito requeridas para iniciar la app sin errores:

- `NEXT_PUBLIC_COGNITO_DOMAIN`
- `NEXT_PUBLIC_COGNITO_CLIENT_ID`
- `NEXT_PUBLIC_COGNITO_REDIRECT_URI`
- `NEXT_PUBLIC_COGNITO_LOGOUT_URI`
