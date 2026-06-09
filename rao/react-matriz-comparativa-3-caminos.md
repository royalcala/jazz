# React: matriz comparativa de los 3 caminos

Esta matriz resume que comparten y que cambia entre:

1. Localfirst
2. Hybrid
3. Betterauth

## 1) Matriz rapida

| Dimension | Localfirst | Hybrid | Betterauth |
| --- | --- | --- | --- |
| Onboarding inicial | Sin login obligatorio | Sin login obligatorio | Login obligatorio |
| Primer CRUD sin cuenta | Si | Si | No |
| Portabilidad entre dispositivos | Depende de backup/restore de identidad | Si, al registrar cuenta y usar JWT | Si, via login cuenta |
| Upgrade de anonimo a cuenta | No aplica | Si, con proof token | No aplica |
| Provider en frontend | secret local | secret o jwtToken (segun sesion) | jwtToken |
| Servidor auth separado | No | Si (Hono + Better Auth) | Si (Hono + Better Auth) |
| JWT/JWKS | No requerido | Si | Si |
| Complejidad operativa | Baja | Media | Media |
| Friccion UX inicial | Muy baja | Baja | Alta (relativa) |
| Riesgo principal | Perder identidad si no hay backup | Operar bien enlace identidad + auth | Friccion de alta inicial |

## 2) Lo que SI es igual en los 3

1. Definir schema de datos con Jazz en schema.ts.
2. Definir permisos en permissions.ts.
3. Hacer CRUD desde React con useDb/useAll.
4. Flujo de desarrollo para features:
   cambiar schema -> ajustar permisos -> actualizar UI -> validar en dev.
5. Regla de ownership por session.user_id y createdBy (misma idea base).

Referencias de ejemplo:

1. [starters/react-localfirst/schema.ts](../starters/react-localfirst/schema.ts)
2. [starters/react-hybrid/schema.ts](../starters/react-hybrid/schema.ts)
3. [starters/react-betterauth/schema.ts](../starters/react-betterauth/schema.ts)
4. [starters/react-localfirst/permissions.ts](../starters/react-localfirst/permissions.ts)
5. [starters/react-hybrid/permissions.ts](../starters/react-hybrid/permissions.ts)
6. [starters/react-betterauth/permissions.ts](../starters/react-betterauth/permissions.ts)

## 3) Lo que cambia de verdad

### 3.1 Identidad y sesion

1. Localfirst: identidad local via secret del dispositivo.
2. Hybrid: inicia con secret local, luego puede pasar a jwtToken de cuenta.
3. Betterauth: identidad siempre via cuenta + JWT.

### 3.2 Infraestructura runtime

1. Localfirst: frontend + sync server (sin auth server separado).
2. Hybrid: frontend + sync server + auth server.
3. Betterauth: frontend + sync server + auth server.

### 3.3 Recuperacion de acceso

1. Localfirst: recovery phrase y/o passkey backup.
2. Hybrid: backup localfirst mas cuenta portable.
3. Betterauth: cuenta portable desde el inicio.

## 4) Regla de decision rapida

1. Quiero aprender rapido y minimizar friccion inicial: Localfirst.
2. Quiero anonimo al inicio y cuenta despues: Hybrid.
3. Quiero cuenta obligatoria desde dia 1: Betterauth.

## 5) Riesgos por camino

1. Localfirst: usuario sin backup puede perder acceso a su identidad.
2. Hybrid: complejidad extra por enlazar identidad local + cuenta.
3. Betterauth: mayor abandono potencial por requerir registro inmediato.

## 6) Documentos de detalle

1. [react-localfirst-camino-1.md](./react-localfirst-camino-1.md)
2. [react-hybrid-camino-2.md](./react-hybrid-camino-2.md)
3. [react-betterauth-camino-3.md](./react-betterauth-camino-3.md)
4. [react-how-to.md](./react-how-to.md)
