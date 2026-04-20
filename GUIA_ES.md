# Guía Completa de Warden

Warden es un servidor compatible con Bitwarden diseñado para ejecutarse en **Cloudflare Workers**. Esta guía está dirigida tanto a **desarrolladores** que desean entender cómo funciona el código internamente, como a **usuarios y administradores** que buscan implementar y mantener su propia instancia.

---

## 1. Para Desarrolladores: Entendiendo el Proyecto

Warden no utiliza código de Vaultwarden; es una implementación independiente y mínima en Rust (compilado a WebAssembly) y JavaScript, pensada para funcionar dentro de los límites computacionales de la capa gratuita de Cloudflare.

### 1.1 Arquitectura General

El sistema se compone de varios servicios de Cloudflare:
- **Cloudflare Workers**: El núcleo de la aplicación. Gestiona el enrutamiento HTTP y la lógica de negocio.
- **Cloudflare D1**: Una base de datos SQLite distribuida donde se almacenan todos los datos (usuarios, contraseñas, carpetas, etc.).
- **Cloudflare KV / R2**: Almacenamiento opcional para los archivos adjuntos. R2 tiene prioridad si está configurado.
- **Durable Objects (DO)**: Utilizado de forma opcional para descargar de trabajo al Worker principal. Las operaciones muy pesadas en CPU (como el cifrado PBKDF2 en registros/inicios de sesión o la importación masiva) se envían a un DO para evitar los límites estrictos de CPU por petición en la capa gratuita.
- **Static Assets**: El frontend (Web Vault de Vaultwarden) se sirve estáticamente directamente desde la red global de Cloudflare junto con el Worker.

### 1.2 Estructura del Proyecto

*   `src/`: Contiene el código fuente de la aplicación backend.
    *   `src/entry.js`: Punto de entrada principal en JavaScript. Intercepta las peticiones antes de enviarlas al código compilado en Rust (WASM). Aquí se maneja la subida/descarga de archivos por R2 (para evitar conversiones costosas de tipos) y el enrutamiento a Durable Objects (`HEAVY_DO`).
    *   `src/lib.rs`: El punto de entrada en Rust. Inicializa el router de Axum y el manejo de eventos (incluyendo tareas programadas/cron).
    *   `src/router.rs`: Define todas las rutas de la API y las enlaza a sus controladores.
    *   `src/handlers/`: Los controladores (handlers) agrupados por funcionalidad (`accounts`, `ciphers`, `folders`, `auth`, etc.).
    *   `src/models/`: Las estructuras de datos (Structs) para la serialización y deserialización de JSON, tanto para peticiones (Requests) como para respuestas (Responses).
    *   `src/durable/`: Implementación del Durable Object en Rust que reutiliza el mismo router de Axum para peticiones costosas.
*   `migrations/`: Archivos SQL de migración aplicados mediante Wrangler para actualizar el esquema de la base de datos D1.
*   `sql/schema.sql`: Esquema base de la base de datos.
*   `public/`: Archivos estáticos de la interfaz web (se descargan en tiempo de despliegue).
*   `docs/`: Documentación adicional técnica original en inglés.

### 1.3 Walkthrough: El Ciclo de Vida de una Petición

¿Qué pasa cuando un cliente (como la extensión del navegador de Bitwarden) hace una petición a la API?

1.  **Recepción en `src/entry.js`**: La petición llega primero al archivo JavaScript.
2.  **Rutas Rápidas (Fast-paths)**:
    *   Si la petición es una subida/descarga de un archivo adjunto, el código en JS la maneja directamente para procesar el flujo de datos (stream) directamente a/desde Cloudflare R2 sin pasar por WebAssembly, lo que ahorra mucha memoria y CPU.
3.  **Filtrado por CPU (Heavy DO)**:
    *   Si la petición requiere mucho uso de CPU (ej. `/identity/connect/token` con usuario/contraseña, importación, registro), y el Durable Object (`HEAVY_DO`) está configurado, `entry.js` redirige la petición a este DO. El DO ejecuta el mismo código en Rust, pero cuenta con un límite de CPU mucho mayor (30 segundos frente a pocos milisegundos en el Worker normal).
4.  **Ejecución en Rust (WASM)**:
    *   Si no es interceptada por lo anterior, la petición pasa a la instancia principal de Rust (WASM) en `src/lib.rs`.
    *   El enrutador (`axum` en `src/router.rs`) analiza la URL y el método HTTP, y dirige la solicitud a la función correspondiente en `src/handlers/`.
