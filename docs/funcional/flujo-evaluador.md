# Flujo del evaluador

## Acceso y equipo

El frontend protege el árbol del módulo con `employee_evaluations.evaluations#view`. La pantalla “Mis Evaluaciones” consulta el periodo publicado más reciente y presenta fechas, días restantes, tutorial, avance y equipo paginado.

La asignación se congela al publicar el periodo desde `EmployeeManager`; solo entran relaciones con `managerId` no nulo y con jefe y colaborador `status=1`. Proyecto y centro de costo se usan como datos de visualización, no como criterio de elegibilidad.

[RIESGO] `EvaluationsPage.tsx` usa un identificador numérico de evaluador fijo para el resumen, pero no lo envía al listar el equipo. El backend acepta `evaluatorId` por query cuando no hay `req.user`; como el middleware está desactivado, la lista responde 400 en la configuración inspeccionada. Este flujo es **Parcial**.

## Diligenciamiento

1. El evaluador abre una tarjeta de colaborador.
2. El detalle carga snapshots del periodo y datos guardados.
3. Cada pregunta se responde con una escala visual fija: 1 Muy Bajo, 2 Bajo, 3 Moderado, 4 Alto y 5 Muy Alto.
4. Se seleccionan cero o más fortalezas.
5. Se seleccionan cero o más oportunidades de mejora; cada una abre un campo de compromiso libre.
6. Cada cambio dispara autoguardado con `debounce` de dos segundos. También existe **Guardar Borrador**.
7. El avance visual considera todas las preguntas, al menos una fortaleza y al menos una debilidad con algún compromiso no vacío.

El backend del borrador reemplaza todas las fortalezas y debilidades, hace `upsert` de las respuestas incluidas, recalcula el puntaje y cambia `PENDING` a `IN_PROGRESS`. No elimina respuestas omitidas del payload.

## Finalización

La UI habilita **Finalizar y Enviar** al llegar a 100 % de su cálculo de avance. El backend, en cambio, solo valida que exista una respuesta por cada pregunta del periodo. No valida:

- valores dentro de 1–5;
- que pregunta, fortaleza o debilidad pertenezcan al mismo periodo;
- al menos una fortaleza;
- al menos una debilidad;
- compromiso no vacío;
- vigencia por fechas del periodo;
- que el usuario sea el evaluador asignado.

Tras finalizar, el estado pasa a `COMPLETED`, se fija `evaluatedAt`, se calcula el resultado y se intenta encolar el correo al evaluado.

## Edición y consulta posterior

- `COMPLETED` se muestra en modo solo lectura y habilita **Imprimir**.
- El backend rechaza nuevos borradores o una segunda finalización cuando el estado ya es `COMPLETED`.
- No se encontró reapertura, corrección administrativa ni control de versión de una evaluación final.
- El detalle, la impresión y el PDF no validan pertenencia del registro al usuario.

## Tutorial

Existe un modal con video de YouTube. El identificador es una constante de demostración (`W92uuRynack`) con comentario para reemplazarlo; su contenido funcional real queda **Pendiente de verificar**.

Evidencia principal: `EvaluationsPage.tsx`, `EvaluationsForm.tsx`, `useEvaluationForm.ts`, `QuestionSectionCard.tsx`, `evaluations.controller.js`, `evaluations.service.js` y `evaluations.helper.js`.
