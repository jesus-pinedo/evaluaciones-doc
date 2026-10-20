# MEJ-002 — Auditoría del cálculo de puntuaciones

**Proyecto:** PROING — Evaluación de Desempeño HF-011

**Fecha de auditoría:** 2026-10-16

**Estado:** **Auditoría terminada; reglas funcionales pendientes de aprobación**

**Alcance:** cálculo de respuestas, puntaje total, avance, clasificación, grupos, PDF y consumo puntual en reportes/correo.

**Intervención realizada:** documentación únicamente. No se modificaron API, React, Prisma, datos, periodos, snapshots ni evaluaciones históricas.

## 1. Contexto funcional

HF-011 conserva tres indicadores que deben tratarse por separado:

1. **Avance del formulario:** porcentaje de tareas diligenciadas. Incluye preguntas, selección de fortalezas y selección de debilidades con compromiso.
2. **Puntaje por grupo:** resultado de las preguntas pertenecientes a un grupo. **No está implementado actualmente** ni en Node.js ni en React.
3. **Puntaje total:** porcentaje de puntos obtenidos sobre el máximo de todas las preguntas del periodo.

La auditoría confirma que el total vigente trata todas las preguntas por igual. El grupo solo organiza preguntas visualmente; no aporta un peso matemático. Evidencia: `api/src/modules/employee-evaluations/evaluations/evaluations.helper.js#calculateAndSaveScore` (líneas 235-297), `sisproing-app/src/modules/employee-evaluations/evaluations/hooks/useEvaluationForm.ts#stats` (líneas 110-170) y los modelos `QuestionGroup`/`PeriodGroup` de `api/prisma/modules/evaluations.prisma` (líneas 64-79 y 149-169).

Esta revisión usa el código actual como fuente de verdad. No se consultó la base de datos ni se ejecutaron operaciones sobre evaluaciones reales.

## 2. Resumen ejecutivo

| Hallazgo | Estado | Evidencia principal |
|---|---|---|
| Node.js calcula `suma de respuestas / (cantidad total de preguntas × 5) × 100`. | **Implementado** | `evaluations.helper.js#calculateAndSaveScore`, líneas 253-269. |
| React calcula en vivo la misma fórmula total. | **Implementado** | `useEvaluationForm.ts#stats`, líneas 155-167. |
| El resultado se redondea a un decimal con `toFixed(1)` y luego se convierte a número. | **Implementado** | `evaluations.helper.js`, líneas 266-270; `useEvaluationForm.ts`, líneas 164-167. |
| Las preguntas sin respuesta siguen en el denominador y equivalen a cero en el puntaje parcial. | **Implementado** | `evaluations.helper.js`, líneas 253-269; `useEvaluationForm.ts`, líneas 155-167. |
| El backend recalcula y persiste `score` también al guardar borrador, no solo al finalizar. | **Implementado** | `evaluations.service.js#saveDraft`, líneas 433-504. |
| No existe puntaje por grupo. | **No encontrado** | El helper cuenta todas las preguntas del periodo sin agrupar (`evaluations.helper.js`, líneas 236-269); React aplana todas las secciones (`useEvaluationForm.ts`, líneas 118-120). |
| No existe campo de peso ni interpretación de porcentajes incluidos en nombres de grupos. | **No encontrado** | `evaluations.prisma`, modelos `QuestionGroup` y `PeriodGroup`, líneas 64-79 y 149-169; el DTO copia `description` como `title` en `evaluations.helper.js`, líneas 130-167. |
| La evaluación completada devuelve desde Node el `score` persistido. | **Implementado** | `evaluations.helper.js#formatEvaluationForFrontend`, líneas 75-88. |
| El formulario React ignora ese `meta.score` para el encabezado y recalcula desde respuestas, incluso en modo completado. | **Inconsistencia** | `useEvaluationForm.ts`, líneas 108-170; `EvaluationsForm.tsx`, líneas 128-132. |
| El PDF usa `meta.score`, que para una evaluación completada es el valor persistido. | **Implementado** | `EvaluationPrintPage.tsx`, líneas 467-479; `evaluationPdf.service.js`, líneas 24-30 y 77-83. |
| La clasificación persistida y la clasificación mostrada en PDF siguen algoritmos distintos. | **Inconsistencia** | `evaluations.helper.js`, líneas 272-295; `EvaluationPrintPage.tsx#getScaleResult`, líneas 70-76 y 118-132. |
| Los reportes actuales son de avance; no consultan ni agregan `score` o `ratingLabel`. | **No encontrado** | `reports.service.js#getDashboardProgress`, líneas 41-83 y 86-201; `getEvaluationsList`, líneas 357-420. |

## 3. Fórmulas actuales

### 3.1 Valor de cada pregunta

React presenta cinco opciones fijas y envía el número seleccionado:

| Valor | Etiqueta visual |
|---:|---|
| 1 | Muy Bajo |
| 2 | Bajo |
| 3 | Moderado |
| 4 | Alto |
| 5 | Muy Alto |

Evidencia: `sisproing-app/src/modules/employee-evaluations/evaluations/components/QuestionSectionCard.tsx#QUESTION_SCALE`, líneas 24-31, y asignación mediante `field.onChange(scale.value)`, líneas 142-165.

No existe una fórmula separada por pregunta: el entero elegido es el aporte directo al numerador. Si se expresa ese aporte individual como porcentaje respecto del máximo fijo, sería:

`porcentaje_pregunta = respuesta / 5 × 100`

Esa expresión es una equivalencia matemática para explicar el total; el sistema no la calcula ni la almacena por pregunta. Prisma guarda la respuesta en `EvaluationAnswer.score Int?` (`api/prisma/modules/evaluations.prisma`, líneas 327-343).

El backend convierte el valor recibido con `Number(score)` y lo guarda mediante `upsert`, tanto en borrador como al finalizar (`evaluations.service.js`, líneas 433-452 y 559-581). No hay validación explícita de rango 1–5, de pertenencia de la pregunta al periodo ni de valor entero en esas funciones. La interfaz normal sí limita la selección a los cinco botones indicados. Por API, una respuesta mayor que 5 o una pregunta snapshot de otro periodo puede entrar al numerador mientras el denominador sigue contando solo las preguntas del periodo de la evaluación; esto permite resultados fuera de 0–100 si la base acepta el valor (`evaluations.helper.js`, líneas 236-269; relaciones de `EvaluationAnswer` en `evaluations.prisma`, líneas 327-343).

### 3.2 Puntaje total implementado en Node.js

Sean:

