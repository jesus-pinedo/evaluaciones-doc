# MEJ-007 — Consulta de la evaluación anterior durante el diligenciamiento

**Proyecto:** PROING — Evaluación de Desempeño HF-011  
**Fecha de propuesta:** 2026-10-21  
**Estado:** PROPUESTA UX/TÉCNICA — pendiente de aceptación funcional y de implementación.  
**Origen:** el responsable necesita consultar la evaluación previa de una persona mientras diligencia su evaluación del nuevo periodo. No está definido explícitamente en el requerimiento funcional original (UC-002), que sí establece el formulario HF-011, fortalezas, debilidades y compromisos.

## 1. Propuesta UX recomendada

**Panel lateral de solo lectura, activado mediante «Ver evaluación anterior».** Por defecto permanece cerrado y no realiza consultas hasta ser abierto.

- Desktop ancho: modo vista dividida con formulario actual y panel de evaluación anterior **simultáneamente visibles e interactuables**. El formulario conserva su autoguardado, campos, estado, scroll y navegación. El panel tiene scroll independiente y se puede cerrar sin desmontar ni resetear el formulario.
- Ventanas estrechas y móvil: panel ocupando ancho útil/fullscreen con control explícito para regresar. No degradar la usabilidad de respuestas 1–5.
- Colocar botón cerca del encabezado «Evaluación de Desempeño» o barra fija superior, sin desplazar el puntaje/progreso/indicador de guardado.
- Panel histórico: empleado, nombre/fechas del periodo, evaluador anterior **si está permitido**, score **persistido** y clasificación, fortalezas, debilidades y compromisos; grupos y sus preguntas/respuestas (1–5) como datos **de solo lectura**.
- Acción secundaria «Ver PDF completo» (abre en pestaña nueva o vista expandida) para documento original/rectificado según disponibilidad y reglas de custodia. No incrustar PDF como único medio de consulta: en un lateral angosto resulta difícil leer 3 páginas y no es útil para comparar varias respuestas.
- No asumir coincidencia de preguntas/grupos entre periodos. Mantener las secciones de cada periodo según sus snapshots, sin cruzar por posición o texto.
- Si no existe evaluación anterior completada accesible: mostrar «No hay evaluación anterior disponible», sin error, sin PDF. No mostrar 0/0 ficticio ni hacer que falle el formulario actual.

## 2. Semántica de «anterior»

Resolver **por empleado evaluado**, nunca por evaluador previo. A partir de `currentEvaluationId` autorizado, obtener `employeeId` y periodo actual; elegir la última evaluación de ese mismo empleado con `status='COMPLETED'` perteneciente a un periodo **anterior** al actual, ordenando por fecha de finalización del periodo (y desempate determinista). No elegir una evaluación de otra persona, ni depender de un periodo fijo (`6`/2025), ni de `isActive` del periodo como sinónimo de vigente. Definir explícitamente tratamiento de evaluaciones inactivas según autorización/visibilidad existente; por defecto no exponer evaluaciones invalidadas o inactivas sin regla comprobada.

El evaluador actual puede ser **distinto** del anterior; aun así, funcionalmente requiere ver el histórico de su evaluado cuando tiene una evaluación **actual asignada**.

## 3. Seguridad: precondición de implementación

La auditoría MEJ-004 documentó que `GET /evaluations/:id/pdf` exige ahora permiso `employee_evaluations.reports#view` **o** que `req.user.id === evaluation.evaluatorId`. Esta regla **no permite naturalmente** que un nuevo jefe consulte el PDF firmado por un jefe anterior. Además, MEJ-004 dejó documentado el riesgo de **IDOR** en `GET /evaluations/:id` y operaciones de edición. No resolver el nuevo caso aflojando permisos globales del PDF o del detalle.

Diseñar endpoint(s) **acotados al recurso actual** y protegidos por autorización en servidor, por ejemplo:

