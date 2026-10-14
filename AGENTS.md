# Instrucciones para agentes — Evaluaciones HF-011

## Propósito y fuente de verdad

HF-011 administra la evaluación anual de desempeño: Gestión Humana configura y publica periodos; los jefes evalúan a sus colaboradores; el sistema conserva respuestas y resultados, notifica y ofrece seguimiento.

`evaluaciones-docs/` es la fuente central de documentación. El requerimiento documenta lo deseado; para afirmar cómo funciona la versión actual, el código es la evidencia principal. No presentar una tarea planeada o una pantalla aislada como funcionalidad operativa.

## Repositorios

- `api/`: Express, Prisma/PostgreSQL, BullMQ/Redis, correo, PDF y scheduler.
- `sisproing-app/`: React, rutas, formularios, reportes y SSO del módulo.
- `evaluaciones-docs/`: documentación funcional, técnica, decisiones y mejoras.
- `sisproing/` (PHP legado): integración externa, fuera del alcance de la auditoría actual.

## Alcance permitido

Inspeccionar solo evaluaciones de desempeño, periodos, catálogos, preguntas/respuestas, asignaciones, cálculo, estados, confirmaciones, reportes, notificaciones/cron de evaluaciones y autenticación/autorización usada por el módulo. Un servicio compartido se inspecciona solo en las funciones llamadas por evaluaciones.

No auditar inventario, ventas, compras, facturación, transportes, operaciones ni otros dominios. No inspeccionar `.env`, secretos, credenciales ni datos personales reales. No inspeccionar PHP sin una nueva autorización expresa.

## Ruta de lectura

1. Leer [contexto funcional](docs/funcional/contexto-funcional.md) y [reglas de negocio](docs/funcional/reglas-negocio.md).
2. Consultar [estado de implementación](docs/auditoria/estado-implementacion.md) y [pendientes](docs/auditoria/pendientes-validacion.md).
3. Para contratos, usar [endpoints](docs/tecnico/endpoints.md), [modelo de datos](docs/tecnico/modelo-datos.md) y [seguridad/SSO](docs/tecnico/seguridad-sso.md).

## Reglas principales confirmadas

- Publicar un periodo copia catálogos activos y crea las evaluaciones desde `EmployeeManager` para jefe y colaborador activos (`status = 1`).
- Una evaluación es única por periodo, evaluador y evaluado.
- Estados de evaluación: `PENDING`, `IN_PROGRESS`, `COMPLETED`.
- El puntaje es `(suma de respuestas / (número de preguntas del periodo × 5)) × 100`, con un decimal. No hay ponderación por grupo.
- Finalizar exige respuesta para todas las preguntas, pero el backend no exige fortalezas, debilidades ni compromisos.
- Los compromisos actuales son texto libre dentro de cada debilidad seleccionada; no existe catálogo de compromisos.
- `isActive` del periodo representa publicación, mientras `DRAFT`, `SCHEDULED`, `ACTIVE` y `FINISHED` se derivan en tiempo de consulta.
- El middleware JWT está desactivado en `api/src/app.js`; las restricciones visuales del frontend no equivalen a autorización del backend.

## Cómo documentar cambios

Separar siempre:

- requerimiento original;
- diseño propuesto;
- comportamiento verificado en código;
- decisión o validación pendiente.

Para cada afirmación técnica indicar repositorio, ruta y función/modelo. Usar los estados **Requerido**, **Implementado**, **Parcial**, **No encontrado** y **Pendiente de verificar**. No incluir datos personales ni grandes bloques de código.

Las mejoras futuras van en `docs/mejoras/MEJ-NNN-descripcion.md` e incluyen: problema, evidencia actual, comportamiento deseado, reglas, criterios de aceptación, cambios por repositorio, pruebas, riesgos y decisiones pendientes. Las decisiones transversales van en `docs/decisiones/`.

## Restricciones de modificación

En una auditoría documental, modificar solo `evaluaciones-docs/`. No cambiar API, frontend, esquema, migraciones, configuración, colas o tareas programadas; no enviar correos ni generar commits automáticamente.
