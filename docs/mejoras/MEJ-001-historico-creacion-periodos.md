# MEJ-001 — Histórico y creación de periodos

**Proyecto:** PROING — Evaluación de Desempeño HF-011  
**Fecha de microauditoría:** 2026-10-14  
**Estado:** microauditoría terminada; **P1.1 implementado y parcialmente validado visualmente**; **P1.2 implementado y flujo funcional principal validado (6/6 comprobaciones manuales del responsable)**, con casos técnicos/concurrencia pendientes; **P1.3 implementado localmente y validado técnicamente, pendiente de checklist manual**; P1.4 pendiente de implementación
**Prioridad:** P1  
**Alcance:** creación, borrador, publicación, estados, histórico y selección de periodos.  

## 1. Resultado ejecutivo

La existencia de un periodo anterior con `isActive=true` **no es la condición que oculta ni impide crear otro periodo en React**. No existe en el frontend una condición del tipo «si hay un periodo activo, ocultar Crear».

La causa observable en la interfaz es una combinación de diseño y estado:

1. La página de listado `/employee-evaluations/periods` **no tiene botón ni enlace para crear**; solo lista, edita, desactiva y abre vista previa. La creación está en otra opción del menú, **Configuración de Períodos**, ruta `/employee-evaluations/configurations/periods`.
2. Esa opción del menú solo se muestra cuando el usuario cumple `can('view', 'employee_evaluations.configurations')`.
3. Al entrar a Configuración, React consulta `GET /periods/draft`. Si encuentra cualquier registro con `isActive=false`, guarda su ID en `draftId`; desde ese momento **Guardar hace `PUT` sobre ese registro**, no `POST` para crear uno nuevo.
4. Si no encuentra un `isActive=false`, el formulario queda vacío y Guardar sí ejecuta `POST`, creando el nuevo borrador.

Por tanto, la condición exacta que cambia creación por edición es **`draftId !== null`**, derivada de que el backend haya encontrado exactamente un periodo con **`isActive=false`**. El periodo histórico con `isActive=true` no llena `draftId`; solo puede bloquear el guardado o la publicación si las fechas del nuevo periodo se traslapan con él.

La forma segura, con el comportamiento actual, es **mantener el periodo histórico publicado (`isActive=true`)**, crear un periodo con un ID nuevo y fechas no traslapadas, y publicarlo para que genere sus propios snapshots y asignaciones. No se debe desactivar, reutilizar ni editar el periodo anterior para abrir espacio al nuevo.

## 2. Alcance y límites de la revisión

Se inspeccionaron exclusivamente:

- `sisproing-app/src/modules/employee-evaluations/periods/`;
- `sisproing-app/src/modules/employee-evaluations/layouts/ConfigurationLayout.tsx`;
- rutas, menú y permisos visuales que llevan a periodos;
- `api/src/modules/employee-evaluations/periods/`;
- generación de snapshots y asignaciones al publicar;
- selección de periodo usada por evaluaciones, reportes y recordatorio diario;
- modelos Prisma directamente relacionados con periodos, snapshots y evaluaciones.

No se modificaron React, API, esquema, base de datos, publicaciones, colas ni notificaciones. No se revisaron otros módulos de negocio. La autenticación habilitada posteriormente se tomó como condición actual y no se reabrió la auditoría general de seguridad.

## 3. Causa exacta y evidencia en código

### 3.1 La lista no expone la acción de crear

**Estado: Implementado, pero incompleto funcionalmente.**

`PeriodsPage` carga todos los periodos y renderiza pestañas y tarjetas, pero no incluye una acción «Crear periodo» ni navegación hacia la configuración:

- React: `sisproing-app/src/modules/employee-evaluations/periods/PeriodsPage.tsx`, líneas 185–260, componente `PeriodsPage`.
- Las únicas acciones por tarjeta son vista previa, edición y eliminación/desactivación: `periods/components/PeriodsItem.tsx`, líneas 79–118.

La creación vive en una ruta distinta:

- React: `sisproing-app/src/modules/employee-evaluations/routes/index.tsx`, líneas 29–45: `/employee-evaluations/configurations/periods` monta `CreatePeriodPage` dentro de `ConfigurationLayout`.
- React: `sisproing-app/src/shared/layauts/Layaut.tsx`, líneas 19–24: la opción `configurations` navega a esa ruta; `periods` navega al listado.

Esto explica que un usuario ubicado en «Periodos» no encuentre cómo crear uno nuevo, aunque técnicamente la pantalla exista.

### 3.2 Condición visual de acceso a la pantalla de creación

**Estado: Implementado.**

La opción **Configuración de Períodos** se filtra por:

`can('view', 'employee_evaluations.configurations')`

Evidencia:

- React: `sisproing-app/src/components/SideMenu.tsx`, líneas 30–45 y 75–82, `DEFAULT_ITEMS` y `visibleItems`.
- React: `sisproing-app/src/modules/employee-evaluations/routes/index.tsx`, líneas 19–38: primero exige vista de evaluaciones y luego vista de configuraciones.

No hay consulta de `isActive` para mostrar u ocultar esa opción.

### 3.3 Condición que decide crear o actualizar

**Estado: Implementado con ambigüedad de negocio.**

Al montar `ConfigurationLayout`:

- llama `periodsService.getDraft()`;
- si recibe un periodo, copia sus datos al formulario y ejecuta `setDraftId(draft.id)`;
- si no recibe ninguno, limpia el formulario y ejecuta `setDraftId(null)`.

Al guardar, la bifurcación exacta es:

- si `draftId` tiene valor: `periodsService.update(draftId, payload)`;
- si `draftId` es nulo: `periodsService.save(payload)`, que delega en `create` y hace `POST`.

Evidencia:

- React: `sisproing-app/src/modules/employee-evaluations/layouts/ConfigurationLayout.tsx`, líneas 51–85 y 95–120, `loadDraft` y `handleSave`.
- React: `sisproing-app/src/modules/employee-evaluations/periods/periods.service.ts`, líneas 47–55, 97–127, `create`, `getDraft`, `save` y `publish`.
- API: `api/src/modules/employee-evaluations/periods/periods.service.js`, líneas 338–373, `findDraftPeriod`: define borrador exclusivamente como `isActive=false`.

**Conclusión:** lo que impide iniciar un registro nuevo desde el formulario no es un periodo anterior `isActive=true`, sino la existencia de un registro `isActive=false` que React adopta automáticamente como el borrador actual.

### 3.4 Problema adicional después de publicar

**Estado: Defecto comprobado por lectura.**

Tras publicar, `ConfigurationLayout` muestra un modal, pero no limpia `draftId` ni el formulario. Si se elige **Cerrar** en vez de navegar a la lista, el componente conserva el ID del periodo que acaba de pasar a `isActive=true`. Un Guardar posterior intenta actualizar ese periodo publicado.

Evidencia: `ConfigurationLayout.tsx`, líneas 122–137 y 226–258. La acción de cierre solo cambia `showSuccessModal` e `isPublishing`; no reinicia `draftId` ni `periodForm`.

## 4. Comportamiento actual verificado

### 4.1 Creación de borrador

**Estado: Parcial.**

`createPeriod` aplica estas reglas:

- nombre, inicio y fin obligatorios;
- fechas parseables;
- fin mayor o igual al inicio;
- ningún traslape, con extremos inclusivos, contra otro periodo `isActive=true`;
- crea el registro con `isActive=false`.

Evidencia: `api/src/modules/employee-evaluations/periods/periods.service.js`, líneas 15–31, 41–68 y 76–107.

Observaciones:

- La validación de traslape se ejecuta incluso al crear un borrador. Si el periodo anterior termina el mismo día en que empieza el nuevo, se considera traslape (`newStart <= oldEnd` y `newEnd >= oldStart`).
- `createPeriod` no consulta si ya existe otro borrador. Se pueden crear varios mediante llamadas directas o clientes concurrentes.
- `findDraftPeriod` presupone uno solo y devuelve error 500 si encuentra más de uno. La UI, ante ese error, deja `draftId=null`, por lo que un nuevo Guardar podría crear otro borrador y agravar el estado.
- El servicio captura `P2002` como «nombre duplicado», pero el modelo `EvaluationPeriod` declarado no tiene `@unique` sobre `name`: `api/prisma/modules/evaluations.prisma`, líneas 261–285. La unicidad efectiva del esquema desplegado queda **Pendiente de verificar**; esta auditoría no consultó la base.

### 4.2 Publicación

**Estado: Implementado con restricciones incompletas.**

`publishPeriod` exige:

1. que el periodo exista;
2. que todavía tenga `isActive=false`;
3. que sus fechas no se traslapen con otro `isActive=true`;
4. que exista al menos un grupo, pregunta, fortaleza, debilidad y escala global activos.

