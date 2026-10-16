# P1.4 — Vista previa y snapshots históricos

**Estado:** alcance aprobado para desarrollar; pendiente de implementación y pruebas. **Fecha:** 2026-10-16.

## Validación previa: P1.3

El responsable mostró la pantalla «Mis Evaluaciones» con el mensaje: «No hay un periodo de evaluación vigente para la fecha actual». En la captura no aparece listado de colaboradores ni se reutiliza el periodo histórico 2025 como campaña operativa. **Caso sin periodo vigente: validado visualmente.**

No se consideran verificadas por esa captura la ausencia efectiva de jobs de recordatorios, las fechas límite, el solapamiento de periodos o el acceso a datos en PostgreSQL. Las seis pruebas aisladas con `node:test` fueron reportadas correctas por Codex. El tutorial sigue apareciendo aunque no haya periodo: observación de UX no bloqueante.

## Problema P1.4

La auditoría halló que la vista previa de periodos lee los grupos actuales mediante `groupsService.listAll()` en `sisproing-app/src/modules/employee-evaluations/periods/hooks/usePreviewModal.ts`. Así, un periodo histórico puede mostrar preguntas actuales y no las que fueron guardadas al publicarlo.

## Alcance

1. Revisar exclusivamente snapshots de periodos en `api/src/modules/employee-evaluations/periods/` y modelos relacionados del esquema Prisma, y la vista previa de `sisproing-app/src/modules/employee-evaluations/periods/`.
2. Verificar si hay un endpoint de lectura por `periodId` con grupos y preguntas de snapshot. Reutilizarlo si sirve; de lo contrario, agregar un endpoint **solo de lectura**.
3. Garantizar que al abrir la vista previa de un periodo publicado se consulten **sus propias entidades snapshot**, y nunca catálogos globales actuales.
4. Evitar que una vista previa reutilice datos en caché de otro periodo.
5. Mostrar el histórico en modo solo lectura y sin requerir que esté vigente por fecha.
6. Si faltan snapshots para un periodo antiguo, mostrar un aviso claro. No rellenar silenciosamente con catálogos actuales.
7. Si la vista previa cubre otros catálogos (escalas, fortalezas y debilidades), leer también sus snapshots; no introducir componentes nuevos que no sean necesarios.

## Criterios de aceptación

- La vista previa de 2025 muestra grupos y preguntas originales, según los snapshots de 2025.
- Dos periodos con configuraciones distintas muestran su propia configuración al alternar las vistas.
- Cambios posteriores a catálogos globales no alteran el histórico.
- Las consultas usan `periodId`; no el primer periodo publicado ni el vigente.
- Nunca se actualizan ni regeneran snapshots, evaluaciones, respuestas ni resultados del histórico.
- Si faltan datos históricos se informa, sin sustituirlos por catálogos actuales.

## Exclusiones

No desarrollar aún el selector de reportes de P3, la descarga individual de PDF de P4, puntuaciones de P2, publicación de periodos, ni seguridad general; no modificar módulos ajenos al HF-011.

## Validación y ejecución

Typecheck/build/lint dirigido y checklist manual. Pruebas automatizadas puntuales solo si aportan valor inmediato, sin nueva infraestructura global de testing. No ejecutar publicaciones, migraciones, correos, cron jobs ni cambios sobre registros reales; no crear commits automáticamente.

**Documento base:** [MEJ-001](MEJ-001-historico-creacion-periodos.md).
