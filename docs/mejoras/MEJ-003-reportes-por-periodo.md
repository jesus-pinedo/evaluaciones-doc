# MEJ-003 — Selector de periodo en el dashboard de reportes

**Proyecto:** PROING — Evaluación de Desempeño HF-011
**Fecha de registro:** 2026-10-20
**Prioridad:** P3 (tercera de la hoja de ruta, después de P1 y P2)
**Estado:** **AUDITORÍA FUNCIONAL Y TÉCNICA — SIN IMPLEMENTAR.** P1 y P2 están terminados técnicamente y no se tocaron. No se modificó código, cálculos, periodos, snapshots, PDF ni notificaciones.
**Alcance inspeccionado:** exclusivamente `api/src/modules/employee-evaluations/reports/` y `sisproing-app/src/modules/employee-evaluations/reports/`, más las dependencias mínimas de periodos usadas por esos archivos (`periods/currentPeriod.service.js`, `periods/periods.service.js#findAll`).

---

## 1. Resumen ejecutivo

El dashboard de seguimiento (`/employee-evaluations/reports`) **no tiene selector de periodo, ni visual ni funcional**. El backend elige el periodo de forma implícita con `evaluationPeriod.findFirst({ where: { isActive: true } })` y **no existe ningún parámetro `periodId`** en ningún endpoint de `reports/`. Como resultado:

- Solo puede verse **un** periodo por vez: el que tenga `isActive=true` en base de datos (que, según `reglas-negocio.md`, significa «publicado», no necesariamente «vigente por fechas»).
- Si hay más de un periodo publicado (`isActive=true`) simultáneamente — la regla `RN-C03` solo bloquea solapamiento de fechas, no lo prohíbe —, `findFirst` toma el primero que Prisma devuelva, sin orden explícito.
- No hay forma de consultar un periodo histórico finalizado ni uno futuro programado desde este dashboard: solo `PeriodsPage`/`CreatePeriodPage` (otro módulo) permiten verlos.
- Todos los indicadores que muestra el dashboard hoy son de **avance operativo** (conteos de estado y confirmaciones). **No hay ningún indicador de puntaje/desempeño** en este dashboard; esos valores viven en `evaluations.helper.js` y se consultan evaluación por evaluación, no aquí.

La implementación mínima consiste en: (a) exponer `periodId` como parámetro explícito en los dos endpoints de `reports/`, (b) agregar un selector real en el frontend reutilizando el listado de periodos que ya expone `GET /api/employee-evaluations/periods`, y (c) decidir el comportamiento por defecto y de error cuando no hay periodo o no hay datos.

---

## 2. Archivos revisados

### Backend (`api/src/modules/employee-evaluations/reports/`)
| Archivo | Rol |
|---|---|
| `reports.route.js` | Define `GET /dashboard-progress` y `GET /evaluations`. |
| `reports.controller.js` | `getDashboardProgress` (sin leer `req.query`, hay un comentario explícito: *"En el futuro, aquí extraeremos filtros del query"*) y `getEvaluationsList` (lee `page`, `limit`, `status`, `evaluatorId`, `search`; **no lee `periodId`**). |
| `reports.service.js` | `getDashboardProgress()`, `getEvaluationsList()` y `getUsersForDailyReminder()` (este último es de notificaciones, no del dashboard; se revisó solo para contrastar el patrón de selección de periodo). |

Dependencia externa mínima revisada: `periods/currentPeriod.service.js` (`findCurrentPublishedPeriod`, usado solo por los recordatorios, **no** por el dashboard) y `periods/periods.service.js` (`findAll`, `getPeriodStatus`, usados por `GET /api/employee-evaluations/periods`, que **sí** podría reutilizarse para poblar un selector).