Luego, dentro de una única transacción:

- copia catálogos activos a snapshots del periodo;
- genera evaluaciones desde relaciones `EmployeeManager` cuyo jefe y colaborador estén activos (`status=1`);
- cambia el periodo a `isActive=true`.

Evidencia:

- API: `api/src/modules/employee-evaluations/periods/periods.service.js`, líneas 157–247, `publishPeriod`.
- API: `api/src/modules/employee-evaluations/periods/snapshots/periodSnapshots.service.js`, líneas 21–137, `createSnapshotsForPeriod`.
- API: `api/src/modules/employee-evaluations/evaluations/evaluations.service.js`, líneas 28–91, `generateEvaluationsForPeriod`.

Restricciones que **no** existen hoy:

- no se exige que el periodo sea futuro o esté vigente; puede publicarse uno ya vencido;
- no se exige que se genere al menos una evaluación;
- no se valida un único borrador al crear;
- no se bloquea explícitamente la edición genérica de un periodo ya publicado;
- no existe estado persistido de cierre/finalización;
- clasificación no forma parte del conteo mínimo, aunque el mensaje de error dice «criterios»;
- no se valida que cada pregunta activa pertenezca a un grupo activo antes del snapshot.

### 4.3 Protección frente a duplicación de asignaciones

**Estado: Implementado para la tupla principal; parcial para el flujo completo.**

La base declarada impone una evaluación única por:

`periodId + evaluatorId + employeeId`

Además, `createMany` usa `skipDuplicates=true`. Evidencia:

- `api/prisma/modules/evaluations.prisma`, líneas 288–324, modelo `Evaluation`.
- `api/src/modules/employee-evaluations/evaluations/evaluations.service.js`, líneas 71–81.

Esto evita duplicar la misma asignación dentro del mismo periodo, incluso si `EmployeeManager` trae relaciones repetidas. Que el mismo jefe–colaborador tenga una evaluación en dos periodos distintos es correcto y esperado, porque `periodId` forma parte de la llave.

La publicación secuencial repetida del mismo periodo se rechaza cuando ya está `isActive=true`, y una falla durante snapshots/asignaciones debe revertir la transacción completa. No obstante, no hay bloqueo explícito de fila ni prueba automatizada de dos publicaciones concurrentes. La respuesta exacta ante esa carrera queda **Pendiente de verificar** en una prueba de integración segura; no debe probarse en producción ni enviando notificaciones.

### 4.4 Edición y desactivación

**Estado: Defectuoso.**

El servicio genérico `update` permite recibir `name`, fechas e `isActive` sin distinguir borrador, programado, vigente o vencido. Por tanto, el backend no protege el histórico contra una llamada que cambie fechas, nombre o `isActive` de un periodo publicado.

Además, `update` intenta parsear siempre `startDate` y `endDate`, aunque no vengan en el payload. La acción de UI «Eliminar» para un programado envía solamente `{ isActive:false, updatedBy }`; por lectura, esa llamada falla al validar fechas ausentes antes de actualizar.

Evidencia:

- API: `api/src/modules/employee-evaluations/periods/periods.service.js`, líneas 426–545, `update`.
- React: `sisproing-app/src/modules/employee-evaluations/periods/PeriodsPage.tsx`, líneas 140–173, `handleSoftDelete`.

Si se llegara a desactivar un periodo publicado, no se convertiría en histórico cerrado: pasaría a ser considerado `DRAFT` por toda la lógica actual, aun si conserva snapshots, evaluaciones y resultados.

## 5. Estados y tratamiento de vencidos

### 5.1 Estados reales del backend

**Estado: Implementado como cálculo, no como ciclo persistido.**

El backend deriva el estado en cada lectura:

| Condición | Estado API |
|---|---|
| `isActive=false` | `DRAFT` |
| publicado y fecha actual anterior al inicio | `SCHEDULED` |
| publicado y fecha actual posterior al fin | `FINISHED` |
| publicado dentro del rango | `ACTIVE` |

Evidencia: `api/src/modules/employee-evaluations/periods/periods.service.js`, líneas 400–423, `getPeriodStatus` y `mapPeriodToDto`.

`isActive=true` significa, en la práctica, **publicado**. Un vencido no cambia automáticamente la bandera a `false` y no existe proceso de cierre persistido.

### 5.2 Inconsistencia `FINISHED` / `FINALIZED`

**Estado: Defecto comprobado.**

React declara y filtra `FINALIZED`, pero la API retorna `FINISHED`:

- React: `sisproing-app/src/modules/employee-evaluations/periods/periods.service.ts`, líneas 7–18, tipo `PeriodDto`.
- React: `sisproing-app/src/modules/employee-evaluations/periods/PeriodsPage.tsx`, líneas 15–20, 27–32 y 73–76.
- React: `sisproing-app/src/modules/employee-evaluations/periods/components/PeriodsItem.tsx`, líneas 22–31.
- API: `api/src/modules/employee-evaluations/periods/periods.service.js`, línea 408.

Efectos:

- la pestaña **Finalizados** no encuentra los periodos `FINISHED`;
- la pestaña inicial es **Activos**, por lo que un periodo vencido tampoco aparece al entrar;
- solo se descubre en **Todos**;
- la tarjeta usa un `else` genérico y muestra «Finalizado» tanto para `FINISHED` como para `DRAFT`, ocultando la diferencia;
- `DRAFT` tampoco forma parte del tipo ni tiene pestaña propia.

### 5.3 Límite de fecha

**Estado: Riesgo identificado por lectura; límite exacto pendiente de prueba.**

`getPeriodStatus` compara el instante actual (`new Date()`) contra fechas de base `@db.Date`, sin normalizar ambos lados como fechas de negocio en `America/Bogota`. En consecuencia, el periodo puede pasar a `FINISHED` durante su propia fecha final, en vez de al terminar el día de negocio. Debe unificarse la semántica de fecha inclusiva y probar el límite exacto contra PostgreSQL/Prisma antes de implementar.

## 6. Riesgo de seleccionar el periodo incorrecto

**Estado: Riesgo alto comprobado.**

Existen tres comportamientos distintos:

1. `getActivePeriodOrThrow` filtra `isActive=true` y elige el de `startDate` más reciente, sin validar que hoy esté dentro del rango. Un periodo futuro recién publicado desplaza inmediatamente al anterior en «Mi equipo».
   - `api/src/modules/employee-evaluations/periods/periods.service.js`, líneas 375–397 y 587–608.
   - `api/src/modules/employee-evaluations/evaluations/evaluations.controller.js`, líneas 156–193.
2. Dashboard, detalle de reportes y recordatorio diario usan `findFirst({isActive:true})` sin `orderBy` ni ventana de fechas. Si hay varios publicados no traslapados, la selección no expresa cuál debe ganar.
   - `api/src/modules/employee-evaluations/reports/reports.service.js`, líneas 13–40, 208–247 y 320–334.
3. El scheduler ejecuta a las 08:00 y delega en esa selección; no comprueba `startDate <= hoy <= endDate`. Puede preparar recordatorios de un periodo futuro o vencido.
   - `api/src/modules/employee-evaluations/schedulers/daily-reminder.scheduler.js`, líneas 10–53.

Publicar un nuevo periodo futuro, aun sin tocar el histórico, puede por tanto cambiar de inmediato qué periodo muestran evaluaciones/reportes o cuál usa el cron. Esta es la principal razón para no limitar la corrección a un botón «Nuevo».

## 7. Histórico y snapshots

### 7.1 Lo que sí preserva el histórico

**Estado: Implementado.**

Cada publicación crea grupos, preguntas, fortalezas, debilidades, clasificaciones y escala propios del periodo, además de asignaciones con su `periodId`. Las respuestas y resultados referencian esas entidades del periodo. Crear y publicar un nuevo ID no sobrescribe por sí mismo los snapshots ni evaluaciones anteriores.

### 7.2 Vista previa histórica incorrecta

**Estado: Defecto comprobado.**

La vista previa de una tarjeta no consulta snapshots por el ID seleccionado. `usePreviewModal` llama una sola vez a `groupsService.listAll()` y reutiliza en memoria esos grupos globales para cualquier periodo.

Evidencia:

- React: `sisproing-app/src/modules/employee-evaluations/periods/hooks/usePreviewModal.ts`, líneas 21–45.
- React: `sisproing-app/src/modules/employee-evaluations/periods/components/previewModal.tsx`, líneas 90–127.

Por ello, el título puede corresponder a un periodo histórico mientras las preguntas corresponden al catálogo global actual.

## 8. Comportamiento funcional recomendado

### 8.1 Modelo conceptual mínimo

Separar explícitamente estas dimensiones:

