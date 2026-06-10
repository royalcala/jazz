# React + Jazz: Event-Driven vs API HTTP

Este tema responde una duda comun: si conviene mas exponer una API HTTP clasica o disparar microservicios por eventos/cambios de base de datos.

La recomendacion practica en Jazz suele ser un patron hibrido:

1. API HTTP para comandos de negocio y validacion sincrona.
2. Event-driven para side effects asincros (email, push, webhooks, analitica, integraciones).

## 1) API HTTP vs Event-Driven

### API HTTP (request/response)

Conviene cuando necesitas:

1. Respuesta inmediata al usuario (`200/400/403`).
2. Validacion de negocio en tiempo real.
3. Contratos simples para frontends y terceros.

Ventajas:

1. Flujo facil de depurar.
2. Errores y estados claros por request.
3. Control directo de autorizacion por request (`forRequest`).

Costos:

1. Acoplamiento temporal (si downstream cae, la request sufre).
2. Escala peor para side effects pesados.

### Event-Driven

Conviene cuando necesitas:

1. Procesar trabajos en background.
2. Integrar multiples consumidores (email + push + CRM).
3. Reintentos y tolerancia a fallos transitorios.

Ventajas:

1. Desacopla productores y consumidores.
2. Mejor throughput para integraciones externas.
3. Facil agregar nuevos consumidores sin tocar la API principal.

Costos:

1. Mayor complejidad operativa (idempotencia, retries, DLQ).
2. Consistencia eventual (no todo queda listo al instante).

## 2) En Jazz, que significa “escuchar cambios de DB y detonar microservicios”

En Jazz tienes dos caminos reales:

1. Suscripciones (`subscribeAll`) sobre consultas, para procesos vivos que reaccionan a cambios.
2. Patron outbox en tablas de dominio/eventos, procesado por workers backend.

Para side effects de produccion (email/notificaciones), el camino recomendado suele ser outbox + worker, porque te permite control robusto de reintentos, estados y deduplicacion.

## 3) Recomendacion de arquitectura (la que mejor balancea)

No elegir solo uno. Usar ambos con responsabilidades claras.

1. Entrada de negocio:
   API HTTP en Hono/Express/Fastify.
2. Escritura de dominio:
   guardar cambio principal y evento outbox en la misma transaccion/batch.
3. Confirmacion:
   esperar durabilidad (`wait`) segun criticidad.
4. Procesamiento async:
   worker backend con `asBackend()` consume outbox y llama servicios externos.
5. Cierre:
   marcar evento como `sent`/`failed` y programar reintento.

## 4) Blueprint concreto

### 4.1 Command API (HTTP)

```ts
import { jazzContext } from "./jazz-context";
import { app } from "../schema";

export async function createOrder(req: Request) {
  const db = await jazzContext.forRequest(req, app);

  const result = await db.transaction((tx) => {
    const order = tx.insert(app.orders, { status: "created" });
    tx.insert(app.outbox_events, {
      eventId: crypto.randomUUID(),
      type: "order.created",
      aggregateId: order.value.id,
      status: "pending",
      attempts: 0,
    });
    return order.value;
  });

  await result.wait({ tier: "edge" });
  return Response.json({ ok: true, order: result.value });
}
```

### 4.2 Worker event-driven

```ts
import { dbBackend } from "./jazz-context";
import { app } from "../schema";

export async function processOutboxBatch() {
  const pending = await dbBackend.all(
    app.outbox_events.where({ status: "pending" }).limit(100),
  );

  for (const evt of pending) {
    try {
      // idempotencia por evt.eventId en proveedor externo
      await sendEmailOrNotification(evt);

      dbBackend.update(app.outbox_events, evt.id, {
        status: "sent",
        sentAt: new Date().toISOString(),
      });
    } catch {
      dbBackend.update(app.outbox_events, evt.id, {
        status: "pending",
        attempts: evt.attempts + 1,
        nextRetryAt: computeNextRetry(evt.attempts),
      });
    }
  }
}
```

## 5) Microservicios por cambios de DB: si, pero con reglas

Si quieres disparar microservicios por cambios:

1. Usa eventos de negocio explicitos (outbox), no cambios implicitos de cualquier tabla.
2. Define `eventId` idempotente para evitar duplicados.
3. Maneja retries con backoff.
4. Separa errores transitorios vs permanentes.
5. Evita side effects directos dentro de request critica cuando puedas delegarlos.

## 6) Guia de decision rapida

Elige API HTTP dominante cuando:

1. La accion debe completar antes de responder.
2. Tienes pocos side effects y baja carga.

Elige Event-Driven dominante cuando:

1. Hay muchos consumidores downstream.
2. Necesitas resiliencia y throughput alto.

Elige hibrido (recomendado en la mayoria):

1. HTTP para comandos + autorizacion.
2. Outbox/eventos para integraciones async.

## 7) Que soporta Jazz que ayuda aqui

1. Scopes backend/request para identidad y permisos (`asBackend`, `forRequest`).
2. Writes con `wait({ tier })` para confirmar propagacion por nivel.
3. `transaction` y `batch` para agrupar cambios.
4. `subscribeAll` para consumidores reactivos en procesos vivos.

## 8) Referencias del repo

1. Contexto backend (scopes y auth):
   [packages/jazz-tools/src/backend/create-jazz-context.ts](../packages/jazz-tools/src/backend/create-jazz-context.ts)
2. Exportes publicos backend:
   [packages/jazz-tools/src/backend/index.ts](../packages/jazz-tools/src/backend/index.ts)
3. API de DB (`transaction`, `batch`, `subscribeAll`):
   [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts)
4. Setup backend oficial:
   [docs/content/docs/getting-started/server-setup.mdx](../docs/content/docs/getting-started/server-setup.mdx)
5. Queries/subscriptions:
   [docs/content/docs/reading/queries.mdx](../docs/content/docs/reading/queries.mdx)
6. Sync model y consistencia eventual:
   [docs/content/docs/concepts/how-sync-works.mdx](../docs/content/docs/concepts/how-sync-works.mdx)
7. Ejemplo backend TS:
   [examples/docs/todo-server-ts/src/main.ts](../examples/docs/todo-server-ts/src/main.ts)
