# React + Jazz: side effects de negocio, anti-entropy y no-perdida de eventos

Esta guia aterriza como operar side effects de negocio (email, push, webhooks) sobre Jazz sin perder eventos y sin procesar duplicados peligrosos.

## 1) Problema real

Cuando una app es local-first y reactiva, la duda tipica es:

1. Como se sincroniza sin enviar toda la base siempre.
2. Como evitar que se me escape un evento cuando se corta la conexion.
3. Como evitar ejecutar el mismo side effect dos veces.

## 2) Idea clave: anti-entropy en simple

Anti-entropy no significa "stream de eventos exactly-once".

Anti-entropy significa:

1. Mantener convergencia de estado entre nodos.
2. Al reconectar, re-publicar suscripciones y reconciliar faltantes.
3. Reaplicar de forma idempotente lo que ya existe, sin rehacer trabajo inutil.

En Jazz, esto se ve en:

1. Reconexion con replay de subscriptions activas.
2. Replay de historial local cuando aplica.
3. Short-circuit para replay idempotente de row batches ya presentes.

## 3) Que garantiza Jazz y que no

Si garantiza:

1. Writes locales inmediatos y sincronizacion posterior.
2. Reenvio de pendientes y replay de subscriptions al reconectar.
3. Replicacion por demanda de query (no envia todo siempre).

No garantiza por si solo:

1. Exactly-once para side effects externos.
2. Que un callback en memoria sea suficiente como unica fuente de verdad.

Conclusion operativa:

1. Sync/subscription te da convergencia de datos.
2. Para side effects necesitas outbox durable + consumidor idempotente.

## 4) Diagrama operativo (recomendado)

```text
[Client]
  -> escribe evento de negocio en tabla outbox (status=pending)
  -> sync local->edge->global

[Sync + Permissions]
  -> valida permisos/autorizacion
  -> si se acepta, el evento queda visible para consumidores

[Worker de side effects]
  -> trigger rapido: subscribeAll(outbox pending)
  -> red de seguridad: polling periodico de catch-up
  -> toma lote pending ordenado
  -> lock logico (processing + processing_owner + processing_at)
  -> llama proveedor externo con idempotency_key=event_id
  -> exito: status=sent, sent_at
  -> error transitorio: attempts++, next_retry_at, status=pending
  -> error permanente: status=failed, last_error

[Observabilidad]
  -> metricas: pending, sent, failed, retries
  -> alerta si pending viejo > umbral
```

## 5) Patron recomendado: subscription + catch-up polling

Usa ambos, no solo uno.

1. `subscribeAll` para latencia baja (reaccion casi inmediata).
2. Polling cada N segundos para recuperar huecos (caidas, reinicios, cortes).

Consulta de catch-up sugerida:

1. `status = pending`
2. `next_retry_at <= now`
3. `attempts < max_attempts`
4. orden por `created_at` ascendente

## 6) Estados minimos de outbox

Campos recomendados:

1. `event_id` (unico global, idempotency key).
2. `type` (order.confirmed, invoice.issued, etc.).
3. `payload` (datos inmutables para ejecutar side effect).
4. `status` (`pending|processing|sent|failed`).
5. `attempts`, `next_retry_at`, `last_error`.
6. `created_at`, `sent_at`.
7. opcional: `processing_owner`, `processing_at` para evitar doble consumo.

## 7) Como evitar duplicados

1. `event_id` unico en tu outbox.
2. Idempotency key en proveedor externo (si lo soporta).
3. Handler idempotente: si ya esta `sent`, no reprocesar.
4. Lock de procesamiento con TTL para workers concurrentes.
5. Reintentos con backoff (lineal o exponencial).

## 8) Como evitar faltantes

1. Nunca depender solo del callback en memoria.
2. Persistir siempre la intencion en outbox.
3. Ejecutar catch-up polling periodico.
4. Monitorear "edad de pending" para detectar atasco.
5. Mantener runbook de replay manual por rango de tiempo.

## 9) Flujo de fallo y recuperacion (ejemplo)

Caso: se cae red despues de crear evento y antes de mandar email.

1. Evento queda `pending` en outbox.
2. Suscripcion puede perder ese instante en vivo.
3. Al volver worker, polling detecta el pendiente.
4. Se envia email con idempotency key.
5. Se marca `sent`.

No hay perdida funcional, porque la fuente de verdad fue la fila durable.

## 10) Checklist de produccion

1. Outbox con estados y metadatos de retry.
2. Consumidor idempotente.
3. Subscription + catch-up polling.
4. Limite de intentos y politica de errores permanentes.
5. Dashboards de pendientes/retries/fallos.
6. Alertas por lag de procesamiento.

## 11) Referencias del repo

1. Transporte de sync y reconexion:
   [docs/content/docs/reference/internals.mdx](../docs/content/docs/reference/internals.mdx)
2. Flujo local-first, replay y tiers:
   [docs/content/docs/concepts/how-sync-works.mdx](../docs/content/docs/concepts/how-sync-works.mdx)
3. Nota de subscriptions (tier solo en primer snapshot):
   [docs/content/docs/reading/queries.mdx](../docs/content/docs/reading/queries.mdx)
4. Durabilidad de writes por tier:
   [docs/content/docs/writing/writing-data.mdx](../docs/content/docs/writing/writing-data.mdx)
5. Cliente TS subscribe (2 fases):
   [packages/jazz-tools/src/runtime/client.ts](../packages/jazz-tools/src/runtime/client.ts)
6. Runtime subscriptions (execute + immediate tick):
   [crates/jazz-tools/src/runtime_core/subscriptions.rs](../crates/jazz-tools/src/runtime_core/subscriptions.rs)
7. Query subscriptions con sync y replay a servidor nuevo:
   [crates/jazz-tools/src/query_manager/subscriptions.rs](../crates/jazz-tools/src/query_manager/subscriptions.rs)
8. Sync inbox idempotent replay short-circuit:
   [crates/jazz-tools/src/sync_manager/inbox.rs](../crates/jazz-tools/src/sync_manager/inbox.rs)
9. Nota de tests sobre replay en reconnect:
   [crates/jazz-tools/src/sync_manager/tests.rs](../crates/jazz-tools/src/sync_manager/tests.rs)
