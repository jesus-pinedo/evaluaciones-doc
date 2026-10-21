# MEJ-004 / P4 — Consulta y descarga individual de evaluaciones HF-011

**Proyecto:** PROING — Módulo de Evaluación de Desempeño
**Fecha de aprobación funcional:** 2026-10-20
**Estado:** Diseño funcional aprobado; implementación y pruebas pendientes.
**Origen:** Revisión visual del dashboard «Seguimiento por Evaluador» y su modal: localizar la evaluación de un empleado requiere conocer su evaluador, lo que dificulta la búsqueda y descarga individual.

## 1. Decisión aprobada

Crear una **segunda vista dentro de Reportes**, llamada **«Consulta de Evaluaciones»**, independiente de la presentación agrupada por evaluador. Conservar **«Seguimiento»** (dashboard actual) sin cambios funcionales. El usuario autorizado podrá buscar una evaluación por empleado, independientemente de quién la evaluó, y descargar su PDF cuando esté `COMPLETED`.

Flujo: **Reporte → Consulta de Evaluaciones → seleccionar periodo → buscar empleado → descargar PDF de esa evaluación**.

El botón «Reenviar» presente en el dashboard actual fue colocado para un futuro flujo de correo/acuse, pero no tiene funcionalidad; **no reutilizarlo como descarga** ni implementar envíos en P4. Mantenerlo sin cambios; cualquier renombrado o habilitación requiere un alcance distinto.

## 2. Alcance UI aprobado

- Mostrar dos vistas o pestañas claramente diferenciadas bajo Reportes: **Seguimiento** (existente) y **Consulta de Evaluaciones** (nueva). No crear un módulo independiente de navegación principal ni rediseñar el dashboard.
- Nueva consulta con: selector de **periodo publicado** (históricos incluidos), búsqueda por **nombre/apellido del empleado o identificación** (si ese dato está disponible y autorizado), filtro de **estado de evaluación**, y tabla paginada.
- Tabla orientada a cada evaluación individual, con al menos: empleado, evaluador, estado de evaluación, estado de acuse, periodo consultado (si no está claro en encabezado) y acción **«Descargar PDF»**. Otros campos solo si ya están disponibles en DTO o su incorporación mínima resulta justificada.
- Distinguir explícitamente **`COMPLETED`** de **acuse confirmado/pendiente**: una evaluación completada puede tener acuse pendiente; el botón PDF depende de completada, no de acuse confirmado.
- Descargar exclusivamente el PDF de la evaluación seleccionada por su **evaluationId**; no deducirlo del evaluador ni del periodo vigente implícito.
- Estados de carga, error y sin resultados; paginación en servidor; limpiar datos y descartar respuestas tardías al cambiar de periodo, filtro, búsqueda o página.
- Selección inicial de periodo **consistente con P3**: publicado vigente por fecha Bogotá; si no existe, publicado más reciente. Si la navegación conserva el `periodId` de Seguimiento, validar que sea publicado y no asumir permisos implícitos.

## 3. Recursos existentes a verificar antes de programar

Documentados, pero **no equivalen a una verificación de código en este paso**:
- `GET /api/employee-evaluations/reports/evaluations`: consulta paginada con `periodId` (P3), `evaluatorId`, `status`, `search`, `page`, `limit`. Confirmar si **omitir `evaluatorId` permite la búsqueda global por empleado** y si `search` incluye identificación del evaluado; si no, ajustar únicamente lo mínimo en `reports/`.
- `GET /api/employee-evaluations/evaluations/:id/pdf`: generación individual existente con Puppeteer y vista `EvaluationPrintPage.tsx`. Comprobar ruta, contrato, autenticación, autorización, descarga y contenido; **reutilizar**, no crear otro motor PDF.
- `GET /api/employee-evaluations/periods`: fuente actual del selector publicado P3; verificar los permisos y el campo `status` `DRAFT`/etc.

## 4. Control de acceso — precondición obligatoria

La documentación técnica de endpoints (auditoría previa) advierte que la ruta PDF podría **no validar ownership/acceso por evaluación** y que existen dudas de middleware efectivo de autenticación. **Antes de exponer una descarga desde la vista de Gestión Humana, verificar y asegurar autenticación y autorización en el servidor** para la consulta de resultados individuales y la descarga PDF. Ocultar un botón React no constituye autorización.

