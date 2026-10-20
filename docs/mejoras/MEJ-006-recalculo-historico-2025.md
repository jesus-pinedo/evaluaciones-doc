# MEJ-006 — Recálculo controlado de puntuaciones históricas HF-011 (aplazado)

**Proyecto:** PROING — Evaluación de Desempeño
**Fecha de acuerdo:** 2026-10-20
**Estado:** **BACKLOG / APLAZADO HASTA EL FINAL**. No autorizado todavía para ejecución ni para generar SQL de actualización definitivo.
**Dependencia:** P2 finalizado técnicamente; antes de desplegar cualquier rectificación histórica, verificar datos y respaldos autorizados.

## 1. Motivo y decisión

Las evaluaciones completadas del periodo histórico 2025 guardaron sus `score` mediante la fórmula LEGACY (suma de respuestas / total de preguntas × 5, porcentaje a un decimal). En P2 se aprobó y probó la ponderación por los ocho grupos según pesos explícitos: **Orientación a resultados 20 %, Calidad 10 %, Relaciones interpersonales 10 %, Trabajo en equipo 5 %, Planeación 15 %, Innovación y comunicaciones 5 %, Actitud 15 %, SST 20 %**.

El responsable decide **posponer hasta el final** la eventual actualización masiva de los scores históricos; la prioridad inmediata es revisar la pantalla «Mis Evaluaciones» y su periodo operativo.

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

**Estado hoy:** plan documentado, ninguna modificación de datos.
