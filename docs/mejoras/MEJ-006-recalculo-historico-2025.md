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
