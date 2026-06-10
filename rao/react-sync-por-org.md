# React + Jazz: CRUD reactivo y sync scoping por org (Etapa 5)

Este documento aterriza la Etapa 5 del roadmap: construir UI reactiva de lectura/escritura y garantizar que el sync quede scopeado por organizacion activa.

## 1) Objetivo de la etapa

1. Implementar CRUD funcional en React sobre Jazz.
2. Evitar mezcla de datos entre orgs en la UI.
3. Mantener seguridad server-side con permissions como guardrail.

## 2) Resultado esperado

Al cerrar esta etapa deberias tener:

1. Componentes CRUD funcionando con `useAll` y `useDb`.
2. Estado de org activa conectado a todas las queries de negocio.
3. Estado de carga/error visible para lectura y acciones.
4. Prueba de cambio de org activa validada extremo a extremo.

## 3) Modelo mental correcto

Jazz sincroniza por suscripciones de query, no por "servidor por org":

1. El cliente abre una suscripcion (`useAll`) sobre una query.
2. El servidor evalua esa query.
3. Solo devuelve filas que cumplen dos condiciones:
    - hacen match con el filtro,
    - pasan permisos de lectura.

Por eso el aislamiento por org se logra con la combinacion de:

1. `orgId` en el modelo de datos.
2. Queries filtradas por org activa.
3. Permissions por membership/rol.

## 4) Precondiciones de Etapa 4

Antes de implementar esta etapa, confirma:

1. Tus tablas de negocio incluyen `orgId` (o equivalente `workspaceId`).
2. Existe tabla puente de membresias (`organizationMembers`/`workspaceMembers`).
3. `permissions.ts` bloquea lectura/escritura cross-org.

Si esto no esta cerrado, la UI puede verse bien pero quedar insegura.

## 5) Patron de implementacion recomendado en React

### 5.1 Fuente de verdad de org activa

1. Define `activeOrgId` en estado global de app (context/store) o en un layout superior.
2. Cambiar org activa debe actualizar todas las queries derivadas.
3. Si no hay org activa, no ejecutes query de negocio.

### 5.2 Query derivada por org

En React con Jazz, una forma robusta es usar query condicional:

```tsx
import { useAll } from "jazz-tools/react";
import { app } from "../schema";

function useOrgTodos(activeOrgId: string | null) {
   const query = activeOrgId
      ? app.todos.where({ orgId: activeOrgId }).orderBy("$createdAt", "desc")
      : undefined;

   return useAll(query);
}
```

Comportamiento esperado:

1. `activeOrgId === null`: no hay consulta, no se abre suscripcion.
2. `activeOrgId` valido: se suscribe solo al subconjunto de esa org.
3. Al cambiar org: React re-renderiza, cambia query y cambia snapshot sincronizado.

### 5.3 Componente CRUD completo (scoped)

```tsx
import { useMemo, useState } from "react";
import { useAll, useDb } from "jazz-tools/react";
import { app } from "../schema";

type OrgTodoWidgetProps = {
   activeOrgId: string | null;
};

export function OrgTodoWidget({ activeOrgId }: OrgTodoWidgetProps) {
   const db = useDb();
   const [actionError, setActionError] = useState<string | null>(null);
   const [isSaving, setIsSaving] = useState(false);

   const query = useMemo(
      () =>
         activeOrgId
            ? app.todos.where({ orgId: activeOrgId }).orderBy("$createdAt", "desc")
            : undefined,
      [activeOrgId],
   );

   const todos = useAll(query);

   function add(formData: FormData) {
      if (!activeOrgId) return;

      const title = String(formData.get("title") ?? "").trim();
      if (!title) return;

      try {
         setActionError(null);
         setIsSaving(true);
         db.insert(app.todos, { orgId: activeOrgId, title, done: false });
      } catch (error) {
         setActionError(error instanceof Error ? error.message : "No se pudo crear el todo");
      } finally {
         setIsSaving(false);
      }
   }

   function toggle(todoId: string, done: boolean) {
      try {
         setActionError(null);
         db.update(app.todos, todoId, { done: !done });
      } catch (error) {
         setActionError(error instanceof Error ? error.message : "No se pudo actualizar el todo");
      }
   }

   function remove(todoId: string) {
      try {
         setActionError(null);
         db.delete(app.todos, todoId);
      } catch (error) {
         setActionError(error instanceof Error ? error.message : "No se pudo borrar el todo");
      }
   }

   if (!activeOrgId) return <p>Selecciona una organizacion para ver tus datos.</p>;
   if (!todos) return <p>Cargando datos de la organizacion activa...</p>;

   return (
      <section>
         <h2>Todos de la org activa</h2>
         <form action={add}>
            <input name="title" placeholder="Nueva tarea" aria-label="Nueva tarea" />
            <button type="submit" disabled={isSaving}>
               {isSaving ? "Guardando..." : "Agregar"}
            </button>
         </form>

         {actionError ? <p role="alert">{actionError}</p> : null}

         {todos.length === 0 ? (
            <p>Sin tareas para esta organizacion.</p>
         ) : (
            <ul>
               {todos.map((todo) => (
                  <li key={todo.id}>
                     <label>
                        <input
                           type="checkbox"
                           checked={todo.done}
                           onChange={() => toggle(todo.id, todo.done)}
                        />
                        <span>{todo.title}</span>
                     </label>
                     <button type="button" onClick={() => remove(todo.id)}>
                        Eliminar
                     </button>
                  </li>
               ))}
            </ul>
         )}
      </section>
   );
}
```

