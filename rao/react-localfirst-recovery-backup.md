# React localfirst: Recovery Phrase y Passkey Backup

Este documento explica como recuperar una identidad localfirst en Jazz cuando el usuario pierde storage del navegador o cambia de dispositivo.

## 1) Problema que resuelven

En Camino 1 (React localfirst), la identidad principal vive en el dispositivo.

Si se pierde esa identidad local:

1. El usuario ya no puede probar que es el mismo principal.
2. Sus datos pueden seguir en sync backend, pero quedan inaccesibles para esa nueva identidad.

Recovery Phrase y Passkey Backup sirven para restaurar esa identidad original.

## 2) Recovery Phrase: que es

1. Es una frase de 24 palabras.
2. Representa el secret local (32 bytes) de la identidad.
3. Permite reconstruir exactamente el mismo secret y recuperar acceso.

Implementacion base:

1. API exportada en [packages/jazz-tools/src/passphrase.ts](../packages/jazz-tools/src/passphrase.ts).
2. Logica en [packages/jazz-tools/src/runtime/recovery-phrase.ts](../packages/jazz-tools/src/runtime/recovery-phrase.ts).

Detalles tecnicos importantes:

1. Usa wordlist english de BIP39.
2. Valida longitud exacta de 24 palabras.
3. Valida checksum de la frase.
4. Convierte entre secret base64url y mnemonic.

## 3) Passkey Backup: que es

1. Usa WebAuthn/passkeys del dispositivo para guardar y recuperar el secret.
2. Requiere verificacion de usuario (biometria/PIN) al restaurar.
3. Puede dar mejor UX que frase manual para usuarios no tecnicos.

Implementacion base:

1. API exportada en [packages/jazz-tools/src/passkey-backup.ts](../packages/jazz-tools/src/passkey-backup.ts).
2. Logica en [packages/jazz-tools/src/runtime/passkey-backup.ts](../packages/jazz-tools/src/runtime/passkey-backup.ts).

Detalles tecnicos importantes:

1. Exige userVerification required.
2. Exige residentKey required.
3. Usa userHandle de 32 bytes para recuperar el secret.
4. Si el navegador no soporta WebAuthn, falla con error not-supported.

## 4) Donde se usa en el starter React localfirst

UI de backup/restore:

1. [starters/react-localfirst/src/auth-backup.tsx](../starters/react-localfirst/src/auth-backup.tsx).

Integracion en la app:

1. [starters/react-localfirst/src/App.tsx](../starters/react-localfirst/src/App.tsx).

Ese componente hace:

1. Mostrar recovery phrase desde el secret actual.
2. Restaurar identidad pegando frase.
3. Crear backup con passkey.
4. Restaurar identidad con passkey.

## 5) Flujo de recuperacion (paso a paso)

### 5.1 Con Recovery Phrase

1. Usuario guarda su frase de 24 palabras.
2. Si pierde storage, abre app en dispositivo nuevo.
3. Pega la frase en Restore from recovery phrase.
4. RecoveryPhrase.toSecret reconstruye secret.
5. auth.login(secret) restaura identidad local.
6. La app vuelve a leer/escribir datos del principal original.

### 5.2 Con Passkey

1. Usuario crea passkey backup desde su cuenta localfirst.
2. En nuevo dispositivo o storage limpio, usa Restore with passkey.
3. BrowserPasskeyBackup.restore obtiene secret.
4. auth.login(secret) restaura identidad.

## 6) Diferencias practicas

Recovery Phrase:

1. Pro: no depende de proveedor de passkeys.
2. Pro: portable offline si guardaste las palabras.
3. Contra: UX manual, propensa a errores de escritura.

Passkey:

1. Pro: UX simple para usuario final.
2. Pro: verificacion fuerte del usuario.
3. Contra: depende de soporte WebAuthn y configuracion de rpId/appHostname.

Recomendacion practica: ofrecer ambos.

## 7) Nota critica sobre appHostname (passkeys)

