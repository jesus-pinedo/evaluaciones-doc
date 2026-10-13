# Contexto funcional — Módulo de Evaluación de Desempeño (HF-011)

**Proyecto:** PROING — Evaluación de Desempeño  
**Versión del documento:** 0.1 (línea base para documentación en Codex)  
**Fecha de revisión documental:** 2026-10-13  
**Alcance:** descripción del dominio y de los requisitos, no auditoría de implementación.

> **Cómo leer este documento:** `[REQ]` identifica una regla del requerimiento original, `[PLAN]` una especificación contenida en la descomposición técnica, `[HIST]` algo discutido u observado anteriormente en el proyecto, y `[POR VALIDAR]` una decisión no demostrada. Esta distinción evita convertir hipótesis de implementación en requisitos de negocio.

## 1. Propósito y objetivo del módulo

El módulo administra el **proceso anual de evaluación de desempeño** de los colaboradores de PROING. Gestión Humana define el periodo y la parametrización de la evaluación; los jefes inmediatos evalúan al personal a cargo mediante el formulario **HF-011**; el sistema notifica a los involucrados y ofrece herramientas de seguimiento, consolidación y reimpresión. [REQ]

Los objetivos de negocio son:

- Estandarizar las preguntas, los criterios y las opciones de respuestas de la evaluación.
- Permitir a cada evaluador consultar sus colaboradores, incluso si pertenecen a distintos proyectos o centros de costo.
- Ayudar a que las evaluaciones se completen dentro del plazo definido por Gestión Humana.
- Dar visibilidad a Gestión Humana y Gerencia sobre el avance y los resultados consolidados.
- Mantener evidencia consultable de las evaluaciones enviadas.

**Límite de alcance:** este documento describe evaluaciones de desempeño y su relación con la asignación de jefe inmediato. No define por sí mismo procesos de nómina, sanciones, aumentos salariales u otros procesos de Gestión Humana. [REQ]

## 2. Actores y responsabilidades

| Actor | Responsabilidades funcionales | Fuente |
|---|---|---|
| Gestión Humana (incluido su director/administrador) | Definir periodo, administrar parámetros, asegurar información de jefes inmediatos y consultar avance/resultados. | [REQ] |
| Evaluador / jefe inmediato | Revisar equipo a cargo; diligenciar, guardar y enviar evaluaciones HF-011; consultar o reimprimir resultados según permisos. | [REQ] |
| Colaborador evaluado | Recibir notificación una vez enviada su evaluación. | [REQ] |
| Gerencia | Consultar reportes de seguimiento y compilados. | [REQ] |
| Sistema / procesos automáticos | Enviar avisos de creación, inicio, vencimiento/pendientes y notificación posterior al envío. | [REQ] |
| Administrador del módulo | Gestionar configuración y permisos según diseño de roles. | [PLAN] |

**Advertencia:** los roles técnicos `EVAL_ADMIN`, `EVAL_HR`, `EVAL_MANAGER` y `EVAL_READONLY` son una **propuesta del plan técnico**, no un catálogo confirmado de roles productivos. [PLAN]

## 3. Conceptos y entidades de negocio

- **Periodo de evaluación:** ventana con fecha de inicio y fecha de fin, definida por Gestión Humana para la evaluación anual. [REQ]
- **Evaluador:** jefe inmediato responsable de evaluar a uno o varios colaboradores. [REQ]
- **Evaluado:** colaborador que forma parte del personal a cargo del evaluador. [REQ]
- **Relación jefe–colaborador:** información administrada en la plataforma, con referencia al formato **HF-002**. Es crítica para generar las asignaciones de evaluación. [REQ]
- **Evaluación HF-011:** registro por colaborador y proceso, con preguntas y respuestas, fortalezas, debilidades y compromisos. [REQ]
- **Tema o grupo:** agrupación configurable de preguntas del formulario. [REQ]
- **Clasificación:** nivel/categoría configurable usado para analizar resultados. Una debilidad puede asociarse a múltiples clasificaciones. [REQ]
- **Fortaleza:** opción configurable que el evaluador puede seleccionar, pudiendo elegir varias. [REQ]
- **Debilidad:** opción configurable que el evaluador puede seleccionar, pudiendo elegir varias. [REQ]
- **Compromiso:** opción configurable asociada a debilidades; se pueden seleccionar varios. [REQ]
- **Escala de puntuación:** catálogo previsto para registrar valores de las preguntas. [PLAN]
- **Proyecto / centro de costo:** dimensiones usadas para identificar y filtrar personal y resultados. [REQ]
- **Snapshot del periodo:** copia de la configuración de catálogos al crear un periodo para preservar su versión histórica. Esta es una **decisión de diseño prevista**, no un requisito textual del documento original. [PLAN]

