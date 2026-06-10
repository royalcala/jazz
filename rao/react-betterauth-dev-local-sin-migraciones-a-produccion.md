# React + Jazz: Better Auth persistente desde dev/local, sin migraciones tempranas, y paso a produccion

Esta guia es para arrancar rapido en local con Better Auth persistente y Jazz en modo de iteracion, evitando diferencias fuertes entre desarrollo y produccion.

## 1) Objetivo

1. Empezar en dev/local sin frenar por migraciones desde el dia 1.
2. Mantener persistencia real de usuarios/sesiones desde el inicio.
3. Aprovechar auto-publicacion de schema y permisos en desarrollo.
4. Dejar claro que cambia al pasar a produccion.

## 2) Stack recomendado para esta etapa

1. Starter `react-betterauth` (selfhosted).
2. Better Auth con adapter persistente tambien en dev/local.
3. Jazz con workflow de desarrollo sin migraciones formales al inicio.
4. `pnpm` como gestor de paquetes.

## 3) Como empezar en dev/local

1. Scaffold del proyecto con `create-jazz`.
2. Elegir React + BetterAuth + Selfhosted.
3. Instalar dependencias y correr `pnpm dev`.

En esta fase:

1. Puedes cambiar `schema.ts` y `permissions.ts` con ciclo rapido.
2. El plugin de desarrollo publica schema/permisos al iniciar y en cambios.
3. No necesitas bloquearte creando migraciones en cada iteracion temprana.

## 4) Que significa "sin migraciones" en dev

"Sin migraciones" no significa "sin costo".

1. Jazz no exige migracion para cada cambio de schema mientras iteras.
2. Pero si cambias schema y hay datos historicos en hashes anteriores, esos datos pueden quedar no legibles desde el schema actual hasta que crees la migracion.
3. Esto es aceptable en dev/local cuando priorizas velocidad de producto.

## 5) Better Auth persistente en dev/local

En el starter Better Auth:

1. Cambia el adapter de memoria por uno persistente desde el inicio del proyecto.
2. Esto hace que login/sesiones y debugging se parezcan mas a produccion.
3. Si necesitas limpiar estado, usa reset manual coordinado en lugar de depender de reinicios.

Patron recomendado:

1. Mantener persistencia en auth y Jazz.
2. Resetear ambos juntos solo cuando haga falta reiniciar baseline de desarrollo.

## 6) Momento de cambiar a modo produccion

Cuando ya tengas:

1. Flujo de producto estable.
2. Schema mas maduro.
3. Necesidad de endurecer secretos, observabilidad y operaciones.

## 7) Que cambia en produccion

### 7.1 Auth

1. Mantener el mismo adapter persistente (sin cambio de paradigma entre dev/prod).
2. Gestionar secretos (`BETTER_AUTH_SECRET`, claves JWT/JWKS) fuera del repo.

### 7.2 Sync server Jazz

1. Usar server dedicado de Jazz (no solo el loop de dev plugin).
2. Fijar `appId` estable por entorno.
3. Configurar `adminSecret`, `backendSecret` y, si aplica auth externa, `jwksUrl`.

### 7.3 Schema y permisos

1. Crear migraciones para cambios de schema que deben conservar compatibilidad.
2. Publicar con `deploy` para enviar migracion + schema + permisos juntos.
3. Mantener trazabilidad de hashes (`fromHash`/`toHash`).

## 8) Tabla dev vs produccion

1. Auth store:
dev/local: persistente.
produccion: persistente.

2. Schema evolution:
dev/local: iteracion rapida, migraciones bajo demanda.
produccion: migraciones revisadas y publicadas.

3. Publicacion:
dev/local: auto-push de schema/permisos del loop de desarrollo.
produccion: `pnpm dlx jazz-tools@alpha deploy <appId>`.

4. Persistencia usuarios:
dev/local: durable.
produccion: durable.

5. Reset de ambiente:
dev/local: manual y coordinado (auth + Jazz) cuando sea necesario.
produccion: no reset, solo migraciones/deploy controlado.

## 9) Checklist de salida a produccion

1. Verificar adapter persistente de Better Auth en staging/prod.
2. Congelar y versionar schema actual.
3. Crear/pulir migraciones pendientes.
4. Ejecutar deploy de schema+migraciones+permisos.
5. Verificar lectura de datos historicos en entorno staging.
6. Verificar flujo JWT/JWKS con Jazz server.
7. Habilitar observabilidad y alertas basicas.

## 10) Referencias del repo

1. Starter React Better Auth: [starters/react-betterauth/README.md](../starters/react-betterauth/README.md)
2. Setup cliente y auto-push en desarrollo: [docs/content/docs/getting-started/client-setup.mdx](../docs/content/docs/getting-started/client-setup.mdx)
3. Server setup (self-hosted): [docs/content/docs/getting-started/server-setup.mdx](../docs/content/docs/getting-started/server-setup.mdx)
4. Migrations workflow y limites de no migrar: [docs/content/docs/schemas/migrations.mdx](../docs/content/docs/schemas/migrations.mdx)
5. Definicion de tablas (cuando crear y push migration en app compartida): [docs/content/docs/schemas/defining-tables.mdx](../docs/content/docs/schemas/defining-tables.mdx)
6. Arquitectura general dev->prod: [react-arquitectura-dev-prod-hybrid-event-driven.md](./react-arquitectura-dev-prod-hybrid-event-driven.md)