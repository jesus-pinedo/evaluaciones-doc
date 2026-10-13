# Endpoints del módulo

Base general: `/api`. En el montaje inspeccionado, salvo el diseño lógico de la ruta pública, ninguna ruta aplica efectivamente el middleware de autenticación.

## Periodos

| Método | Ruta | Comportamiento | Estado |
|---|---|---|---|
| GET | `/employee-evaluations/periods` | Lista; filtro opcional `isActive`. | Implementado |
| GET | `/employee-evaluations/periods/draft` | Devuelve el único borrador; 404 si no hay, error si hay varios. | Implementado |
| POST | `/employee-evaluations/periods` | Crea borrador. | Implementado |
| GET | `/employee-evaluations/periods/:id` | Detalle. | Implementado |
| PUT | `/employee-evaluations/periods/:id` | Actualiza nombre/fechas/activo. | Parcial: fechas se requieren de facto. |
| POST | `/employee-evaluations/periods/:id/publish` | Snapshots, asignaciones y publicación. | Implementado |

Los controladores contienen `remove` y `deactivate`, pero no hay rutas registradas para ellos.

## Catálogos

Cada base expone `POST /`, `GET /`, `GET /:id`, `PUT /:id` y `DELETE /:id`:

- `/employee-evaluations/configurations/classifications`
- `/employee-evaluations/configurations/groups`
- `/employee-evaluations/configurations/questions`
- `/employee-evaluations/configurations/strengths`
- `/employee-evaluations/configurations/weaknesses`

Las lecturas son **Implementadas**. Las mutaciones son **Parciales** por ausencia de identidad/autorización y porque `DELETE` es físico.

`/employee-evaluations/configurations/scoring` publica el mismo CRUD. Solo `GET /` y `GET /:id` son coherentes con el modelo. Crear/actualizar/eliminar presentan errores de firma, nombres de campos/modelo y exportación, por lo que son **No operativos según inspección estática**.

## Evaluaciones

| Método | Ruta | Entrada relevante | Comportamiento |
|---|---|---|---|
| GET | `/employee-evaluations/evaluations/my-team/overview` | `evaluatorId` fallback | Periodo publicado más reciente y totales. |
| GET | `/employee-evaluations/evaluations/my-team/evaluations` | `evaluatorId`, `status`, `search`, `page`, `pageSize` | Lista del equipo. |
| GET | `/employee-evaluations/evaluations/:id` | `includeDetails` | Estructura de formulario y datos guardados. |
| PUT | `/employee-evaluations/evaluations/:id/draft` | `{answers,strengths,weaknesses}` | Guarda borrador y score parcial; exige `req.user.id`. |
| POST | `/employee-evaluations/evaluations/:id/finalize` | mismo payload | Completa, puntúa y encola correo; exige `req.user.id`. |
| GET | `/employee-evaluations/evaluations/:id/pdf` | ID | Genera PDF con Puppeteer. |

El orden de definición evita que las rutas `my-team/*` sean absorbidas por `/:id`.

## Reportes

| Método | Ruta | Comportamiento |
|---|---|---|
| GET | `/employee-evaluations/reports/dashboard-progress` | KPI, evaluadores y proyectos del periodo publicado encontrado. |
| GET | `/employee-evaluations/reports/evaluations` | Detalle paginado; filtros `status`, `search`, `evaluatorId`. |

No hay endpoints de consolidados de puntuación, exportación o filtros de proyecto/centro de costo.

## Público y autenticación

| Método | Ruta | Comportamiento |
|---|---|---|
| POST | `/auth/exchange` | Intercambia código Redis por JWT/usuario. |
| POST | `/public/acknowledge/confirm-receipt` | Valida token con scope y registra acuse. |

El frontend contiene un cliente para `/auth/me`, pero no se encontró esa ruta en `api/src/modules/auth/auth.routes.js`.

## Contratos inconsistentes relevantes

- El frontend ofrece búsqueda por cargo; API busca nombre/apellido/proyecto.
- El backend retorna periodo `FINISHED`; el tipo/filtro frontend espera `FINALIZED`.
- El resumen de equipo acepta `evaluatorId` y la UI envía un identificador fijo; la lista no recibe ese fallback desde la UI.
- Los controladores de catálogos ignoran IDs de auditoría enviados por el frontend y toman `req.user.id`.
- La API no valida ownership de `:id` en detalle, borrador, finalización o PDF.
