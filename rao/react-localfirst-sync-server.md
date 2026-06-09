# React localfirst: Sync Server (detalle)

Este documento explica el Sync Server en el Camino 1 (React localfirst): que es, cuando corre y si es lo unico que necesitas levantar.

## 1) Que es el Sync Server

1. Es el servicio de sincronizacion de Jazz.
2. Recibe cambios del cliente y replica/entrega estado segun permisos.
3. Es el backend de datos para tu app localfirst.

## 2) Es lo unico que se ocupa correr

Respuesta corta: no exactamente.

En desarrollo local (Camino 1):

1. Tu ejecutas un solo comando: pnpm dev.
2. Ese comando levanta Vite (frontend).
3. El plugin jazzPlugin arranca automaticamente el Sync Server local.

Entonces, operativamente corres un comando; tecnicamente hay frontend + sync server.

## 3) Evidencia en el starter

1. El starter usa jazzPlugin en [starters/react-localfirst/vite.config.ts](../starters/react-localfirst/vite.config.ts).
2. El README del starter indica que el server local de Jazz se inicia automaticamente con el plugin en [starters/react-localfirst/README.md](../starters/react-localfirst/README.md).
3. Tambien indica que inyecta VITE_JAZZ_APP_ID y VITE_JAZZ_SERVER_URL en .env en primer arranque, en [starters/react-localfirst/README.md](../starters/react-localfirst/README.md).

## 4) Que hace internamente el plugin

En desarrollo, jazzPlugin:

1. Inicializa runtime gestionado en configureServer.
2. Arranca la infraestructura de sync para dev.
3. Escribe VITE_JAZZ_APP_ID y VITE_JAZZ_SERVER_URL para que React conecte.
4. Libera recursos al cerrar el server de Vite.

Referencia de implementacion: [packages/jazz-tools/src/dev/vite.ts](../packages/jazz-tools/src/dev/vite.ts).

## 5) Produccion: que debes correr

Depende del hosting:

1. Cloud hosted:
- Frontend desplegado.
- Sync server lo provee Jazz Cloud.
- Configuras variables cloud.

2. Self-hosted:
- Frontend desplegado.
- Debes correr tu propio Jazz sync server.
- En localfirst, habilitar allow-local-first-auth en produccion.

Referencia: [starters/react-localfirst/README.md](../starters/react-localfirst/README.md).

## 6) Flujo minimo para Camino 1

1. npm create jazz@latest mi-app -- --starter react-localfirst --hosting selfhosted
2. cd mi-app
3. pnpm install
4. pnpm dev
5. Abrir http://localhost:5173

Con eso ya tienes frontend + sync server activos para desarrollo.

## 7) Preguntas rapidas

1. Puedo correr solo frontend sin sync server:
- Para CRUD real sincronizado, no.

2. Puedo correr solo sync server sin frontend:
- Si, pero no veras app cliente.

3. En Camino 1 necesito servidor de auth aparte:
- No. Eso aparece en hybrid/betterauth.

## 8) Traza request por request (CRUD)

Para ubicar el codigo del flujo:

1. Entrada y provider: [starters/react-localfirst/src/main.tsx](../starters/react-localfirst/src/main.tsx).
2. Operaciones CRUD: [starters/react-localfirst/src/todo-widget.tsx](../starters/react-localfirst/src/todo-widget.tsx).
3. Reglas de autorizacion: [starters/react-localfirst/permissions.ts](../starters/react-localfirst/permissions.ts).

### 8.1 Insert

1. Usuario envia formulario Add en la UI.
2. Se ejecuta db.insert(app.todos, { ... }) en el cliente.
3. El runtime envia la mutacion con identidad local del dispositivo.
4. El sync server valida la identidad de sesion.
5. Se evalua permisos: allowInsert.always.
6. Si pasa, persiste fila y la asocia internamente al creador ($createdBy).

### 8.2 Read

1. Cliente suscribe/consulta con useAll(app.todos).
2. Sync server evalua policy.todos.allowRead.where({ $createdBy: session.user_id }).
3. Solo devuelve filas del usuario/sesion actual.

### 8.3 Update

1. Usuario marca checkbox en todo.
2. Se ejecuta db.update(app.todos, id, { done: ... }).
3. Sync server valida identidad actual.
4. Evalua allowUpdate.where({ $createdBy: session.user_id }).
5. Solo actualiza si la fila pertenece a esa sesion.

### 8.4 Delete

1. Usuario presiona boton de borrado.
2. Se ejecuta db.delete(app.todos, id).
3. Sync server valida identidad actual.
4. Evalua allowDelete.where({ $createdBy: session.user_id }).
5. Solo elimina si la fila pertenece a esa sesion.

## 9) Como valida el backend que eres tu

1. El cliente usa useLocalFirstAuth para obtener secret local (identidad local).
2. JazzProvider se configura con appId, serverUrl y secret.
3. El backend valida pruebas criptograficas derivadas de ese secret.
4. Con eso construye session.user_id confiable en servidor.
5. Los permisos usan session.user_id para autorizar cada operacion.

Puntos clave:

1. La clave privada no se comparte como dato libre del cliente.
2. El backend no confia en un user_id declarado por frontend.
3. La autorizacion final se decide en permissions.ts.

## 10) Señales de que el sync server esta sano

1. pnpm dev inicia sin errores de plugin jazz.
2. Se inyectan variables VITE_JAZZ_APP_ID y VITE_JAZZ_SERVER_URL.
3. Puedes crear un todo y aparece de inmediato.
4. Al recargar, el dato sigue disponible para la misma identidad local.

## 11) Fallas comunes y causa probable

1. Falta VITE_JAZZ_APP_ID o VITE_JAZZ_SERVER_URL:
- El plugin no inicializo o no pudo escribir/inyectar entorno.

2. Insert funciona pero no ves datos al volver:
- Cambio de identidad local (storage limpiado o sesion distinta).

3. En produccion self-hosted falla auth localfirst:
- Falta habilitar allow-local-first-auth en el server.

4. Quieres portabilidad de cuenta entre dispositivos:
- Este camino no la da por defecto, pasar a hybrid o usar backup/restore de identidad.

## 12) Siguiente profundizacion recomendada

1. Revisar [starters/react-localfirst/src/auth-backup.tsx](../starters/react-localfirst/src/auth-backup.tsx) para recovery phrase y passkey backup.
2. Hacer una prueba controlada: crear datos, limpiar storage, restaurar identidad y validar acceso.
3. Guia dedicada: [react-localfirst-recovery-backup.md](./react-localfirst-recovery-backup.md).