- Restringir la consulta global y la descarga a roles/permisos reales de consulta de evaluaciones de Gestión Humana; no inventar códigos de rol ni exponer todas las evaluaciones a evaluadores sin facultad.
- Verificar que un usuario sin permisos no pueda consultar ni descargar evaluaciones arbitrarias cambiando `periodId`, `evaluationId` o `evaluatorId`; evitar IDOR.
- Reutilizar el esquema de seguridad ya existente y hacer el menor ajuste imprescindible en rutas/servicios de evaluaciones si la descarga carece del control debido.
- **Si no se puede determinar con certeza el modelo de autorización**, detener la exposición de consulta/descarga y solicitar una decisión técnica explícita. No habilitar un endpoint inseguro.

## 5. Alcance excluido

No modificar P1, P2 ni P3 existente salvo navegación mínima entre vistas. No recalcular score ni actualizar evaluaciones históricas 2025; no tocar respuestas, snapshots, periodos o ponderaciones. No cambiar diseño/composición de PDF ni notificaciones, correo, firma/acuse o «Reenviar». No nuevas migraciones, reportes de desempeño, filtros de proyecto/centro de costo ni refactorización amplia. No crear commits automáticamente en repos locales.

## 6. Implementación por etapas

**P4.1 — Verificación técnica dirigida + contrato concreto** de endpoints, paginación, búsqueda por empleado, autorización y PDF. Producir resultado; si seguridad/contrato ya permiten implementar sin ambigüedad, continuar a P4.2 en el mismo encargo.

**P4.2 — Implementación mínima** de segunda vista dentro de Reportes, búsqueda/paginación/filtros, descarga autenticada para `COMPLETED`, reutilizando PDF existente. Ajustar consulta o permiso backend solo si es imprescindible y está justificado.

**P4.3 — Validación integrada y manual** con datos sintéticos/autorizados; demostrar búsqueda sin evaluador, PDF correcto por periodo/empleado, autorización servidor, histórico intacto y sin efectos secundarios.

## 7. Criterios de aceptación

1. Desde Reportes se puede alternar entre Seguimiento y Consulta de Evaluaciones sin perder el comportamiento del dashboard P3.
2. Un usuario con permisos adecuados busca por empleado sin conocer al evaluador, filtra por periodo, navega resultados paginados y distingue estado de evaluación del acuse.
3. Un registro `COMPLETED` habilita Descargar PDF; `PENDING` o `IN_PROGRESS` no permite documento final, incluso mediante URL directa, conforme a reglas del endpoint.
4. Cada archivo corresponde al `evaluationId`, empleado y periodo seleccionado, incluido histórico 2025; no se mezclan respuestas entre periodos.
5. La descarga no envía correos, no marca acuse ni modifica score, status, respuestas o snapshots.
6. Un acceso no autorizado o manipulación de IDs es rechazado por el servidor; no basta con ocultar el botón.
7. Typecheck/build, pruebas dirigidas de API/frontend y sesión real autorizada o aislada; documentar los obstáculos del entorno sin atribuir pruebas no realizadas.
8. Documentar evidencia de lo cambiado y dejar puntos manuales restantes sin cerrar falsamente.

## 8. Estado de la implementación

**Pendiente de Claude/Codex.** Este documento registra exclusivamente la decisión funcional aprobada y las condiciones técnicas de seguridad; no certifica cambios de código local.

## 9. Implementación — P4.1 + P4.2 (2026-10-20)

**Estado:** implementado según el diseño aprobado en las secciones 1-7. Ejecutado por Claude en este encargo. No se modificó P1/P2/P3 existente (Vista "Seguimiento" intacta en su lógica), no se generaron commits, no se ejecutaron migraciones, no se recalcularon scores ni se tocó el contenido del PDF ni correos/notificaciones.

### 9.1 Contratos verificados en código (P4.1)