### Frontend (`sisproing-app/src/modules/employee-evaluations/reports/`)
| Archivo | Rol |
|---|---|
| `pages/EvaluationDashboardPage.tsx` | Página real montada en la ruta `/employee-evaluations/reports` (ver `routes/index.tsx:63`). Llama `reportService.getDashboardProgress()` sin argumentos en un único `useEffect` con dependencias `[]`. |
| `ReportsPage.tsx` | Componente placeholder (`<h2>Página de Reportes</h2>`), montado en la ruta muerta `/employee-evaluations/reports_temp`. No forma parte del flujo real; se documenta para evitar confusión, no se audita más a fondo. |
| `services/report.service.ts` | Único método `getDashboardProgress()`, sin parámetros, llama `GET /employee-evaluations/reports/dashboard-progress` sin query string. |
| `types/reportTypes.ts` | `ReportPeriod` solo tiene `id`, `name`, `endDate`, `remainingDays` (no `startDate` ni `status`). |
| `components/DashboardFilters.tsx` | Contiene dos `<select>` **deshabilitados** ("Todos los Proyectos", "Todos los Centros de Costo") con comentario `{/* Selects Visuales (Deshabilitados para MVP) */}`. **No incluye ningún control de periodo.** Solo muestra el nombre del periodo (vía el `<h1>` en la página) y "Cierre: {fecha} (Quedan N días)". |
| `components/KPICards.tsx` | Tarjetas de `totalEmployees`, `completed`, `inProgress`, `pending`, `accepted`/tasa de aceptación. Todos son conteos de **avance operativo**, no puntajes. |
| `components/Charts.tsx` | Dona de estado global (`completed` vs resto) y barras por proyecto (`completed/total`). También avance operativo. |
| `components/EvaluatorTable.tsx` | Tabla de evaluadores con conteos por estado; al hacer clic abre `EvaluatorDetailModal`. |
| `components/EvaluatorDetailModal.tsx` | Llama `GET /employee-evaluations/reports/evaluations` con `evaluatorId` y `status`, **sin `periodId`**. |
| `components/DashboardSkeleton.tsx` | Solo UI de carga; no interactúa con periodo. |

---

## 3. Respuestas a las preguntas planteadas

### 1. ¿Cómo selecciona actualmente el backend el periodo?

De forma **implícita y única**, en ambos métodos de `reports.service.js`:

```js
const period = await prisma.evaluationPeriod.findFirst({
  where: { isActive: true },
  select: { id: true, name: true, endDate: true }, // getDashboardProgress
});
```
```js
const period = await prisma.evaluationPeriod.findFirst({
  where: { isActive: true },
  select: { id: true } // getEvaluationsList
});
```

No hay `orderBy`, por lo que si existiera más de un periodo con `isActive=true` (publicado), Prisma devuelve el primero según el orden físico/índice de la tabla, no por fecha. Esto es consistente con lo ya documentado en `reglas-negocio.md` (RN-C07: *"Periodo activo" en consultas significa `isActive=true`, no necesariamente fecha vigente*) y con la advertencia de la hoja de ruta sobre no asumir que `isActive=true` equivale a "vigente por fechas". El módulo de reportes **no usa** `findCurrentPublishedPeriod()` (el selector por fecha de Bogotá que sí usan los recordatorios); usa el criterio más simple y más ambiguo de los dos existentes en el código.

Si no hay ningún periodo con `isActive=true`, ambos métodos lanzan `throw new Error(...)` (no `AppError`), que el controlador reenvía con `next(error)` al manejador de errores global sin código HTTP ni código de error estructurado explícito en este archivo.

### 2. ¿Qué consultas requieren recibir `periodId`?

Todas las consultas de `reports.service.js` dependen de un `periodId` para acotar `evaluation.findMany`/`count`, pero **ninguna lo recibe como parámetro hoy**; todas lo derivan internamente del `findFirst({ isActive: true })`:

- `getDashboardProgress`: `prisma.evaluation.findMany({ where: { periodId: period.id, isActive: true }, ... })` — alimenta KPIs globales, tabla de evaluadores y gráfico de proyectos.
- `getEvaluationsList`: `where.periodId = period.id` — alimenta el modal de drill-down (`EvaluatorDetailModal`).

Para habilitar el selector, **ambas** funciones necesitan aceptar `periodId` explícito (por query string) y usarlo en lugar de `findFirst({ isActive: true })`.

### 3. ¿Qué indicadores muestra actualmente el dashboard?

Únicamente **indicadores de avance operativo** (no de desempeño/puntaje):

- KPIs globales: total de personal, evaluaciones `COMPLETED`, `IN_PROGRESS`, `PENDING`, y `accepted` (subconjunto de `COMPLETED` con `notificationReceived=true`, es decir, confirmación/acuse de recibido).
- Tasa de aceptación: `accepted / completed`.
- Dona de estado global: `completed` vs. resto, sobre `totalEmployees`.
- Barras por proyecto: `completed / total` por proyecto del **evaluado**, redondeado a entero.
- Tabla por evaluador (jefe): mismos cuatro contadores de estado por evaluador, ordenada por `stats.pending` descendente; permite drill-down al listado de evaluaciones de ese evaluador/estado.
- Encabezado: nombre del periodo y "días restantes" (`remainingDays`, calculado como `endDate - hoy` en días calendario, sin ajuste de zona horaria de Bogotá, a diferencia del cálculo que sí usan los recordatorios).

**No hay ningún puntaje, porcentaje de desempeño, clasificación/`ratingLabel` ni resultado por grupo** en este dashboard. Esos valores (calculados y persistidos por P2 en `evaluations.helper.js`) no se leen ni se muestran aquí. Confirma la distinción pedida por el encargo: el dashboard de HF-011 es un tablero de **seguimiento/avance**, no de **resultados de desempeño**.

