# MEJ-006 — Recálculo controlado de puntuaciones históricas HF-011 (aplazado)

**Proyecto:** PROING — Evaluación de Desempeño
**Fecha de acuerdo:** 2026-10-20
**Estado:** **REACTIVADO — AUDITORÍA Y SIMULACIÓN DE SOLO LECTURA (2026-10-21)**. El responsable solicitó retomar la corrección histórica. **No autorizado todavía ningún UPDATE, migración o cambio de snapshots, respuestas, acuses o PDFs.**
**Dependencia:** P2 finalizado técnicamente; antes de desplegar cualquier rectificación histórica, verificar datos y respaldos autorizados.

## 1. Motivo y decisión

Las evaluaciones completadas del periodo histórico 2025 guardaron sus `score` mediante la fórmula LEGACY (suma de respuestas / total de preguntas × 5, porcentaje a un decimal). En P2 se aprobó y probó la ponderación por los ocho grupos según pesos explícitos: **Orientación a resultados 20 %, Calidad 10 %, Relaciones interpersonales 10 %, Trabajo en equipo 5 %, Planeación 15 %, Innovación y comunicaciones 5 %, Actitud 15 %, SST 20 %**.

El responsable decidió **posponer inicialmente** la eventual actualización masiva de scores históricos para resolver primero «Mis Evaluaciones»; el 2026-10-21 **retomó MEJ-006** para preparar el diagnóstico y un dry-run sin escrituras. La corrección real sigue condicionada a revisar y aprobar el reporte.

**Decisión técnica de partida:** no actualizar `evaluation.period_groups.weight_percentage` de snapshots ya publicados como paso de corrección, porque actualmente `NULL` determina `LEGACY` y rellenar pesos cambia su interpretación durante las operaciones. No modificar catálogos ni snapshot global de 2025 para conseguir el recálculo. Usar un mapa **verificado** `periodGroupId -> weightPercentage` exclusivamente para la simulación/corrección offline.

La corrección no debe confundirse con la regla aplicada por el Excel original, que promediaba los cuatro subgrupos de Cultura Organizacional y les daba 10 % efectivos cada uno. **La regla aprobada para el software** usa sus pesos explícitos 5/15/5/15.

## 2. Caso de control documentado

**Evaluación histórica PDF «Evaluacion_8120.pdf»** (identificador impreso 8120; corroborar con PK real antes de operar):
- Periodo mostrado: «Evaluación de Desempeño - Año 2025».
- 33 respuestas, sumatoria 102 puntos sobre máximo 165.
- Puntuación LEGACY en PDF: **61,8 %** (102/165 × 100).
- Puntaje ponderado según los ocho pesos: **61,4 %**.
- Diferencia: **−0,4 puntos porcentuales**.
- Clasificación textual actual: desempeño «bueno»; verificar escala snapshot real antes de decidir `ratingLabel`.
- Documento marcado como **recibido digitalmente**. Rectificar un score puede cambiar el PDF que se descarga después respecto al documento efectivamente recibido/confirmado; **preservar copia/evidencia y procedencia del resultado anterior**.

Este caso sirve como prueba de regresión, no como prueba de que todas las evaluaciones tendrán esa diferencia.

## 3. Procedimiento propuesto para cuando se autorice

### Etapa A — Auditoría de solo lectura y alcance
1. Identificar periodo(s) 2025 concretos por PK; verificar snapshots `PeriodGroup`, `PeriodQuestion`, `PeriodScoreScale` y la relación real entre respuestas, preguntas y evaluaciones.
2. Construir/validar mapa de los **IDs exactos de grupo snapshot** a pesos aprobados; ninguna asociación por posición de despliegue ni parseo de texto sin verificación humana.
3. Identificar únicamente evaluaciones `COMPLETED` del periodo autorizado. Reportar cuentas por estado, `score` nulo, respuestas inválidas/incompletas, preguntas sin grupo o datos huérfanos; **no corregir silenciosamente anomalías**.
4. Corroborar el ID 8120 del PDF con la PK y sus respuestas reales.

### Etapa B — Simulación `dry-run` sin escritura
1. Reutilizar exactamente la fórmula ponderada validada de P2, suministrando los pesos aprobados de los snapshots históricos **en memoria**.
2. Calcular subtotal por grupo sin redondeo intermedio, total con un decimal final.
3. Generar reporte por `evaluationId`: `periodId`, `oldScore`, `newScore`, diferencia, `oldRatingLabel`, clasificación propuesta con `PeriodScoreScale`, bandera de acuse, validación de completitud, incidencias.
4. Informar cantidad de registros sin diferencia, cambiados, casos en límites de clasificación y casos excluidos. Validar explícitamente 8120 = **61,4 %** bajo el mapa aprobado.

### Etapa C — Respaldo y aprobación
1. Tomar backup/restauración ensayada según procedimientos de operación, exportar resultados originales y registrar checksum/fecha/versión de fórmula/mapa de pesos.
2. Diseñar trazabilidad independiente de `score` y `ratingLabel` anteriores, quién/cuándo autorizó y nueva clasificación; no sustituir ni borrar el documento original recibido.
3. Acordar antes de escribir cómo tratar PDF histórico/acuse ya efectuado: distinguir versión original recibida de posible documento rectificado; no representar rectificación como aceptación retroactiva por el empleado.
4. Aprobar explícitamente el alcance de los IDs y diferencias producidas por `dry-run`.

