# React + Jazz: Edge vs Global (funcionalidad y separacion de responsabilidades)

Esta guia explica como operan juntos cliente local, edge y global en Jazz, que datos se mueven siempre y cuales se mueven bajo demanda.

## 1) Objetivo

1. Entender que resuelve cada capa (`local`, `edge`, `global`).
2. Aclarar si el edge replica todo o solo el scope activo.
3. Definir responsabilidades operativas por nivel.

## 2) Roles de cada nivel

### 2.1 Local (cliente)

1. Es la fuente de UX inmediata (lectura/render local-first).
2. Captura writes offline y los propaga cuando hay red.
3. No sustituye responsabilidades de reconciliacion entre regiones.

### 2.2 Edge (regional)

1. Es el primer hop de red para clientes de una region.
2. Recibe subscriptions/queries de clientes y las propaga upstream.
3. Mantiene el working set que su trafico regional va tocando.

### 2.3 Global (core)

1. Es la capa de reconciliacion global entre regiones/edges.
2. Actua como autoridad de convergencia cross-region.
3. Propaga catalogo y datos hacia edges conforme se asientan.

## 3) Que se sincroniza siempre vs bajo demanda

### 3.1 Siempre (baseline de topologia)

1. Catalogo (schema/permisos) se sincroniza entre core y edges.
2. En reconexion, un edge fresco puede recuperar catalogo aun sin query de cliente.

Evidencia:

1. [crates/jazz-tools/tests/edge_server_sync.rs](../crates/jazz-tools/tests/edge_server_sync.rs)
2. [crates/jazz-tools/src/runtime_core/sync.rs](../crates/jazz-tools/src/runtime_core/sync.rs)

### 3.2 Bajo demanda (scope-driven)

1. El edge reenvia QuerySubscription upstream.
2. El upstream responde con scope asentado (`QuerySettled`) y batches relevantes.
3. El edge entrega a downstream solo filas en scope de sus clientes.

Evidencia:

1. [crates/jazz-tools/src/query_manager/subscriptions.rs](../crates/jazz-tools/src/query_manager/subscriptions.rs)
2. [crates/jazz-tools/src/query_manager/server_queries.rs](../crates/jazz-tools/src/query_manager/server_queries.rs)
3. [crates/jazz-tools/src/sync_manager/inbox.rs](../crates/jazz-tools/src/sync_manager/inbox.rs)
4. [crates/jazz-tools/src/sync_manager/forwarding.rs](../crates/jazz-tools/src/sync_manager/forwarding.rs)
5. [crates/jazz-tools/src/sync_manager/types.rs](../crates/jazz-tools/src/sync_manager/types.rs)

## 4) Respuesta directa: edge replica todo o no

No replica todo siempre.

1. No hay full mirror continuo core -> edge por defecto.
2. El edge va acumulando datos segun queries/subscriptions y writes que pasan por el.
3. Si tu carga consulta casi todo, en la practica ese edge puede terminar con gran cobertura.

## 5) Flujo de lectura y escritura en conjunto

### 5.1 Lectura (simplificado)

```text
cliente -> edge (suscripcion/query)
edge -> global (forward subscription)
global -> edge (scope + row batches)
edge -> cliente (solo scope relevante)
```

### 5.2 Escritura (simplificado)

```text
cliente -> edge (write)
edge -> global (upstream sync)
global -> otros edges (replicacion)
otros edges -> clientes suscritos (si scope aplica)
```

## 6) Separacion de responsabilidades

### 6.1 Global

1. Convergencia y reconciliacion inter-regional.
2. Fuente de verdad para settlement global.
3. Distribucion de cambios entre edges.

### 6.2 Edge

1. Ingreso regional de trafico.
2. Reduccion de latencia para clientes cercanos.
3. Cache/working set operativo por demanda de su region.

### 6.3 Cliente

1. Experiencia local-first y resiliencia offline.
2. Suscripciones reactivas y writes de producto.
3. Seleccion de tier por pantalla/caso de uso.

## 7) Implicaciones operativas

1. Agregar un edge no implica copiar todo automaticamente.
2. Debes monitorear crecimiento del working set por region.
3. Las queries muy amplias pueden incrementar fuerte el dataset del edge.
4. El catalogo debe mantenerse consistente antes de exponer nuevas features.

## 8) Checklist de arquitectura

1. Definir que rutas/clientes apuntan a cada edge.
2. Definir pantallas con `tier: local`, `edge` o `global`.
3. Monitorear latencia edge->global y backlog de replicacion.
4. Verificar despliegue de schema/permisos en todos los edges activos.

## 9) Referencias

1. [docs/content/docs/concepts/how-sync-works.mdx](../docs/content/docs/concepts/how-sync-works.mdx)
2. [docs/content/docs/reference/durability-tiers.mdx](../docs/content/docs/reference/durability-tiers.mdx)
3. [crates/jazz-tools/src/server/builder.rs](../crates/jazz-tools/src/server/builder.rs)
4. [crates/jazz-tools/src/runtime_core/sync.rs](../crates/jazz-tools/src/runtime_core/sync.rs)
5. [crates/jazz-tools/src/sync_manager/mod.rs](../crates/jazz-tools/src/sync_manager/mod.rs)
6. [crates/jazz-tools/src/sync_manager/inbox.rs](../crates/jazz-tools/src/sync_manager/inbox.rs)
7. [crates/jazz-tools/src/sync_manager/forwarding.rs](../crates/jazz-tools/src/sync_manager/forwarding.rs)
8. [crates/jazz-tools/src/query_manager/server_queries.rs](../crates/jazz-tools/src/query_manager/server_queries.rs)
9. [crates/jazz-tools/tests/edge_server_sync.rs](../crates/jazz-tools/tests/edge_server_sync.rs)
10. Complemento de topologia: [react-topologia-sync-edge-global.md](./react-topologia-sync-edge-global.md)