- `R` = conjunto de registros de respuesta asociados a la evaluación;
- `Q` = conjunto de preguntas snapshot del periodo;
- `r_i` = `score` de cada respuesta;
- máximo fijo por pregunta = `5`.

La fórmula real es:

`score = redondear_1_decimal((Σ r_i / (|Q| × 5)) × 100)`

Implementación exacta:

- carga todas las respuestas de la evaluación y cuenta todas las preguntas del periodo: `evaluations.helper.js#calculateAndSaveScore`, líneas 235-247;
- suma `curr.score || 0`: líneas 253-257;
- calcula `totalQuestions × 5`: líneas 259-262;
- aplica `parseFloat(...toFixed(1))`: líneas 264-270;
- persiste el resultado en `Evaluation.score`: líneas 288-295.

Si el periodo no tuviera preguntas, el resultado queda en `0` por la guarda `maxPossibleScore > 0` (`evaluations.helper.js`, líneas 264-270). La publicación normal exige que exista al menos una pregunta activa (`periods.service.js`, líneas 229-255), pero el helper es defensivo frente a denominador cero.

### 3.3 Puntaje total implementado en React

React implementa:

`score_visual = redondear_1_decimal((Σ respuestas_del_formulario / (cantidad_de_preguntas_en_secciones × 5)) × 100)`

El hook aplana las preguntas de todas las secciones, obtiene todos los valores del objeto `answers`, suma cada valor usando cero como fallback y aplica el mismo redondeo de un decimal (`useEvaluationForm.ts#stats`, líneas 118-120 y 155-167).

En el flujo normal, React y Node coinciden porque ambos usan máximo fijo 5, todas las preguntas como denominador y `toFixed(1)`. Existe una diferencia estructural: Node calcula sobre filas actualmente persistidas, mientras React calcula sobre el estado local de React Hook Form. Durante los dos segundos del `debounce`, el valor visual puede ir por delante de la base (`useEvaluationForm.ts`, líneas 190-217). Después de guardar, el endpoint solo retorna éxito; el hook no reemplaza el cálculo local con el `score` persistido (`useEvaluationForm.ts`, líneas 172-188; `evaluations.controller.js#finalizeEvaluation`, líneas 138-143).

### 3.4 Puntaje por grupo

**No existe un cálculo por grupo en el código actual.**

- Node cuenta `evaluation.period.questions.length` y no agrupa por `groupId` en `calculateAndSaveScore` (`evaluations.helper.js`, líneas 236-269).
- React convierte todas las secciones en una sola lista con `flatMap` y calcula un único total (`useEvaluationForm.ts`, líneas 118-120 y 155-169).
- El formulario usa cada grupo solo como tarjeta/sección (`EvaluationsForm.tsx`, líneas 143-152; `QuestionSectionCard.tsx`, líneas 68-83).
- El PDF imprime encabezados de grupo y respuestas, pero no subtotaliza ni calcula porcentajes de grupo (`EvaluationPrintPage.tsx`, líneas 483-527).

Si P2 decide mostrar un resultado simple por grupo sin ponderaciones, la fórmula matemáticamente consistente con el total actual sería:

`porcentaje_grupo = redondear_1_decimal((suma de respuestas del grupo / (preguntas del grupo × 5)) × 100)`

Esta es una **alternativa propuesta**, no comportamiento implementado.

## 4. Implementación del backend

### 4.1 Borrador y puntuación parcial

`saveDraft` realiza `upsert` de cada respuesta incluida en el payload y no elimina respuestas omitidas (`evaluations.service.js`, líneas 433-453). Luego ejecuta `calculateAndSaveScore` dentro de la misma transacción y cambia `PENDING` a `IN_PROGRESS` (`evaluations.service.js`, líneas 488-504).

Por tanto:

- `score` se guarda desde el primer borrador, aunque el comentario del modelo lo describa como resultado final;
- una pregunta nunca respondida cuenta en el denominador y aporta cero;
- omitir de un payload una respuesta ya guardada no la borra: conserva su valor anterior;
- el puntaje parcial está expresado contra el máximo de **todo** el cuestionario, no contra las preguntas respondidas.

La lectura de una evaluación no completada vuelve a calcular un valor desde respuestas y lo entrega como `meta.score`, en vez de usar necesariamente el parcial almacenado (`evaluations.helper.js#formatEvaluationForFrontend`, líneas 53-80).

### 4.2 Finalización y puntuación definitiva

`finalizeEvaluation` guarda las respuestas recibidas, consulta todos los IDs de respuesta ya existentes y exige que cada pregunta snapshot del periodo tenga un registro (`evaluations.service.js`, líneas 559-581 y 609-631). Después invoca el mismo `calculateAndSaveScore` y cambia el estado a `COMPLETED` (`evaluations.service.js`, líneas 633-644).

La completitud valida **existencia del registro**, no el rango ni la validez del valor. `EvaluationAnswer.score` es nullable y las sumas usan `score || 0` (`evaluations.prisma`, línea 336; `evaluations.helper.js`, líneas 253-257). En el flujo React normal se reciben números 1–5, pero el backend no protege por sí mismo esa regla.

No hay una fórmula especial de cierre: borrador y finalización llaman al mismo helper. Una evaluación `COMPLETED` no se puede guardar ni finalizar otra vez mediante estos servicios (`evaluations.service.js`, líneas 425-431 y 551-557).

### 4.3 Persistencia y posibilidad de recálculo

El modelo `Evaluation` guarda:

- `score Float?`;
- `ratingLabel String?`.

Evidencia: `api/prisma/modules/evaluations.prisma`, líneas 288-324, especialmente 317-319. Las respuestas originales permanecen en `EvaluationAnswer` y tienen unicidad por evaluación/pregunta (`evaluations.prisma`, líneas 327-343).

No se encontró un endpoint de recálculo masivo o individual independiente. El recálculo persistente ocurre únicamente al guardar borrador o finalizar (`evaluations.service.js`, líneas 488-489 y 633-634). Como ambos flujos rechazan `COMPLETED`, el camino normal no sobrescribe el `score` histórico final.

En una consulta, `formatEvaluationForFrontend` sí calcula siempre un `computedScore`, pero para una evaluación completada con `storedScore` numérico entrega el persistido (`evaluations.helper.js`, líneas 53-80). Si una evaluación completada tuviera `score = null`, entregaría el cálculo actual como fallback sin persistirlo.

### 4.4 Clasificación final (`ratingLabel`)

Después de redondear el score a un decimal, Node busca el primer `PeriodScoreScale` que cumpla simultáneamente:

