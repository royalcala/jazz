# React + Jazz: arquitectura recomendada (dev sin friccion -> produccion con Better Auth + event-driven)

Esta guia define una ruta practica para empezar rapido en desarrollo y evolucionar a una arquitectura de produccion sin rehacer todo.

## 1) Objetivo

Cubrir este escenario:

1. Arrancar hoy en modo dev sin meterte de lleno en migraciones.
2. Pasar a produccion con registro de usuarios (Better Auth).
3. Mantener frontend React + sync backend de Jazz.
4. Agregar backend de negocio event-driven (API + workers + subscriptions/catch-up).

## 2) Recomendacion resumida

1. Usa starter `react-hybrid`.
2. Usa `pnpm` como package manager.
3. Usa monorepo con `turbo` desde el inicio si ya sabes que tendras web/api/worker.
4. Separa responsabilidades:
   1. `web` para UI y experiencia local-first.
   2. `api` para comandos HTTP y reglas de negocio sincronas.
   3. `worker` para side effects asincronos.
   4. `sync server` de Jazz como plano de datos/sync.

## 3) Fase 1 (dev rapido)

### 3.1 Arranque

1. Scaffold con `create-jazz` y elige React + Hybrid + Selfhosted.
2. Corre `pnpm install` y `pnpm dev`.

En este modo, el plugin de desarrollo te resuelve gran parte del setup para iterar rapido.

### 3.2 Sobre migraciones en dev

1. No bloquees avance por migraciones formales al inicio.
2. Define schema y permisos, valida flujo de producto primero.
3. Introduce migraciones cuando ya haya cambios estructurales reales entre versiones.

## 4) Fase 2 (preproduccion)

### 4.1 Auth real

1. Activa Better Auth con persistencia real (no memory adapter).
2. Configura JWT/JWKS para que Jazz server valide tokens externos.

### 4.2 Sync server dedicado

1. Corre Jazz server fuera del proceso de Vite dev.
2. Configura `appId`, `adminSecret`, `backendSecret` y `jwksUrl`.

## 5) Fase 3 (produccion)

### 5.1 Arquitectura recomendada

1. `web` React publica comandos y consume consultas/suscripciones.
2. `api` ejecuta comandos de negocio y escribe estado + outbox.
3. `worker` consume outbox de forma idempotente y ejecuta integraciones externas.
4. `jazz-tools server` provee sync y almacenamiento.

### 5.2 Patron event-driven recomendado

1. Entrada por API HTTP para validacion/errores inmediatos.
2. Side effects por outbox durable.
3. Trigger de baja latencia via subscriptions.
4. Red de seguridad via catch-up polling periodico.
5. Idempotencia fuerte por `event_id`.

## 6) Estructura de monorepo sugerida

```text
apps/
  web/            # React + Jazz client
  api/            # Hono/Node API (comandos)
  worker/         # consumidores outbox, jobs
packages/
  schema/         # schema compartido
  permissions/    # permissions compartidas
  jazz-context/   # createJazzContext centralizado
  domain/         # tipos/validaciones/eventos
```

## 7) Por que pnpm + turbo aqui

1. `pnpm` simplifica workspaces y lockfile unico.
2. `turbo` te permite orquestar `dev`, `build`, `test`, `lint` por app/paquete con cache.
3. Encaja natural con una topologia web + api + worker + paquetes compartidos.

## 8) Checklist minimo para no bloquearte

1. Elegir `react-hybrid` como base.
2. Definir `schema.ts` y `permissions.ts` temprano.
3. Crear `api` y `worker` separados (aunque sea con endpoints/jobs minimos).
4. Implementar outbox antes de integrar proveedores externos.
5. Dejar migraciones formales para cuando haya cambios estructurales reales.
6. En produccion, usar Better Auth persistente y secretos gestionados fuera del repo.

## 9) Referencias del repo

1. Starter recomendado: [starters/react-hybrid/README.md](../starters/react-hybrid/README.md)
2. Quickstart de create-jazz: [docs/content/docs/quickstart.mdx](../docs/content/docs/quickstart.mdx)
3. Server setup (self-hosted y flags): [docs/content/docs/getting-started/server-setup.mdx](../docs/content/docs/getting-started/server-setup.mdx)
4. Install backend TS: [docs/content/docs/install/typescript-server.mdx](../docs/content/docs/install/typescript-server.mdx)
5. Event-driven vs API HTTP: [react-event-driven-vs-api-http.md](./react-event-driven-vs-api-http.md)
6. Outbox + anti-entropy: [react-side-effects-negocio-outbox-anti-entropy.md](./react-side-effects-negocio-outbox-anti-entropy.md)
7. Auth server/JWT/JWKS: [react-temas-roadmap.md](./react-temas-roadmap.md)