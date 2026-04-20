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

Esta es la mejor opción para asegurar que tu servidor esté siempre actualizado fácilmente.

1.  Haz un "Fork" de este repositorio a tu cuenta de GitHub.
2.  En tu repositorio en GitHub, ve a **Settings > Secrets and variables > Actions**.
3.  Añade los siguientes **Secrets** obligatorios:
    *   `CLOUDFLARE_ACCOUNT_ID`: Tu ID de cuenta de Cloudflare (se ve en el panel lateral derecho de Cloudflare).
    *   `CLOUDFLARE_API_TOKEN`: Un token con permisos para editar Workers, D1 y KV/R2. (Se crea en tu perfil de Cloudflare).
    *   `D1_DATABASE_ID`: El ID de la base de datos que creaste en Cloudflare.
4.  Si usas R2 para archivos adjuntos, crea el bucket en Cloudflare y añade un secret llamado `R2_NAME` con el nombre del bucket.
5.  Ve a la pestaña **Actions** en tu repositorio, selecciona "Deploy Prod" y pulsa **Run workflow**.

### 2.4 Variables de Entorno Fundamentales

Una vez desplegado el worker, **debes configurar estas variables de entorno en el panel de Cloudflare** (Workers & Pages > Tu Worker > Settings > Variables and Secrets) como secretos:

*   `ALLOWED_EMAILS`: Quién puede registrarse (ej. `tu@email.com` o `*@tudominio.com`).
*   `JWT_SECRET`: Una cadena de texto aleatoria y larga para firmar sesiones.
*   `JWT_REFRESH_SECRET`: Otra cadena de texto aleatoria y larga.

> **Importante:** Si no configuras estas tres variables, el servidor fallará.

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
