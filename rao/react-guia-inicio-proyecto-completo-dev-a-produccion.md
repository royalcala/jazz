# React + Jazz: guia completa para iniciar proyecto (dev local -> produccion)

Esta guia integra en un solo lugar los puntos clave para arrancar con Jazz y evitar rehacer arquitectura cuando pases de desarrollo a produccion.

## 1) Objetivo de esta guia

1. Empezar rapido en dev/local.
2. Tener auth funcional desde temprano.
3. Operar sync sin friccion.
4. Evitar bloqueo por migraciones en fase temprana.
5. Llegar a produccion con camino claro (auth persistente, migraciones, deploy).

## 2) Decision inicial de camino

### Opcion recomendada para este caso

1. `react-betterauth` (full auth) desde el inicio.
2. Login obligatorio para todos los usuarios desde dia 1.
3. Sin flujo anonimo como camino principal.

## 3) Monorepo con pnpm + Turborepo

Si ya sabes que tendras `web` + `api` + `worker`, usar monorepo desde el inicio evita refactors fuertes.

### 3.1 Estructura sugerida

```text
apps/
  web/
  api/
  worker/
packages/
  schema/
  permissions/
  jazz-context/
  domain/
```

Ubicacion de Jazz Inspector en esta estructura:

1. No forma parte del runtime de negocio (`web`, `api`, `worker`).
2. Entra como herramienta de operacion/debugging.
3. Puedes usarlo hosted (sin agregar paquete al proyecto) o como paquete interno de tooling en monorepo.

### 3.2 Raiz del monorepo

`pnpm-workspace.yaml`:

```yaml
packages:
  - apps/*
  - packages/*
```

`package.json` raiz (ejemplo):

```json
{
  "private": true,
  "packageManager": "pnpm@10",
  "scripts": {
    "dev": "turbo run dev --parallel",
    "build": "turbo run build",
    "test": "turbo run test",
    "lint": "turbo run lint"
  },
  "devDependencies": {
    "turbo": "^2"
  }
}
```

