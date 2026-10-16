# MEJ-001 — P1.4 Vista previa con snapshots históricos

**Fecha de implementación:** 2026-10-16

**Estado:** implementado localmente y validado técnicamente; checklist manual pendiente

**Alcance:** vista previa de periodos en HF-011

## 1. Objetivo

Garantizar que la vista previa de un periodo publicado utilice exclusivamente los grupos y preguntas snapshot asociados a su `periodId`. Un borrador vacío, que aún no tiene snapshots, puede usar la configuración global activa siempre que la interfaz identifique claramente esa fuente.

Esta implementación es únicamente de lectura. No regenera snapshots, no modifica periodos, evaluaciones, respuestas o resultados y no cambia la publicación.

## 2. Hallazgos verificados

### Modelos snapshot existentes

`api/prisma/modules/evaluations.prisma` relaciona `EvaluationPeriod` con:

| Modelo | Contenido preservado |
|---|---|
| `PeriodClassification` | Clasificaciones del periodo. |
| `PeriodGroup` | Grupos/áreas de preguntas. |
| `PeriodQuestion` | Preguntas y relación con `PeriodGroup`. |
| `PeriodStrength` | Fortalezas. |
| `PeriodWeakness` | Debilidades y clasificación copiada como texto. |
| `PeriodScoreScale` | Escala de puntuación. |
| `Evaluation` | Asignaciones y resultados del periodo. |

Para la vista previa actual se necesitan `PeriodGroup` y `PeriodQuestion`. Las demás tablas permanecen intactas y continúan siendo consumidas por el formulario/detalle de evaluación.

### Servicios encontrados

- `groups.service.js#findAll` entrega `QuestionGroup` y sus preguntas globales; no recibe `periodId`.
- `questions.service.js#findAll` entrega preguntas globales y permite filtrar por `groupId`; no recibe `periodId`.
- `evaluations.service.js#getEvaluationById` incluye snapshots del periodo, pero exige un `evaluationId` y no corresponde a una vista previa administrativa del periodo.
- `periodSnapshots.service.js#createSnapshotsForPeriod` ya centralizaba la escritura transaccional al publicar, pero no tenía operación de lectura.

Por tanto, no existía un servicio ni endpoint apropiado para consultar grupos/preguntas snapshot directamente por `periodId`.

### Comportamiento anterior

`sisproing-app/.../periods/hooks/usePreviewModal.ts` consultaba una vez `groupsService.listAll()` y conservaba los grupos globales en memoria. Todas las tarjetas posteriores reutilizaban esa misma colección, independientemente del periodo seleccionado. Un histórico podía mostrar preguntas actuales y el cambio entre periodos podía conservar datos del anterior.

## 3. Contrato de lectura implementado

```http
GET /api/employee-evaluations/periods/:id/preview
```

Respuesta:

```json
{
  "data": {
    "periodId": 123,
    "source": "PERIOD_SNAPSHOT",
    "groups": []
  }
}
```

Fuentes posibles:

- `PERIOD_SNAPSHOT`: periodo publicado; grupos y preguntas provienen exclusivamente de `PeriodGroup` y `PeriodQuestion` filtrados por el mismo `periodId`.
- `GLOBAL_DRAFT`: borrador sin snapshots/evaluaciones; grupos y preguntas provienen de los catálogos globales activos y la interfaz muestra una advertencia.

Preguntas sin grupo se conservan en un grupo visual «Preguntas sin grupo»; no se omiten silenciosamente.

### Rechazos seguros

| Código | HTTP | Condición |
|---|---:|---|
| `EVAL_PERIOD_NOT_FOUND` | 404 | El periodo no existe. |
| `EVAL_PERIOD_SNAPSHOTS_MISSING` | 409 | Un publicado no tiene grupos/preguntas snapshot completos o las relaciones son inconsistentes. |
| `EVAL_PERIOD_SNAPSHOT_STATE_INCONSISTENT` | 409 | Un registro `isActive=false` conserva snapshots o evaluaciones; no se trata como borrador vacío. |

En los dos conflictos nunca se consulta la configuración global como reemplazo.

## 4. Protección contra reutilización entre periodos

Cada apertura de vista previa:

1. registra el nuevo periodo seleccionado;
2. limpia grupos, fuente y error anteriores;
3. abre el modal en estado de carga;
4. consulta `/:periodId/preview` sin caché compartida;
5. acepta la respuesta solo si corresponde al identificador interno de la solicitud más reciente.

Cerrar el modal invalida cualquier solicitud pendiente. Una respuesta lenta de un periodo anterior no puede sobrescribir la vista de uno abierto después.

## 5. Archivos modificados

API:

