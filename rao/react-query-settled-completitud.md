# React + Jazz: registro de query, QuerySettled y completitud

Este documento explica, con base en codigo real del repo, como Jazz registra una query reactiva, como la sincroniza y como decide que el primer snapshot ya esta completo para el tier solicitado.

## 1) Pregunta central

Cuando usas `useAll(...)`:

1. si, la query se registra;
2. si, se abre suscripcion real;
3. si, existe una senal explicita de "query asentada" (`QuerySettled`);
4. el primer snapshot solo se considera completo cuando se cumple el tier requerido.

## 2) Flujo de extremo a extremo

```text
React useAll
  -> SubscriptionsOrchestrator (key + cache entry)
  -> Db.subscribeAll
  -> Client.subscribe (runtime createSubscription/executeSubscription)
  -> QueryManager server/local (scope + required_tier)
  -> SyncManager (upstream/downstream)
  -> QuerySettled aplicado
  -> primer snapshot desbloqueado
  -> deltas continuos
```

## 3) Dónde se registra la query en cliente

En React, `useAllBase` calcula una key estable y, en subscribe, registra la query y abre la entrada de cache.

Referencias:

1. `computeKey` en hook React: [packages/jazz-tools/src/react-core/use-all.ts](../packages/jazz-tools/src/react-core/use-all.ts#L28)
2. `makeQueryKey` + `getCacheEntry` en subscribe: [packages/jazz-tools/src/react-core/use-all.ts](../packages/jazz-tools/src/react-core/use-all.ts#L45)
3. registro en `queryDefinitions` dentro del orchestrator: [packages/jazz-tools/src/subscriptions-orchestrator.ts](../packages/jazz-tools/src/subscriptions-orchestrator.ts#L224)

## 4) Dónde se abre la suscripcion real

El orchestrator llama `db.subscribeAll(...)` y ahi ya entra al runtime real.

Referencias:

1. apertura de suscripcion en orchestrator: [packages/jazz-tools/src/subscriptions-orchestrator.ts](../packages/jazz-tools/src/subscriptions-orchestrator.ts#L392)
2. `subscribeAll` en Db: [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts#L2047)
3. `client.subscribe(...)` desde Db: [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts#L2112)
4. runtime 2-phase subscribe (`createSubscription` / `executeSubscription`): [packages/jazz-tools/src/runtime/client.ts](../packages/jazz-tools/src/runtime/client.ts#L1457)

## 5) Como sabe el backend que "falta" data

No intenta traer toda la base. Se guia por suscripciones activas y por el tier requerido de cada query.

Cuando se crea la suscripcion:

1. se guarda `durability_tier` requerido;
2. se calcula estado inicial de `query_frontier_settled_tier`;
3. si corresponde, se envia `QuerySubscription` upstream.

Referencias:

1. creacion de subscription y tier inicial: [crates/jazz-tools/src/query_manager/subscriptions.rs](../crates/jazz-tools/src/query_manager/subscriptions.rs#L160)
2. envio upstream de suscripciones: [crates/jazz-tools/src/query_manager/subscriptions.rs](../crates/jazz-tools/src/query_manager/subscriptions.rs#L398)
3. chequeo de servidores conectados o pendientes: [crates/jazz-tools/src/sync_manager/mod.rs](../crates/jazz-tools/src/sync_manager/mod.rs#L454)

## 6) Como decide que el primer snapshot ya esta completo

La condicion formal esta en `subscription_query_frontier_satisfied`:

1. si no hay tier requerido -> satisfecha;
2. si hay tier requerido -> satisfecha cuando `query_frontier_settled_tier >= required_tier`.

Referencias:

1. `apply_query_settled(...)`: [crates/jazz-tools/src/query_manager/manager.rs](../crates/jazz-tools/src/query_manager/manager.rs#L1156)
2. `subscription_query_frontier_satisfied(...)`: [crates/jazz-tools/src/query_manager/manager.rs](../crates/jazz-tools/src/query_manager/manager.rs#L1183)

## 7) Por que a veces "espera" antes de mostrar

Si la suscripcion no esta satisfecha para el tier y hay upstream disponible/pendiente, Jazz mantiene la suscripcion esperando el frontier inicial.

Referencia:

1. rama "waiting for initial frontier": [crates/jazz-tools/src/query_manager/manager.rs](../crates/jazz-tools/src/query_manager/manager.rs#L1546)

## 8) Rol de QuerySettled en el pipeline

`QuerySettled` llega por sync, se encola y despues se aplica al manager para actualizar el tier asentado de la query.

Referencias:

1. inbox de `SyncPayload::QuerySettled` (server->client/server): [crates/jazz-tools/src/sync_manager/inbox.rs](../crates/jazz-tools/src/sync_manager/inbox.rs#L1215)
2. aplicacion de `pending_query_settled` en manager process: [crates/jazz-tools/src/query_manager/manager.rs](../crates/jazz-tools/src/query_manager/manager.rs#L1368)
3. liberacion por watermark de stream en runtime core: [crates/jazz-tools/src/runtime_core/ticks.rs](../crates/jazz-tools/src/runtime_core/ticks.rs#L497)

## 9) Despues del primer snapshot

Una vez asentada la query al tier requerido:

1. se entrega primer resultado completo;
2. la suscripcion sigue viva;
3. los cambios posteriores llegan como deltas (`Added`, `Updated`, `Removed`).

Referencia:

1. calculo y aplicacion de deltas en cliente: [packages/jazz-tools/src/runtime/subscription-manager.ts](../packages/jazz-tools/src/runtime/subscription-manager.ts)

## 10) Diferencia con one-shot

`db.all(...)` y `db.one(...)`:

1. ejecutan query puntual;
2. no dejan suscripcion activa.

`useAll(...)` y `subscribeAll(...)`:

1. si registran y mantienen suscripcion;
2. mantienen actualizaciones continuas.

Referencias:

1. `query(...)` (one-shot): [packages/jazz-tools/src/runtime/client.ts](../packages/jazz-tools/src/runtime/client.ts#L1347)
2. `subscribe(...)` (reactivo): [packages/jazz-tools/src/runtime/client.ts](../packages/jazz-tools/src/runtime/client.ts#L1443)

## 11) Checklist mental rapido

1. Persistente != completo.
2. Persistente: guardas lo que ya llego/local.
3. Completo: primer snapshot asentado al tier requerido.
4. Reactivo: despues del primer snapshot, solo deltas.

## 12) Referencias funcionales

1. [docs/content/docs/concepts/how-sync-works.mdx](../docs/content/docs/concepts/how-sync-works.mdx)
2. [docs/content/docs/reading/queries.mdx](../docs/content/docs/reading/queries.mdx)
3. [docs/content/docs/reference/durability-tiers.mdx](../docs/content/docs/reference/durability-tiers.mdx)
4. [react-reading-queries.md](./react-reading-queries.md)
5. [react-topologia-sync-edge-global.md](./react-topologia-sync-edge-global.md)