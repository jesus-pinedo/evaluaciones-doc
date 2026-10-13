# Seguridad y SSO

## Flujo diseñado

1. El legado entrega `?sso_token=` o `?code=` al frontend.
2. `AuthProvider` llama `POST /api/auth/exchange` con `code` y `state:null`.
3. `auth.service.js#exchangeCode` lee `sisproing:sso:code:<code>` en Redis.
4. El API firma un JWT con `sub`, nombre, correo, roles y tenant; duración por defecto 12 horas.
5. React guarda JWT, usuario y expiración local (8 horas por defecto) en `localStorage`.
6. Los permisos `module#action` controlan rutas y menú del frontend.
7. `apiFetch` agrega `Authorization: Bearer <token>`.

## Controles encontrados

- `authenticate` valida formato Bearer, firma/expiración JWT y asigna `req.user.id` desde `sub`.
- `ProtectedRoute` evalúa permisos exactos recibidos en roles.
- La página pública de acuse evita redirección a login.
- El token de acuse exige scope `ACKNOWLEDGE_RECEIPT` y expiración.

## Hallazgos críticos

1. **Middleware desactivado.** `api/src/app.js` contiene `//app.use(authenticate);`. Los routers de evaluaciones no añaden middleware. El JWT enviado no se procesa.
2. **Sin autorización backend.** No se verifican roles/permisos para configuración, periodos o reportes.
3. **Sin autorización por recurso.** Detalle, PDF, borrador y finalización no comprueban que `userId === evaluatorId` ni acceso administrativo.
4. **Identidad por query.** Overview y lista aceptan `evaluatorId` de query si falta `req.user`; permite suplantación si la ruta fuera operativa sin autenticación.
5. **Operaciones incompatibles con middleware desactivado.** Guardar/finalizar devuelven 401 al no existir `req.user`; altas de catálogos pueden fallar por `createdBy` nulo.
6. **Código SSO reutilizable.** La eliminación de la clave Redis está comentada; el código no es de un solo uso mientras exista.
7. **`state` no validado.** El controlador recibe `state`, pero el servicio lo ignora.
8. **Secretos con comportamiento divergente.** El middleware usa un fallback de desarrollo para `JWT_SECRET`; el firmante SSO usa la variable sin fallback. No se inspeccionó configuración efectiva.
9. **Ruta de impresión pública.** React la monta fuera de `ProtectedRoute` y el API permite lectura por ID.
10. **Limpieza incompleta de URL.** El frontend acepta `sso_token` o `code`, pero después del intercambio solo elimina `sso_token`; si se usa `code`, permanece en la URL y puede reintentarse en una recarga.

## Roles y permisos

El frontend no impone nombres fijos de rol; aplana `user.roles[].permissions`. Permisos observados:

- `employee_evaluations.evaluations#view/create/edit`;
- `employee_evaluations.configurations#view/create/edit/delete`;
- `employee_evaluations.periods#view/create/edit/delete`;
- `employee_evaluations.reports#view`.

Las rutas usan solo un subconjunto: `evaluations#view`, `configurations#view/edit` y `reports#view`. No hay verificación backend equivalente. Los roles del mock son ejemplos de desarrollo, no catálogo productivo.

## Riesgo de datos

Por tratarse de evaluaciones de desempeño, los endpoints exponen datos laborales y resultados individuales. Hasta activar autenticación y autorización por recurso, la confidencialidad no está asegurada por la implementación inspeccionada.

## Pendiente del legado

No se verificó quién genera la clave Redis, su TTL, entropía, asociación a `state`, origen permitido ni permisos reales. Esa revisión requiere el repositorio PHP y queda fuera de esta fase.