## 4. Ciclo funcional de un periodo

### 4.1 Configuración previa

1. Gestión Humana parametriza preguntas, grupos, clasificaciones, fortalezas, debilidades, compromisos y sus asociaciones. [REQ]
2. Gestión Humana crea un periodo de evaluación con fechas de inicio y fin. [REQ]
3. El sistema informa automáticamente a Gestión Humana que debe revisar la asociación de jefes inmediatos, en especial en HF-002, **antes del inicio** del proceso. [REQ]
4. El plan técnico plantea copiar los catálogos como snapshots al crear el periodo para que modificaciones posteriores no cambien evaluaciones históricas. [PLAN]

### 4.2 Inicio y ejecución

1. Al iniciar el periodo, los jefes inmediatos reciben una comunicación con el personal pendiente a evaluar. [REQ]
2. El evaluador ingresa al módulo y visualiza el personal bajo su responsabilidad de todos los proyectos o centros de costo correspondientes. [REQ]
3. El sistema presenta un tutorial visual del proceso. [REQ]
4. Para cada colaborador, el evaluador abre HF-011 y selecciona/responde los ítems configurados. [REQ]
5. El evaluador selecciona una o más fortalezas, debilidades y compromisos. [REQ]
6. Guarda y envía la evaluación; a continuación se genera notificación para el evaluado y se permite reimprimirla. [REQ]
7. El evaluador repite el proceso con los demás colaboradores de su equipo. [REQ]

### 4.3 Seguimiento y cierre

1. Mientras transcurre el periodo, el sistema envía **alertas diarias** a evaluadores con pendientes, fecha de finalización y tiempo restante. [REQ]
2. Gestión Humana y Gerencia consultan avance y pendientes. [REQ]
3. Gestión Humana y Gerencia consultan resultados consolidados en general o filtrados por proyecto. [REQ]
4. El plan técnico contempla una operación explícita de cierre y bloqueo de edición fuera del estado permitido. Los detalles exactos de cierre, reapertura, vencimiento automático y extensiones no están definidos en el requerimiento original. [PLAN] [POR VALIDAR]

## 5. Flujo detallado del evaluador (UC-002)

**Precondiciones:** existe un periodo habilitado y el usuario tiene colaboradores asignados. [REQ]

| Paso | Acción | Resultado esperado |
|---|---|---|
| 1 | Iniciar sesión y entrar al módulo | Se reconoce al evaluador y el periodo aplicable. |
| 2 | Consultar listado de equipo | Aparecen colaboradores a cargo, incluso de distintos proyectos/centros de costo. |
| 3 | Consultar guía/tour | Se explican los pasos necesarios para evaluar. |
| 4 | Elegir un colaborador | Se abre formulario HF-011 con sus preguntas y catálogos. |
| 5 | Responder y seleccionar opciones | Se registran respuestas, fortalezas, debilidades y compromisos. |
| 6 | Guardar/enviar | Se persiste la evaluación y se registra su envío. |
| 7 | Consultar/reimprimir | Se dispone de una vista final para impresión. |
| 8 | Continuar con el equipo | Se repite el proceso por colaborador. |

La posibilidad de **guardar borradores** y distinguir **borrador/enviada** está prevista expresamente en el plan técnico, pero no se detalla en el procedimiento del requerimiento original. [PLAN]

## 6. Configuración funcional