- **Publicación:** `DRAFT` frente a `PUBLISHED`.
- **Vigencia calculada:** `SCHEDULED`, `ACTIVE`, `FINISHED` según fechas y zona horaria.
- **Cierre administrativo:** no introducirlo en P1 salvo decisión expresa; si se necesita, debe ser un estado distinto y persistido, no reutilizar `isActive`.

Para la implementación mínima se puede conservar temporalmente `isActive` como bandera de publicación, pero el contrato y la UI deben nombrarlo y tratarlo de esa forma.

### 8.2 Flujo recomendado

1. En el listado, mostrar una acción visible **Nuevo periodo**.
2. Al pulsarla, consultar si existe un borrador:
   - si existe, ofrecer continuar ese borrador;
   - si no existe, abrir un formulario vacío;
   - nunca adoptar silenciosamente un histórico desactivado como borrador.
3. Guardar crea exactamente un registro nuevo `DRAFT` o actualiza únicamente ese borrador.
4. Publicar valida datos, catálogos, traslapes y asignaciones; dentro de una transacción genera snapshots/asignaciones y marca publicado.
5. Después de publicar, limpiar el estado local y navegar al periodo publicado. No permitir que el formulario conserve su ID como borrador.
6. Los periodos publicados se vuelven inmutables respecto de contenido snapshot e identidad. Cualquier prórroga/corrección de fechas requiere una operación explícita y una decisión de negocio.
7. Las vistas históricas consultan siempre por `periodId` y usan snapshots del periodo.
8. Los procesos operativos que necesiten el periodo vigente aplican una regla única: `published && startDate <= fechaBogota <= endDate`.
9. Si no existe vigente, devolver estado vacío explícito; no escoger silenciosamente uno vencido o futuro.

## 9. Reglas de negocio propuestas

| ID | Regla propuesta | Motivo |
|---|---|---|
| RN-P01 | `isActive` se interpreta temporalmente como «publicado»; nunca como «vigente hoy». | Evita desactivar históricos y confundir selección. |
| RN-P02 | Solo puede existir un borrador editable a la vez. La regla debe aplicarse en backend y, de ser viable con el esquema, mediante restricción transaccional. | Evita que `findDraftPeriod` falle y que React edite uno arbitrario. |
| RN-P03 | Un borrador nuevo debe tener ID propio; nunca reutiliza el ID de un periodo publicado o histórico. | Conserva snapshots, respuestas y resultados. |
| RN-P04 | Los periodos publicados no pueden traslaparse. El límite es inclusivo: el nuevo debe iniciar después de la fecha final del anterior, salvo que negocio apruebe otra semántica. | Mantiene una sola campaña vigente por fecha. |
| RN-P05 | Publicar es idempotente desde la perspectiva del cliente: una repetición no crea snapshots ni asignaciones adicionales y devuelve un resultado reconocible. | Protege frente a doble clic, reintentos y pérdida de respuesta. |
| RN-P06 | Una evaluación es única por periodo, evaluador y evaluado. Repetir la relación en otro periodo crea una evaluación distinta y válida. | Preserva el histórico anual. |
| RN-P07 | La publicación debe fallar si no hay al menos una asignación elegible, salvo aprobación expresa de periodos vacíos. | Evita campañas publicadas sin destinatarios. |
| RN-P08 | `FINISHED` es el único término del contrato API/UI para vencido en P1. `FINALIZED` se elimina o se mapea explícitamente. | Corrige el histórico invisible. |
| RN-P09 | Inicio y fin son fechas de negocio inclusivas en `America/Bogota`; `FINISHED` comienza al día siguiente de `endDate`. | Evita finalizar durante la fecha de cierre. |
| RN-P10 | Reportes, histórico y vistas previas reciben `periodId` explícito. No cambian silenciosamente de periodo. | Evita mezclar campañas. |
| RN-P11 | Recordatorios solo operan sobre el periodo publicado vigente por fecha; si hay cero o más de uno, no envían y registran una condición operativa. | Evita notificaciones del periodo incorrecto. |
| RN-P12 | Desactivar no equivale a cerrar, finalizar ni borrar. P1 no debe ofrecer esa acción sobre un publicado hasta definir cancelación/anulación. | Protege datos históricos. |

## 10. Archivos que habría que modificar

### React — `sisproing-app`

| Archivo | Cambio mínimo esperado |
|---|---|
| `src/modules/employee-evaluations/periods/PeriodsPage.tsx` | Agregar «Nuevo periodo», pestaña/representación de borrador y contrato `FINISHED`; evitar tratar borrador como finalizado. |
| `src/modules/employee-evaluations/periods/components/PeriodsItem.tsx` | Mapear estados exhaustivamente y limitar acciones según borrador/publicación/vigencia. |
| `src/modules/employee-evaluations/periods/periods.service.ts` | Corregir el tipo a `DRAFT \| SCHEDULED \| ACTIVE \| FINISHED`; soportar el contrato para nuevo/continuar borrador y snapshots por periodo. |
| `src/modules/employee-evaluations/layouts/ConfigurationLayout.tsx` | Diferenciar modo nuevo/editar borrador; limpiar `draftId` al publicar; no actualizar publicados; manejar conflicto de borrador único. |
| `src/modules/employee-evaluations/periods/CreatePeriodPage.tsx` | Mostrar el modo y el borrador que se está editando; aplicar mensajes de traslape y fechas. |
| `src/modules/employee-evaluations/periods/hooks/usePreviewModal.ts` | Consultar snapshots mediante `periodId`; no reutilizar catálogos globales entre periodos. |
| `src/modules/employee-evaluations/periods/components/previewModal.tsx` | Renderizar la estructura snapshot retornada para el periodo. |
| `src/modules/employee-evaluations/routes/index.tsx` y/o `src/components/SideMenu.tsx` | Alinear la navegación y permisos visuales de listar/crear; la matriz definitiva de permisos queda fuera de esta microauditoría. |

### API — `api`

| Archivo | Cambio mínimo esperado |
|---|---|
| `src/modules/employee-evaluations/periods/periods.service.js` | Enforzar borrador único, separar actualización de borrador de operaciones sobre publicado, unificar estado/fechas y seleccionar vigente por fecha. |
| `src/modules/employee-evaluations/periods/periods.controller.js` | Exponer errores/contratos claros para borrador existente, conflicto, publicación repetida y ausencia de asignaciones. |
| `src/modules/employee-evaluations/periods/periods.route.js` | Incorporar, si se aprueba, endpoint explícito de snapshots/preview o de creación controlada; no usar `PUT` genérico para cerrar/desactivar. |
| `src/modules/employee-evaluations/periods/snapshots/periodSnapshots.service.js` | Mantener copia transaccional y soportar lectura de snapshots del periodo; revisar idempotencia. |
| `src/modules/employee-evaluations/evaluations/evaluations.service.js` | Mantener llave por `periodId`; devolver conteo elegible/creado y probar duplicados. |
| `src/modules/employee-evaluations/evaluations/evaluations.controller.js` | Evitar selección implícita incorrecta; usar vigente por fecha para operación actual y `periodId` para histórico. |
| `src/modules/employee-evaluations/reports/reports.service.js` | Recibir `periodId` en consultas históricas y aplicar una selección determinista solo donde proceda. P3 ampliará el dashboard; P1 debe evitar selección errónea. |
| `src/modules/employee-evaluations/schedulers/daily-reminder.scheduler.js` | No producir jobs fuera de la vigencia ni ante selección ambigua. |
| `prisma/modules/evaluations.prisma` | Si se aprueba, representar estado/publicación de forma explícita y agregar la garantía de borrador único/idempotencia que resulte viable. Requiere diseño de migración sin alterar históricos. |

## 11. Casos de prueba propuestos

Todas las pruebas de publicación deben ejecutarse en ambiente aislado, con cola/correo deshabilitados y datos sintéticos.