### 4. ¿Cuáles son los endpoints involucrados?

| Método | Ruta | Controlador | Uso actual del frontend |
|---|---|---|---|
| `GET` | `/api/employee-evaluations/reports/dashboard-progress` | `getDashboardProgress` | `EvaluationDashboardPage` al montar (sin filtros). |
| `GET` | `/api/employee-evaluations/reports/evaluations` | `getEvaluationsList` | `EvaluatorDetailModal` con `page`, `limit`, `status`, `evaluatorId`. |

Ninguno acepta `periodId`. Como referencia para el selector (no modificado en este alcance): `GET /api/employee-evaluations/periods` (con filtro opcional `isActive`) ya devuelve todos los periodos con `id`, `name`, `startDate`, `endDate`, `isActive` y `status` derivado (`DRAFT`/`SCHEDULED`/`ACTIVE`/`FINISHED`), ordenados por `startDate desc`.

### 5. ¿Cómo se muestran los reportes históricos?

**No se muestran.** No existe ningún punto de entrada en `reports/` para elegir un periodo distinto al único `isActive=true` resuelto por el backend. Un periodo `FINISHED` (vencido) solo es visible desde `PeriodsPage` (listado/gestión de periodos) y desde el `preview` de snapshots (`periods/:id/preview`, parte de P1), no desde el dashboard de reportes. La "evidencia visual" mencionada en el roadmap (captura con periodo 2025) corresponde a un momento en que ese periodo 2025 era el único o el primero con `isActive=true`, no a una consulta histórica deliberada.

### 6. ¿Cómo incorporar un selector de periodo con el menor cambio posible?

Propuesta de cambio mínimo, sin tocar P1/P2 ni refactorizar:

**Backend (`reports.controller.js` + `reports.service.js`):**
1. Leer `req.query.periodId` en ambos controladores.
2. En `reports.service.js`, reemplazar el `findFirst({ isActive: true })` por:
   - si llega `periodId`: `prisma.evaluationPeriod.findUnique({ where: { id: Number(periodId) } })`, validando que exista (404/`AppError` si no);
   - si **no** llega `periodId`: mantener un valor por defecto (ver pendiente de decisión en la sección 8) para no romper el comportamiento actual.
3. Añadir `startDate` y `status` (reutilizando `getPeriodStatus`, hoy solo en `periods.service.js`) a la selección de `period` que devuelve `getDashboardProgress`, para que el frontend pueda mostrar banderas como "Periodo finalizado" o "Periodo futuro" sin inventar esa lógica en el cliente.
4. No tocar `getUsersForDailyReminder` (usa `findCurrentPublishedPeriod`, es de notificaciones, fuera de alcance).

**Frontend:**
1. Agregar un `<select>` de periodo en `DashboardFilters.tsx` (o un componente nuevo al lado), poblado con `GET /api/employee-evaluations/periods` (ya existe, no requiere endpoint nuevo).
2. Mantener el `periodId` seleccionado en estado de `EvaluationDashboardPage.tsx` y pasarlo como dependencia del `useEffect` que llama a `reportService.getDashboardProgress(periodId)`.
3. Extender `report.service.ts` para aceptar `periodId` y agregarlo al query string.
4. Propagar el mismo `periodId` seleccionado a `EvaluatorDetailModal` (vía prop) para que su llamada a `/reports/evaluations` lo incluya; hoy ese componente no recibe el periodo en absoluto.
5. Ampliar `ReportPeriod` en `reportTypes.ts` con `startDate` y `status`.

Este diseño no requiere nuevas tablas, migraciones ni cambios en P1/P2; reutiliza el listado de periodos y el campo `periodId` que ya existe en el modelo `Evaluation`.

### 7. ¿Cómo garantizar que todos los indicadores utilicen el mismo `periodId`?

Hoy la consistencia es "gratuita" porque solo hay una fuente de periodo (el `findFirst` interno) y todas las consultas de `getDashboardProgress` parten de `evaluations.findMany({ where: { periodId: period.id, ... } })`: KPIs, tabla de evaluadores y gráfico de proyectos se derivan del **mismo arreglo en memoria**, así que no pueden desincronizarse entre sí dentro de una misma respuesta.

El riesgo aparece en el **drill-down**: `EvaluatorDetailModal` hace una llamada HTTP aparte (`/reports/evaluations`) que hoy resuelve su propio periodo de forma independiente (`findFirst` propio en `getEvaluationsList`). Si se agrega el selector sin pasar el `periodId` explícitamente al modal, un usuario podría ver el dashboard de un periodo histórico y, al abrir el detalle de un evaluador, recibir datos de otro periodo (el que el backend siga resolviendo por defecto). La forma de garantizarlo es **una sola**: que el `periodId` seleccionado en la página viaje como prop/parámetro explícito a cada componente y llamada que hoy resuelve el periodo por su cuenta, en lugar de permitir que cada endpoint elija su propio periodo implícito.

