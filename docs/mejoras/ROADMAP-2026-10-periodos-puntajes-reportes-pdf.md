# Hoja de ruta — Periodos, puntajes, reportes, PDF y reasignación

**Proyecto:** PROING — Evaluación de Desempeño HF-011  
**Fecha:** 2026-10-14  
**Estado:** plan priorizado; pendiente de análisis e implementación, una mejora a la vez  
**Alcance:** exclusivamente `api/src/modules/employee-evaluations/`, sus dependencias necesarias de notificaciones y `sisproing-app/src/modules/employee-evaluations/`. Este plan no autoriza revisar otros módulos de negocio.

## 1. Objetivo y decisiones de alcance

Evolucionar el módulo existente para (1) gestionar correctamente los periodos históricos y crear uno nuevo, (2) revisar y corregir los cálculos de puntuación, (3) consultar reportes de un periodo seleccionado y (4) descargar el PDF de una evaluación individual y (5) permitir, en fase posterior, la reasignación manual y auditada de evaluadores durante el periodo.

- **Orden de prioridad acordado:** P1 → P2 → P3 → P4 → P5. La reasignación manual de evaluador (P5) es **prioridad baja** y no se adelanta a P1.
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
| **P5** | MEJ-005 | Reasignación administrativa de evaluador | Transferencia manual, individual y auditable de la asignación de una evaluación durante el periodo, sin modificar HF-002 automáticamente. |

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

Fórmula documentada actualmente, **pendiente de ratificación de negocio**:

`porcentaje_total = redondear_1_decimal((suma_puntos / (numero_preguntas * 5)) * 100)`.

Las preguntas sin respuesta cuentan en el máximo del periodo; no hay peso por grupo. El frontend también tiene un **porcentaje de avance del formulario** que considera fortalezas/debilidades y **no es el puntaje de desempeño**.

### Trabajo previsto

1. Localizar **todas** las operaciones de cálculo visual de puntaje, porcentaje por grupo, total, categoría y avance; no confundir estos conceptos.
2. Comparar con el cálculo persistido en Node y con la lógica de los reportes/impresión que consume ese valor.
3. Acordar la fórmula exacta de **cada grupo** y del **total**, tratamiento de preguntas sin responder, redondeo y escala.
4. Proponer como base, si Gestión Humana lo aprueba, **sin ponderaciones**, con máximo 5 por pregunta; calcular el total desde las respuestas, no promediando porcentajes de grupos ya redondeados.
5. Corregir divergencias con pruebas de casos numéricos (grupos de distinto tamaño, respuestas parciales, todas completas y extremos de 1 a 5).
6. No recalcular ni sobrescribir silenciosamente los resultados históricos.

### Criterios de aceptación propuestos

- El puntaje que muestra el formulario coincide con el guardado en base y el mostrado en PDF/reporte para la misma evaluación.
- Los porcentajes por grupo y el total tienen fórmulas documentadas y pruebas.
- No se confunde «avance del formulario» con «resultado del desempeño».
- Las evaluaciones finalizadas históricas no se modifican sin una decisión explícita y un procedimiento auditado.

### Decisión pendiente esencial

**¿La corrección aplica únicamente a evaluaciones nuevas o también se recalcularán las antiguas?** No se debe cambiar el histórico por defecto.

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

## 6. P4 / MEJ-004 — Descargar PDF individual

### Evidencia actual

Existe `GET /api/employee-evaluations/evaluations/:id/pdf`, la generación con Puppeteer y la vista `EvaluationPrintPage.tsx`. El correo de evaluación completada también adjunta PDF. Conviene **reutilizar** estas capacidades antes de crear un generador nuevo.

### Ubicación funcional recomendada

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

## 7. P5 / MEJ-005 — Reasignación manual de evaluador (prioridad baja)

Durante una campaña, un colaborador puede cambiar de proyecto, área o jefe después de que se publicaron y generaron las evaluaciones. La futura mejora permitiría a un administrador autorizado reasignar **una evaluación concreta** sin regenerar todas las asignaciones, preservando `periodId`, `evaluationId`, el histórico y un registro auditable del cambio. No se plantea evaluar siempre con el jefe actual ni modificar la fuente de jefes HF-002.

- **Propuesta inicial:** permitir `PENDING` con motivo obligatorio e historial de evaluador anterior/nuevo, administrador y fecha.
- **`IN_PROGRESS`:** pendiente de definir, ya que puede contener borradores de otro evaluador.
- **`COMPLETED`:** no reasignable por la operación ordinaria.
- **Dependencias:** consultas por evaluador, recordatorios, reportes y controles de acceso de la nueva acción; no constituye una reforma general de seguridad.
- **Fase:** backlog, sin análisis técnico nuevo ni implementación en P1.

Ver [MEJ-005 — Reasignación manual de evaluador](MEJ-005-reasignacion-manual-evaluador.md).

## 8. Forma de ejecutar cada mejora

Para **cada** MEJ-001…MEJ-005:

1. Revisar el comportamiento actual y el código **solo de los archivos afectados**.
2. Cerrar decisiones funcionales pendientes con el responsable del proyecto.
3. Registrar diseño, rutas/contratos necesarios, criterios de aceptación, pruebas y posibles impactos en otros tres entregables.
4. Implementar **solo cuando se autorice esa mejora**, con cambios acotados.
5. Validar con datos de prueba; especialmente conservar resultados históricos.
6. Actualizar su documento y registrar qué quedó verificado.

**Estado actualizado:** P1 tiene microauditoría documentada y decisiones de borrador único/inmutabilidad aprobadas; pendiente implementación. P2, P3, P4 y P5 en cola, con P5 como último bloque. Ninguna modificación de código queda autorizada por este documento.

## 9. Fuentes de referencia

- [Contexto funcional](../funcional/contexto-funcional.md)
- [Reglas de negocio](../funcional/reglas-negocio.md)
- [Estado de implementación](../auditoria/estado-implementacion.md)
- [Diferencias frente al requerimiento](../auditoria/diferencias-requerimiento.md)
- [Pendientes de validación](../auditoria/pendientes-validacion.md)
- [Backend](../tecnico/backend.md), [Frontend](../tecnico/frontend.md) y [Endpoints](../tecnico/endpoints.md)
