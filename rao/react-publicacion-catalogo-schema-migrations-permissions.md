# React + Jazz: Publicacion de catalogo (schema/migrations/permissions) - Etapa 10

Esta guia cubre la Etapa 10 del roadmap: como publicar de forma consistente el catalogo de tu app (schema, migraciones y permissions) sin romper compatibilidad.

## 1) Alcance de esta guia

1. Que publica exactamente el catalogo.
2. Pipeline operativo recomendado para publicar.
3. Diferencia entre publicar por comandos puntuales vs deploy integral.
4. Validaciones previas y checks de estado.
5. Fallos comunes y como resolverlos.

## 2) Fuentes reales en esta repo

CLI publica:

1. validate: [packages/jazz-tools/src/cli.ts](../packages/jazz-tools/src/cli.ts)
2. schema hash/export: [packages/jazz-tools/src/cli.ts](../packages/jazz-tools/src/cli.ts)
3. permissions status: [packages/jazz-tools/src/cli.ts](../packages/jazz-tools/src/cli.ts)
4. migrations push: [packages/jazz-tools/src/cli.ts](../packages/jazz-tools/src/cli.ts)
5. deploy: [packages/jazz-tools/src/cli.ts](../packages/jazz-tools/src/cli.ts)

Implementacion del catalogo (core del flujo):

1. pushSchema/pushPermissions/pushMigration/deploy: [packages/jazz-tools/src/dev/catalogue.ts](../packages/jazz-tools/src/dev/catalogue.ts)
2. exports dev del catalogo: [packages/jazz-tools/src/dev/index.ts](../packages/jazz-tools/src/dev/index.ts)

Docs base:

1. migraciones: [docs/content/docs/schemas/migrations.mdx](../docs/content/docs/schemas/migrations.mdx)
2. permissions (lifecycle de publicacion): [docs/content/docs/auth/permissions.mdx](../docs/content/docs/auth/permissions.mdx)
3. server setup (admin-secret, edge/upstream): [docs/content/docs/getting-started/server-setup.mdx](../docs/content/docs/getting-started/server-setup.mdx)

## 3) Que es publicar el catalogo

En Etapa 10, publicar catalogo significa sincronizar estado declarativo local hacia el server:

1. schema estructural compilado.
2. edge(s) de migracion entre hashes cuando haga falta.
3. permissions bundle apuntando al schema hash correcto.

Idea clave:

1. No es solo subir schema.
2. Debe quedar conectado el grafo de versiones y el head de permissions consistente con ese schema.

## 4) Dos modos de publicacion

### 4.1 Comandos puntuales (granular)

Util cuando quieres controlar paso por paso:

1. validate local.
2. schema hash/export para inspeccion.
3. migrations push puntual.
4. permissions status.

### 4.2 Deploy integral (recomendado en flujo normal)

Comando unico:

1. publica schema si no existe.
2. verifica conectividad desde el schema de permissions anterior al nuevo.
3. empuja migracion faltante cuando corresponde.
4. publica permissions con parent bundle esperado.

Ese flujo esta implementado en [packages/jazz-tools/src/dev/catalogue.ts](../packages/jazz-tools/src/dev/catalogue.ts).

## 5) Runbook recomendado de Etapa 10

### 5.1 Preflight local

```bash
pnpm dlx jazz-tools@alpha validate
pnpm dlx jazz-tools@alpha schema hash
```

Objetivo:

1. confirmar schema/permissions compilan.
2. detectar warning de tablas sin policy explicita.

### 5.2 Revisar estado remoto de permissions

```bash
pnpm dlx jazz-tools@alpha permissions status <appId>
```

Objetivo:

1. saber a que schema hash apunta el head actual.
2. estimar si la publicacion requerira cerrar gap de migracion.

### 5.3 Publicar catalogo

Opcion recomendada:

```bash
pnpm dlx jazz-tools@alpha deploy <appId>
```

Opcion granular (si ya sabes exactamente que hacer):

```bash
pnpm dlx jazz-tools@alpha migrations push <appId> <fromHash> <toHash>
```

Despues, vuelve a ejecutar status para confirmar head final.

## 6) Reglas operativas importantes

1. Permission-only changes no crean schema hash ni requieren migracion.
2. Si schema nuevo no esta conectado al schema anterior de permissions, deploy intenta empujar migracion; si no encuentra archivo y el cambio requiere transformacion, falla con mensaje accionable.
3. Si no hay permissions.ts, deploy publica schema y omite permissions.
4. Si schema ya esta almacenado, se salta publish de schema y continua con el resto.

## 7) A donde se publica (core/global vs edge)

El publish va al server-url que configures en el comando.

Regla recomendada de operacion:

1. En produccion, apunta a core/global (fuente autoritativa del catalogo).
2. Edge con upstream puede propagar, pero no es el target preferido para gobierno de catalogo.

Adicional:

1. deploy y migrations push requieren admin-secret.
2. En modo edge, upstream tambien requiere admin-secret.

Referencia:

1. [docs/content/docs/getting-started/server-setup.mdx](../docs/content/docs/getting-started/server-setup.mdx)

## 8) MCP de Jazz en VS Code (opcional, recomendado)

Que es:

1. El paquete jazz-tools trae un servidor MCP para exponer docs de Jazz a asistentes AI compatibles.
2. Sirve para que el asistente consulte APIs reales de tu version instalada y reduzca respuestas desalineadas.

Cuando ayuda en este roadmap:

1. Etapas de lectura/escritura para buscar APIs concretas rapido.
2. Etapa 9/10 para validar comandos y lifecycle de schema/migrations/permissions.
3. Etapa 11 para auth/JWT/JWKS sin depender de memoria del modelo.

Herramientas MCP que expone:

1. list_pages
2. search_docs
3. get_doc

Requisitos y notas:

1. Node 22.12+ recomendado por el propio docs de MCP.
2. Es tooling de DX, no cambia runtime ni seguridad de produccion.

Referencia oficial:

1. https://jazz.tools/docs/reference/mcp

## 9) Errores comunes y resolucion

1. Missing server URL/admin secret:
   - Configura server-url y admin-secret (o variables de entorno).
2. New permissions schema not connected:
   - crea/pusha migracion between hashes y reintenta deploy.
3. No migration file found:
   - valida si realmente hay transformacion de filas; si si, genera/revisa archivo y pushea.
4. Server permissions head apunta a hash viejo:
   - revisa status, luego deploy para retarget del head.

## 10) Checklist de salida de Etapa 10

1. Pipeline de catalogo definido (preflight + publish + post-check).
2. Equipo entiende cuando usar deploy integral vs comandos granulares.
3. Head de permissions validado despues de cada publicacion.
4. Procedimiento documentado para desconexion de hashes y push de migraciones.
5. Publicacion en entorno correcto (preferentemente core/global en produccion).

## 11) Relacion con etapas vecinas

1. Depende de Etapa 9 (migraciones): [react-migraciones-evolucion-schema.md](./react-migraciones-evolucion-schema.md)
2. Prepara Etapa 11 (operacion auth server/JWT/JWKS).
