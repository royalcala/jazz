# Jazz + React how-to (paso a paso)

Esta guia te lleva desde cero hasta una app React funcionando con Jazz.

## 1) Elegir modo de inicio

Antes de crear la app, elige uno de estos 3 caminos:

1. React localfirst
- Ideal para arrancar rapido.
- No requiere login al inicio.
- Starter base: [starters/react-localfirst](../starters/react-localfirst)

2. React hybrid
- Empieza localfirst y permite upgrade opcional a cuenta Better Auth.
- Starter base: [starters/react-hybrid](../starters/react-hybrid)

3. React betterauth
- Login obligatorio desde el inicio (email/password).
- Starter base: [starters/react-betterauth](../starters/react-betterauth)

Si estas empezando con Jazz, usa React localfirst primero.

Detalle tecnico completo del camino 1:

- [react-localfirst-camino-1.md](./react-localfirst-camino-1.md)
- Sync server del camino 1: [react-localfirst-sync-server.md](./react-localfirst-sync-server.md)
- Recovery phrase y passkey backup: [react-localfirst-recovery-backup.md](./react-localfirst-recovery-backup.md)

Detalle tecnico completo del camino 2:

- [react-hybrid-camino-2.md](./react-hybrid-camino-2.md)

Detalle tecnico completo del camino 3:

- [react-betterauth-camino-3.md](./react-betterauth-camino-3.md)

Vista comparativa de los 3 caminos:

- [react-matriz-comparativa-3-caminos.md](./react-matriz-comparativa-3-caminos.md)

Roadmap tecnico para desarrollar por etapas:

- [react-temas-roadmap.md](./react-temas-roadmap.md)
- Entregable etapa 1 (branches + entornos): [react-branches-entornos.md](./react-branches-entornos.md)
- Entregable etapa 2 (schema + flujo): [react-schema-datos-y-flujo.md](./react-schema-datos-y-flujo.md)

## 2) Crear el proyecto

Opcional interactivo:

1. Ejecuta: npm create jazz@latest
2. Elige framework React
3. Elige hosting hosted o selfhosted
4. Elige auth localfirst, hybrid o betterauth

Opcional directo por flag (sin prompts):

1. Ejecuta: npm create jazz@latest mi-app -- --starter react-localfirst --hosting selfhosted
2. Para hybrid: cambia starter por react-hybrid
3. Para betterauth: cambia starter por react-betterauth

Referencia CLI: [packages/create-jazz/README.md](../packages/create-jazz/README.md)

## 3) Levantar la app por primera vez

1. Entra al proyecto: cd mi-app
2. Instala deps: pnpm install
3. Levanta desarrollo: pnpm dev

Si elegiste React localfirst:

- Vite arranca en 5173.
- El plugin de Jazz levanta server local y completa VITE_JAZZ_APP_ID y VITE_JAZZ_SERVER_URL automaticamente.

Si elegiste React hybrid o betterauth:

- Arrancan 2 procesos: Hono (auth) + Vite (frontend).
- Se usa puerto 3001 para auth y 5173 para UI.
- Debes tener BETTER_AUTH_SECRET (si no existe, el starter lo genera en desarrollo con script de entorno).

## 4) Entender los archivos que vas a tocar

Para React localfirst:

1. [starters/react-localfirst/src/main.tsx](../starters/react-localfirst/src/main.tsx): provider y bootstrap de cliente
2. [starters/react-localfirst/schema.ts](../starters/react-localfirst/schema.ts): tablas y tipos
3. [starters/react-localfirst/permissions.ts](../starters/react-localfirst/permissions.ts): permisos por fila
4. [starters/react-localfirst/src/todo-widget.tsx](../starters/react-localfirst/src/todo-widget.tsx): ejemplo CRUD

Para React hybrid / betterauth:

1. [starters/react-hybrid/server/auth.ts](../starters/react-hybrid/server/auth.ts) o [starters/react-betterauth/server/auth.ts](../starters/react-betterauth/server/auth.ts)
2. [starters/react-hybrid/server/index.ts](../starters/react-hybrid/server/index.ts) o [starters/react-betterauth/server/index.ts](../starters/react-betterauth/server/index.ts)
3. [starters/react-hybrid/src/main.tsx](../starters/react-hybrid/src/main.tsx) o [starters/react-betterauth/src/main.tsx](../starters/react-betterauth/src/main.tsx)

## 5) Primer cambio real en tu app

Secuencia recomendada:

1. Agrega una columna en schema.ts
2. Ajusta permisos en permissions.ts
3. Usa la nueva columna en el componente React
4. Guarda y valida en navegador

En desarrollo, Jazz repush de schema se hace automaticamente con los plugins del starter.

## 6) Variables de entorno (resumen)

Selfhosted localfirst:

- Puedes iniciar sin .env manual.

Cloud hosted:

- VITE_JAZZ_APP_ID
- VITE_JAZZ_SERVER_URL
- JAZZ_ADMIN_SECRET
- BACKEND_SECRET

Hybrid o BetterAuth ademas requieren:

- BETTER_AUTH_SECRET

## 7) Flujo recomendado para aprender rapido

1. Crea app con react-localfirst selfhosted
2. Modifica schema + permissions + UI hasta sentirte comodo
3. Migra a react-hybrid cuando quieras cuenta opcional
4. Usa react-betterauth solo si tu producto requiere login obligatorio desde el dia 1

## 8) Referencias oficiales de starters React

1. [starters/react-localfirst/README.md](../starters/react-localfirst/README.md)
2. [starters/react-hybrid/README.md](../starters/react-hybrid/README.md)
3. [starters/react-betterauth/README.md](../starters/react-betterauth/README.md)
