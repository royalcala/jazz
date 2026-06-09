# React + Jazz: roadmap de temas (punto por punto)

Este documento es el plan de trabajo para cubrir los temas clave de Jazz en React sin omitir detalles de produccion.

## 1) Orden recomendado

1. Branches y entornos (aislamiento de datos)
2. Schema de datos
3. Data patterns (patrones de modelado)
4. Access control y permissions
5. CRUD reactivo en UI
6. Auth e identidad por camino (localfirst/hybrid/betterauth)
7. Migraciones y evolucion de schema
8. Publicacion de catalogo (schema/migrations/permissions)
9. Operacion de auth server (JWT/JWKS) en hybrid/betterauth
10. Recuperacion de identidad (recovery phrase/passkey)
11. Produccion y operacion continua

## 2) Detalle por etapa

### Etapa 1: Branches y entornos

Objetivo:

1. Definir estrategia de aislamiento de datos por ambiente y linea de trabajo.

Entregables:

1. Convencion de `env` y `userBranch` por entorno (local, staging, prod).
2. Politica de uso de branches para pruebas, QA y preproduccion.
3. Checklist para evitar mezcla de datos entre ambientes/branches.

Notas clave:

1. En Jazz, una rama efectiva combina `env`, `schemaHash` y `userBranch`.
2. En esta etapa se fija primero `env` y `userBranch`; `schemaHash` lo aporta Jazz automaticamente.
3. `env` y `userBranch` estan totalmente aislados (no se mezclan en query).

Referencias:

1. [docs/content/docs/concepts/branches.mdx](../docs/content/docs/concepts/branches.mdx)
2. [docs/content/partials/create-jazz-client-reference.mdx](../docs/content/partials/create-jazz-client-reference.mdx)
3. [examples/docs/todo-client-localfirst-react/src/branch-snippets.tsx](../examples/docs/todo-client-localfirst-react/src/branch-snippets.tsx)
4. Guia detallada de branches y entornos: [react-branches-entornos.md](./react-branches-entornos.md)

### Etapa 2: Schema de datos

Objetivo:

1. Diseñar tablas, tipos y relaciones.

Entregables:

1. schema.ts versionado en repo.
2. Validacion basica de consultas de lectura/escritura.

Referencias:

1. [starters/react-localfirst/schema.ts](../starters/react-localfirst/schema.ts)
2. [starters/react-hybrid/schema.ts](../starters/react-hybrid/schema.ts)
3. [starters/react-betterauth/schema.ts](../starters/react-betterauth/schema.ts)
4. Guia detallada de schema y flujo: [react-schema-datos-y-flujo.md](./react-schema-datos-y-flujo.md)

### Etapa 3: Data patterns

Objetivo:

1. Elegir patrones de modelado recurrentes antes de cerrar reglas de acceso.

Entregables:

1. Patron principal seleccionado (nested, collaborative list, etc.).
2. Patrones secundarios para casos de sharing/group ownership.
3. Ejemplos de insercion/lectura acordes al patron elegido.

Referencias:

1. [docs/content/docs/recipes/data-patterns/nested-data.mdx](../docs/content/docs/recipes/data-patterns/nested-data.mdx)
2. [docs/content/docs/recipes/data-patterns/real-time-collaborative-list.mdx](../docs/content/docs/recipes/data-patterns/real-time-collaborative-list.mdx)
3. [docs/content/docs/concepts/local-first-data-model.mdx](../docs/content/docs/concepts/local-first-data-model.mdx)

### Etapa 4: Access control y permissions

Objetivo:

1. Definir control de acceso por operacion (read/insert/update/delete) con reglas de negocio reales.

Entregables:

1. permissions.ts con reglas de ownership y/o colaboracion.
2. Casos de prueba de acceso permitido y denegado.
3. Reglas de claims/sesion cuando aplique auth externa.

Referencias:

1. [docs/content/docs/auth/permissions.mdx](../docs/content/docs/auth/permissions.mdx)
2. [docs/content/docs/recipes/access-control/user-owned-data.mdx](../docs/content/docs/recipes/access-control/user-owned-data.mdx)
3. [docs/content/docs/recipes/access-control/group-permissions.mdx](../docs/content/docs/recipes/access-control/group-permissions.mdx)
4. [docs/content/docs/recipes/access-control/shared-access.mdx](../docs/content/docs/recipes/access-control/shared-access.mdx)
5. [starters/react-localfirst/permissions.ts](../starters/react-localfirst/permissions.ts)

### Etapa 5: CRUD reactivo en UI

Objetivo:

1. Implementar UX de lectura/escritura sobre Jazz.

Entregables:

1. Componentes CRUD funcionales.
2. Manejo de estados de carga/error en acciones.

Referencias:

1. [starters/react-localfirst/src/todo-widget.tsx](../starters/react-localfirst/src/todo-widget.tsx)
2. [starters/react-hybrid/src/todo-widget.tsx](../starters/react-hybrid/src/todo-widget.tsx)
3. [starters/react-betterauth/src/todo-widget.tsx](../starters/react-betterauth/src/todo-widget.tsx)

### Etapa 6: Auth e identidad por camino

Objetivo:

