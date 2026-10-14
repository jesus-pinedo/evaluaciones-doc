# MEJ-005 — Reasignación manual de evaluador durante un periodo

**Proyecto:** PROING — Evaluación de Desempeño HF-011  
**Fecha de registro:** 2026-10-14  
**Prioridad:** P5 — Baja (último bloque de la hoja de ruta, después de P4)  
**Estado:** necesidad funcional registrada; diseño pendiente de microauditoría, aprobación e implementación  
**Origen:** durante la campaña anterior (~mes y medio), personas cambiaron de proyecto/jefe y Gestión Humana solicitó modificar manualmente el evaluador de asignaciones ya generadas.

## 1. Contexto y objetivo

Hoy las evaluaciones se generan como **snapshot de la relación jefe–colaborador al publicar el periodo**, desde `EmployeeManager` / HF-002. Los movimientos de personal posteriores no cambian automáticamente la asignación de la evaluación.

Se desea que un **administrador del módulo de Evaluaciones**, en casos excepcionales y documentados, pueda **reasignar una evaluación existente a otro evaluador** durante un periodo operativo sin reconstruir todas las asignaciones ni cambiar automáticamente las relaciones vigentes en HF-002.

Esto es una mejora **de prioridad baja**. **No forma parte de la implementación de P1** ni autoriza cambios anticipados en API, frontend, Prisma, notificaciones o PHP.

## 2. Evidencia de la auditoría previa

- `docs/funcional/reglas-negocio.md`: la asignación se copia al publicar desde `EmployeeManager`; los cambios posteriores de jefe no actualizan evaluaciones existentes.
- `docs/tecnico/modelo-datos.md`: `Evaluation` almacena `periodId`, `evaluatorId`, `employeeId`, estado y auditoría. Hay una restricción `@@unique([periodId, evaluatorId, employeeId])` en el modelo declarado.
- `docs/tecnico/backend.md`: no se documentó un servicio administrativo de reasignación; `generateEvaluationsForPeriod` crea las asignaciones iniciales.
- `docs/auditoria/estado-implementacion.md`: la reasignación después de publicar no figura como implementada.
- `docs/mejoras/MEJ-001-historico-creacion-periodos.md`: los periodos publicados deben preservar sus snapshots y resultados; la reasignación administrativa, si se aprueba, debe tratarse como **excepción explícita**, no como habilitación para editar libremente el histórico.

**Límite de evidencia:** este registro no constituye una nueva inspección del código. Antes de diseñar el endpoint o la migración, confirmar el modelo y los flujos reales con una microauditoría acotada.

## 3. Escenario de negocio

1. Gestión Humana publica el periodo y el sistema genera la evaluación del colaborador C a cargo del evaluador A.
2. Durante el periodo, C cambia de proyecto, área o jefe, y se solicita que la evaluación pendiente la diligencie el evaluador B.
3. Un administrador autorizado localiza **el periodo**, **la evaluación** y **el evaluador actual**.
4. Selecciona el **nuevo evaluador**, aporta **motivo obligatorio** y confirma la operación.
5. El sistema registra la reasignación sin crear otra evaluación para el mismo colaborador/periodo.
6. La evaluación deja de aparecer en pendientes del evaluador A y pasa a los pendientes de B. El cambio es visible para Gestión Humana.
7. Se determina si procede notificar a A/B, y cómo quedan los recordatorios diarios.

**Importante:** un cambio de proyecto no implica necesariamente cambio de evaluador. La acción debe ser **manual, explícita y puntual**.

## 4. Reglas funcionales propuestas (no aprobadas aún)

| ID | Regla propuesta | Estado |
|---|---|---|
| RN-R01 | Solo un administrador autorizado de Evaluaciones puede reasignar. No basta poder abrir el listado. | Pendiente de aprobación |
| RN-R02 | La reasignación se realiza para una evaluación concreta, identificada por `evaluationId` y periodo. | Pendiente de aprobación |
| RN-R03 | **PENDING:** permitir reasignación tras verificar nuevo evaluador, motivo y ausencia de colisión. | Propuesta preferida para MVP |
| RN-R04 | **IN_PROGRESS:** no transferir automáticamente hasta decidir tratamiento del borrador, autoría y firmas. Opciones: bloquear para primera versión, o transferencia excepcional auditada sin perder respuestas. | Pendiente de decisión |
| RN-R05 | **COMPLETED:** impedir reasignación por defecto; corrección/reapertura sería flujo administrativo distinto, auditado. | Propuesta preferida |
| RN-R06 | No se permite reasignar a la misma persona ni al propio evaluado; validar que el nuevo evaluador es elegible según las reglas que apruebe GH. | Pendiente |
| RN-R07 | Conservar `evaluationId`, `periodId`, `employeeId`, snapshots y resultado histórico que corresponda; **no crear duplicados**. | Pendiente de diseño técnico |
| RN-R08 | Guardar historial **append-only**: evaluador anterior y nuevo, periodo, evaluación, usuario administrador, fecha/hora, motivo, estado anterior y, si corresponde, referencias a notificaciones. | Propuesta requerida para auditoría |
| RN-R09 | No modificar la relación HF-002/EmployeeManager como efecto colateral; ese registro sigue su flujo propio en GH. | Propuesta |
| RN-R10 | Permitir durante la vigencia del periodo; fuera de plazo solo mediante excepción administrativa explícitamente aprobada. | Pendiente |
| RN-R11 | La nueva asignación debe reflejarse en listas, dashboard, detalle y recordatorios; evitar enviar recordatorios futuros al evaluador anterior. | Propuesta |
| RN-R12 | El historial de movimientos debe seguir siendo consultable incluso cuando el periodo finalice. | Propuesta |

