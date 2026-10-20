# Hoja de ruta — Periodos, puntajes, reportes, PDF y reasignación

**Proyecto:** PROING — Evaluación de Desempeño HF-011  
**Fecha:** 2026-10-14  
**Estado:** plan priorizado; pendiente de análisis e implementación, una mejora a la vez  
**Alcance:** exclusivamente `api/src/modules/employee-evaluations/`, sus dependencias necesarias de notificaciones y `sisproing-app/src/modules/employee-evaluations/`. Este plan no autoriza revisar otros módulos de negocio.

## 1. Objetivo y decisiones de alcance

Evolucionar el módulo existente para (1) gestionar correctamente los periodos históricos y crear uno nuevo, (2) revisar y corregir los cálculos de puntuación, (3) consultar reportes de un periodo seleccionado y (4) descargar el PDF de una evaluación individual y (5) documentar, cuando se priorice, las condiciones del trigger ya existente y funcional que actualiza al evaluador.

- **Orden de prioridad acordado:** P1 → P2 → P3 → P4 → P5. P5 queda **pendiente de documentación técnica detallada del trigger existente**, con prioridad baja; no se adelanta a P1.
- **Estrategia:** documentar y resolver cada mejora antes de comenzar la siguiente; no implementar las cuatro en una sola intervención.
- **Fuera de alcance por ahora:** parches generales de seguridad y permisos, refactor global, módulos no relacionados, nuevos reportes compilados, rediseño visual completo y cambios en código PHP legado.
- **Actualización posterior a la auditoría:** el responsable del proyecto indica que **ya descomentó el middleware de autenticación**. Por tanto, el diagnóstico previo de `app.use(authenticate)` comentado es una foto histórica, **no una confirmación del estado actual**. No se programa una intervención general de seguridad en esta hoja de ruta. Cada cambio sí debe respetar los controles que ya existan y no ampliar inadvertidamente el acceso a datos.
- **Evidencia visual:** una captura del dashboard muestra el periodo 2025 con resultados ya cerrados de facto. Esta imagen ejemplifica el caso de uso histórico; no valida por sí sola cómo consulta los datos el API.

## 2. Prioridades y resultados

| Orden | ID | Mejora | Entregable |
|---|---|---|---|
| **P1** | MEJ-001 | Histórico y creación de periodos | Listado coherente de periodos, lectura de histórico y creación/publicación controlada de uno nuevo. |
| **P2** | MEJ-002 | Revisión y corrección de puntajes | Regla de cálculo aprobada e idéntica en React y Node; pruebas numéricas. |
| **P3** | MEJ-003 | Dashboard y consultas por periodo | Selector de periodo y estadísticas estrictamente asociadas al elegido. |
| **P4** | MEJ-004 | PDF individual | Descarga de evaluaciones completadas desde el detalle de Gestión Humana y consulta del evaluador, reutilizando el PDF existente. |
| **P5** | MEJ-005 | Documentar trigger existente de cambio de evaluador | **Confirmado por responsable:** trigger implementado y funcional. **Pendiente futuro:** especificar tablas, filtros de periodo/estado y comportamiento histórico. Sin cambios ahora. |

### Dependencia transversal

Desde **P1** debe quedar identificada la llave `periodId` como referencia para el resto del módulo, sin asumir que `isActive=true` equivale a «hoy dentro de la vigencia». **P3** aplicará esta referencia explícita a todo el dashboard.

## 3. P1 / MEJ-001 — Histórico y creación de periodos

### Problema y evidencia

La inspección estática documentó:
- `DRAFT`, `SCHEDULED`, `ACTIVE`, `FINISHED` son estados **derivados** de `isActive` y fechas; un vencido puede conservar `isActive=true`.
- El frontend consulta `FINALIZED` donde el backend devuelve `FINISHED`.
- `findDraftPeriod` presupone un borrador único, pero `createPeriod` no impide duplicados.
- El sistema crea snapshots de catálogos y asignaciones al **publicar**, no simplemente al crear el borrador.
- Algunas consultas escogen un «periodo activo» por bandera, sin acotar siempre por fecha.