1. `api/src/modules/employee-evaluations/periods/snapshots/periodSnapshots.service.js`
2. `api/src/modules/employee-evaluations/periods/periods.service.js`
3. `api/src/modules/employee-evaluations/periods/periods.controller.js`
4. `api/src/modules/employee-evaluations/periods/periods.route.js`

React:

1. `sisproing-app/src/modules/employee-evaluations/periods/periods.service.ts`
2. `sisproing-app/src/modules/employee-evaluations/periods/hooks/usePreviewModal.ts`
3. `sisproing-app/src/modules/employee-evaluations/periods/components/previewModal.tsx`
4. `sisproing-app/src/modules/employee-evaluations/periods/PeriodsPage.tsx`

Documentación:

1. `evaluaciones-docs/docs/mejoras/MEJ-001-P1-4-snapshots-historicos.md`
2. `evaluaciones-docs/docs/mejoras/MEJ-001-historico-creacion-periodos.md`

No se modificó el esquema Prisma ni se creó migración.

## 6. Validación técnica

| Validación | Resultado |
|---|---|
| Inspección de modelos Prisma y relaciones por `periodId` | Correcto. |
| `node --check` dirigido a servicio, controlador y ruta | Correcto. |
| API `npm run type-check` | Correcto. |
| API `npm run build` | Correcto. |
| React `npm run typecheck` | Correcto. |
| ESLint dirigido a servicio, hook, modal y página de periodos | Correcto. |
| React `npm run build` | Correcto; solo advertencias informativas por datos desactualizados de Browserslist/baseline. |
| `git diff --check` en API, React y documentación | Correcto. |

Las consultas añadidas son exclusivamente `findUnique`/`findMany`. No se ejecutó el endpoint contra una base real y no hubo escrituras históricas.

## 7. Checklist manual — pendiente

Ejecutar en ambiente aislado y sin publicar, regenerar snapshots ni modificar catálogos durante la preparación:

| Caso | Acción | Resultado esperado | Estado |
|---|---|---|---|
| Histórico A | Abrir un publicado con preguntas conocidas A. | Banner de configuración histórica; solo grupos/preguntas snapshot A. | Pendiente |
| Histórico B diferente | Cerrar A y abrir un publicado con preguntas B. | Solo snapshots B; ninguna pregunta de A. | Pendiente |
| Cambio repetido A→B→A | Alternar rápidamente las vistas. | Cada apertura consulta su ID; ninguna respuesta anterior sobrescribe la actual. | Pendiente |
| Catálogo global modificado | Comparar un histórico con catálogos actuales distintos. | El histórico no cambia ni incorpora preguntas globales nuevas. | Pendiente |
| Borrador vacío | Abrir un `DRAFT` sin relaciones. | Banner ámbar `GLOBAL_DRAFT`; muestra configuración global activa actual. | Pendiente |
| Publicado sin grupos snapshot | Consultar fixture publicado incompleto. | 409 `EVAL_PERIOD_SNAPSHOTS_MISSING`; mensaje visible; sin fallback global. | Pendiente |
| Publicado sin preguntas snapshot | Consultar fixture publicado incompleto. | Mismo rechazo; no muestra tabla global. | Pendiente |
| Falso borrador histórico | `isActive=false` con snapshots/evaluaciones en fixture. | 409 `EVAL_PERIOD_SNAPSHOT_STATE_INCONSISTENT`; no se presenta como borrador. | Pendiente |
| Preguntas sin grupo | Abrir periodo que tenga snapshots con `groupId=null`. | Se muestran en «Preguntas sin grupo». | Pendiente |
| Solo lectura | Registrar conteos/fechas antes y después de varias consultas GET. | Ningún periodo, snapshot, evaluación o resultado cambia. | Pendiente |

## 8. Riesgos y pendientes

1. La integridad completa de snapshots históricos solo puede confirmarse con una validación de datos controlada. P1.4 detecta ausencia/inconsistencia en grupos y preguntas, pero no repara registros.
2. El modelo permite conceptualmente datos heredados `isActive=false` con relaciones; se rechazan para evitar mostrar catálogos globales sobre un posible histórico.
3. La vista previa actual representa grupos y preguntas. Incorporar fortalezas, debilidades, clasificaciones o escala a la UI requeriría una decisión funcional posterior; no forma parte de P1.4.
4. No se implementó caché por periodo. La consulta deliberadamente se repite en cada apertura para evitar contaminación entre periodos; una optimización futura debe usar una clave `periodId` y preservar invalidación.
5. El documento solicitado no existía al iniciar P1.4; este archivo se creó durante la implementación y se contrastó con las decisiones ya registradas en `MEJ-001-historico-creacion-periodos.md`.

## 9. Fuera de alcance confirmado

No se modificaron publicación, snapshots existentes, evaluaciones, respuestas, puntajes, PDF, selector de reportes P3, cron, colas ni otros módulos de PROING.
