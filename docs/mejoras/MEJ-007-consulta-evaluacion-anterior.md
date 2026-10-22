# MEJ-007 — Descargar la evaluación anterior durante el diligenciamiento

**Proyecto:** PROING — Evaluación de Desempeño HF-011  
**Decisión funcional:** 2026-10-21  
**Estado:** **Implementación inicial reportada; ajuste de identidad SSO aprobado y pendiente de aplicar/validar**.  
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


## 7. Revisión de identidad SSO y decisión de compatibilidad (2026-10-21)

**Diagnóstico de Codex (sin cambios de código en la revisión):** la implementación inicial de MEJ-007 incorporó el claim global `employeeId` al JWT mediante `auth.service.js`, lo propagó mediante `auth.middleware.js` y añadió `authIdentity.utils.js`. Este dato se toma literalmente de `payload.employeeId` del SSO legado en Redis; **no está garantizado** que exista para todos los usuarios. Se observó `employeeId=null` en una sesión. El endpoint histórico nuevo exigía `req.user.employeeId`; por ello podía retornar 403 y ocultar el botón tanto para JWT antiguos sin claim como para JWT nuevos con claim nulo, aunque la evaluación actual pudiera diligenciarse normalmente.

**Decisión aprobada para el alcance limitado de MEJ-007:** revertir exclusivamente los cambios globales de JWT/SSO añadidos para esta mejora, siempre que no sean utilizados por otra funcionalidad, y reutilizar el mismo mecanismo de identificación ya usado en el módulo de evaluaciones: tras validar JWT y cargar `currentEvaluationId`, exigir `Number(req.user.id) === currentEvaluation.evaluatorId`; **solo después** derivar `currentEvaluation.employeeId` y buscar la evaluación histórica del empleado evaluado. No usar `historical.evaluatorId` como filtro; no recibir `employeeId` o `historicalEvaluationId` del cliente para autorizar. Conservar comprobaciones de permisos/propiedad en servidor y no expandir el acceso de `GET /evaluations/:id/pdf`.

**Límite de seguridad pendiente:** el esquema actual presupone para sus endpoints de evaluador que `JWT.sub` es compatible numéricamente con `Evaluation.evaluatorId`. La comprobación de igualdad mantiene la política preexistente, **pero no demuestra un vínculo SSO usuario↔empleado confiable**. Codex debe verificar con pruebas negativas, contraste de casos reales autorizados y casos de IDs cruzados que no aparezca un acceso indebido; no afirmar que la igualdad por sí sola resuelve el problema transversal. La normalización formal y fiable de `userId ↔ employeeId` se trabajará como mejora de autenticación separada, con contrato SSO comprobado.

**Regresión exigida:** usuario con JWT existente sin `employeeId` puede descargar histórico de un evaluado que tiene asignado; JWT con `employeeId=null` no oculta botón si el usuario está autorizado mediante mecanismo vigente; otro usuario que altera `currentEvaluationId` recibe 403/404; cambio de evaluador entre periodos no restringe búsqueda del evaluado; histórico sin `COMPLETED` no muestra acción; solicitud PDF no altera borrador/autoguardado y no crea acuses. Incluir `node:test`, typecheck, build y prueba con SSO real si hay acceso. No migraciones ni commits automáticos.

**Estado:** cambios iniciales MEJ-007 reportados por Codex, **pendientes del ajuste aprobado**; no desplegar versión que exige el nuevo claim global. Actualizar estado y evidencia después de ejecutar la corrección puntual y sus pruebas.
