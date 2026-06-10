# React + Jazz: Reading (Queries, Filters/Sorting/Pagination, Includes/Relations)

Este documento abre la division de la Etapa 5 (CRUD reactivo y sync scoping por org), enfocada en lectura.

## 1) Alcance de esta guia

Esta guia cubre solo Reading:

1. Queries.
2. Filters, Sorting & Pagination.
3. Includes & Relations.

La parte de Writing (Writing Data, Files & Blobs) va en el siguiente documento.

## 2) De donde sale esta implementacion en la repo

Base oficial (docs):

1. [docs/content/docs/reading/queries.mdx](../docs/content/docs/reading/queries.mdx)
2. [docs/content/docs/reading/filters-and-sorting.mdx](../docs/content/docs/reading/filters-and-sorting.mdx)
3. [docs/content/docs/reading/includes-and-relations.mdx](../docs/content/docs/reading/includes-and-relations.mdx)

Implementacion real del framework (codigo fuente):

1. Hooks React `useAll` y `useAllSuspense`: [packages/jazz-tools/src/react-core/use-all.ts](../packages/jazz-tools/src/react-core/use-all.ts)
2. DSL de query (`where`, `include`, `requireIncludes`, `orderBy`, `limit`, `offset`, `_build`): [packages/jazz-tools/src/typed-app.ts](../packages/jazz-tools/src/typed-app.ts)
3. Ejecucion de query/suscripcion en runtime (`all`, `one`, `subscribeAll`): [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts)
4. Traduccion a relation IR (`translateBuilderToRelationIr`, `translateQuery`): [packages/jazz-tools/src/runtime/query-adapter.ts](../packages/jazz-tools/src/runtime/query-adapter.ts)

Base practica (React snippets):

1. [examples/docs/todo-client-localfirst-react/src/TodoList.tsx](../examples/docs/todo-client-localfirst-react/src/TodoList.tsx)
2. [examples/docs/todo-client-localfirst-react/src/ConcurrentTodoList.tsx](../examples/docs/todo-client-localfirst-react/src/ConcurrentTodoList.tsx)
3. [examples/docs/todo-client-localfirst-react/src/shared-access-snippets.tsx](../examples/docs/todo-client-localfirst-react/src/shared-access-snippets.tsx)
4. [examples/docs/todo-client-localfirst-react/src/group-permissions-snippets.tsx](../examples/docs/todo-client-localfirst-react/src/group-permissions-snippets.tsx)
5. [starters/react-localfirst/src/todo-widget.tsx](../starters/react-localfirst/src/todo-widget.tsx)
6. [starters/react-hybrid/src/todo-widget.tsx](../starters/react-hybrid/src/todo-widget.tsx)
7. [starters/react-betterauth/src/todo-widget.tsx](../starters/react-betterauth/src/todo-widget.tsx)

## 2.1 Mapa rapido a codigo real

Puntos exactos donde se implementa lo que usas en Reading:

1. `useAllBase` y estado `undefined` hasta resolver: [packages/jazz-tools/src/react-core/use-all.ts](../packages/jazz-tools/src/react-core/use-all.ts#L16)
2. API publica `useAll`: [packages/jazz-tools/src/react-core/use-all.ts](../packages/jazz-tools/src/react-core/use-all.ts#L109)
3. API publica `useAllSuspense`: [packages/jazz-tools/src/react-core/use-all.ts](../packages/jazz-tools/src/react-core/use-all.ts#L130)
4. `where` en DSL: [packages/jazz-tools/src/typed-app.ts](../packages/jazz-tools/src/typed-app.ts#L837)
5. `include` y `requireIncludes` en DSL: [packages/jazz-tools/src/typed-app.ts](../packages/jazz-tools/src/typed-app.ts#L856), [packages/jazz-tools/src/typed-app.ts](../packages/jazz-tools/src/typed-app.ts#L864)
6. `orderBy`, `limit`, `offset` en DSL: [packages/jazz-tools/src/typed-app.ts](../packages/jazz-tools/src/typed-app.ts#L870), [packages/jazz-tools/src/typed-app.ts](../packages/jazz-tools/src/typed-app.ts#L879), [packages/jazz-tools/src/typed-app.ts](../packages/jazz-tools/src/typed-app.ts#L885)
7. Serializacion del query builder (`_build`): [packages/jazz-tools/src/typed-app.ts](../packages/jazz-tools/src/typed-app.ts#L1008)
8. Ejecucion de `db.all`/`db.one`: [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts#L1925)
9. Suscripcion `subscribeAll`: [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts#L2047)
10. Lowering a relation IR: [packages/jazz-tools/src/runtime/query-adapter.ts](../packages/jazz-tools/src/runtime/query-adapter.ts#L711)
11. Lowering de `orderBy`/`offset`/`limit`: [packages/jazz-tools/src/runtime/query-adapter.ts](../packages/jazz-tools/src/runtime/query-adapter.ts#L751), [packages/jazz-tools/src/runtime/query-adapter.ts](../packages/jazz-tools/src/runtime/query-adapter.ts#L772), [packages/jazz-tools/src/runtime/query-adapter.ts](../packages/jazz-tools/src/runtime/query-adapter.ts#L780)
12. Traduccion final a query runtime: [packages/jazz-tools/src/runtime/query-adapter.ts](../packages/jazz-tools/src/runtime/query-adapter.ts#L799)

## 3) Reading: Queries

### 3.1 Que es una query en Jazz

Una query define un conjunto de filas. En React, `useAll(query)` crea una suscripcion reactiva a ese conjunto.

Patron base de la repo:

```tsx
import { useAll } from "jazz-tools/react";
import { app } from "../schema";

const todos = useAll(app.todos);
```

Mismo patron en starters reales:

1. [starters/react-localfirst/src/todo-widget.tsx](../starters/react-localfirst/src/todo-widget.tsx#L6)
2. [starters/react-hybrid/src/todo-widget.tsx](../starters/react-hybrid/src/todo-widget.tsx#L6)
3. [starters/react-betterauth/src/todo-widget.tsx](../starters/react-betterauth/src/todo-widget.tsx#L6)

### 3.2 Estado de carga en React

`useAll(...)` puede devolver `undefined` en el primer fetch. Luego devuelve `[]` o filas.

Patron recomendado:

1. `undefined` -> estado de carga.
2. `[]` -> estado vacio.
3. `rows.length > 0` -> estado con datos.

Referencia:

1. [examples/docs/todo-client-localfirst-react/src/TodoList.tsx](../examples/docs/todo-client-localfirst-react/src/TodoList.tsx)
2. Comportamiento en hook base: [packages/jazz-tools/src/react-core/use-all.ts](../packages/jazz-tools/src/react-core/use-all.ts#L94)

### 3.3 Durabilidad de lectura (`tier`)

El primer resultado de una suscripcion puede pedirse con distintos tiers:

1. `local`: solo storage local. Mas rapido.
2. `edge`: espera sync server cercano.
3. `global`: espera servidor central.

Importante: el tier controla el primer snapshot. Los updates siguientes llegan cuando alcanzan el nodo local.

Comportamiento exacto en suscripciones (`useAll`):

1. `local`: responde primero con lo que ya existe en local (normalmente inmediato) y luego sigue escuchando cambios.
2. `edge`: espera el primer snapshot desde edge y luego tambien sigue escuchando cambios.
3. `global`: espera el primer snapshot desde global y luego tambien sigue escuchando cambios.

Regla practica:

1. El tier no desactiva la reactividad.
2. El tier solo cambia el umbral del primer resultado.

No confundir con lecturas one-shot:

1. `db.all(...)` y `db.one(...)` no quedan suscritas.
2. Solo `useAll(...)` y `subscribeAll(...)` mantienen escucha continua.

Ejemplo:

```tsx
const todosAtEdge = useAll(app.todos, { tier: "edge" });
```

Referencia:

1. [docs/content/docs/reading/queries.mdx](../docs/content/docs/reading/queries.mdx)
2. Entrada de opciones en hooks React: [packages/jazz-tools/src/react-core/use-all.ts](../packages/jazz-tools/src/react-core/use-all.ts#L18)
3. Semantica de tiers y flujo local/edge/global: [docs/content/docs/concepts/how-sync-works.mdx](../docs/content/docs/concepts/how-sync-works.mdx)

### 3.4 Queries condicionales

Si no quieres ejecutar una query en cierto estado de UI, pasa `undefined`.

```tsx
const filtered = useAll(filter ? app.todos.where({ title: { contains: filter } }) : undefined);
```

Referencia:

1. [examples/docs/todo-client-localfirst-react/src/TodoList.tsx](../examples/docs/todo-client-localfirst-react/src/TodoList.tsx)
2. Uso real de query condicional: [examples/docs/todo-client-localfirst-react/src/TodoList.tsx](../examples/docs/todo-client-localfirst-react/src/TodoList.tsx#L32)

### 3.5 Queries scopeadas por org (Etapa 5)

Para evitar mezcla entre orgs, la query debe derivarse de `activeOrgId`.

```tsx
const query = activeOrgId
  ? app.todos.where({ orgId: activeOrgId }).orderBy("$createdAt", "desc")
  : undefined;

const todos = useAll(query, { tier: "edge" });
```

Notas:

1. Si `activeOrgId` es `null`, no se abre suscripcion.
2. Cambiar org activa cambia la query y el snapshot sincronizado.
3. Esto complementa permissions; no las reemplaza.

## 4) Reading: Filters, Sorting & Pagination

### 4.1 Filtros con `where(...)`

Reglas clave que usa Jazz:

1. Las condiciones de `where` se combinan por `AND`.
2. Si necesitas `OR`, resuelvelo con multiples queries o composicion en capa de app.

Ejemplos reales:

```tsx
const pending = useAll(app.todos.where({ done: false }));
const byTitle = useAll(app.todos.where({ title: { contains: search } }));
```

Referencias:

1. [docs/content/docs/reading/filters-and-sorting.mdx](../docs/content/docs/reading/filters-and-sorting.mdx)
2. [examples/docs/todo-client-localfirst-react/src/TodoList.tsx](../examples/docs/todo-client-localfirst-react/src/TodoList.tsx)
3. Construccion de condiciones en DSL: [packages/jazz-tools/src/typed-app.ts](../packages/jazz-tools/src/typed-app.ts#L1036)

### 4.2 Sorting con `orderBy(...)`

Siempre ordena antes de paginar para estabilidad.

```tsx
const sorted = app.todos.where({ done: false }).orderBy("title", "asc");
```

Referencia:

1. [docs/content/docs/reading/filters-and-sorting.mdx](../docs/content/docs/reading/filters-and-sorting.mdx)
2. Lowering de `orderBy` en runtime: [packages/jazz-tools/src/runtime/query-adapter.ts](../packages/jazz-tools/src/runtime/query-adapter.ts#L751)

### 4.3 Paginacion con `limit` y `offset`

Patron real en React:

```tsx
let query = app.todos
  .orderBy("id", "desc")
  .limit(25)
  .offset(page * 25);
```

Referencia:

1. [examples/docs/todo-client-localfirst-react/src/ConcurrentTodoList.tsx](../examples/docs/todo-client-localfirst-react/src/ConcurrentTodoList.tsx)
2. Lowering de `offset`/`limit`: [packages/jazz-tools/src/runtime/query-adapter.ts](../packages/jazz-tools/src/runtime/query-adapter.ts#L772), [packages/jazz-tools/src/runtime/query-adapter.ts](../packages/jazz-tools/src/runtime/query-adapter.ts#L780)

### 4.4 Filtro + paginacion + transiciones en React

La repo muestra este enfoque para UX fluida:

1. `useDeferredValue` para no bloquear input.
2. `useTransition` para cambios de pagina/filtro.
3. `useAllSuspense(query)` para cargar por suspense.

Referencia:

1. [examples/docs/todo-client-localfirst-react/src/ConcurrentTodoList.tsx](../examples/docs/todo-client-localfirst-react/src/ConcurrentTodoList.tsx)
2. Uso real de `useAllSuspense`: [examples/docs/todo-client-localfirst-react/src/ConcurrentTodoList.tsx](../examples/docs/todo-client-localfirst-react/src/ConcurrentTodoList.tsx#L107)

## 5) Reading: Includes & Relations

### 5.1 Includes (`include(...)`)

`include` resuelve relaciones en el mismo resultado.

Ejemplo real:

```tsx
const shares = useAll(
  app.todoShares.where({ user_id: session!.user_id }).include({ todo: true }),
);
```

Referencia:

1. [examples/docs/todo-client-localfirst-react/src/shared-access-snippets.tsx](../examples/docs/todo-client-localfirst-react/src/shared-access-snippets.tsx)
2. Constructor `include(...)` en DSL: [packages/jazz-tools/src/typed-app.ts](../packages/jazz-tools/src/typed-app.ts#L856)

### 5.2 Relations inversas

Jazz tambien soporta relaciones inversas derivadas automaticamente.

Referencia oficial:

1. [docs/content/docs/reading/includes-and-relations.mdx](../docs/content/docs/reading/includes-and-relations.mdx)
2. Restricciones `gather + include` y `hop + include`: [packages/jazz-tools/src/runtime/query-adapter.ts](../packages/jazz-tools/src/runtime/query-adapter.ts#L717), [packages/jazz-tools/src/runtime/query-adapter.ts](../packages/jazz-tools/src/runtime/query-adapter.ts#L720)

### 5.3 Datos faltantes y `requireIncludes()`

Como Jazz es distribuido/offline-first:

1. Una FK puede existir pero la fila relacionada aun no estar sincronizada.
2. Con `include`, Jazz aplica semantica de inner join para esa referencia.
3. `requireIncludes()` fuerza descartar filas si faltan includes forward o la FK es `null`.

Referencia oficial:

1. [docs/content/docs/reading/includes-and-relations.mdx](../docs/content/docs/reading/includes-and-relations.mdx)

### 5.4 Includes + scoping por org

En etapa 5, la regla operativa es:

1. Scopear primero por org/workspace activa (`orgId` o `workspaceId`).
2. Agregar `include(...)` despues, solo en relaciones que realmente necesitas renderizar.

Patron de scoping real en la repo:

```tsx
const docs = useAll(app.documents.where({ workspaceId }));
```

Referencia:

1. [examples/docs/todo-client-localfirst-react/src/group-permissions-snippets.tsx](../examples/docs/todo-client-localfirst-react/src/group-permissions-snippets.tsx#L125)

Patron de include real en la repo:

```tsx
const shares = useAll(
  app.todoShares.where({ user_id: session!.user_id }).include({ todo: true }),
);
```

Referencia:

1. [examples/docs/todo-client-localfirst-react/src/shared-access-snippets.tsx](../examples/docs/todo-client-localfirst-react/src/shared-access-snippets.tsx#L79)

## 6) Checklist de salida de Reading

1. Todas las pantallas de lectura usan query explicita (sin acceso global accidental).
2. Toda query multi-tenant esta scopeada por org activa.
3. Paginacion siempre sobre query ordenada.
4. Includes usados donde realmente reducen round-trips de UI.
5. Estados de carga y vacio diferenciados (`undefined` vs `[]`).

## 7) Relacion con el documento de Etapa 5

Esta guia complementa:

1. [react-sync-por-org.md](./react-sync-por-org.md)

Uso sugerido:

1. Primero leer esta guia para lectura reactiva.
2. Luego cerrar Writing Data y Files & Blobs en el documento siguiente.

## 8) Referencias

1. [docs/content/docs/reading/queries.mdx](../docs/content/docs/reading/queries.mdx)
2. [docs/content/docs/reading/filters-and-sorting.mdx](../docs/content/docs/reading/filters-and-sorting.mdx)
3. [docs/content/docs/reading/includes-and-relations.mdx](../docs/content/docs/reading/includes-and-relations.mdx)
4. [docs/content/docs/concepts/how-sync-works.mdx](../docs/content/docs/concepts/how-sync-works.mdx)
5. [examples/docs/todo-client-localfirst-react/src/TodoList.tsx](../examples/docs/todo-client-localfirst-react/src/TodoList.tsx)
6. [examples/docs/todo-client-localfirst-react/src/ConcurrentTodoList.tsx](../examples/docs/todo-client-localfirst-react/src/ConcurrentTodoList.tsx)
7. [examples/docs/todo-client-localfirst-react/src/shared-access-snippets.tsx](../examples/docs/todo-client-localfirst-react/src/shared-access-snippets.tsx)
8. [examples/docs/todo-client-localfirst-react/src/group-permissions-snippets.tsx](../examples/docs/todo-client-localfirst-react/src/group-permissions-snippets.tsx)
9. [starters/react-localfirst/src/todo-widget.tsx](../starters/react-localfirst/src/todo-widget.tsx)
10. [starters/react-hybrid/src/todo-widget.tsx](../starters/react-hybrid/src/todo-widget.tsx)
11. [starters/react-betterauth/src/todo-widget.tsx](../starters/react-betterauth/src/todo-widget.tsx)
12. [packages/jazz-tools/src/react-core/use-all.ts](../packages/jazz-tools/src/react-core/use-all.ts)
13. [packages/jazz-tools/src/typed-app.ts](../packages/jazz-tools/src/typed-app.ts)
14. [packages/jazz-tools/src/runtime/db.ts](../packages/jazz-tools/src/runtime/db.ts)
15. [packages/jazz-tools/src/runtime/query-adapter.ts](../packages/jazz-tools/src/runtime/query-adapter.ts)