Fuentes: `docs/funcional/reglas-negocio.md`, `docs/tecnico/backend.md`, `docs/tecnico/frontend.md`, `docs/auditoria/diferencias-requerimiento.md`.

### Comportamiento deseado para diseñar

- Listar todos los periodos con nombre, fechas, estado comprensible y acciones correspondientes.
- Poder abrir periodos anteriores en **modo de consulta**, preservando resultados y snapshots originales.
- Crear un periodo nuevo como borrador, revisarlo y publicarlo con validaciones; no reemplazar ni reutilizar datos de periodos pasados.
- Diferenciar claramente «publicado» de «vigente por fechas».
- Evitar que un borrador nuevo se mezcle con otro existente o que una publicación duplique asignaciones.
- Garantizar que crear/publicar un periodo futuro no modifique las evaluaciones históricas ni haga que el cron de recordatorios seleccione un periodo equivocado.
- Mostrar la vista previa histórica con **snapshots del periodo**, no con catálogos globales que pueden haber cambiado.

### Criterios de aceptación propuestos

1. Un periodo ya vencido sigue apareciendo en el histórico sin perder preguntas, respuestas, puntuaciones, asignaciones ni PDF.
2. El estado visible es consistente entre API y React; desaparece la divergencia `FINISHED` / `FINALIZED`.
3. Se crea y publica un nuevo periodo con fechas válidas, sin traslapes prohibidos ni duplicaciones.
4. El periodo nuevo tiene sus propias asignaciones y snapshots; las evaluaciones anteriores permanecen intactas.
5. Se distingue el borrador de los periodos publicados y se controla la existencia de borradores múltiples.
6. El flujo se prueba con al menos un periodo pasado y otro futuro, sin ejecutar notificaciones reales durante las pruebas.

### Por decidir al trabajar P1

- Política de uno o varios borradores simultáneos.
- Si «finalizado por fecha» debe seguir siendo estado calculado o generar cierre persistido.
- Qué acciones de corrección/prórroga/cierre se permiten en cada estado.
- Comportamiento exacto del sistema cuando no existe periodo vigente por fecha.
- Si el periodo histórico necesita UI adicional en «Periodos» o basta el listado actual ajustado.

### Rutas para analizar primero

- API: `periods/periods.service.js`, `periods/periods.route.js`, `periods/snapshots/periodSnapshots.service.js`, `evaluations/evaluations.service.js`, `schedulers/daily-reminder.scheduler.js` (solo selección de periodo).
- React: `periods/PeriodsPage.tsx`, `periods/CreatePeriodPage.tsx`, `periods/components/PeriodsItem.tsx`, `periods/components/previewModal.tsx`, `periods/periods.service.ts`.

## 4. P2 / MEJ-002 — Corrección del cálculo de puntajes

### Evidencia actual

La auditoría identifica cálculo tanto en:
- **Frontend:** `sisproing-app/src/modules/employee-evaluations/evaluations/hooks/useEvaluationForm.ts` (cálculo visual inmediato); revisar también las tarjetas de preguntas y cualquier vista previa.
- **Backend:** `api/src/modules/employee-evaluations/evaluations/evaluations.helper.js#calculateAndSaveScore` y el flujo de borrador/finalización de `evaluations.service.js`.

Fórmula LEGACY identificada en la auditoría original (la decisión ponderada posterior se registra más abajo):

`porcentaje_total = redondear_1_decimal((suma_puntos / (numero_preguntas * 5)) * 100)`.

En LEGACY, las preguntas sin respuesta cuentan en el máximo del periodo y no hay peso por grupo. El frontend también tiene un **porcentaje de avance del formulario** que considera fortalezas/debilidades y **no es el puntaje de desempeño**.

