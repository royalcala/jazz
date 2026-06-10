# React + Jazz: persistencia interna, modelo de registros, branches, usuarios y visor externo

Esta guia aterriza como persiste Jazz en runtime real, como se modelan fisicamente los registros y que implica conectarte con un visor externo.

## 1) Resumen corto

Si, Jazz usa SQLite en caminos concretos.

1. Node backend con `createJazzContext({ driver: { type: "persistent" } })` usa `NapiRuntime` y almacenamiento persistente en disco (SQLite).
2. El server Rust con `StorageBackend::Persistent` elige backend por features de compilacion: RocksDB primero, luego SQLite, y si no hay backend persistente compilado cae a memoria.
3. Browser persistente no usa SQLite: usa OPFS BTree.

## 2) Donde persiste segun entorno

### Browser

1. Driver persistente local en OPFS BTree.
2. Modelo de clave-valor con namespaces por prefijo.

### Node (NAPI)

1. Runtime nativo en `jazz-napi`.
2. Persistencia SQLite en archivo local.

### Server Rust

1. `StorageBackend::Persistent` selecciona backend segun features disponibles.
2. Puedes fijar backend explicito con `StorageBackend::Sqlite` o `StorageBackend::RocksDb`.

## 3) Modelo fisico real en SQLite

En SQLite no hay una tabla SQL por cada tabla de tu dominio (`users`, `orders`, etc.).

El layout fisico principal es:

1. Una sola tabla `kv(key BLOB PRIMARY KEY, value BLOB)`.
2. `WAL` habilitado, `synchronous=NORMAL`, `WITHOUT ROWID`.
3. Una key de manifiesto (`__jazz_store_manifest`) para validar tipo/formato de store.

Eso significa que cualquier visor SQL externo ve una KV store, no un modelo relacional app-level listo para consultar con joins de dominio.

## 4) Modelo logico sobre KV (como se guardan los registros)

Jazz mapea entidades logicas a keys con prefijos estables.

### Namespaces clave

1. `raw:{table}:{local_key}` para raw tables.
2. `idx:{table}:{column}:{branch}` para indices por columna y branch.
3. `catrow:{objectIdHex}` para entradas del catalogo.
4. `rowtable:{visible|history}:{table}:{schemaHash}` para tablas fisicas de filas visibles e historicas.

### Filas visibles e historicas

1. Vista visible (lectura actual): key por `branch:rowIdHex`.
2. Historial: key por `rowIdHex:branch:batchIdHex`.

### Tablas internas de metadatos

1. `__row_locator`
2. `__visible_row_table_locator`
3. `__history_row_batch_table_locator`
4. `__branch_ord_by_name`
5. `__branch_name_by_ord`
6. `__branch_ord_meta`
7. `__raw_table_header`
8. `__local_batch_record`
9. `__sealed_batch_submission`
10. `__authoritative_batch_settlement`
11. `__acknowledged_rejected_batch`

Estas tablas son parte del motor y no son tablas de dominio de la app.

## 5) Branches y versionado de schema

El branch efectivo que usa Jazz no es solo `main`.

Se compone como:

1. `env`
2. `schemaHash.short()`
3. `userBranch`

Formato: `{env}-{schemaHashShort}-{userBranch}`.

Implicacion importante:

1. Cambios de schema generan nuevos branches efectivos.
2. El almacenamiento separa versiones por ese branch compuesto.
3. El `row_locator` guarda metadatos como tabla origen y schema hash origen para resolver lecturas/migraciones.

## 6) Usuarios y registros: que si existe y que no

### Lo que si existe

1. `Session` con `user_id`, `claims`, `auth_mode` para evaluacion de permisos.
2. Metadatos de autoria (`created_by`, `created_at`) y actualizacion (`updated_by`, `updated_at`) en historia de filas.
3. API backend para cambiar contexto:
   1. `forRequest` y `forSession` (permisos + autoria del usuario).
   2. `withAttribution*` (solo autoria, sin impersonar permisos).

### Lo que no existe como magia del motor

