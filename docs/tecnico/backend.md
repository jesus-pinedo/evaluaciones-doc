# Backend

## Organización

`api/src/modules/employee-evaluations/`:

- `configurations/`: `classifications`, `groups`, `questions`, `strength`, `weaknesses`, `scoring`.
- `periods/`: CRUD parcial, publicación y `snapshots/periodSnapshots.service.js`.
- `evaluations/`: equipo, detalle, borrador, finalización, puntuación y PDF.
- `reports/`: dashboard, detalle y fuente de recordatorios.
- `schedulers/`: cron diario.

El router se monta en `/api/employee-evaluations` desde `src/routes/index.js`. El acuse se expone en `/api/public` desde `src/routes/public.routes.js`.

## Servicios principales

### Periodos

`periods.service.js` implementa validación de fechas, solapamiento, borrador, publicación, consulta y actualización. `publishPeriod` es el orquestador transaccional.

Aspectos relevantes:

- permite más de un periodo publicado si no se solapan;
- `findDraftPeriod` falla si existen varios borradores, pero `createPeriod` no evita crearlos;
- `getActivePeriodOrThrow` filtra solo `isActive=true` y ordena por inicio, sin validar la fecha;
- `update` exige en la práctica ambas fechas porque llama siempre a `parseAndValidateDates`, aunque el contrato diga actualización parcial;
- no hay ruta registrada para `remove` o `deactivate`.

### Evaluaciones

`evaluations.service.js`:

- `generateEvaluationsForPeriod`: crea asignaciones snapshot;
- `getMyTeamEvaluationsOverview` / `getMyTeamEvaluations`: resumen y lista;
- `getEvaluationById`: agrega snapshots y respuestas;
- `saveDraft`: upserts/reemplazos, score parcial y `IN_PROGRESS`;
- `finalizeEvaluation`: completitud de preguntas, score, `COMPLETED` y job de correo;
- `acknowledgeReceipt`: valida enlace y registra acuse.

No se comprueba evaluador propietario, rol, fechas ni correspondencia de IDs con el periodo. Los endpoints de lectura reciben cualquier ID.

### Puntuación

`evaluations.helper.js#calculateAndSaveScore` calcula el porcentaje sobre máximo fijo 5 y busca un rango snapshot. `formatEvaluationForFrontend` prepara `meta`, estructura del formulario y valores guardados.

### Reportes

`reports.service.js` carga todas las evaluaciones del primer periodo publicado encontrado y agrega en memoria. El dashboard carece de filtros backend por periodo/proyecto/centro de costo. La consulta detallada filtra por estado, evaluador y nombre.

## Catálogos

Los catálogos ofrecen CRUD, pero con inconsistencias:

- los `DELETE` son físicos aunque varios comentarios digan “lógico”;
- los modelos históricos snapshot protegen periodos ya publicados, pero borrar puede fallar por FK en relaciones globales;
- `createdBy` es obligatorio en varios modelos y `req.user` no está disponible actualmente;
- scoring usa nombres incompatibles (`minScore`, `min_values`, `minValue`) y un cliente Prisma inexistente (`scoringConfiguration`) para actualizar;
- `scoring.controller.create` declara parámetros `(res, req, next)` en orden incorrecto y `remove` llama a `service.delete`, que no se exporta.

La lectura de escala sí usa `scoreScale.findMany` y alimenta UI/publicación.

## Dependencias compartidas inspeccionadas

- `src/core/middlewares/auth.middleware.js`: verificación JWT.
- `src/modules/auth/`: intercambio SSO.
- `src/modules/employees/employees.service.js#getEmployeeSeniority`: solo llamado al preparar PDF; su implementación interna no fue auditada más allá de este contrato.
- `src/modules/notifications/queues/email.queue.js`: productores de evaluación.
- servicios `mailer`, `template` y `token` usados por esos productores.
- `src/shared/utils/date.utils.js#calculateRemainingDays`.

## Pruebas

No se encontraron pruebas `test/spec` específicas de `employee-evaluations` en `api/`. No se ejecutaron suites generales porque excederían el alcance delimitado.