## 6) Manejo de carga y errores (sin sorpresas)

Diferencia tres estados para evitar UX confusa:

1. Sin org activa: no hay query.
2. Query en primer fetch: `useAll(...)` devuelve `undefined`.
3. Resultado estable: `[]` o filas.

En acciones CRUD:

1. Muestra feedback local (`isSaving`, `actionError`).
2. No ocultes errores de autorizacion.
3. Si un write falla por permissions, refleja mensaje y manten UI consistente.

## 7) Prueba clave: cambio de org activa

Esta prueba es obligatoria para cerrar Etapa 5.

### 7.1 Caso minimo manual

1. Usuario U1 pertenece a Org A y Org B.
2. En Org A crea tarea "A-1".
3. Cambia a Org B.
4. "A-1" no debe aparecer.
5. Crea tarea "B-1".
6. Vuelve a Org A.
7. Debe ver "A-1" y no "B-1".

### 7.2 Caso de seguridad

1. Forza en frontend una query sin `orgId` (solo para prueba controlada).
2. Verifica que permissions siguen impidiendo leer filas sin membership.
3. Corrige de inmediato esa query al patron scopeado.

## 8) Errores comunes en esta etapa

1. Mantener `useAll(app.todos)` sin filtro por org activa.
2. Guardar `orgId` en UI pero olvidar enviarlo en `insert`.
3. Confiar en filtro frontend como unica defensa (sin permissions fuertes).
4. No resetear estados transitorios al cambiar org (formularios, seleccion, paginacion).
5. Mezclar aislamiento por org con aislamiento por branch (`env`/`userBranch`).

## 9) Branches vs orgs (recordatorio)

No mezcles responsabilidades:

1. Branches (`env`, `userBranch`) separan ambientes y carriles de trabajo.
2. Orgs separan negocio dentro del mismo ambiente.

Regla operativa:

1. Usa branches para local/staging/prod.
2. Usa `orgId` + permissions para multi-tenant de producto.

## 10) Checklist de salida de Etapa 5

1. Componentes CRUD usan query derivada con `where({ orgId: activeOrgId })`.
2. Todas las escrituras incluyen `orgId` correcto.
3. Hay estados de carga/error para lectura y acciones.
4. Cambio de org activa rehidrata la lista correcta sin mezcla.
5. Permissions bloquean cross-org aunque exista bug de query en frontend.

## 11) Siguiente paso recomendado

Despues de cerrar esta etapa, sigue con Etapa 6 (auth e identidad por camino):

1. Confirmar provider de app segun localfirst/hybrid/betterauth.
2. Conectar sesion/claims con las reglas de permissions ya definidas.
3. Documentar criterio de eleccion del camino de auth.

## 12) Referencias

1. [docs/content/docs/concepts/how-sync-works.mdx](../docs/content/docs/concepts/how-sync-works.mdx)
2. [docs/content/docs/reading/queries.mdx](../docs/content/docs/reading/queries.mdx)
3. [docs/content/docs/auth/permissions.mdx](../docs/content/docs/auth/permissions.mdx)
4. [examples/docs/todo-client-localfirst-react/src/group-permissions-snippets.tsx](../examples/docs/todo-client-localfirst-react/src/group-permissions-snippets.tsx)
5. [starters/react-localfirst/src/todo-widget.tsx](../starters/react-localfirst/src/todo-widget.tsx)
6. [starters/react-hybrid/src/todo-widget.tsx](../starters/react-hybrid/src/todo-widget.tsx)
7. [starters/react-betterauth/src/todo-widget.tsx](../starters/react-betterauth/src/todo-widget.tsx)
8. [react-access-control-permissions.md](./react-access-control-permissions.md)