### Etapa D — Corrección transaccional controlada (no autorizada todavía)
1. Validar que el puntaje previo en base coincide con el valor leído en la simulación (evitar concurrencia).
2. Actualizar **solo** `score` y, cuando corresponda de forma inequívoca, `ratingLabel` de las evaluaciones `COMPLETED` autorizadas; **nunca** respuestas, acuses, fechas, grupos snapshot, preguntas o pesos.
3. Ejecutar lotes transaccionales con bitácora, errores explícitos, idempotencia y posibilidad de reversión operativa.
4. Verificación posterior contra export/backup, recálculo independiente y pruebas de PDF/histórico; registrar evidencia por evaluación.

## 4. Restricciones

- **No ejecutar ahora** SQL UPDATE, migraciones, backfill, creación de tablas o actualizaciones de scores.
- No convertir snapshots históricos de LEGACY a WEIGHTED por rellenar `weight_percentage`.
- No refactorizar la fórmula P2 ni reabrir P1/P3/P4.
- Si se descubre que el sistema recalcula un score `COMPLETED` al consultar, tratarlo como defecto separado y **detener cualquier migración** hasta resolverlo.
- Considerar el estado conocido de migraciones Prisma (`P3005`); no ejecutar `prisma migrate deploy` como parte del procedimiento sin reconciliar su historial.

## 5. Próxima acción futura

Cuando el responsable solicite reabrir este punto: **preparar el `dry-run` de solo lectura primero**, contrastar la matriz contra 8120 y pedir aprobación explícita de resultados antes de diseñar o ejecutar el script correctivo.

**Estado al reabrir (2026-10-21):** autorizado a preparar la auditoría de solo lectura y el dry-run, **sin modificar datos**. Un reporte final debe preceder toda solicitud explícita de ejecución SQL correctiva.


## 6. Reactivación y puerta de control para el primer entregable (2026-10-21)

**Alcance concreto del próximo bloque:** analizar exclusivamente la campaña identificada en la base como «Evaluación de Desempeño - Año 2025», fijando su `periodId` real antes de calcular. No incluir automáticamente otras campañas, aunque también sean LEGACY.

**Entregable de diagnóstico y simulación:** script explícitamente `READ-ONLY`/`dry-run`, invocable de forma manual contra conexión de **solo lectura** y sin tareas programadas; reporte CSV/JSON con `evaluationId`, `periodId`, `oldScore`, `newScore`, `delta`, `oldRatingLabel`, `newRatingLabel` propuesto, `notificationReceived`, integridad/completitud de preguntas, estado y motivos de exclusión. Resumen agregado de cambios, cruces de clasificación y anomalías. No incluir nombres/documentos personales innecesarios en reportes compartidos.

**Verificaciones previas al cálculo:**
1. Verificar contra la base los IDs reales del periodo y de los **ocho grupos de snapshot**, y presentar mapa de pesos revisable. El texto de descripción sirve solo de referencia humana para emparejar; la ejecución efectiva debe usar IDs explícitos validados.
2. Inspeccionar la estructura Prisma/SQL vigente y verificar respuestas `EvaluationAnswer` vs. `PeriodQuestion` del mismo periodo; descartar errores de grupo, duplicados, valores fuera de 1–5, respuesta ausente o inexistencia de las 8 agrupaciones; no inferir, normalizar ni corregir registros silenciosamente.
3. Confirmar de manera independiente que el viejo puntaje persistido es coherente con el cálculo LEGACY, o reportar discrepancia de origen antes de comparar o proponer reemplazo.
4. Usar la **misma implementación matemática probada en P2**, con pesos de grupos **en memoria** y redondeo solo final a un decimal, sin convertir los snapshots guardados a WEIGHTED.
5. Confirmar que el `UUID: 8120` del PDF es el `evaluationId` real, con las respuestas de la base; caso testigo **61,8 % → 61,4 %**. Si no coincide, explicar y detener antes de preparar el script de escritura.
6. Consultar la escala snapshot `PeriodScoreScale` del periodo y reportar rangos solapados/huecos, porque `ratingLabel` puede no corresponder exactamente a todo decimal con un decimal.

**Separación de autorizaciones:** reabrir el plan **no autoriza** modificaciones en base de datos, creación/edición de PDF, actualización de calificaciones, alteraciones de acuse, migraciones o generación de correos. Para pasar de `dry-run` a `UPDATE` será necesaria nueva aprobación expresa del responsable, después de respaldo, trazabilidad y política de PDF rectificado.

## 7. MEJ-006.1 — Implementación del dry-run de solo lectura (2026-10-21)

### 7.1 Estado de la verificación real

**Implementado:** se preparó el ejecutable manual `api/scripts/mej-006-dry-run-2025.mjs`, apoyado por `api/src/modules/employee-evaluations/evaluations/mej006DryRun.utils.js`. El cálculo importa y usa directamente `calculateEvaluationScore` de `evaluationScore.utils.js`; no replica la fórmula P2 ni llama a `calculateAndSaveScore`.

**Pendiente de verificar:** en el entorno de trabajo no existe `MEJ006_READONLY_DATABASE_URL`, no se expuso otra conexión identificada y autorizada de solo lectura, y no se encontró el archivo `Evaluacion_8120.pdf` en los repositorios habilitados. En cumplimiento de `AGENTS.md`, no se inspeccionaron archivos `.env`, credenciales ni datos personales y no se utilizó `DB_URL` como alternativa. Por ello todavía no se pueden afirmar:

- el `periodId` real de «Evaluación de Desempeño - Año 2025»;
- los ocho `PeriodGroup.id` reales ni sus descripciones exactas;
- los conteos reales por estado, los registros de respuesta o la escala snapshot;
- el resumen real de diferencias;
- que `8120` sea la PK PostgreSQL correspondiente al PDF ni que sus 33 respuestas sumen 102.