| Catálogo | Objetivo | Reglas sustentadas |
|---|---|---|
| Periodo | Definir inicio y final de campaña | Fechas configurables por Gestión Humana. [REQ] |
| Clasificaciones | Organizar análisis del desempeño | Debilidades con más de dos clasificaciones posibles. [REQ] |
| Temas / grupos | Estructurar HF-011 | Preguntas agrupables y configurables. [REQ] |
| Preguntas | Definir ítems a valorar | Deben ser configurables. [REQ] |
| Escala | Puntuar preguntas | Parametrización sugerida en plan técnico. [PLAN] |
| Fortalezas | Opciones de atributos positivos | Listado configurable; selección múltiple por evaluación. [REQ] |
| Debilidades | Opciones de oportunidades de mejora | Listado configurable; selección múltiple por evaluación. [REQ] |
| Compromisos | Opciones de acciones acordadas | Listado configurable, asociado a debilidades; selección múltiple. [REQ] |

No se encuentra una fórmula cerrada de calificación, un porcentaje de ponderación por grupo, mínimos/máximos obligatorios de selecciones ni umbrales de aprobación en el requerimiento base. [POR VALIDAR]

## 7. Reglas de negocio documentadas

| ID | Regla | Evidencia |
|---|---|---|
| RN-001 | Gestión Humana define las fechas del periodo anual. | [REQ] UC-001 |
| RN-002 | Tras crear un periodo se notifica a Gestión Humana para revisar/actualizar jefes inmediatos en HF-002 antes del inicio. | [REQ] UC-001 |
| RN-003 | Al iniciar el periodo se informa a los jefes sobre personal pendiente; durante el periodo hay comunicación diaria. | [REQ] UC-001 / UC-003 |
| RN-004 | Cada evaluador visualiza colaboradores de todos sus proyectos o centros de costo. | [REQ] requisitos funcionales |
| RN-005 | El formulario HF-011 usa preguntas y catálogos parametrizables. | [REQ] configuración |
| RN-006 | Se admiten varias fortalezas, debilidades y compromisos por evaluación. | [REQ] procedimiento |
| RN-007 | Cada compromiso configurable debe estar asociado a una debilidad. | [REQ] configuraciones |
| RN-008 | Las debilidades pueden relacionarse con más de dos clasificaciones para el análisis final. | [REQ] configuración |
| RN-009 | Si un colaborador tiene evaluación pendiente durante el periodo, el cambio de su jefe inmediato debe advertirse y bloquearse. | [REQ] requisitos funcionales |
| RN-010 | Durante el periodo se envían alertas diarias con vencimiento, pendientes y tiempo restante. | [REQ] UC-003 |
| RN-011 | Cuando se guarda y envía una evaluación se notifica al evaluado y se habilita la reimpresión. | [REQ] procedimiento |
| RN-012 | Gestión Humana y Gerencia consultan realizadas/pendientes y compilados generales o por proyecto. | [REQ] UC-004 / UC-005 |
| RN-013 | Evitar solapamiento de periodos activos; conservar snapshots de configuración al crear el periodo. | [PLAN] B.1/B.2 |
| RN-014 | Confirmar pertenencia del evaluado al equipo y vigencia del periodo antes de registrar respuestas. | [PLAN] C.3 |
| RN-015 | Unicidad de evaluación por combinación periodo, evaluador y evaluado. | [PLAN] C.1 |

**Distinción importante:** RN-013, RN-014 y RN-015 son decisiones propuestas en `tareas.md`; todavía requieren contraste con la implementación y eventual aprobación funcional.

## 8. Notificaciones y eventos

| Evento | Destinatario | Contenido mínimo expresado en fuentes | Momento |
|---|---|---|---|
| Creación de periodo | Gestión Humana | Aviso y recordatorio de revisar HF-002 / jefes inmediatos; fechas de periodo en el plan. | Tras crear el periodo, antes de iniciar. [REQ] [PLAN] |
| Inicio de periodo | Jefes inmediatos | Inicio del proceso y personal pendiente a evaluar. | Desde el inicio; el requerimiento pide envío diario. [REQ] |
| Recordatorio de pendientes | Evaluadores con pendientes | Fecha final, tiempo restante y personal pendiente; avance agregado en plan. | Diariamente durante la ventana. [REQ] [PLAN] |
| Envío de HF-011 | Evaluado | Notificación de la evaluación enviada. | Después del envío. [REQ] |

