# React + Jazz: Etapa 1 - branches y entornos

Este documento aterriza la Etapa 1 del roadmap: definir aislamiento de datos antes de modelar schema.

## 1) Objetivo de la etapa

1. Evitar mezcla de datos entre desarrollo, staging y produccion.
2. Definir una convención clara de `env` y `userBranch` por ambiente.
3. Acordar como usar branches para QA, pruebas y preproduccion.

## 2) Explicación simple (30 segundos)

Piensa en Jazz como cajones separados de datos.

1. Tu defines dos llaves del cajon: `env` y `userBranch`.
2. Jazz agrega una tercera llave automaticamente: `schemaHash`.
3. Solo ves datos del cajon donde estas parado.

Ejemplo rapido:

1. Si escribes en `env=dev` y `userBranch=main`, ese dato no aparece en `env=prod`.
2. Si escribes en `env=staging` y `userBranch=qa`, ese dato no aparece en `env=staging` y `userBranch=main`.

## 3) Como funciona un branch en Jazz

En Jazz, la rama efectiva combina tres partes:

1. `env` (lo define tu app): por ejemplo `dev`, `staging`, `prod`.
2. `schemaHash` (lo genera Jazz): hash de la version del schema.
3. `userBranch` (lo define tu app): por ejemplo `main`, `qa`, `release-candidate`.

Forma resultante:

1. `env-schemaHash-userBranch`

## 4) Regla mental importante

1. `env` y `userBranch` estan totalmente aislados entre si.
2. Las versiones por `schemaHash` se componen en lectura via migraciones.
3. Primero defines `env` y `userBranch`; luego schema/migrations se montan encima.

## 5) Ejemplo paso a paso (que pasa en la practica)

1. Abres tu app local con `env=dev` y `userBranch=main`.
2. Creas un todo: "Comprar café".
3. Cambias a `env=staging` y `userBranch=qa`.
4. Ese todo no aparece, porque ahora estas en otro espacio de datos.
5. Vuelves a `env=dev` y `userBranch=main`.
6. El todo vuelve a aparecer.

## 6) Convencion sugerida para tu equipo

1. Local dev: `env=dev`, `userBranch=main`.
2. QA funcional: `env=staging`, `userBranch=qa`.
3. Release candidate: `env=staging`, `userBranch=rc`.
4. Produccion: `env=prod`, `userBranch=main`.

## 7) Snippet base de configuracion

```tsx
createJazzClient({
  appId,
  serverUrl,
  env: "staging",
  userBranch: "qa",
});
```

## 8) Checklist de salida de Etapa 1

1. Existe convención documentada de `env` y `userBranch`.
2. Cada ambiente tiene un branch objetivo definido.
3. CI/CD y configuracion de deploy respetan esa convención.
4. El equipo entiende que `env/userBranch` no se mezclan en query.

## 9) Siguiente paso recomendado

Despues de cerrar esta etapa, sigue con Etapa 2 (schema de datos):

1. Definir tablas y relaciones de dominio.
2. Exportar `app` en `schema.ts`.
3. Conectar UI y backend al mismo contrato.

## 10) Referencias

1. [docs/content/docs/concepts/branches.mdx](../docs/content/docs/concepts/branches.mdx)
2. [docs/content/partials/create-jazz-client-reference.mdx](../docs/content/partials/create-jazz-client-reference.mdx)
3. [examples/docs/todo-client-localfirst-react/src/branch-snippets.tsx](../examples/docs/todo-client-localfirst-react/src/branch-snippets.tsx)