La consulta inicial que debe ejecutarse con el rol autorizado es:

```sql
SELECT id, name, start_date, end_date, is_active
FROM evaluation.evaluation_periods
WHERE name = 'Evaluación de Desempeño - Año 2025'
ORDER BY id;
```

Si retorna cero o más de una fila, se debe documentar la discrepancia y no elegir un período por aproximación. El `periodId` confirmado se suministra expresamente al script; este vuelve a exigir que el nombre coincida exactamente.

### 7.2 Mapa obligatorio de grupos históricos

El mapa real queda pendiente de completar después del inventario. No se asigna ningún peso por orden ni por coincidencia automática de texto. Cada entrada debe contener el ID snapshot, la descripción copiada exactamente del inventario, una clave de negocio explícita y el peso aprobado:

| Clave de negocio | `PeriodGroup.id` real | Descripción snapshot exacta | Peso |
|---|---:|---|---:|
| `ORIENTACION_A_RESULTADOS` | Pendiente | Pendiente | 20 % |
| `CALIDAD` | Pendiente | Pendiente | 10 % |
| `RELACIONES_INTERPERSONALES` | Pendiente | Pendiente | 10 % |
| `TRABAJO_EN_EQUIPO` | Pendiente | Pendiente | 5 % |
| `PLANEACION` | Pendiente | Pendiente | 15 % |
| `INNOVACION_Y_COMUNICACIONES` | Pendiente | Pendiente | 5 % |
| `ACTITUD` | Pendiente | Pendiente | 15 % |
| `RESPONSABILIDADES_EN_SST` | Pendiente | Pendiente | 20 % |

Formato del archivo revisado, donde los valores ilustrativos deben reemplazarse por los IDs y textos reales:

```json
[
  {
    "periodGroupId": 0,
    "snapshotDescription": "DESCRIPCIÓN EXACTA DEL SNAPSHOT",
    "businessGroup": "ORIENTACION_A_RESULTADOS",
    "weightPercentage": 20
  }
]
```

El script rechaza IDs faltantes, desconocidos o duplicados; claves de negocio faltantes o duplicadas; descripciones que no coinciden exactamente; pesos diferentes a la matriz aprobada; cantidad distinta de ocho grupos; grupos sin preguntas; preguntas sin grupo válido; y cualquier `weight_percentage` persistido no nulo. Este último control demuestra que el snapshot continúa en modo `LEGACY`; el mapa solo existe en memoria durante la simulación.

### 7.3 Controles de solo lectura

El ejecutable:

1. acepta exclusivamente `MEJ006_READONLY_DATABASE_URL`; no cae en `DB_URL` ni carga `.env`;
2. abre `BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ READ ONLY`;
3. verifica `transaction_read_only = on`, acceso `SELECT` a las seis tablas consultadas, rol no superusuario y ausencia de privilegios `INSERT`, `UPDATE`, `DELETE` y `TRUNCATE` sobre esas tablas;
4. usa solamente sentencias `SELECT` dentro de la transacción y finaliza con `ROLLBACK`;
5. no contiene `UPDATE`, migraciones, colas, correo, cron ni llamadas a `calculateAndSaveScore`.

Incluye todas las evaluaciones `COMPLETED` del período sin filtrar por `is_active`; el CSV conserva esa bandera. También incluye preguntas snapshot activas e inactivas, porque la fórmula P2 opera sobre el snapshot completo.

### 7.4 Ejecución

Desde `api/`, primero se genera el inventario verificable:

```bash
MEJ006_READONLY_DATABASE_URL='postgresql://ROL_SOLO_LECTURA@HOST/BASE' \
npm run audit:mej-006 -- \
  --period-id=ID_REAL \
  --inventory-only \
  --output-dir=/RUTA/SEGURA/mej-006
```

El inventario contiene período, grupos, preguntas con descripción y grupo, escala, totales por estado/actividad, evaluaciones completadas y respuestas sin datos personales. Después de revisar los ocho emparejamientos y guardar el JSON de pesos:

```bash
MEJ006_READONLY_DATABASE_URL='postgresql://ROL_SOLO_LECTURA@HOST/BASE' \
npm run audit:mej-006 -- \
  --period-id=ID_REAL \
  --weights-file=/RUTA/SEGURA/pesos-2025-revisados.json \
  --output-dir=/RUTA/SEGURA/mej-006
```

Se generan:

- `mej-006-period-ID.csv`, con las doce columnas solicitadas;
- `mej-006-period-ID-summary.json`, con totales y control 8120;
- `mej-006-period-ID-audit.json`, con inventario, mapa aplicado, preguntas, escala, integridad de rangos, conteos de respuestas y desglose no personal del control 8120.

El proceso sale con código `2` después de escribir los reportes si el control 8120 no coincide; esto impide considerar el lote listo sin ocultar la evidencia observada.

### 7.5 Validaciones y exclusiones

Por evaluación se comprueba que cada respuesta apunte a una pregunta del mismo período, que no haya duplicados, que el valor sea entero entre 1 y 5, que existan respuestas para todas las preguntas y que ningún grupo quede incompleto. Antes de proponer el resultado ponderado se calcula el valor `LEGACY` con el mismo helper P2 y se exige coincidencia exacta con el score persistido a un decimal.

La clasificación propuesta reproduce el contrato actual de `calculateAndSaveScore`: selecciona el rango inclusivo que contiene el score y usa `PeriodScoreScale.description` como `ratingLabel`. Un hueco, solapamiento o discrepancia del `ratingLabel` histórico excluye la evaluación y queda registrado en `exclusionReason`. El inventario también reporta huecos y solapamientos de toda la escala a resolución de 0,1 puntos.

