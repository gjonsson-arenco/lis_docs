# LIS Documentation

Documentación centralizada para el sistema LIS (Laboratory Information System). Este repositorio contiene toda la documentación técnica, guías de implementación y análisis arquitectónicos de los proyectos que componen el LIS.

## Estructura

### 📋 [Backend](./backend/)
Documentación del backend Laravel con integración de broker gateway.
- **README.md** - Descripción general del backend
- **docs/admission-valuations-api.md** - API de valuaciones de admisión
- **docs/orders-patients-api.md** - API de órdenes y pacientes
- **docs/settings-catalogs-api.md** - API de configuración y catálogos
- **docs/user-administration-api.md** - API de administración de usuarios
- **docs/valuation-pricing-process.md** - Proceso de valuación y precios

### 🎨 [Frontend](./frontend/)
Documentación del frontend monorepo con Next.js y Mantine UI.
- **README.md** - Descripción general del frontend
- **display-player-README.md** - Módulo display player
- **lis-app-README.md** - Aplicación LIS principal
- **docs/app-architecture.md** - Arquitectura de la aplicación
- **docs/admission-valuations-api.md** - Integración de valuaciones
- **docs/orders-patients-api.md** - Integración de órdenes
- **docs/settings-catalogs-api.md** - Integración de configuración
- **docs/user-administration-api.md** - Integración de usuarios

### 🔌 [Broker Gateway](./broker-gateway/)
Documentación del gateway broker NestJS con adaptador Swiss Medical.
- **README.md** - Descripción general del broker gateway
- **ARCHITECTURE_ANALYSIS.md** - Análisis arquitectónico del broker gateway
- **docs/COMPLETION_REPORT.md** - Reporte de implementación
- **docs/E2E_TESTING_GUIDE.md** - Guía de testing end-to-end
- **docs/SMG_IMPLEMENTATION.md** - Documentación de implementación Swiss Medical
- **docs/SWISS_MEDICAL_TESTING.md** - Guía de testing Swiss Medical
- **docs/endpoints.http** - Endpoints HTTP para testing
- **docs/web SWISS MEDICAL - API PRESTADORES - DOC TECNICA_v2 (1).docx** - Documentación técnica de Swiss Medical

### 🔬 [Clinical Matcher](./clinical-matcher/)
Documentación del motor de matching clínico basado en FastAPI.
- **README.md** - Descripción general del clinical matcher
- **MATCHING_PROCESS.md** - Proceso de matching clínico

## Proyectos del Workspace

| Proyecto | Descripción | Tecnología |
|----------|-------------|-----------|
| **lis-backend** | API REST con integración broker gateway | Laravel 11, PHP 8.3 |
| **lis-front-monorepo** | Frontend web con múltiples módulos | Next.js 14, TypeScript, Mantine UI |
| **lis-broker-gateway** | Gateway adapter para brokers de seguros | NestJS, TypeScript |
| **lis-clinical-matcher** | Motor de matching de datos clínicos | FastAPI, Python |

## Acceso a Repositorios

- Backend: https://github.com/gjonsson-arenco/lis_backend
- Frontend: https://github.com/gjonsson-arenco/lis_frontend
- Broker Gateway: https://github.com/gjonsson-arenco/lis_broker_gateway
- Clinical Matcher: https://github.com/gjonsson-arenco/lis_clinical_matcher
- Documentation: https://github.com/gjonsson-arenco/lis_docs

## Inicio Rápido

Para levantar todos los servicios:

```bash
# Desde la raíz del workspace
task "LIS: Levantar todos los servicios"
```

O de forma individual:

- **Backend**: `php artisan serve` en lis-backend/
- **Frontend**: `pnpm dev` en lis-front-monorepo/
- **Broker Gateway**: `pnpm start:dev` en lis-broker-gateway/
- **Clinical Matcher**: `python -m uvicorn app.main:app --reload` en lis-clinical-matcher/

## Notas

- Toda la documentación del proyecto está centralizada en este repositorio
- Para cambios en la documentación, actualizar aquí y luego sincronizar a los proyectos individuales si es necesario
- Mantener un estándar de nomenclatura y formato en toda la documentación
