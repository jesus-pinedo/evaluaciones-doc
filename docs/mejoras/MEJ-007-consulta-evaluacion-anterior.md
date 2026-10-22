# MEJ-007 — Descargar la evaluación anterior durante el diligenciamiento

**Proyecto:** PROING — Evaluación de Desempeño HF-011  
**Decisión funcional:** 2026-10-21  
**Estado:** **Implementación y validación funcional en pruebas confirmadas por el responsable; pendiente exclusivamente de despliegue y comprobación postdespliegue en producción**.  
**Origen:** el responsable requiere consultar la evaluación previa mientras diligencia la evaluación actual, sin restarle espacio ni protagonismo al formulario de 2026.

## 1. Decisión de experiencia de usuario

**Un botón secundario «Descargar evaluación anterior» debajo del título «Evaluación de Desempeño»**, preferiblemente a la derecha de la línea descriptiva del periodo/cargo o en una segunda línea si no hay espacio.

- Icono de descarga, variante secundaria/outline, sin aspecto de acción principal ni interferir con Guardado, progreso o Puntaje.
- **Un clic descarga el PDF de la última evaluación completada anterior del mismo empleado**. No abrir panel, modal, vista dividida, ni incrustar un PDF.
- Archivo con nombre descriptivo y seguro, por ejemplo `Evaluacion_Anterior_2025_8120.pdf`, derivado del periodo y evaluationId reales, no de un ID histórico fijo.
- Durante descarga mostrar estado de carga y prevenir clics repetidos; ante error mostrar mensaje no intrusivo y conservar formulario intacto.
- **Si no hay evaluación anterior COMPLETED elegible:** ocultar la acción y, si es útil, mostrar un mensaje discreto «Sin evaluación anterior disponible». No renderizar botón que descargue un documento vacío.
- Adaptación responsive conservando ancho completo del formulario y la escala 1–5.
- No cambiar maquetación, autoguardado, estado de preguntas, fortalezas, debilidades o compromisos de la evaluación actual.
- **Diseño sustituido:** se descarta la propuesta inicial de panel lateral, formulario dividido, modal grande o previsualización embebida. El PDF es la única funcionalidad nueva de consulta.

## 2. Qué se entiende por evaluación anterior

Desde la evaluación actual **autorizada**, tomar el `employeeId` del **evaluado** y su periodo; buscar la **última evaluación de estado `COMPLETED`** de ese mismo empleado perteneciente a un periodo anterior, ordenada por fecha de finalización del periodo (desempate determinista), sin fijar 2025 ni `periodId=6`. Si el evaluador anterior era otro jefe, esto no debe impedir la descarga siempre que el usuario esté autorizado sobre la evaluación actual. No usar el estado `isActive` del periodo como único criterio temporal. No exponer evaluaciones individuales marcadas inactivas/invalidadas sin verificar primero las reglas actuales de visibilidad. No descargar `PENDING` o `IN_PROGRESS`.

## 3. Seguridad — requisito obligatorio

La auditoría [MEJ-004](MEJ-004-consulta-descarga-evaluaciones.md) documenta que el PDF actual exige permiso de Reportes o ser el **evaluador histórico** (`req.user.id === evaluation.evaluatorId`), y que el endpoint de detalle y edición tiene un riesgo de IDOR preexistente. El evaluador **actual** podría ser distinto del anterior; **NO** relajar globalmente las reglas de `GET /evaluations/:id/pdf` ni habilitar la consulta arbitraria de `evaluationId`.

Implementación mínima recomendada, después de inspeccionar los contratos existentes:

1. `GET /api/employee-evaluations/evaluations/:currentEvaluationId/previous`: solo metadatos de disponibilidad (por ejemplo `available`, `periodName`) obtenidos bajo autorización de la evaluación actual. No devolver respuestas ni datos extra.
2. `GET /api/employee-evaluations/evaluations/:currentEvaluationId/previous/pdf`: **descarga autenticada** del PDF de la evaluación anterior resuelta por el servidor. Reutilizar el generador de PDF actual sin duplicarlo. Opcionalmente omitir la primera ruta si existe una fuente autorizada ya disponible en el DTO para saber la existencia del anterior.