**No especificado:** hora de envío, zona horaria, destinatarios de copia, política de reintentos, tratamiento de correos ausentes, desduplicación y contenido exacto de plantillas. [POR VALIDAR]

## 9. Reportes y consultas

### 9.1 Seguimiento — UC-004

Permite a Gestión Humana y Gerencia identificar cuáles empleados ya fueron evaluados y cuáles siguen pendientes. [REQ]

El plan técnico amplía esta definición con filtros por periodo/proyecto/evaluador/centro de costo, cantidades, porcentaje de avance, detalle por evaluador y exportación a Excel/CSV. [PLAN]

### 9.2 Compilados — UC-005

Permite consultar las evaluaciones **en general** y en vista **filtrada por proyecto**. [REQ]

El plan técnico propone agregaciones por clasificaciones, grupos/proyectos y distribución de puntuaciones, con gráficas y exportaciones. [PLAN]

### 9.3 Evaluación individual

El evaluador puede **reimprimir** una evaluación final. [REQ] El plan técnico contempla vista de impresión en navegador y posible PDF futuro. [PLAN]

### 9.4 Cálculo de resultados

El documento fuente **no define** matemáticamente cómo calcular calificaciones individuales, por grupo, totales o compilados. [POR VALIDAR]

**Contexto histórico a comprobar:** en conversaciones previas se revisaron resultados por grupo y se señaló que el total mostrado en la web no aplicaba ponderaciones. Esto **no constituye una regla funcional aprobada** ni prueba el comportamiento del código actual. Antes de modificar reportes se debe levantar y probar la fórmula vigente con ejemplos concretos. [HIST]

## 10. Estados y fechas

- El requerimiento hace referencia a periodo delimitado por fecha inicio/fin y a evaluaciones realizadas/pendientes. [REQ]
- El plan técnico propone estados **Borrador, Activo, Cerrado** para periodos y **borrador, enviado** para evaluaciones. [PLAN]
- Históricamente se han visto etiquetas de métricas como `completed`, `inProgress`, `pending` y `accepted` en respuestas de API; debe identificarse qué entidad y transición representa cada una. [HIST] [POR VALIDAR]
- No está aclarado si un periodo vence automáticamente por fecha, si un cron actualiza un estado persistido, si se puede reabrir, cómo manejar zonas horarias o si se aceptan evaluaciones luego del vencimiento. [POR VALIDAR]

> **No asumir** que `accepted` significa evaluación aprobada, ni que equivale a acuse de notificación: debe revisarse la implementación.

## 11. Integración con otros módulos

### 11.1 Sistema legado y autenticación

El requisito original describe un módulo dentro de sistema web PHP. El plan de implementación que evolucionó posteriormente define frontend React, backend Node, PostgreSQL/Prisma, acceso desde el sistema PHP y SSO con intercambio de token temporal respaldado por Redis. [REQ] [PLAN]

**Contexto de proyecto reportado:** el módulo se está desarrollando en un frontend y API compartidos con otros módulos (`sisproing-app` y `api`), junto con el legado `sisproing`. Se debe verificar en código las rutas exactas del SSO, sus permisos, la configuración de Redis y el contrato de autenticación. [HIST]

### 11.2 Fuentes de personal / asignación de jefe

La fuente funcional de asignaciones es la información de jefe inmediato mantenida por Gestión Humana (HF-002). [REQ] En conversaciones del proyecto se ha usado `EmployeeManager`/relaciones de jefes y una condición de empleado activo; nombres y filtros efectivos deben verificarse contra Prisma, SQL y servicios. [HIST]

### 11.3 Notificaciones y proceso programado

