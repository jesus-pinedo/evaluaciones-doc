# Frontend

## Rutas

| Ruta | Pantalla | Protección declarada |
|---|---|---|
| `/employee-evaluations/evaluations` | Equipo del evaluador | `evaluations#view` |
| `/employee-evaluations/evaluations/:evaluationId` | Formulario / consulta | Hereda `evaluations#view` |
| `/employee-evaluations/configurations/*` | Catálogos y borrador | `evaluations#view` + `configurations#view` |
| `/employee-evaluations/periods` | Lista/edición de periodos | `evaluations#view` + `configurations#edit` |
| `/employee-evaluations/reports` | Dashboard | `evaluations#view` + `reports#view` |
| `/employee-evaluations/evaluations/:evaluationId/print` | Vista imprimible | Sin `ProtectedRoute` |
| `/public/confirm-receipt` | Acuse del evaluado | Pública por diseño |

El `ProtectedRoute` padre envuelve todo el módulo y exige `evaluations#view`, incluso para configuración y reportes. Guardar y finalizar no exigen `create/edit` en la ruta o en los botones. Las páginas de catálogos tampoco distinguen `create/edit/delete`.

## Pantallas y comportamiento

### Configuración

`ConfigurationLayout.tsx` mantiene el borrador, navega entre catálogos y permite guardar/publicar. La escala es de solo lectura. Las demás páginas implementan edición inline y borrado con confirmación nativa.

### Periodos

`PeriodsPage.tsx` consulta todos y filtra localmente. Permite editar fechas de programados/activos y “eliminar” programados mediante `isActive=false`. Hay desacuerdo `FINALIZED` (frontend) vs `FINISHED` (API).

La vista previa usa catálogos globales actuales (`groupsService.listAll`), no los snapshots del periodo seleccionado; por tanto, no representa necesariamente el formulario histórico.

### Evaluador

`EvaluationsPage.tsx` ofrece tutorial, resumen, búsqueda por nombre/proyecto, filtros y paginación. El placeholder anuncia búsqueda por cargo, pero el backend no filtra por cargo.

`useEvaluationForm.ts` usa React Hook Form, observa respuestas/selecciones, calcula avance y score en vivo y autoguarda a los dos segundos. `COMPLETED` activa modo lectura.

La escala por pregunta está codificada localmente en `QuestionSectionCard.tsx`; no usa `formConfig.scales`.

### PDF

`EvaluationPrintPage.tsx` acepta estado inyectado por Puppeteer o consulta el API. Presenta datos del evaluado, preguntas, fortalezas, oportunidades/compromisos, escala y trazabilidad de evaluador/acuse.

La ruta no tiene `ProtectedRoute`; si no hay estado inyectado, intenta leer la evaluación por ID. El backend tampoco autoriza ese recurso.

### Reportes

El dashboard muestra KPI, gráficos y tabla. Los filtros de proyecto/centro de costo están deshabilitados. El botón para notificar a todos no ejecuta acción. No hay exportación.

## Sesión y HTTP

`AuthProvider.tsx` intercambia `sso_token` o `code`, persiste JWT/usuario/vencimiento en `localStorage`, deriva permisos y redirige al legado al cerrar/no encontrar sesión. `httpClient.ts` adjunta el JWT.

El TTL local por defecto es 8 horas; el JWT del backend usa 12 horas. La divergencia puede cerrar la sesión del frontend antes de la expiración del token.

## Pruebas

No se encontraron pruebas `test/spec` específicas del módulo en `sisproing-app/`.
