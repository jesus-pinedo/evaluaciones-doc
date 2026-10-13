# Evidencias inspeccionadas

## Documentación previa

- `evaluaciones-docs/AGENTS.md`
- `evaluaciones-docs/README.md`
- `evaluaciones-docs/funcional-contex.md`

No existía `evaluaciones-docs/docs/` antes de esta auditoría.

## Backend `api/`

### Dominio de evaluaciones

- `src/modules/employee-evaluations/index.js`
- rutas, controladores y servicios de `configurations/classifications/`, `groups/`, `questions/`, `scoring/`, `strength/` y `weaknesses/`
- `periods/periods.route.js`, `periods.controller.js`, `periods.service.js`
- `periods/snapshots/periodSnapshots.service.js`
- `evaluations/evaluations.route.js`, `evaluations.controller.js`, `evaluations.service.js`, `evaluations.helper.js`, `evaluationPdf.service.js`
- `reports/reports.route.js`, `reports.controller.js`, `reports.service.js`
- `schedulers/daily-reminder.scheduler.js`

### Datos y montaje

- `prisma/modules/evaluations.prisma`
- modelos relacionados de `prisma/modules/human-resources.prisma`
- fragmento de evaluaciones de `prisma/schema.prisma`
- migraciones de evaluación `20251124164050_init_evaluation_schema`, `20251125133830_add_score_scale_models` y `add_check_constraint.sql`
- `src/app.js`, `src/server.js`, `src/routes/index.js`, `src/routes/public.routes.js`

### Seguridad y notificaciones usadas por evaluaciones

- `src/core/middlewares/auth.middleware.js`
- `src/modules/auth/auth.routes.js`, `auth.controller.js`, `auth.service.js`
- `src/modules/notifications/queues/email.queue.js`
- `src/modules/notifications/queues/workers/main.worker.js`
- `src/modules/notifications/queues/workers/processors/index.js`, `evaluation.processor.js`, `evaluation-reminder.processor.js`
- `src/modules/notifications/templates/evaluation-completed.mjml`, `evaluation-reminder.mjml`
- `src/modules/notifications/services/token.service.js`, `mailer.service.js`, `template.service.js`
- `src/config/redis.js`

No se leyeron `.env`, secretos, datos de base ni procesadores de notificaciones ajenos, salvo nombres de registro necesarios para ubicar los procesadores de evaluación.

## Frontend `sisproing-app/`

### Dominio de evaluaciones

- `src/modules/employee-evaluations/routes/index.tsx`
- `layouts/ConfigurationLayout.tsx`
- servicios y páginas de `configurations/classifications`, `groups`, `questions`, `scoring`, `strengths` y `weaknesses` en las secciones de carga, validación y mutación
- `periods/periods.service.ts`, `CreatePeriodPage.tsx`, `PeriodsPage.tsx`
- `periods/components/EditPeriodForm.tsx`, `PeriodsItem.tsx`, `previewModal.tsx`
- `periods/hooks/usePreviewModal.ts`
- `evaluations/evaluations.service.ts`, `hooks/useEvaluationForm.ts`
- `evaluations/pages/EvaluationsPage.tsx`, `EvaluationsForm.tsx`, `EvaluationPrintPage.tsx`, `EvaluationAcknowledgement.tsx`
- componentes funcionales `QuestionSectionCard.tsx`, `StrengthsCard.tsx`, `OpportunitiesCard.tsx`, `EmployeeCard.tsx`, `FilterBar.tsx`, `TutorialAlert.tsx` y `AuditFooter.tsx`
- `reports/pages/EvaluationDashboardPage.tsx`, `services/report.service.ts`, `types/reportTypes.ts`
- componentes de reportes `KPICards.tsx`, `EvaluatorTable.tsx`, `EvaluatorDetailModal.tsx`, `DashboardFilters.tsx`, `Charts.tsx`

### Integración

- `src/app-routes.tsx`, `src/routes/ProtectedRoute.tsx`
- `src/auth/AuthProvider.tsx`, `permissions.ts`, `useAuth.ts`
- `src/modules/auth/services/auth.service.ts`, `src/types/auth.ts`
- `src/lib/httpClient.ts`, `src/services/public.service.ts`
- fragmentos de `src/components/SideMenu.tsx` y `src/shared/layauts/Layaut.tsx` usados por la navegación del módulo

## Exclusiones confirmadas

No se auditó el repositorio PHP `sisproing/` ni código funcional de inventario, ventas, compras, facturación, transportes, operaciones u otros módulos. No se ejecutaron aplicaciones, cron jobs, workers, correos, migraciones o consultas de datos.
