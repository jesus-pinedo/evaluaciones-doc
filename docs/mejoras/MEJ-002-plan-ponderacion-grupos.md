# MEJ-002 — Plan acotado para ponderación por grupos

**Fecha:** 2026-10-20  
**Estado:** P2.1 implementado y probado funcionalmente según confirmación del responsable; columnas aplicadas manualmente por SQL. P2.2 y P2.3 implementados y validados técnica e integralmente en entornos aislados. Pendiente únicamente la comprobación operativa de solo lectura sobre el histórico real de 2025.
**Fuente:** auditoría [MEJ-002](MEJ-002-calculo-puntuaciones.md), formulario de grupos compartido por el responsable y plantilla HF-011 Rev9 inspeccionada en sus fórmulas internas.

## Decisiones del responsable

1. **No corregir, recalcular ni actualizar los puntajes ya almacenados.** El histórico, incluido 2025, queda intacto. Cualquier corrección histórica se hará en **otro proceso posterior** con aprobación independiente.
2. La mejora actual busca **representar y persistir el peso numérico por grupo** y posteriormente aplicar la regla de ponderación a evaluaciones de **periodos nuevos**, no reparar automáticamente evaluaciones finalizadas.
3. **Sin refactoring:** no centralizar por ahora todo el motor de puntuación, reescribir servicios, PDF, correo, clasificación ni abordar defectos ajenos a esta regla. Los cambios deben ser quirúrgicos en código existente.
4. **No obtener el peso mediante parsing de `description`**. Los porcentajes del nombre actual son datos de presentación y no son una fuente confiable de persistencia.
5. **No ejecutar migraciones ni modificaciones sobre producción** desde Codex sin aprobación explícita; proponer migración/versionamiento compatible y validar en entorno aislado.

## Datos observados en la captura

| Grupo | Peso declarado en el nombre |
|---|---:|
| Calidad | 10 % |
| Cultura Organizacional - Actitud | 15 % |
| Cultura Organizacional - Innovación y Comunicaciones | 5 % |
| Cultura Organizacional - Planeación | 15 % |
| Cultura Organizacional - Trabajo en Equipo | 5 % |
| Orientación a Resultados | 20 % |
| Relaciones Interpersonales | 10 % |
| Responsabilidades en SST | 20 % |
| **Suma** | **100 %** |

Es una evidencia fuerte de ponderación intencional, **pero la fórmula exacta de la plantilla Excel HF-011 Rev9 aún debe verificarse en sus celdas/fórmulas** antes de aprobar el comportamiento matemático definitivo.

## Propuesta de fórmula para validar contra Excel

Porcentaje por grupo (sin redondear antes de la agregación):

`scoreGrupo = (sum(respuestas validas del grupo) / (totalPreguntasGrupo * 5)) * 100`

Total ponderado, si los pesos de los grupos aplicables suman 100 %:

`scoreTotal = sum(scoreGrupo * (pesoGrupo / 100))`

Redondear el **total final** a un decimal según la convención existente, y no usar subtotales visuales ya redondeados en el cálculo.

**Pendiente de decisión:** tratamiento de preguntas sin grupo, grupos vacíos o sin preguntas, respuestas sin diligenciar, y comportamiento en un periodo que carezca de todos los pesos. No inventar valores ni normalizar automáticamente pesos incompletos.

## Fases de implementación — una por una

### P2.1 — Persistencia y edición del peso (implementado técnicamente)

Alcance permitido:

1. Analizar estrictamente el CRUD real de `QuestionGroup` y el proceso de publicación de `PeriodGroup`; verificar las formas de request/response, los modelos Prisma y el formulario React.
2. Proponer/implementar campo `weightPercentage` (nombre final conforme a convenciones del repositorio) como decimal en grupo global y grupo snapshot, inicialmente **nullable** para compatibilidad histórica. Ejemplo conceptual: `Decimal? @db.Decimal(5,2) @map("weight_percentage")`.
3. Actualizar la edición/guardado/consulta de grupos para presentar **nombre** y **peso (%)** como datos separados, con validación numérica del rango permitido. No extraer automáticamente los pesos de descripciones existentes.
4. Copiar el peso al snapshot **solo durante la publicación de nuevos periodos**, sin actualizar snapshots ya publicados.
5. Mostrar un **total de pesos configurados** para orientar al administrador, y preparar la validación de suma 100 % antes de publicar un nuevo periodo ponderado. No bloquear aún periodos históricos por falta de pesos.
6. Generar propuesta/archivos de migración si corresponden, **sin ejecutarlos**. Explicar el SQL exacto, efectos sobre valores `NULL`, compatibilidad y despliegue.
7. No cambiar todavía `calculateAndSaveScore` ni `useEvaluationForm.ts`; esos cambios corresponderán a P2.2 tras validar la fórmula del Excel.
8. No insertar/actualizar datos reales ni ejecutar publicaciones de prueba en entornos productivos.

