# Camino 1 detallado: React localfirst

Este documento detalla solo el camino React localfirst para entender identidad, validacion en backend, permisos y flujo real de escritura.

## 1) Cuando elegir este camino

1. Quieres aprender Jazz rapido sin montar auth server desde el inicio.
2. Estas validando producto y UX primero.
3. Prefieres onboarding sin friccion para usuarios nuevos.

## 2) Mapa de componentes que participan

1. Frontend React con provider localfirst en [starters/react-localfirst/src/main.tsx](../starters/react-localfirst/src/main.tsx).
2. Plugin de dev que levanta Jazz local en [starters/react-localfirst/vite.config.ts](../starters/react-localfirst/vite.config.ts).
3. Schema de datos en [starters/react-localfirst/schema.ts](../starters/react-localfirst/schema.ts).
4. Permisos por fila en [starters/react-localfirst/permissions.ts](../starters/react-localfirst/permissions.ts).
5. CRUD de ejemplo en [starters/react-localfirst/src/todo-widget.tsx](../starters/react-localfirst/src/todo-widget.tsx).
6. Recuperacion/backup de identidad en [starters/react-localfirst/src/auth-backup.tsx](../starters/react-localfirst/src/auth-backup.tsx).

Detalle del servicio de sincronizacion:

1. [react-localfirst-sync-server.md](./react-localfirst-sync-server.md).

Detalle de recuperacion de identidad:

1. [react-localfirst-recovery-backup.md](./react-localfirst-recovery-backup.md).

## 3) Flujo de identidad (que valida el backend)

1. En el cliente, useLocalFirstAuth obtiene o genera un secret local.
2. Ese secret se pasa al JazzProvider junto con appId y serverUrl.
3. El backend Jazz valida pruebas criptograficas derivadas de ese secret y construye una sesion confiable.
4. La identidad efectiva de permisos queda en session.user_id.
5. Los permisos permiten leer/editar/borrar solo filas creadas por ese user_id usando el predicado $createdBy.

Resumen importante:

1. La clave privada permanece en el dispositivo.
2. El backend no confia en un user_id arbitrario enviado por cliente.
3. El backend confia en prueba criptografica + reglas de permisos.

## 4) Flujo de una escritura de extremo a extremo

1. El usuario crea un todo en la UI.
2. React ejecuta db.insert sobre app.todos.
3. Jazz runtime firma/adjunta identidad local.
4. El servidor valida identidad de sesion.
5. Se aplica permisos.ts para allowInsert/allowRead/allowUpdate/allowDelete.
6. La fila se persiste y queda marcada con su creador interno.
7. Consultas posteriores se filtran por $createdBy == session.user_id.

## 5) Donde vive cada cosa

1. Identidad privada: storage local del navegador/dispositivo.
2. Datos: almacenamiento del backend de sync (local o cloud).
3. Reglas de acceso: archivo de permisos de tu app.

Implicacion:

1. Si se pierde el storage local, puedes perder acceso a la identidad.
2. Los datos pueden seguir en backend, pero sin esa identidad quedan inaccesibles.
3. Para mitigarlo, usar recovery phrase o passkey backup (AuthBackup).

## 6) Paso a paso para arrancar hoy

1. Crear app: npm create jazz@latest mi-app -- --starter react-localfirst --hosting selfhosted
2. Entrar: cd mi-app
3. Instalar: pnpm install
4. Correr: pnpm dev
5. Abrir: http://localhost:5173
6. Cambiar schema: editar schema.ts
7. Ajustar permisos: editar permissions.ts
8. Conectar UI: editar src/todo-widget.tsx

## 7) Variables de entorno en este camino

1. En selfhosted dev, puedes iniciar sin .env manual.
2. El plugin jazzPlugin completa VITE_JAZZ_APP_ID y VITE_JAZZ_SERVER_URL en primer arranque.
3. En cloud, debes definir VITE_JAZZ_APP_ID, VITE_JAZZ_SERVER_URL, JAZZ_ADMIN_SECRET y BACKEND_SECRET.

Referencia de entorno y despliegue: [starters/react-localfirst/README.md](../starters/react-localfirst/README.md).

## 8) Produccion selfhosted (punto critico)

1. El servidor requiere habilitar local-first auth explicitamente.
2. Si no habilitas allow-local-first-auth, las conexiones anonimas fallaran.

Referencia: [starters/react-localfirst/README.md](../starters/react-localfirst/README.md).

## 9) Checklist de troubleshooting

1. Error de VITE_JAZZ_APP_ID o VITE_JAZZ_SERVER_URL: revisar plugin y .env.
2. No puedo escribir datos: revisar permisos.ts y session.user_id.
3. En prod da auth error: verificar allow-local-first-auth.
4. Usuario perdio cuenta al limpiar navegador: usar restore via recovery phrase o passkey.

## 10) Cuando pasar al camino 2 (hybrid)

1. Cuando ya necesitas cuenta portable entre dispositivos.
2. Cuando necesitas recuperacion robusta sin depender solo de storage local.
3. Cuando quieres onboarding localfirst pero upgrade opcional a cuenta.