### 8. ¿Qué sucede cuando un periodo no tiene evaluaciones?

Dos casos distintos, con comportamiento actual verificado en código:

- **No existe ningún periodo con `isActive=true`:** ambas funciones de `reports.service.js` hacen `throw new Error('No hay un periodo de evaluación activo configurado.')` / `'No hay periodo activo.'`. Son `Error` planos, no `AppError`; el controlador los pasa con `next(error)` al manejador global, sin código HTTP ni código de error estructurado propio de este módulo (a diferencia de `periods.service.js`, que sí usa `AppError` con código HTTP y `errorCode`). El frontend solo captura esto como un `catch` genérico en el `useEffect` (`console.error`) y dejaría `reportData` en `null`, mostrando una página vacía sin mensaje para el usuario (`if (!reportData) return null;`).
- **El periodo existe pero no tiene evaluaciones (`evaluations.length === 0`):** no se lanza error. `globalStats` queda con todos los contadores en 0, `evaluatorsMap`/`projectsMap` quedan vacíos, y el frontend renderiza KPIs en cero, gráficos vacíos y una tabla de evaluadores vacía, sin ningún mensaje explícito de "este periodo no tiene evaluaciones registradas". No hay un estado vacío diseñado; simplemente se ven ceros.

Al introducir el selector, un `periodId` que no exista en la tabla debe devolver un error claro y estructurado (404/`AppError`), **no** reusar el mensaje genérico actual, y un periodo válido sin evaluaciones debe diferenciarse visualmente de un error (mensaje tipo "Sin evaluaciones registradas para este periodo" en vez de una fila de ceros silenciosa), tal como pide el encargo original ("no cambiar silenciosamente a otro periodo").

### 9. ¿Qué debe ocurrir cuando se cambia de periodo?

Con el diseño propuesto (cambio de estado de React que dispara un nuevo `useEffect`/`fetch`):

- La petición a `/dashboard-progress` se repite con el nuevo `periodId`; como el backend arma `globalStats`, `evaluators` y `projects` desde una sola consulta, no quedan datos residuales del periodo anterior dentro de esa respuesta.
- Debe mostrarse un estado de carga (`DashboardSkeleton`, ya existe) mientras se resuelve la nueva petición, para evitar que se vean momentáneamente los números del periodo anterior superpuestos con el nombre del nuevo periodo.
- Si el modal de detalle (`EvaluatorDetailModal`) está abierto al cambiar de periodo, debe cerrarse o refrescarse con el nuevo `periodId`; hoy no hay ninguna lógica que lo contemple porque el modal no sabe nada del periodo.
- El "Cierre: {fecha} (Quedan N días)" de `DashboardFilters` debe dejar de mostrarse (o cambiar de forma) cuando el periodo seleccionado no es el vigente por fechas, porque hoy ese texto asume implícitamente un periodo activo en curso (es el mismo riesgo que señala el roadmap: *"no presentar 'quedan 0 días' como si fuera una campaña vigente"*).

### 10. ¿Cuáles son los riesgos de implementación?

1. **Doble fuente de "periodo por defecto".** Ya existen dos criterios distintos en el código (`findFirst({isActive:true})` en reportes vs. `findCurrentPublishedPeriod` por fecha de Bogotá en recordatorios). Si P3 introduce un tercer criterio para el valor por defecto del selector sin unificarlo, el dashboard podría mostrar "por defecto" un periodo distinto al que usan los recordatorios o al que el usuario esperaría como "vigente". Esto requiere una decisión funcional explícita (ver sección 8 de este documento y la pendiente ya anotada en el roadmap).
2. **Múltiples periodos `isActive=true` simultáneos.** La regla `RN-C03` permite varios periodos publicados no solapados en fechas. El `findFirst` sin `orderBy` ya es ambiguo hoy; si se mantiene como *fallback* sin `periodId`, la ambigüedad persiste aunque se agregue el selector.
3. **Drill-down desincronizado.** Si se olvida propagar `periodId` al modal (`EvaluatorDetailModal`) y a su llamada a `/reports/evaluations`, el detalle puede mostrar datos de un periodo distinto al que ve el usuario en el dashboard — ver sección 7.
4. **Manejo de errores inconsistente.** `reports.service.js` usa `Error` planos en vez de `AppError` (patrón que sí sigue `periods.service.js`); si no se homologa, un `periodId` inexistente o inválido puede terminar en un error 500 genérico en lugar de un 404/400 claro.
5. **Confusión entre "sin datos" y "error".** Como se documentó en la pregunta 8, hoy ambos casos no se distinguen bien en el frontend; agregar el selector sin resolver esto puede hacer que un periodo histórico sin evaluaciones parezca un error del sistema, o viceversa.
6. **`remainingDays` y el texto de cierre en periodos no vigentes.** El cálculo de días restantes y el texto "Cierre: … (Quedan N días)" están pensados para un periodo activo en curso; aplicarlos sin cambios a un periodo `FINISHED` o `SCHEDULED` mostraría información engañosa (días negativos o un conteo sin sentido) si no se ajusta la UI según el `status` del periodo.
7. **Consistencia con P1/P2 sin modificarlos.** El selector debe leer el mismo campo `periodId` y el mismo modelo `EvaluationPeriod`/`Evaluation` que P1 y P2 ya usan, sin crear una ruta paralela de resolución de periodo ni tocar `evaluations.helper.js` o `periods.service.js`. El riesgo es que, al implementar, se termine "arreglando" de paso el criterio de periodo activo en otros módulos — eso excede el alcance de P3.
8. **Campo `status` no expuesto hoy en la respuesta de reportes.** Mostrar correctamente banderas de "histórico"/"vigente"/"futuro" en el dashboard obliga a agregar el cálculo de estado (hoy solo existe en `periods.service.js#getPeriodStatus`, privado a ese archivo) a la respuesta de `reports.service.js`, o a duplicarlo — se recomienda lo primero para no duplicar la regla de negocio en dos archivos.