Los registros excluidos permanecen en el CSV; no se omiten silenciosamente. Los totales de cambios y las estadísticas de diferencias se calculan solo sobre registros `VALID`, mientras `completedEvaluations` y `excludedEvaluations` conservan el universo completo.

### 7.6 Pruebas ejecutadas

Se agregó `api/tests/employee-evaluations/mej-006-dry-run.test.js` con datos sintéticos, sin conexión a PostgreSQL. Resultados:

| Prueba | Resultado |
|---|---|
| Control sintético 8120: 33 respuestas, suma 102, `61,8 % -> 61,4 %`, acuse confirmado | **Aprobada** |
| Inclusión de evaluación `COMPLETED` inactiva | **Aprobada** |
| Exclusión por respuesta faltante/grupo incompleto | **Aprobada** |
| Exclusión cuando el persistido no coincide con LEGACY | **Aprobada** |
| Rechazo de mapa con descripción o peso no aprobado | **Aprobada** |
| Contrato y minimización de datos del CSV | **Aprobada** |
| Detección de huecos y solapamientos de escala | **Aprobada** |
| `node --test tests/employee-evaluations/mej-006-dry-run.test.js` | **6/6 aprobadas** |
| Regresión P2 `node --test tests/employee-evaluations/weighted-score.test.js` | **20/20 aprobadas** |
| Suite completa `node --test tests/employee-evaluations/*.test.js` | **38 aprobadas, 0 fallidas, 1 omitida** por no suministrar la base PostgreSQL aislada opcional |
| `npm run type-check` | **Aprobada** |
| `git diff --check` | **Aprobada** |

El lote sintético de control contenía cuatro evaluaciones `COMPLETED`: una sin cambios, una con cambio de `-0,4` puntos y dos excluidas por inconsistencias; ninguna cambió de clasificación, una tenía acuse confirmado, la diferencia promedio de las dos filas válidas fue `-0,2`, la máxima negativa `-0,4` y no hubo diferencia positiva. Este resumen solo demuestra el comportamiento de los contadores y la reutilización de la fórmula; **no confirma ningún dato histórico real**.

### 7.7 Precauciones para una futura actualización

- Revisar conjuntamente el CSV, el mapa de IDs y todas las exclusiones antes de diseñar cualquier escritura.
- Preservar export firmado/checksum de `score`, `rating_label`, acuse y procedencia del PDF original antes de corregir.
- No rellenar `period_groups.weight_percentage` del período 2025 ni cambiar respuestas, acuses o PDFs originales.
- Resolver primero cualquier discrepancia del caso 8120; no forzar `61,4 %` si la base no contiene las 33 respuestas/suma 102 documentadas.
- Definir la política de versión rectificada del PDF sin presentar el acuse anterior como aceptación de una nueva calificación.
- Una futura herramienta de escritura debe ser separada, transaccional, idempotente, limitada a IDs aprobados y exigir que el valor original siga coincidiendo con el export revisado. No se preparó ni ejecutó SQL `UPDATE` en MEJ-006.1.


## 8. Identificador de periodo confirmado por el responsable (2026-10-21)

El responsable confirmó que el periodo histórico a auditar corresponde a **`periodId = 6`**. Se registra como identificador informado por el responsable; el script debe **verificarlo contra PostgreSQL** y exigir que el nombre de la campaña coincida con «Evaluación de Desempeño - Año 2025» antes de procesar.

La conexión `MEJ006_READONLY_DATABASE_URL` **no es una base de datos adicional**: puede apuntar a la misma base `sisproing` configurada en el API, pero requiere un rol PostgreSQL de solo lectura. Por diseño, `mej-006-dry-run-2025.mjs` no carga `.env` ni usa `DB_URL` como sustituto. La transacción `READ ONLY` no reemplaza la exigencia de privilegios exclusivamente `SELECT`. No introducir credenciales en el repositorio ni en el documento.

El primer comando tras configurar la variable es:

```bash
npm run audit:mej-006 -- \\
  --period-id=6 \\
  --inventory-only \\
  --output-dir=./tmp/mej-006
```

**Estado:** `periodId` conocido por declaración del responsable, pendiente de confirmación contra la base; inventario y dry-run reales aún no ejecutados. **Ningún cambio a PostgreSQL autorizado.**


## 9. Inventario real periodo 6 y conciliación pendiente con Excel manual (2026-10-21)

**Fuente verificada:** archivo exportado de conexión **solo lectura** al PostgreSQL de pruebas: `mej-006-period-6-inventory.json` (`generatedAt=2026-10-21T22:17:22.778Z`). La conexión no tiene privilegios de mutación, las seis tablas eran legibles y la transacción estaba en solo lectura. Este inventario corresponde a una **copia de pruebas de producción** cuyo corte es aproximadamente un mes anterior; los conteos deberán reconciliarse nuevamente al planificar producción.

### 9.1 Grupos e IDs del snapshot

| Grupo histórico | ID | Preguntas | Peso aprobado | weight_percentage almacenado |
|---|---:|---:|---:|---|
| Orientación a resultados | 45 | 2 | 20 % | NULL |
| Calidad | 46 | 5 | 10 % | NULL |
| Relaciones interpersonales | 47 | 4 | 10 % | NULL |
| Trabajo en equipo | 48 | 3 | 5 % | NULL |
| Planeación | 49 | 3 | 15 % | NULL |
| Innovación y comunicaciones | 50 | 4 | 5 % | NULL |
| Actitud | 51 | 4 | 15 % | NULL |
| Responsabilidades en SST | 52 | 8 | 20 % | NULL |
| **Total** | **8 IDs** | **33** | **100 %** | **LEGACY** |