`minValue <= score` y `maxValue >= score`

Evidencia: `evaluations.helper.js#calculateAndSaveScore`, líneas 272-286. Si encuentra escala, persiste **`scale.description`**, no `scale.label`, en `ratingLabel` (`evaluations.helper.js`, líneas 284-295).

La escala es snapshot del periodo: al publicar se copian `label`, `description`, `minValue`, `maxValue` y `sortOrder` desde el catálogo global (`periodSnapshots.service.js#createSnapshotsForPeriod`, líneas 286-303). Los modelos usan límites enteros (`PeriodScoreScale.minValue/maxValue Int`) y no tienen restricción de cobertura, continuidad o no solapamiento (`evaluations.prisma`, líneas 242-259).

Riesgos de la clasificación backend:

- `findFirst` no incluye `orderBy`; si existen rangos solapados, no se define por código cuál gana (`evaluations.helper.js`, líneas 275-282).
- como el score tiene un decimal y los límites son enteros, rangos consecutivos configurados como, por ejemplo, `0–59` y `60–69` dejan sin categoría valores `59.1–59.9`;
- si hay hueco, no se encuentra escala y `ratingLabel` queda `null`;
- el DTO de detalle no incluye `ratingLabel`: `formatEvaluationForFrontend` desestructura `score` pero no `ratingLabel` y construye `meta` sin él (`evaluations.helper.js`, líneas 9-19 y 82-117). Por ello, el valor persistido no es consumido por el formulario o el PDF actual.

## 5. Implementación del frontend

### 5.1 Actualización visual

`useWatch` observa `answers`, `strengths` y `weaknesses` (`useEvaluationForm.ts`, líneas 52-69). El `useMemo` de estadísticas se vuelve a ejecutar con cualquiera de esos cambios (`useEvaluationForm.ts`, líneas 110-170). Al seleccionar una respuesta, `QuestionSectionCard` actualiza el campo inmediatamente y dispara autoguardado (`QuestionSectionCard.tsx`, líneas 142-165).

El resultado visible del encabezado proviene de `stats.score`, no de `data.meta.score` (`EvaluationsForm.tsx`, líneas 128-132). Esto se aplica también cuando `isReadOnly` es verdadero, ya que el cálculo de `stats` no bifurca por estado (`useEvaluationForm.ts`, líneas 108-170).

Consecuencia: al abrir una evaluación histórica completada, el API entrega el valor persistido en `meta.score`, pero el encabezado React lo vuelve a derivar de las respuestas y de la estructura actual recibida. Con la fórmula actual y snapshots intactos debería coincidir; si la fórmula React cambia, hay datos anómalos o un resultado histórico fue calculado con otra versión, puede diferir del PDF.

### 5.2 Escala por pregunta y escala global

La escala de respuesta 1–5 está codificada en `QuestionSectionCard.tsx` y no usa `formConfig.scales` (`QuestionSectionCard.tsx`, líneas 24-31 y 94-107). En cambio, `formConfig.scales` representa rangos porcentuales del resultado total y se muestra en `GlobalScaleInfo` (`evaluations.helper.js`, líneas 119-128; `GlobalScaleInfo.tsx`, líneas 24-55).

Son dos conceptos diferentes:

- escala de pregunta: cinco valores fijos 1–5;
- escala de clasificación final: rangos configurables 0–100 copiados al periodo.

El color del recuadro de puntaje del formulario tampoco usa los rangos configurables: aplica umbrales fijos `>=90`, `>=70`, `>=50` (`StickyHeader.tsx`, líneas 16-20 y 80-84). Es una señal visual, no el `ratingLabel` final, pero puede sugerir una clasificación distinta de la escala del periodo.

### 5.3 Avance del formulario

React calcula:

`avance = redondear_entero((preguntas respondidas válidas + tarea_fortalezas + tarea_debilidades) / (total_preguntas + 2) × 100)`

Donde:

- una pregunta cuenta si pertenece a las secciones y su valor es numérico mayor que cero;
- fortalezas aporta una tarea si hay al menos una seleccionada;
- debilidades aporta una tarea si existe al menos una selección y alguno de sus compromisos contiene texto no vacío.

Evidencia: `useEvaluationForm.ts`, líneas 118-153. El avance se muestra como porcentaje y barra, mientras el puntaje se presenta en un recuadro separado (`StickyHeader.tsx`, líneas 69-84).

Node calcula un avance equivalente para datos normales y fuerza 100 % cuando el estado es `COMPLETED` (`evaluations.helper.js#formatEvaluationForFrontend`, líneas 21-51). Hay una diferencia defensiva: React solo cuenta respuestas pertenecientes a preguntas visibles y con número mayor que cero, mientras Node usa `answers.length` sin validar allí ID ni valor (`useEvaluationForm.ts`, líneas 122-131; `evaluations.helper.js`, líneas 27-46). Ante una fila inválida, cero o ajena al periodo, los avances pueden diferir. Además, `EvaluationsForm` usa `stats.progress` recalculado en React y no `meta.progressPercentage` (`EvaluationsForm.tsx`, líneas 128-132). React también usa ese avance para habilitar “Finalizar y Enviar” (`EvaluationsForm.tsx`, líneas 47-49 y 216-224).

El avance del formulario no participa en el puntaje de desempeño. Fortalezas, debilidades y compromisos afectan el avance, pero no el numerador ni el denominador de `score` (`useEvaluationForm.ts`, líneas 133-169; `evaluations.helper.js`, líneas 53-73).

Existe además otro “avance” fuera del formulario: “Mi equipo” usa evaluaciones completadas sobre asignadas (`evaluations.service.js#getMyTeamEvaluationsOverview`, líneas 155-172) y el dashboard agrega estados, no respuestas (`reports.service.js`, líneas 86-201). Esos indicadores tampoco son puntaje de desempeño.

### 5.4 Sincronización con persistencia

La UI cambia el puntaje en el mismo render en que cambia una respuesta. El guardado se ejecuta con `debounce` de 2 segundos (`useEvaluationForm.ts`, líneas 190-217). Por ello puede existir temporalmente:

`score visual nuevo ≠ score persistido anterior`

Tras un guardado exitoso, el cliente solo marca `saveStatus='saved'`; no usa una respuesta de score del backend ni vuelve a consultar el detalle (`useEvaluationForm.ts`, líneas 172-188). Al finalizar, el endpoint también devuelve solo éxito y React cambia únicamente `meta.status` a `COMPLETED` (`useEvaluationForm.ts`, líneas 228-253; `evaluations.controller.js`, líneas 138-143).