---

## 4. Decisiones funcionales pendientes antes de implementar

1. **Valor por defecto del selector cuando no se envía `periodId`:** ¿el periodo vigente por fecha (como los recordatorios), el publicado más reciente por `startDate`, o mantener el `findFirst({isActive:true})` actual? El roadmap ya deja anotada una recomendación ("vigente por fecha; si no existe, el publicado más reciente") pero **sin aprobación registrada**.
2. **Comportamiento ante `periodId` inexistente o no publicado (`DRAFT`):** ¿error 404 explícito, o se permite consultar borradores desde este dashboard? (El roadmap pide no cambiar silenciosamente de periodo; falta decidir el código/mensaje exacto.)
3. **Mensaje de estado vacío** para un periodo publicado sin evaluaciones generadas, distinto del mensaje de error.
4. **Alcance de "días restantes"/alerta de cierre** cuando el periodo seleccionado no es el vigente: ¿ocultar el bloque, reemplazarlo por la etiqueta de estado (`FINISHED`/`SCHEDULED`), o mostrar ambas cosas?
5. **Si el drill-down (`EvaluatorDetailModal`) debe cerrarse automáticamente** al cambiar de periodo, o si puede quedar abierto mostrando datos de un periodo distinto al del fondo (no recomendado, pero debe decidirse explícitamente).
6. Confirmar que, como pide el encargo, **no se agreguen** en esta mejora los filtros de proyecto/centro de costo (hoy deshabilitados en `DashboardFilters.tsx`) ni indicadores de puntaje/desempeño: el alcance de P3 es exclusivamente el selector de periodo sobre los indicadores operativos ya existentes.

---

## 5. Lo que esta auditoría NO hizo

De acuerdo con las restricciones del encargo: no se modificó código de `api/` ni de `sisproing-app/`; no se tocaron cálculos de P2, respuestas, evaluaciones, periodos, snapshots, PDF ni notificaciones; no se auditaron otros módulos de PROING; no se generaron commits.


## 6. Decisiones funcionales aprobadas para P3.2 — 2026-10-20

El responsable del proyecto **aprobó expresamente** las ocho reglas propuestas para implementar el selector de periodos del dashboard. Esta aprobación aplica exclusivamente a P3.2, no autoriza refactoring ni modificaciones en P1/P2.

| # | Tema | Regla aprobada |
|---|---|---|
| 1 | Periodo predeterminado | Priorizar el **periodo publicado vigente por fecha** (días calendario de Bogotá), utilizando el criterio existente de P1; cuando no exista, seleccionar el **periodo publicado más reciente** mediante orden determinista (por ejemplo `startDate DESC, id DESC`). Sin periodos publicados: estado vacío explícito. |
| 2 | `periodId` inválido o inexistente | Validar sintaxis del parámetro (400 cuando sea inválido) y devolver **404** cuando el identificador no exista; **nunca sustituirlo silenciosamente por otro periodo**. |
| 3 | Periodo `DRAFT` | No mostrarlo en el selector y rechazar consultas de reportes para ese `periodId` (404 coherente con el acceso a periodos publicados). |
| 4 | Periodo publicado sin evaluaciones | Mostrar **«Este periodo aún no tiene evaluaciones registradas»**; diferenciarlo de una falla HTTP o de carga. |
| 5 | Encabezado/cierre | `ACTIVE`: fecha de cierre y días restantes; `FINISHED`: fecha de finalización sin «quedan N días»; `SCHEDULED`: fecha de inicio, sin simular campaña vigente. |
| 6 | Drill-down | **Cerrar `EvaluatorDetailModal` al cambiar de periodo**, y en cualquier caso enviar al endpoint del modal el mismo `periodId` seleccionado. |
| 7 | Filtros de proyecto y centro de costo | Permanecen deshabilitados; no se implementan en P3.2. |
| 8 | Indicadores | Se mantienen únicamente los KPI/gráficas/tablas **operativos** existentes; no incluir score, `ratingLabel` ni métricas de desempeño. |

