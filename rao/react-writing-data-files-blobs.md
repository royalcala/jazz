# React + Jazz: Writing (Writing Data, Files & Blobs)

Este documento cierra la parte de Writing de la Etapa 5 (CRUD reactivo y sync scoping por org), con foco en implementacion real de la repo.

## 1) Alcance de esta guia

Esta guia cubre solo Writing:

1. `insert`, `update`, `delete`, `upsert`, `restore`.
2. Durabilidad (`wait({ tier })`) y semantica local-first.
3. Transacciones y batches.
4. Validacion en cliente y backend.
5. Rechazo server-side y rollback local.
6. Files & Blobs con tablas convencionales.

## 2) De donde sale esta implementacion en la repo

Base oficial (docs):

1. [docs/content/docs/writing/writing-data.mdx](../docs/content/docs/writing/writing-data.mdx)
2. [docs/content/docs/writing/files-and-blobs.mdx](../docs/content/docs/writing/files-and-blobs.mdx)

Implementacion real del framework (codigo fuente):

1. `Db` (API publica de writes): [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts#L1694)
2. Conversion de payload de write (`toInsertRecord`, `toUpdateRecord`): [packages/jazz-tools/src/runtime/value-converter.ts](../packages/jazz-tools/src/runtime/value-converter.ts#L113)
3. Write handles, `wait`, errores de rechazo: [packages/jazz-tools/src/runtime/client.ts](../packages/jazz-tools/src/runtime/client.ts#L554)
4. Queue de permission checks en server path: [crates/jazz-tools/src/sync_manager/inbox.rs](../crates/jazz-tools/src/sync_manager/inbox.rs#L1508)
5. Evaluacion autoritativa de permisos del write: [crates/jazz-tools/src/query_manager/server_queries.rs](../crates/jazz-tools/src/query_manager/server_queries.rs#L1343)
6. Rechazo aplicado y rollback local: [crates/jazz-tools/src/runtime_core/ticks.rs](../crates/jazz-tools/src/runtime_core/ticks.rs#L231)

Base practica (snippets/apps):

1. CRUD y durabilidad en TS docs snippets: [examples/docs/todo-client-localfirst-ts/src/docs-snippets.ts](../examples/docs/todo-client-localfirst-ts/src/docs-snippets.ts#L223)
2. CRUD y durabilidad en React docs snippets: [examples/docs/todo-client-localfirst-react/src/docs-snippets.ts](../examples/docs/todo-client-localfirst-react/src/docs-snippets.ts#L89)
3. Files & Blobs snippets: [examples/docs/todo-client-localfirst-ts/src/files-and-blobs-snippets.ts](../examples/docs/todo-client-localfirst-ts/src/files-and-blobs-snippets.ts)
4. App completa de upload: [examples/file-upload-react/src/App.tsx](../examples/file-upload-react/src/App.tsx)

## 3) Writing Data: API base

### 3.1 CRUD directo

Patron base de writes:

```ts
db.insert(app.todos, { title: "Write docs", done: false, owner_id: ownerId, projectId });
db.update(app.todos, todoId, { done: true });
db.delete(app.todos, todoId);
```

Referencia real:

1. [examples/docs/todo-client-localfirst-ts/src/docs-snippets.ts](../examples/docs/todo-client-localfirst-ts/src/docs-snippets.ts#L223)
2. Implementacion `Db.insert/update/delete`: [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts#L1694), [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts#L1743), [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts#L1779)

### 3.2 Upsert con ID conocido

Usa `upsert` cuando el ID ya existe en tu dominio:

```ts
db.upsert(app.todos, { title: "Imported", done: false }, { id: importedTodoId });
```

Referencia:

1. [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts#L1744)
2. Path runtime de upsert: [packages/jazz-tools/src/runtime/client.ts](../packages/jazz-tools/src/runtime/client.ts#L1308)

### 3.3 Restore de filas soft-deleted

`restore` vuelve visible una fila borrada logicamente:

```ts
const { value: restored } = db.restore(app.todos, todoId, {
  title: "Restored task",
  done: false,
  owner_id: ownerId,
  projectId,
});
```

Referencia:

1. [examples/docs/todo-client-localfirst-ts/src/docs-snippets.ts](../examples/docs/todo-client-localfirst-ts/src/docs-snippets.ts#L236)
2. Implementacion `Db.restore`: [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts#L1715)

### 3.4 Updates parciales y nullables

`update` solo toca campos enviados.

1. Campos omitidos quedan igual.
2. `undefined` no actualiza.
3. `null` limpia solo columnas nullable.

Referencia:

1. [examples/docs/todo-client-localfirst-ts/src/docs-snippets.ts](../examples/docs/todo-client-localfirst-ts/src/docs-snippets.ts#L252)

## 4) Durabilidad: local-first + wait por tier

En Jazz, el write se aplica local primero y regresa handle inmediato. `wait({ tier })` controla el umbral de confirmacion.

1. `local`: confirma persistencia local.
2. `edge`: confirma llegada al sync server cercano.
3. `global`: confirma propagacion al core/global.

Patron:

```ts
const { id } = await db.insert(app.todos, { title: "task", done: false }).wait({ tier: "edge" });
await db.update(app.todos, id, { done: true }).wait({ tier: "global" });
```

Referencias:

1. [examples/docs/todo-client-localfirst-ts/src/docs-snippets.ts](../examples/docs/todo-client-localfirst-ts/src/docs-snippets.ts#L259)
2. `WriteHandle.wait`: [packages/jazz-tools/src/runtime/client.ts](../packages/jazz-tools/src/runtime/client.ts#L583)
3. `waitForBatch` en cliente runtime: [packages/jazz-tools/src/runtime/client.ts](../packages/jazz-tools/src/runtime/client.ts#L1533)

## 5) Validacion en cliente y backend

Si, hay validacion en ambos lados, pero por capas:

1. Cliente: valida y transforma payload antes de enviar (columnas/tipos basicos).
2. Core/backend: vuelve a validar estructura y aplica autorizacion/policies de forma autoritativa.

### 5.1 Validacion cliente

Conversion/guardrails del lado TypeScript:

1. [packages/jazz-tools/src/runtime/value-converter.ts](../packages/jazz-tools/src/runtime/value-converter.ts#L113)
2. [packages/jazz-tools/src/runtime/value-converter.ts](../packages/jazz-tools/src/runtime/value-converter.ts#L149)

### 5.2 Validacion/authorization backend

Flujo real:

1. El payload entra y se encola para permission check: [crates/jazz-tools/src/sync_manager/inbox.rs](../crates/jazz-tools/src/sync_manager/inbox.rs#L1508)
2. QueryManager server evalua permiso autoritativo: [crates/jazz-tools/src/query_manager/server_queries.rs](../crates/jazz-tools/src/query_manager/server_queries.rs#L1343)
3. Se valida contenido JSON/estructura en path server: [crates/jazz-tools/src/query_manager/server_queries.rs](../crates/jazz-tools/src/query_manager/server_queries.rs#L1505), [crates/jazz-tools/src/query_manager/server_queries.rs](../crates/jazz-tools/src/query_manager/server_queries.rs#L1709)

Regla operativa:

1. La validacion cliente mejora UX.
2. La validacion backend decide aceptacion/rechazo real.

## 6) Que pasa si el servidor rechaza un write

Cuando el backend no autoriza:

1. Se publica `BatchFate::Rejected`.
2. El runtime cliente aplica rollback local de los cambios optimistas.
3. `wait({ tier })` rechaza con error estructurado.
4. Si no esperaste (`wait`), se dispara `onMutationError`.

Referencias clave:

1. Rechazo de batch en server: [crates/jazz-tools/src/sync_manager/permissions.rs](../crates/jazz-tools/src/sync_manager/permissions.rs#L244)
2. Aplicacion local del fate recibido: [crates/jazz-tools/src/runtime_core/ticks.rs](../crates/jazz-tools/src/runtime_core/ticks.rs#L231)
3. Rollback local por filas de batch: [crates/jazz-tools/src/runtime_core/ticks.rs](../crates/jazz-tools/src/runtime_core/ticks.rs#L295)
4. Retraccion de insert/update overlay: [crates/jazz-tools/src/query_manager/indices.rs](../crates/jazz-tools/src/query_manager/indices.rs#L528), [crates/jazz-tools/src/query_manager/manager.rs](../crates/jazz-tools/src/query_manager/manager.rs#L2304)
5. Restore de delete rechazado: [crates/jazz-tools/src/query_manager/indices.rs](../crates/jazz-tools/src/query_manager/indices.rs#L576)
6. Notificacion a waiters: [crates/jazz-tools/src/runtime_core/durability.rs](../crates/jazz-tools/src/runtime_core/durability.rs#L100)

### 6.1 Errores y observabilidad en app

1. `wait` puede lanzar `PersistedWriteRejectedError` con `code` y `reason`.
2. `db.onMutationError(...)` captura rechazos no esperados explicitamente.

Referencias:

1. [packages/jazz-tools/src/runtime/client.ts](../packages/jazz-tools/src/runtime/client.ts#L554)
2. [packages/jazz-tools/src/runtime/client.ts](../packages/jazz-tools/src/runtime/client.ts#L942)
3. [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts#L1645)

## 7) Transacciones y batches

Jazz agrega dos patrones avanzados frente a CRUD clasico:

1. `transaction(...)`: conjunto de writes que se validan/settlean como unidad.
2. `batch(...)`: agrupa writes visibles y confirma juntos.

Referencia API:

1. [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts#L1796)
2. [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts#L1835)

## 8) Files & Blobs

Para binarios grandes no guardes bytes inline en tus filas de dominio; usa el flujo convencional `files` + `file_parts`.

### 8.1 Crear desde Blob/Stream

```ts
const file = await db.createFileFromBlob(app, blob, { tier: "edge" });
await db.insert(app.uploads, { owner_id, label: "Profile", fileId: file.id }).wait({ tier: "edge" });
```

Referencias:

1. [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts#L1984)
2. [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts#L1997)
3. [examples/docs/todo-client-localfirst-ts/src/files-and-blobs-snippets.ts](../examples/docs/todo-client-localfirst-ts/src/files-and-blobs-snippets.ts#L7)

### 8.2 Cargar Blob/Stream

Referencias:

1. [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts#L2009)
2. [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts#L2020)
3. [examples/docs/todo-client-localfirst-ts/src/files-and-blobs-snippets.ts](../examples/docs/todo-client-localfirst-ts/src/files-and-blobs-snippets.ts#L36)

### 8.3 Borrado manual (por ahora)

Hoy no hay cascade automatico completo para esta convencion. Debes borrar `file_parts` y `files` antes del row padre de negocio.

Referencia:

1. [examples/docs/todo-client-localfirst-ts/src/files-and-blobs-snippets.ts](../examples/docs/todo-client-localfirst-ts/src/files-and-blobs-snippets.ts#L57)
2. Ejemplo de app real: [examples/file-upload-react/src/App.tsx](../examples/file-upload-react/src/App.tsx#L154)

## 9) Que tiene Jazz en writes que no es CRUD clasico

Comparado con CRUD clasico request/response al backend:

1. Writes optimistas locales por defecto (UX inmediata).
2. Confirmacion desacoplada por tiers (`local/edge/global`).
3. Rechazo tardio autoritativo con rollback local consistente.
4. `batchId` y destino de durabilidad como parte del modelo de write.
5. Transacciones/batches con settlement en autoridad.
6. Replay de errores de mutacion y ack de rechazados.
7. Flujo de archivos chunked integrado sobre tablas Jazz.

## 10) Checklist de salida de Writing

1. Todas las pantallas de escritura distinguen `write local aplicado` vs `write confirmado`.
2. Hay manejo explicito de `wait` y/o `onMutationError`.
3. Se documenta que permisos definitivos viven en backend.
4. En Files/Blobs, se usan tablas convencionales y borrado manual ordenado.
5. Casos de rechazo server-side estan probados en UX (mensaje y estado final correcto).

## 11) Siguientes subtemas de writing para profundizar

1. Estrategia UX por tier (`local` inmediato vs `edge/global` bloqueante por accion).
2. Politica de retries y mensajes para `PersistedWriteRejectedError`.
3. Diseno de flujos largos con `beginTransaction`/`beginBatch` en pantallas complejas.
4. Patrones de upload grandes (streaming, previsualizacion y limpieza de blobs huerfanos).
5. Pruebas E2E de rollback visible cuando policy deniega en servidor.

## 12) Relacion con otros documentos

1. Reading de Etapa 5: [react-reading-queries.md](./react-reading-queries.md)
2. CRUD reactivo por org: [react-sync-por-org.md](./react-sync-por-org.md)
3. Internals de completitud de query: [react-query-settled-completitud.md](./react-query-settled-completitud.md)