1. Elegir y configurar flujo de identidad correcto para producto.

Entregables:

1. Provider de app alineado al camino seleccionado.
2. Documentacion de por que se eligio ese camino.

Referencias:

1. [react-matriz-comparativa-3-caminos.md](./react-matriz-comparativa-3-caminos.md)
2. [starters/react-localfirst/src/main.tsx](../starters/react-localfirst/src/main.tsx)
3. [starters/react-hybrid/src/main.tsx](../starters/react-hybrid/src/main.tsx)
4. [starters/react-betterauth/src/main.tsx](../starters/react-betterauth/src/main.tsx)
5. [docs/content/docs/auth/authentication.mdx](../docs/content/docs/auth/authentication.mdx)
6. [docs/content/docs/auth/sessions.mdx](../docs/content/docs/auth/sessions.mdx)
7. [docs/content/docs/auth/local-first-auth.mdx](../docs/content/docs/auth/local-first-auth.mdx)

### Etapa 7: Migraciones y evolucion de schema

Objetivo:

1. Evolucionar schema sin romper compatibilidad de datos entre versiones.

Entregables:

1. Proceso claro para detectar cambios estructurales.
2. Migraciones revisadas cuando haya transformaciones de filas.
3. Registro de hashes from/to y edge publicado.

Notas clave:

1. No todo cambio requiere archivo de migracion manual.
2. Si no hay transformaciones de filas, se puede empujar sin stub revisado.
3. Si hay transformaciones de filas, usar defineMigration y pushMigration.
4. Aqui se retoma branches en modo versionado: cada schema hash vive en su propia rama y Jazz compone lectura entre versiones via lenses/migraciones.

Referencias:

1. [packages/jazz-tools/src/migrations.ts](../packages/jazz-tools/src/migrations.ts)
2. [packages/jazz-tools/src/cli.ts](../packages/jazz-tools/src/cli.ts)
3. [examples/docs/todo-server-ts/docs/migrations-workflow.sh](../examples/docs/todo-server-ts/docs/migrations-workflow.sh)
4. [examples/docs/todo-server-ts/migrations/20260318-add-description-a01f5c72ec47-311995e9a178.ts](../examples/docs/todo-server-ts/migrations/20260318-add-description-a01f5c72ec47-311995e9a178.ts)

### Etapa 8: Publicacion de catalogo

Objetivo:

1. Publicar de forma consistente schema, migrations y permissions.

Entregables:

1. Pipeline reproducible de push/deploy.
2. Validacion de estado de permissions head.

Referencias:

1. [packages/jazz-tools/src/cli.ts](../packages/jazz-tools/src/cli.ts)
2. [packages/jazz-tools/src/dev/index.ts](../packages/jazz-tools/src/dev/index.ts)

### Etapa 9: Auth server y JWT/JWKS (si aplica)

Objetivo:

1. Operar auth seguro en hybrid/betterauth.

Entregables:

1. Config correcta de BETTER_AUTH_SECRET.
2. jwksUrl correcto en plugin.
3. Refresh de token validado en cliente.

Referencias:

1. [starters/react-hybrid/server/auth.ts](../starters/react-hybrid/server/auth.ts)
2. [starters/react-betterauth/server/auth.ts](../starters/react-betterauth/server/auth.ts)
3. [starters/react-hybrid/vite.config.ts](../starters/react-hybrid/vite.config.ts)
4. [starters/react-betterauth/vite.config.ts](../starters/react-betterauth/vite.config.ts)
5. [docs/content/docs/recipes/auth/auth-provider-integration.mdx](../docs/content/docs/recipes/auth/auth-provider-integration.mdx)
6. [docs/content/docs/recipes/auth/better-auth-adapter.mdx](../docs/content/docs/recipes/auth/better-auth-adapter.mdx)

### Etapa 10: Recuperacion de identidad

Objetivo:

1. Evitar perdida de acceso por borrado de storage o cambio de dispositivo.

Entregables:

1. UX de backup/restore en app.
2. Playbook de soporte para incidentes de acceso.

Referencias:

1. [react-localfirst-recovery-backup.md](./react-localfirst-recovery-backup.md)
2. [packages/jazz-tools/src/runtime/recovery-phrase.ts](../packages/jazz-tools/src/runtime/recovery-phrase.ts)
3. [packages/jazz-tools/src/runtime/passkey-backup.ts](../packages/jazz-tools/src/runtime/passkey-backup.ts)

### Etapa 11: Produccion y operacion continua

Objetivo:

1. Pasar de demo a operacion estable.

Entregables:

1. Sustituir memory adapter de Better Auth por DB persistente.
2. Gestion de secretos y rotacion.
3. Monitoreo basico y alertas de auth/sync.
4. Pruebas de regresion (incluyendo migraciones y permisos).

Referencias:

1. [starters/react-hybrid/README.md](../starters/react-hybrid/README.md)
2. [starters/react-betterauth/README.md](../starters/react-betterauth/README.md)
3. [starters/react-localfirst/README.md](../starters/react-localfirst/README.md)

## 3) Seguimiento sugerido

1. Marcar estado por etapa: pendiente, en curso, hecho.
2. Definir owner por etapa.
3. Definir criterio de salida por etapa.
