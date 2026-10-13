# Contexto funcional — HF-011

**Revisión:** 2026-10-13  
**Naturaleza:** requerimiento contrastado con inspección estática de código.

## Objetivo

[REQ] El módulo estandariza el proceso anual de evaluación de desempeño. Gestión Humana define el periodo y los catálogos; cada jefe diligencia HF-011 para su personal; el evaluado recibe el resultado; Gestión Humana y Gerencia siguen el avance y consultan consolidados.

[CÓDIGO] La implementación se distribuye entre `api/src/modules/employee-evaluations/` y `sisproing-app/src/modules/employee-evaluations/`. La persistencia se modela en `api/prisma/modules/evaluations.prisma` y reutiliza empleados, jefes y proyectos de `api/prisma/modules/human-resources.prisma`.

## Actores y alcance implementado

| Actor | Intención funcional | Estado comprobado |
|---|---|---|
| Gestión Humana / configurador | Mantener catálogos, crear/publicar periodos y consultar avance. | **Parcial:** hay pantallas y CRUD, publicación y dashboard; faltan autorización backend, cierre/reapertura formal y consolidados de resultados. |
| Evaluador / jefe | Ver equipo, diligenciar, guardar, finalizar e imprimir. | **Parcial:** el flujo está codificado, pero la lista depende de una identidad que el API no carga porque el middleware está desactivado. |
| Evaluado | Recibir resultado y confirmar recepción. | **Implementado en código:** correo con PDF y enlace firmado, página pública e idempotencia; la operación real del worker/SMTP no fue verificada. |
| Gerencia / consulta | Ver avance y resultados. | **Parcial:** existe dashboard de avance, sin segregación backend por rol y sin reporte consolidado de puntajes. |
| Sistema | Generar asignaciones, recordatorios y notificaciones. | **Parcial:** asignación al publicar y cron diario existen; no se encontraron avisos de creación/inicio de periodo. |

## Conceptos implementados

- **Periodo:** nombre, fechas y `isActive`. `isActive=false` se interpreta como borrador; al publicar pasa a `true`.
- **Snapshot:** copia por periodo de grupos, preguntas, fortalezas, debilidades, clasificaciones y escala. La clasificación copiada queda como texto de la debilidad.
- **Asignación/evaluación:** tupla periodo–evaluador–evaluado generada desde `EmployeeManager`.
- **Respuesta:** valor entero asociado a una pregunta snapshot.
- **Fortaleza:** selección múltiple de fortalezas snapshot.
- **Oportunidad de mejora:** selección múltiple de debilidades snapshot con compromiso de texto libre por selección.
- **Confirmación:** acuse del evaluado representado por `notificationReceived` y `notificationReceivedAt`.

## Ciclo observado

1. Se mantienen catálogos globales.
2. Se crea un periodo borrador.
3. Al publicar se validan catálogos no vacíos, se copian snapshots, se crean asignaciones y se marca publicado.
4. El evaluador consulta asignaciones, abre el formulario y el sistema autoguarda con espera de dos segundos o permite guardado manual.
5. Al finalizar se comprueba que todas las preguntas tengan registro, se persiste puntaje/etiqueta, se marca `COMPLETED` y se encola un correo.
6. El evaluado recibe PDF y enlace de confirmación. El clic registra el acuse una sola vez.
7. El dashboard muestra avance global, por evaluador y por proyecto.
8. A las 08:00, si la configuración lo habilita, se encolan recordatorios para jefes con evaluaciones `PENDING` o `IN_PROGRESS`.

## Diferencias conceptuales importantes

- [REQ] compromisos configurables; [CÓDIGO] compromiso libre por debilidad.
- [REQ] una debilidad puede tener varias clasificaciones; [CÓDIGO] tiene una sola `classificationId` opcional.
- [REQ] avisos al crear e iniciar el periodo; [CÓDIGO] solo se hallaron recordatorio diario y notificación al completar.
- [REQ] consolidados generales/por proyecto; [CÓDIGO] solo avance por estado y proyecto, sin agregaciones de puntaje o clasificación.
- [PLAN] roles técnicos específicos; [CÓDIGO] el frontend consume permisos del SSO, pero el backend no los hace cumplir.

Para el detalle, consultar [reglas de negocio](reglas-negocio.md) y [diferencias](../auditoria/diferencias-requerimiento.md).