1. `GET /reports/evaluations` (`api/src/modules/employee-evaluations/reports/reports.service.js#getEvaluationsList`): **ya permitía** omitir `evaluatorId` → consulta global, desde antes de este cambio. Confirmado leyendo el código (no solo la documentación).
2. `search` (antes de este cambio) solo buscaba `name/lastName1/lastName2` del empleado evaluado — **no incluía identificación**. `EmployeeProfile.documentNumber` (columna `rhdocumento`) existe en `prisma/schema.prisma`; se agregó al `OR` de búsqueda.
3. El DTO de `/reports/evaluations` ya incluía `evaluationId` (campo `id`), datos del empleado, `status` y `isConfirmed` (acuse), pero **no incluía al evaluador**. Se agregó `evaluator: { id, fullName }`. También se agregó `period` a la respuesta (mismo objeto que ya devuelve `dashboard-progress`), de forma aditiva, para que la vista nueva pueda mostrar qué periodo quedó seleccionado por defecto sin duplicar el criterio de resolución en el frontend.
4. `GET /evaluations/:id/pdf` (`evaluationPdf.service.js` + `evaluations.controller.js#getEvaluationPdf`): generaba el PDF para **cualquier** `evaluationId` existente, sin validar `status` ni ownership — el parámetro `user` se recibía pero no se usaba en ninguna verificación (ni en el controlador ni dentro de `generateEvaluationPdf`). Confirmado leyendo el código línea por línea.
5. Autenticación/autorización reales (verificado en código, no asumido de la documentación histórica):
   - `api/src/app.js:35` → `app.use(authenticate)` **está activo**: toda ruta bajo `/api` exige un JWT válido (`Authorization: Bearer`). Esto confirma lo que el roadmap ya anotaba ("el responsable ya descomentó el middleware"); el hallazgo de `seguridad-sso.md` sobre middleware desactivado es, efectivamente, una foto histórica.
   - Autorización por **rol/permiso**: antes de este cambio, no existía ningún uso de `req.user.roles` en todo `api/src` (búsqueda sin resultados). El JWT sí incluye `roles[].permissions[]` (`auth.service.js#exchangeCode`, alimentado por el SSO legado), y el frontend ya lo usa para ocultar rutas (`sisproing-app/src/auth/permissions.ts`, `ProtectedRoute module="employee_evaluations.reports" action="view"`), pero **solo en el cliente**. Esto significa que, antes de este cambio, cualquier usuario autenticado (sin importar su rol) podía:
     - Consultar **todas** las evaluaciones de la organización vía `/reports/evaluations` omitiendo `evaluatorId`.
     - Descargar el PDF de **cualquier** evaluación (completada o no) solo cambiando el `evaluationId` en la URL.
   - Esto confirma con certeza — no como sospecha — el riesgo que esta mejora (§4) pedía verificar antes de exponer la nueva vista.

### 9.2 Control de acceso implementado (precondición de seguridad §4)

Se reutilizó el modelo de roles/permisos **ya existente** en el JWT; no se inventó ningún rol ni código de permiso nuevo:

- **Nuevo:** `api/src/core/middlewares/permissions.middleware.js` — `userHasPermission(user, module, action)` y `requirePermission(module, action)`, que replican en el servidor la misma lógica que ya usa el frontend (`flattenPermissions` + `makeKey` de `sisproing-app/src/auth/permissions.ts`).
- `GET /reports/evaluations` ahora exige el permiso `employee_evaluations.reports#view` (el mismo permiso que ya protege la pestaña "Reportes" en el frontend) — `reports.route.js`. No se añadió el chequeo a `GET /reports/dashboard-progress` (Vista "Seguimiento"), por alcance y para no tocar sus funciones existentes; queda documentado como riesgo residual (ver 9.5.1).
- `GET /evaluations/:id/pdf` ahora exige, en el servidor:
  1. `evaluationId` entero positivo (400 si no).
  2. La evaluación debe existir y estar activa (404 si no).
  3. Autorización por recurso: permitido si el usuario tiene `employee_evaluations.reports#view` (consulta de Gestión Humana) **o** si es el evaluador de esa evaluación (`req.user.id === evaluation.evaluatorId`, el uso ya existente de "Mis Evaluaciones" → Imprimir). Cualquier otro usuario recibe `403 EVAL_PDF_FORBIDDEN`, incluso manipulando el id.
  4. Solo se genera el documento si `status === 'COMPLETED'` (`403 EVAL_PDF_NOT_READY` para `PENDING`/`IN_PROGRESS`, incluso por URL directa) — antes de este cambio esto **no** se validaba en el servidor, solo se ocultaba el botón "Imprimir" en el frontend.
- No se modificó la autenticación (`authenticate`) ni el contrato de login/SSO.

### 9.3 Archivos modificados

Backend (`api/src/`):
- `core/middlewares/permissions.middleware.js` (nuevo)
- `modules/employee-evaluations/reports/reports.route.js`
- `modules/employee-evaluations/reports/reports.service.js`
- `modules/employee-evaluations/evaluations/evaluations.service.js`
- `modules/employee-evaluations/evaluations/evaluations.controller.js`

