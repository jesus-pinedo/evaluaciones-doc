# Modelo de datos

Fuente: `api/prisma/modules/evaluations.prisma` y relaciones estrictamente necesarias de `human-resources.prisma`.

## Catálogos globales

| Modelo / tabla | Campos principales | Relaciones y restricciones |
|---|---|---|
| `Classification` / `evaluation.classifications` | descripción, auditoría, activo | descripción única; 1:N con `Weakness`. |
| `Strength` / `strengths` | descripción, auditoría, activo | descripción única. |
| `Weakness` / `weaknesses` | nombre, descripción, `classificationId`, auditoría | una clasificación opcional; unicidad sobre descripción. |
| `QuestionGroup` / `question_groups` | descripción, auditoría, activo | descripción única; 1:N preguntas. |
| `Question` / `questions` | descripción, `groupId`, auditoría, activo | grupo opcional; descripción única. |
| `ScoreScale` / `score_scales` | label, descripción, min/max, orden, activo | sin constraints declarados para rangos, orden o solapamiento. |

## Periodo y snapshots

`EvaluationPeriod` contiene nombre, fechas, `isActive` y auditoría. Se relaciona con:

- `PeriodClassification`;
- `PeriodGroup` → `PeriodQuestion`;
- `PeriodStrength`;
- `PeriodWeakness` con clasificación copiada como texto;
- `PeriodScoreScale`;
- `Evaluation`.

Restricciones snapshot destacadas:

- grupo, fortaleza y debilidad son únicos por periodo y descripción/nombre;
- pregunta es única por periodo y descripción;
- no se conserva el ID global de origen;
- no existe snapshot/modelo de compromisos configurables.

## Evaluación

`Evaluation` referencia periodo, evaluador `Employee` y evaluado `Employee`. Guarda:

- `status`: `PENDING`, `IN_PROGRESS`, `COMPLETED`;
- `isActive`;
- `score` y `ratingLabel`;
- `notificationReceived` y fecha;
- `evaluatedAt` y auditoría.

Restricción única: `(periodId, evaluatorId, employeeId)`.

Detalles:

- `EvaluationAnswer`: una respuesta por evaluación/pregunta snapshot; `score Int?` sin `CHECK` de rango.
- `EvaluationStrength`: una selección por evaluación/fortaleza snapshot.
- `EvaluationWeakness`: una selección por evaluación/debilidad snapshot y `commitmentDescription` libre.

No hay FK compuesta que garantice que pregunta, fortaleza o debilidad pertenezcan al mismo periodo que la evaluación; esa validación tampoco aparece en el servicio.

## Recursos Humanos usados

- `Employee`: activo por `status`, correo, perfil, proyecto, jefe y roles evaluador/evaluado.
- `EmployeeManager`: un jefe opcional por empleado; `employeeId` es único.
- `EmployeeProfile`: nombre, apellidos, documento, correo y profesión.
- `EmployeeProjectAssignment`: empleado–proyecto y estado; única por empleado/proyecto.
- `EmployeeCguno`: cargo y centro de costo.
- `Project` y `StatusCode`: descripciones compartidas.

## Migraciones y estado real

Las migraciones versionadas de evaluación encontradas no reflejan por completo los nombres/modelos actuales del esquema Prisma y no contienen en la búsqueda dirigida la creación de todas las tablas finales (`evaluations`, respuestas y selecciones). No se ejecutó introspección de base. La correspondencia exacta entre esquema declarado y base desplegada queda **Pendiente de verificar**.
