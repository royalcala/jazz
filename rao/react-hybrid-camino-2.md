# Camino 2 detallado: React hybrid

Este documento explica el camino React hybrid en Jazz: app localfirst desde el primer uso, con opcion de upgrade a cuenta Better Auth sin perder identidad.

## 1) Que problema resuelve

1. Mantener onboarding rapido sin login obligatorio.
2. Permitir que despues el usuario cree cuenta y haga su identidad portable.
3. Evitar que el cambio a cuenta rompa el acceso a los datos ya creados localfirst.

## 2) Cuando elegir hybrid

1. Quieres UX localfirst inmediata.
2. Quieres opcion de cuenta para recuperar entre dispositivos.
3. No quieres forzar login en el primer minuto.

## 3) Componentes que participan

1. Frontend React: [starters/react-hybrid/src/main.tsx](../starters/react-hybrid/src/main.tsx).
2. Formularios auth: [starters/react-hybrid/src/sign-up-form.tsx](../starters/react-hybrid/src/sign-up-form.tsx) y [starters/react-hybrid/src/sign-in-form.tsx](../starters/react-hybrid/src/sign-in-form.tsx).
3. Cliente Better Auth: [starters/react-hybrid/src/auth-client.ts](../starters/react-hybrid/src/auth-client.ts).
4. Backend auth Hono: [starters/react-hybrid/server/app.ts](../starters/react-hybrid/server/app.ts) y [starters/react-hybrid/server/index.ts](../starters/react-hybrid/server/index.ts).
5. Verificacion de prueba localfirst: [starters/react-hybrid/server/auth.ts](../starters/react-hybrid/server/auth.ts).
6. Config Vite + JWKS: [starters/react-hybrid/vite.config.ts](../starters/react-hybrid/vite.config.ts).
7. Permisos de datos: [starters/react-hybrid/permissions.ts](../starters/react-hybrid/permissions.ts).

## 4) Como funciona (alto nivel)

Estado A: usuario anonimo

1. JazzProvider usa secret local via useLocalFirstAuth.
2. Usuario puede leer y escribir datos localfirst.

Estado B: usuario con cuenta

1. Better Auth crea sesion.
2. Frontend obtiene JWT y JazzProvider cambia a jwtToken.
3. Sync server valida JWT por JWKS y mantiene misma identidad logica.

## 5) Flujo de upgrade localfirst a cuenta

1. En sign up, el cliente pide prueba de identidad local:
- db.getLocalFirstIdentityProof con audience react-localfirst-signup.

2. Cliente envia proofToken junto a email/password al backend auth.

3. Backend verifica proofToken con jazz-napi:
- verifyLocalFirstIdentityProof.

4. Si verifica, backend crea usuario Better Auth con provedUserId.

5. Desde ese momento, el JWT de Better Auth representa esa identidad.

Resultado:

1. El usuario no pierde continuidad entre modo anonimo y cuenta.

## 6) Que corre en desarrollo

Con pnpm dev se levantan dos procesos:

1. Hono auth server en 3001.
2. Vite frontend en 5173.

Adicional:

1. jazzPlugin tambien levanta la parte de sync para Jazz en dev.
2. Vite proxea /api a Hono.

Scripts: [starters/react-hybrid/package.json](../starters/react-hybrid/package.json).

## 7) Variables de entorno claves

1. BETTER_AUTH_SECRET: obligatorio para firmar sesiones Better Auth.
2. PORT: opcional para servidor auth (default 3001).
3. VITE_JAZZ_APP_ID y VITE_JAZZ_SERVER_URL: app y sync endpoint.
4. JAZZ_ADMIN_SECRET y BACKEND_SECRET: aplican en cloud hosted.

Referencia completa: [starters/react-hybrid/README.md](../starters/react-hybrid/README.md).

## 8) Identidad y autorizacion de datos

Permisos en starter:

1. Lectura, update y delete filtrados por createdBy igual a session.user_id.
2. Insert permitido y Jazz registra creador de la fila.

Archivo: [starters/react-hybrid/permissions.ts](../starters/react-hybrid/permissions.ts).

## 9) Ventajas y tradeoffs

Ventajas:

1. Onboarding rapido.
2. Upgrade a cuenta sin friccion.
3. Mejor recuperacion entre dispositivos que localfirst puro.

Tradeoffs:

1. Mayor complejidad operativa que localfirst.
2. Debes operar y asegurar Better Auth server.
3. El starter usa memory adapter para usuarios; hay que migrar a persistente en produccion.

## 10) Checklist de produccion

1. Cambiar Better Auth memory adapter por base de datos persistente.
2. Configurar secretos y origenes confiables.
3. Verificar jwksUrl y rotacion de tokens.
4. Probar flujo completo: anonimo -> signup -> signin -> restore acceso.

## 11) Inicio rapido

1. npm create jazz@latest mi-app -- --starter react-hybrid --hosting selfhosted
2. cd mi-app
3. pnpm install
4. Configurar BETTER_AUTH_SECRET en .env
5. pnpm dev
6. Abrir http://localhost:5173

## 12) Documentos relacionados

1. [react-how-to.md](./react-how-to.md)
2. [react-localfirst-camino-1.md](./react-localfirst-camino-1.md)
3. [react-localfirst-sync-server.md](./react-localfirst-sync-server.md)
4. [react-localfirst-recovery-backup.md](./react-localfirst-recovery-backup.md)