- `GET /api/employee-evaluations/evaluations/:currentEvaluationId/previous`
- `GET /api/employee-evaluations/evaluations/:currentEvaluationId/previous/pdf` (si el mecanismo PDF existente no permite a nuevo evaluador, ofrecer flujo autenticado por blob)

El servidor debe: autenticar JWT; **comprobar de forma confiable que el usuario puede evaluar el `currentEvaluationId`** (resolver identidad SSO/userId/employeeId según contrato real, sin asumir coincidencia numérica ni confiar en parámetros cliente); derivar empleado y periodo desde la evaluación actual; resolver solamente el historial de ese empleado; devolver datos estrictamente de solo lectura; no exponer historial de otros empleados cambiando IDs. Para usuarios de Reportes, conservar la autorización separada existente sin elevar permisos automáticamente.

No introducir JWT en URL de PDF. Para vista/descarga PDF, usar petición autenticada `Authorization: Bearer` y `Blob`/object URL con limpieza en frontend. Si existen PDF antiguos aceptados antes de MEJ-006, NO representarlos como un nuevo acuse al mostrar score rectificado: preserve trazabilidad documental.

## 4. Implementación incremental sugerida

**MEJ-007.1 — Inspección y contrato:** Auditar `EvaluationsPage`, `EvaluationsForm`, `useEvaluationForm`, `StickyHeader`, DTO `getEvaluationById`, método PDF, control real de acceso a evaluación actual y tablas de evaluación/periodos. Precisar si la UI histórica puede renderizarse con componentes read-only existentes sin tener hooks mutantes. Evaluar implicación de cambios de jefe.

**MEJ-007.2 — Backend:** Servicio read-only de evaluación anterior por `currentEvaluationId`, autorización por evaluación actual, orden estricto de periodos, DTO reducido/serializado (respuestas y fortalezas/debilidades/compromisos). Endpoint PDF secundario únicamente si el PDF existente no cubre el acceso por cambio de jefe. No actualizar datos ni generar acuses.

**MEJ-007.3 — Frontend:** Botón contextual, panel responsivo con scroll independiente; grupos y respuestas anteriores; resumen y PDF secundario. Estado de carga, error, no histórico, navegación y lazy loading. Conservar estado de edición y autoguardado cuando se abre/cierra el panel.

**MEJ-007.4 — Validación:** Usuario actualmente autorizado con evaluación anterior del mismo jefe; evaluador actual distinto al anterior; sin histórico; anterior no completada; evaluación actual pendiente/en curso; persistencia del borrador durante consulta; fecha/periodo correcto; JWT sin identificador empleado; intentos IDOR; acceso PDF con JWT; 8120 como histórico corregido (debe reflejar **61,4 % persistido** una vez desplegados P2/MEJ-006); mobile/responsive y PDF válido. No ejecutar migraciones ni correos.

## 5. Criterios de aceptación

1. Mientras responde el formulario 2026, el evaluador autorizado abre y lee el histórico sin perder respuestas ni interrumpir el autoguardado.
2. El histórico corresponde al **mismo empleado**, último periodo anterior completado y accesible, aunque haya cambiado el evaluador.
3. Se pueden ver puntaje histórico persistido, grupos/preguntas/respuestas, fortalezas/debilidades/compromisos. Nunca recalcular el score de `COMPLETED` desde snapshots `NULL`.
4. PDF disponible como acceso secundario seguro; no se habilitan enlaces públicos ni IDs arbitrarios.
5. Backend niega acceso a historial si el usuario no está autorizado para la evaluación **actual** (y mantiene los permisos existentes de Reportes).
6. Abrir/cerrar panel no modifica `score`, respuestas, `status`, `notificationReceived`, snapshots ni acuses.
7. Vista pequeña se adapta sin romper las respuestas 1–5. Existen estados vacíos y de error razonables.
8. No se generan migraciones ni commits automáticos; se documenta evidencia real de pruebas.