`turbo.json` (ejemplo):

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "dev": {
      "cache": false,
      "persistent": true
    },
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", "build/**", ".next/**", "server-dist/**"]
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "lint": {
      "outputs": []
    }
  }
}
```

### 3.3 Flujo operativo

1. `pnpm install` en la raiz.
2. `pnpm dev` para levantar web/api/worker en paralelo.
3. `pnpm build` para build completo ordenado por dependencias.

## 4) Arquitectura base recomendada

Separa responsabilidades desde el principio:

1. `web`: React + Jazz client.
2. `api`: comandos HTTP (validacion de negocio sincrona).
3. `worker`: side effects asincros (emails, webhooks, integraciones).
4. `sync server`: `jazz-tools server`.

## 5) APP_ID: de donde sale y donde se pone

`APP_ID` es el namespace de datos/sync de tu app. No te obliga a usar cloud.

Formas validas de obtenerlo:

1. Cloud: generar en dashboard (tambien te entrega secretos de deploy).
2. Self-hosted/local: generar UUID con CLI.

```bash
pnpm dlx jazz-tools@alpha create app
# devuelve un UUID (ese es tu APP_ID)
```

3. Dev plugin (Vite/Next/Svelte/Expo): si no hay valor, genera uno en primer arranque y lo persiste en `.env`.

Donde configurarlo:

1. Sync server self-hosted: argumento posicional de `server`.

```bash
pnpm dlx jazz-tools@alpha server <APP_ID>
```

2. Cliente web: variable publica segun bundler.
3. Deploy de schema/permisos: argumento de `deploy`.

```bash
pnpm dlx jazz-tools@alpha deploy <APP_ID>
```

Variables publicas por bundler:

1. Vite: `VITE_JAZZ_APP_ID`
2. Next: `NEXT_PUBLIC_JAZZ_APP_ID`
3. SvelteKit: `PUBLIC_JAZZ_APP_ID`
4. Expo: `EXPO_PUBLIC_JAZZ_APP_ID`

Regla operativa:

1. Un `APP_ID` por ambiente (`dev`, `staging`, `prod`).
2. Mantenerlo estable dentro de cada ambiente.
3. Si cambias `APP_ID`, Jazz lo interpreta como otra app (otro namespace).

## 6) Runbook dia 1 (comandos concretos)

Escenario: monorepo con `web`, `api`, `worker` y sync self-hosted.

1. Instalar dependencias en raiz:

```bash
pnpm install
```

2. Definir variables minimas de entorno:

```bash
export JAZZ_APP_ID="<tu-app-id>"
export JAZZ_ADMIN_SECRET="<tu-admin-secret>"
export JAZZ_SERVER_URL="http://127.0.0.1:1625"
```

3. Levantar server global local (Terminal A):

```bash
pnpm dlx jazz-tools@alpha server "$JAZZ_APP_ID" \
  --port 1625 \
  --data-dir ./.data/core \
  --admin-secret "$JAZZ_ADMIN_SECRET"
```

4. Levantar apps del monorepo (Terminal B):

```bash
pnpm dev
```

4.1 Modo dev para auto-push de `schema.ts` y `permissions.ts`:

1. Debe estar corriendo el proceso dev del frontend que tiene activo `jazzPlugin`.
2. En monorepo puedes correr todo (`pnpm dev`) o solo web (`pnpm --filter ./apps/web dev`).
3. Deja ese proceso encendido mientras editas `schema.ts` y `permissions.ts`.
4. El plugin detecta cambios y publica schema/permisos automaticamente en dev (sin reiniciar).

5. Validar baseline:

1. Login funciona (Better Auth).
2. Lectura/escritura basica funciona en web.
3. `schema.ts` y `permissions.ts` auto-pushean en dev (confirma en logs del proceso dev).

## 7) Topologia progresiva: primero global, luego edges bajo demanda

Empieza simple y escala sin rediseñar.

Fase 1 (inicio):

1. Un solo sync server global.
2. Todos los clientes apuntan a ese `serverUrl`.

Fase 2 (cuando crece latencia/regiones):

1. Agregas edges regionales.
2. Cada edge corre `jazz-tools server` con `--upstream-url` apuntando al global.
3. Requiere `--admin-secret` para enlazar upstream.

Ejemplo de edge (US):

```bash
pnpm dlx jazz-tools@alpha server "$JAZZ_APP_ID" \
  --port 2625 \
  --data-dir ./.data/edge-us \
  --admin-secret "$JAZZ_ADMIN_SECRET" \
  --upstream-url "http://127.0.0.1:1625"
```

Ejemplo de edge (EU):

```bash
pnpm dlx jazz-tools@alpha server "$JAZZ_APP_ID" \
  --port 3625 \
  --data-dir ./.data/edge-eu \
  --admin-secret "$JAZZ_ADMIN_SECRET" \
  --upstream-url "http://127.0.0.1:1625"
```

Regla de producto:

1. Cada usuario entra por su edge mas cercano.
2. Global reconcilia entre regiones.
3. Puedes sumar edges "on demand" por region/cliente enterprise sin tocar el modelo de datos.

## 8) Better Auth persistente desde dev/local

Si Jazz sera persistente, conviene que Auth tambien lo sea desde el inicio.

1. Usa adapter persistente de Better Auth tambien en dev/local.
2. Evitas diferencias de comportamiento entre dev y produccion.
3. Reducen sorpresas de sesiones/usuarios al reiniciar procesos.

## 9) Schema y permisos en dev/local sin bloquearte

En desarrollo:

1. Itera rapido en `schema.ts` y `permissions.ts`.
2. El loop de desarrollo puede auto-publicar schema/permisos.
3. No necesitas crear migracion formal por cada cambio temprano.

Importante:

1. Sin migracion, datos historicos de hashes viejos pueden quedar no legibles en schema nuevo.
2. En local esto puede ser aceptable durante iteracion.

## 10) Branching como herramienta principal en equipo

Usa branching para aislar trabajo:

1. `env` por ambiente (`dev`, `staging`, `prod`).
2. `userBranch` por feature/linea.
3. No mezclar desarrollo experimental con rama estable de release.

Nota:

1. Branching ayuda a aislar.
2. No reemplaza migraciones cuando necesitas compatibilidad entre versiones de schema con datos historicos.

## 11) Seed en local: patron recomendado

No hay un "seed command" unico tipo ORM clasico como ruta principal en docs.
El patron usado en ejemplos es seed idempotente desde codigo:

1. Leer primero (idealmente tier global para evitar carreras).
2. Si vacio, insertar datos iniciales.
3. Nunca asumir que el seed corre una sola vez.

Ademas, para modelos de membresia:

1. Usar bootstrap insert para el primer admin cuando aplique.

### 11.1 Donde corre el seed

Depende del tipo de dato que quieres inicializar:

1. Cliente (`web`): datos personales por usuario, demos locales o UX de primer uso.
2. Backend (`api`/job): datos compartidos de negocio (catalogos, defaults de tenant, plantillas globales).

Regla:

1. Si el dato lo deben compartir muchos usuarios, siembralo en backend.
2. Si el dato es solo del usuario actual, puede sembrarse en cliente.

### 11.2 Cuando correr el seed

1. Arranque de entorno nuevo (dev/staging) para baseline inicial.
2. Despues de reset manual coordinado (Auth + Jazz).
3. En arranque de app o login, solo si la logica es idempotente.

### 11.3 Patron seed en cliente (idempotente)

```ts
import type { Db } from "jazz-tools";
import { app } from "../schema";

export async function seedClientDefaultProject(db: Db) {
  // Leer en tier global reduce duplicados cuando hay varios clientes inicializando a la vez.
  const existing = await db.all(app.projects.where({ name: "Default" }), { tier: "global" });

  if (existing.length === 0) {
    db.insert(app.projects, { name: "Default" });
  }
}
```

Uso recomendado:

1. Llamarlo cuando ya tengas sesion/contexto listo.
2. No depender de que "solo corre una vez".

### 11.4 Patron seed en backend (idempotente)

```ts
import { createJazzContext } from "jazz-tools/backend";
import { app } from "../schema";
import { permissions } from "../permissions";

const context = createJazzContext({
  app,
  permissions,
  // serverUrl/appId/secrets segun tu entorno
});

export async function seedSharedCatalog() {
  const db = context.asBackend();

  const rows = await db.all(app.catalog.where({ code: "DEFAULT_STATUS" }), { tier: "global" });

  if (rows.length === 0) {
    db.insert(app.catalog, { code: "DEFAULT_STATUS", label: "Pending" });
  }
}
```

Uso recomendado:

1. Ejecutarlo en bootstrap de servicio o endpoint interno de inicializacion.
2. En produccion, proteger esa ruta/ejecucion (no exponerla al cliente final).

## 12) Event-driven de negocio (API + worker)

Patron recomendado:

1. API HTTP para comando principal y respuesta inmediata.
2. Misma unidad de escritura: dominio + evento en outbox.
3. Worker consume outbox con idempotencia.
4. Trigger rapido con subscripciones.
5. Red de seguridad con catch-up polling periodico.

Esto evita perder eventos y reduce acoplamiento con servicios externos.

## 13) Paso a produccion: que cambia

### 13.1 Auth

1. Mantener el adapter persistente en todos los ambientes.
2. Secretos reales fuera del repo (`BETTER_AUTH_SECRET`, claves JWT/JWKS).

### 13.2 Sync server

1. Correr `jazz-tools server` dedicado.
2. Configurar `appId`, `adminSecret`, `backendSecret`.
3. Si auth externa: `jwksUrl`.

### 13.3 Schema/migraciones/permisos

1. Crear migraciones para cambios de schema que deben conservar compatibilidad.
2. Publicar con deploy formal:

```bash
pnpm dlx jazz-tools@alpha deploy <appId>
```

3. Este flujo publica migracion + schema + permisos.

## 14) Reseteo de DB: cuando si y cuando no

En dev/local:

1. Si rompes mucho schema y quieres velocidad, reset manual coordinado puede ser valido.
2. Haz reset en conjunto de Jazz + Auth para mantener consistencia del entorno.
3. Debe ir con seed idempotente para recuperar baseline rapido.

Ejemplo de reset coordinado (con procesos detenidos):

```bash
rm -rf ./.data/core ./.data/edge-us ./.data/edge-eu
rm -f ./.data/auth.db
```

Nota:

1. Ajusta rutas segun donde persistas Jazz (`--data-dir`) y tu adapter de Better Auth.

En staging/prod:

1. Evitar reset.
2. Usar migraciones y deploy controlado.

## 15) Operacion y debugging

1. Usa Jazz Inspector para validar tablas, schema hash y permisos publicados.
2. Usa Live Query para observar subscripciones activas.
3. Trata `adminSecret` como credencial de infraestructura.

### 15.1 Jazz Inspector: que ocupas para correrlo y donde entra

Donde entra:

1. Capa de operacion/soporte.
2. No reemplaza `api` ni `worker`.
3. No se embebe en la UI de producto por defecto.

Que necesitas para conectarlo:

1. `serverUrl` del sync server.
2. `appId`.
3. `adminSecret`.
4. `env` (normalmente `dev`, `staging`, `prod`).
5. `branch` (normalmente `main`).

Opciones de ejecucion:

1. Hosted (recomendado para operacion diaria):
  1. Abrir `https://v2.inspector.jazz.tools/`.
  2. Conectar con `serverUrl`, `appId`, `adminSecret`, `env`, `branch`.
2. Standalone local (si trabajas dentro de este monorepo):

```bash
pnpm -C packages/inspector dev
```

Luego abrir `http://localhost:5173`.

3. Extension DevTools (opcional para debugging de runtime en navegador):

```bash
pnpm -C packages/inspector build:extension
```

Y cargar `packages/inspector/dist-extension` en `chrome://extensions` (Load unpacked).

## 16) Checklist rapido de inicio

1. Elegir starter `react-betterauth`.
2. Levantar dev con `pnpm dev`.
3. Configurar Better Auth con adapter persistente tambien en local.
4. Definir `schema.ts` y `permissions.ts` base.
5. Implementar seed idempotente minimo.
6. Separar `api` y `worker` aunque sea en version minima.
7. Implementar outbox para side effects.
8. Definir estrategia de ramas (`env` + `userBranch`).
9. Definir criterio de "ready for prod" para activar migraciones formales.

## 17) Referencias del repo

1. Quickstart: [docs/content/docs/quickstart.mdx](../docs/content/docs/quickstart.mdx)
2. Server setup: [docs/content/docs/getting-started/server-setup.mdx](../docs/content/docs/getting-started/server-setup.mdx)
3. Client setup (auto-push + deploy para prod): [docs/content/docs/getting-started/client-setup.mdx](../docs/content/docs/getting-started/client-setup.mdx)
4. Migrations: [docs/content/docs/schemas/migrations.mdx](../docs/content/docs/schemas/migrations.mdx)
5. Defining tables: [docs/content/docs/schemas/defining-tables.mdx](../docs/content/docs/schemas/defining-tables.mdx)
6. Group permissions bootstrap insert: [docs/content/docs/recipes/access-control/group-permissions.mdx](../docs/content/docs/recipes/access-control/group-permissions.mdx)
7. React betterauth starter: [starters/react-betterauth/README.md](../starters/react-betterauth/README.md)
8. Seed idempotente ejemplo: [examples/docs/todo-client-localfirst-ts/src/docs-snippets.ts](../examples/docs/todo-client-localfirst-ts/src/docs-snippets.ts)
9. Guia betterauth dev->prod: [react-betterauth-dev-local-sin-migraciones-a-produccion.md](./react-betterauth-dev-local-sin-migraciones-a-produccion.md)
10. Turborepo en este monorepo: [turbo.json](../turbo.json)
11. Workspace package manager: [pnpm-workspace.yaml](../pnpm-workspace.yaml)
12. Topologia global + edge: [react-topologia-sync-edge-global.md](./react-topologia-sync-edge-global.md)
13. Edge vs Global (responsabilidades y flujo real): [react-edge-vs-global-funcionalidad-responsabilidades.md](./react-edge-vs-global-funcionalidad-responsabilidades.md)
14. Inspector (referencia oficial): [docs/content/docs/reference/inspector.mdx](../docs/content/docs/reference/inspector.mdx)
15. Inspector package (modo standalone/extension): [packages/inspector/README.md](../packages/inspector/README.md)
16. Guia detallada de Inspector: [react-jazz-inspector-reading-writing-como-correr.md](./react-jazz-inspector-reading-writing-como-correr.md)
17. Writing data y tiers: [docs/content/docs/writing/writing-data.mdx](../docs/content/docs/writing/writing-data.mdx)
18. Permissions y testApp.seed: [docs/content/docs/auth/permissions.mdx](../docs/content/docs/auth/permissions.mdx)