Frontend (`sisproing-app/src/modules/employee-evaluations/`):
- `reports/ReportsLayout.tsx` (nuevo) — pestañas "Seguimiento" / "Consulta de Evaluaciones"
- `reports/pages/EvaluationsQueryPage.tsx` (nuevo) — Vista 2
- `reports/services/report.service.ts` — `getEvaluationsList`, `downloadEvaluationPdf`
- `reports/types/reportTypes.ts` — tipos de la nueva vista + `period` en la respuesta de `/reports/evaluations`
- `routes/index.tsx` — monta `ReportsLayout` con `EvaluationDashboardPage` (index) y `EvaluationsQueryPage` (`/consulta`)

No se tocó ningún archivo de P1/P2, ni `EvaluationDashboardPage.tsx`, `DashboardFilters.tsx`, `KPICards.tsx`, `Charts.tsx`, `EvaluatorTable.tsx`, `EvaluatorDetailModal.tsx`, `evaluationPdf.service.js`, `EvaluationPrintPage.tsx`, correo o notificaciones.

### 9.4 Comportamiento implementado

- **Vista 2 "Consulta de Evaluaciones"** (pestaña dentro de `/employee-evaluations/reports/consulta`):
  - Selector de periodo publicado (histórico incluido), mismo criterio de valor por defecto que P3 (vigente por fecha de Bogotá; si no existe, el publicado más reciente).
  - Búsqueda por nombre/apellido/identificación del empleado **evaluado** (no filtra ni requiere conocer al evaluador).
  - Filtro por estado (`PENDING`/`IN_PROGRESS`/`COMPLETED`/todos).
  - Tabla paginada en servidor (15 por página): empleado, evaluador, estado, acuse, acción.
  - Botón «Descargar PDF» habilitado solo si `status === 'COMPLETED'` (deshabilitado visualmente para los demás estados, y rechazado también en el servidor).
  - Descarta respuestas HTTP obsoletas al cambiar periodo/filtro/búsqueda/página (mismo patrón `requestSeq` que `EvaluationDashboardPage`).
  - La descarga usa `Authorization: Bearer` vía `apiFetch` (nunca token en URL) y dispara un `<a download>` sobre el blob recibido, igual criterio que "Mis Evaluaciones" → Imprimir.
  - El botón «Reenviar» no se tocó ni se asoció a esta descarga (sigue sin funcionalidad, fuera de alcance).
- **Vista 1 "Seguimiento":** sin cambios de lógica, estado ni llamadas. Se envolvió en un layout de pestañas (`ReportsLayout`) que no modifica su comportamiento; ver el único efecto colateral visual conocido en 9.5.3.

### 9.5 Riesgos y pendientes

