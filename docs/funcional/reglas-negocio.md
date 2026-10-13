# Reglas de negocio implementadas

## Periodos

| ID | Regla comprobada | Evidencia / observación |
|---|---|---|
| RN-C01 | Nombre, inicio y fin son obligatorios; fin no puede ser anterior al inicio. | `periods.service.js#parseAndValidateDates`. |
| RN-C02 | Crear guarda un borrador con `isActive=false`. | `createPeriod`. |
| RN-C03 | No se permite solapamiento con otro periodo `isActive=true`. | `ensureNoActiveOverlap`; permite varios periodos publicados no solapados. |
| RN-C04 | Publicar exige al menos un grupo, pregunta, fortaleza, debilidad y escala activos. | `publishPeriod`; no exige clasificación. |
| RN-C05 | Publicar copia catálogos y asignaciones dentro de una transacción. | `publishPeriod`, `createSnapshotsForPeriod`, `generateEvaluationsForPeriod`. |
| RN-C06 | El estado visible se deriva: no publicado=`DRAFT`; publicado futuro=`SCHEDULED`; dentro de fechas=`ACTIVE`; vencido=`FINISHED`. | `getPeriodStatus`. |
| RN-C07 | “Periodo activo” en consultas significa `isActive=true`, no necesariamente fecha vigente. | `getActivePeriodOrThrow` y reportes. |

No hay cierre automático persistido. Un periodo vencido continúa con `isActive=true`.

## Elegibilidad y asignación

- Fuente: `gestion_humana.gh_empleados_jefes` mediante `EmployeeManager`.
- Deben existir `managerId`, jefe activo (`status=1`) y colaborador activo (`status=1`).
- No hay regla de antigüedad, proyecto, centro de costo o cargo.
- La relación se copia al publicar; los cambios posteriores de jefe no actualizan las evaluaciones existentes.
- La base impone una evaluación única por `periodId + evaluatorId + employeeId`.
- No se encontró bloqueo del cambio de jefe por evaluación pendiente en el API. Su posible existencia en PHP queda pendiente.

## Estados de evaluación

| Estado | Entrada | Salida permitida en código |
|---|---|---|
| `PENDING` | Asignación generada. | Guardar pasa a `IN_PROGRESS`; finalizar pasa a `COMPLETED`. |
| `IN_PROGRESS` | Primer borrador. | Más borradores o finalizar. |
| `COMPLETED` | Finalización. | Sin transición; borrador/finalización se rechazan. |

No hay cancelación, reapertura, aprobación ni estado específico de acuse. `notificationReceived` es un indicador separado.

## Validaciones de respuestas

- El frontend ofrece valores 1–5, fijos en `QuestionSectionCard.tsx`.
- El modelo guarda `score Int?` sin restricción de rango.
- El backend convierte con `Number()` pero no valida rango, entero, `NaN` ni pertenencia de la pregunta al periodo.
- Finalizar exige que todas las preguntas snapshot tengan algún registro de respuesta.
- Fortalezas y debilidades se reemplazan por completo en cada guardado; respuestas no incluidas permanecen.
- Solo `COMPLETED` bloquea edición backend. No se validan fechas del periodo ni identidad del evaluador.

## Avance

El formulario usa:

`avance = (preguntas con valor > 0 + tarea_fortalezas + tarea_debilidades) / (preguntas + 2) × 100`

- `tarea_fortalezas=1` si existe al menos una.
- `tarea_debilidades=1` si existe una debilidad y al menos uno de sus compromisos no está vacío.
- `COMPLETED` fuerza 100 % al volver a cargar.

El overview del evaluador define “pendientes” como `PENDING + IN_PROGRESS`, aunque la lista los separa.

## Puntaje y clasificación

La fórmula real, tanto en frontend como backend, es:

`puntaje = redondear_1_decimal((suma de respuestas / (total de preguntas del periodo × 5)) × 100)`

Las preguntas sin respuesta aportan cero porque permanecen en el denominador. No hay ponderación por grupo, pregunta, clasificación, proyecto ni fortaleza/debilidad.

**Ejemplo:** periodo con 10 preguntas y suma de 37 puntos. Máximo `10 × 5 = 50`; resultado `37 / 50 × 100 = 74,0 %`.

Para asignar categoría, el backend busca el primer `PeriodScoreScale` cuyo rango inclusivo cumpla `minValue <= 74 <= maxValue`. Persiste `scale.description` en `ratingLabel`, no `scale.label`. Si los rangos se solapan, el resultado depende del primer registro devuelto; no hay orden ni restricción de no solapamiento.

La escala configurable sirve para clasificar el porcentaje total y mostrar información. No controla los valores 1–5 de cada pregunta.

## Reportes

- Dashboard global: totales por estado y confirmadas.
- Por evaluador: cantidades por estado y confirmadas.
- Por proyecto del evaluado: total, completadas y porcentaje.
- No se calculan resultados por grupo ni total promedio consolidado.
- La tabla del evaluador calcula avance como `(COMPLETED + IN_PROGRESS) / total`, mientras la dona global usa solo `COMPLETED / total`.

## Reglas requeridas no implementadas de forma equivalente

- varias clasificaciones por debilidad;
- compromisos parametrizados;
- bloqueo de cambios de jefe durante pendientes;
- autorización por rol y por propiedad de la evaluación;
- cierre/reapertura controlados;
- ponderaciones o consolidados de resultado, si Gestión Humana los espera.