En condiciones normales las fórmulas coinciden y la divergencia es transitoria. No obstante, el diseño no verifica explícitamente que el resultado persistido sea igual al mostrado.

## 6. Grupos y ponderaciones

### 6.1 Naturaleza de los porcentajes en los nombres

Los textos como “CALIDAD (10%)” o “CULTURA ORGANIZACIONAL - ACTITUD (15%)” son actualmente solo `description`:

- `QuestionGroup` tiene `id`, `description`, auditoría, `isActive` y relación con preguntas; no tiene peso (`evaluations.prisma`, líneas 64-79);
- `PeriodGroup` copia `description` y tampoco tiene peso (`evaluations.prisma`, líneas 149-169);
- el helper pasa `g.description` sin transformación como `section.title` (`evaluations.helper.js`, líneas 130-167);
- `QuestionSectionCard` y el PDF imprimen ese título literalmente (`QuestionSectionCard.tsx`, líneas 75-83; `EvaluationPrintPage.tsx`, líneas 509-525).

No se encontró código que extraiga `%`, use una expresión regular sobre el nombre o convierta el texto entre paréntesis en número. Tampoco existe una columna de ponderación en los modelos globales o snapshot.

### 6.2 Efecto real de grupos con distinta cantidad de preguntas

Aunque no exista peso explícito, la fórmula total actual produce una ponderación implícita por cantidad de preguntas. Un grupo con 10 preguntas puede aportar hasta 50 puntos al numerador y otro con 2 preguntas solo hasta 10. En términos de porcentaje de grupo, el método actual equivale a:

`total = Σ(porcentaje_grupo × cantidad_preguntas_grupo) / Σ(cantidad_preguntas_grupo)`

La equivalencia proviene de la fórmula global de `calculateAndSaveScore`, que suma todas las respuestas y usa todas las preguntas en el denominador (`evaluations.helper.js`, líneas 253-269). No es una ponderación de negocio configurada: cada **pregunta** tiene el mismo peso y los grupos con más preguntas pesan más por construcción.

### 6.3 Qué requeriría aplicar ponderaciones

No se deben inferir pesos desde `description`. Para aplicar ponderaciones de negocio de forma auditable se necesitaría, como diseño futuro:

- campo numérico explícito de peso en grupo global y grupo snapshot;
- validación de signo, precisión y suma esperada;
- definición de conducta para grupos sin preguntas y preguntas sin grupo;
- fórmula versionada y cálculo desde valores sin redondear por grupo;
- exposición del peso en contratos API/UI;
- estrategia explícita para evaluaciones históricas, sin reescribir 2025 por defecto.

Nada de lo anterior está implementado en el modelo o en las fórmulas auditadas (`evaluations.prisma`, líneas 64-79 y 149-193; `evaluations.helper.js`, líneas 235-297).

## 7. Clasificación, PDF, reportes y correo

### 7.1 PDF

El servicio de PDF consulta el detalle mediante `getEvaluationById`, serializa el DTO y lo inyecta en la página React de impresión (`evaluationPdf.service.js`, líneas 24-30 y 77-83). Para una evaluación completada con score guardado, `meta.score` es el persistido por la regla de `formatEvaluationForFrontend` (`evaluations.helper.js`, líneas 75-88).

El PDF muestra `meta.score.toFixed(1)` (`EvaluationPrintPage.tsx`, líneas 467-479), pero **no usa `ratingLabel` persistido**. Vuelve a clasificar el score de esta forma:

1. `Math.round(score)` a entero;
2. busca la primera escala del DTO cuyo rango inclusivo contenga ese entero;
3. muestra primero `scale.label`, con `description` como fallback.

Evidencia: `EvaluationPrintPage.tsx#getScoreScaleDescription/getScaleResult`, líneas 70-76 y 118-132. Las escalas del DTO llegan ordenadas por `sortOrder` porque `getEvaluationById` así las consulta (`evaluations.service.js`, líneas 369-379).

Esto difiere de Node, que clasifica el decimal sin `Math.round`, usa un `findFirst` sin orden y persiste `description` (`evaluations.helper.js`, líneas 272-295). En límites o solapamientos, el texto del PDF puede no representar el `ratingLabel` almacenado.

### 7.2 Reportes web

El dashboard y el listado detallado actuales son reportes de seguimiento. Sus consultas no seleccionan `score` ni `ratingLabel`; agregan estados, confirmaciones y porcentajes de evaluaciones completadas (`reports.service.js#getDashboardProgress`, líneas 41-83 y 86-201; `getEvaluationsList`, líneas 357-420).

Por tanto, hoy no existe en reportes una fórmula paralela de resultado, promedio por grupo o promedio total. Los porcentajes visibles allí son avance operativo, por ejemplo `completed / total × 100` por proyecto (`reports.service.js`, líneas 178-188), no puntuación de desempeño.

### 7.3 Correo de finalización

El correo sí muestra `{{score}}` (`api/src/modules/notifications/templates/evaluation-completed.mjml`, líneas 65-84). Sin embargo, `finalizeEvaluation` carga el objeto `evaluation` **antes** de recalcular, luego llama a `calculateAndSaveScore`, pero encola `score: evaluation.score` desde el objeto previo (`evaluations.service.js`, líneas 529-549, 633-660).

Por ello, el cuerpo del correo puede recibir `null` o el puntaje parcial anterior, mientras el PDF adjunto vuelve a consultar el detalle y pretende usar el valor persistido (`evaluation.processor.js`, líneas 7-26; `evaluationPdf.service.js`, líneas 24-30). Este es un riesgo concreto de discrepancia entre correo y PDF. Además, el job se agrega dentro de la transacción antes de que esta retorne (`evaluations.service.js`, líneas 526-527 y 646-668), por lo que debe validarse en pruebas integradas si el worker puede leer antes del commit.

## 8. Matriz React vs Node.js