Criterios de aceptación: el CRUD conserva descripción independiente del peso; se muestra 100 % para los ocho pesos de referencia cuando se ingresan manualmente; el nuevo snapshot copia exactamente el peso vigente al publicar en entorno aislado; snapshots históricos permanecen sin cambios; API y React compilan. Los registros anteriores con `NULL` siguen siendo legibles.

**Resultado 2026-10-20:** implementación técnica terminada. Se añadieron campos nullable, contratos CRUD, edición React, total acumulado, copia a snapshots y pruebas unitarias. La migración no fue ejecutada y las verificaciones integradas con base/UI quedan pendientes de un ambiente autorizado. El detalle está registrado en [MEJ-002, sección 17](MEJ-002-calculo-puntuaciones.md#17-implementación-p21--persistencia-y-edición-del-peso-2026-10-20).

### Validación reportada de P2.1 y migración manual (2026-10-20)

El responsable confirma que **aplicó manualmente las columnas SQL de P2.1 en PostgreSQL** y **realizó pruebas funcionales satisfactorias**. No reportó fallos de CRUD o UI; se registra como validación realizada por él, no como inspección independiente de base de datos. **No se recalcularon ni corrigieron puntuaciones**.

La base `sisproing` presenta historial Prisma incompleto; `prisma migrate status` mostró tres migraciones pendientes anteriores y `migrate deploy` falló con `P3005`. Después de la aplicación SQL manual **NO ejecutar `prisma migrate deploy` ni repetir la migración** sin reconciliar el historial/estado físico de las tablas. Registrar en la operación que las columnas existen por SQL manual, pero la migración `20261020100000_add_group_weight_percentage` **no está necesariamente marcada como aplicada** en `_prisma_migrations`. Regularizarlo en tarea técnica separada, sin inventar migraciones ejecutadas.

La fórmula ponderada **no está implementada aún**. No publicar campañas nuevas esperando puntuación ponderada hasta validar P2.2.

### P2.2 — Fórmula ponderada en nuevas evaluaciones (después de P2.1)

Precondiciones: contraste de la fórmula Excel, pesos ya persistidos, política de grupos/preguntas sin peso aprobada y compatibilidad para periodos antiguos definida.

Alcance: modificaciones puntuales a `evaluations.helper.js#calculateAndSaveScore`, `useEvaluationForm.ts#stats` y al DTO de grupos para consumir `weightPercentage`; pruebas numéricas contra una evaluación controlada de la plantilla.

**Invariante obligatorio:** `COMPLETED` históricos no se recalculan ni modifican, y **los valores históricos persistidos se siguen mostrando**, aunque ahora existan nuevos pesos configurados. No se interpreta automáticamente un `NULL` como cero ni como peso extraído de la descripción. Definir una separación explícita LEGACY/WEIGHTED (o mecanismo equivalente simple) para distinguir periodos antiguos de nuevos sin una refactorización general.

**Resultado 2026-10-20:** implementado con los ocho pesos explícitos `20/10/10/5/15/5/15/20`. La inspección del Excel confirmó que sus fórmulas asignan de forma efectiva `10 %` a cada subgrupo de Cultura Organizacional, pero el responsable aprobó deliberadamente usar en el software `5/15/5/15`. Node y React aplican la misma fórmula, la publicación nueva exige configuración ponderada válida y los periodos completamente sin pesos conservan LEGACY. Detalle y pruebas en [MEJ-002, secciones 18 y 19](MEJ-002-calculo-puntuaciones.md#18-validación-previa-p22--plantilla-excel-rev9-2026-10-20).

### Contrato de cálculo propuesto para P2.2 (sin refactoring)

- **Cálculo bruto del grupo:** `(suma de puntajes válidos de sus preguntas / (N_preguntas_del_grupo * 5)) * 100`.
- **Total ponderado:** `sum((porcentaje_grupo_sin_redondear * weightPercentage_grupo) / 100)` si todos los pesos son explícitos, válidos y suman 100 %.
- **Redondeo:** un decimal solo al total para persistencia y visualización acordada; subtotales no deben redondearse antes de ponderar.
- **Legacy:** los snapshots históricos sin pesos siguen usando la fórmula previa cuando sea necesario calcular un borrador; las evaluaciones `COMPLETED` históricas deben presentar el `score` persistido. No actualizar ni recalcular puntajes de 2025.
- **Nuevos periodos:** antes de publicar, exigir pesos explícitos en todos los grupos utilizados, suma exactamente 100 %, ninguna pregunta sin grupo y grupos que aportan peso con al menos una pregunta; si el Excel muestra otra regla, **detener la implementación del cálculo y documentar la discrepancia**.
- **Snapshots:** el cálculo usa exclusivamente `PeriodGroup.weightPercentage` del periodo, nunca `QuestionGroup` global ni el porcentaje incrustado en `description`.
- **Sin pesos/mix nulos en periodo ya publicado:** solo un histórico completamente sin pesos puede usar `LEGACY`; mezcla de pesos nulos y no nulos debe generar un error explícito, no aplicar una suma parcial silenciosa. La distinción entre `LEGACY` y `WEIGHTED` puede basarse en snapshots si es inequívoca; no crear una nueva arquitectura de fórmulas salvo necesidad demostrada.
- **Validar contra plantilla:** la captura contiene pesos que suman 100 %, pero las fórmulas internas de `HF-011 Evaluación de Desempeño Rev9 (2).xlsx` **siguen sin verificarse**. Codex debe inspeccionar el archivo original si está disponible en el workspace y confirmar fórmula de cada grupo y total. Si no puede acceder, documentar ese límite y detenerse antes de elegir por su cuenta una fórmula distinta.
- **Protección histórica:** las evaluaciones completadas conservan `score` persistido incluso en el encabezado React; no modificar `ratingLabel` histórico, PDFs, correos, reglas de clasificación ni reportes excepto que un cambio mínimo del resultado visual sea indispensable.
- **Validación:** ejemplos numéricos, casos de distinto número de preguntas, borrador parcial, escala 1–5, pesos 100 %, pesos `NULL`, grupo vacío, pregunta sin grupo, completada 2025 y dos periodos con pesos distintos. No crear registros ni enviar notificaciones reales en producción.

### P2.3 — Validación y cierre

Probar escenarios de múltiples grupos, diferentes cantidades de preguntas, pesos 100 %, fronteras y preguntas sin grupo; comparar con la hoja original. Revisar posibles discrepancias visibles en formulario/PDF sin rediseñar PDF/correos/clasificación en esta fase, salvo ajuste mínimo imprescindible y autorizado.

**Resultado 2026-10-20:** validación técnica terminada sin defectos funcionales detectados. Node confirmó mínimo `20 %`, máximo `100 %`, borrador incompleto, distinta cantidad de preguntas, redondeo y el caso discriminante `28 %`. Una sesión React contra un API sintético confirmó actualización inmediata, avance independiente, autoguardado, finalización y reapertura con `28 %`. Una prueba Prisma sobre PostgreSQL 15 desechable confirmó la persistencia del mismo resultado y que pesos/respuestas no cambiaron. La publicación se validó mediante la función bloqueante con todos los casos inválidos solicitados y el grupo vacío de peso cero permitido. No se usaron datos reales ni se ejecutaron migraciones o publicaciones. Matriz y evidencia completa en [MEJ-002, sección 20](MEJ-002-calculo-puntuaciones.md#20-p23--validación-funcional-e-integrada-2026-10-20).

El cierre técnico de P2 está completo. El cierre operativo total queda condicionado a consultar, con acceso explícitamente autorizado y de solo lectura, una muestra de evaluaciones `COMPLETED` del periodo real 2025 y confirmar en React que muestran el score persistido sin alteraciones.

## Exclusiones expresas

- **No corrección histórica de scores**, rating labels, respuestas ni PDF previos.
- No migración de puntajes, no actualización masiva, no nuevos reportes.
- No refactoring arquitectónico, no rediseño general de catálogo o formulario.
- No modificar inventario, transportes u otros módulos PROING.
- No actualizar pesos de grupos/snapshots históricos automáticamente a partir del texto.

## Referencias

- [MEJ-002 auditoría de cálculo](MEJ-002-calculo-puntuaciones.md)
- [Modelo de datos auditado](../tecnico/modelo-datos.md)
- [Reglas de negocio auditadas](../funcional/reglas-negocio.md)
