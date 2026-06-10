# React + Jazz: Data patterns de modelado (Etapa 3)

Este documento aterriza la Etapa 3 del roadmap: elegir el patron de modelado correcto antes de cerrar access control y permissions.

## 1) Objetivo de la etapa

1. Elegir un patron de datos base para tu producto.
2. Evitar rediseños grandes al llegar a permissions.
3. Dejar listo el terreno para auth y reglas de acceso.

## 2) Explicacion simple

Un data pattern es la forma en que estructuras entidades y relaciones para que luego:

1. Las consultas sean naturales.
2. Los permisos sean expresables.
3. El trabajo colaborativo sea predecible.

Si eliges mal el patron, luego access control se vuelve dificil o confuso.

## 3) Patrones principales en Jazz

### 3.1 User-owned data

Cada usuario ve solo sus filas.

Cuando usarlo:

1. App personal.
2. Datos sin colaboracion.

### 3.2 Shared access

Un dueño comparte acceso con otros usuarios concretos.

Cuando usarlo:

1. Compartir documentos o tableros por invitacion.
2. Colaboracion puntual entre pocas personas.

### 3.3 Group permissions (org/workspace)

Modelo de organizacion o workspace con tabla de membresias y roles.

Cuando usarlo:

1. Producto multi-tenant tipo SaaS B2B.
2. Necesitas roles: admin, writer, reader, etc.

### 3.4 Nested data con herencia de permisos

Jerarquia tipo Project -> Task -> Comment donde acceso hereda desde arriba.

Cuando usarlo:

1. Tienes jerarquia clara de dominio.
2. Quieres minimizar duplicacion de reglas.

### 3.5 Real-time collaborative list

Multiples usuarios editan la misma coleccion y ven cambios en vivo.

Cuando usarlo:

1. Colaboracion simultanea.
2. Necesitas UX tipo "todos ven lo mismo al instante".

## 4) Recomendacion para app con orgs

Si tu producto maneja organizaciones, la base mas robusta suele ser combinar:

1. Group permissions para ownership y roles.
2. Nested data para heredar acceso en entidades hijas.
3. Patrón colaborativo para vistas en tiempo real.

Blueprint minimo sugerido:

1. organizations
2. memberships
3. projects
4. tasks
5. comments

Con esta forma, en Etapa 4 (access control) puedes expresar reglas limpias con membership + herencia.

## 5) Matriz de decision rapida

1. App personal sin equipos: user-owned.
2. App con compartir puntual: shared access.
3. App de equipos/empresa: group permissions.
4. Dominio jerarquico fuerte: nested data.
5. Edicion concurrente frecuente: collaborative list.

## 6) Errores comunes

1. Empezar con user-owned y luego intentar meter orgs sin tabla de memberships.
2. Mezclar ownership y sharing en una sola columna ambigua.
3. Modelar jerarquias sin decidir herencia de acceso.
4. Pasar a permissions sin congelar primero el patron de datos.

## 7) Checklist de salida de Etapa 3

1. Patron principal elegido.
2. Patrones secundarios definidos (si aplica).
3. Entidades y relaciones ajustadas al patron.
4. Casos de lectura/escritura principales comprobados contra ese patron.
5. El equipo entiende por que se eligio ese patron y no otro.

## 8) Siguiente paso recomendado

Despues de cerrar esta etapa, sigue con Etapa 4 (access control y permissions):

1. Traducir el patron elegido a politicas read/insert/update/delete.
2. Definir ownership, colaboracion y reglas por rol.
3. Cubrir casos permitidos y denegados con pruebas.

## 9) Referencias oficiales

1. [docs/content/docs/recipes/data-patterns/nested-data.mdx](../docs/content/docs/recipes/data-patterns/nested-data.mdx)
2. [docs/content/docs/recipes/data-patterns/real-time-collaborative-list.mdx](../docs/content/docs/recipes/data-patterns/real-time-collaborative-list.mdx)
3. [docs/content/docs/recipes/access-control/user-owned-data.mdx](../docs/content/docs/recipes/access-control/user-owned-data.mdx)
4. [docs/content/docs/recipes/access-control/shared-access.mdx](../docs/content/docs/recipes/access-control/shared-access.mdx)
5. [docs/content/docs/recipes/access-control/group-permissions.mdx](../docs/content/docs/recipes/access-control/group-permissions.mdx)
6. [docs/content/docs/concepts/local-first-data-model.mdx](../docs/content/docs/concepts/local-first-data-model.mdx)