En ambos: JWT válido; comprobar acceso a la **evaluación actual identificada por la ruta** y que pertenece al usuario evaluador autorizado (resolver cuidadosamente la correspondencia SSO `userId`/`employeeId`, sin asumir identidad numérica coincidente); derivar el empleado evaluado y su periodo **en el servidor**; no aceptar `historicalEvaluationId` ni `employeeId` de query/cliente como fuente de autorización; seleccionar sólo el histórico permitido de ese empleado. Rechazar IDOR con 403/404 según convención existente. Usar `Authorization: Bearer` en `apiFetch`, descargar como Blob, revocar URL objeto y no colocar JWT en URL.

Cuando no existe histórico, la ruta puede retornar respuesta 404 controlada o `available:false` en metadatos, nunca un PDF parcial o vacío.

## 4. Integridad histórica y P2 / MEJ-006

El PDF debe mostrar el **score persistido** de la evaluación `COMPLETED` (rectificado en MEJ-006 cuando aplique; caso de control `8120 = 61,4 %`) y sus respuestas snapshot originales, sin invocar un recálculo LEGACY a partir de pesos `NULL`. **No alterar `notificationReceived`, acuses o documentos originales**; descargar una versión generada tras rectificación no equivale a que el empleado la haya aceptado. Si el PDF actual muestra 61,8 para 8120 cuando la BD contiene 61,4, detenerse a corregir la lectura/versión desplegada antes de dar por finalizada MEJ-007.

## 5. Plan de implementación mínimo

- **MEJ-007.1 Diagnóstico dirigido:** inspeccionar React (`EvaluationsForm`, título actual, `useEvaluationForm` y servicio HTTP), Node (detalle actual, PDF, autorización por recurso, periodos), disponibilidad de evaluación anterior. Confirmar alcance exacto antes del cambio.
- **MEJ-007.2 Backend:** resolución read-only por `currentEvaluationId`, seguridad efectiva en servidor y descarga PDF reutilizando motor existente; información de disponibilidad mínima. Sin tocar el flujo común de edición ni abrir permisos indiscriminados.
- **MEJ-007.3 Frontend:** botón secundario discreto debajo/cerca del título; ocultar cuando no haya anterior; descargar Blob autenticado y manejar loading/error. No modal/panel/vista dividida.
- **MEJ-007.4 Pruebas:** evaluación actual `PENDING` o `IN_PROGRESS` con histórico `COMPLETED`; evaluador previo distinto; ausencia de histórico; histórico `PENDING` excluido; IDOR con otro `currentEvaluationId`; PDF de empleado/periodo correctos; caso 8120=61,4; sin pérdida de autoguardado/inputs; sin `UPDATE`, migraciones, notificaciones, cambios de acuse o regeneración persistente de PDF.

## 6. Criterios de aceptación

1. Al evaluar a una persona con histórico elegible, aparece **«Descargar evaluación anterior»** como acción secundaria junto al título, sin ocupar espacio adicional significativo.
2. Un clic descarga el PDF **correcto** del mismo empleado, del periodo anterior completado más reciente, aunque haya cambiado el jefe.
3. Sin histórico elegible, el botón no aparece; ausencia de histórico nunca afecta el formulario actual.
4. La descarga exige autorización por la **evaluación actual**; manipular IDs de ruta no permite acceder a PDFs de otros empleados.
5. PDF refleja `score` guardado de `COMPLETED` y respeta versiones/documentos/acuse históricos.
6. La operación es solo lectura, no altera el progreso, respuestas, autoguardado ni los metadatos de la evaluación actual.
7. Typecheck, pruebas dirigidas y validación responsiva. Documentar resultados reales y dejar pendientes visibles.
8. Sin migraciones ni commits locales automáticos.