El requerimiento establece comunicaciones diarias. El plan técnico sugiere cron/job y servicio de correo desde Node. [REQ] [PLAN] Los detalles reales de colas, cron, plantillas y registro de envíos requieren lectura de código. [POR VALIDAR]

## 12. Criterios funcionales de aceptación iniciales

1. Gestión Humana puede configurar un periodo con inicio/fin y recibir aviso de revisión HF-002. [REQ]
2. Un evaluador puede ver todos sus colaboradores asignados, aunque pertenezcan a diferentes proyectos/centros de costo. [REQ]
3. El evaluador encuentra un tutorial y puede abrir HF-011 de un colaborador. [REQ]
4. El formulario utiliza preguntas configurables y permite seleccionar múltiples fortalezas, debilidades y compromisos según listas definidas. [REQ]
5. Cuando termina y envía la evaluación, el evaluado recibe notificación y el evaluador puede reimprimir. [REQ]
6. Los responsables reciben alertas diarias con sus pendientes y el vencimiento. [REQ]
7. El sistema bloquea el cambio de jefe inmediato cuando hay evaluación pendiente bajo las condiciones del requerimiento. [REQ]
8. Gestión Humana/Gerencia pueden distinguir evaluaciones realizadas/pendientes y obtener compilado general o por proyecto. [REQ]

Estos criterios son un punto de partida. Para llevarlos a pruebas automatizadas faltan definiciones de estados, datos válidos, criterios de cálculo, permisos y casos límite.

## 13. Alcance NO sustentado / decisiones abiertas

El requerimiento original no especifica los siguientes elementos, por lo que **no deben convertirse automáticamente en requisitos**:

- Antigüedad mínima del colaborador (por ejemplo, tres meses) para participar.
- Peso o ponderación de temas/grupos, fórmula del porcentaje global y redondeo.
- Restricción exacta de una evaluación por persona si cambia de evaluador durante el periodo.
- Regla de asignación/snapshot del personal cuando se crea el periodo.
- Flujo detallado de Gestión Humana (el propio documento original lo marca como faltante).
- Proceso de aceptación por el evaluado y significado de aceptación o confirmación de notificación.
- Estados finales y procedimiento de cierre, reapertura, vencimiento y prórroga.
- Política de confidencialidad, visibilidad y retención de evaluaciones.
- Contenido exacto del formulario HF-011: los anexos están enlazados desde el PDF, pero sus datos no forman parte de esta línea base documental.

Consultar el registro detallado en [`pendientes-de-validacion.md`](pendientes-de-validacion.md).

## 14. Matriz de trazabilidad

| Caso de uso del requerimiento | Secciones de este documento | Paquetes del plan |
|---|---|---|
| UC-001 Configuración y notificación del periodo | 4, 6, 7, 8, 10 | A, B, D |
| UC-002 Realizar evaluación HF-011 | 2, 3, 5, 6, 7 | C, F |
| UC-003 Alertas automáticas | 4, 7, 8 | D |
| UC-004 Reporte de seguimiento | 9.1, 12 | E |
| UC-005 Reporte compilado | 9.2, 9.4, 12 | E |
| Integración y control de acceso | 11 | F (complemento técnico) |

## 15. Documentos de origen

1. **Requerimiento original:** `1. Requerimiento de Módulo de Evaluación de Desempeño (1).pdf`, secciones Descripción, Alcance, UC-001 a UC-005, Procedimiento, Requisitos Funcionales y Configuraciones del Sistema.
2. **Plan de tareas:** `tareas.md`, capas A (Catálogos), B (Periodos), C (HF-011), D (Alertas), E (Reportes) y F (Integración).
3. **Estimaciones:** `estimaciones.md`, estimación de esfuerzo por capa, útil para priorización; **no es prueba de estado real**.
4. **Conversaciones anteriores del proyecto:** observaciones sobre esquema Prisma, relación jefe–evaluado, notificaciones, reportes y SSO; evidencias históricas que precisan confirmación con repositorios.

**Anexos externos:** el PDF menciona modelos de análisis, HF-011 y un archivo de compilados. Estos anexos no fueron revisados en esta versión; pueden contener requisitos adicionales.