Se preparó el JSON de mapa de pesos con el contrato exacto del script (`periodGroupId`, `snapshotDescription`, `businessGroup`, `weightPercentage`), **sin escribir en DB**. El responsable debe validar el mapeo contra el inventario y ejecutar el dry-run oficial antes de cualquier corrección.

### 9.2 Universo del inventario

- `COMPLETED is_active=true`: **1.384**; `COMPLETED is_active=false`: **15**; total **1.399 COMPLETED**.
- `IN_PROGRESS is_active=false`: **2**; `PENDING is_active=false`: **136**. Estos 138 no entran al recálculo.
- **46.167** respuestas para los 1.399 `COMPLETED` (= 33 por evaluación), sin duplicaciones de par evaluación/pregunta ni puntuaciones fuera de 1–5 en este inventario.
- **467** completadas con `notificationReceived=true`.
- La evaluación `8120` figura con `periodId=6`, `score=61,8`, `COMPLETED` y acuse confirmado. Su score ponderado estimado a partir del inventario es **61,4**.

### 9.3 Verificación matemática independiente (no es el dry-run oficial)

Se ejecutó un análisis **local sobre el JSON**: por cada evaluación, puntuación LEGACY `sum(answers)/(33*5)*100`, ponderación por los ocho grupos sin redondeo intermedio y un decimal final (redondeo half-up para valores positivos). Este análisis **no sustituye** la ejecución del código de P2 en el script oficial.

- **1.390/1.399** scores almacenados coinciden con LEGACY; **9 difieren**.
- Los IDs discrepantes (score almacenado / LEGACY recalculado / ponderado estimado) son:
  - `7986`: `91,0 / 89,7 / 90,5`
  - `8002`: `88,0 / 86,7 / 88,1`
  - `8431`: `83,0 / 83,6 / 82,8`
  - `8710`: `66,0 / 69,1 / 66,5`
  - `8833`: `80,0 / 78,8 / 79,5`
  - `8942`: `88,0 / 88,5 / 88,4`
  - `8975`: `92,0 / 92,1 / 92,1`
  - `9095`: `74,0 / 72,7 / 74,0`
  - `9231`: `77,0 / 76,4 / 77,0`
- **No interpretar estas nueve diferencias como corrupción ni sobrescribirlas:** puede haber correcciones anteriores no documentadas. Dos coinciden exactamente con el ponderado estimado (`9095`, `9231`), pero eso no demuestra el origen.
- Diferencia teórica al aplicar pesos a los 1.399 scores almacenados: **1.206 cambian, 193 no cambian** (cifras exploratorias, incluyen los nueve excepcionales y casos de escala no válida; NO representan las filas autorizadas para update).

### 9.4 Escala del periodo: condición bloqueante para clasificación automática

La escala snapshot es: **20–40**, **41–60**, **60–80**, **81–100** con extremos inclusivos. Al trabajar con resultados a un decimal:

- **60,0** pertenece a dos categorías: **37** scores actuales y **37** scores ponderados estimados caen exactamente allí.
- **80,1–80,9** no pertenecen a ninguna: **44** scores almacenados y **63** scores ponderados estimados caen allí.
- No se debe asignar `ratingLabel` automáticamente sin resolver la regla de límites y documentar la decisión. El dry-run oficial excluye o marca ambigüedades de escala según su validación.

### 9.5 Conciliación con Excel de correcciones anteriores: archivo pendiente

Se encontró disponible el libro **`HF-011 Evaluación de Desempeño Rev9 (2).xlsx`**, que es la **plantilla del formulario** y **no contiene el archivo de calificaciones corregidas por `evaluationId`**. El responsable solicitó cruzar también los IDs/valores del Excel corregido previamente en otro proceso. **No se ha hecho todavía ese cruce por falta de la fuente correcta.**

Se preparó un libro de conciliación derivado del JSON con los **1.399 `evaluationId`**, `score` persistido, `legacy` recalculado, resultado ponderado estimado, etiquetas del snapshot y los **9 casos excepcionales**. El campo del score corregido del Excel queda sin llenar hasta recibir el archivo verdadero.

**Protocolo cuando llegue el Excel:** cruzar por **`evaluationId`**, NUNCA por orden de fila; detectar IDs exclusivos en DB/Excel, duplicados, faltantes, scores almacenados que ya coinciden con Excel y diferencias Excel vs cálculo P2. Clasificar las nueve anomalías sin forzar su recalculo y generar la matriz definitiva de actualización **solo tras revisión y aprobación explícita**.

No se ejecutaron SQL UPDATE, modificaciones de snapshots, correos, acuses ni migraciones.


## 10. Conciliación con compilado Excel manual (2026-10-21)

**Nueva fuente:** `Compilado_evaluaciones_desempeño (1).xlsx`, hoja única `_WITH_group_scores_AS_SELECT_e_`, 1.384 filas de empleados, columnas de ocho porcentajes de grupos (F:M) y `PUNTAJE TOTAL` entero (N). El archivo **NO** incluye `evaluationId`, por lo cual el cruce con el JSON por IDs **no es una vinculación inequívoca de personas**. La cédula existe en Excel, pero no está en el inventario JSON (deliberadamente minimizado).

