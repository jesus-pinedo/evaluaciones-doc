# Instrucciones para Codex — Documentación de Evaluaciones HF-011

## Propósito

Este repositorio documenta el dominio funcional, las decisiones y las mejoras del módulo de Evaluación de Desempeño de PROING. No es un repositorio de código ejecutable.

## Reglas de trabajo

1. Consultar primero `docs/funcional/contexto-funcional.md` para interpretar necesidades del módulo.
2. Separar siempre **requerimiento original**, **diseño propuesto**, **comportamiento verificado en código** y **decisión pendiente**.
3. No dar por implementado ningún ítem basándose únicamente en `tareas.md` o `estimaciones.md`.
4. Antes de modificar reglas de negocio (estados, fechas, notificaciones, cómputo, permisos), identificar los consumidores e impactos en API, frontend y legado PHP.
5. Para describir comportamiento existente, aportar ruta de archivo, función o consulta del repositorio inspeccionado; si no se inspeccionó, etiquetar `[POR VALIDAR]`.
6. Para cada mejora documentar: problema, comportamiento actual comprobado, comportamiento deseado, reglas, criterios de aceptación, cambios por repositorio, pruebas y decisiones pendientes.
7. Mantener documentación en español. Usar términos funcionales de Gestión Humana y nombres técnicos exactos cuando sean verificables.
8. No agregar datos personales de colaboradores, credenciales, tokens, correos reales ni información de evaluaciones individuales.

## Ubicación sugerida de futuras mejoras

`docs/mejoras/MEJ-001-descripcion-corta.md`, `MEJ-002-...`, etc.