| ID | Caso | Resultado esperado |
|---|---|---|
| CP-01 | Existe un periodo publicado vencido y no hay borrador; abrir «Nuevo periodo». | Formulario vacío; Guardar crea un ID nuevo. El histórico conserva `isActive=true`. |
| CP-02 | Existe un borrador; pulsar «Nuevo periodo». | Se informa que existe y se ofrece continuarlo; no se crea un segundo borrador. |
| CP-03 | Dos clientes intentan crear borrador simultáneamente. | Uno crea; el otro recibe conflicto controlado. Queda un solo borrador. |
| CP-04 | Nuevo rango no se traslapa con el histórico. | Guardado y publicación permitidos. |
| CP-05 | Nuevo inicio es igual al fin del periodo anterior. | Se rechaza como traslape si se aprueba semántica inclusiva. |
| CP-06 | Fin anterior al inicio o fecha inválida. | Error 400 específico; no se crea registro. |
| CP-07 | Publicar sin alguno de los catálogos mínimos. | Se rechaza; no quedan snapshots, evaluaciones ni cambio de publicación parcial. |
| CP-08 | Publicar sin relaciones elegibles jefe–colaborador. | Se rechaza si se aprueba RN-P07; el periodo continúa en borrador. |
| CP-09 | Publicación exitosa. | Snapshots y asignaciones pertenecen solo al nuevo `periodId`; el periodo anterior no cambia. |
| CP-10 | Doble clic/reintento de publicación. | Una sola publicación lógica; sin snapshots ni asignaciones duplicadas. |
| CP-11 | Misma pareja jefe–colaborador en dos periodos. | Una evaluación en cada periodo; ninguna colisión cruzada. |
| CP-12 | Relación `EmployeeManager` duplicada para el mismo par. | Una sola evaluación para la tupla del nuevo periodo. |
| CP-13 | Cerrar el modal de publicación sin navegar. | El formulario se reinicia o queda bloqueado como publicado; Guardar no hace `PUT` sobre el publicado. |
| CP-14 | Listado con un borrador, uno futuro, uno vigente y uno vencido. | Cada uno aparece una vez en la pestaña correcta; el vencido usa `FINISHED`; el borrador no se etiqueta Finalizado. |
| CP-15 | Fecha actual igual a `endDate` en Bogotá. | Estado `ACTIVE` hasta terminar el día; `FINISHED` al día siguiente. |
| CP-16 | Abrir vista previa de dos periodos con preguntas diferentes. | Cada modal muestra snapshots de su propio `periodId`; no reutiliza el catálogo global. |
| CP-17 | Hay histórico vencido y nuevo periodo futuro publicado. | La vista operativa no cambia al futuro antes de su inicio; el histórico sigue consultable por ID. |
| CP-18 | No existe periodo vigente por fecha. | Vista operativa muestra estado vacío; no cae silenciosamente al vencido o futuro. |
| CP-19 | Cron con periodo futuro o vencido. | No genera jobs ni llama a envío. |
| CP-20 | Cron con más de un candidato vigente por una inconsistencia. | No genera jobs; registra error operativo para corrección. |
| CP-21 | Intentar editar nombre/fechas/`isActive` de un publicado mediante endpoint genérico. | Operación rechazada; el histórico queda inmutable. |
| CP-22 | Fallo inducido durante snapshots o asignaciones. | Rollback total: periodo continúa borrador y no quedan datos parciales. |

No se encontraron pruebas automatizadas específicas de periodos en los repositorios React o API durante esta revisión.

## 12. Riesgos sobre periodos anteriores

| Riesgo | Nivel | Mitigación propuesta |
|---|---|---|
| Desactivar el histórico para «liberar» el nuevo | Crítico | Mantenerlo publicado; prohibir `isActive=false` sobre publicados hasta definir anulación. |
| Reutilizar el ID anterior desde el formulario | Crítico | Crear ID nuevo; limpiar `draftId` al publicar; backend solo permite editar borradores. |
| Cambiar fechas de un publicado | Alto | Inmutabilidad por defecto y operación explícita para prórroga, si se aprueba. |
| Dashboard/evaluaciones/cron escogen otro publicado | Alto | Regla única de vigencia y `periodId` explícito para histórico. |
| Recordatorios del futuro o del vencido | Alto | Filtro por ventana de fechas en Bogotá antes de encolar. |
| Histórico visible con preguntas actuales | Alto | Preview por snapshots del `periodId`. |
| Vencido oculto por `FINISHED`/`FINALIZED` | Medio | Unificar contrato y pestañas. |
| Vencimiento anticipado en la fecha final | Medio | Comparación por fecha local inclusiva. |
| Borradores múltiples | Alto | Restricción backend/transaccional y respuesta 409. |
| Doble publicación concurrente | Medio | Idempotencia, bloqueo/condición atómica y prueba integrada. |
| Confundir asignaciones repetidas entre años con duplicados | Medio | Definir duplicado solo dentro del mismo `periodId`; consultas siempre acotadas por periodo. |

No se propone recalcular, migrar, desactivar ni volver a publicar periodos anteriores como parte de P1. Si existen registros históricos con `isActive=false` pero con snapshots/evaluaciones, deben inventariarse en una validación controlada antes de migrar estados; no deben ser tratados automáticamente como borradores.

## 13. Decisiones aprobadas y pendientes antes de implementar

**Aprobaciones del responsable (2026-10-14):** un **único borrador global** y **periodos publicados/finalizados inalterables por defecto**. Una posible reasignación excepcional de evaluador queda fuera de P1 y se abordará en [MEJ-005](MEJ-005-reasignacion-manual-evaluador.md), mediante una acción específica y con auditoría.

1. **Borradores: APROBADO.** Un único borrador global.
2. **Terminología:** aprobar `DRAFT`, `SCHEDULED`, `ACTIVE`, `FINISHED`; eliminar `FINALIZED` del contrato actual.
3. **Fin de vigencia:** confirmar que inicio y fin son inclusivos por día en `America/Bogota`.
4. **Traslapes:** confirmar si dos periodos pueden compartir el día límite. Recomendación: no; el siguiente inicia al día posterior.
5. **Publicación futura:** confirmar que se permite publicar por anticipado sin que se convierta en periodo operativo hasta `startDate`.
6. **Sin periodo vigente:** aprobar estado vacío explícito. Recomendación: no usar automáticamente el publicado más reciente en flujos operativos.
7. **Periodo sin asignaciones:** decidir si publicar con cero evaluaciones es válido. Recomendación: rechazar.
8. **Inmutabilidad: APROBADO como regla general.** Bloquear edición genérica de periodos publicados e históricos. La política para prórrogas/correcciones continúa pendiente y se diseñará como excepción explícita.
9. **Cierre persistido:** decidir si `FINISHED` seguirá calculado o habrá cierre administrativo. Recomendación para la primera entrega: mantenerlo calculado y no introducir cierre persistido todavía.
10. **Cancelación/anulación:** decidir el significado y efectos; no reutilizar `isActive=false` mientras no exista la regla.
11. **Vista histórica:** confirmar que debe mostrar los snapshots completos del periodo, no catálogos actuales.
12. **Selección predeterminada en reportes:** P3 decidirá la UX completa; P1 debe aprobar que toda consulta histórica acepte `periodId` y que ninguna selección sea no determinista.
13. **Periodo desactivado existente:** definir tratamiento de cualquier `isActive=false` que ya tenga snapshots o evaluaciones antes de aplicar la regla de borrador único.

## 14. Criterios de aceptación de MEJ-001

1. Desde el listado existe una ruta visible y comprensible para crear o continuar un borrador.
2. Solo existe un borrador y no se confunde con históricos desactivados.
3. Un periodo vencido se muestra como `FINISHED` y permanece consultable.
4. Crear/publicar un periodo nuevo no actualiza registros, snapshots, evaluaciones, respuestas ni resultados anteriores.
5. Publicar crea snapshots y asignaciones asociados exclusivamente al nuevo `periodId`.
6. No se generan asignaciones duplicadas dentro del nuevo periodo.
7. Un periodo futuro no desplaza la operación vigente antes de su fecha inicial.
8. El cron no genera recordatorios fuera de la ventana vigente.
9. La vista previa histórica usa snapshots del periodo seleccionado.
10. Las pruebas integradas se ejecutan sin notificaciones reales.

## 15. Propuesta mínima de implementación

La intervención mínima recomendada se divide en cuatro cambios acotados:

1. **Corregir contrato y navegación React.** Agregar «Nuevo periodo» en el listado; representar `DRAFT`, `SCHEDULED`, `ACTIVE` y `FINISHED`; enlazar a un formulario con modo explícito nuevo/continuar.
2. **Cerrar el ciclo del borrador en API.** Garantizar un único borrador, permitir actualizar solo borradores, limpiar el estado React después de publicar y rechazar modificaciones genéricas de publicados.
3. **Hacer determinista la vigencia.** Crear una única función backend que resuelva el periodo vigente por fechas inclusivas de Bogotá; usarla en «Mi equipo» y recordatorios. En consultas históricas, exigir `periodId`. El ajuste completo del dashboard queda para P3, pero P1 debe impedir la selección arbitraria.
4. **Preservar el histórico.** Mantener cada periodo anterior con su ID, bandera de publicación, snapshots y evaluaciones; servir la vista previa desde snapshots del periodo. No migrar ni recalcular resultados históricos.

## 16. Estrategia de validación P1 — proyecto sin suite de pruebas existente

**Contexto confirmado por el responsable (2026-10-14):** `api` y `sisproing-app` todavía no cuentan con una suite automatizada específica para este proyecto. Este plan **no debe exigir adoptar Jest/Vitest u otra infraestructura completa** como condición previa para implementar P1.