<<<<<<< HEAD
## 7. Implementación realizada (2026-10-21)

### 7.1 Diagnóstico y decisiones técnicas verificadas

- El formulario vigente está en `sisproing-app/src/modules/employee-evaluations/evaluations/pages/EvaluationsForm.tsx`; el título y periodo/cargo se renderizan antes del `<form>`. La acción nueva se agregó en ese encabezado mediante un componente aislado, sin modificar `useEvaluationForm`, autoguardado, respuestas, finalización ni navegación.
- `apiFetch` ya agrega el JWT en `Authorization: Bearer` y soporta `responseType: 'blob'`; se reutilizó ese contrato. El JWT no se coloca en query string ni URL de descarga.
- El generador existente es `api/src/modules/employee-evaluations/evaluations/evaluationPdf.service.js#generateEvaluationPdf`; la ruta nueva lo invoca directamente con el histórico resuelto por el servidor. No se creó ni duplicó un motor PDF.
- El DTO que alimenta la vista PDF, `evaluations.helper.js#formatEvaluationForFrontend`, usa `Evaluation.score` persistido cuando `status=COMPLETED`. La prueba dirigida confirma que un valor persistido `61.4` llega como `meta.score=61.4`, sin recalcular LEGACY.
- El flujo operativo vigente identifica al evaluador mediante `JWT.sub → req.user.id`: overview, listado y autorización del PDF existente comparan ese valor con `Evaluation.evaluatorId`. Aunque el modelo Prisma relaciona `evaluatorId` con `Employee.id`, no está demostrado que el SSO y Gestión Humana compartan formalmente el mismo dominio de IDs. MEJ-007 conserva esta política vigente únicamente para autorizar la evaluación actual; no modifica el contrato global SSO/JWT ni presenta esa comparación como solución definitiva de identidad.

### 7.2 Backend implementado

Nuevas rutas autenticadas bajo el middleware JWT global:

- `GET /api/employee-evaluations/evaluations/:currentEvaluationId/previous`
- `GET /api/employee-evaluations/evaluations/:currentEvaluationId/previous/pdf`

Ambas rutas:

1. reciben únicamente el ID de la evaluación actual;
2. toman el usuario autenticado de `req.user.id`, ya derivado de `JWT.sub` por el middleware vigente;
3. comprueban que la evaluación actual existe, está activa y que `Number(req.user.id) === Number(current.evaluatorId)`;
4. derivan `current.employeeId` y el inicio del periodo en servidor;
5. buscan `COMPLETED`, `isActive=true`, del mismo evaluado y con `period.endDate < current.period.startDate`;
6. ordenan por fin de periodo descendente, inicio descendente, `periodId` descendente e `evaluationId` descendente;
7. no filtran por evaluador histórico, por lo que admiten un jefe anterior diferente;
8. no aceptan `employeeId` ni `historicalEvaluationId` del cliente.

La ruta de disponibilidad devuelve solo `available` y, cuando corresponde, `periodName` y un nombre de archivo seguro. La descarga vuelve a resolver el histórico, responde 404 si ya no existe y entrega `Cache-Control: private, no-store`. No se modificaron las reglas del endpoint general `GET /evaluations/:id/pdf`.

El `employeeId` del JWT no participa en esta autorización: un token existente sin ese claim y un token que contenga `employeeId=null` siguen usando `sub/req.user.id`, sin exigir renovación de sesión. Si el usuario autenticado no es propietario de la evaluación actual, el servidor responde 403 antes de consultar cualquier histórico. La búsqueda posterior utiliza exclusivamente `currentEvaluation.employeeId`, que es el empleado **evaluado**, y nunca el identificador del evaluador histórico.

**Riesgo conservado y delimitado:** esta comparación numérica mantiene compatibilidad con el flujo actual, pero no demuestra que `userId` y `Employee.id` sean identidades equivalentes. La normalización formal del contrato SSO debe tratarse como una mejora transversal separada, después de verificar el origen del payload y su cobertura; no debe introducirse de forma incidental en MEJ-007.