| Concepto | React | Node.js | ¿Coinciden? |
|---|---|---|---|
| Puntaje de pregunta | Botones fijos 1–5; guarda el número (`QuestionSectionCard.tsx`, líneas 24-31 y 142-165). | Convierte con `Number()` y persiste `Int?`, sin validación explícita 1–5 (`evaluations.service.js`, líneas 436-452; `evaluations.prisma`, línea 336). | **Parcial.** Coinciden en el flujo UI normal; Node admite entradas no protegidas por la misma regla. |
| Puntaje por grupo | No calcula; aplana todas las secciones (`useEvaluationForm.ts`, líneas 118-120). | No calcula; usa todas las preguntas del periodo (`evaluations.helper.js`, líneas 236-269). | **Sí:** ambos carecen de subtotal de grupo. |
| Puntaje total | `Σ respuestas / (preguntas × 5) × 100` (`useEvaluationForm.ts`, líneas 155-167). | Misma fórmula (`evaluations.helper.js`, líneas 253-269). | **Sí** en el flujo normal. |
| Redondeo | `parseFloat(toFixed(1))` y presentación `toFixed(1)` (`useEvaluationForm.ts`, líneas 164-167; `StickyHeader.tsx`, líneas 80-84). | `parseFloat(toFixed(1))`, almacenado en `Float?` (`evaluations.helper.js`, líneas 264-270; `evaluations.prisma`, línea 318). | **Sí** para score. No para clasificación del PDF, que primero usa `Math.round`. |
| Preguntas sin responder | Permanecen en el denominador; valores ausentes no suman (`useEvaluationForm.ts`, líneas 155-167). | Permanecen en el denominador; faltantes no están en respuestas y aportan cero (`evaluations.helper.js`, líneas 253-269). | **Sí** para borrador parcial. Al finalizar Node exige registro para cada pregunta (`evaluations.service.js`, líneas 609-631). |
| Escalas | Pregunta fija 1–5; clasificación global recibida desde API; color del score con umbrales fijos (`QuestionSectionCard.tsx`, líneas 24-31; `StickyHeader.tsx`, líneas 16-20). | Máximo fijo 5; clasificación con snapshot configurable 0–100 (`evaluations.helper.js`, líneas 259-280). | **Parcial.** El máximo coincide; señales visuales y clasificación no comparten una única regla. |
| Clasificación final | Formulario no consume `ratingLabel`; PDF redondea score a entero, busca rango y muestra `label` (`EvaluationPrintPage.tsx`, líneas 70-76 y 118-132). | Compara score con un decimal, toma `findFirst` sin orden y persiste `description` (`evaluations.helper.js`, líneas 272-295). | **No.** Algoritmo, orden y texto elegido difieren. |
| Avance de formulario | Preguntas visibles con valor numérico `> 0` + fortaleza + debilidad con compromiso, dividido por preguntas + 2 (`useEvaluationForm.ts`, líneas 118-153). | Usa cantidad de filas de respuesta + las dos tareas; `COMPLETED` fuerza 100 % (`evaluations.helper.js`, líneas 21-51). | **Parcial.** Coincide con datos válidos; filas cero/inválidas/ajenas pueden divergir. La UI ignora `meta.progressPercentage` y recalcula incluso completadas. |
| Fuente al consultar histórico completado | El encabezado usa cálculo propio desde respuestas (`EvaluationsForm.tsx`, líneas 128-132). | El DTO entrega `score` persistido cuando es numérico (`evaluations.helper.js`, líneas 75-88). | **No en diseño**, aunque hoy normalmente arrojan el mismo número. |

## 9. Ejemplos numéricos

### 9.1 Grupo de dos preguntas: ejemplo solicitado

Respuestas: `5` y `4`.

- obtenido: `5 + 4 = 9`;
- máximo: `2 × 5 = 10`;
- porcentaje simple del grupo: `9 / 10 × 100 = 90,0 %`.

El resultado de 90,0 % es el que produciría la fórmula propuesta de grupo, pero el sistema actual no lo calcula ni lo muestra como subtotal. Las dos respuestas sí aportan 9 puntos al cálculo total (`evaluations.helper.js`, líneas 253-269).

### 9.2 Pregunta sin responder durante borrador

Grupo de dos preguntas; solo la primera tiene respuesta `5`:

- obtenido: `5`;
- máximo total: `2 × 5 = 10`;
- score parcial actual: `5 / 10 × 100 = 50,0 %`.

No resulta 100 % sobre lo respondido. La pregunta faltante permanece en el denominador, tanto en React como en Node (`useEvaluationForm.ts`, líneas 155-167; `evaluations.helper.js`, líneas 253-269).

### 9.3 Comparación A, B y C con grupos de distinto tamaño

Se usa el mismo conjunto de respuestas:

| Grupo | Nombre ilustrativo | Preguntas | Respuestas | Obtenido / máximo | Resultado de grupo |
|---|---|---:|---|---:|---:|
| G1 | CALIDAD (10%) | 2 | 5, 4 | 9 / 10 | 90,0 % |
| G2 | PLANEACIÓN (15%) | 4 | 2, 2, 2, 2 | 8 / 20 | 40,0 % |

#### Método A — suma de todas las respuestas

`A = (9 + 8) / ((2 + 4) × 5) × 100 = 17 / 30 × 100 = 56,7 %`

Este es el método implementado en Node y React (`evaluations.helper.js`, líneas 253-269; `useEvaluationForm.ts`, líneas 155-167). G1 representa 2/6 del máximo y G2 representa 4/6: el grupo con más preguntas influye más.

#### Método B — promedio simple de porcentajes por grupo

`B = (90,0 + 40,0) / 2 = 65,0 %`

Cada grupo pesa 50 %, sin importar que uno tenga el doble de preguntas. **No está implementado.**

#### Método C — promedio ponderado por grupo

Si los textos 10 % y 15 % se interpretaran hipotéticamente como pesos y se normalizaran sobre los grupos del ejemplo:

`C = (90,0 × 10 + 40,0 × 15) / (10 + 15) = 60,0 %`

G1 pesa 40 % y G2 pesa 60 % del total del ejemplo. **No está implementado y el sistema no interpreta esos textos como pesos.** En un formulario real debería definirse si los pesos deben sumar 100 % y qué hacer si no lo hacen.

Los tres resultados —56,7 %, 65,0 % y 60,0 %— son correctos para reglas distintas. La diferencia no es de redondeo: responde a qué unidad se considera de igual importancia (pregunta, grupo o peso funcional).

### 9.4 Diferencia potencial en clasificación

Supónganse rangos enteros `0–59` y `60–79`, y un score `59,6`:

- Node compara `59,6` directamente: no pertenece a `0–59` ni a `60–79`; `ratingLabel` queda `null` (`evaluations.helper.js`, líneas 275-286).
- el PDF ejecuta `Math.round(59,6) = 60` y selecciona el segundo rango (`EvaluationPrintPage.tsx`, líneas 118-126).

Este ejemplo demuestra una divergencia posible con el diseño actual; no afirma que esos sean los rangos reales del periodo 2025, porque no se consultaron datos productivos.

## 10. Posibles inconsistencias y defectos

### Alta prioridad para P2

