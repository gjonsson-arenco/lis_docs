# LIS Frontend Architecture

## 1. Overview

This repository is a `pnpm` + `turborepo` monorepo.

- Main app: `apps/lis` (Next.js App Router)
- Secondary apps: `apps/display-player`, `apps/lis-web` (scaffold/placeholder)
- Shared packages:
	- `packages/api-client`: reusable HTTP client + error model
	- `packages/types`: shared domain types
	- `packages/ui`: shared app providers
	- `packages/config`: shared constants
	- `packages/utils`: shared env helpers

Root orchestration:

- `pnpm-workspace.yaml`: workspaces under `apps/*` and `packages/*`
- `turbo.json`: task graph for `dev`, `build`, `lint`, `test`, `typecheck`

## 2. Runtime Stack (apps/lis)

- Framework: Next.js `16.1.x` + React `19`
- UI: Mantine + Tabler Icons
- Async/cache state: TanStack React Query
- Local UI/domain state: Zustand
- Forms/validation: React Hook Form + Zod
- Testing: Vitest + Testing Library

## 3. Frontend Routing and Composition

### 3.1 Root layout

- File: `apps/lis/src/app/layout.tsx`
- Responsibilities:
	- Registers fonts and global styles
	- Injects Mantine color scheme script
	- Wraps app in `AppProviders`

### 3.2 Route groups

- `(auth)` group: auth entry route (renders login client)
- `login` route: direct login page
- `auth/callback`: OAuth callback handler page
- `(workspace)` group: protected application shell with sidebar + content area

Files:

- `apps/lis/src/app/(auth)/page.tsx`
- `apps/lis/src/app/login/page.tsx`
- `apps/lis/src/app/auth/callback/page.tsx`
- `apps/lis/src/app/(workspace)/layout.tsx`

### 3.3 Protected shell

- `apps/lis/src/app/(workspace)/layout.tsx`
	- Wraps content with `ProtectedRoute`
	- Renders responsive `AppSidebar`
	- Manages collapse state for small vs large screens

## 4. Providers and Global Context

- `apps/lis/src/providers/app-providers.tsx` re-exports shared providers from `@arenco/ui`
- `packages/ui/src/app-providers.tsx` defines the provider composition:
	- `QueryClientProvider` (single `QueryClient` instance)
	- `MantineProvider`

This makes data-fetching and UI theme available app-wide from the root layout.

## 5. Authentication Architecture

Core files:

- `apps/lis/src/auth/authConfig.ts`
- `apps/lis/src/auth/authService.ts`
- `apps/lis/src/routes/ProtectedRoute.tsx`

### 5.1 Config

- `authConfig.ts` resolves required Cognito env vars via shared utils.
- Supports legacy fallback env names (`VITE_*`) for compatibility.
- Exposes helper builders:
	- `buildAuthorizeUrl`
	- `buildTokenUrl`
	- `buildLogoutUrl`

### 5.2 Token/session lifecycle

- Tokens are stored in browser `localStorage`:
	- `access_token`
	- `id_token`
	- `refresh_token`
- `validateSessionWithBackend` calls auth backend `/api/auth/me` to map user + tenant + permissions.
- `logout` clears tokens and redirects to Cognito logout URL.

### 5.3 Route protection

- `ProtectedRoute` checks `getAccessToken()` on client.
- If missing token, browser is redirected to `/login`.

## 6. API/HTTP Layer

### 6.1 Shared transport

- `packages/api-client/src/create-http-client.ts` is the core transport wrapper.
- Features:
	- Axios instance creation
	- Optional bearer token injection via `getAuthToken`
	- Request config mapping (`headers`, `params`, `signal`)
	- Normalized errors as `HttpError`

### 6.2 App-level API clients

- `apps/lis/src/services/http/api-clients.ts`
- Declares logical clients:
	- `core`
	- `auth`
	- `catalog`
- Base URLs are read from public env vars.
- `core`/`catalog` include bearer token + `X-Tenant-ID` default header.

### 6.3 Domain APIs

Under `apps/lis/src/services/apis/`:

- `auth.api.ts`: email login + SSO start (with mock fallback)
- `settings-catalogs.api.ts`: CRUD for admin catalogs
- `users-administration.api.ts`: list/update users and tenant roles
- `team-members.api.ts`: team members fetch with mock fallback

Pattern:

- Domain modules call `apiClients.*`
- DTOs are mapped to app-level types where needed

## 7. State Management Strategy

Zustand stores in `apps/lis/src/store/`:

- `app-store.ts`
	- App metadata and logged user snapshot
	- Session helper actions (`setLoggedUser`, `clearLoggedUser`)
- `settings-catalogs-store.ts`
	- UI state for selected catalog, editor modal, filters
- `chat-store.ts`
	- In-memory chat messages, send/reaction actions

Guideline currently used:

- React Query handles server state
- Zustand handles client-local and interaction state

## 8. Feature Module Shape

Feature pages are mostly built via `*-client.tsx` components under `apps/lis/src/components/`.

Example: `settings-catalogs-client.tsx`

- Uses React Query for list + mutations
- Uses Zustand for local page orchestration
- Uses Mantine for UI composition

This pattern appears across workspace feature areas (admission, cashier, chat, users, settings, etc).

## 9. Internal Import Conventions

TypeScript path aliases (see `apps/lis/tsconfig.json`):

- App-local: `@arenco/lis/*` -> `apps/lis/src/*`
- Shared packages imported as:
	- `@arenco/api-client`
	- `@arenco/config`
	- `@arenco/types`
	- `@arenco/ui`
	- `@arenco/utils`

This keeps feature code decoupled from relative path depth and enables package extraction.

## 10. Practical Mental Model

When touching a feature, think in this order:

1. Route + page shell (`app/...`)
2. Feature client component (`components/...-client.tsx`)
3. API module (`services/apis/...`)
4. Shared transport (`services/http` + `@arenco/api-client`)
5. Local state (`store/...`) if UI orchestration is needed

This is the current baseline architecture as of 2026-03-10.