### Trabajo previsto

1. Localizar **todas** las operaciones de cálculo visual de puntaje, porcentaje por grupo, total, categoría y avance; no confundir estos conceptos.
2. Comparar con el cálculo persistido en Node y con la lógica de los reportes/impresión que consume ese valor.
3. Acordar la fórmula exacta de **cada grupo** y del **total**, tratamiento de preguntas sin responder, redondeo y escala.
4. **Decisión posterior aprobada:** grupos WEIGHTED usan peso numérico de snapshot (total de pesos 100 %), calculando subtotales sin redondeo intermedio; los históricos LEGACY mantienen su fórmula.
5. Corregir divergencias con pruebas de casos numéricos (grupos de distinto tamaño, respuestas parciales, todas completas y extremos de 1 a 5).
6. No recalcular ni sobrescribir silenciosamente los resultados históricos.

### Criterios de aceptación propuestos

- El puntaje que muestra el formulario coincide con el guardado en base y el mostrado en PDF/reporte para la misma evaluación.
- Los porcentajes por grupo y el total tienen fórmulas documentadas y pruebas.
- No se confunde «avance del formulario» con «resultado del desempeño».
- Las evaluaciones finalizadas históricas no se modifican sin una decisión explícita y un procedimiento auditado.

### Decisión pendiente esencial

**Resuelto:** solo se aplica la regla ponderada a periodos nuevos con pesos; **no recalcular ni actualizar** evaluaciones históricas de 2025. Cualquier corrección de puntajes existentes se realizará en otro proceso autorizado.


**Evidencia a revisar en P2 (2026-10-16):** la vista histórica 2025 muestra porcentajes integrados en los nombres de grupos, por ejemplo Calidad (10%), Actitud (15%), Innovación y Comunicaciones (5%) y Planeación (15%). Determinar si son texto descriptivo o ponderaciones requeridas; no cambiar cálculo ni datos históricos sin aprobar fórmula y alcance de corrección.

