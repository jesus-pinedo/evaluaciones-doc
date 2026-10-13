# Arquitectura del módulo

## Componentes

| Capa | Tecnología | Responsabilidad |
|---|---|---|
| UI | React 19, React Router, React Hook Form | Configuración, equipo, formulario, PDF visual, acuse y dashboard. |
| API | Express 5 | Rutas HTTP, orquestación y validaciones de aplicación. |
| Datos | Prisma 6 + PostgreSQL | Catálogos, snapshots, periodos, asignaciones, respuestas y acuses. |
| SSO | Redis + JWT | Intercambio de código legado por sesión interna. |
| Asíncrono | BullMQ + Redis | Correos de resultado y recordatorios. |
| Programación | node-cron | Disparo diario de recordatorios. |
| Documentos | Puppeteer | Render de la ruta React de impresión a PDF. |
| Correo | MJML/Handlebars + Nodemailer | Plantillas HTML y entrega SMTP. |

## Flujo principal

1. El legado genera un código temporal en Redis y navega al frontend. El frontend lo intercambia por JWT y usuario.
2. React llama al API con `Authorization: Bearer` mediante `apiFetch`.
3. El API monta `/api/employee-evaluations`, usa servicios Prisma y devuelve DTO JSON.
4. Al publicar, PostgreSQL recibe en una transacción snapshots y evaluaciones.
5. Al finalizar, el API actualiza la evaluación y agrega un job a Redis.
6. El worker crea PDF/HTML y usa SMTP.
7. El enlace público vuelve al frontend y este registra el acuse en un endpoint público.

[RIESGO] El paso 2 envía JWT, pero `api/src/app.js` tiene comentado `app.use(authenticate)`. Por tanto, el flujo de identidad se corta antes de los controladores.

## Límites e integraciones

- Personal, jerarquía y proyectos proceden de modelos de Gestión Humana compartidos.
- No se inspeccionó código PHP. El contrato del código SSO se dedujo solo desde Node/React.
- No se verificó infraestructura productiva, despliegue de worker, contenido de Redis, SMTP ni datos PostgreSQL.

## Evidencia principal

- Backend: `api/src/app.js`, `src/routes/index.js`, `src/modules/employee-evaluations/`, `src/modules/notifications/` en archivos específicos de evaluaciones.
- Datos: `api/prisma/modules/evaluations.prisma` y fragmentos relacionados de `human-resources.prisma`.
- Frontend: `sisproing-app/src/modules/employee-evaluations/`, `src/auth/`, `src/lib/httpClient.ts`, `src/routes/ProtectedRoute.tsx`.
