# React + Jazz: SDK backend (createJazzContext) y API de integraciones

Este tema explica como crear el archivo de backend SDK de Jazz y que operaciones conviene centralizar ahi para integraciones como email/notificaciones.

## 1) Que archivo crear

Recomendacion: crear un modulo dedicado tipo `server/jazz-context.ts` (o `src/server/jazz-context.ts`) y exportar desde ahi el contexto compartido.

Objetivo:

1. Inicializar Jazz una sola vez por proceso.
2. Evitar reconfiguracion por request.
3. Exponer handles con alcance correcto (`asBackend`, `forRequest`, etc.).

## 2) Plantilla base del archivo backend SDK

```ts
import { createJazzContext } from "jazz-tools/backend";
import { app } from "../schema";
import permissions from "../permissions";

const appId = process.env.JAZZ_APP_ID ?? process.env.VITE_JAZZ_APP_ID;
if (!appId) {
  throw new Error("Missing JAZZ_APP_ID (or VITE_JAZZ_APP_ID)");
}

const serverUrl = process.env.JAZZ_SERVER_URL ?? process.env.VITE_JAZZ_SERVER_URL;
const backendSecret = process.env.BACKEND_SECRET ?? process.env.JAZZ_BACKEND_SECRET;

export const jazzContext = createJazzContext({
  appId,
  app,
  permissions,
  driver: {
    type: "persistent",
    dataPath: process.env.JAZZ_DATA_PATH ?? "./data/jazz.db",
  },
  serverUrl,
  backendSecret,
  adminSecret: process.env.JAZZ_ADMIN_SECRET,
  jwksUrl: process.env.JAZZ_JWKS_URL,
  env: process.env.JAZZ_ENV ?? "prod",
  userBranch: process.env.JAZZ_USER_BRANCH ?? "main",
  allowLocalFirstAuth: false,
});

// Handle para trabajo del sistema (jobs, cron, webhooks internos)
export const dbBackend = jazzContext.asBackend();
```

Notas:

1. El backend SDK viene de `jazz-tools/backend`, no de un paquete separado para Hono.
2. Si usas Jazz server conectado (`serverUrl`), define `backendSecret`.
3. Si tu backend valida JWT externa, define `jwksUrl` (o `jwtPublicKey`, pero no ambos).

## 3) Que se puede hacer en ese archivo/contexto

### 3.1 Scopes de identidad y autorizacion

1. `context.asBackend()`:
   para trabajo del backend con permisos de sistema.
2. `await context.forRequest(req)`:
   para ejecutar como el usuario autenticado del request.
3. `context.forSession(session)`:
   para scope por sesion ya resuelta.
4. `context.withAttribution(principalId)`:
   mantiene permisos backend, pero atribuye writes a usuario/sujeto.
5. `await context.withAttributionForRequest(req)`:
   igual que arriba, derivando identidad del request.
6. `context.db()`:
   handle general; en backends conectados, preferir `asBackend` o `forRequest`.

### 3.2 Operaciones de datos disponibles

Con el `Db` resultante puedes usar el mismo API principal:

1. Lectura: `all`, `one`.
2. Escritura: `insert`, `update`, `delete`, `upsert`, `restore`.
3. Durabilidad: `write.wait({ tier: "edge" | "global" | "local" })`.
4. Multiples writes: `transaction(...)` y `batch(...)`.
5. Subscripciones: `subscribeAll(...)` cuando aplique en procesos vivos.
6. Lifecycle: `context.flush()` y `await context.shutdown()`.

## 4) Uso recomendado en Hono

```ts
import { Hono } from "hono";
import { jazzContext, dbBackend } from "./jazz-context";
import { app } from "../schema";

const api = new Hono();

// Ejecuta como usuario autenticado (permite policy por usuario)
api.get("/api/todos", async (c) => {
  const requesterDb = await jazzContext.forRequest(c.req.raw, app);
  const rows = await requesterDb.all(app.todos.where({ done: false }));
  return c.json(rows);
});

// Ejecuta como backend (job/sistema)
api.post("/api/jobs/mark-stale", async (c) => {
  const stale = await dbBackend.all(app.todos.where({ done: false }));
  const tx = dbBackend.transaction((t) => {
    for (const row of stale) {
      t.update(app.todos, row.id, { done: true });
    }
    return stale.length;
  });
  await tx.wait({ tier: "edge" });
  return c.json({ updated: tx.value });
});
```

## 5) Integraciones externas (email/notificaciones) sin romper local-first

Patron recomendado: outbox.

1. API del usuario escribe evento/intencion en Jazz (`notification_jobs`, por ejemplo).
2. Worker/proceso backend lee pendientes con `asBackend()`.
3. Envia email/push usando secreto server-side (Resend, SES, FCM, etc.).
4. Actualiza estado en Jazz (`sent`, `failed`, `retry_count`, `next_retry_at`).

Ventajas:

1. No expones secrets en frontend.
2. Puedes reintentar y auditar.
3. Mantienes permisos y trazabilidad de quien disparo la accion.

## 6) Checklist de produccion

1. Reemplazar adapters en memoria de auth por almacenamiento persistente.
2. Guardar `BACKEND_SECRET`, `JAZZ_ADMIN_SECRET`, `BETTER_AUTH_SECRET` en secret manager.
3. Usar `forRequest` para operaciones de usuario y `asBackend` para jobs/sistema.
4. Definir politica de retries para integraciones externas.
5. Cerrar contexto en shutdown (`await jazzContext.shutdown()`).

## 7) Referencias del repo

1. Contexto backend y API (`createJazzContext`, `asBackend`, `forRequest`):
   [packages/jazz-tools/src/backend/create-jazz-context.ts](../packages/jazz-tools/src/backend/create-jazz-context.ts)
2. Exportes publicos de backend:
   [packages/jazz-tools/src/backend/index.ts](../packages/jazz-tools/src/backend/index.ts)
3. Guia oficial de server setup:
   [docs/content/docs/getting-started/server-setup.mdx](../docs/content/docs/getting-started/server-setup.mdx)
4. Ejemplo TS backend con context setup:
   [examples/docs/todo-server-ts/src/main.ts](../examples/docs/todo-server-ts/src/main.ts)
5. Request-scoped handler (`forRequest`) y attribution:
   [examples/docs/todo-server-ts/src/request-context.ts](../examples/docs/todo-server-ts/src/request-context.ts)
6. Starters con backend Hono minimo (health + auth):
   [starters/react-hybrid/server/app.ts](../starters/react-hybrid/server/app.ts)
   [starters/react-betterauth/server/app.ts](../starters/react-betterauth/server/app.ts)
