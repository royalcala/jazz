# React + Jazz: Access control y permissions (Etapa 4)

Este documento aterriza la Etapa 4 del roadmap: convertir el modelo de datos en reglas reales de acceso.

## 1) Objetivo de la etapa

1. Definir quien puede leer, insertar, actualizar y eliminar cada tabla.
2. Asegurar enforcement server-side de las reglas.
3. Evitar huecos de seguridad antes de seguir con auth avanzada.

## 2) Explicacion simple

Piensa en `permissions.ts` como el guardia de la base:

1. Si no hay regla explicita para una operacion, en modo policy bundle compilado se niega.
2. Las reglas se evalúan por fila (row-level).
3. El cliente no decide seguridad; el servidor aplica las politicas.

## 3) Dónde vive y flujo de trabajo

1. `permissions.ts` vive junto a `schema.ts`.
2. Define politicas con `definePermissions(app, ...)`.
3. Valida antes de publicar.
4. Publica con deploy (no requiere migration si solo cambian permissions).

Comandos utiles:

```bash
pnpm dlx jazz-tools@alpha validate
pnpm dlx jazz-tools@alpha deploy <appId>
```

## 4) Patrones de reglas que vas a usar

### 4.1 Owner-only (simple)

1. Cada usuario solo puede tocar sus filas.
2. Usar `session.user_id` y/o `$createdBy`.

### 4.2 Membership + role (org/workspace)

1. El acceso se decide por membresia y rol en una tabla puente.
2. Ideal para productos con organizaciones/equipos.

### 4.3 Inherited access (allowedTo)

1. Entidades hijas heredan acceso de la entidad padre.
2. Evita duplicar reglas en jerarquias Project -> Task -> Comment.

## 5) Patron org multi-membership (usuarios en varias orgs)

Este es el patron para tu caso:

1. Un usuario puede pertenecer a muchas orgs.
2. Una org puede tener muchos usuarios.
3. Se modela con tabla puente de memberships (many-to-many).

Schema base recomendado:

```ts
import { schema as s } from "jazz-tools";

const schema = {
  organizations: s.table({
    name: s.string(),
  }),
  organizationMembers: s.table({
    orgId: s.ref("organizations"),
    user_id: s.string(),
    role: s.enum("admin", "writer", "reader"),
  }),
  projects: s.table({
    orgId: s.ref("organizations"),
    name: s.string(),
  }),
};

type AppSchema = s.Schema<typeof schema>;
export const app: s.App<AppSchema> = s.defineApp(schema);
```

Como se ve el many-to-many:

1. Usuario U1 con dos filas en `organizationMembers` para org A y org B.
2. Org A con varias filas en `organizationMembers` para U1, U2, U3.

## 6) Plantilla base de permissions para org/workspace

```ts
import { schema as s } from "jazz-tools";
import { app } from "./schema";

export default s.definePermissions(app, ({ policy, session, anyOf }) => {
  const isMember = (orgId: string) =>
    policy.organizationMembers.exists.where({ orgId, user_id: session.user_id });

  const hasRole = (orgId: string, role: "reader" | "writer" | "admin") =>
    policy.organizationMembers.exists.where({ orgId, user_id: session.user_id, role });

  // projects
  policy.projects.allowRead.where((project) => isMember(project.orgId));
  policy.projects.allowInsert.where((project) =>
    anyOf([hasRole(project.orgId, "writer"), hasRole(project.orgId, "admin")]),
  );
  policy.projects.allowUpdate.where((project) =>
    anyOf([
      hasRole(project.orgId, "writer"),
      hasRole(project.orgId, "admin"),
    ]),
  );
  policy.projects.allowDelete.where((project) =>
    anyOf([
      hasRole(project.orgId, "admin"),
    ]),
  );

  // organizations
  policy.organizations.allowRead.where((org) => isMember(org.id));
  policy.organizations.allowInsert.always();
  policy.organizations.allowUpdate.where((org) => hasRole(org.id, "admin"));
  policy.organizations.allowDelete.where((org) => hasRole(org.id, "admin"));

  // organizationMembers
  policy.organizationMembers.allowRead.where((member) => isMember(member.orgId));
  policy.organizationMembers.allowInsert.where((member) => hasRole(member.orgId, "admin"));
  policy.organizationMembers.allowUpdate.where((member) => hasRole(member.orgId, "admin"));
  policy.organizationMembers.allowDelete.where((member) =>
    anyOf([hasRole(member.orgId, "admin"), { user_id: session.user_id }]),
  );
});
```

Nota: ajusta los nombres de tablas/roles a tu schema real.

## 7) Alta de usuarios por org (flujo recomendado)

1. Crear org:
   - Insert en `organizations`.
   - Insert inmediato en `organizationMembers` con `role=admin` para el creador.
2. Agregar usuario a org:
   - Solo admin de esa org puede insertar una fila en `organizationMembers`.
3. Usuario en multiples orgs:
   - Tiene una fila por cada org donde participa.
4. Salir de org:
   - Puede borrar su propia fila de membership.

## 8) Casos minimos que debes probar

1. Usuario sin membresia no puede leer ni escribir.
2. Reader solo lee.
3. Writer/Admin editan contenido de otros segun regla.
4. Un usuario no puede escalar su propio rol sin permiso.
5. Un usuario puede pertenecer simultaneamente a dos orgs y ver solo los datos de cada una.

## 9) Errores comunes

1. Permitir `allowInsert.always()` donde debería haber validacion de pertenencia.
2. Basar todo en `$createdBy` cuando necesitas colaboracion por rol.
3. No probar updates de campos sensibles (ejemplo: `role`, `owner_id`).
4. Confiar en filtros del frontend como si fueran seguridad.
5. No crear la fila inicial de admin en la org al momento del bootstrap.

## 10) Checklist de salida de Etapa 4

1. Todas las tablas con politicas explicitas de read/insert/update/delete.
2. Casos permitidos y denegados cubiertos en pruebas.
3. Validacion local sin warnings criticos.
4. Permissions publicadas al entorno correcto.
5. Flujo de alta de miembros por org validado extremo a extremo.

## 11) Siguiente paso recomendado

Despues de cerrar esta etapa, sigue con Etapa 5 (CRUD reactivo):

1. Exponer UX segun permisos.
2. Mostrar acciones habilitadas/deshabilitadas por fila.
3. Manejar errores de autorizacion de forma amigable.

## 12) Referencias oficiales

1. [docs/content/docs/auth/permissions.mdx](../docs/content/docs/auth/permissions.mdx)
2. [docs/content/docs/recipes/access-control/user-owned-data.mdx](../docs/content/docs/recipes/access-control/user-owned-data.mdx)
3. [docs/content/docs/recipes/access-control/shared-access.mdx](../docs/content/docs/recipes/access-control/shared-access.mdx)
4. [docs/content/docs/recipes/access-control/group-permissions.mdx](../docs/content/docs/recipes/access-control/group-permissions.mdx)
5. [examples/docs/todo-server-ts/permissions.ts](../examples/docs/todo-server-ts/permissions.ts)
6. [examples/docs/todo-client-localfirst-react/src/group-permissions-snippets.tsx](../examples/docs/todo-client-localfirst-react/src/group-permissions-snippets.tsx)
