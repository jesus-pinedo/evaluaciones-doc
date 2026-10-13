# PROING — Documentación del módulo de Evaluación de Desempeño (HF-011)

Repositorio de documentación funcional, técnica y de evolución del módulo de Evaluación de Desempeño de PROING.

## Por dónde empezar

1. [`docs/funcional/contexto-funcional.md`](docs/funcional/contexto-funcional.md): objetivo, actores, procesos, reglas y alcance funcional.
2. [`docs/funcional/pendientes-de-validacion.md`](docs/funcional/pendientes-de-validacion.md): decisiones y ambigüedades por confirmar con Gestión Humana o en el código.
3. [`AGENTS.md`](AGENTS.md): instrucciones de trabajo para agentes de código.

## Estructura propuesta

```text
.
├── AGENTS.md
├── README.md
└── docs/
    ├── funcional/
    │   ├── contexto-funcional.md
    │   └── pendientes-de-validacion.md
    ├── tecnico/       # Arquitectura, API, datos, integración, despliegue
    ├── decisiones/    # ADR y decisiones funcionales/técnicas
    └── mejoras/       # Propuestas, alcance, criterios de aceptación
```

## Convenciones de evidencia

- **[REQ]** Requerimiento original: `1. Requerimiento de Módulo de Evaluación de Desempeño (1).pdf`.
- **[PLAN]** Plan de implementación: `tareas.md`, y de esfuerzo: `estimaciones.md`.
- **[HIST]** Información compartida en conversaciones anteriores del proyecto; debe validarse contra el código y la base de datos actuales.
- **[POR VALIDAR]** Aspecto que no se puede dar por confirmado con las fuentes disponibles.

> Una funcionalidad descrita como requisito o como tarea planeada **no implica que esté implementada**. La documentación del comportamiento actual debe actualizarse luego de revisar los repositorios `api`, `sisproing-app` y `sisproing`.
