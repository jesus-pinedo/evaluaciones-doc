# PROING — Evaluación de Desempeño HF-011

Fuente central de documentación funcional y técnica del módulo de Evaluación de Desempeño. El requerimiento expresa la intención; el código de `api/` y `sisproing-app/` es la evidencia del comportamiento implementado.

## Línea base auditada

- Fecha: 2026-10-13.
- Backend: `api/`, rama `main`, commit `97fb100193dccd6e630e8145b9ee1d4a6f9e0965`.
- Frontend: `sisproing-app/`, rama `main`, commit `abcb4b30dc05d38413d8d71ce5dae92e8e638614`.
- Documentación: `evaluaciones-docs/`, rama `main`, base `8459bfa9603ed63cb869f2cf4711037fc92307aa`.
- Método: inspección estática delimitada. No se conectó a base de datos, Redis, SMTP ni al sistema legado; no se ejecutaron correos, workers, cron jobs ni migraciones.

## Estado resumido

El código contiene catálogos, periodos con snapshots, asignación evaluador–evaluado, formulario con borrador y finalización, PDF, acuse por enlace, dashboard y recordatorios diarios. Sin embargo, la autenticación del API está desactivada en el montaje actual y no hay autorización por recurso. Esto impide considerar el módulo listo para operación segura y afecta las operaciones que dependen de `req.user`.

## Índice

### Funcional

- [Contexto funcional](docs/funcional/contexto-funcional.md)
- [Flujo de Gestión Humana](docs/funcional/flujo-gestion-humana.md)
- [Flujo del evaluador](docs/funcional/flujo-evaluador.md)
- [Flujo del evaluado](docs/funcional/flujo-evaluado.md)
- [Reglas de negocio y puntuación](docs/funcional/reglas-negocio.md)
- [Notificaciones](docs/funcional/notificaciones.md)

### Técnico

- [Arquitectura](docs/tecnico/arquitectura.md)
- [Backend](docs/tecnico/backend.md)
- [Frontend](docs/tecnico/frontend.md)
- [Modelo de datos](docs/tecnico/modelo-datos.md)
- [Endpoints](docs/tecnico/endpoints.md)
- [Seguridad y SSO](docs/tecnico/seguridad-sso.md)
- [Cron jobs y procesamiento de notificaciones](docs/tecnico/cronjobs-notificaciones.md)

### Auditoría

- [Estado de implementación](docs/auditoria/estado-implementacion.md)
- [Diferencias entre requerimiento y código](docs/auditoria/diferencias-requerimiento.md)
- [Pendientes de validación](docs/auditoria/pendientes-validacion.md)
- [Evidencias inspeccionadas](docs/auditoria/evidencias-inspeccionadas.md)

## Convenciones

- **Requerido:** aparece en el requerimiento original.
- **Implementado:** se confirmó una ruta completa y coherente en el código inspeccionado.
- **Parcial:** existe código, pero faltan partes o hay defectos que limitan el flujo.
- **No encontrado:** no se halló implementación en el alcance inspeccionado; no significa que no exista fuera de él.
- **Pendiente de verificar:** requiere ejecución, configuración, datos o inspección del legado.

Las etiquetas `[REQ]`, `[CÓDIGO]`, `[RIESGO]` y `[POR VALIDAR]` distinguen la fuente de cada afirmación.