**Resultado del cotejo local, sin DB:** se calculó para cada evaluación `COMPLETED` activa el vector de **ocho porcentajes de grupo, redondeados a entero**, a partir de las 33 respuestas snapshot del JSON. Los multiconjuntos de estos vectores coinciden **exactamente** entre las **1.384 activas** de PostgreSQL y las **1.384 filas** del Excel (misma cantidad y multiplicidad para cada uno de los 1.012 vectores distintos). **923 evaluaciones activas** tienen vector que aparece una sola vez; **461** tienen vector compartido, sin identidad individual comprobada por esa sola clave. Los **15 `COMPLETED` inactivos** no forman parte del archivo Excel, y permanecen sujetos a revisión separada.

**Nueve puntajes manuales confirmados por el responsable:** el responsable confirmó que los IDs `7986, 8002, 8431, 8710, 8833, 8942, 8975, 9095, 9231` fueron corregidos manualmente conforme al Excel. El cotejo por vector demuestra que para cada uno de los nueve el `score` actualmente persistido **coincide exactamente** con el `PUNTAJE TOTAL` entero del Excel. Para `8833` hay 10 filas candidatas por el mismo vector; para `9231` hay 2; las otras 7 cuentan con vector único. Se mantiene pendiente confirmar la identidad mediante relación `Evaluation.employeeId → Employee/documento` y `CEDULA` de Excel, antes de un `UPDATE`.

| `evaluationId` | Score BD | Score LEGACY | Total Excel (entero) | P2 ponderado (1 decimal) |
|---:|---:|---:|---:|---:|
| 7986 | 91,0 | 89,7 | 91 | 90,5 |
| 8002 | 88,0 | 86,7 | 88 | 88,1 |
| 8431 | 83,0 | 83,6 | 83 | 82,8 |
| 8710 | 66,0 | 69,1 | 66 | 66,5 |
| 8833 | 80,0 | 78,8 | 80 | 79,5 |
| 8942 | 88,0 | 88,5 | 88 | 88,4 |
| 8975 | 92,0 | 92,1 | 92 | 92,1 |
| 9095 | 74,0 | 72,7 | 74 | 74,0 |
| 9231 | 77,0 | 76,4 | 77 | 77,0 |

**Diferencia de precisión:** el Excel compila un resultado entero, mientras P2 aprobado conserva **un decimal**. Solo **199/1.384** puntajes guardados coinciden numéricamente con el entero del Excel y 1.185 difieren; no equivale a decir que 1.185 fueron corregidos incorrectamente. El total del Excel coincide con redondear a entero el valor ponderado **sin redondeo previo** en **1.343/1.384** casos; en **41** difiere, lo que exige revisar criterio histórico de redondeo y **NO** copiar indiscriminadamente `PUNTAJE TOTAL` para actualizar `score`.

**Artefacto generado:** `MEJ-006-conciliacion-Excel-manual-vs-JSON-periodo-6.xlsx` con hojas `Resumen`, `Conciliacion`, `Nueve corregidos`, `Grupos y pesos`, `Metodo`. Incluye 1.399 evaluaciones completadas, sin cédulas ni nombres para minimizar exposición. Los cálculos de diferencia quedan como fórmulas en el reporte.

**Siguiente control:** ejecutar el **dry-run oficial de Codex** con mapa de IDs 45–52 y obtener CSV/JSON de exclusiones, especialmente escala `60` solapada y hueco `80,1–80,9`. Para conciliación individual definitiva, Codex debe añadir, en modo solo lectura, un cruce local seguro con `CEDULA` del Excel contra los campos de identidad de `Employee`, sin transmitir ni registrar datos personales innecesarios y sin alterar las validaciones del simulador. No se ha autorizado ningún SQL UPDATE.


## 11. SQL transaccional de corrección preparado para pruebas (2026-10-21)

**Autorización vigente:** el responsable solicitó generar el SQL para ejecutarlo **primero sobre la copia de pruebas**, verificar allí con nuevo inventario JSON y considerar después producción. No se ejecutó SQL sobre ninguna base desde ChatGPT. La conexión de solo lectura se mantiene para auditoría; el SQL correctivo requiere un rol separado con permisos de escritura.

Se generaron cuatro artefactos descargables en esta conversación, usando el inventario real `mej-006-period-6-inventory.json` con timestamp `2026-10-21T22:17:22.778Z`:

- `MEJ-006-periodo-6-01-PREVALIDACION.sql`: transacción y tablas temporales; recalcula todos los puntajes desde PostgreSQL y compara scores, labels, flags, grupos, 33 preguntas, huellas MD5 de respuestas y conteos, finaliza con `ROLLBACK`.
- `MEJ-006-periodo-6-02-APLICAR.sql`: mismos controles; crea backup persistente `evaluation.mej006_backup_period6_v1` para las **1.399** completadas y aplica un `UPDATE ... FROM` solo a las filas cambiadas, con chequeo posterior y `COMMIT`.
- `MEJ-006-periodo-6-03-VERIFICAR.sql`: coteja todos los 1.399 valores actualizados, acuses, datos de la evaluación y huellas de respuestas con el respaldo.
- `MEJ-006-periodo-6-04-REVERTIR.sql`: restitución transaccional de `score` y `rating_label` originales desde backup, solo si el estado actual sigue idéntico al nuevo plan aprobado.

**Resultados previstos de la matriz local:** **1.399** completadas; **1.206** cambian `score`; **193** sin cambios de score; **42** cambian `rating_label`; **405** con `notification_received=true` cuyo score cambia. Caso `8120: 61,8 → 61,4`. Los **9 puntajes manuales** están incluidos con comprobación de preimagen exacta, no se tratan como corrupción. Estos números deben verificarse mediante la prevalidación conectada.