### Contrato mínimo de implementación autorizado

- Reutilizar `GET /api/employee-evaluations/periods` para cargar el selector, filtrando periodos publicados sin alterar P1.
- Extender **los dos endpoints existentes** `GET /api/employee-evaluations/reports/dashboard-progress` y `GET /api/employee-evaluations/reports/evaluations` para recibir `periodId` explícito, con resolución determinista compartida en la medida mínima necesaria dentro de reportes. No crear endpoints nuevos.
- Mantener el `periodId` seleccionado en `EvaluationDashboardPage.tsx` y propagarlo a `report.service.ts` y a `EvaluatorDetailModal`, sin que el modal elija otro periodo por su cuenta.
- Evitar mostrar datos de periodos diferentes cuando las solicitudes se solapan: limpiar el estado anterior, mostrar loading y descartar respuestas obsoletas (cancelación o identificador de solicitud).
- Preservar paginación, búsqueda, estados, KPIs y el comportamiento actual de los reportes salvo la selección explícita de periodo y el ajuste de encabezados/estados vacíos.
- Pruebas dirigidas de selección inicial, alternancia 2025/otro periodo, detalle consistente, 400/404, borradores, periodo sin evaluaciones, tres estados de periodo y respuesta obsoleta.
- **No modificar** fórmula de P2, `score`, `ratingLabel`, evaluaciones, snapshots, periodos, PDFs, notificaciones ni otros módulos. Sin migraciones ni recálculos históricos.
- Documentar cambios y resultados en este archivo; no generar commits automáticamente.

**Estado:** decisión aprobada; **implementación P3.2 pendiente de Claude/Codex**.


## 6. Implementación — P3.2 (2026-10-20)

**Estado:** implementado según las decisiones funcionales aprobadas arriba (sección "Decisiones funcionales pendientes" queda resuelta por esas decisiones). No se tocó P1 ni P2; no se generaron commits ni se ejecutaron migraciones.

### 6.1 Archivos modificados

Backend (`api/src/modules/employee-evaluations/reports/`):
- `reports.service.js`
- `reports.controller.js`

Frontend (`sisproing-app/src/modules/employee-evaluations/reports/`):
- `pages/EvaluationDashboardPage.tsx`
- `components/DashboardFilters.tsx`
- `components/EvaluatorDetailModal.tsx`
- `services/report.service.ts`
- `types/reportTypes.ts`
- `context/ReportPeriodContext.ts` **(archivo nuevo, ver nota)**

**Nota sobre el archivo nuevo:** el encargo listó cinco archivos de frontend a modificar, pero no `EvaluatorTable.tsx`, que es quien efectivamente renderiza `EvaluatorDetailModal` y le pasa sus props. Para (a) propagar el `periodId` seleccionado al modal y (b) cerrarlo automáticamente al cambiar de periodo, sin tocar `EvaluatorTable.tsx`, se introdujo un React Context mínimo de una sola línea de superficie (`ReportPeriodContext`, un `createContext<number | null>(null)`). `EvaluationDashboardPage.tsx` lo provee (`<ReportPeriodContext.Provider value={selectedPeriodId}>` envolviendo el contenido del dashboard) y `EvaluatorDetailModal.tsx` lo consume con `useContext`. Ningún otro archivo fuera de la lista original fue tocado.

No se modificaron `EvaluatorTable.tsx`, `KPICards.tsx`, `Charts.tsx`, `DashboardSkeleton.tsx`, `ReportsPage.tsx` (placeholder muerto) ni ningún archivo de `periods/` (ni backend ni frontend): solo se **importan** tipos y funciones ya exportados por P1 (`findCurrentPublishedPeriod`, `calculateRemainingBogotaDays`, `getBogotaDateOnly`, `toPrismaDateOnly` del backend; `periodsService.listAll()` y el tipo `PeriodDto` del frontend), sin cambiar su comportamiento.

### 6.2 Contratos de endpoint (cambios)

Ambos endpoints existentes aceptan ahora `periodId` opcional por query string; ningún endpoint nuevo.

