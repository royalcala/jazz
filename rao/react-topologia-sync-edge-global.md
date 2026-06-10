# React + Jazz: topologia de sync (local, edge, global)

Este documento aterriza la etapa de topologia de sync para entender como se despliega Jazz y como se enlazan sus nodos en una app React.

## 1) Objetivo de la etapa

1. Entender que roles cumplen `local`, `edge` y `global`.
2. Saber levantar topologia core + edge en self-host.
3. Definir como conectar clientes y que `tier` usar por escenario.

## 2) Modelo mental correcto

Piensa en Jazz como una red de replicas por niveles:

1. `local`: replica del cliente/dispositivo.
2. `edge`: primer servidor cercano al cliente.
3. `global`: servidor core de reconciliacion global.

Flujo simplificado:

```text
writes: app -> local -> edge -> global
reads:  query subscription <- local <- edge <- global
```

Importante:

1. El cliente siempre renderiza desde su replica local.
2. `tier` no cambia eso; solo cambia el umbral del primer resultado.

## 3) Son dos servicios distintos o el mismo server

Es el mismo server con distinto modo:

1. Modo core/global: `jazz-tools server` sin `--upstream-url`.
2. Modo edge: `jazz-tools server` con `--upstream-url`.

Base de codigo:

1. Deteccion de topologia core vs edge: [crates/jazz-tools/src/server/builder.rs](../crates/jazz-tools/src/server/builder.rs)
2. Regla de tier local del server (`GlobalServer` en core, `EdgeServer` en edge): [crates/jazz-tools/src/server/builder.rs](../crates/jazz-tools/src/server/builder.rs)

## 4) Como se enlaza edge con global

Un edge se conecta upstream al core/global via WebSocket app-scoped.

1. Activas edge con `--upstream-url`.
2. Debes pasar `--admin-secret` (o `JAZZ_ADMIN_SECRET`).
3. El server construye automaticamente la ruta `/apps/<APP_ID>/ws` para sync upstream.

Base de codigo:

1. Validacion edge requiere admin secret: [crates/jazz-tools/src/server/builder.rs](../crates/jazz-tools/src/server/builder.rs)
2. Conversion de URL upstream a WS app-scoped: [crates/jazz-tools/src/server/builder.rs](../crates/jazz-tools/src/server/builder.rs)
3. Inicio de upstream sync del edge: [crates/jazz-tools/src/server/builder.rs](../crates/jazz-tools/src/server/builder.rs)

## 5) Arranque local minimo (core + 2 edges)

Ejemplo practico para pruebas regionales en local.

### 5.1 Variables

```bash
export APP_ID="tu-app-id"
export ADMIN_SECRET="tu-admin-secret"
```

### 5.2 Levantar core/global

```bash
npx jazz-tools@alpha server "$APP_ID" \
  --port 1625 \
  --data-dir ./data-core \
  --admin-secret "$ADMIN_SECRET"
```

### 5.3 Levantar edge US

```bash
npx jazz-tools@alpha server "$APP_ID" \
  --port 2625 \
  --data-dir ./data-edge-us \
  --admin-secret "$ADMIN_SECRET" \
  --upstream-url "http://127.0.0.1:1625"
```

### 5.4 Levantar edge EU

```bash
npx jazz-tools@alpha server "$APP_ID" \
  --port 3625 \
  --data-dir ./data-edge-eu \
  --admin-secret "$ADMIN_SECRET" \
  --upstream-url "http://127.0.0.1:1625"
```

## 6) A que URL apuntas el cliente

Regla de producto:

1. El cliente debe apuntar al edge mas cercano (su `serverUrl`).
2. Si un usuario de otra region escribe en otro edge, el dato llega via global.
3. El cliente no necesita conocer todos los nodos, solo su endpoint de entrada.

## 7) Como usar local, edge y global en queries

En lecturas reactivas (`useAll`):

1. `local`: primer resultado inmediato desde local y sigue escuchando cambios.
2. `edge`: espera primer snapshot confirmado por edge y sigue escuchando cambios.
3. `global`: espera primer snapshot confirmado por global y sigue escuchando cambios.

No confundir con one-shot:

1. `db.all` y `db.one` no dejan suscripcion activa.
2. `useAll` y `subscribeAll` si mantienen escucha continua.

## 8) Criterio practico de seleccion de tier

1. Pantallas de trabajo diario: `local` (o `edge` si necesitas primer snapshot de red).
2. Pantallas de verificacion compartida entre usuarios: `edge`.
3. Casos de consistency fuerte inicial/auditoria: `global`.

## 9) Checklist de salida

1. Existe diagrama/logica de core y edges por ambiente.
2. Se puede levantar topologia local con al menos 1 core y 1 edge.
3. Clientes apuntan al edge correspondiente por entorno/region.
4. Se definio policy de `tier` por pantalla critica.
5. El equipo distingue suscripciones reactivas vs lecturas one-shot.

## 10) Referencias

1. [docs/content/docs/concepts/how-sync-works.mdx](../docs/content/docs/concepts/how-sync-works.mdx)
2. [docs/content/docs/getting-started/server-setup.mdx](../docs/content/docs/getting-started/server-setup.mdx)
3. [examples/docs/todo-server-rs/docs/self-host-cli.sh](../examples/docs/todo-server-rs/docs/self-host-cli.sh)
4. [crates/jazz-tools/src/server/builder.rs](../crates/jazz-tools/src/server/builder.rs)
5. [crates/jazz-tools/tests/edge_server_sync.rs](../crates/jazz-tools/tests/edge_server_sync.rs)
6. [docs/content/docs/reference/durability-tiers.mdx](../docs/content/docs/reference/durability-tiers.mdx)
7. [docs/content/docs/reading/queries.mdx](../docs/content/docs/reading/queries.mdx)