- **Obligatorio:** checklist manual de aceptación de P1 sobre ambiente de pruebas con datos representativos, registro de resultados por caso y comprobaciones de integridad por `periodId` antes/después. Sin publicar ni enviar notificaciones reales en producción durante pruebas.
- **Obligatorio para publicación:** prueba integrada controlada de guardado de borrador, publicación transaccional, snapshots, asignaciones, preservación histórica y selección de periodo; usar ambiente aislado, datos sintéticos y colas/correos deshabilitados.
- **Deseable, no bloqueante para iniciar:** pruebas automatizadas pequeñas para una función pura de fechas, estados y traslapes, si Codex puede ejecutarlas con las herramientas ya disponibles (por ejemplo `node:test` nativo si la versión de Node lo permite). Inspeccionar `package.json` y herramientas existentes antes de elegir framework.
- **No requerido en P1:** montar un sistema integral de unit testing, cobertura global ni pruebas automáticas de todos los módulos de PROING.
- **Regla de seguridad de datos:** nunca crear periodos ni asignaciones de prueba en producción, ni ejecutar triggers de notificaciones para validar la interfaz. Comprobar con SQL de solo lectura o fixture aislado la integridad del histórico.

La lista de 22 casos de la sección 11 es el **plan de aceptación**, no la promesa de que existen 22 tests automatizados. Marcar en la ejecución cada caso como probado manualmente, automatizado, pendiente o no aplicable, con evidencia.

**Orden sugerido:** resolver primero contratos y selección por periodo; implementar backend de borrador/vigencia y UI; realizar verificación manual e integración segura de cada incremento. Agregar automatizaciones puntuales solo si aportan valor inmediato y no desvían el alcance P1.

Esta propuesta no requiere modificar evaluaciones históricas ni desactivar el periodo anterior. Requiere decisiones funcionales de la sección 13 antes de autorizar cambios de código o esquema.


## 17. Avance de implementación — P1.1 (reporte de Codex, 2026-10-14)

**Estado:** implementado localmente en `sisproing-app` **según reporte del ejecutor**; **no verificado de manera independiente ni aceptado funcionalmente todavía**. La documentación central registra el reporte; no afirma que los cambios hayan sido publicados en la rama remota.

### Cambios reportados (exclusivamente React)

- Estados unificados a `DRAFT`, `SCHEDULED`, `ACTIVE`, `FINISHED`; se retiró `FINALIZED`.
- Pestaña «Borradores» y listado inicial en «Todos» para mostrar el histórico.
- Acción visible «Nuevo periodo».
- Si existe borrador: diálogo para «Continuar borrador» en lugar de generar o editar uno silenciosamente.
- Sin borrador: formulario de creación vacío.
- Indicación visible de edición/continuación del borrador.
- Si `GET /periods/draft` falla: bloqueo de Guardar/Publicar para evitar duplicados.
- Después de publicar: limpieza de `draftId` y del formulario.

**Archivos reportados:**

1. `sisproing-app/src/modules/employee-evaluations/periods/periods.service.ts`
2. `sisproing-app/src/modules/employee-evaluations/periods/PeriodsPage.tsx`
3. `sisproing-app/src/modules/employee-evaluations/periods/components/PeriodsItem.tsx`
4. `sisproing-app/src/modules/employee-evaluations/layouts/ConfigurationLayout.tsx`

**Validaciones técnicas declaradas por Codex:**

- `npm run typecheck`: correcto.
- ESLint dirigido a los cuatro archivos: correcto.
- `npm run build`: correcto.
- `git diff --check`: correcto.
- Lint global: 63 errores y 6 advertencias **reportados como preexistentes en otros archivos**, no imputados a P1.1; no se hizo confirmación independiente.

### Aceptación pendiente

La aprobación de P1.1 requiere ejecutar y registrar el checklist manual en un ambiente seguro (sin publicar un periodo real): pestañas, estados, creación sin borrador, continuación con borrador, bloqueo ante error `getDraft`, reinicio del formulario tras publicación simulada y permanencia del histórico. Las pruebas de flujo exitoso de publicación real corresponden a P1.2 y posteriores en un entorno aislado.

**Límite:** no hubo cambios de API, PostgreSQL, notificaciones ni periodos históricos, según el reporte de Codex.

### Próximo incremento — P1.2 (sin iniciar)

- Garantizar desde API un borrador único, incluyendo concurrencia y respuesta de conflicto controlada, sin sobrescribir registros existentes.
- Restringir el `update` genérico a borradores; no permitir modificar un periodo publicado/histórico por ese endpoint.
- Revisar la semántica de `isActive` (publicación, no vigencia).
- Mantener integridad de snapshots y asignaciones al publicar; evitar duplicados/reintentos.
- Antes de imponer unicidad, inspeccionar con consultas de **solo lectura** si existen borradores múltiples o periodos `isActive=false` con datos históricos.
- No introducir un framework general de testing: checklist manual e integración segura son mínimos; pruebas automatizadas puntuales son opcionales.

**No se autoriza aquí ejecutar migraciones, publicar periodos reales ni tocar otros módulos.**


## 18. Decisión aprobada — Descartar borrador y crear otro (2026-10-14)

**Situación observada:** en capturas del entorno de pruebas, la pantalla «Periodos» lista un borrador «Evaluacion 2027» (fechas 15/10/2026 a 31/10/2026) y el histórico «Evaluación de Desempeño - Año 2025» como `FINISHED`. Al pulsar «Nuevo periodo», React presenta correctamente «Ya existe un periodo en borrador» y ofrece «Continuar borrador». El responsable confirma que «Evaluacion 2027» **es un borrador de prueba** y **aprueba añadir «Descartar borrador»** para poder iniciar uno nuevo.

**Estado de P1.1:** confirmación visual parcial de listado «Todos», etiqueta «Borrador», etiqueta «Finalizado», botón «Nuevo periodo» y diálogo «Continuar borrador». No se han completado todos los casos manuales ni se ha hecho una publicación de prueba.

### Reglas aprobadas y diseño mínimo de P1.2

1. **Un solo borrador global a la vez:** se mantiene la decisión de P1.
2. Cuando existe borrador, «Nuevo periodo» mostrará como mínimo **Cancelar**, **Continuar borrador** y **Descartar borrador**; se puede mostrar la acción de descartar también en la tarjeta `DRAFT` si la interfaz queda consistente.
3. **Descartar requiere confirmación explícita**, mostrando nombre del borrador e informando que no puede recuperarse. No descartar por entrar al formulario ni por cancelar un diálogo.
4. El **backend** realizará la validación definitiva antes de eliminar: el registro existe, sigue siendo borrador (nunca publicado), **no tiene snapshots, evaluaciones, respuestas ni resultados históricos asociados** y no existe ninguna operación concurrente que lo haya publicado o transformado. En caso de duda o datos relacionados, **rechazar sin borrar**.
5. No reutilizar `PUT {isActive:false}` como descarte; implementar una operación explícita y delimitada del recurso borrador. Si se usa `DELETE`, el endpoint debe verificar el estado y las relaciones antes de actuar, de forma transaccional/atómica donde corresponda.
6. Si el descarte se completa, limpiar el estado local y permitir abrir un **formulario nuevo vacío**. La creación real será un `POST` y deberá generar **otro `periodId`**.
7. Si otro usuario guardó, descartó o publicó ese borrador desde que se abrió el diálogo, responder con conflicto o no encontrado y **recargar el listado**, sin afectar el periodo nuevo/publicado.
8. Descartar está restringido a **borradores sin historial**. **Nunca** se permite descartar el periodo histórico 2025, un `SCHEDULED`, `ACTIVE` o `FINISHED` ni un `isActive=false` heredado que tenga snapshots/evaluaciones.
9. No eliminar ni cambiar datos reales por esta aprobación: el responsable aprobará/ejecutará **cada acción de descarte en la aplicación**, incluso en el entorno de pruebas. Codex no debe borrar «Evaluacion 2027» ni ningún otro registro automáticamente.
10. Los cambios quedan acotados al módulo de periodos de `api` y `sisproing-app`; no hay migraciones ni notificaciones autorizadas por esta decisión.

### Validaciones propuestas para P1.2 (sin suite de testing preexistente)

