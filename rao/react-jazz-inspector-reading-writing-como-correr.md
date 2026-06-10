# React + Jazz: Jazz Inspector (reading, writing y como correrlo)

Esta guia resume que puedes hacer con Jazz Inspector, que necesitas para ejecutarlo y como operarlo en modo web o extension.

## 1) Que es Jazz Inspector

Jazz Inspector es una UI para inspeccionar y operar datos de Jazz.

Puede correr en dos modos:

1. App web standalone.
2. Panel de Chrome DevTools (extension).

## 2) Es solo lectura o tambien escritura

No es solo lectura. En Data Explorer permite:

1. Leer tablas con filtros, orden, paginacion y relaciones.
2. Insertar filas nuevas.
3. Editar filas existentes.
4. Borrar filas.
5. Acumular cambios en cola y confirmar con Save changes.

Limitaciones:

1. Campos binarios (`Bytea`) son read-only en el formulario del inspector.

## 3) Requisitos para correrlo

### Requisitos base

1. Node.js `>=22.12`.
2. `pnpm` instalado.
3. Dependencias del monorepo instaladas (`pnpm install` en raiz).

### Para modo standalone

Necesitas datos de conexion al server Jazz:

1. `serverUrl`
2. `appId`
3. `adminSecret`
4. `env` (opcional, por defecto `dev`)
5. `branch` (opcional, por defecto `main`)

### Para modo extension

1. Chrome/Chromium con `chrome://extensions`.
2. Build de extension generado desde `packages/inspector`.
3. Una app Jazz activa para que el panel detecte runtime.

## 4) Como correr en modo standalone

Desde la raiz del repo:

```bash
pnpm -C packages/inspector dev
```

Luego abre:

1. `http://localhost:5173`

Y configura la conexion con `serverUrl`, `appId`, `adminSecret`, `env`, `branch`.

## 5) Como buildar y cargar la extension

Desde la raiz del repo:

```bash
pnpm -C packages/inspector build:extension
```

Despues en Chrome:

1. Abrir `chrome://extensions`.
2. Activar Developer mode.
3. Load unpacked apuntando a `packages/inspector/dist-extension`.
4. Abrir DevTools en una pagina con runtime Jazz.
5. Ir al panel Jazz Inspector.

## 6) Que vistas usar para cada necesidad

### Data Explorer

1. Navegar tablas.
2. Filtrar/sortear/paginar.
3. Insert/update/delete.
4. Revisar schema por tabla.

### Live Query

1. Observar suscripciones activas.
2. Ver tablas, tiers, propagacion y estado de queries.
3. Trazar que consulta esta viva y donde.

## 7) Nota de durabilidad por modo

En el grid de mutaciones:

1. Standalone espera mutaciones con tier `edge`.
2. Extension espera mutaciones con tier `local`.

Esto afecta cuando el inspector considera confirmado un write.

## 8) Troubleshooting rapido

1. Si la extension abre pero no muestra datos: valida que haya runtime Jazz activo en esa pagina.
2. Si standalone conecta pero no lista tablas: valida `appId`, `adminSecret`, `env` y `branch`.
3. Si no arranca el paquete: valida Node `>=22.12` y que `pnpm install` haya corrido en raiz.

## 9) Referencias del repo

1. README del Inspector: [packages/inspector/README.md](../packages/inspector/README.md)
2. Scripts/dependencias/engine: [packages/inspector/package.json](../packages/inspector/package.json)
3. Rutas del Inspector (Data Explorer + Live Query): [packages/inspector/src/routes.tsx](../packages/inspector/src/routes.tsx)
4. Grid con mutaciones (`insert`, `update`, `delete`): [packages/inspector/src/components/data-explorer/TableDataGrid.tsx](../packages/inspector/src/components/data-explorer/TableDataGrid.tsx)
5. Reglas de formulario y campos read-only: [packages/inspector/src/components/data-explorer/row-mutation-form.ts](../packages/inspector/src/components/data-explorer/row-mutation-form.ts)
6. Contexto runtime/propagacion (`standalone` vs `extension`): [packages/inspector/src/contexts/devtools-context.tsx](../packages/inspector/src/contexts/devtools-context.tsx)