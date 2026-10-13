# Pendientes de validación

## Operación e infraestructura

1. Confirmar que el esquema PostgreSQL desplegado coincide con `prisma/modules/evaluations.prisma`; las migraciones versionadas no cubren claramente el modelo final.
2. Confirmar si existe un proceso productivo independiente para `main.worker.js`.
3. Verificar valores efectivos de `ENABLE_EMAIL_CRONS`, zona horaria del host, Redis, SMTP, `APP_BASE_URL` y secretos JWT sin documentar secretos.
4. Ejecutar en ambiente seguro pruebas integradas de publicación, borrador, finalización, PDF, correo, reintento y acuse.
5. Determinar cómo se monitorean jobs agotados y correos no entregados.

## Negocio

1. Aprobar o corregir la fórmula absoluta sin ponderaciones y el redondeo a un decimal.
2. Definir si la escala por pregunta debe ser fija 1–5 o parametrizada.
3. Definir si fortalezas, debilidades y todos los compromisos seleccionados son obligatorios al finalizar.
4. Definir rangos de rating, tratamiento de límites, huecos/solapamientos y si se persiste `label` o `description`.
5. Aclarar si una debilidad requiere varias clasificaciones y cómo se consolidan.
6. Resolver catálogo versus texto libre para compromisos.
7. Definir cierre, vencimiento, reapertura, prórroga, anulación y corrección de evaluaciones.
8. Definir qué significa acuse/aceptación y la vigencia correcta del enlace (30 o 60 días).
9. Definir cuál proyecto/centro de costo mostrar cuando un colaborador tiene varios.
10. Definir filtros y métricas del consolidado de resultados.

## SSO, roles y privacidad

1. Inspeccionar, en una fase autorizada, la generación del código SSO en PHP: TTL, un solo uso, `state`, permisos y redirect permitido.
2. Definir matriz oficial de roles/permisos para GH, Gerencia, evaluador, evaluado y solo lectura.
3. Aplicar y probar autenticación en API; decidir excepciones públicas.
4. Definir reglas de acceso por evaluación, periodo, reporte y PDF.
5. Establecer retención, confidencialidad, trazabilidad de lectura/descarga y tratamiento de datos personales.

## Integración de personal

1. Verificar en PHP si HF-002 bloquea cambios de jefe durante pendientes.
2. Definir el comportamiento si jefe o empleado se inactivan después de publicar.
3. Definir traslado de jefe, empleado sin jefe, autoasignación, duplicados y cambios de proyecto.
4. Confirmar si se requiere antigüedad mínima u otros criterios de elegibilidad.

## Defectos que requieren confirmación con prueba

- Altas de catálogos con `createdBy` obligatorio y `req.user` ausente.
- Lista de equipo sin `evaluatorId` de query y middleware desactivado.
- Datos del correo completado (`firstName` y score previo).
- CRUD de scoring.
- pestaña de periodos finalizados (`FINISHED` vs `FINALIZED`).
- búsqueda anunciada por cargo.
- selección del periodo por reportes/cron cuando existen varios publicados.
- preview histórico usando catálogos globales.

## Fuentes no revisadas

- repositorio PHP `sisproing/`;
- requerimiento/anexos originales no presentes en `evaluaciones-docs/` durante esta auditoría;
- datos reales y configuración de ambientes;
- logs de ejecución y evidencias de entrega de correo.