- **Borrador normal sin dependencias:** confirmar descarte, desaparece del listado, siguiente «Nuevo periodo» muestra formulario vacío y crea ID nuevo si se guarda.
- **Cancelar el diálogo:** no borra ni modifica ningún registro.
- **Histórico publicado:** no ofrece «Descartar» y un intento directo al endpoint se rechaza.
- **Borrador anómalo con snapshots o evaluaciones:** rechazo seguro; no depender de cascadas ni borrar relaciones.
- **Competencia entre dos administradores:** si el primero publica/descarta, el segundo recibe conflicto/no encontrado; sin borrados sobre un publicado.
- **Error de servidor/red:** mostrar el error, conservar o refrescar datos con seguridad; no afirmar éxito sin confirmación.
- **Borradores múltiples heredados:** antes de introducir una restricción de unicidad o migración, inventariarlos en lectura; no borrarlos automáticamente.
- **Regresión histórica:** comparar por `periodId` el histórico antes y después de la prueba en ambiente aislado.

**Pendiente complementario de P1.2:** la garantía de borrador único **debe implementarse en backend** y manejar concurrencia. El botón de React por sí solo no la garantiza. Evaluar solución sin migración cuando sea segura; si exige un índice/constraint nuevo, **presentar el plan y solicitar aprobación antes de ejecutar cualquier migración**.

**Próximo paso:** pedir a Codex implementación acotada a P1.2 tras leer esta decisión. No ampliar aún a P1.3 (resolver periodo vigente) ni P1.4 (vista previa histórica).

## 19. Avance de implementación — P1.2 (2026-10-14)

**Estado:** implementado localmente en `api` y `sisproing-app`; compilación, tipos y lint dirigido correctos. No se ejecutaron migraciones, descartes reales, publicaciones, notificaciones ni cambios sobre periodos históricos. La aceptación funcional permanece pendiente del checklist manual en un ambiente seguro.

### Contrato implementado

Se agregó la operación explícita:

`DELETE /api/employee-evaluations/periods/:id/draft`

Payload requerido:

```json
{
  "updatedAt": "fecha ISO recibida al consultar el borrador"
}
```

La versión `updatedAt` evita descartar silenciosamente un borrador que otro administrador editó después de abrir la confirmación.

Respuestas de rechazo relevantes:

| Código | HTTP | Condición |
|---|---:|---|
| `EVAL_PERIOD_NOT_FOUND` | 404 | El registro no existe o ya fue descartado. |
| `EVAL_PERIOD_NOT_DRAFT` | 409 | El periodo ya está publicado; incluye `SCHEDULED`, `ACTIVE` y `FINISHED`. |
| `EVAL_PERIOD_DRAFT_CHANGED` | 409 | Cambió la versión o el estado durante la operación. |
| `EVAL_PERIOD_DRAFT_HAS_HISTORY` | 409 | Tiene snapshots, evaluaciones o una FK relacionada. |
| `EVAL_PERIOD_DRAFT_EXISTS` | 409 | Ya existe un borrador al intentar crear otro. |
| `EVAL_PERIOD_MULTIPLE_DRAFTS` | 409 | Hay más de un `isActive=false` heredado; no se crea ni borra automáticamente. |
| `EVAL_PERIOD_PUBLISHED_IMMUTABLE` | 409 | Se intentó editar un periodo publicado mediante el `PUT` genérico. |
| `EVAL_PERIOD_STATE_CHANGE_NOT_ALLOWED` | 400 | Se intentó cambiar `isActive` mediante el `PUT` genérico. |

### Salvaguardas implementadas en backend

1. Crear, editar, publicar y descartar participan en el mismo bloqueo asesor transaccional de PostgreSQL (`pg_advisory_xact_lock`). El bloqueo se comparte entre instancias del API y se libera automáticamente al confirmar o revertir.
2. La creación consulta el borrador dentro de la transacción bloqueada. Si existe uno, devuelve 409; si detecta al menos dos, devuelve conflicto de inconsistencia sin alterar registros.
3. El descarte vuelve a leer el periodo dentro de la transacción y rechaza todo `isActive=true`.
4. Antes de eliminar cuenta `classifications`, `groups`, `questions`, `strengths`, `weaknesses`, `scoreScales` y `evaluations`. Cualquier conteo positivo bloquea la operación. Respuestas, selecciones y resultados dependen de `Evaluation`, por lo cual la presencia de una evaluación bloquea todo el histórico subordinado.
5. La eliminación usa `deleteMany` condicionado por `id`, `isActive=false` y `updatedAt`; debe afectar exactamente una fila. No elimina relaciones ni ejecuta borrado manual en cascada.
6. Una restricción FK concurrente (`P2003`) se convierte en `EVAL_PERIOD_DRAFT_HAS_HISTORY`; la operación se revierte.
7. La publicación ahora toma el mismo bloqueo y comprueba/activa el borrador dentro de una única transacción. Si descarte o publicación gana la carrera, la otra operación recibe 404/409 y no deja snapshots parciales.
8. El `PUT /periods/:id` solo actualiza borradores. Ya no acepta cambios de `isActive` ni modificaciones sobre periodos publicados.

### Evaluación de migración

**No se requiere migración para P1.2.** La unicidad entre operaciones normales del API queda serializada mediante el bloqueo transaccional y una comprobación dentro de la misma transacción.

Como endurecimiento futuro a nivel de base podría evaluarse un índice único parcial equivalente a «máximo un registro con `is_active=false`». **No se creó ni ejecutó.** Antes de proponerlo para aprobación se requiere:

1. inventario de solo lectura de todos los `isActive=false`;
2. clasificación de registros heredados con snapshots/evaluaciones;
3. separar conceptualmente borrador de desactivado/anulado, porque hoy comparten la bandera;
4. plan explícito para resolver cualquier duplicado sin borrarlo automáticamente.

Aplicar el índice con el modelo actual podría bloquear por registros históricos desactivados y por eso queda fuera de P1.2.

### Cambios de interfaz

- El diálogo de «Nuevo periodo» conserva **Cancelar** y **Continuar borrador**, y agrega **Descartar borrador**.
- Descartar abre una segunda confirmación con el nombre y la advertencia de que no puede recuperarse.
- Solo la confirmación final invoca el endpoint.
- Al completar, el borrador se retira del estado local y se abre Configuración de Períodos; al no existir borrador, el formulario queda vacío y el siguiente guardado crea otro `periodId`.
- Ante 404, 409 o error de red se muestra el mensaje del backend y se recarga el listado sin afirmar éxito.
- Se retiraron de las tarjetas las acciones genéricas Editar/Eliminar para periodos publicados o programados. `DRAFT` conserva **Continuar borrador** y todos conservan vista previa.

### Archivos modificados en P1.2

API:

1. `api/src/modules/employee-evaluations/periods/periods.service.js`
2. `api/src/modules/employee-evaluations/periods/periods.controller.js`
3. `api/src/modules/employee-evaluations/periods/periods.route.js`

React:

1. `sisproing-app/src/modules/employee-evaluations/periods/periods.service.ts`
2. `sisproing-app/src/modules/employee-evaluations/periods/PeriodsPage.tsx`
3. `sisproing-app/src/modules/employee-evaluations/periods/components/PeriodsItem.tsx`

`ConfigurationLayout.tsx` conserva los cambios de P1.1 para limpiar `draftId` al publicar; no requirió cambios adicionales para P1.2.

Documentación:

1. `evaluaciones-docs/docs/mejoras/MEJ-001-historico-creacion-periodos.md`

### Resultados de validación técnica

| Repositorio | Validación | Resultado |
|---|---|---|
| `api` | `node --check` sobre servicio, controlador y rutas | Correcto |
| `api` | `npm run type-check` | Correcto |
| `api` | `npm run build` | Correcto |
| `api` | Lint | No existe script de lint configurado |
| `sisproing-app` | `npm run typecheck` | Correcto |
| `sisproing-app` | ESLint dirigido a los archivos React/servicio de periodos modificados y `ConfigurationLayout.tsx` | Correcto |
| `sisproing-app` | `npm run build` | Correcto; solo advertencias informativas por datos desactualizados de Browserslist/baseline |
| Ambos | `git diff --check` | Correcto |

No se añadió un framework de pruebas ni se ejecutaron endpoints mutantes contra una base real.

### Checklist manual de P1.2 — pendiente de ejecución

Ejecutar solo en ambiente aislado, con datos sintéticos y sin publicar ni enviar notificaciones:

