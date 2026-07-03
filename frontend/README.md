# Arenco LIS Monorepo

Monorepo con `pnpm workspaces` + `turborepo` para separar apps y paquetes compartidos.

## Estructura

```text
apps/
	lis/                # App actual de LIS en Next.js
	lis-web/            # Placeholder de nomenclatura
	display-player/     # Scaffold base para sistema de pantallas (Raspberry)
packages/
	api-client/         # Cliente HTTP reusable
	types/              # Tipos de dominio compartidos
	ui/                 # Componentes/proveedores UI reutilizables
	config/             # Constantes/config compartible
	utils/              # Utilidades genericas
```

## Comandos

```bash
pnpm install
pnpm dev
pnpm build
pnpm lint
pnpm test
```

Comandos por workspace:

```bash
pnpm --filter @arenco/lis dev
pnpm --filter @arenco/lis build
```

## Convencion de imports internos

- App LIS: `@arenco/lis/*`
- Paquetes compartidos: `@arenco/*`

## Notas de migracion

- La app original se movio de la raiz a `apps/lis`.
- Extraccion conservadora aplicada:
- `@arenco/api-client`: capa HTTP base.
- `@arenco/types`: tipos de catalogos y administracion de usuarios.
- `@arenco/ui`: `AppProviders`.
- `@arenco/config` y `@arenco/utils`: constantes y helpers de entorno.
