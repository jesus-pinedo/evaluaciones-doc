# Estado de implementación

## Matriz funcional

| Área | Estado | Evidencia resumida |
|---|---|---|
| Catálogos de clasificación, grupos, preguntas, fortalezas y debilidades | **Parcial** | CRUD API/UI; borrado físico, auditoría/seguridad problemáticas. |
| Escala de clasificación del resultado | **Parcial** | Lectura y snapshot funcionan en código; mutaciones del API inconsistentes. |
| Catálogo de compromisos | **No encontrado** | Se usa texto libre por debilidad. |
| Creación de periodo borrador | **Implementado** | UI + POST + modelo; no garantiza borrador único. |
| Publicación y snapshots | **Implementado** | Transacción de snapshots, asignaciones y activación. |
| Cierre/reapertura/prórroga | **No encontrado** | Solo estado derivado y `isActive` genérico. |
| Asignación jefe–colaborador | **Implementado** | Snapshot desde `EmployeeManager`, activos `status=1`. |
| Bloqueo de cambio de jefe por pendiente | **No encontrado** | Sin mutaciones de `EmployeeManager` en API; PHP no revisado. |
| Lista del equipo | **Parcial** | Consulta implementada; flujo de identidad roto y resumen con ID codificado. |
| Tutorial | **Parcial** | Modal presente con video demo. |
| Borrador/autoguardado | **Parcial** | UI y servicio; depende de `req.user` ausente. |
| Finalización | **Parcial** | Preguntas completas, score y estado; validaciones/ownership insuficientes. |
| Impresión/PDF | **Implementado en código** | Vista React + Puppeteer; acceso no protegido. |
| Correo al evaluado | **Parcial** | Cola, plantilla, PDF y enlace; payload defectuoso y worker prod no confirmado. |
| Acuse del evaluado | **Implementado en código** | JWT con scope e idempotencia. |
| Recordatorio diario | **Implementado en código** | 08:00 Bogotá, bandera de entorno, BullMQ; fechas/operación por verificar. |
| Aviso de creación/inicio de periodo | **No encontrado** | Sin productor o plantilla. |
| Dashboard de avance | **Implementado** | KPI, evaluador, proyecto y detalle. |
| Consolidados de resultado | **No encontrado** | Sin promedio de score, grupos, clasificaciones o exportación. |
| SSO frontend | **Implementado en código** | Exchange, JWT local y permisos. |
| Autenticación/autorización backend | **No implementada efectivamente** | Middleware global comentado y sin guards por ruta/recurso. |

## Arquitectura confirmada

- Express organiza routers, controladores y servicios por dominio.
- Prisma usa esquema PostgreSQL `evaluation` y relaciones con `gestion_humana`/`public`.
- Los periodos publicados preservan catálogos y asignaciones mediante snapshots.
- React consume DTOs específicos, calcula feedback inmediato y presenta permisos en cliente.
- BullMQ desacopla envío de correos; node-cron produce recordatorios; Puppeteer genera el PDF.

## Riesgos prioritarios

1. Exposición y mutación sin autenticación/autorización backend.
2. Falta de ownership en operaciones por ID.
3. Flujo evaluador no funcional de extremo a extremo con el montaje actual de identidad.
4. Valores de respuesta y pertenencia de snapshots no validados.
5. Worker de producción no evidenciado; correo completado con nombre/puntaje potencialmente incorrectos.
6. Recordatorios fuera de ventana y selección ambigua cuando hay varios periodos publicados.
7. Desacuerdos de contrato (`FINISHED/FINALIZED`, scoring, IDs codificados).
8. Sin pruebas automatizadas específicas del módulo.

## Alcance de evidencia

Se inspeccionaron archivos del dominio bajo:

- `api/src/modules/employee-evaluations/`;
- archivos de notificaciones referenciados directamente por evaluaciones;
- `api/src/modules/auth/`, middleware, montaje de rutas y fragmentos Prisma relacionados;
- `sisproing-app/src/modules/employee-evaluations/`;
- auth, rutas, HTTP y menú directamente usados por el módulo.

No se analizaron módulos de inventario, ventas, compras, facturación, transportes, operaciones u otros dominios. Tampoco se auditó el repositorio PHP.