**Decisión provisional sobre clasificación a validar en pruebas y aprobar antes de producción:** usar continuidad del etiquetado histórico observado: `[20,41)` deficiente, `[41,60)` bajo, `[60,81)` bueno y `[81,100]` sobresaliente. Con ello **60 = bueno** (como los scores existentes) y **80,1–80,9 = bueno**, sin modificar los límites snapshot `PeriodScoreScale`; los 42 cambios se derivan de esa regla. El origen almacena intervalos solapados/huecos; **es imprescindible aprobar explícitamente esta normalización antes de producción**. Los nuevos labels son las descripciones existentes del snapshot.

**Precaución producción:** la base de pruebas es una copia anterior a producción. Los SQL contienen los 1.399 IDs, preimágenes de score/label/acuse/isActive, hash de respuestas y el inventario de preguntas/grupos; si un solo caso difiere en producción, **el script aborta, no se deben eliminar validaciones ni forzar los registros**. Reauditar contra producción con conexión solo lectura, tomar backup y preservar evidencia de PDF/acuse originales antes de dar autorización final. `mej-006-dry-run-2025.mjs` excluye nueve scores corregidos manualmente y valores fuera de rango histórico; tras UPDATE su validación LEGACY excluirá la mayoría. Usar `--inventory-only` y `03-VERIFICAR.sql` para cotejo posterior, no interpretar el dry-run LEGACY posactualización como prueba de regresión válida.

**SHA256 artefactos:** PRE `cc1fa0eb986b23b3d15aec3adf77b69bc9f066c32bcfd88c4868cabe7e3c2080`; APLICAR `251c0b714f95a92d52999076a654f7ee91441c335d288f02d50e679caa1bb967`; VERIFICAR `b3925f6971c32635fa24649be4be3c0db92670a6b62e0ce26ac371ce06517626`; REVERTIR `8cbc00b6c6b2d557a72557da36bc2e284119b8a37e91a644c527e48954144308`.

**Pendiente:** validar SQL ejecutándolo realmente en PostgreSQL de pruebas, confirmar que los nombres físicos de columnas son correctos, inventario posterior, decisión sobre clasificación y PDFs antes de producción.


## 12. Validación post-UPDATE en pruebas y puerta a producción (2026-10-21)

**Evidencia de ejecución en pruebas:** el responsable informó haber ejecutado `01-PREVALIDACION.sql`, `02-APLICAR.sql` y `03-VERIFICAR.sql`, este último con **cero diferencias de huellas de respuestas**. Aportó el JSON posterior `mej-006-period-6-inventory(1).json` (`generatedAt=2026-10-21T23:26:45.747Z`) generado mediante `--inventory-only` con rol de solo lectura.

**Cotejo de los dos JSON reales, por cada `evaluationId`:**

| Control | Verificado |
|---|---:|
| Universo completado | 1.399 antes = 1.399 después; sin IDs nuevos/faltantes |
| Puntaje `score` modificado | **1.206** |
| Puntaje sin cambios | **193** |
| `ratingLabel` modificado | **42** |
| Puntaje modificado con `notificationReceived=true` | **405** |
| `notificationReceived`, `status`, `isActive` modificados | **0** |
| Respuestas `responseRecords` modificadas | **0 de 46.167**, listas JSON idénticas |
| Grupos, preguntas, escalas, contadores y periodo modificados | **0** |
| Caso `8120` | **61,8 % → 61,4 %**; acuse `true` preservado |

**Control aritmético independiente:** se calcularon nuevamente en memoria, desde los **46.167 registros de respuesta del JSON posterior**, los puntajes P2 de los 1.399 `COMPLETED` usando IDs `45–52`, división por las preguntas de cada grupo, pesos 20/10/10/5/15/5/15/20 y redondeo decimal **solo al final a un decimal**. **0 diferencias frente a `score` persistido**, **0 diferencias de clasificación** frente a la política de límites usada en SQL, **1.399/1.399 con exactamente 33 respuestas**.

**Conclusión técnica en pruebas:** corrección satisfactoria. No hay evidencia en el JSON de alteración de respuestas, acuses, estados, grupos, preguntas o escalas. **No implica que los PDFs existentes o la aceptación original estén salvaguardados documentalmente**, ni que las condiciones de producción sean idénticas.

**Antes de ejecutar producción:**

1. Confirmar inequívocamente host/base productivos y planear un periodo de ejecución sin mutaciones simultáneas. Tomar respaldo externo de BD o del ámbito afectado, independiente de la tabla `evaluation.mej006_backup_period6_v1`.
2. Conservar las versiones originales de PDFs/acuse ya aceptadas: los **405** casos de `score` cambiado con acuse y las **42** reclasificaciones requieren decisión de Gestión Humana sobre evidencia histórica / documentos rectificados. Probar de nuevo PDF y reportes (por ejemplo `8120`) en el ambiente de pruebas, y confirmar que no se recalcula al método LEGACY en la visualización.
3. Generar `--inventory-only` sobre producción actual y contrastar contra la matriz del SQL de forma completa. La copia de pruebas tiene un mes de desfase; el mismo SQL solo es aplicable si coincide **cada ID, score viejo, label, estado, acuse, pregunta y respuesta**, además del universo y snapshot. **Si `01-PREVALIDACION.sql` falla, NO ejecutar `02-APLICAR.sql`, no editar salvaguardas; regenerar la matriz desde el inventario productivo autorizado.**
4. Bajo controles anteriores, ejecutar `01-PREVALIDACION.sql` completo sobre producción y comprobar 1.399/1.206/193/42/405 y control 8120. Solo después, con autorización operativa expresa, ejecutar `02-APLICAR.sql` completo (respaldo + UPDATE + COMMIT).
5. Ejecutar `03-VERIFICAR.sql` y un nuevo inventario post; confrontar IDs, score, clasificación, respuesta, acuses, `isActive`, escalas y grupos contra la foto pre. Verificar muestra de PDFs/reportes; conservar `04-REVERTIR.sql` para contingencia, sujeto a los checks de reversión.

