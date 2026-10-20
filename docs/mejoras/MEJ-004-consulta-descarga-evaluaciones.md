# MEJ-004 / P4 — Consulta y descarga individual de evaluaciones HF-011

**Proyecto:** PROING — Módulo de Evaluación de Desempeño
**Fecha de aprobación funcional:** 2026-10-20
**Estado:** Diseño funcional aprobado; implementación y pruebas pendientes.
**Origen:** Revisión visual del dashboard «Seguimiento por Evaluador» y su modal: localizar la evaluación de un empleado requiere conocer su evaluador, lo que dificulta la búsqueda y descarga individual.

## 1. Decisión aprobada

Crear una **segunda vista dentro de Reportes**, llamada **«Consulta de Evaluaciones»**, independiente de la presentación agrupada por evaluador. Conservar **«Seguimiento»** (dashboard actual) sin cambios funcionales. El usuario autorizado podrá buscar una evaluación por empleado, independientemente de quién la evaluó, y descargar su PDF cuando esté `COMPLETED`.

Flujo: **Reporte → Consulta de Evaluaciones → seleccionar periodo → buscar empleado → descargar PDF de esa evaluación**.

El botón «Reenviar» presente en el dashboard actual fue colocado para un futuro flujo de correo/acuse, pero no tiene funcionalidad; **no reutilizarlo como descarga** ni implementar envíos en P4. Mantenerlo sin cambios; cualquier renombrado o habilitación requiere un alcance distinto.

## 2. Alcance UI aprobado

- Mostrar dos vistas o pestañas claramente diferenciadas bajo Reportes: **Seguimiento** (existente) y **Consulta de Evaluaciones** (nueva). No crear un módulo independiente de navegación principal ni rediseñar el dashboard.
- Nueva consulta con: selector de **periodo publicado** (históricos incluidos), búsqueda por **nombre/apellido del empleado o identificación** (si ese dato está disponible y autorizado), filtro de **estado de evaluación**, y tabla paginada.
- Tabla orientada a cada evaluación individual, con al menos: empleado, evaluador, estado de evaluación, estado de acuse, periodo consultado (si no está claro en encabezado) y acción **«Descargar PDF»**. Otros campos solo si ya están disponibles en DTO o su incorporación mínima resulta justificada.
- Distinguir explícitamente **`COMPLETED`** de **acuse confirmado/pendiente**: una evaluación completada puede tener acuse pendiente; el botón PDF depende de completada, no de acuse confirmado.
- Descargar exclusivamente el PDF de la evaluación seleccionada por su **evaluationId**; no deducirlo del evaluador ni del periodo vigente implícito.
- Estados de carga, error y sin resultados; paginación en servidor; limpiar datos y descartar respuestas tardías al cambiar de periodo, filtro, búsqueda o página.
- Selección inicial de periodo **consistente con P3**: publicado vigente por fecha Bogotá; si no existe, publicado más reciente. Si la navegación conserva el `periodId` de Seguimiento, validar que sea publicado y no asumir permisos implícitos.

## 3. Recursos existentes a verificar antes de programar

Documentados, pero **no equivalen a una verificación de código en este paso**:
- `GET /api/employee-evaluations/reports/evaluations`: consulta paginada con `periodId` (P3), `evaluatorId`, `status`, `search`, `page`, `limit`. Confirmar si **omitir `evaluatorId` permite la búsqueda global por empleado** y si `search` incluye identificación del evaluado; si no, ajustar únicamente lo mínimo en `reports/`.
- `GET /api/employee-evaluations/evaluations/:id/pdf`: generación individual existente con Puppeteer y vista `EvaluationPrintPage.tsx`. Comprobar ruta, contrato, autenticación, autorización, descarga y contenido; **reutilizar**, no crear otro motor PDF.
- `GET /api/employee-evaluations/periods`: fuente actual del selector publicado P3; verificar los permisos y el campo `status` `DRAFT`/etc.

## 4. Control de acceso — precondición obligatoria

La documentación técnica de endpoints (auditoría previa) advierte que la ruta PDF podría **no validar ownership/acceso por evaluación** y que existen dudas de middleware efectivo de autenticación. **Antes de exponer una descarga desde la vista de Gestión Humana, verificar y asegurar autenticación y autorización en el servidor** para la consulta de resultados individuales y la descarga PDF. Ocultar un botón React no constituye autorización.

- Restringir la consulta global y la descarga a roles/permisos reales de consulta de evaluaciones de Gestión Humana; no inventar códigos de rol ni exponer todas las evaluaciones a evaluadores sin facultad.
- Verificar que un usuario sin permisos no pueda consultar ni descargar evaluaciones arbitrarias cambiando `periodId`, `evaluationId` o `evaluatorId`; evitar IDOR.
- Reutilizar el esquema de seguridad ya existente y hacer el menor ajuste imprescindible en rutas/servicios de evaluaciones si la descarga carece del control debido.
- **Si no se puede determinar con certeza el modelo de autorización**, detener la exposición de consulta/descarga y solicitar una decisión técnica explícita. No habilitar un endpoint inseguro.

## 5. Alcance excluido

No modificar P1, P2 ni P3 existente salvo navegación mínima entre vistas. No recalcular score ni actualizar evaluaciones históricas 2025; no tocar respuestas, snapshots, periodos o ponderaciones. No cambiar diseño/composición de PDF ni notificaciones, correo, firma/acuse o «Reenviar». No nuevas migraciones, reportes de desempeño, filtros de proyecto/centro de costo ni refactorización amplia. No crear commits automáticamente en repos locales.

## 6. Implementación por etapas

**P4.1 — Verificación técnica dirigida + contrato concreto** de endpoints, paginación, búsqueda por empleado, autorización y PDF. Producir resultado; si seguridad/contrato ya permiten implementar sin ambigüedad, continuar a P4.2 en el mismo encargo.

**P4.2 — Implementación mínima** de segunda vista dentro de Reportes, búsqueda/paginación/filtros, descarga autenticada para `COMPLETED`, reutilizando PDF existente. Ajustar consulta o permiso backend solo si es imprescindible y está justificado.

**P4.3 — Validación integrada y manual** con datos sintéticos/autorizados; demostrar búsqueda sin evaluador, PDF correcto por periodo/empleado, autorización servidor, histórico intacto y sin efectos secundarios.

## 7. Criterios de aceptación

1. Desde Reportes se puede alternar entre Seguimiento y Consulta de Evaluaciones sin perder el comportamiento del dashboard P3.
2. Un usuario con permisos adecuados busca por empleado sin conocer al evaluador, filtra por periodo, navega resultados paginados y distingue estado de evaluación del acuse.
3. Un registro `COMPLETED` habilita Descargar PDF; `PENDING` o `IN_PROGRESS` no permite documento final, incluso mediante URL directa, conforme a reglas del endpoint.
4. Cada archivo corresponde al `evaluationId`, empleado y periodo seleccionado, incluido histórico 2025; no se mezclan respuestas entre periodos.
5. La descarga no envía correos, no marca acuse ni modifica score, status, respuestas o snapshots.
6. Un acceso no autorizado o manipulación de IDs es rechazado por el servidor; no basta con ocultar el botón.
7. Typecheck/build, pruebas dirigidas de API/frontend y sesión real autorizada o aislada; documentar los obstáculos del entorno sin atribuir pruebas no realizadas.
8. Documentar evidencia de lo cambiado y dejar puntos manuales restantes sin cerrar falsamente.

## 8. Estado de la implementación

**Pendiente de Claude/Codex.** Este documento registra exclusivamente la decisión funcional aprobada y las condiciones técnicas de seguridad; no certifica cambios de código local.
