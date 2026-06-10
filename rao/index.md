# Jazz repo overview

Este documento resume como esta organizado el framework Jazz en este monorepo y que configuraciones estan soportadas hoy.

## 1) Componentes principales de Jazz

### Runtime y motor

- Rust workspace (motor y utilidades):
	- [crates/jazz-tools](../crates/jazz-tools) (crate Rust principal; binario/CLI jazz-tools)
	- [crates/opfs-btree](../crates/opfs-btree) (almacenamiento OPFS/B-Tree)
	- [crates/wasm-tracing](../crates/wasm-tracing) (tracing para entorno WASM)
- Bindings del motor:
	- [crates/jazz-wasm](../crates/jazz-wasm) (WebAssembly para navegador)
	- [crates/jazz-napi](../crates/jazz-napi) (bindings nativos para Node.js via N-API)
	- [crates/jazz-rn](../crates/jazz-rn) (bindings para React Native)

### SDK TypeScript y modulos

- Paquete principal: jazz-tools ([packages/jazz-tools](../packages/jazz-tools))
- Superficies exportadas mas importantes de jazz-tools:
	- core SDK (.)
	- backend
	- better-auth-adapter
	- react, vue, svelte, react-native, expo
	- permissions, testing, dev
	- integraciones dev: dev/next, dev/vite, dev/expo, dev/sveltekit
	- utilidades: passphrase, passkey-backup

### Tooling y DX

- [packages/create-jazz](../packages/create-jazz): CLI para scaffold de apps desde starters
- [packages/inspector](../packages/inspector): app/herramienta de inspeccion de datos/estado
- [docs](../docs): sitio de documentacion
- [dev/local-telemetry](../dev/local-telemetry): stack local para observabilidad

### Plantillas y apps de referencia

- [starters/](../starters): plantillas oficiales para combinaciones framework + modo de auth/arquitectura
- [examples/](../examples): ejemplos completos (clientes local-first, servidores, cloudflare worker, auth, etc.)

## 2) Configuraciones posibles hoy (create-jazz)

La CLI create-jazz soporta esta matriz:

- Frameworks: next, react, sveltekit, ts
- Modos de auth/arquitectura: localfirst, hybrid, betterauth
- Hosting: hosted (Jazz Cloud) o selfhosted

Starters concretos disponibles (12):

1. next-localfirst
2. next-hybrid
3. next-betterauth
4. react-localfirst
5. react-hybrid
6. react-betterauth
7. sveltekit-localfirst
8. sveltekit-hybrid
9. sveltekit-betterauth
10. ts-localfirst
11. ts-hybrid
12. ts-betterauth

Notas de composicion:

- localfirst: base con jazz-tools
- hybrid: combina jazz-tools + better-auth + jazz-napi (runtime Node nativo cuando aplica)
- betterauth: combina jazz-tools + better-auth (en TS tambien incluye jazz-napi)

## 3) Configuracion de entorno para modo hosted

Cuando eliges hosting hosted, create-jazz provisiona app y rellena .env con llaves segun framework:

- next:
	- NEXT_PUBLIC_JAZZ_APP_ID
	- NEXT_PUBLIC_JAZZ_SERVER_URL
	- JAZZ_ADMIN_SECRET
	- BACKEND_SECRET
- react y ts:
	- VITE_JAZZ_APP_ID
	- VITE_JAZZ_SERVER_URL
	- JAZZ_ADMIN_SECRET
	- BACKEND_SECRET
- sveltekit:
	- PUBLIC_JAZZ_APP_ID
	- PUBLIC_JAZZ_SERVER_URL
	- JAZZ_ADMIN_SECRET
	- BACKEND_SECRET

## 4) Lectura rapida del monorepo

- Raiz: orquestacion con pnpm + turbo + workspace Rust
- [crates/](../crates): motor y bindings Rust/WASM/NAPI/RN
- [packages/](../packages): SDK TS, create-jazz e inspector
- [starters/](../starters): plantillas listas para iniciar proyectos
- [examples/](../examples): casos de uso y referencia practica
- [docs/](../docs): documentacion oficial

## 5) Punto de partida recomendado para estudiar Jazz

1. Revisar [packages/jazz-tools](../packages/jazz-tools) (API publica del SDK)
2. Revisar [starters](../starters) del framework que uses (flujo de app real)
3. Revisar [crates/jazz-wasm](../crates/jazz-wasm) y [crates/jazz-napi](../crates/jazz-napi) (runtime engine bindings)
4. Revisar [examples](../examples) para casos de uso end-to-end

## 6) Siguiente lectura (detalle API)

- Ver [jazz-tools-api-map.md](./jazz-tools-api-map.md) para un mapa tecnico modulo por modulo de la API publica de jazz-tools.

## 7) Guia practica por framework

- React paso a paso: [react-how-to.md](./react-how-to.md)
- React camino 1 detallado (localfirst): [react-localfirst-camino-1.md](./react-localfirst-camino-1.md)
- React sync server (camino 1): [react-localfirst-sync-server.md](./react-localfirst-sync-server.md)
- React camino 2 detallado (hybrid): [react-hybrid-camino-2.md](./react-hybrid-camino-2.md)
- React camino 3 detallado (betterauth): [react-betterauth-camino-3.md](./react-betterauth-camino-3.md)
- React matriz comparativa (3 caminos): [react-matriz-comparativa-3-caminos.md](./react-matriz-comparativa-3-caminos.md)
- React roadmap tecnico (etapas): [react-temas-roadmap.md](./react-temas-roadmap.md)
- React etapa 1 (branches + entornos): [react-branches-entornos.md](./react-branches-entornos.md)
- React etapa 2 (schema + flujo): [react-schema-datos-y-flujo.md](./react-schema-datos-y-flujo.md)
- React etapa 3 (data patterns): [react-data-patterns.md](./react-data-patterns.md)
- React etapa 4 (access control + permissions): [react-access-control-permissions.md](./react-access-control-permissions.md)
- React sync por org: [react-sync-por-org.md](./react-sync-por-org.md)
- React reading (queries/filters/includes): [react-reading-queries.md](./react-reading-queries.md)
- React writing (data/files/blobs): [react-writing-data-files-blobs.md](./react-writing-data-files-blobs.md)
- React topologia sync (local/edge/global): [react-topologia-sync-edge-global.md](./react-topologia-sync-edge-global.md)
- React internals de QuerySettled/completitud: [react-query-settled-completitud.md](./react-query-settled-completitud.md)
- React migraciones y evolucion de schema: [react-migraciones-evolucion-schema.md](./react-migraciones-evolucion-schema.md)
- React publicacion de catalogo (schema/migrations/permissions): [react-publicacion-catalogo-schema-migrations-permissions.md](./react-publicacion-catalogo-schema-migrations-permissions.md)
- React backend SDK + API de integraciones externas: [react-backend-sdk-api-integraciones.md](./react-backend-sdk-api-integraciones.md)
- React arquitectura event-driven vs API HTTP: [react-event-driven-vs-api-http.md](./react-event-driven-vs-api-http.md)
- React side effects de negocio (outbox + anti-entropy): [react-side-effects-negocio-outbox-anti-entropy.md](./react-side-effects-negocio-outbox-anti-entropy.md)

