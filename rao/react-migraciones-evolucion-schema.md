# React + Jazz: Migraciones y evolucion de schema (Etapa 9)

Esta guia cubre la Etapa 9 del roadmap: como evolucionar schema sin romper compatibilidad entre clientes en distintas versiones.

## 1) Alcance de esta guia

1. Modelo mental de migraciones en Jazz (hash + lens + branches por schema).
2. Cuando necesitas archivo de migracion y cuando no.
3. Flujo operativo real con `jazz-tools` (`create`, `push`, `deploy`).
4. Uso de `fromHash` y `toHash` para historial viejo.
5. Estructura de `defineMigration` con ejemplos reales.

## 2) De donde sale esta implementacion en la repo

Base oficial:

1. [docs/content/docs/schemas/migrations.mdx](../docs/content/docs/schemas/migrations.mdx)

Implementacion real del tooling:

1. `defineMigration(...)`: [packages/jazz-tools/src/migrations.ts](../packages/jazz-tools/src/migrations.ts#L971)
2. Flujo `migrations create`: [packages/jazz-tools/src/cli.ts](../packages/jazz-tools/src/cli.ts#L1031)
3. Flujo `migrations push`: [packages/jazz-tools/src/cli.ts](../packages/jazz-tools/src/cli.ts#L1157)

Ejemplos reales:

1. Workflow shell: [examples/docs/todo-server-ts/docs/migrations-workflow.sh](../examples/docs/todo-server-ts/docs/migrations-workflow.sh)
2. Migracion agregando columna con default: [examples/docs/todo-server-ts/migrations/20260318-add-description-a01f5c72ec47-311995e9a178.ts](../examples/docs/todo-server-ts/migrations/20260318-add-description-a01f5c72ec47-311995e9a178.ts)
3. Migracion dropeando columna con `backwardsDefault`: [examples/docs/todo-server-ts/migrations/20260318-drop-legacy-priority-311995e9a178-73b65d082ab8.ts](../examples/docs/todo-server-ts/migrations/20260318-drop-legacy-priority-311995e9a178-73b65d082ab8.ts)

## 3) Modelo mental correcto en Jazz

Jazz no opera como migracion clasica de "reescribir toda la base y cortar".

En su lugar:

1. Cada version de `schema.ts` tiene hash.
2. Los datos quedan particionados por rama de schema hash.
3. Las migraciones son "lenses" que traducen entre versiones en lectura/escritura.
4. Clientes viejos y nuevos pueden convivir mientras exista camino de lenses.

Punto clave:

1. Cambiar schema no implica reescribir inmediatamente todas las filas historicas.
2. La compatibilidad depende de tener camino de migracion publicado entre hashes relevantes.

## 4) Cuando hace falta migracion y cuando no

Regla operativa del CLI:

1. Si no hay transformacion de filas requerida, no necesitas archivo revisado de migracion.
2. Si hay transformacion de filas, debes crear/revisar/pushear migracion.

Esto se ve en el flujo de `createMigration`:

1. [packages/jazz-tools/src/cli.ts](../packages/jazz-tools/src/cli.ts#L1089)
2. [packages/jazz-tools/src/cli.ts](../packages/jazz-tools/src/cli.ts#L1111)

## 5) Jazz vs ORM SQL clasico (Drizzle, etc.)

Se parece en la experiencia de equipo, pero no es el mismo modelo interno.

Parecido:

1. Mantienes `schema.ts` declarativo como fuente actual.
2. Vas acumulando historial de migraciones entre versiones.
3. Usas comandos de tooling para generar/publicar.

Diferente en Jazz:

1. No es solo SQL imperativo one-way.
2. Las migraciones son edges/lenses entre hashes de schema.
3. El runtime usa ese camino para compatibilidad entre clientes viejos y nuevos.
4. No todo cambio estructural exige archivo de migracion manual.

## 6) Migraciones manuales o generadas

Respuesta corta: ambas.

1. Jazz genera el stub automaticamente con `migrations create` cuando corresponde.
2. El equipo revisa y ajusta manualmente el bloque `migrate` cuando hay logica de transformacion.
3. Si el cambio no requiere transformacion de filas, puede no generarse archivo revisado y aun asi se puede publicar el edge.

Donde se ve esto en el tooling:

1. Flujo de generacion y decisiones en `createMigration`: [packages/jazz-tools/src/cli.ts](../packages/jazz-tools/src/cli.ts#L1031)
2. Mensaje de "no reviewed migration file needed": [packages/jazz-tools/src/cli.ts](../packages/jazz-tools/src/cli.ts#L1089)
3. API declarativa de definicion: [packages/jazz-tools/src/migrations.ts](../packages/jazz-tools/src/migrations.ts#L971)

## 7) Matriz de decision rapida (caso -> accion)

| Caso | Archivo de migracion manual | Que ejecutar | Revision manual |
| --- | --- | --- | --- |
| Solo cambio en `permissions.ts` | No | `deploy` (o flujo de permissions) | No |
| Cambio estructural simple sin transformacion de filas (ej. add nullable) | Normalmente no | `migrations create` y luego `migrations push`/`deploy` | Baja |
| Cambio estructural con transformacion de datos | Si | `migrations create --name ...`, editar `migrate`, `migrations push` | Si |
| Drop de columna que clientes viejos aun esperan | Si (con `backwardsDefault`) | `migrations create`, ajustar `migrate`, `migrations push` | Si |
| Rename ambiguo (add/drop compatible que puede confundirse) | Si | `migrations create`, resolver draft lens, `migrations push` | Si, obligatoria |
| Datos historicos no alcanzables desde schema actual | Si (generalmente) | `migrations create --fromHash ... --toHash ...`, luego `migrations push` | Si |

Regla operativa:

1. Si `migrations create` detecta que no hay transformacion de filas, puede no generar archivo revisado.
2. Si hay transformacion o ambiguedad, revisa y edita manualmente antes de publicar.
3. Para produccion, publica preferentemente contra core/global (fuente autoritativa).

## 8) Flujo recomendado (dia a dia)

### 8.1 Primera vez (baseline)

```bash
pnpm dlx jazz-tools@alpha migrations create
```

Esto crea snapshot inicial y no genera migration file si no hay baseline anterior.

Referencia:

1. [packages/jazz-tools/src/cli.ts](../packages/jazz-tools/src/cli.ts#L1076)

### 8.2 Cambio normal de schema

1. Edita `schema.ts`.
2. (Opcional) valida:

```bash
pnpm dlx jazz-tools@alpha validate
```

3. Crea stub:

```bash
pnpm dlx jazz-tools@alpha migrations create --name add-description
```

4. Revisa/edita `migrate`.
5. Publica:

```bash
pnpm dlx jazz-tools@alpha migrations push <appId> <fromHash> <toHash>
```

Workflow de referencia:

1. [examples/docs/todo-server-ts/docs/migrations-workflow.sh](../examples/docs/todo-server-ts/docs/migrations-workflow.sh)

### 8.3 Publicar todo en una sola pasada

```bash
pnpm dlx jazz-tools@alpha deploy <appId>
```

`deploy` coordina schema + migracion (si hace falta) + permissions.

## 9) Migrar historial viejo con from/to explicitos

Si tienes datos viejos no alcanzables desde schema actual, crea edge especificando hashes:

```bash
pnpm dlx jazz-tools@alpha migrations create <appId> --fromHash <fromHash>
pnpm dlx jazz-tools@alpha migrations create <appId> --fromHash <fromHash> --toHash <toHash>
```

Notas:

1. `toHash` por default es schema local actual.
2. Si falta snapshot local, puede resolver hashes desde servidor y guardarlos.

Referencia:

1. [docs/content/docs/schemas/migrations.mdx](../docs/content/docs/schemas/migrations.mdx)
2. [packages/jazz-tools/src/cli.ts](../packages/jazz-tools/src/cli.ts#L1043)

## 10) Anatomia de una migracion

Una migracion se define con `defineMigration`:

1. `fromHash`.
2. `toHash`.
3. `from` schema.
4. `to` schema.
5. `migrate` declarativo por tabla/columna.

Referencia API:

1. [packages/jazz-tools/src/migrations.ts](../packages/jazz-tools/src/migrations.ts#L971)

Ejemplo real (agregar columna):

1. [examples/docs/todo-server-ts/migrations/20260318-add-description-a01f5c72ec47-311995e9a178.ts](../examples/docs/todo-server-ts/migrations/20260318-add-description-a01f5c72ec47-311995e9a178.ts)

Ejemplo real (drop con compatibilidad hacia atras):

1. [examples/docs/todo-server-ts/migrations/20260318-drop-legacy-priority-311995e9a178-73b65d082ab8.ts](../examples/docs/todo-server-ts/migrations/20260318-drop-legacy-priority-311995e9a178-73b65d082ab8.ts)

## 11) Comandos utiles para inspeccion

Hash local actual:

```bash
pnpm dlx jazz-tools@alpha schema hash
```

Export de schema compilado:

```bash
pnpm dlx jazz-tools@alpha schema export
```

Estos comandos ayudan a comparar estado local vs publicado antes de crear/pushar migraciones.

## 12) Errores comunes en Etapa 9

1. Cambiar schema y asumir que todo historico queda automaticamente legible sin camino de lens.
2. Ignorar warning de datos no alcanzables por schema actual.
3. Push de permissions apuntando a hash nuevo sin cerrar gap de migracion cuando aplica.
4. Tratar migracion como script imperativo ad-hoc en lugar de edge declarativo revisado.

## 13) Checklist de salida de Etapa 9

1. Existe proceso repetible para `create -> review -> push`.
2. El equipo distingue cambios con/sin transformacion de filas.
3. Se documentan `fromHash` y `toHash` por edge publicado.
4. Hay comandos de verificacion previos (`schema hash`, `validate`, `schema export`).
5. Existe criterio para usar `deploy` integral vs `migrations push` puntual.

## 14) Relacion con otras etapas

1. Etapa 1 (branches): [react-branches-entornos.md](./react-branches-entornos.md)
2. Etapa 2 (schema): [react-schema-datos-y-flujo.md](./react-schema-datos-y-flujo.md)
3. Etapa 10 (catalogo): siguiente paso natural tras estabilizar migraciones.
