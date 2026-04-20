# Guía Completa de Warden

Warden es un servidor compatible con Bitwarden diseñado para ejecutarse en **Cloudflare Workers**. Esta guía está dirigida tanto a **desarrolladores** que desean entender cómo funciona el código internamente, como a **usuarios y administradores** que buscan implementar y mantener su propia instancia.

---

## 1. Para Desarrolladores: Entendiendo el Proyecto

> 💡 **Nota para principiantes:** Esta primera sección explica cómo está programada la aplicación por dentro y es **muy técnica**. Si no sabes de programación o solo quieres instalar tu propio servidor para guardar tus contraseñas, **puedes saltarte toda esta sección e ir directamente al Punto 2 (Guía de Despliegue)**.

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

## 2. Guía de Despliegue (Instalación) para Usuarios

Aquí te explicaremos paso a paso cómo crear tu propio servidor de contraseñas. Puedes hacerlo de dos formas: usando botones en la web (Opción 1, recomendada para todos) o usando comandos en la pantalla negra de tu computadora (Opción 2, solo para expertos).

### Requisitos Previos (Para ambas opciones)

- Necesitarás crearte una cuenta gratuita en **[Cloudflare](https://dash.cloudflare.com)** (la empresa de servidores de internet que alojará tus contraseñas).
- Necesitarás una cuenta gratuita en **[GitHub](https://github.com)** (la página donde está guardado el código de este programa).

---

### Opción 1: Instalación Automática desde la Web (Recomendada)

Esta es la mejor opción. Funciona como si tuvieras "un robot" que hace todo el trabajo pesado de instalación y actualizaciones por ti cada vez que aprietas un botón, directamente desde internet.

Para que este "robot" (llamado GitHub Actions) pueda entrar a tu cuenta de Cloudflare a construirte el servidor, tienes que darle unas "llaves de acceso" y decirle cómo se llaman las cosas. Vamos a buscar esos datos paso a paso.

#### Paso A: Buscar tus datos en Cloudflare

Inicia sesión en tu panel de control de Cloudflare ([dash.cloudflare.com](https://dash.cloudflare.com/)). Vamos a buscar cuatro cosas:

**1. Tu ID de cuenta (`CLOUDFLARE_ACCOUNT_ID`)**
*   **¿Qué es?** Es tu número de cliente. Le dice al robot a qué cuenta debe ir.
*   **¿Dónde lo encuentro?**
    1. En el panel principal de Cloudflare, haz clic en "Workers & Pages" en el menú de la izquierda.
    2. Mira a la derecha de la pantalla. Busca un número largo con letras debajo de donde dice "Account ID" (ID de cuenta).
    3. Haz clic en "Click to copy" (clic para copiar) y guárdalo en un bloc de notas por ahora.

**2. Tu llave secreta del robot (`CLOUDFLARE_API_TOKEN`)**
*   **¿Qué es?** Es una contraseña larguísima que autoriza al robot a tocar ciertas cosas de tu cuenta.
*   **¿Dónde lo encuentro?**
    1. Haz clic en el icono de tu perfil arriba a la derecha y selecciona "My Profile" (Mi Perfil).
    2. En el menú de la izquierda, selecciona "API Tokens".
    3. Haz clic en el botón azul "Create Token" (Crear Token).
    4. Busca la plantilla que se llama "Edit Cloudflare Workers" y haz clic en "Use template".
    5. **Muy importante:** En la sección que dice "Permissions" (Permisos), verás uno que dice "Workers:Edit". Debes añadir otro permiso para que el robot pueda crear la base de datos: Haz clic en "+ Add more", y en las tres cajitas selecciona "Account", luego "D1", y por último "Edit".
    6. Baja hasta el final y haz clic en "Continue to summary" y luego en "Create Token".
    7. **Copia el texto gigante que aparece en pantalla.** Esta es tu llave secreta. Guárdala en tu bloc de notas. *Nota: Cloudflare no te la volverá a mostrar nunca más, así que no cierres la ventana o guárdala bien por ahora.*

**3. Tu base de datos y su ID (`D1_DATABASE_ID`)**
*   **¿Qué es?** La base de datos es la "caja fuerte virtual" vacía donde se guardarán todas tus contraseñas. D1 es simplemente el nombre que Cloudflare le da a sus cajas fuertes. Necesitamos crear una vacía y anotar su número identificador.
*   **¿Dónde lo encuentro?**
    1. Vuelve a la página principal de Cloudflare, menú de la izquierda, y ve a "Storage & databases" -> "D1".
    2. Haz clic en "Create database" (Crear base de datos).
    3. Escribe un nombre, por ejemplo: `warden-db` y pulsa Create.
    4. En la pantalla que aparece, busca la sección "Database ID" (ID de base de datos) y copia esos números y letras. Pégalos en tu bloc de notas.

**4. Opcional: Disco duro para archivos adjuntos (`R2_NAME`)**
*   **¿Qué es?** Si quieres poder guardar fotos o documentos PDF junto con tus contraseñas, necesitas "R2" (un disco duro en la nube). Si no haces esto, Warden limitará el tamaño de los archivos que puedes subir.
*   **¿Dónde lo encuentro?**
    1. En el panel izquierdo de Cloudflare, ve a "Storage & databases" -> "R2".
    2. (Si es tu primera vez, Cloudflare te pedirá que pongas una tarjeta de crédito para poder usar R2. Aunque es gratis para usuarios normales, lo piden para evitar abusos).
    3. Haz clic en "Create bucket" (Crear cubo o carpeta).
    4. Escribe un nombre inventado por ti (en minúsculas), por ejemplo: `warden-archivos` y créalo.
    5. Ese nombre exacto (`warden-archivos`) es el dato que necesitas guardar en tu bloc de notas.

#### Paso B: Entregarle las llaves al Robot (GitHub)

Ahora nos vamos a mudar de página: entra a [GitHub.com](https://github.com).

1.  Busca el botón que dice **"Fork"** en la parte superior derecha de esta página. Hacer un "Fork" es como decir "quiero una copia de todo este código guardada en mi cuenta para usarla yo mismo". Sigue las instrucciones para hacer la copia.
2.  Una vez hecha la copia, estarás en tu propia versión del código. Ve a la pestaña de **Settings** (Configuración, arriba a la derecha).
3.  En el menú de la izquierda, busca **"Secrets and variables"** y luego haz clic en **"Actions"**.
4.  Aquí vamos a esconder nuestras llaves. Haz clic en el botón verde **"New repository secret"** y agrega estos valores uno a uno (los que tienes en tu bloc de notas):
    *   Casilla `Name`: Escribe `CLOUDFLARE_ACCOUNT_ID` — Casilla `Secret`: Pega tu ID de cuenta. Pulsa "Add secret".
    *   Casilla `Name`: Escribe `CLOUDFLARE_API_TOKEN` — Casilla `Secret`: Pega tu llave larga. Pulsa "Add secret".
    *   Casilla `Name`: Escribe `D1_DATABASE_ID` — Casilla `Secret`: Pega el ID de tu base de datos. Pulsa "Add secret".
    *   *(Si hiciste el paso opcional de R2)* Casilla `Name`: Escribe `R2_NAME` — Casilla `Secret`: Pega el nombre que inventaste (ej. `warden-archivos`). Pulsa "Add secret".

#### Paso C: ¡Encender el Robot!

1. Ve a la pestaña **Actions** en la parte superior de tu página de GitHub.
2. Si te sale un mensaje de advertencia verde, haz clic en "I understand my workflows, go ahead and enable them" (Entiendo cómo funciona, adelante).
3. En la lista de la izquierda, haz clic en **"Deploy Prod"**.
4. A la derecha, verás un botón que dice **"Run workflow"**. Haz clic ahí, y luego en el botón verde que aparece.
5. El robot acaba de empezar a trabajar. Verás un circulito amarillo girando. En unos 2 o 3 minutos, si todo sale bien, se pondrá en verde. ¡Felicidades, el servidor ya está instalado en internet!

#### Paso D: Claves Maestras Finales (Muy Importante)

Tu servidor está vivo, pero ahora mismo está bloqueado por seguridad. Necesita saber quién eres tú y crear contraseñas maestras invisibles para que nadie más pueda entrar.

Debes volver una última vez a [Cloudflare](https://dash.cloudflare.com/):

1. Ve a "Workers & Pages" en el menú de la izquierda.
2. ¡Allí aparecerá tu servidor! Probablemente se llame `warden-worker`. Haz clic sobre él.
3. Ve a la pestaña **Settings** (Configuración) y en el menú de más a la izquierda busca **Variables and Secrets**.
4. En la sección "Secrets" (Secretos), vas a añadir tres configuraciones (haz clic en "Add" para cada una):

*   `ALLOWED_EMAILS` : Escribe el correo electrónico que vas a usar para registrarte (ej. `mi-correo@gmail.com`). Esto impide que extraños se registren en tu servidor.
*   `JWT_SECRET` : Inventa una contraseña larguísima y aporrea el teclado (ej. `a98db25kasjdhk234jhlkj`). Esto no te lo pedirá nunca el teléfono; sirve para que el servidor cree inicios de sesión seguros por debajo.
*   `JWT_REFRESH_SECRET` : Aporrea el teclado otra vez y escribe algo diferente a lo anterior.

**Si no guardas estas tres variables, tu aplicación de Bitwarden te dará error al intentar crear tu cuenta.**

---

### Opción 2: Instalación Manual (Solo para expertos en la Terminal)

Si eres desarrollador y prefieres hacerlo localmente con la herramienta de comandos `wrangler`:

1.  **Clonar el repositorio**:
    ```bash
    git clone https://github.com/tu-usuario/warden-worker.git
    cd warden-worker
    ```

2.  **Crear la base de datos D1**:
    ```bash
    wrangler d1 create warden-db
    ```

3.  **Configurar el ID de la base de datos**:
    Crea un archivo `.env` en la raíz del proyecto con el siguiente contenido:
    ```
    D1_DATABASE_ID="tu-database-id-aqui"
    ```

4.  **(Opcional) Crear un bucket R2 para archivos adjuntos**:
    ```bash
    wrangler r2 bucket create warden-attachments
    ```
    Luego, descomenta la configuración de R2 en el archivo `wrangler.toml`.

5.  **Descargar el Frontend (La página web)**:
    Ejecuta estos comandos en tu terminal para descargar la interfaz gráfica:
    ```bash
    export BW_WEB_VERSION="v2025.12.0"
    wget "https://github.com/dani-garcia/bw_web_builds/releases/download/${BW_WEB_VERSION}/bw_web_${BW_WEB_VERSION}.tar.gz"
    tar -xzf "bw_web_${BW_WEB_VERSION}.tar.gz" -C public/
    rm "bw_web_${BW_WEB_VERSION}.tar.gz"
    find public/web-vault -type f -name '*.map' -delete
    ```

6.  **Inicializar la base de datos y Desplegar**:
    ```bash
    # Crear las tablas en la nube
    wrangler d1 execute vault1 --file sql/schema.sql --remote

    # Aplicar actualizaciones de esquema si existen
    wrangler d1 migrations apply vault1 --remote

    # Subir código a Cloudflare
    wrangler deploy
    ```
    (No olvides configurar los Secrets finales `ALLOWED_EMAILS`, `JWT_SECRET` y `JWT_REFRESH_SECRET` mediante la terminal o el panel web).

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
