# Camino 3 detallado: React betterauth

Este documento explica el camino React betterauth en Jazz: acceso con cuenta obligatoria desde el inicio, usando Better Auth + JWT.

## 1) Que problema resuelve

1. Forzar autenticacion desde el primer uso.
2. Dar identidad portable entre dispositivos sin depender de un secreto local anonimo.
3. Mantener flujo clasico de cuenta (sign up/sign in/sign out) para productos que lo exigen.

## 2) Cuando elegir betterauth

1. Tu producto requiere login obligatorio por compliance o negocio.
2. No quieres modo anonimo inicial.
3. Prefieres onboarding tipo cuenta desde dia 1.

## 3) Componentes que participan

1. Frontend provider y refresh de JWT: [starters/react-betterauth/src/main.tsx](../starters/react-betterauth/src/main.tsx).
2. Pantalla principal con guard por sesion: [starters/react-betterauth/src/App.tsx](../starters/react-betterauth/src/App.tsx).
3. Formulario sign in/sign up: [starters/react-betterauth/src/sign-in-form.tsx](../starters/react-betterauth/src/sign-in-form.tsx).
4. Cliente Better Auth: [starters/react-betterauth/src/auth-client.ts](../starters/react-betterauth/src/auth-client.ts).
5. Backend auth Hono: [starters/react-betterauth/server/app.ts](../starters/react-betterauth/server/app.ts) y [starters/react-betterauth/server/index.ts](../starters/react-betterauth/server/index.ts).
6. Config Better Auth y JWT: [starters/react-betterauth/server/auth.ts](../starters/react-betterauth/server/auth.ts).
7. Integracion JWKS en Vite plugin: [starters/react-betterauth/vite.config.ts](../starters/react-betterauth/vite.config.ts).
8. Permisos de datos: [starters/react-betterauth/permissions.ts](../starters/react-betterauth/permissions.ts).

## 4) Como funciona (alto nivel)

1. Usuario crea cuenta o inicia sesion con email/password.
2. Better Auth crea sesion y emite JWT.
3. Frontend obtiene JWT y lo pasa a JazzProvider como jwtToken.
4. Sync server de Jazz valida JWT usando JWKS.
5. Si es valido, habilita CRUD segun permisos de session.user_id.

## 5) Flujo de autenticacion

1. El formulario llama authClient.signUp.email o authClient.signIn.email.
2. useSession detecta sesion activa.
3. authClient.token devuelve JWT para Jazz.
4. JazzProvider se configura con appId, serverUrl y jwtToken.
5. Cuando expira, JwtRefresh pide token nuevo y actualiza db.updateAuthToken.

## 6) Que corre en desarrollo

Con pnpm dev se levantan dos procesos:

1. Hono auth server en 3001.
2. Vite frontend en 5173.

Adicional:

1. jazzPlugin levanta/integra la capa de sync para dev.
2. Vite proxea /api al servidor Hono.

Scripts: [starters/react-betterauth/package.json](../starters/react-betterauth/package.json).

## 7) Variables de entorno claves

1. BETTER_AUTH_SECRET: obligatorio para firmar sesiones/tokens Better Auth.
2. PORT: opcional (default 3001).
3. VITE_JAZZ_APP_ID y VITE_JAZZ_SERVER_URL: endpoint de app/sync.
4. JAZZ_ADMIN_SECRET y BACKEND_SECRET: para cloud hosted.

Referencia completa: [starters/react-betterauth/README.md](../starters/react-betterauth/README.md).

## 8) Identidad y autorizacion de datos

1. La identidad principal viene de la cuenta Better Auth (JWT subject).
2. El sync server valida token por JWKS antes de aceptar operaciones.
3. Permisos filtran por createdBy igual a session.user_id.

Archivo de permisos: [starters/react-betterauth/permissions.ts](../starters/react-betterauth/permissions.ts).

## 9) Diferencia vs Hybrid

betterauth:

1. Requiere login para usar app.
2. No ofrece fase anonima localfirst inicial.

hybrid:

1. Permite usar app anonimamente primero.
2. Luego enlaza identidad local a cuenta.

## 10) Ventajas y tradeoffs

Ventajas:

1. Modelo mental simple de cuenta obligatoria.
2. Portabilidad directa entre dispositivos via login.
3. Menor riesgo de usuarios anonimos no respaldados.

Tradeoffs:

1. Mayor friccion inicial por auth obligatoria.
2. Operacion de backend auth necesaria.
3. Starter usa memory adapter para usuarios; no apto produccion sin cambiarlo.

## 11) Checklist de produccion

1. Reemplazar memory adapter por base persistente.
2. Gestionar BETTER_AUTH_SECRET y rotacion con cuidado.
3. Configurar origenes confiables y APP_ORIGIN correctos.
4. Validar flujo token refresh y expiracion.
5. Probar signin/signout desde multiples dispositivos.

## 12) Inicio rapido

1. npm create jazz@latest mi-app -- --starter react-betterauth --hosting selfhosted
2. cd mi-app
3. pnpm install
4. Configurar BETTER_AUTH_SECRET en .env
5. pnpm dev
6. Abrir http://localhost:5173

## 13) Documentos relacionados

1. [react-how-to.md](./react-how-to.md)
2. [react-localfirst-camino-1.md](./react-localfirst-camino-1.md)
3. [react-hybrid-camino-2.md](./react-hybrid-camino-2.md)