1. **Fuente distinta en histórico:** Node entrega score persistido para `COMPLETED`, pero el encabezado del formulario lo recalcula desde respuestas. Evidencia: `evaluations.helper.js`, líneas 75-88; `EvaluationsForm.tsx`, líneas 128-132.
2. **Clasificación no unificada:** backend clasifica decimal/descripción/sin orden; PDF clasifica entero/label/orden de DTO. Evidencia: `evaluations.helper.js`, líneas 272-295; `EvaluationPrintPage.tsx`, líneas 70-76 y 118-132.
3. **Correo con score anterior:** el job recibe `evaluation.score` del objeto cargado antes del recálculo. Evidencia: `evaluations.service.js`, líneas 529-549 y 633-660; la plantilla lo imprime en `evaluation-completed.mjml`, líneas 73-84.
4. **Regla 1–5 y pertenencia no protegidas en backend:** el máximo supone 5, pero guardar/finalizar solo aplica `Number()` y no comprueba que la pregunta pertenezca al periodo. Una entrada ajena o mayor que 5 puede producir score fuera de 0–100. Evidencia: `evaluations.service.js`, líneas 436-452 y 563-580; `evaluations.helper.js`, líneas 236-269.

### Prioridad media

5. **No existe score por grupo**, aunque los nombres sugieren porcentajes y podrían llevar al usuario a esperar ponderación. Evidencia: `evaluations.prisma`, líneas 149-193; `useEvaluationForm.ts`, líneas 118-120 y 155-169.
6. **Peso implícito por cantidad de preguntas:** no está explicado en la interfaz. Evidencia matemática derivada de `evaluations.helper.js`, líneas 253-269.
7. **El cliente no reconcilia con el servidor después de guardar/finalizar:** recibe solo éxito. Evidencia: `useEvaluationForm.ts`, líneas 172-188 y 228-253; `evaluations.controller.js`, líneas 138-143.
8. **Color del puntaje con umbrales independientes:** puede no coincidir con la escala snapshot. Evidencia: `StickyHeader.tsx`, líneas 16-20; escalas del periodo en `evaluations.helper.js`, líneas 119-128.
9. **Rangos sin garantías de continuidad/no solapamiento:** el modelo no impone esas reglas y la consulta de clasificación no ordena. Evidencia: `evaluations.prisma`, líneas 242-259; `evaluations.helper.js`, líneas 275-282.

## 11. Impacto sobre evaluaciones históricas de 2025

### Situación actual

- `score` y `ratingLabel` están persistidos en `Evaluation` (`evaluations.prisma`, líneas 317-319).
- las respuestas originales siguen disponibles en `EvaluationAnswer` (`evaluations.prisma`, líneas 327-343).
- una evaluación completada no puede editarse o finalizarse nuevamente por los servicios normales (`evaluations.service.js`, líneas 425-431 y 551-557).
- el API de detalle conserva el `score` persistido para una evaluación completada (`evaluations.helper.js`, líneas 75-88).
- el PDF usa ese `meta.score` persistido (`EvaluationPrintPage.tsx`, líneas 467-479).
- los reportes actuales no muestran ni recalculan resultados de desempeño (`reports.service.js`, líneas 41-83 y 357-420).

### Riesgos si se modifica la fórmula

1. **Cambiar solo `calculateAndSaveScore`:** afectaría futuros guardados/finalizaciones y borradores editables. No cambiaría por sí solo los `score` de evaluaciones 2025 ya completadas, porque no hay recálculo en la consulta ni endpoint de recálculo y los flujos de escritura rechazan `COMPLETED`.
2. **Cambiar el cálculo de React:** cambiaría inmediatamente el puntaje del encabezado al abrir también evaluaciones históricas completadas, porque usa `stats.score` recalculado. El PDF podría seguir mostrando el persistido y aparecer una diferencia.
3. **Cambiar `formatEvaluationForFrontend`:** si deja de priorizar el valor persistido, podría alterar formulario y PDF históricos en tiempo de consulta aunque la base no se actualice.
4. **Cambiar la clasificación del PDF:** puede alterar la etiqueta mostrada en PDFs regenerados del histórico, aunque `score` y `ratingLabel` almacenados permanezcan intactos, porque el PDF reclasifica y no usa `ratingLabel`.
5. **Agregar pesos sin snapshot histórico:** interpretar retroactivamente nombres como “(10%)” cambiaría resultados y dependería de texto no estructurado. Los modelos actuales no conservan un peso verificable (`evaluations.prisma`, líneas 149-169).

La protección recomendada es que 2025 siga leyendo el `score` persistido y una clasificación histórica estable, sin ejecutar actualizaciones ni recálculos. Cualquier migración histórica debe ser una decisión separada, con comparación previa, respaldo, trazabilidad y aprobación funcional.

## 12. Alternativas de corrección

### Alternativa A — preguntas con igual peso

Mantener la regla actual:

`total = Σ respuestas / (total preguntas × 5) × 100`

Y, si se requiere mostrar grupos:

`grupo = Σ respuestas del grupo / (preguntas del grupo × 5) × 100`

El total debe calcularse desde respuestas crudas o, equivalentemente, ponderando cada grupo por cantidad de preguntas, nunca promediando subtotales ya redondeados. Ventajas: continuidad con el score persistido actual, explicación simple y ausencia de cambios de modelo. Implicación: grupos con más preguntas pesan más.

### Alternativa B — grupos con igual peso

`total = Σ porcentaje_grupo_sin_redondear / cantidad_grupos`

Ventaja: cada competencia/grupo influye igual. Implicación: cada pregunta de un grupo pequeño pesa más que cada pregunta de un grupo grande. Requiere definir grupos vacíos/sin grupo y supone un cambio funcional frente a 2025.

### Alternativa C — grupos con peso configurable

`total = Σ(porcentaje_grupo_sin_redondear × peso_grupo) / Σ pesos`

Ventaja: representa una decisión explícita de negocio. Implicación: requiere campos globales y snapshot, validaciones, contratos, UI, pruebas y versionamiento. Los porcentajes no deben extraerse del nombre.

### Alternativa D — usar solo preguntas respondidas en borrador

Cambiar únicamente el score provisional a:

`parcial = Σ respuestas / (preguntas respondidas × 5) × 100`

Esto puede ser más intuitivo durante el diligenciamiento, pero ya no expresa progreso hacia el máximo total y produciría saltos al responder valores bajos. Si se adopta, debe etiquetarse claramente como promedio parcial y mantenerse separado del avance. No corresponde a la fórmula vigente.