### 7.3 Frontend implementado

`PreviousEvaluationDownloadButton` consulta disponibilidad al cargar la evaluación y no renderiza nada cuando `available=false` o la consulta auxiliar falla. Si hay histórico, muestra el botón outline **«Descargar evaluación anterior»** debajo del periodo/cargo, con icono de descarga.

Durante la descarga deshabilita la acción y muestra **«Descargando...»**. Descarga el Blob autenticado mediante un enlace temporal con nombre entregado por el servidor, elimina el enlace y revoca la URL objeto. Los errores de descarga usan la notificación no intrusiva existente. No hay modal, panel, vista dividida ni previsualización.

### 7.4 Archivos modificados por MEJ-007

**API**

- `src/modules/employee-evaluations/evaluations/previousEvaluation.utils.js` (nuevo)
- `src/modules/employee-evaluations/evaluations/evaluations.service.js`
- `src/modules/employee-evaluations/evaluations/evaluations.controller.js`
- `src/modules/employee-evaluations/evaluations/evaluations.route.js`
- `tests/employee-evaluations/mej-007-previous-evaluation.test.mjs` (nuevo)

**Frontend**

- `src/modules/employee-evaluations/evaluations/components/PreviousEvaluationDownloadButton.tsx` (nuevo)
- `src/modules/employee-evaluations/evaluations/components/Icons.tsx`
- `src/modules/employee-evaluations/evaluations/evaluations.service.ts`
- `src/modules/employee-evaluations/evaluations/pages/EvaluationsForm.tsx`

**Documentación**

- `docs/mejoras/MEJ-007-consulta-evaluacion-anterior.md`

No se modificaron esquema Prisma, migraciones, periodos, snapshots, respuestas, calificaciones, acuses ni datos históricos. No se enviaron correos ni se crearon commits.

Tras la revisión específica de autenticación, `api/src/modules/auth/auth.service.js` y `api/src/core/middlewares/auth.middleware.js` quedaron exactamente sin cambios atribuibles a MEJ-007. El helper temporal `authIdentity.utils.js` fue eliminado porque quedó sin referencias.

### 7.5 Pruebas ejecutadas

| Verificación | Resultado |
|---|---|
| Línea base previa: `npm run typecheck` en frontend | **Aprobada** |
| Línea base previa: `npm run type-check` en API | **Aprobada** |
| `node --check` de los archivos JS backend tocados/nuevos | **Aprobada** |
| `node --test tests/employee-evaluations/mej-007-previous-evaluation.test.mjs` | **8/8 aprobadas** |
| JWT existente sin claim `employeeId` | **Aprobada:** autentica, conserva `sub → req.user.id` y autoriza la evaluación actual propia |
| JWT con `employeeId=null` | **Aprobada:** autentica y autoriza la evaluación actual propia sin depender del claim |
| IDOR: `currentEvaluationId` perteneciente a otro evaluador | **Aprobada:** 403 lógico antes de ejecutar la consulta histórica |
| Selección: mismo evaluado, solo `COMPLETED` activo, periodo anterior, sin filtro por jefe histórico, orden determinista | **Aprobada** |
| Ausencia de histórico retorna resultado vacío controlado | **Aprobada** |
| Descarga/resolución de histórico no modifica la evaluación actual | **Aprobada:** únicamente `findUnique` y `findFirst`; ninguna operación de escritura |
| Contrato PDF: nombre seguro y `score` persistido del caso sintético 8120 = `61.4` | **Aprobada** |
| `npm run type-check` en API después del cambio | **Aprobada** |
| `npm run build` en API después del cambio | **Aprobada** |
| `npm run typecheck` y `npm run build` en frontend después del cambio | **Aprobadas** |
| Render Chromium de la vista PDF del build, con fixture sintético 8120 | **Aprobada:** A4, 2 páginas, 397.153 bytes, sin errores de página; contiene `8120` y `61.4%` antes de emitir el PDF |
| `git diff --check` en API y frontend | **Aprobada** |