5.  **Interacción con la Base de Datos**:
    *   El controlador (handler) valida los datos y realiza consultas SQL a Cloudflare D1 usando la librería `worker::d1`.
    *   Se procesan los resultados, se empaquetan en estructuras de JSON (definidas en `src/models/`) y se devuelven al cliente como respuesta HTTP.

---

## 2. Guía de Despliegue para Usuarios

Puedes desplegar Warden de dos maneras: manualmente a través de la interfaz de comandos (CLI) o de forma automatizada mediante GitHub Actions (recomendado para producción).

### 2.1 Requisitos Previos

- Una cuenta de [Cloudflare](https://dash.cloudflare.com).
- Node.js y la herramienta `wrangler` instalada si vas a usar CLI.

### 2.2 Opción 1: Despliegue Manual (CLI)

1.  **Clonar el repositorio**:
    ```bash
    git clone https://github.com/tu-usuario/warden-worker.git
    cd warden-worker
    ```

2.  **Crear la base de datos D1**:
    ```bash
    wrangler d1 create warden-db
    ```
    *Wrangler te devolverá un `database_id`. Guárdalo.*

3.  **Configurar el ID de la base de datos**:
    Crea un archivo `.env` en la raíz del proyecto (¡no lo subas a git!) con el siguiente contenido:
    ```
    D1_DATABASE_ID="tu-database-id-aqui"
    ```

4.  **(Opcional) Crear un bucket R2 para archivos adjuntos**:
    ```bash
    wrangler r2 bucket create warden-attachments
    ```
    Luego, descomenta la configuración de R2 en el archivo `wrangler.toml`. Si no haces esto, los adjuntos se guardarán en KV (con límite de 25MB).

5.  **Descargar el Frontend (Web Vault)**:
    Ejecuta estos comandos en tu terminal para descargar la interfaz web de Vaultwarden:
    ```bash
    export BW_WEB_VERSION="v2025.12.0"
    wget "https://github.com/dani-garcia/bw_web_builds/releases/download/${BW_WEB_VERSION}/bw_web_${BW_WEB_VERSION}.tar.gz"
    tar -xzf "bw_web_${BW_WEB_VERSION}.tar.gz" -C public/
    rm "bw_web_${BW_WEB_VERSION}.tar.gz"
    find public/web-vault -type f -name '*.map' -delete
    ```

6.  **Inicializar la base de datos y Desplegar**:
    ```bash
    # Crear las tablas iniciales
    wrangler d1 execute vault1 --file sql/schema.sql --remote

    # Aplicar posibles migraciones
    wrangler d1 migrations apply vault1 --remote

    # Desplegar a Cloudflare
    wrangler deploy
    ```

### 2.3 Opción 2: Despliegue Automático con CI/CD (GitHub Actions)

Esta es la mejor opción para asegurar que tu servidor esté siempre actualizado de forma sencilla y segura en la nube, sin necesidad de usar herramientas locales.

Para que GitHub Actions pueda crear y actualizar los recursos en tu cuenta de Cloudflare, necesita autorización e identificadores.

#### Paso a paso para conseguir los datos en Cloudflare

Primero, inicia sesión en tu panel de control de Cloudflare ([dash.cloudflare.com](https://dash.cloudflare.com/)).

**1. Obtener el `CLOUDFLARE_ACCOUNT_ID`**
*   **¿Qué es y para qué sirve?** Es el identificador único de tu cuenta. Permite a GitHub saber exactamente en qué cuenta de Cloudflare debe crear la aplicación.
*   **¿Dónde lo encuentro?**
    1. En el panel principal de Cloudflare, haz clic en "Workers & Pages" en el menú de la izquierda.
    2. Mira la barra lateral derecha. Busca la sección que dice "Account ID" o "ID de cuenta".
    3. Haz clic en "Click to copy" para copiar la cadena de letras y números.

**2. Obtener el `CLOUDFLARE_API_TOKEN`**
*   **¿Qué es y para qué sirve?** Es una contraseña (token) especial que le da permiso a GitHub para hacer cambios en tu nombre, pero limitados solo a lo necesario (crear base de datos y actualizar el código del Worker).
*   **¿Dónde lo encuentro?**
    1. Haz clic en el icono de tu perfil arriba a la derecha y selecciona "My Profile" (Mi Perfil).
    2. En el menú izquierdo, selecciona "API Tokens".
    3. Haz clic en el botón azul "Create Token" (Crear Token).
    4. Busca la plantilla "Edit Cloudflare Workers" y haz clic en "Use template" (Usar plantilla).
    5. **Muy importante:** En la sección "Permissions", además del que ya viene por defecto ("Workers:Edit"), debes agregar otro haciendo clic en "+ Add more". Elige "Account" (Cuenta) -> "D1" -> "Edit" (Editar).
    6. (Opcional) Si quieres usar R2 para archivos adjuntos, agrega un tercer permiso: "Account" -> "Workers R2 Storage" -> "Edit".
    7. En "Account Resources" y "Zone Resources" selecciona tu cuenta y dominio respectivo o "All accounts" si prefieres.
    8. Haz clic en "Continue to summary" y luego en "Create Token".
    9. **Copia el token que se muestra en pantalla.** Esta es la única vez que podrás verlo completo.

**3. Crear la base de datos y obtener el `D1_DATABASE_ID`**
*   **¿Qué es y para qué sirve?** D1 es la base de datos SQL de Cloudflare. Necesitamos crear la base de datos vacía primero y obtener su ID para decirle a nuestra app "guarda todos los usuarios y contraseñas aquí".
*   **¿Dónde lo encuentro?**
    1. Vuelve al panel principal, menú de la izquierda, y ve a "Storage & databases" (Almacenamiento y bases de datos) -> "D1".
    2. Haz clic en "Create database" (Crear base de datos).
    3. Ponle un nombre, por ejemplo: `warden-db` (o "vault1") y pulsa Create.
    4. En la pantalla de la base de datos, entra a ella. Busca la sección "Database ID" (ID de base de datos) y copia ese valor.

**4. (Opcional) Crear bucket de R2 y configurar `R2_NAME`**
*   **¿Qué es y para qué sirve?** Si subes un archivo (ej. una foto o un PDF) a tu gestor de contraseñas, R2 es el disco duro en la nube que lo guardará. Es más rápido y permite archivos más grandes que KV.
*   **¿Dónde lo encuentro?**
    1. En el panel izquierdo, ve a "Storage & databases" -> "R2".
    2. Si es tu primera vez, Cloudflare te pedirá activar R2 (requiere registrar un método de pago, aunque la capa gratuita es muy amplia).
    3. Haz clic en "Create bucket" (Crear bucket).
    4. Escribe un nombre único, por ejemplo: `warden-attachments` y créalo.
    5. El nombre que escribiste (`warden-attachments`) es tu `R2_NAME`.

#### Cómo configurar estos secretos en GitHub y desplegar

Ahora que tienes los datos, vamos a automatizar el despliegue:

1.  Haz un "Fork" de este repositorio a tu cuenta personal de GitHub (arriba a la derecha, botón "Fork").
2.  Ve a tu nuevo repositorio en GitHub, y navega a **Settings > Secrets and variables > Actions**.
3.  Haz clic en el botón verde **"New repository secret"** para agregar cada uno de estos datos, uno por uno:
    *   `Name:` `CLOUDFLARE_ACCOUNT_ID` — `Secret:` Pega tu ID de cuenta.
    *   `Name:` `CLOUDFLARE_API_TOKEN` — `Secret:` Pega tu token.
    *   `Name:` `D1_DATABASE_ID` — `Secret:` Pega el ID de tu base de datos D1.
    *   *(Opcional)* `Name:` `R2_NAME` — `Secret:` Pega el nombre exacto de tu bucket (ej. `warden-attachments`).
4.  Una vez guardados los secretos, ve a la pestaña **Actions** en tu repositorio de GitHub.
5.  Si te pide permiso para activar las acciones, haz clic en "I understand my workflows, go ahead and enable them".
6.  En la barra lateral izquierda, selecciona "Deploy Prod".
7.  Haz clic a la derecha en "Run workflow", y vuelve a presionar el botón verde "Run workflow".
8.  Espera unos minutos hasta que termine con éxito. ¡El servidor ya estará desplegado en Cloudflare!

### 2.4 Variables de Entorno Fundamentales (Después de Desplegar)

Una vez que GitHub Actions finalizó el despliegue con éxito, el servidor existe en la nube, pero todavía necesita claves de seguridad maestras para funcionar y generar las sesiones de los usuarios.

Debes entrar nuevamente a [Cloudflare](https://dash.cloudflare.com/):

1. Ve a "Workers & Pages".
2. Haz clic sobre tu Worker recién desplegado (probablemente se llame `warden-worker`).
3. Ve a la pestaña **Settings** (Configuración) y en el menú lateral de configuración selecciona **Variables and Secrets**.
4. En la sección "Secrets", haz clic en "Add" para agregar las siguientes variables:

*   `ALLOWED_EMAILS`: ¿Quién puede registrarse? (ej. escribe `tu@email.com` si solo la usas tú, o `*@tudominio.com` si tienes un dominio propio para tu familia/empresa). Protege el servidor de registros públicos indeseados.
*   `JWT_SECRET`: Una cadena de texto aleatoria muy larga e impredecible (ej. `a98db25...`). Sirve para cifrar y firmar los tokens de sesión; si alguien no tiene esto, no puede generar accesos válidos.
*   `JWT_REFRESH_SECRET`: Otra cadena de texto aleatoria y larga distinta a la anterior. Sirve para renovar las sesiones de forma segura sin tener que volver a ingresar la contraseña maestra.

> **Importante:** Si no configuras estas tres variables y guardas los cambios en Cloudflare, el servidor no iniciará y la app móvil/web dará un error.

---

## 3. Mantenimiento y Backups

Warden está diseñado para requerir un mantenimiento mínimo, pero es vital proteger tus datos.

### 3.1 Copias de Seguridad (Backups Automáticos)

Si usaste GitHub Actions, puedes configurar backups automáticos a un servicio de almacenamiento compatible con S3 o WebDAV (ej. Nextcloud). Los backups se ejecutan todos los días a las 04:00 UTC.

**No uses R2 de la misma cuenta de Cloudflare para backups.** Si Cloudflare suspende tu cuenta, perderás tanto la base de datos principal como los backups.

Añade los siguientes Secrets en GitHub para activar los backups hacia S3 (por ejemplo, AWS S3 o Backblaze B2):
*   `S3_ACCESS_KEY_ID`: Tu ID de clave de acceso.
*   `S3_SECRET_ACCESS_KEY`: Tu clave secreta.
*   `S3_BUCKET`: Nombre del bucket de almacenamiento.
*   `S3_REGION`: Región (ej. `us-east-1` o `auto`).
*   `S3_ENDPOINT`: URL del servidor (necesario si no es AWS S3).
*   `BACKUP_ENCRYPTION_KEY`: (Recomendado) Una contraseña fuerte para encriptar los datos de respaldo.

### 3.2 Restauración de una Copia de Seguridad

Si alguna vez pierdes tus datos:
1. Descarga el backup desde tu almacenamiento.
2. Si está encriptado, desencríptalo en tu terminal:
   ```bash
   openssl enc -aes-256-cbc -d -pbkdf2 -iter 100000 \
     -in vault1_prod_YYYY-MM-DD.sql.gz.enc \
     -out backup.sql.gz \
     -pass pass:"TU_CLAVE_DE_ENCRIPTACION"
   ```
3. Descomprímelo: `gunzip backup.sql.gz`
4. Aplícalo a D1 (cuidado, esto sobrescribe datos existentes):
   ```bash
   # Agrega PRAGMA foreign_keys=OFF; al principio del archivo si da errores de dependencia
   echo -e "PRAGMA foreign_keys=OFF;\n$(cat backup.sql)" > backup.sql

   wrangler d1 execute tu-nombre-de-base-de-datos --remote --file=backup.sql
   ```

### 3.3 Viaje en el Tiempo (D1 Time Travel)

Cloudflare D1 permite restaurar el estado de tu base de datos a cualquier punto en los últimos 30 días sin necesitar un archivo de backup externo. Útil para deshacer borrados accidentales masivos.

Para ver información o restaurar:
```bash
wrangler d1 time-travel info tu-nombre-de-base-de-datos
wrangler d1 time-travel restore tu-nombre-de-base-de-datos --timestamp=2024-01-15T12:00:00Z
```

### 3.4 Limpieza Automática (Cron)

El proyecto incluye un cron job (tarea programada) en `wrangler.toml` que se ejecuta diariamente a las 03:00 UTC. Esta tarea limpia la papelera (archivos "soft-deleted" que llevan más de 30 días, configurable con la variable `TRASH_AUTO_DELETE_DAYS`) y borra archivos adjuntos temporales huérfanos. No necesitas intervenir en este proceso.