## 13. Recomendación técnica fundamentada

Mientras Gestión Humana no confirme pesos formales por competencia, se recomienda aprobar como base la **Alternativa A: cada pregunta con igual peso**, por estas razones:

- es la regla ya aplicada por Node y React (`evaluations.helper.js`, líneas 253-269; `useEvaluationForm.ts`, líneas 155-167);
- es compatible con los `score` históricos persistidos;
- no existe campo de peso que permita reproducir de forma fiable otra fórmula (`evaluations.prisma`, líneas 64-79 y 149-169);
- evita interpretar semántica desde nombres libres;
- permite añadir porcentajes informativos por grupo sin alterar el total.

La recomendación de implementación para una fase posterior, sujeta a aprobación, es:

1. definir una única función/regla canónica en backend para score total, score por grupo y clasificación;
2. validar en backend que cada respuesta sea un entero 1–5 y que su pregunta pertenezca al periodo de la evaluación;
3. devolver al guardar/finalizar el resultado canónico y reconciliar el estado React;
4. usar el `score` persistido como única fuente para evaluaciones completadas, tanto en formulario como en PDF;
5. mostrar el score durante borrador como **provisional** y mantenerlo visualmente separado del avance;
6. calcular subtotales de grupo desde respuestas sin redondear, redondeando solo el valor presentado;
7. clasificar una sola vez con el score redondeado acordado, rangos completos/no solapados y criterio determinista;
8. decidir si el texto oficial es `label` o `description` y reutilizar exactamente ese resultado en formulario, PDF y correo;
9. corregir el payload del correo para usar el resultado recién calculado y validar el orden commit/job;
10. congelar el comportamiento histórico 2025; si en el futuro se adopta ponderación, versionar la fórmula y guardar pesos en snapshots.

Esta recomendación no constituye autorización para implementar cambios.

## 14. Decisiones funcionales pendientes de aprobación

1. ¿Se confirma que cada pregunta tiene el mismo peso y que los porcentajes de los nombres son solo texto informativo?
2. Si se desean pesos, ¿cuáles son los pesos oficiales, deben sumar 100 % y desde qué periodo aplican?
3. ¿Debe mostrarse un porcentaje por grupo? ¿Se redondea a un decimal únicamente para presentación?
4. ¿El score parcial conserva todas las preguntas en el denominador o usa solo respondidas?
5. ¿La escala de pregunta continúa fija en 1–5 o debe parametrizarse?
6. ¿Qué precisión oficial debe persistirse: un decimal u otra?
7. ¿La clasificación se determina con el decimal exacto, con el score redondeado a un decimal o con un entero?
8. ¿Cómo deben definirse los límites para cubrir 0–100 sin huecos ni solapamientos?
9. ¿El resultado textual oficial es `label` o `description`? ¿Debe persistirse también el ID de la escala aplicada?
10. ¿Las evaluaciones `COMPLETED` deben mostrar siempre el valor persistido, incluso si una fórmula futura produciría otro resultado?
11. ¿La nueva regla aplicará solo a periodos nuevos? Recomendación: **sí**, salvo procedimiento histórico aprobado por separado.
12. ¿Cómo se presentará una evaluación histórica con `score` o `ratingLabel` nulo/anómalo, sin recalcularla silenciosamente?

## 15. Criterios de aceptación propuestos para la futura implementación de P2

1. Para un mismo conjunto de respuestas, formulario, API, PDF y correo presentan el mismo score y clasificación.
2. La escala 1–5 se valida en backend y frontend.
3. Avance, score de grupo y score total tienen etiquetas y fórmulas independientes.
4. Grupos de distinto tamaño producen el resultado aprobado y cubierto por pruebas numéricas A/B/C.
5. El redondeo ocurre en el punto definido y no se promedian resultados intermedios ya redondeados.
6. Los rangos de clasificación cubren 0–100, no se solapan y tienen precedencia determinista.
7. Las evaluaciones completadas de 2025 no se actualizan y regenerar su PDF no cambia silenciosamente su resultado.
8. Se incluyen pruebas para: cero preguntas defensivo, borrador parcial, todas respondidas, extremos 1 y 5, decimal limítrofe, rango ausente/solapado y respuestas ajenas al periodo.

## 16. Conclusión para aprobación

- **Fórmula Node.js actual:** `redondear_1_decimal(Σ respuestas / (total preguntas del periodo × 5) × 100)`.
- **Fórmula React actual:** la misma, calculada sobre el estado local del formulario.
- **Fórmula por grupo:** no existe; solo hay agrupación estructural/visual.
- **Método total implementado:** método A, equivalente a ponderar grupos por cantidad de preguntas.
- **Porcentajes en nombres de grupos:** texto literal, sin campo de peso ni interpretación matemática.
- **Principales inconsistencias:** React recalcula completadas; clasificación backend/PDF distinta; correo puede usar score anterior; backend no valida 1–5/pertenencia.
- **Recomendación:** conservar inicialmente el método A, añadir grupo solo como subtotal informativo, centralizar cálculo/clasificación en backend y no modificar 2025.
- **Riesgo histórico principal:** cambiar React o la reclasificación del PDF puede cambiar lo visible sin alterar la base, provocando diferencias entre formulario, PDF y resultado persistido.

## 17. Implementación P2.1 — Persistencia y edición del peso (2026-10-20)

### 17.1 Estado y límites

**Estado:** **Implementado técnicamente; migración y prueba integrada pendientes de autorización.**

P2.1 incorpora persistencia explícita y edición del peso de grupos sin modificar `calculateAndSaveScore`, `useEvaluationForm.ts`, scores, respuestas, evaluaciones completadas, PDF, correos o reportes. No se ejecutaron migraciones, publicaciones ni operaciones sobre datos reales.

La suma de 100 % se presenta como ayuda administrativa, pero no bloquea todavía la publicación. La fórmula ponderada continúa reservada para P2.2.

### 17.2 Campos nuevos

Se añadieron a Prisma:

| Modelo | Campo | Tipo | Columna | Compatibilidad |
|---|---|---|---|---|
| `QuestionGroup` | `weightPercentage` | `Decimal? @db.Decimal(5, 2)` | `weight_percentage` | `NULL` permitido para grupos existentes. |
| `PeriodGroup` | `weightPercentage` | `Decimal? @db.Decimal(5, 2)` | `weight_percentage` | `NULL` permitido para snapshots históricos. |

Fuentes: `api/prisma/modules/evaluations.prisma#QuestionGroup`, `#PeriodGroup` y el esquema ensamblado `api/prisma/schema.prisma`.

