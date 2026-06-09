# Jazz Tools API map

Este archivo documenta la API publica de jazz-tools por modulo (subpath exports) para entender rapido que importar en cada capa.

## 1) Entrada principal

Import path: jazz-tools

Que incluye:

- DSL de schema: table, col, migrate, getCollectedSchema, etc.
- API tipada de app/schema: defineSchema, defineApp, defineSliceableApp, defineTable.
- Namespace schema como fachada unificada.
- Runtime client y DB (re-export de runtime/index).
- Drivers y tipos de almacenamiento.
- Permisos y utilidades de dev-tools.

Uso recomendado:

- Empezar siempre desde jazz-tools para definir schema y app.
- Pasar a subpaths solo cuando necesites integracion especifica (frontend, backend, dev plugin, testing).

## 2) Backend

Import path: jazz-tools/backend

Exports clave:

- createJazzContext
- JazzContext
- tipos de schema/backend context
- Db y tipos de query (QueryBuilder, QueryOptions, TableProxy)

Uso recomendado:

- Servidores TS/Node que resuelven contexto de sesion y operan DB en backend.

## 3) Frontend por framework

### React

Import path: jazz-tools/react

Exports clave:

- createJazzClient, createExtensionJazzClient
- JazzProvider, JazzClientProvider
- hooks: useDb, useJazzClient, useSession, useAll, useAllSuspense
- auth hooks: useLocalFirstAuth, useAuthState
- attachDevTools

### Vue

Import path: jazz-tools/vue

Exports clave:

- createJazzClient, createExtensionJazzClient
- JazzProvider
- composables: useDb, useJazzClient, useSession, useAll
- auth composable: useLocalFirstAuth
- attachDevTools

### Svelte

Import path: jazz-tools/svelte

Exports clave:

- JazzSvelteProvider
- createJazzClient, createExtensionJazzClient
- contexto: getDb, getSession, getJazzContext
- QuerySubscription
- LocalFirstAuth
- BrowserAuthSecretStore, generateAuthSecret
- attachDevTools

### React core (capa compartida)

Import path: jazz-tools/react-core

Exports clave:

- JazzClientProvider, JazzProvider
- hooks: useDb, useJazzClient, useSession, useAll, useAllSuspense, useAuthState
- createUseLocalFirstAuth

## 4) Mobile

### React Native

Import path: jazz-tools/react-native

Exports clave:

- createJazzClient
- createDb, Db
- loadJazzRn (loader del binding jazz-rn)
- JazzProvider, JazzClientProvider
- hooks: useDb, useSession, useAll, useAllSuspense

### Expo

Import path: jazz-tools/expo

Exports clave:

- useLocalFirstAuth
- ExpoAuthSecretStore y helper expoAuthSecretStore

Import path adicional:

- jazz-tools/expo/polyfills

## 5) Auth integration

Import path: jazz-tools/better-auth-adapter

Export clave:

- jazzAdapter(config)

Uso recomendado:

- Integrar Better Auth sobre DB Jazz en escenarios hybrid/betterauth.

## 6) Runtime y utilidades

Import path recomendado: jazz-tools

Nota:

- runtime es parte publica funcionalmente, pero se consume por re-export desde jazz-tools (no como subpath runtime dedicado en package exports).

Bloques clave:

- JazzClient runtime
- Db, createDb, transacciones y batches
- API de queries y suscripciones
- fetch/publish de schema/permissions
- file storage helper
- worker bridge
- auth secret store

Import path: jazz-tools/permissions

Exports clave:

- definePermissions
- createSessionContext
- relationToIr, relationExistsToPolicy
- anyOf, allOf
- tipos de policy context y relaciones

Import path: jazz-tools/passphrase

- RecoveryPhrase, RecoveryPhraseError

Import path: jazz-tools/passkey-backup

- BrowserPasskeyBackup, PasskeyBackupError

## 7) Dev tooling

Import path: jazz-tools/dev

Exports clave:

- startLocalJazzServer
- pushSchema, pushMigration, pushPermissions, deploy
- watchSchema
- jazzPlugin (Vite)
- withJazz (Next)
- withJazzExpo (Expo)
- jazzSvelteKit (SvelteKit)

Subpaths directos de integracion:

- jazz-tools/dev/vite
- jazz-tools/dev/next
- jazz-tools/dev/expo
- jazz-tools/dev/sveltekit

## 8) Testing

Import path: jazz-tools/testing

Exports clave:

- startLocalJazzServer
- pushSchemaCatalogue
- createPolicyTestApp, PolicyTestApp
- TestingServer (via jazz-napi)

Uso recomendado:

- Tests de integracion black-box (alineado con el estilo del repo) para validar schema, permisos y sincronizacion.

## 9) Checklist rapido de seleccion de modulo

1. Definir schema/app: jazz-tools
2. App web React/Vue/Svelte: jazz-tools/react|vue|svelte
3. App RN/Expo: jazz-tools/react-native y jazz-tools/expo
4. Backend TS: jazz-tools/backend
5. Dev server y push de schema: jazz-tools/dev
6. Tests de integracion: jazz-tools/testing
7. Better Auth: jazz-tools/better-auth-adapter

## 10) Nota de version

Jazz 2.0 esta en alpha; esta superficie puede moverse entre prereleases. Conviene fijar version y revisar changelog al actualizar.

Nota adicional:

- El subpath jazz-tools/_dev/schema-hash existe para integraciones internas de plugins (Next/SvelteKit). No se recomienda usarlo como API de aplicacion.