`GET /api/employee-evaluations/reports/dashboard-progress?periodId=<id>`
`GET /api/employee-evaluations/reports/evaluations?page=&limit=&status=&evaluatorId=&search=&periodId=<id>`

Nueva forma de la respuesta de `dashboard-progress` (antes `period` nunca era `null` y no existía `hasEvaluations`):

```json
{
  "success": true,
  "data": {
    "period": {
      "id": 7,
      "name": "Evaluaciones 2026",
      "startDate": "2026-01-01",
      "endDate": "2026-01-11",
      "status": "ACTIVE",
      "remainingDays": 4
    },
    "hasEvaluations": true,
    "globalStats": { "totalEmployees": 0, "completed": 0, "inProgress": 0, "pending": 0, "accepted": 0 },
    "evaluators": [],
    "projects": []
  }
}
```

- `period: null` → no existe ningún periodo publicado (estado vacío, no es un error).
- `period` presente + `hasEvaluations: false` → el periodo está publicado pero aún no tiene evaluaciones generadas.
- `remainingDays` solo tiene valor cuando `status === 'ACTIVE'`; en `FINISHED`/`SCHEDULED` es `null` (el frontend muestra fecha de finalización o de inicio en su lugar, no días restantes).
- `startDate` se agregó al objeto `period` (antes solo existía `endDate`).

Errores (antes ambos endpoints lanzaban `Error` plano sin código/HTTP estructurado cuando no había periodo activo; ahora):
- **400** `EVAL_REPORT_INVALID_PERIOD_ID` — `periodId` no es un entero positivo.
- **404** `EVAL_REPORT_PERIOD_NOT_FOUND` — el `periodId` no existe o corresponde a un periodo `DRAFT` (no publicado).
- Nunca se sustituye silenciosamente el `periodId` pedido por otro periodo.

### 6.3 Selección predeterminada implementada (sin `periodId`)

`resolveDefaultPublishedPeriod()` en `reports.service.js`:
1. Reutiliza `findCurrentPublishedPeriod()` (P1, sin modificar) para el periodo vigente por fecha de Bogotá.
2. Si esa función reporta ambigüedad (`EVAL_PERIOD_CURRENT_AMBIGUOUS`, caso defensivo que no debería ocurrir bajo `RN-C03` pero que la función ya contempla), este módulo —sin alterar `findCurrentPublishedPeriod`— desempata de forma determinista entre los candidatos que ella identificó (`startDate desc, id desc`).
3. Si no hay periodo vigente por fecha, usa el publicado más reciente (`isActive=true`, `startDate desc, id desc`).
4. Si no hay ningún periodo publicado, retorna `null` → el dashboard muestra el estado vacío "No hay periodos publicados disponibles para consultar.", no un error.

### 6.4 Selector de periodo (frontend)

- `DashboardFilters.tsx` agrega un `<select>` funcional poblado con `periodsService.listAll()` (ya existente, sin endpoint nuevo), filtrado para excluir `DRAFT` en el cliente (`p.status !== 'DRAFT'`). Los dos `<select>` de proyecto/centro de costo siguen deshabilitados, sin cambios.
- La carga inicial del dashboard se hace **sin** `periodId` explícito; el backend aplica la selección predeterminada (6.3) y la página fija `selectedPeriodId` a partir de `reportData.period.id` una vez llega la respuesta — una sola fuente de verdad para "cuál es el periodo por defecto", sin duplicar esa regla en el frontend.
- Cambiar el selector dispara `fetchDashboard(periodId)` con el id explícito.
- **Carrera de respuestas:** `EvaluationDashboardPage.tsx` usa un contador (`requestSeq`) incrementado en cada llamada; una respuesta o error cuya secuencia ya no es la vigente se descarta silenciosamente, de modo que una respuesta lenta de un periodo anterior nunca sobrescribe los datos del periodo seleccionado después.

### 6.5 Errores y estados vacíos (frontend)

- Error de carga (400/404/otro): se mantiene visible el selector de periodo (para poder corregir) y se muestra un mensaje distinto según el código HTTP, sin inventar datos ni cambiar de periodo solo.
- `period === null`: tarjeta "No hay periodos publicados disponibles para consultar."
- `period` presente y `hasEvaluations === false`: banner "Este periodo aún no tiene evaluaciones registradas.", y de todas formas se muestran KPIs/gráficas/tabla (en cero), conservando los componentes existentes tal como pedía el encargo.
- Encabezado (`DashboardFilters`) según `period.status`: `ACTIVE` → "Cierre: fecha (Quedan N días)"; `FINISHED` → "Periodo finalizado: fecha"; `SCHEDULED` → "Periodo programado — inicia: fecha". Nunca se muestran días restantes para un periodo histórico o programado.

### 6.6 Detalle de evaluador (modal)