| Caso | Acción | Resultado esperado | Estado |
|---|---|---|---|
| Borrador normal | Nuevo periodo → Descartar → confirmar | Desaparece el borrador; abre formulario vacío; el histórico conserva sus datos (HTTP 200 no verificado explícitamente). | **Validado manualmente (responsable, 2026-10-14)** |
| Cancelación inicial | Abrir diálogo y pulsar Cancelar | No se llama DELETE y no cambia el listado. | Pendiente |
| Cancelación final | Pulsar Descartar y luego Volver | No se llama DELETE y el borrador permanece. | Pendiente |
| Continuación | Pulsar Continuar borrador | Abre el mismo ID con sus datos; no crea otro registro. | Pendiente |
| Publicado/histórico | Invocar directamente DELETE sobre `SCHEDULED`, `ACTIVE` o `FINISHED` | 409 `EVAL_PERIOD_NOT_DRAFT`; no cambia ninguna fila. | Pendiente |
| Borrador con snapshots | Intentar descartar un `isActive=false` anómalo con snapshots | 409 `EVAL_PERIOD_DRAFT_HAS_HISTORY`; no se elimina padre ni relaciones. | Pendiente |
| Borrador con evaluación/resultados | Intentar descartar un borrador anómalo relacionado | 409 `EVAL_PERIOD_DRAFT_HAS_HISTORY`; evaluaciones, respuestas y resultados permanecen. | Pendiente |
| Versión obsoleta | Editar el borrador en otra sesión y confirmar desde el diálogo antiguo | 409 `EVAL_PERIOD_DRAFT_CHANGED`; se refresca el listado. | Pendiente |
| Publicación concurrente simulada | Una sesión publica y otra intenta descartar | Gana una sola operación; descarte recibe 409 si quedó publicado, sin snapshots parciales. No ejecutar publicación real. | Pendiente |
| Doble descarte | Dos sesiones confirman el mismo borrador | Una obtiene 200; la otra 404; no se afecta otro periodo. | Pendiente |
| Doble creación | Dos clientes hacen POST simultáneo sin borrador previo | Uno crea; el otro obtiene 409 `EVAL_PERIOD_DRAFT_EXISTS`; queda un borrador. | Pendiente |
| Borradores heredados múltiples | Consultar/crear con dos `isActive=false` preparados en fixture aislado | 409 `EVAL_PERIOD_MULTIPLE_DRAFTS`; no se crea ni borra nada. | Pendiente |
| Edición de publicado | Enviar `PUT` a un periodo publicado | 409 `EVAL_PERIOD_PUBLISHED_IMMUTABLE`; fechas, nombre y estado permanecen. | Pendiente |
| Cambio genérico de estado | Enviar `PUT {isActive:false}` | Rechazo; no transforma un publicado en borrador. | Pendiente |
| Error de red/servidor | Interrumpir DELETE | La UI muestra error, no afirma éxito y conserva/recarga el estado seguro. | Pendiente |

P1.2 no resuelve periodo vigente, reportes por periodo ni vista previa histórica; esos trabajos continúan reservados para P1.3/P1.4 y mejoras posteriores.


## 20. Resultado de la prueba funcional P1.2 — 2026-10-14

El responsable del proyecto reportó **seis comprobaciones manuales exitosas (6/6)** en la interfaz. Se documentan como observaciones de usuario, no como ejecución independiente de pruebas automatizadas o inspección SQL.

| Nº | Comprobación reportada | Resultado |
|---|---|---|
| 1 | «Evaluacion 2027» estaba en borrador y sin evaluaciones ni snapshots asociados. | Aprobada por el responsable |
| 2 | «Descartar borrador» solicitó confirmación. | Aprobada |
| 3 | Al confirmar, desapareció únicamente «Evaluacion 2027». | Aprobada |
| 4 | «Nuevo periodo» abrió el formulario vacío. | Aprobada |
| 5 | Se guardó un nuevo borrador con ID diferente. | Aprobada |
| 6 | El histórico de 2025 permaneció intacto. | Aprobada |

**Estado de cierre:** **flujo funcional principal de P1.2 validado**. Quedan **sin validar**: pruebas explícitas de concurrencia, token `updatedAt` obsoleto, conflictos 404/409, rechazo de `DELETE` a publicados, borradores anómalos con relaciones históricas, operaciones paralelas y error de red. La validación funcional no equivale a cobertura integral de las protecciones backend.

**Siguiente etapa:** P1.3 — selección correcta del periodo vigente por fechas en Bogotá para «Mis Evaluaciones» y recordatorios; no confundir `isActive=true` (publicado) con vigencia. Reportes por `periodId` y su selector visual quedan principalmente para P3. No ejecutar envíos ni publicaciones reales durante la implementación y las pruebas.


## 21. P1.3 — Selección determinista del periodo vigente (aprobado para trabajar el 2026-10-14)

**Decisión del responsable:** continuar con P1.3, conservando la separación de alcance respecto de P3 y P1.4.

### Reglas funcionales aprobadas

1. `EvaluationPeriod.isActive=true` significa **publicado**, no «vigente hoy». No desactivar automáticamente los publicados que ya finalizaron.
2. Un periodo operativo es el **publicado cuya ventana incluye el día de negocio actual en `America/Bogota`**, comparando las fechas de calendario inclusivamente: `startDate <= hoyBogota <= endDate`.
3. Un publicado `SCHEDULED` (futuro) no reemplaza la campaña operativa antes de su fecha inicial. Un `FINISHED` no se usa para la operación actual tras su fecha final.
4. Si **no existe** periodo operativo, «Mis Evaluaciones» muestra un estado informativo/sin periodo vigente; no selecciona silenciosamente un publicado futuro ni uno vencido.
5. Los **recordatorios diarios** se generan solo para el periodo que realmente está vigente. Fuera de la ventana no deben producirse jobs de recordatorio.
6. Si por datos anómalos aparecen **varios candidatos vigentes**, evitar elegir uno arbitrariamente; registrar la ambigüedad y **no encolar recordatorios**. El comportamiento de error/estado vacío para la vista operativa debe ser explícito.
7. La selección histórica debe ser por `periodId` cuando se solicite un periodo específico; **el selector de periodo del dashboard y sus filtros son P3**. P1.3 debe inventariar los consumidores de selección implícita y corregir los de operación actual sin rediseñar reportes.
8. La comparación debe respetar la semántica de `@db.Date` de Prisma/PostgreSQL y el calendario de Bogotá, sin depender del instante UTC ni terminar anticipadamente durante el día final.

### Alcance técnico aprobado

- Localizar usos directos/indirectos de `getActivePeriodOrThrow`, consultas `findFirst({isActive:true})` y funciones equivalentes **dentro del módulo de evaluaciones y su scheduler de recordatorios**. Registrar cuáles son operativos, históricos o de reportes.
- Implementar una regla/servicio centralizado para resolver el periodo **vigente por fecha**.
- Integrar la selección correcta en «Mis Evaluaciones» (resumen/listado) y en el productor/consulta de recordatorios. Evitar que una parte use el periodo más reciente publicado y otra el primero.
- Mantener contratos existentes cuando sea posible; si el cambio de respuesta de «sin periodo» afecta UI, ajustar únicamente la vista del módulo necesaria para mostrar el estado informativo.
- **No cambiar** publicación, borradores, generación de asignaciones, cálculo de puntajes, PDF, reasignación por trigger ni snapshots históricos en este incremento.
- **No implementar todavía P3:** nuevo selector de periodo en reporte, filtros o consolidaciones. Si las consultas del dashboard representan un riesgo inmediato de seleccionar periodo arbitrario, documentar el impacto y proponer un ajuste mínimo antes de ampliar el alcance.

### Aceptación y pruebas

- Día de inicio en Bogotá: periodo operativo desde ese día.
- Día de fin en Bogotá: permanece operativo hasta terminar ese día.
- Día siguiente al fin: no hay periodo operativo.
- Solo periodo histórico 2025: «Mis Evaluaciones» sin campaña vigente; recordatorios sin jobs.
- Publicado futuro y publicado vencido: no seleccionar ninguno.
- Publicado vigente junto a otros publicados no solapados: seleccionar el vigente únicamente.
- Dos periodos simultáneamente vigentes por anomalía: no selección arbitraria ni recordatorios.
- Confirmar mismo `periodId` en resumen, listado y destinatarios del recordatorio.
- Confirmar que no se alteran registros, snapshots ni resultados de años anteriores.
- Comprobar compilación y lint dirigido; pruebas unitarias pequeñas de fechas/selección **opcionales**, sin introducir una suite de testing global.
- Ninguna prueba debe publicar periodos reales ni disparar correos o jobs productivos. Para comprobar recordatorios, usar dobles/stubs o inspección controlada sin encolar.

**Estado:** reglas acordadas; **pendiente implementación y verificación con Codex**. P1.1–P1.2 permanecen documentados con sus pruebas y límites de cobertura.


## 22. Avance de implementación — P1.3 (2026-10-14)

**Estado:** implementación local terminada y validación técnica correcta. El checklist manual con datos sintéticos permanece pendiente. No se consultaron ni modificaron datos reales, no se ejecutaron cron jobs y no se encolaron correos.

### Comportamiento anterior y nuevo

