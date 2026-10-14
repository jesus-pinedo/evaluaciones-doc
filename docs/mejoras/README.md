# Mejoras

Las propuestas futuras se documentan como `MEJ-NNN-descripcion-corta.md` con problema, evidencia, comportamiento deseado, reglas, criterios de aceptación, cambios por repositorio, pruebas, riesgos y decisiones pendientes.

La auditoría establece una línea base; no define todavía un plan general de refactorización.

## Hoja de ruta priorizada (octubre de 2026)

Ver [Hoja de ruta — histórico, puntajes, reportes y PDF individual](ROADMAP-2026-10-periodos-puntajes-reportes-pdf.md).

**Orden de trabajo:** MEJ-001 Histórico y creación de periodos → MEJ-002 Puntajes → MEJ-003 Reportes por periodo → MEJ-004 PDF individual → **MEJ-005 Verificación de reasignación existente (prioridad baja, pendiente)**.

[MEJ-005 — Verificar si el cambio de empleado ya actualiza al evaluador](MEJ-005-reasignacion-manual-evaluador.md).

El plan es documental: ninguna mejora modifica API, React ni notificaciones hasta que se autorice su implementación.

**Nota sobre P5:** antes de diseñar cualquier funcionalidad nueva, verificar si el flujo existente de cambio de jefe/proyecto ya modifica `evaluatorId`, y si selecciona por periodo vigente, `isActive` o estado de evaluación. No asumir que falta la funcionalidad.