1. No hay una tabla interna universal `users` que Jazz imponga para tus datos de negocio.
2. Columnas como `owner_id`, `author_id`, `assignee_id` son convencion de tu schema/app.

## 7) Se puede conectar un visor externo directo

Si, pero con expectativas correctas.

### Si puedes

1. Abrir el archivo SQLite y hacer inspeccion read-only de keys/valores.
2. Ver estados internos (catalogo, headers, lotes, locators, etc.).
3. Construir un decoder propio para convertir bytes a filas de dominio.

### No esperes

1. Queries SQL de negocio directas como si fuera Postgres con tablas de dominio.
2. Un contrato estable para escribir directo en `kv` sin romper invariantes.

Recomendacion operativa:

1. Para observabilidad funcional, usa API/suscripciones e Inspector.
2. Para debugging profundo, usa lector externo read-only sobre SQLite + decoder.
3. Nunca escribas directo en `kv` desde herramientas externas.

## 8) Checklist de integracion con visor externo

1. Detecta runtime y backend real (OPFS vs SQLite vs RocksDB).
2. Si es SQLite, abre en solo lectura.
3. Inspecciona primero `__jazz_store_manifest` y `__raw_table_header`.
4. Mapea keys por prefijo (`raw:`, `idx:`, `catrow:`).
5. Obtiene descriptores de fila desde catalogo/header para decodificar bytes.
6. Trata branch como compuesto (`env-hash-branch`).
7. No uses escritura directa en store.

## 9) Referencias del repo

1. Backend context TS (driver persistent -> NAPI): [packages/jazz-tools/src/backend/create-jazz-context.ts](../packages/jazz-tools/src/backend/create-jazz-context.ts)
2. NAPI runtime (SQLite-backed storage): [crates/jazz-napi/src/lib.rs](../crates/jazz-napi/src/lib.rs)
3. Seleccion de storage en server Rust: [crates/jazz-tools/src/server/builder.rs](../crates/jazz-tools/src/server/builder.rs)
4. Implementacion SQLite (`kv`, WAL, manifest): [crates/jazz-tools/src/storage/sqlite.rs](../crates/jazz-tools/src/storage/sqlite.rs)
5. Contrato de storage y tablas internas: [crates/jazz-tools/src/storage/storage_trait.rs](../crates/jazz-tools/src/storage/storage_trait.rs)
6. Tipos de storage y row raw table ids: [crates/jazz-tools/src/storage/mod.rs](../crates/jazz-tools/src/storage/mod.rs)
7. Key codec (raw/index/catalogue/visible/history): [crates/jazz-tools/src/storage/key_codec.rs](../crates/jazz-tools/src/storage/key_codec.rs)
8. OPFS BTree storage (browser path): [crates/jazz-tools/src/storage/opfs_btree/mod.rs](../crates/jazz-tools/src/storage/opfs_btree/mod.rs)
9. Branch compuesto y schema hash: [crates/jazz-tools/src/query_manager/types/branch.rs](../crates/jazz-tools/src/query_manager/types/branch.rs)
10. Schema context y armado de branch por query: [crates/jazz-tools/src/schema_manager/context.rs](../crates/jazz-tools/src/schema_manager/context.rs)
11. Session y WriteContext en motor Rust: [crates/jazz-tools/src/query_manager/session.rs](../crates/jazz-tools/src/query_manager/session.rs)
12. Metadata de autoria: [crates/jazz-tools/src/metadata.rs](../crates/jazz-tools/src/metadata.rs)
13. Documentacion de sesiones (column names sin significado especial): [docs/content/docs/auth/sessions.mdx](../docs/content/docs/auth/sessions.mdx)
14. Internals avanzados (modelo table-first y rows visibles/historia): [docs/content/docs/reference/internals.mdx](../docs/content/docs/reference/internals.mdx)
15. Inspector package: [packages/inspector](../packages/inspector)

## 10) Nota de versionado/documentacion

Algunas referencias historicas todavia mencionan Fjall en comentarios o docs viejos, pero la implementacion actual revisada para NAPI persistente esta basada en SQLite.