La migración preparada es `api/prisma/migrations/20261020100000_add_group_weight_percentage/migration.sql`. Agrega ambas columnas nullable y restricciones `CHECK` para aceptar únicamente `NULL` o valores entre 0 y 100. No contiene `UPDATE`, backfill ni modificación de snapshots existentes.

### 17.3 Contratos API actualizados

Base: `/api/employee-evaluations/configurations/groups`.

| Operación | Cambio |
|---|---|
| `GET /` y `GET /:id` | Cada grupo incluye `weightPercentage: number | null`. Prisma Decimal se normaliza a número para el contrato JSON. |
| `POST /` | Acepta `{ description, weightPercentage }`; el peso puede ser número, cadena decimal o `null`. |
| `PUT /:id` | Acepta `{ description, weightPercentage }`; `undefined` conserva el peso actual y `null` lo deja explícitamente sin configurar. |
| `GET /periods/:id/preview` | Cada grupo global/snapshot incluye `weightPercentage`; “Preguntas sin grupo” usa `null`. |

Fuentes: `api/src/modules/employee-evaluations/configurations/groups/groups.controller.js`, `groups.service.js`, `groupWeight.utils.js` y `periods/snapshots/periodSnapshots.service.js`.

### 17.4 Validaciones implementadas

- rango inclusivo: 0–100;
- máximo dos decimales;
- `NULL` y cadena vacía se interpretan como peso no configurado;
- `undefined` en actualización conserva el valor actual;
- la base queda protegida adicionalmente por `CHECK` una vez se ejecute la migración;
- no se extrae ni transforma ningún porcentaje contenido en `description`.

La descripción y el peso se editan en campos separados en `sisproing-app/src/modules/employee-evaluations/configurations/groups/GroupsPage.tsx`. La pantalla muestra el peso de cada fila, el total acumulado y la cantidad de grupos sin peso. La suma distinta de 100 % es una advertencia, no un bloqueo.

### 17.5 Conservación en snapshots

`periodSnapshots.service.js#createSnapshotsForPeriod` usa `buildPeriodGroupSnapshotData` para copiar al nuevo `PeriodGroup`:

- `description` sin alteraciones;
- `weightPercentage` exacto o `NULL`;
- `periodId` y auditoría existentes.

Esta lógica solo corre al publicar un periodo nuevo. La migración añade una columna nullable y no actualiza filas de `period_groups`, por lo que los snapshots publicados previamente permanecen en `NULL` y siguen siendo consultables.

### 17.6 Archivos modificados por P2.1

#### API

- `prisma/modules/evaluations.prisma`;
- `prisma/schema.prisma` (generado desde los módulos);
- `prisma/migrations/20261020100000_add_group_weight_percentage/migration.sql`;
- `src/modules/employee-evaluations/configurations/groups/groupWeight.utils.js`;
- `src/modules/employee-evaluations/configurations/groups/groups.controller.js`;
- `src/modules/employee-evaluations/configurations/groups/groups.service.js`;
- `src/modules/employee-evaluations/periods/snapshots/periodGroupSnapshot.utils.js`;
- `src/modules/employee-evaluations/periods/snapshots/periodSnapshots.service.js`;
- `tests/employee-evaluations/group-weight.test.js`.

#### React

- `src/modules/employee-evaluations/configurations/groups/GroupsPage.tsx`;
- `src/modules/employee-evaluations/configurations/groups/groups.service.ts`;
- `src/modules/employee-evaluations/periods/periods.service.ts` (contrato de vista previa).

#### Documentación

- `docs/mejoras/MEJ-002-calculo-puntuaciones.md`;
- `docs/mejoras/MEJ-002-plan-ponderacion-grupos.md`.

### 17.7 Validación ejecutada

| Validación | Resultado |
|---|---|
| `npx prisma validate --schema prisma/schema.prisma` | **Aprobada**. |
| Pruebas Node dirigidas de P2.1 y selección de periodo | **12/12 aprobadas**. |
| API `npm run type-check` | **Aprobada**. |
| API `npm run build` | **Aprobada**. |
| React `npm run typecheck` | **Aprobada**. |
| ESLint dirigido a archivos React modificados | **Aprobado**. |
| React `npm run build` | **Aprobada**; solo avisos informativos de datos de navegadores desactualizados. |

Las pruebas de `group-weight.test.js` cubren límites, decimales, rechazo de valores inválidos, suma de los ocho pesos de referencia y copia del peso/`NULL` al objeto del snapshot.

### 17.8 Checklist de verificación controlada

- [ ] Ejecutar la migración únicamente en un ambiente autorizado y confirmar que los registros existentes quedan con `weight_percentage = NULL`.
- [ ] Crear un grupo con peso y comprobar request, respuesta y persistencia.
- [ ] Editar el peso de un grupo sin alterar su descripción.
- [ ] Consultar el grupo y verificar que el API devuelve `number | null`.
- [ ] Ingresar manualmente los ocho pesos de referencia y comprobar total visible de 100 %.
- [ ] Confirmar lectura y edición controlada de grupos antiguos con peso `NULL`.
- [ ] Publicar un periodo exclusivamente en ambiente aislado y verificar que `PeriodGroup.weightPercentage` copia el valor global vigente.
- [ ] Confirmar que snapshots de periodos anteriores permanecen en `NULL` y sin actualizaciones.
- [ ] Confirmar que ningún `score` o `ratingLabel` cambió durante las verificaciones.

No se publicó ningún periodo durante esta implementación.

### 17.9 Riesgos y despliegue pendiente

1. **Orden de despliegue:** ejecutar la migración antes de iniciar API/React que consulten la nueva columna.
2. **Tablas efectivas:** el SQL usa los nombres mapeados actuales `evaluation.question_groups` y `evaluation.period_groups`; deben verificarse en el ambiente objetivo antes de autorizar despliegue, debido a la divergencia histórica de migraciones documentada en el proyecto.
3. **Pesos incompletos:** `NULL` preserva compatibilidad, pero un periodo nuevo todavía puede publicarse con suma distinta de 100 %; la política bloqueante se decidirá antes de P2.2.
4. **Precisión:** API y UI admiten dos decimales; no se implementó redondeo ni fórmula ponderada de resultados.
5. **Histórico:** regenerar el cliente Prisma no altera datos; la migración preparada solo añade columnas/constraints nullable. No existe backfill automático desde nombres.

**Migración pendiente:** `20261020100000_add_group_weight_percentage` está creada pero no ejecutada en ningún ambiente por esta tarea.

