# Cron jobs y procesamiento de notificaciones

## Scheduler de recordatorios

Archivo: `api/src/modules/employee-evaluations/schedulers/daily-reminder.scheduler.js`.

- Se inicializa al arrancar `src/server.js`.
- Solo agenda si `ENABLE_EMAIL_CRONS === 'true'`.
- Expresión: `0 8 * * *`.
- Zona horaria explícita: `America/Bogota`.
- Frecuencia: todos los días a las 08:00.
- Fuente: `reportsService.getUsersForDailyReminder`.
- Salida: un job `send-evaluation-reminder` por evaluador elegible.

El código contiene un comentario sobre limitar a tres en desarrollo, pero la línea `slice` está comentada; actualmente no limita.

## Selección de pendientes

1. Busca un `EvaluationPeriod` con `isActive=true`, sin orden definido.
2. Selecciona empleados activos con evaluaciones `PENDING` o `IN_PROGRESS` del periodo.
3. Cuenta pendientes por evaluador.
4. Omite evaluadores sin correo.

No valida que hoy esté dentro de las fechas. Si hay varios periodos publicados no solapados, `findFirst` puede escoger uno no determinístico.

## Cola y worker

- Cola Redis/BullMQ: `notifications`.
- Productor: `src/modules/notifications/queues/email.queue.js`.
- Worker: `src/modules/notifications/queues/workers/main.worker.js`.
- Procesadores relevantes: `evaluation.processor.js` y `evaluation-reminder.processor.js`.
- Concurrencia: 5.
- Reintentos: 3 con backoff exponencial.

`src/server.js` importa el worker solo cuando `NODE_ENV === 'development'`. No se encontró script de package ni otro import para iniciar ese worker en producción. La existencia de un proceso separado en infraestructura es **Pendiente de verificar**; el código del servidor productivo por sí solo no consumiría los jobs.

## Fallos y reintentos

Los handlers lanzan error ante fallo de PDF, plantilla o SMTP; BullMQ reintenta. Los listeners solo registran logs. No se encontró:

- dead-letter queue;
- alerta operacional;
- persistencia de estado de entrega;
- reenvío manual;
- deduplicación por destinatario/fecha/evaluación;
- outbox transaccional.

## Diferencia entre definición y operación

Queda confirmado que la tarea está definida y se registra al arrancar el servidor si la bandera está activa. No se verificó que producción tenga la bandera, Redis, worker, SMTP, variables de URL o assets configurados, ni que el cron haya ejecutado exitosamente.