El lint dirigido del componente nuevo y del icono frontend pasó. El lint dirigido de archivos frontend ya existentes reportó cuatro usos preexistentes de `any` (`evaluations.service.ts` y `EvaluationsForm.tsx`) fuera de las líneas agregadas. El repositorio API no tiene ESLint instalado/configurado localmente; `npx eslint` intentó usar ESLint 10 y no encontró `eslint.config.*`, por lo que se usaron `node --check`, typecheck y pruebas ejecutables como validación backend.

La suite general `npm test -- --runInBand` continúa fallando por una incompatibilidad preexistente: Jest 30 descubre seis archivos construidos con `node:test`, ejecuta sus casos (la salida del runner Node registra 50 aprobados y 1 omitido) y después marca cada archivo con «Your test suite must contain at least one test». El test MEJ-007 se dejó como `.test.mjs` para ejecutarlo explícitamente con `node --test` y evitar agregar otro falso fallo al patrón Jest configurado para `.test.js`.

### 7.6 Pendientes de validación integrada

- Verificar en una mejora transversal separada si `JWT.sub/userId` y `gestion_humana.gh_empleados.em_id` comparten realmente el mismo dominio. MEJ-007 conserva la política operativa actual, pero no resuelve ese contrato.
- Ejecutar en navegador los casos con datos autorizados: jefe anterior diferente, ausencia de histórico, histórico `PENDING` excluido, manipulación de `currentEvaluationId` y descarga real.
- Descargar el PDF productivo de control 8120 y confirmar visualmente `61,4 %`. La prueba automatizada demuestra que el código usa el valor persistido y que la vista lo renderiza; no consultó la base productiva. La evidencia de MEJ-006 registra la actualización productiva reportada a 61,4, pero mantiene pendiente esa validación funcional.

### 7.7 Validación funcional por responsable y pase a producción (2026-10-21)

El responsable confirmó que la función **«Descargar evaluación anterior» funciona correctamente** durante las pruebas de la evaluación actual. Esta confirmación funcional se agrega a las pruebas automatizadas (8/8), typechecks y builds de API/React ya reportados. **Todavía no se ha desplegado MEJ-007 a producción.** El ajuste de identidad global inicialmente propuesto se revirtió: no se exige claim `employeeId` nuevo ni renovar sesiones.

**Checklist operativo para despliegue:**

1. Revisar y registrar diff/commit final separado en API y frontend; confirmar expresamente que `auth.service.js` y `auth.middleware.js` no incluyen cambios de MEJ-007 y que `authIdentity.utils.js` temporal no forma parte del build.
2. Desplegar API antes que frontend para exponer `GET /evaluations/:currentEvaluationId/previous` y `GET /evaluations/:currentEvaluationId/previous/pdf`, sin cambios de esquema, migraciones, backfill, cron ni correos.
3. Desplegar `sisproing-app`, comprobando que el bundle corresponde a la versión final con botón secundario y que React/API conservan los cambios P2 que muestran el `score` persistido de evaluaciones completadas.
4. Smoke test productivo con usuario autorizado: caso con histórico descargable, ausencia de histórico, jefe histórico diferente si disponible, enlace PDF con periodo/empleado correctos, formulario/guardado no afectados y respuesta 403 para `currentEvaluationId` ajeno. Comprobar 8120: `score=61,4 %` en PDF si es accesible.
5. Monitorear logs HTTP/API, errores 401/403/404/5xx y descarga del navegador; preparar rollback a las builds previas si falla. Mantener abierta la deuda transversal de identidad SSO `userId` vs `Employee.id`.

**Estado: VALIDADO EN PRUEBAS, PENDIENTE DE DESPLIEGUE A PRODUCCIÓN.**
