# Respaldo en Google Drive — Biblioteca

Biblioteca puede respaldar tu colección en tu propia cuenta de Google Drive. **Es un respaldo, no una sincronización en vivo**: `localStorage` en este navegador sigue siendo la única fuente de verdad de lo que ves en pantalla. Cada guardado local también empuja el estado actual a **un solo archivo JSON** (`biblioteca_backup.json`) que la app crea en tu Drive. Restaurar desde ese archivo es siempre una acción explícita y confirmada — un respaldo desactualizado nunca sobrescribe silenciosamente datos más nuevos en el dispositivo.

## Configuración (una sola vez)

1. Ve a [console.cloud.google.com](https://console.cloud.google.com/) y crea (o selecciona) un proyecto.
2. En **APIs & Services → Library**, busca y habilita la **Google Drive API**.
3. En **APIs & Services → OAuth consent screen**, configúrala (el tipo "External" está bien para uso personal; agrega tu propia cuenta de Google bajo "Test users" mientras la app no esté verificada).
4. En **APIs & Services → Credentials → Create Credentials → OAuth Client ID**:
   - Tipo de aplicación: **Web application**.
   - En "Authorized JavaScript origins", agrega el origen exacto donde sirves la app, por ejemplo `https://tuapp.example.com` y `http://localhost:PUERTO` para desarrollo local.
   - No se necesita redirect URI — todo ocurre del lado del cliente.
5. Copia el Client ID resultante (termina en `.apps.googleusercontent.com`) y pégalo en `biblioteca_enhanced.html`, reemplazando el valor de `CLIENT_ID` dentro de `DRIVE_CONFIG`:

   ```js
   const DRIVE_CONFIG = {
     CLIENT_ID: 'TU_CLIENT_ID_AQUI.apps.googleusercontent.com',
     ...
   };
   ```

Hasta que reemplaces ese valor, la app muestra un mensaje claro de "Google Drive aún no está configurado" en vez de fallar de forma confusa.

## Cómo funciona

- **Alcance de permisos**: solo `drive.file` — la app únicamente puede ver/modificar los archivos que ella misma crea, nunca el resto de tu Drive. Es también el alcance más estrecho que Google permite sin revisión de seguridad para apps no verificadas.
- **Conectar**: pide un token (ventana emergente de Google) y revisa si ya existe un respaldo. Si existe, te pregunta explícitamente si quieres restaurarlo (reemplaza los datos locales) o mantener los datos locales y sobrescribir el respaldo. Si no existe, simplemente crea el primer respaldo con tus libros actuales.
- **Respaldo automático**: cada vez que guardas localmente (agregar/editar/borrar un libro, avanzar en la lectura, etc.) se programa un respaldo con 2.5 segundos de espera (debounce), para no saturar la API con cada tecla. Un respaldo lento o fallido nunca bloquea ni revierte el guardado local.
- **Respaldar ahora**: botón manual que cancela cualquier espera pendiente y sube los datos de inmediato.
- **Restaurar desde Drive**: siempre detrás de una confirmación explícita ("esto reemplaza todos los libros de este dispositivo"). Nunca es automático.
- **Reconexión silenciosa**: al abrir la app, si estuviste conectado antes, intenta reconectar en silencio. Si falla (revocaste el acceso, borraste cookies, etc.), aparece un banner para reconectar manualmente — nunca falla de forma invisible.
- **Desconectar**: revoca el token, borra todo el estado en memoria y la bandera de "intentar reconectar", y tus libros se quedan intactos en el dispositivo.
- El token de acceso **nunca se guarda** en `localStorage` ni en cookies — solo se guarda una bandera booleana (`biblioteca_drive_connected`) que significa "intenta reconectar en silencio la próxima vez". El token siempre se vuelve a pedir a Google.

## Si cambias el modelo de datos de un libro

Si agregas un campo nuevo a los objetos de `books` (por ejemplo, una nueva propiedad como `rating` o `tags`), no necesitas tocar el módulo de Drive: el respaldo usa el arreglo `books` completo tal cual está en memoria (`{ version, exportedAt, books }`), igual que `exportLibrary()`. Solo revisa `applyRestoredBooks()` si ese campo nuevo necesita una migración especial al restaurar respaldos antiguos que no lo tenían.