**Estado:** **VALIDADO EN PRUEBAS, PRODUCCIÓN NO EJECUTADA**. El mismo SQL depende de que el inventario productivo sea exactamente el esperado; la validación de evidencia documental de PDFs y aprobación de normalización de límites siguen siendo decisiones funcionales previas.


## 13. Producción: error de columna ausente y DDL correctivo preparado (2026-10-21)

Al ejecutar `01-PREVALIDACION.sql` en el servidor reportado de producción (`sisproing`, rol `desarrollos`, servidor `172.31.9.6`), el bloque de validación `Query 13` falló porque **`evaluation.period_groups.weight_percentage` no existe**. Por ello **no pasó la prevalidación ni se autorizó el `02-APLICAR.sql` en producción**.

En P2.1 están definidas dos columnas nuevas (Prisma `Decimal? @db.Decimal(5,2)`): `evaluation.question_groups.weight_percentage NUMERIC(5,2) NULL` y `evaluation.period_groups.weight_percentage NUMERIC(5,2) NULL`. La migración local preparada es `api/prisma/migrations/20261020100000_add_group_weight_percentage/migration.sql` (historial Prisma con divergencias conocidas; no ejecutar `prisma migrate deploy` indiscriminadamente).

Se entregó el SQL **`MEJ-006-PRODUCCION-00-esquema-pesos-grupos.sql`** para aplicar ambas columnas con `ADD COLUMN IF NOT EXISTS`, restricciones CHECK de 0–100, validación de tipo y verificación del periodo 6 con ocho snapshots y pesos `NULL`; no contiene `UPDATE` de pesos, respuestas ni puntajes. **Este archivo se ha preparado, no ejecutado desde ChatGPT.** Los nombres de restricciones de este script deben conciliarse con la migración Prisma antes de una posterior ejecución formal de migraciones, para evitar restricciones lógicamente duplicadas o migraciones pendientes marcadas erróneamente.

**Siguientes controles en producción:** confirmar backup/host; ejecutar DDL, verificar columna y `weight_percentage=NULL` de periodo 6; inspeccionar también si otros periodos nuevos requieren snapshot ponderado válido, **sin rellenarlos automáticamente**; correr `01-PREVALIDACION.sql` de nuevo completamente y revisar controles de 1.399/1.206/193/42/405. Si difiere cualquier registro productivo, detener y regenerar matriz desde inventario productivo: no omitir controles. Solo pasar a `02-APLICAR.sql` luego de las salvaguardas de respaldo y PDF/acuse que figuran en sección 12.


## 14. Aplicación productiva reportada y verificación parcial (2026-10-21)

**Ejecución reportada por el responsable en producción.** Tras agregar las columnas de pesos P2 a `evaluation.question_groups` y `evaluation.period_groups`, el responsable ejecutó `02-APLICAR.sql` y `03-VERIFICAR.sql` en el servidor identificado previamente como `172.31.9.6` / base `sisproing`. Las capturas entregadas muestran:

- Resumen del SQL de aplicación: **1.399 completadas; 1.206 cambios de score planificados; 193 sin cambios; 42 cambios de clasificación; 405 acuses asociados a cambios de score**. Este resumen, por sí solo, no acredita el número afectado físicamente por el `UPDATE`.
- Consulta 3 de `03-VERIFICAR.sql`: **`discrepancias_post_update = 0`**. Compara la tabla productiva `evaluation.evaluations` con `evaluation.mej006_backup_period6_v1` en identidad, periodo, estado, `is_active`, acuse y fecha de acuse, `score` y `rating_label` esperados.
- **Existe respaldo persistente**, pues `03-VERIFICAR.sql` pudo consultar la tabla `evaluation.mej006_backup_period6_v1`; confirmar igualmente `universo_backup=1399` mediante la primera consulta de ese archivo.

**Pendientes de evidencia para cierre definitivo:** capturas o resultados de la primera consulta de `03-VERIFICAR.sql` (`universo_backup=1399`), cuarta consulta (muestra `8120: old_score 61,8; expected_new_score=score_actual=61,4`), quinta consulta (**`respuestas_con_huella_diferente=0`**), y un inventario postproductivo con `--inventory-only` para cotejo integral. Conservar el respaldo original, no reejecutar `02-APLICAR.sql`; su `CREATE TABLE` está concebido para fallar si ya existe. El `ROLLBACK` al final del archivo 03 es **normal**, porque 03 comienza con una transacción `READ ONLY`.

**Control funcional y documental pendiente:** verificar reporte/PDF de muestra 8120 en producción; conservar el PDF originalmente aceptado separado de la versión rectificada, sin considerar el acuse original como aceptación del nuevo puntaje. Aprobar registro y comunicación con Gestión Humana para las 42 reclasificaciones y 405 evaluaciones con acuse cuyo puntaje varió. Mantener resguardo externo del backup antes de considerar la tarea cerrada.

**Estado:** **ACTUALIZACIÓN PRODUCTIVA REPORTADA; contraste post-update de score/estado/acuse OK con cero discrepancias; integridad completa de huellas y PDF pendiente de evidencia en este hilo.**