En [starters/react-localfirst/src/auth-backup.tsx](../starters/react-localfirst/src/auth-backup.tsx), PASSKEY_APP_HOSTNAME esta sin fijar por defecto.

Para produccion:

1. Define un hostname estable del producto.
2. Mantenerlo consistente evita problemas de restauracion entre entornos.

## 8) Errores comunes y significado

Recovery Phrase:

1. invalid-length: no son 24 palabras.
2. invalid-word: alguna palabra no existe en la wordlist.
3. invalid-checksum: frase mal escrita o alterada.
4. invalid-secret: secret base no valido.

Passkey:

1. not-supported: navegador/dispositivo sin WebAuthn suficiente.
2. create-failed: no se pudo crear credencial.
3. get-failed: no se pudo recuperar credencial.
4. verification-failed: no hubo verificacion de usuario.
5. invalid-credential: credencial no contiene secret esperado.

## 9) Checklist de producto para Camino 1

1. Mostrar backup onboarding en primeras sesiones.
2. Recordatorio periodico de guardar recovery phrase.
3. Boton visible de Restore en pantalla de entrada.
4. Telemetria de errores de restore para soporte.
5. Documentar claramente que perder storage sin backup implica perder acceso.

## 10) Cuando migrar a hybrid

Si tu producto necesita recuperacion de cuenta menos dependiente del usuario final, evalua pasar a camino hybrid con cuenta Better Auth.

## 11) Playbook recomendado de implementacion

Secuencia sugerida para producto real:

1. Primer login localfirst exitoso:
- Mostrar modal ligero: protege tu cuenta local.
- CTA primario: crear backup con passkey.
- CTA secundario: ver recovery phrase.

2. Primera escritura relevante del usuario:
- Si no tiene backup confirmado, mostrar recordatorio no bloqueante.

3. Sesiones siguientes:
- Mostrar recordatorio cada cierto numero de dias hasta completar backup.

4. Pantalla de entrada:
- Siempre ofrecer opcion restore visible.

## 12) Copy UX sugerido

Texto de onboarding (breve):

1. Titulo: Protege tu cuenta local.
2. Mensaje: Si cambias de dispositivo o limpias el navegador, necesitaras backup para recuperar tus datos.
3. CTA principal: Crear backup con passkey.
4. CTA secundaria: Ver frase de recuperacion.

Texto de restore:

1. Titulo: Recuperar cuenta.
2. Opcion 1: Restaurar con passkey.
3. Opcion 2: Restaurar con frase de 24 palabras.

## 13) Estrategia de fallback recomendada

1. Intentar passkey primero cuando el navegador lo soporte.
2. Si passkey falla o no soporta WebAuthn, ofrecer recovery phrase de inmediato.
3. Mantener siempre disponibles ambas rutas en configuracion de cuenta.
4. En equipos compartidos, priorizar phrase manual si politica del producto lo exige.

## 14) Eventos de telemetria utiles

Eventos minimos a instrumentar:

1. backup_prompt_shown
2. passkey_backup_success
3. passkey_backup_error
4. phrase_reveal_success
5. phrase_restore_success
6. phrase_restore_error
7. passkey_restore_success
8. passkey_restore_error

Campos recomendados:

1. browser
2. platform
3. error_code
4. flow_step

## 15) Flujo de soporte cuando el usuario pierde acceso

1. Preguntar si tiene passkey o recovery phrase.
2. Si tiene passkey:
- Guiar restore en dispositivo nuevo.

3. Si tiene phrase:
- Guiar restore por texto y validar 24 palabras.

4. Si no tiene ninguno:
- Explicar limite de localfirst: sin secreto original no se puede probar identidad previa.
- Ofrecer crear cuenta nueva local o migrar a flujo con cuenta (hybrid) para evitar recurrencia.

## 16) Criterio de readiness para salir a produccion

1. Backup prompt implementado y medido.
2. Restore disponible desde entrypoint visible.
3. appHostname de passkey fijado y estable.
4. Mensajes de error mapeados por codigo.
5. Soporte documentado para casos sin backup.
