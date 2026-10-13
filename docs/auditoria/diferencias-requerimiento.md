# Diferencias entre requerimiento e implementación

| Tema | Requerido / esperado | Encontrado en código | Clasificación |
|---|---|---|---|
| Clasificaciones de debilidad | Múltiples, incluso más de dos. | Una FK opcional por debilidad. | Diferencia funcional |
| Compromisos | Catálogo configurable asociado a debilidades; selección múltiple. | Texto libre por cada debilidad seleccionada. | Diferencia funcional |
| Aviso al crear periodo | Recordar a GH revisar HF-002. | No encontrado. | Faltante |
| Aviso al iniciar periodo | Informar a jefes equipo pendiente. | No hay evento de inicio; solo cron diario. | Parcial |
| Recordatorio durante periodo | Diario, con cierre y pendientes. | 08:00 Bogotá; no valida ventana de fechas. | Parcial |
| Bloqueo de cambio de jefe | Bloquear/advertir si hay pendiente. | No encontrado en Node/React. | Faltante / PHP por validar |
| Personal de todos los proyectos | Mostrar equipo completo. | Asigna por jefe sin filtrar proyecto; visualiza un solo proyecto (`take:1`). | Parcial |
| Escala parametrizable | Configuración de puntuación. | Rangos parametrizan rating total; respuesta por pregunta permanece fija 1–5. | Parcial |
| Guardar y enviar | Persistir y cerrar. | Borrador y finalización separados. | Implementado con ampliación |
| Fortalezas/debilidades/compromisos obligatorios | El procedimiento indica selección. | UI los usa para 100 %, backend no los exige. | Regla inconsistente |
| Ponderaciones | No estaban definidas. | Promedio absoluto con máximo 5, sin pesos. | Decisión técnica no aprobada |
| Reporte de seguimiento | Realizadas y pendientes. | Dashboard amplio por estado/evaluador/proyecto. | Implementado |
| Compilado general/proyecto | Resultados consolidados. | Solo avance; sin consolidación de puntajes o clasificaciones. | Parcial/faltante |
| Reimpresión | Evaluación final consultable. | PDF Puppeteer y vista imprimible. | Implementado |
| Confirmación del evaluado | No claramente definida en requisito base. | Acuse firmado, idempotente y visible en reportes. | Extensión implementada |
| Cierre/reapertura | Pendiente de definición. | Sin operaciones formales. | Pendiente |
| Roles/seguridad | Acceso según responsabilidades. | Permisos solo en React; API abierto. | Brecha crítica |
| Estado final de periodo | Cerrado/finalizado esperado. | `FINISHED` calculado, no persistido; UI espera `FINALIZED`. | Inconsistencia |
| Auditoría de usuario | Identificar creador/modificador. | Campos existen, pero identidad API no se establece y UI usa IDs fijos. | Parcial |

## Diferencias internas entre frontend y backend

- El frontend exige fortaleza y debilidad con compromiso para llegar a 100 %; el backend solo exige preguntas.
- El frontend evita editar `COMPLETED`; el backend también, pero no valida propiedad ni fechas.
- El frontend muestra búsqueda por cargo; backend no incluye cargo en el filtro.
- El frontend usa `FINALIZED`; backend devuelve `FINISHED`.
- El frontend calcula avance de tabla por evaluador incluyendo `IN_PROGRESS`; la dona global solo cuenta `COMPLETED`.
- La vista previa de un periodo usa catálogos globales actuales, no snapshots.
- La plantilla dice/en UI se informa expiración de 30 días, pero el token dura 60.

## No confundir

- `accepted` del dashboard es `notificationReceived`; no indica conformidad con la evaluación.
- `isActive` de periodo indica publicado/habilitado, no que la fecha actual esté dentro del rango.
- El botón “Eliminar” de periodo programado solo cambia `isActive=false`; no elimina datos.
- Comentarios que dicen “eliminación lógica” en catálogos no corresponden a la llamada Prisma, que ejecuta `delete`.
