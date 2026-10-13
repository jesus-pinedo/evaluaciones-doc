# Notificaciones de evaluaciones

## Inventario

| Evento | Destinatario | Implementación | Estado |
|---|---|---|---|
| Creación de periodo / revisar HF-002 | Gestión Humana | No se encontró productor, plantilla ni llamada. | **No encontrado** |
| Inicio de periodo | Evaluadores | No se encontró evento específico. El recordatorio puede operar diariamente si el periodo está publicado. | **No encontrado / sustituido parcialmente** |
| Recordatorio diario | Evaluadores con `PENDING` o `IN_PROGRESS` | Cron → consulta → BullMQ → MJML → SMTP. | **Implementado en código; operación por verificar** |
| Evaluación finalizada | Evaluado | Finalización → BullMQ → PDF + enlace → SMTP. | **Implementado en código; datos y worker por verificar** |
| Acuse confirmado | Gestión Humana/evaluador | Solo actualiza base; no se halló notificación. | **No encontrado** |

## Recordatorio diario

`reportsService.getUsersForDailyReminder` selecciona empleados activos que sean evaluadores con alguna evaluación activa en `PENDING` o `IN_PROGRESS`. Usa correo institucional y, como fallback, correo del perfil. Sin correo, omite el registro.

Variables de `evaluation-reminder.mjml`:

- `evaluatorName`;
- `periodName`;
- `pendingCount`;
- `deadlineDate`;
- `daysLeft`;
- `actionUrl`.

La consulta no comprueba `startDate <= hoy <= endDate`; solo `EvaluationPeriod.isActive=true`. Puede recordar antes del inicio o después del vencimiento, mostrando cero días.

## Evaluación finalizada

Variables de `evaluation-completed.mjml`:

- `employeeName`;
- `period`;
- `position`;
- `project`;
- `score`;
- `portalUrl` generado por el worker.

Adjunta `Evaluacion.pdf`. El enlace confirma recepción y lectura; no acepta ni aprueba el resultado.

[RIESGO] `employeeName` se alimenta desde una propiedad inexistente (`employee.firstName`) y `score` desde una instancia no actualizada después de calcular. Debe comprobarse el correo renderizado antes de uso productivo.

## Entrega, errores y duplicados

- Cola BullMQ: `notifications`.
- Correo completado: 3 intentos, backoff exponencial desde 2 segundos.
- Recordatorio: 3 intentos, backoff exponencial desde 5 segundos; se elimina al completar.
- Worker: concurrencia 5; registra eventos de éxito/fallo.
- No hay `jobId` determinista, clave de idempotencia ni tabla de envíos. Repetir el cron o la finalización por condiciones de carrera puede encolar duplicados.
- `acknowledgeReceipt` sí es idempotente.
- La finalización encola dentro de la transacción de Prisma. La cola es externa a PostgreSQL: no existe patrón outbox que garantice atomicidad entre cambio de estado y job.
- El mailer configura BCC por defecto e intercepta destinatarios fuera de producción mediante un helper compartido; los valores efectivos no se inspeccionaron.

Detalles de horario y arranque: [cron jobs](../tecnico/cronjobs-notificaciones.md).
