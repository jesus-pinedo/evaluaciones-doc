# Flujo del evaluado

## Notificación

Al finalizar una evaluación, el servicio encola `send-evaluation-pdf`. El procesador genera:

- un PDF de la evaluación mediante Puppeteer;
- un correo basado en `evaluation-completed.mjml`;
- un enlace firmado a `/public/confirm-receipt?token=...`.

El enlace contiene `evaluationId`, correo y scope `ACKNOWLEDGE_RECEIPT`; el código lo firma por 60 días. La plantilla informa resultado, cargo, proyecto, adjunta el PDF y solicita confirmación.

[RIESGO] El payload se construye con `evaluation.employee.firstName`, propiedad no seleccionada/modelada, y con `evaluation.score` del objeto cargado antes del recálculo. El nombre y puntaje del correo pueden llegar vacíos o desactualizados. Esto requiere prueba integrada.

## Confirmación

1. La ruta pública de React no exige sesión.
2. La página envía el token a `POST /api/public/acknowledge/confirm-receipt`.
3. El backend valida firma, expiración y scope.
4. Si la evaluación existe, marca `notificationReceived=true` y fecha/hora.
5. Si ya estaba confirmada, devuelve éxito con la fecha original; la operación es idempotente.

El acuse significa **recepción/lectura declarada**, no aprobación del resultado. En los reportes aparece como `accepted` o “Confirmado”, nomenclatura que puede inducir a interpretarlo como aceptación.

## Acciones disponibles

La única acción del evaluado encontrada es confirmar mediante el enlace. No se encontró portal autenticado para consultar histórico, descargar de nuevo, comentar, aceptar/rechazar el contenido o impugnarlo.

## Después de confirmar

Solo se actualizan los dos campos de acuse. No cambia `EvaluationStatus`, no se dispara otro correo y no existe un estado posterior. El dashboard incrementa el subconjunto de completadas confirmadas.

## Inconsistencia de vigencia

El comentario del backend y el texto de error del frontend hablan de 30 días, pero `generateAcknowledgmentLink` configura `expiresIn: '60d'`. La vigencia efectiva del código es 60 días.

Evidencia: `api/src/modules/notifications/queues/workers/processors/evaluation.processor.js`, `services/token.service.js`, `evaluations.service.js#acknowledgeReceipt`, `sisproing-app/src/modules/employee-evaluations/evaluations/pages/EvaluationAcknowledgement.tsx`.