### Restricción de unicidad a revisar

La unicidad documentada es `(periodId, evaluatorId, employeeId)`, **no** `(periodId, employeeId)`. Por tanto, pueden coexistir dos filas para el mismo colaborador y periodo si el evaluador difiere. El servicio de reasignación debe validar que no existan asignaciones incompatibles y preservar una sola evaluación lógica por empleado y periodo, si esta regla es aprobada. No cambiar restricciones/migraciones antes de analizar el estado real de datos.

## 5. Alternativas técnicas a comparar cuando llegue P5

**A. Actualizar `evaluatorId` de la evaluación existente dentro de una transacción**, más una tabla de historial de reasignaciones. Pros: preserva `evaluationId`, respuestas y referencias. Contras: se debe preservar claramente quién era el evaluador original y quién creó cada respuesta, sobre todo si el estado es `IN_PROGRESS`.

**B. Cerrar/anular asignación anterior y crear una nueva** con trazabilidad. Pros: separa responsabilidades. Contras: obliga a manejar duplicidad lógica, estados anulados, vistas/reportes y posible traslado de borradores. Para la primera versión parece innecesariamente complejo.

**Preferencia preliminar:** estudiar A para evaluaciones `PENDING`; no tomar la decisión hasta inspeccionar relaciones, consumidores, efectos en PDF y auditoría.

## 6. Ubicación sugerida en la interfaz

Opción preferida por evaluar: **Reporte → Seguimiento por evaluador → Detalle de colaboradores → Acción «Reasignar evaluador»** para un registro `PENDING`, con modal de búsqueda del nuevo evaluador, motivo y confirmación.

Alternativa: pantalla administrativa **Periodos → Asignaciones**. La elección final debe armonizarse con los cambios a reportes de P3.

Solo un perfil autorizado debería ver la acción; el backend debe validar el permiso y la evaluación concreta cuando se implemente. Esto **no** implica iniciar ahora una reforma general de seguridad.

## 7. Revisión técnica futura delimitada

Cuando se alcance P5, auditar exclusivamente:

- `api/src/modules/employee-evaluations/evaluations/evaluations.service.js` y sus controladores/rutas.
- `api/prisma/modules/evaluations.prisma`, y referencias necesarias a `EmployeeManager` / empleados.
- Consultas por evaluador en `api/src/modules/employee-evaluations/reports/reports.service.js`.
- Selector de personal en `sisproing-app/src/modules/employee-evaluations/evaluations/` y detalle del dashboard.
- Productores de recordatorios de evaluación, únicamente para asegurar que el destinatario actualizado reciba los recordatorios.
- PDF y trazabilidad del evaluador únicamente si aparece impacto demostrado.

No inspeccionar otros módulos de negocio y no cambiar PHP legado en esta mejora.

## 8. Criterios de aceptación candidatos

1. En periodo vigente, administrador reasigna evaluación `PENDING` del evaluador A a B con motivo obligatorio.
2. El registro conserva `evaluationId`, periodo y colaborador; no se pierden snapshots.
3. A deja de ver la evaluación entre sus pendientes y B pasa a verla.
4. Totales por evaluador y recordatorios muestran el nuevo responsable sin duplicar el total global.
5. La operación registra quién, cuándo, por qué y los identificadores de ambos evaluadores.
6. El histórico de reasignaciones sigue visible después de vencer el periodo.
7. Reasignación a sí mismo, persona no elegible o asignación duplicada se rechaza con mensaje claro.
8. `IN_PROGRESS` sigue una regla explícita, no se transfiere inadvertidamente ni borra respuestas.
9. `COMPLETED` no puede reasignarse mediante la operación ordinaria.
10. No se alteran las relaciones de empleados en HF-002 ni las evaluaciones de otros periodos.
11. La operación es transaccional; si falla, conserva asignación y auditoría previas.
12. Cambio realizado dentro de la ventana vigente; fuera de ella se respeta la política aprobada.

## 9. Decisiones pendientes para cuando prioricemos P5

1. ¿La primera versión permite solo `PENDING` o también `IN_PROGRESS`?
2. Si hay respuestas de A, ¿continúa B sobre el borrador, se reinicia expresamente o se requiere revisión de GH? ¿Cómo se preserva autoría?
3. ¿Dónde vivirá el control administrativo: reporte o panel de asignaciones?
4. ¿Qué perfiles y criterios habilitan elegir a B?
5. ¿Debemos enviar aviso inmediato al nuevo evaluador y/o al anterior?
6. ¿Se permiten cambios masivos o solo uno por uno? Recomendación inicial: solo individuales.
7. ¿Se permitirá alguna excepción sobre periodo vencido o evaluación completada? Recomendación inicial: fuera del flujo básico.

## 10. Relación con P1

La aprobación de P1 es **un único borrador global y periodos publicados/históricos inalterables por defecto**. P5 **no cambia esa decisión**: una reasignación administrativa puntual, si se aprueba, debe ser una operación explícita y auditada durante un periodo permitido, sin editar snapshots ni reabrir indiscriminadamente periodos finalizados.

**Estado:** documentado en backlog; sin implementación autorizada.