| Flujo | Antes | P1.3 |
|---|---|---|
| Resumen de «Mis Evaluaciones» | `findFirst({isActive:true})`, ordenado por fecha de inicio; podía tomar un publicado futuro o vencido. | Consulta el único publicado cuya ventana contiene `hoyBogota`. Si no existe devuelve `period:null` y estadísticas en cero. |
| Listado de «Mis Evaluaciones» | Resolvía el periodo por separado con el mismo concepto incorrecto de «activo» y podía no coincidir con el resumen. | Usa el mismo resolvedor central; sin vigente devuelve listado/paginación vacíos. |
| Interfaz sin campaña | Mostraba «No hay periodo activo», pero seguía solicitando y renderizando el bloque del equipo. | Muestra un estado informativo de «sin periodo vigente» y no solicita ni presenta el listado operativo. Una ambigüedad muestra un estado de error diferente y explícito. |
| Recordatorio diario | Tomaba el primer `isActive=true` sin fechas ni orden determinista. | Solo arma recordatorios para el único periodo vigente en Bogotá; sin vigente devuelve `[]`. |
| Candidatos solapados | Un `findFirst` elegía uno silenciosamente. | Devuelve 409 `EVAL_PERIOD_CURRENT_AMBIGUOUS`, registra los IDs candidatos y no llega a encolar jobs. |

No se cambia `isActive`: continúa significando **publicado**. Tampoco se modifican publicación, snapshots, asignaciones, evaluaciones históricas ni puntuaciones.

### Función y consulta centralizadas

`periods/currentPeriod.service.js#findCurrentPublishedPeriod` ejecuta una única regla:

```text
isActive = true
AND startDate <= fechaCalendarioBogota
AND endDate >= fechaCalendarioBogota
```

La fecha se construye en `America/Bogota` como `YYYY-MM-DD` y se transforma a medianoche UTC únicamente como representación de la columna Prisma/PostgreSQL `@db.Date`. Las comparaciones son inclusivas; no dependen de la hora UTC del servidor y el periodo continúa vigente durante todo su último día en Bogotá.

La consulta trae como máximo dos candidatos. Cero significa ausencia de campaña; uno es el resultado; dos bastan para declarar ambigüedad sin seleccionar por orden. `currentPeriod.utils.js` contiene la lógica pura de calendario, filtro y clasificación para permitir pruebas aisladas sin base de datos.

### Inventario de consumidores de selección implícita

| Consumidor | Tipo | Situación en P1.3 |
|---|---|---|
| `periods.service.js#getMyTeamOverview` | Operación actual / resumen de «Mis Evaluaciones» | **Corregido:** usa `findCurrentPublishedPeriod`. |
| `evaluations.controller.js#getMyTeamEvaluations` | Operación actual / listado de «Mis Evaluaciones» | **Corregido:** usa el mismo resolvedor y devuelve colección vacía si no hay vigente. |
| `reports.service.js#getUsersForDailyReminder` | Operación actual / productor consultado por el scheduler | **Corregido:** solo consulta pendientes del `periodId` vigente. Una ausencia produce cero recordatorios y una ambigüedad aborta antes de la cola. |
| `reports.service.js#getDashboardProgress` | Dashboard general de reportes | **Pendiente P3:** todavía usa selección implícita `findFirst({isActive:true})`. Debe recibir/seleccionar `periodId` en el rediseño del dashboard; no se amplió P1.3. |
| `reports.service.js#getEvaluationsList` | Drill-down del dashboard general | **Pendiente P3:** todavía elige un publicado implícitamente. Debe compartir el `periodId` explícito del dashboard en P3. |

Las demás consultas de `EvaluationPeriod` encontradas corresponden a CRUD, borrador, publicación, solapamiento o lectura por ID; no son selección implícita de una campaña operativa.

### Archivos modificados en P1.3

API:

1. `api/src/modules/employee-evaluations/periods/currentPeriod.utils.js` — calendario Bogotá, filtro inclusivo y resolución pura de candidatos.
2. `api/src/modules/employee-evaluations/periods/currentPeriod.service.js` — consulta Prisma central y conflicto por ambigüedad.
3. `api/src/modules/employee-evaluations/periods/periods.service.js` — overview con periodo vigente o estado vacío.
4. `api/src/modules/employee-evaluations/evaluations/evaluations.controller.js` — listado con el mismo resolvedor y respuesta vacía sin vigente.
5. `api/src/modules/employee-evaluations/reports/reports.service.js` — recordatorios limitados al vigente.
6. `api/tests/employee-evaluations/current-period.service.test.js` — seis pruebas pequeñas con `node:test`; no introduce otro framework.

React:

1. `sisproing-app/src/modules/employee-evaluations/evaluations/pages/EvaluationsPage.tsx` — espera la resolución del periodo, evita solicitar el listado si no existe y presenta el estado informativo.
2. `sisproing-app/src/modules/employee-evaluations/evaluations/components/ActivePeriodCard.tsx` — cambia la etiqueta visual a «Periodo vigente».

Documentación:

1. `evaluaciones-docs/docs/mejoras/MEJ-001-historico-creacion-periodos.md`.

### Pruebas técnicas realizadas

| Validación | Resultado |
|---|---|
| Seis pruebas `node:test` de calendario Bogotá y resolución | Correcto: inicio inclusivo, fin inclusivo, futuro, finalizado, borrador, publicados no solapados, ambigüedad y frontera de medianoche Bogotá. |
| `node --check` dirigido a los archivos JavaScript modificados | Correcto. |
| API `npm run type-check` | Correcto. |
| API `npm run build` | Correcto. |
| React `npm run typecheck` | Correcto. |
| ESLint dirigido a los dos componentes React modificados | Correcto. |
| React `npm run build` | Correcto; solo advertencias informativas por datos desactualizados de Browserslist/baseline. |
| `git diff --check` en API, React y documentación | Correcto. |

No se ejecutaron endpoints contra la base, publicaciones, migraciones, correos, colas ni el scheduler real.

### Checklist manual de P1.3 — pendiente

Ejecutar únicamente en un ambiente aislado con datos sintéticos y el productor de cola sustituido por un stub:

| Caso | Preparación/acción | Resultado esperado | Estado |
|---|---|---|---|
| Publicado futuro | Único publicado con inicio posterior a hoy Bogotá | «Mis Evaluaciones» informa que no hay vigente; listado vacío; cero jobs. | Pendiente |
| Publicado vigente | Ventana que contiene hoy Bogotá | Resumen, listado y recordatorios usan exactamente el mismo `periodId`. | Pendiente |
| Publicado finalizado | Fin anterior a hoy Bogotá | No se selecciona; no se generan recordatorios. | Pendiente |
| Fecha de inicio | Simular hoy Bogotá igual a `startDate` | Se selecciona el periodo. | Pendiente |
| Fecha de finalización | Simular todo el último día en Bogotá | Se selecciona hasta las 23:59:59.999 de Bogotá; días restantes 0. | Pendiente |
| Día posterior | Simular medianoche Bogotá del día siguiente | Deja de seleccionarse. | Pendiente |
| Ausencia total | Sin publicados dentro de la ventana | Estado informativo, estadísticas/listado vacíos y cero jobs. | Pendiente |
| Publicados no solapados | Histórico, vigente y futuro publicados | Solo se usa el vigente. | Pendiente |
| Anomalía solapada | Dos publicados incluyen hoy | 409 `EVAL_PERIOD_CURRENT_AMBIGUOUS`; no se muestran datos arbitrarios ni se llama `addDailyReminderJobs`. | Pendiente |
| Recordatorios sin vigente | Invocar productor con stub de cola | Retorna `[]`; cola recibe cero jobs. | Pendiente |
| Preservación histórica | Comparar conteos/IDs antes y después de consultas | Ningún periodo, snapshot, evaluación, respuesta o resultado cambia. | Pendiente |

### Riesgos y pendientes

1. El dashboard general y su drill-down siguen expuestos a selección arbitraria mientras no implementen `periodId` explícito en P3. No afectan el flujo corregido de «Mis Evaluaciones» ni el scheduler.
2. Resumen y listado son dos solicitudes HTTP; usan la misma regla determinista, pero un cambio de día exactamente entre ambas puede producir legítimamente un estado distinto. La interfaz solo solicita el listado después de resolver el resumen y el backend vuelve a validar vigencia.
3. La protección por múltiples candidatos depende de que todos los consumidores operativos usen el servicio central. El inventario actual no encontró otros consumidores operativos dentro del módulo.
4. Las pruebas aisladas no sustituyen una validación integrada de Prisma/PostgreSQL `@db.Date`, reloj del host, scheduler, Redis y cola. Esa prueba debe usar ambiente aislado y dobles para notificaciones.
5. No se requiere migración para P1.3. No se añadió constraint de solapamiento ni se corrigieron datos existentes.

P1.3 no implementa selector histórico, filtros del dashboard, P1.4 ni cambios de publicación. P1.1 y P1.2 conservan íntegramente sus reglas aprobadas.