1. **`GET /reports/dashboard-progress` sigue sin exigir el permiso explícitamente en el servidor.** Comparte el mismo problema de fondo que tenía `/reports/evaluations` (cualquier usuario autenticado podía invocarlo), pero queda fuera del alcance mínimo de P4 — que pedía asegurar "la consulta global y la descarga individual" reutilizadas por la nueva vista — y del mandato expreso de no modificar las funciones existentes de Vista 1. **Se recomienda una decisión explícita** para extender `requirePermission('employee_evaluations.reports','view')` también a ese endpoint en una intervención posterior, ya que la Vista 1 ya exige ese mismo permiso en el frontend (por lo que no debería romper a ningún usuario legítimo).
2. **`GET /evaluations/:id` y el flujo de edición/finalización** (`getEvaluationById`/`saveDraft`/`finalizeEvaluation`) tampoco validan ownership en el servidor (un evaluador autenticado podría, cambiando el id, ver/editar una evaluación que no es la suya). Es un hallazgo preexistente, ya documentado en `seguridad-sso.md`, y confirmado aquí en código. **No se tocó** porque está fuera del alcance de P4 ("sin cambios en otros módulos") y una corrección ahí requiere una decisión propia, dado el riesgo de afectar el flujo real de "Mi equipo" sin una validación dedicada.
3. **Efecto visual menor en Vista 1:** `EvaluationDashboardPage.tsx` usa `h-[calc(100vh-4rem)]` asumiendo ser la vista de nivel superior; al envolverla en `ReportsLayout` (que agrega una barra de pestañas), puede aparecer un pequeño exceso de alto o doble scroll. No afecta datos ni funciones; pendiente de confirmación visual en navegador.
4. **No se pudo ejecutar la suite Jest del backend** (`npm test`): mismo problema preexistente ya documentado en MEJ-003 (`ERR_PACKAGE_PATH_NOT_EXPORTED` de `jest-circus`), reconfirmado en este entorno; no relacionado con este cambio ni con este encargo.
5. **No se pudo ejecutar `vite build`** (bundling de producción): mismo problema preexistente ya documentado en MEJ-003 (falta el binario opcional `@rollup/rollup-linux-arm64-gnu`). `tsc -b` (typecheck completo del proyecto, con los archivos nuevos incluidos) **sí se ejecutó y pasó sin errores** antes de llegar a ese paso.
6. **No se ejecutaron pruebas contra base de datos real.** No se insertaron datos sintéticos ni se corrió el flujo end-to-end contra PostgreSQL en este encargo. Lo verificado de forma ejecutable en esta sesión: sintaxis (`node --check`) de los cinco archivos backend tocados/nuevos; typecheck completo de frontend (`tsc -b`, sin errores); ESLint dirigido sobre `reports/` (sin errores nuevos; los 2 preexistentes ya documentados en MEJ-003 — `Charts.tsx` y `EvaluatorDetailModal.tsx` — siguen intactos y no se tocaron); y pruebas manuales (sin framework, por la falla de Jest) de `userHasPermission`/`requirePermission` cubriendo: usuario con el permiso, usuario sin el permiso, usuario sin `roles`, y request sin `req.user` — las cuatro combinaciones devolvieron el resultado esperado (ver salida de consola de esta sesión).
7. Pendiente la validación manual con datos reales o sintéticos autorizados (checklist 9.6).

### 9.6 Checklist de validación manual pendiente

1. Buscar un empleado sin conocer al evaluador — verificado por código (`evaluatorId` sigue sin ser obligatorio); **pendiente** con datos reales en navegador.
2. Buscar por nombre, apellido e identificación — verificado por código (se agregó `documentNumber` al `OR` de búsqueda); **pendiente** con datos reales (depende del formato real de `rhdocumento`).
3. Consultar evaluaciones de 2025 y otro periodo sin mezclar resultados — reutiliza `resolvePeriodForReports`, ya validado en P3; **pendiente** reconfirmar específicamente desde la vista nueva.
4. Confirmar paginación y filtros — implementado (`page`/`limit`/`status`/`search`); **pendiente** con datos reales.
5. Descargar el PDF correcto por `evaluationId` — reutiliza el generador existente sin cambios en su contenido; **pendiente** con datos reales.
6. No habilitar PDF final para evaluaciones incompletas — verificado por código en servidor (`403 EVAL_PDF_NOT_READY`) y en UI (botón deshabilitado); **pendiente** probar con una evaluación real `PENDING`/`IN_PROGRESS`.
7. Descargar una evaluación completada con acuse pendiente — el botón no depende de `isConfirmed`, solo de `status`; **pendiente** con datos reales.
8. Rechazar en servidor consultas y descargas de usuarios no autorizados, incluso manipulando IDs — verificado por lógica (pruebas manuales de `userHasPermission`/`requirePermission` + revisión línea por línea de `getEvaluationPdf`); **pendiente** probar con dos usuarios reales de distinto rol/equipo.
9. Mantener intactos P1, P2 y P3 — no se modificó ningún archivo de esos alcances, salvo el envoltorio de pestañas sobre la Vista 1 (ver riesgo 9.5.3); **pendiente** confirmación visual.
10. Confirmar que descargar no modifica respuestas, scores, estados ni snapshots — el flujo de descarga es de solo lectura (`getEvaluationAccessInfo` y `generateEvaluationPdf` no escriben en base de datos); **pendiente** confirmación con datos reales (comparar antes/después).

**Estado P4:** implementado dentro de los límites establecidos (sin refactoring amplio, sin cambios en P1-P3, sin nuevo generador de PDF, sin recálculos ni migraciones, sin commits automáticos). Queda pendiente la validación manual en navegador/API con datos reales o sintéticos autorizados, y las dos decisiones de alcance señaladas en 9.5.1 y 9.5.2 (permiso explícito en `dashboard-progress`, y ownership en `/evaluations/:id` y su flujo de edición).