**Avance P2.1 (2026-10-20):** Codex reportó la persistencia nullable `weightPercentage` en `QuestionGroup`/`PeriodGroup`, migración SQL preparada sin ejecutar, CRUD y formulario React ajustados, copia del peso a snapshots nuevos y 12/12 pruebas técnicas aprobadas. **Pendiente:** aplicar migración solo en ambiente autorizado, verificar tablas/constraints y ejecutar checklist integrado. **No se han modificado puntajes ni snapshots históricos.** Ver [MEJ-002, sección 17](MEJ-002-calculo-puntuaciones.md#17-implementación-p21--persistencia-y-edición-del-peso-2026-10-20).

**Validación posterior de P2.1 (2026-10-20):** el responsable confirmó pruebas satisfactorias y aplicación de las columnas mediante SQL manual en PostgreSQL. La historia de Prisma sigue sin reconciliarse (P3005; migraciones anteriores pendientes), por lo que no corresponde ejecutar `migrate deploy` hasta resolverlo. **P2.2 se desarrolló después** y conserva scores `COMPLETED` del histórico. Ver [plan P2](MEJ-002-plan-ponderacion-grupos.md).

### Actualización de alcance P2 — 2026-10-20

El responsable decidió **no corregir ni recalcular puntajes históricos ahora**, incluyendo 2025; cualquier ajuste de valores persistidos será un proceso futuro independiente. El trabajo actual de P2 es incorporar **peso numérico por grupo** y posteriormente ajustar el cálculo únicamente para periodos nuevos, **sin refactoring**.

Orden acotado: **P2.1** modelo/catálogo/formulario/snapshot de peso (sin ejecutar migraciones productivas ni recalcular score); **P2.2** verificar fórmula del Excel HF-011 Rev9 y ajustar puntaje ponderado en Node y React para nuevos periodos con distinción explícita frente a legacy; **P2.3** pruebas y cierre. La suma visible de pesos de los ocho grupos es 100 %. La fórmula interna del Excel fue verificada y se aprobó usar pesos explícitos, aunque difieren del promedio igualitario de Cultura Organizacional en el Excel. Grupos sin preguntas/pesos se controlan en la publicación ponderada.

Ver [MEJ-002 — Plan de ponderación por grupos](MEJ-002-plan-ponderacion-grupos.md).

### Avance P2.2 y siguiente paso P2.3 — 2026-10-20

**P2.2 implementado técnicamente según Codex, aún sin validación integrada con BD/UI.** Fórmula aprobada: subtotales de grupo sin redondear × `PeriodGroup.weightPercentage/100`, suma ponderada y redondeo final a un decimal. Node distingue LEGACY/WEIGHTED, bloquea configuraciones incompletas; React replica la fórmula en vivo y utiliza el score persistido para `COMPLETED`. La publicación exige grupos/preguntas y pesos coherentes que sumen 100 %. Pruebas reportadas: **26/26 Node**, builds y typechecks aprobados; no se ejecutaron publicaciones ni cambios de datos.

**Diferencia aprobada frente al Excel:** pesos explícitos `20/10/10/5/15/5/15/20`; el promedio de la celda `P27` del Excel daba `10/10/10/10` como pesos efectivos para Cultura Organizacional. Caso discriminante documentado: software **28 %** vs Excel **36 %**.

**P2.3 validado técnicamente (2026-10-20):** 32 pruebas Node aprobadas, 1/1 prueba Prisma/PostgreSQL en contenedor aislado aprobada, y flujo React con API sintética confirmó 20,0 → 28,0 %, autoguardado, finalización y reapertura con score persistido. Los rechazos de publicación fueron comprobados mediante la función bloqueante, sin publicar periodos reales. **P2 se considera cerrado técnicamente**, con **único pendiente de validación operativa:** comparar una muestra real de `COMPLETED` 2025 mediante acceso autorizado y de solo lectura; sin recálculo ni modificación histórica. No ejecutar `prisma migrate deploy` hasta reconciliar historial `P3005`. Ver [MEJ-002 §20](MEJ-002-calculo-puntuaciones.md#20-p23--validación-funcional-e-integrada-2026-10-20).

## 5. P3 / MEJ-003 — Reporte de seguimiento por periodo

### Problema y evidencia

La pantalla de reporte actual presenta un periodo anterior. La auditoría indica que el backend obtiene un periodo publicado por selección implícita y que el dashboard no recibe filtro de periodo.

Componentes candidatos:
- `api/src/modules/employee-evaluations/reports/reports.service.js` y rutas/controladores de reportes.
- `sisproing-app/src/modules/employee-evaluations/reports/pages/EvaluationDashboardPage.tsx`, `services/report.service.ts`, componentes KPI, gráficas y tabla.

### Comportamiento deseado

- Selector visible de periodo en la cabecera del reporte; nombre, año y fechas del periodo seleccionado.
- Consultar por **`periodId` explícito** los KPI, confirmaciones, gráficas por proyecto, seguimiento por evaluador y detalles.
- Cambiar de periodo sin mezclar resultados de años distintos.
- Permitir consultar un periodo vencido sin que el reporte indique erróneamente que está activo.
- Definir comportamiento por defecto (recomendación pendiente de aprobación: periodo vigente por fecha; si no existe, el publicado más reciente).
- Si el usuario solicita un periodo inexistente o sin datos, mostrar estado vacío/error claro; **no cambiar silenciosamente** a otro periodo.

### Criterios de aceptación propuestos

1. Seleccionar 2025 reproduce exclusivamente sus totales e información.
2. Seleccionar un nuevo periodo muestra únicamente sus asignaciones y estados.
3. KPI, dona, barras, tabla y detalle de evaluador son coherentes con el mismo `periodId`.
4. El título y la fecha de cierre corresponden al periodo seleccionado; no se presenta «quedan 0 días» como si fuera una campaña vigente.
5. Cambiar de periodo no deja datos residuales del anterior.
6. Los filtros de proyecto/centro de costo y nuevos consolidados **no forman parte** de este alcance salvo que se aprueben al detallar P3.

### Avance P3.2 — 2026-10-20

**Implementado según el informe de Claude y documentado en [MEJ-003](MEJ-003-reportes-por-periodo.md#6-implementación--p32-2026-10-20)**: los endpoints existentes `dashboard-progress` y `evaluations` aceptan `periodId` opcional; la resolución predeterminada usa periodo vigente de Bogotá y, de faltar, publicado más reciente. Los errores `periodId` inválido (400), inexistente/borrador (404) y los estados vacíos están contemplados. React incorpora selector y estado por periodo, bloqueo de respuestas obsoletas, encabezado según periodo y modal de evaluador con el mismo `periodId` mediante contexto mínimo.

**Validación:** typecheck React y sintaxis Node aprobados. Jest de backend y build Vite no ejecutables en el entorno de Claude por problemas preexistentes; ESLint dirigido informó dos incidencias previas. **P3.2 aún requiere validación manual en navegador y API con periodos reales autorizados antes del cierre funcional**: histórico 2025, default vigente o último publicado, estados vacíos, 400/404, alternancia rápida, sincronía del detalle y controles deshabilitados. No se modificaron P1, P2, cálculos, datos históricos, PDFs ni notificaciones.

### Validación manual del selector — 2026-10-20

El responsable confirmó que **el filtro de periodo del dashboard funciona correctamente** en su entorno. Se considera validado el flujo principal de selección de periodo. **No se presume verificación completa** de casos de error 400/404, estados sin datos, cambios rápidos y sincronización `periodId` del detalle de evaluador. P3 sigue pendiente de cierre funcional integral; no se alteraron datos ni se recalcularon scores.

### Comprobación del detalle por periodo — 2026-10-20

El responsable confirma haber probado el detalle y haber observado que el payload incluye identificadores de evaluación y `periodId`, que cambian según el periodo consultado. Con la validación previa del selector, **el flujo principal de P3 queda validado funcionalmente por el responsable**. Las pruebas de situaciones excepcionales (400/404, periodo sin datos, alternancia rápida, estados de encabezado) permanecen como verificaciones complementarias; no se afirma cierre exhaustivo de toda la matriz ni pruebas automatizadas. Ver [MEJ-003](MEJ-003-reportes-por-periodo.md).

## 6. P4 / MEJ-004 — Descargar PDF individual

### Evidencia actual

Existe `GET /api/employee-evaluations/evaluations/:id/pdf`, la generación con Puppeteer y la vista `EvaluationPrintPage.tsx`. El correo de evaluación completada también adjunta PDF. Conviene **reutilizar** estas capacidades antes de crear un generador nuevo.

### Propuesta original (sustituida por la decisión aprobada)

- **Gestión Humana — Reporte → Seguimiento por Evaluador → Detalle del evaluador → Fila del colaborador → «Ver / Descargar PDF»**. El reporte es el lugar natural para localizar una evaluación de cualquier periodo.
- **Evaluador — Mis Evaluaciones → Evaluación completada → «Ver / Imprimir / Descargar PDF»**. Mantener o ampliar la capacidad ya existente.
- En ambas rutas la descarga debe corresponder al **ID de evaluación** y al periodo al que pertenece, no a «el periodo activo» obtenido implícitamente.

### Criterios de aceptación propuestos

1. Al abrir un evaluador y colaborador del histórico se puede descargar su HF-011 completado.
2. El contenido del PDF corresponde al periodo y snapshots originales.
3. No se habilita descarga de documento «final» a evaluaciones pendientes o incompletas, salvo decisión funcional expresa.
4. La descarga no cambia el estado ni envía correos.
5. Se identifica si PDF, impresión y correo usan el mismo renderer y qué elementos (puntuación, acuse) son consistentes.
6. Se respetan los accesos ya configurados para las vistas; no se crea una ruta de lectura adicional sin evaluar quién puede usarla.

### Por decidir

- Ubicación definitiva del botón y permisos por rol.
- Descarga directa frente a vista previa + descarga.
- Nombre de archivo y datos que se incluyen en el documento.
- Si la constancia de recibido se incorpora en el PDF y qué ocurre cuando el acuse llega después de generarlo.



### Decisión funcional aprobada — segunda vista en Reportes (2026-10-20)

El responsable aprobó **dos vistas dentro de Reportes**: `Seguimiento` (dashboard existente, sin cambiar) y **`Consulta de Evaluaciones`** (nueva), para buscar directamente por empleado **sin conocer su evaluador**, filtrar por periodo publicado y estado, y descargar el PDF de cada evaluación `COMPLETED` de forma individual. La tabla agrupada por evaluador y su modal siguen siendo herramientas de seguimiento; la descarga no debe colocarse en la fila agregada del evaluador. El botón «Reenviar» se deja para un futuro flujo de notificación/acuse, fuera de P4.

Reutilizar `GET /reports/evaluations` (adaptación mínima para búsqueda global por empleado si es necesaria), `GET /evaluations/:id/pdf` y la selección de periodos P3. **Precondición de seguridad:** comprobar y exigir en el servidor autenticación y autorización real para consultar/descargar un `evaluationId` arbitrario, pues la auditoría previa advirtió ausencia de control ownership en el endpoint PDF. Si el modelo de roles/permisos no se puede determinar, detener exposición de la descarga y reportar el bloqueo.

**Etapas:** P4.1 auditoría técnica dirigida del contrato y permisos → P4.2 implementación de vista/descarga → P4.3 pruebas de integración. No modificar P1/P2/P3, histórico ni correos. Documento de referencia: [MEJ-004 — Consulta y descarga individual](MEJ-004-consulta-descarga-evaluaciones.md).
## 7. P5 / MEJ-005 — Verificar comportamiento existente antes de proponer reasignación (prioridad baja)

Durante una campaña, un colaborador puede cambiar de proyecto, área o jefe después de que se publicaron y generaron las evaluaciones. El responsable confirma que **implementó un trigger para actualizar al evaluador y funciona**. No se realizará revisión ni cambio ahora. A futuro, si se solicita, únicamente se documentará el SQL exacto y sus condiciones sobre periodo, estado e histórico.

- **Pendiente futuro:** describir el trigger real y qué condición usa para identificar evaluaciones afectadas, sin intervenirlo ahora.
- **`IN_PROGRESS`:** comprobar si la actualización existente alcanza evaluaciones con respuestas guardadas y si preserva autoría.
- **`COMPLETED`:** comprobar si la actualización existente toca evaluaciones finalizadas o de años anteriores; no modificar este comportamiento sin decisión.
- **Sin funcionalidad nueva solicitada:** no implementar nueva interfaz de reasignación mientras el trigger vigente resuelva el caso.
- **Fase:** documentación técnica pendiente, al final del backlog; sin revisión inmediata, desarrollo ni cambios en P1.

Ver [MEJ-005 — Reasignación manual de evaluador](MEJ-005-reasignacion-manual-evaluador.md).

## 8. Forma de ejecutar cada mejora

Para **cada** MEJ-001…MEJ-005:

1. Revisar el comportamiento actual y el código **solo de los archivos afectados**.
2. Cerrar decisiones funcionales pendientes con el responsable del proyecto.
3. Registrar diseño, rutas/contratos necesarios, criterios de aceptación, pruebas y posibles impactos en otros tres entregables.
4. Implementar **solo cuando se autorice esa mejora**, con cambios acotados.
5. Validar con datos de prueba; especialmente conservar resultados históricos.
6. Actualizar su documento y registrar qué quedó verificado.

**Estado actualizado (2026-10-14):** P1.1 fue implementado en React y comprobado visualmente en sus principales flujos. **P1.2 quedó implementado en API y React según Codex y el responsable validó manualmente 6/6 pasos del flujo principal:** borrador sin datos relacionados, doble confirmación, descarte exitoso, formulario nuevo vacío, nuevo ID y preservación del histórico 2025. Permanecen pendientes los escenarios técnicos excepcionales (concurrencia, versiones obsoletas, errores 404/409 y rechazos de históricos). **P1.3 — selección de periodo vigente por fechas inclusivas de Bogotá** ya fue implementado técnicamente y su escenario visual sin periodo vigente se comprobó (captura 2026-10-16). Quedan otras pruebas manuales pendientes. Reglas aplicadas: «Mis Evaluaciones» y recordatorios deben usar exclusivamente el periodo publicado vigente, sin seleccionar el futuro o el vencido; P3 conserva el selector histórico del dashboard. **P1.4 — snapshots históricos** se implementó según Codex y su vista previa del periodo 2025 fue validada visualmente (banner de configuración histórica y grupos/preguntas visibles). La correspondencia exacta contra snapshots almacenados y los escenarios de alternancia/error permanecen pendientes. Ver [MEJ-001-P1-4-snapshots-historicos.md](MEJ-001-P1-4-snapshots-historicos.md). Las demás prioridades continúan pendientes. No se han autorizado publicaciones de prueba en producción.

Ver [avance y checklist de P1.1](MEJ-001-historico-creacion-periodos.md#17-avance-de-implementación--p11-reporte-de-codex-2026-10-14).

### Prioridad inmediata — posible desincronización en «Mis Evaluaciones» (2026-10-20)

La captura del evaluador muestra un periodo vigente **20/oct/2026–08/nov/2026** y simultáneamente **0/0 evaluaciones**. El responsable sospecha que la lista puede estar resolviendo otra campaña. La causa **no está verificada**; P1.3 ya diseñó `findCurrentPublishedPeriod` y su uso en los dos endpoints `my-team`. Revisar Network/JSON de ambos endpoints y conteos por `periodId + evaluatorId` en modo de solo lectura antes de cambiar código o generar asignaciones. **«Mis Evaluaciones» debe mostrar únicamente el periodo publicado vigente por fecha Bogotá**, no escoger el primer `isActive=true` arbitrariamente. Ver [incidencia en MEJ-001](MEJ-001-historico-creacion-periodos.md#24-incidencia-observada--mis-evaluaciones-muestra-00-durante-periodo-vigente-2026-10-20).

### Backlog para el final — MEJ-006: corrección de puntajes históricos

El responsable solicitó documentar y **aplazar** hasta el final el recálculo de evaluaciones completadas de 2025 con pesos explícitos P2. No rellenar `weight_percentage` de snapshots LEGACY como método rápido. Futuro: auditoría de solo lectura → `dry-run` con informe de diferencias → respaldo/trazabilidad de puntajes y PDFs con acuse → aprobación explícita → corrección controlada. Caso testigo evaluación 8120: **61,8 % LEGACY → 61,4 % ponderado**. **Sin SQL de actualización ni cambios de datos ahora.** Documento: [MEJ-006 — Recalculo histórico](MEJ-006-recalculo-historico-2025.md).

## 9. Fuentes de referencia

- [Contexto funcional](../funcional/contexto-funcional.md)
- [Reglas de negocio](../funcional/reglas-negocio.md)
- [Estado de implementación](../auditoria/estado-implementacion.md)
- [Diferencias frente al requerimiento](../auditoria/diferencias-requerimiento.md)
- [Pendientes de validación](../auditoria/pendientes-validacion.md)
- [Backend](../tecnico/backend.md), [Frontend](../tecnico/frontend.md) y [Endpoints](../tecnico/endpoints.md)