- `EvaluatorDetailModal.tsx` obtiene el `periodId` activo vía `ReportPeriodContext` y lo agrega a los parámetros de `GET /reports/evaluations`, de modo que el drill-down siempre consulta el mismo periodo que el dashboard de fondo.
- Un `useEffect` separado detecta cambios de `periodId` (comparando contra un `ref` con el valor anterior): si el modal está abierto cuando el periodo cambia, limpia sus datos y llama a `onClose()` automáticamente. Si está cerrado, no hace nada (no interfiere con el flujo normal de apertura/cierre del usuario).
- Al volver a abrirse para cualquier evaluador, siempre usa el `periodId` vigente en ese momento (lectura directa del contexto en cada `fetchDetails()`).

### 6.7 Verificación realizada

- `node --check` sobre `reports.service.js` y `reports.controller.js`: sin errores de sintaxis.
- `npx tsc -b` (typecheck completo del proyecto `sisproing-app`, no solo los archivos tocados): **sin errores**.
- `npx eslint src/modules/employee-evaluations/reports`: 2 errores, ambos preexistentes y ajenos a esta mejora (`Charts.tsx` no tocado, y el `useState<any[]>` de `EvaluatorDetailModal.tsx` que ya existía en el archivo original). Ningún error nuevo introducido.
- **No se pudo ejecutar la suite Jest del backend** (`npm test`): el entorno de este contenedor tiene una incompatibilidad preexistente entre la versión instalada de `jest-circus` y la resolución de `exports` de Node (`ERR_PACKAGE_PATH_NOT_EXPORTED` al resolver `jest-circus/build/runner.js`), que afecta a **todos** los tests del repo (incluidos los ya existentes en `tests/employee-evaluations/*`, no solo los de reportes), no es causada por este cambio y está fuera del alcance de esta mejora corregir.
- **No se pudo ejecutar `npx vite build`** (bundling de producción del frontend): falta el binario opcional `@rollup/rollup-linux-arm64-gnu` en `node_modules` (bug conocido de npm con dependencias opcionales por arquitectura), preexistente y no relacionado con este cambio.
- No se escribieron pruebas automatizadas nuevas para `reports.service.js` porque la suite Jest no es ejecutable en este entorno para verificarlas; en su lugar, la lógica nueva (resolución de periodo, derivación de estado, códigos de error) se revisó manualmente contra cada decisión aprobada y queda cubierta por el checklist manual de la sección 6.8.
- No se crearon commits ni se ejecutaron migraciones de base de datos.

### 6.8 Riesgos y verificación manual pendiente

1. **Sin cobertura automatizada propia.** Por la limitación del entorno de pruebas (6.7), la resolución de periodo por defecto, los códigos 400/404 y el estado vacío no tienen un test automatizado que corra en CI todavía; quedan cubiertos solo por el checklist manual entregado junto con este resumen. Se recomienda, cuando el entorno de Jest se repare, agregar un test dirigido a `reportsService.getDashboardProgress`/`getEvaluationsList` (mockeando Prisma) para los casos de la sección 6.3 y los errores 400/404.
2. **Ambigüedad de "periodo vigente" nunca probada con datos reales.** El desempate determinista para el caso en que `findCurrentPublishedPeriod` reporte ambigüedad es defensivo (no debería ocurrir bajo `RN-C03`); no hay forma de ejercitarlo sin crear periodos solapados de prueba.
3. **`periodsService.listAll()` trae todos los periodos sin paginar.** Si el número de periodos históricos crece mucho con los años, el selector podría volverse largo; no se agregó paginación ni búsqueda al selector porque no estaba en el alcance de P3.2.
4. **No se verificó visualmente en navegador** (no se pudo correr `vite build`/`vite dev` en este entorno): la validación quedó en typecheck + eslint + lectura de código. El checklist manual (sección siguiente de este resumen) cubre justamente lo que falta verificar con la app corriendo.


## 7. Validación manual del selector — 2026-10-20

El responsable del proyecto probó el **filtro/selector de periodo del dashboard** en su entorno y reportó: **«funciona perfecto el filtro»**. Se registra como **validación manual satisfactoria del flujo principal de selección de periodo**, no como comprobación independiente del código o evidencia de todos los casos del checklist P3.2.

**Estado:** P3.2 implementado y selector principal validado por el responsable. Para cierre funcional completo de P3 siguen sin constancia individual de verificación: consistencia del `periodId` en el detalle del evaluador; alternancia rápida sin respuestas obsoletas; estados vacío/errores 400/404 y encabezados ACTIVE/FINISHED/SCHEDULED; además de las pruebas automatizadas que el entorno de Claude no permitió ejecutar. No se modificó código ni datos por este registro. Mantener P1/P2 y los scores históricos intactos.
