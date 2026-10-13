# Flujo de Gestión Humana

## 1. Configuración

La ruta React `/employee-evaluations/configurations/*` exige visualmente tanto el permiso padre `employee_evaluations.evaluations#view` como `employee_evaluations.configurations#view`, y agrupa las pestañas Periodo, Criterios, Debilidades, Fortalezas, Grupos de Preguntas, Preguntas y Evaluación. Esta dependencia adicional puede negar acceso a un configurador que no tenga también permiso de evaluaciones.

| Capacidad | Comportamiento encontrado | Estado |
|---|---|---|
| Clasificaciones/criterios | Listar, crear, editar y eliminar. | **Parcial:** CRUD presente; eliminación física y sin autorización backend. |
| Grupos | CRUD y consulta con preguntas. | **Parcial:** el servicio dice eliminación lógica, pero ejecuta `delete`. |
| Preguntas | CRUD, asociación obligatoria a un grupo en la UI. | **Parcial:** en base y API `groupId` puede ser nulo. |
| Fortalezas | CRUD y selección múltiple posterior. | **Parcial:** eliminación física. |
| Debilidades | CRUD y selección de una clasificación en la UI. | **Parcial:** no soporta varias clasificaciones ni catálogo de compromisos. |
| Escala | Pantalla de consulta de rangos. | **Parcial:** solo lectura en frontend; mutaciones del API presentan contratos inconsistentes. |

Las pantallas envían identificadores numéricos de usuario codificados en varios handlers. Los controladores del API, en general, prefieren `req.user.id`; como la autenticación está desactivada, la auditoría no confirma que las altas de catálogos con `createdBy` obligatorio funcionen. Los valores concretos se omiten por protección de datos.

Evidencia: `sisproing-app/src/modules/employee-evaluations/layouts/ConfigurationLayout.tsx`, páginas bajo `configurations/` y servicios homólogos de `api/src/modules/employee-evaluations/configurations/`.

## 2. Creación y publicación del periodo

1. La pantalla captura nombre, inicio y fin.
2. **Guardar** crea `EvaluationPeriod` con `isActive=false` o actualiza el borrador devuelto por `GET /periods/draft`.
3. El backend exige campos, fechas válidas y `endDate >= startDate`.
4. **Publicar** exige que existan grupos, preguntas, fortalezas, debilidades y escalas activas. No exige clasificaciones activas aunque el mensaje de error menciona “criterios”.
5. Dentro de una transacción se copian snapshots, se crean evaluaciones desde las relaciones jefe–empleado activas y se marca el periodo `isActive=true`.

La publicación no espera a la fecha inicial: el periodo queda publicado y su estado de presentación se calcula como `SCHEDULED`, `ACTIVE` o `FINISHED` según la fecha actual.

## 3. Estados y administración

- `DRAFT`: `isActive=false`.
- `SCHEDULED`: publicado y todavía no inicia.
- `ACTIVE`: publicado y fecha actual dentro del rango.
- `FINISHED`: publicado y fecha final superada.

No hay endpoints dedicados para cerrar o reabrir. El `PUT /periods/:id` acepta `isActive`, por lo que puede desactivar genéricamente; la UI solo lo usa como “eliminar” un periodo programado. Desactivarlo lo vuelve a clasificar como borrador y puede provocar múltiples borradores. No se borran snapshots ni asignaciones.

[RIESGO] El frontend espera `FINALIZED`, mientras el backend retorna `FINISHED`. La pestaña “Finalizados” no coincide con el valor del API.

## 4. Seguimiento

El dashboard muestra:

- total, completadas, en progreso, pendientes y confirmadas;
- dona global de avance;
- porcentaje de completadas por proyecto del evaluado;
- tabla por evaluador con detalle por estado y confirmación.

Los filtros de proyecto y centro de costo están visibles pero deshabilitados. El botón “Notificar a todos” no tiene handler. El detalle por evaluador solicita hasta 100 registros y no ofrece paginación adicional.

## 5. Facultades no encontradas

- cierre formal, reapertura o prórroga;
- reasignación de evaluador/evaluado después de publicar;
- bloqueo comprobado del cambio de jefe en HF-002;
- anulación de una evaluación individual;
- consolidado de puntajes, grupos, fortalezas, debilidades o clasificaciones;
- exportación CSV/Excel;
- reenvío manual de notificación.

Esos puntos son **No encontrados** dentro del alcance revisado, no afirmaciones sobre el legado PHP.
