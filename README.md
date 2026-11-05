# Proyecto integrador: AUTH + STEGO

Backend de Desarrollo de Software Seguro del equipo **Lugmaña, Villacres y Morillo**. Este repositorio contiene dos microservicios independientes: **AUTH**, para identidades y permisos, y **STEGO**, para analizar posibles indicios de esteganografía en imágenes PNG. La interacción se realiza mediante APIs REST; no incluye interfaz web.

## Contenido

- [Requisitos previos](#requisitos-previos)
- [Instalación desde cero](#instalación-desde-cero)
- [Primer uso con Postman](#primer-uso-con-postman)
- [Funcionamiento y comunicación](#funcionamiento-y-comunicación)
- [APIs y permisos](#apis-y-permisos)
- [Método esteganográfico y muestras](#método-esteganográfico-y-muestras)
- [Operación y persistencia](#operación-y-persistencia)
- [Pruebas reproducibles](#pruebas-reproducibles)
- [Solución de problemas](#solución-de-problemas)
- [Documentación técnica](#documentación-técnica)

## Requisitos previos

| Herramienta | Uso |
|---|---|
| Git | Clonar este repositorio. En Windows, instalar Git for Windows, que normalmente incluye OpenSSL. |
| Node.js 24 y npm | Ejecutar los inicializadores de configuración y las herramientas de AUTH. Las imágenes Docker usan Node.js 24. |
| Docker Desktop con contenedores Linux, o Docker Engine en Linux | Ejecutar AUTH, STEGO y PostgreSQL. El motor debe estar iniciado. |
| Docker Compose 2.24.4 o posterior, también compatible con Compose v5 | Iniciar los servicios; las pruebas integradas requieren soporte de `!override`. |
| OpenSSL | Generar la CA y los certificados TLS de desarrollo. |
| Postman Desktop | Seguir el flujo de uso de esta guía; alternativamente se puede utilizar otro cliente REST. |
| Python 3.12, opcional en el equipo | Ejecutar pruebas Python y regenerar muestras. No se necesita para iniciar los contenedores. |
| Postman CLI, opcional | Repetir automáticamente las pruebas de la colección. |

La instalación inicial necesita Internet para descargar dependencias e imágenes. Los puertos locales predeterminados son **3001**, **3443** y **8000**; deben estar libres.

Comprobar las herramientas en una terminal:

```sh
git --version
node --version
npm --version
docker version
docker compose version
```

En Windows, el inicializador busca OpenSSL también en `C:/Program Files/Git/usr/bin/openssl.exe`. Si está instalado en otro lugar, establecer `OPENSSL_PATH` antes de inicializar. Ejemplo en PowerShell, ajustando la ruta:

```powershell
$env:OPENSSL_PATH = "C:\ruta\a\openssl.exe"
```

## Instalación desde cero

Los pasos siguientes funcionan en PowerShell y en una terminal de Linux/macOS, salvo las variantes indicadas. Ejecutarlos en orden. No es necesario instalar PostgreSQL ni ejecutar NestJS o Uvicorn manualmente en el equipo.

### 1. Clonar e instalar las herramientas locales

```sh
git clone https://github.com/arieleMM-dev/ProyectuU1-V1_SOFTWARE_SEGURO_LUGMANA-VILLACRES-MORILLO.git
cd ProyectuU1-V1_SOFTWARE_SEGURO_LUGMANA-VILLACRES-MORILLO/App/ms-users
npm ci
cd ..
```

La terminal queda dentro de **`App`**. Si ya se descargó o copió el código, omitir `git clone`, entrar en `App/ms-users`, ejecutar `npm ci` y volver a `App`.

`npm ci` utiliza `package-lock.json` e instala, entre otras dependencias, `dotenv`, necesaria para el inicializador conjunto. No hace falta repetirlo en cada arranque.

### 2. Generar configuración y credenciales locales

Desde **`App`**:

```sh
node scripts/init-env.cjs
```

El inicializador:

1. Crea `ms-users/.env` y `ms-stego/.env` a partir de los ejemplos.
2. Genera contraseñas aleatorias y la credencial de comunicación entre servicios cuando faltan o contienen marcadores `CHANGE_ME`.
3. Genera un par RSA de 3072 bits para firmar y verificar JWT.
4. Crea la CA y el certificado TLS de desarrollo de AUTH.
5. Copia a STEGO únicamente la clave pública JWT y la CA, y sincroniza su credencial de servicio, emisor y audiencia.

Conserva los valores y las claves existentes. No es necesario pedir al equipo contraseñas ni copiar sus claves privadas. Los `.env` reales y las carpetas `secrets` están excluidos de Git; los `.env.example` sí se versionan.

Para esta demostración se utiliza `NODE_ENV=development`, valor inicial del ejemplo. La siembra de cuentas está deshabilitada en producción.

**Linux:** AUTH se ejecuta como UID 1000 y necesita leer sus archivos montados. El inicializador guarda las claves privadas con permiso `600`. Si el UID del usuario anfitrión es distinto de 1000, conceder al UID 1000 acceso de lectura y paso sobre esos archivos y sus directorios mediante ACL o una política equivalente; no hacer públicas las claves. En Docker Desktop para Windows, los permisos de los montajes se gestionan de forma diferente.

### 3. Construir e iniciar ambos microservicios

Desde **`App`**, con Docker iniciado:

```sh
node scripts/compose.cjs -p seguro-u1-v1 up -d --build --wait
node scripts/compose.cjs -p seguro-u1-v1 ps
```

La primera construcción puede tardar varios minutos. Deben aparecer `auth`, `stego` y `postgres` en ejecución y saludables. AUTH aplica sus migraciones al iniciar. STEGO crea su esquema SQLite automáticamente.

El nombre explícito **`-p seguro-u1-v1`** identifica los contenedores, redes y volúmenes de esta instalación. Mantenerlo en los comandos posteriores. Evita reutilizar por accidente los volúmenes del proyecto anterior, cuyo directorio también se llamaba `App`. Dos instalaciones con el mismo nombre de proyecto y el mismo motor Docker comparten esos recursos; dos proyectos distintos tampoco pueden ocupar los mismos puertos al mismo tiempo.

### 4. Crear las cuentas de demostración

Desde **`App`**:

```sh
node scripts/compose.cjs -p seguro-u1-v1 exec auth node dist/database/seed.js
```

| Usuario | Roles |
|---|---|
| `jperez` | ADMIN |
| `amendoza` | ANALYST |
| `mgomez` | USER |
| `cvega` | USER y ANALYST |
| `lcastro` | USER |

Abrir **`App/ms-users/.env`** en el editor y buscar `SEED_PASSWORD=`. El valor a su derecha es la contraseña local de las cuentas recién sembradas. No hay una contraseña universal en el repositorio.

La siembra puede repetirse, pero no cambia las contraseñas ni los privilegios de usuarios existentes. Cambiar `SEED_PASSWORD` después no actualiza las cuentas ya creadas.

### 5. Comprobar las APIs

| Servicio | Salud | Documentación interactiva |
|---|---|---|
| AUTH | `https://localhost:3443/api/v1/health` | `https://localhost:3443/docs` |
| STEGO | `http://localhost:8000/health` | `http://localhost:8000/docs` |

AUTH también ofrece HTTP local en `http://localhost:3001/api/v1`, pero **la comunicación STEGO → AUTH siempre utiliza HTTPS**. Para verificar AUTH por HTTPS, el cliente debe confiar en la CA generada. No desactivar la verificación TLS.

Los puertos publicados están limitados a `127.0.0.1`. Esta configuración sirve para desarrollo y evaluación local. Un despliegue remoto necesita proteger con TLS también la entrada de STEGO y gestionar certificados y secretos apropiados.

## Primer uso con Postman

### Preparar la colección

1. Abrir Postman Desktop y pulsar **Import**.
2. Seleccionar [`App/docs/pruebas/backend-v1.postman_collection.json`](App/docs/pruebas/backend-v1.postman_collection.json).
3. Abrir **Settings → Certificates**, habilitar **CA Certificates** e importar `App/ms-users/secrets/ca.crt` como CA confiable. No añadirla como certificado de cliente ni proporcionar una clave privada. Mantener activada la verificación SSL/TLS.
4. En **Settings → General → Working directory**, seleccionar `App/docs/pruebas/muestras`. Si Postman no conserva esa selección, adjuntar manualmente el PNG en la solicitud.
5. Seleccionar **No environment** para evitar que variables de otro entorno sobrescriban las de la colección.
6. Abrir el título de la colección → **Variables** y comprobar:

| Variable | Valor local |
|---|---|
| `base_url` | `https://localhost:3443/api/v1` |
| `stego_url` | `http://localhost:8000` |
| `seed_password` | El valor de `SEED_PASSWORD` de tu `.env`. |
| `new_user_password` | Una contraseña temporal de 12 a 128 caracteres para crear cuentas de prueba. |

Mantener contraseñas y JWT en valores locales de Postman; no publicarlos ni incluirlos en el repositorio. Los IDs de usuarios, roles, asignaciones y análisis se obtienen de la base de cada instalación: no reutilizar los UUID de las evidencias como si fueran los propios.

### Analizar una imagen y recuperar el resultado

1. Abrir **AUTH → Login de prueba**. En **Body → raw → JSON**, usar:

   ```json
   {
     "identifier": "amendoza",
     "password": "{{seed_password}}"
   }
   ```

2. Pulsar **Send**. Esperar `200 OK`. La respuesta contiene `accessToken`, `expiresIn: 900` y los datos de la cuenta. El script de la colección guarda el token en `analyst_token`.
3. Abrir **STEGO → Analizar imagen**. En **Headers**, mantener una única fila activa:

   ```text
   Authorization: Bearer {{analyst_token}}
   ```

4. Ir a **Body → form-data**. La fila debe llamarse **`file`**, ser de tipo **File** y contener el archivo real `original_control.png` de la carpeta de muestras. Postman genera el encabezado multipart; no escribirlo manualmente.
5. Pulsar **Send**. Esperar `200`, el estado `sin_indicios`, sus indicadores y `analysis_id`. La colección guarda ese ID al recibir un análisis exitoso.
6. Repetir seleccionando `oculta_lsb.png`. Esperar `200` con estado `sospechoso` y acción `cuarentena`.
7. Abrir **Consultar resultado** y pulsar **Send**. Ejecuta `GET /api/v1/steganography/results/{{analysis_id}}` con el token del analista.

Para utilizar ADMIN, ejecutar **Login ADMIN** y cambiar el encabezado de STEGO a `Bearer {{access_token}}`. Los tokens duran inicialmente **15 minutos**; iniciar sesión nuevamente cuando expiren.

### Comprobar denegaciones

| Prueba | Acción en Postman | Resultado esperado |
|---|---|---|
| Sin JWT | Desmarcar `Authorization`; comprobar que no exista autenticación heredada. | 401 |
| JWT inválido | Usar `Bearer token-invalido`. | 401 |
| USER sin permiso | En **Login de prueba** usar `mgomez`; en STEGO usar `Bearer {{user_token}}`. | 403 |
| Resultado ajeno | Iniciar sesión como `cvega`, usar `Bearer {{other_analyst_token}}` e intentar consultar el análisis creado por `amendoza`. | 403 |
| Rol revocado | Con ADMIN retirar la asignación ANALYST de `amendoza`; reutilizar su JWT en STEGO. Restaurar después la asignación. | 403 |
| AUTH inaccesible | Detener AUTH y analizar con un JWT vigente. Reiniciar AUTH al terminar. | 503; no se analiza. |

Para probar solo autenticación o permisos se puede usar **Body → none**, evitando problemas de lectura de archivos del cliente. El [guion detallado](App/docs/05_guion_sustentacion_postman.md) explica la creación de cuentas, asignaciones, revocación y todos los escenarios T01–T12.

## Funcionamiento y comunicación

```text
Cliente REST / Postman
   |
   +-- Credenciales --> AUTH (NestJS) --> PostgreSQL
   |                      |
   |<-- JWT firmado ------+
   |
   +-- PNG + JWT ------> STEGO (FastAPI)
                            |
                            +-- HTTPS + identidad de servicio --> AUTH
                            |<-- Decisión de permisos vigentes ---+
                            |
                            +-- Trabajador de análisis PNG
                            +-- SQLite + cuarentena privada
                            |
   <------------------------+ Resultado JSON
```

### AUTH y PostgreSQL

AUTH administra usuarios, roles y asignaciones N:M. La tabla `user_roles` tiene su propio ID, claves foráneas y una restricción única por par usuario–rol. Las contraseñas se verifican con **Argon2id** y los JWT se firman mediante **RS256**. Los nombres de rol se normalizan para evitar tratar `admin`, `Admin` y `ADMIN` como identidades distintas.

El JWT identifica al usuario y contiene una versión de sesión; **no contiene roles**. AUTH consulta los roles y asignaciones activos en PostgreSQL. El logout incrementa `tokenVersion` e invalida los tokens anteriores del usuario. La validación de entradas, los guards y el limitador de solicitudes protegen las operaciones.

### STEGO y SQLite

STEGO verifica localmente la firma, expiración, emisor, audiencia y campos exigidos del JWT con la **clave pública** de AUTH. Después solicita una decisión nueva por cada operación protegida. No tiene acceso directo a PostgreSQL ni recibe la clave privada JWT.

Su base propia, **SQLite**, guarda los análisis y la auditoría en `/data/stego.sqlite3`. Los archivos retenidos se guardan en `/data/quarantine` con nombres UUID internos. No hay una descarga pública del archivo.

### Canal de autorización

Dentro de la red Docker, STEGO encuentra a AUTH mediante `https://auth:3443`. `auth` es el nombre del servicio, no una dirección que deba utilizarse desde Postman en el equipo anfitrión.

STEGO llama a `POST /internal/authorization/check` enviando el JWT del usuario, la acción y el recurso. Presenta además **`X-Service-Token`**, su credencial de servicio. La conexión requiere TLS 1.3 y valida la CA de AUTH. El JWT identifica al usuario; la credencial de servicio identifica a STEGO.

AUTH vuelve a verificar la identidad, la versión de sesión y los roles activos. STEGO comprueba que la decisión corresponda al usuario, acción y recurso solicitados. Si falta permiso, responde 403; si AUTH no está disponible, deniega el procesamiento con 503. Una respuesta interna no verificable produce 502. No se conserva una autorización permanente.

Para leer resultados, STEGO obtiene el propietario desde SQLite. ADMIN puede consultar todos; ANALYST solo los propios. Retirar un rol tiene efecto en la siguiente operación aunque el JWT siga vigente.

Cada servicio tiene su proceso, imagen Docker, configuración y API. AUTH funciona con STEGO detenido; STEGO conserva su API de salud si AUTH cae, pero no realiza operaciones protegidas sin una decisión válida.

## APIs y permisos

Las rutas de negocio utilizan `/api/v1`. El contrato interno no lleva ese prefijo.

| Servicio | Método y ruta | Acceso |
|---|---|---|
| AUTH | `GET /api/v1/health` | Público, salud del proceso. |
| AUTH | `POST /api/v1/auth/login` | Público con limitación de solicitudes. |
| AUTH | `GET /api/v1/auth/profile` y alias `/api/v1/auth/me` | Sesión válida. |
| AUTH | `POST /api/v1/auth/logout` | Sesión válida. |
| AUTH | `POST/GET /api/v1/users`; `GET/PATCH/DELETE /api/v1/users/{id}` | ADMIN. |
| AUTH | `GET /api/v1/users/cedula/{cedula}` | ADMIN. |
| AUTH | `POST/GET /api/v1/roles`; `GET/PATCH/DELETE /api/v1/roles/{id}` | ADMIN. |
| AUTH | `POST /api/v1/user-roles` | ADMIN; asignar rol. |
| AUTH | `GET /api/v1/user-roles/user/{userId}` | ADMIN; consultar asignaciones. |
| AUTH | `DELETE /api/v1/user-roles/{id}` | ADMIN; retirar la asignación por su propio ID. |
| AUTH | `POST /internal/authorization/check` | Servicio autenticado por TLS y credencial. |
| STEGO | `GET /health` | Público, salud de STEGO; no acredita disponibilidad de AUTH. |
| STEGO | `POST /api/v1/steganography/analyze` | ADMIN o ANALYST; multipart con un archivo `file`. |
| STEGO | `GET /api/v1/steganography/results/{analysis_id}` | ADMIN o ANALYST propietario. |

Éxitos: **200** con JSON, **201** al crear recursos de AUTH y **204** sin cuerpo al eliminar, retirar una asignación o cerrar sesión. Las listas de AUTH usan `page` y `limit` opcionales, inicialmente 1 y 20.

Errores principales: **400** entrada inválida o PNG corrupto; **401** autenticación inválida; **403** falta de permiso; **404** recurso inexistente; **409** duplicidad/conflicto; **413** tamaño o dimensiones excesivos; **415** formato no admitido; **429** limitación de AUTH; **502/503** fallos de autorización o procesamiento; **500** fallo interno. Un análisis sospechoso válido devuelve **200**, no un error HTTP.

Los [contratos OpenAPI](App/docs/contratos) especifican cuerpos y respuestas. FastAPI genera el contrato de STEGO en `/openapi.json`; Swagger UI lo presenta en `/docs`. AUTH también ofrece Swagger. Postman conserva solicitudes para ejecutar los flujos.

## Método esteganográfico y muestras

Se admite **PNG de 8 bits** en escala de grises (`L`), RGB o RGBA; se excluye alfa del cálculo. No se admiten JPEG, BMP, PNG indexado, de 16 bits ni animado. El servicio valida extensión, MIME `image/png`, firma real, estructura, dimensiones y tamaño.

La técnica `pov_pearson_lsb_png8_v1` calcula un contraste de Pearson sobre las frecuencias de los pares `(0,1), (2,3), ..., (254,255)`. Busca equiprobabilidad compatible con reemplazo LSB; no extrae mensajes ni conoce el encabezado de las muestras.

| Parámetro inicial | Valor |
|---|---|
| Archivo máximo | 5 MiB |
| Dimensiones | 2048 por lado; máximo 4 194 304 píxeles |
| Evidencia estadística mínima | 8192 muestras de canal, 64 parejas elegibles y 10 observaciones por pareja |
| Clasificación | `p ≥ 0.05`: `sospechoso`; `p < 0.01`: `sin_indicios`; intervalo intermedio o evidencia insuficiente: `no_concluyente` |
| Fallos de procesamiento | Estado `error` cuando se registra el análisis fallido; nunca se presenta como limpio |
| Recursos del trabajador | Dos trabajos simultáneos por instancia; 10 s de pared, 5 s de CPU inicialmente y 512 MiB de espacio virtual por proceso |
| Retención | Cuarentena de archivos durante 7 días; purga al iniciar y cada 60 s; metadatos persistentes |

Los parámetros están en `App/ms-stego/.env.example` y su validación en `settings.py`. El trabajador aplica límites y restricciones de red Linux mediante seccomp, sin heredar credenciales. Los contenedores usan usuarios sin privilegios y raíz de solo lectura. El trabajador comparte UID y volumen con STEGO; no es una frontera completa de sistema de archivos frente a un fallo del decodificador.

`sin_indicios` genera advertencia y elimina el archivo tras registrar los resultados; los demás estados de análisis se aíslan en cuarentena. Los rechazos previos de formato se auditan sin generar necesariamente un análisis. La respuesta conserva UUID, usuario, nombre saneado, SHA-256, técnica, indicadores, fecha, estado, acción y limitaciones.

| Muestra incluida | Construcción | Resultado reproducible |
|---|---|---|
| `original_control.png` | Control sintético sin carga. | `sin_indicios` |
| `oculta_lsb.png` | Mensaje real con reemplazo LSB de alta ocupación. | `sospechoso` |
| `oculta_baja_carga.png` | Mensaje de baja ocupación. | `sin_indicios`: falso negativo. |
| `ruido_sin_mensaje.png` | Ruido benigno sin mensaje. | `sospechoso`: falso positivo. |
| `pequena.png` | Imagen 8 × 8. | `no_concluyente` |

El generador comprueba que puede recuperar el mensaje insertado. El [manifiesto](App/docs/pruebas/muestras/manifest.json) registra construcción, hashes e indicadores. Estas muestras no constituyen un corpus independiente: **no se declara precisión general ni detección universal**. Un p-valor no representa la probabilidad de que exista un mensaje. La baja carga y otras técnicas pueden evadir el contraste; el ruido puede producir falsos positivos.

## Operación y persistencia

Todos los comandos siguientes se ejecutan desde **`App`**.

| Operación | Comando |
|---|---|
| Iniciar o actualizar todo | `node scripts/compose.cjs -p seguro-u1-v1 up -d --build --wait` |
| Ver estado | `node scripts/compose.cjs -p seguro-u1-v1 ps` |
| Iniciar/reconstruir solo STEGO | `node scripts/compose.cjs -p seguro-u1-v1 up -d --build --wait stego` |
| Iniciar/reconstruir AUTH y su PostgreSQL | `node scripts/compose.cjs -p seguro-u1-v1 up -d --build --wait auth` |
| Detener STEGO | `node scripts/compose.cjs -p seguro-u1-v1 stop stego` |
| Reanudar STEGO | `node scripts/compose.cjs -p seguro-u1-v1 start stego` |
| Detener AUTH | `node scripts/compose.cjs -p seguro-u1-v1 stop auth` |
| Reanudar AUTH | `node scripts/compose.cjs -p seguro-u1-v1 start auth` |
| Ver registros de AUTH | `node scripts/compose.cjs -p seguro-u1-v1 logs --tail 50 auth` |
| Seguir registros de STEGO | `node scripts/compose.cjs -p seguro-u1-v1 logs -f stego` |
| Apagar conservando datos | `node scripts/compose.cjs -p seguro-u1-v1 down` |

`Ctrl+C` deja de seguir los registros sin apagar el contenedor. Para incorporar cambios del código, usar `up --build`; `restart` no reconstruye las imágenes.

El script `compose.cjs` ejecuta Docker Compose cargando `ms-users/.env`; no inicia un proceso Node adicional del backend. El Compose conjunto está en `App/docker-compose.yml`. El Compose dentro de `ms-users` sirve para AUTH por separado; no es necesario ejecutarlo además del conjunto.

Los volúmenes `seguro-u1-v1_users_postgres_data` y `seguro-u1-v1_stego_data` conservan identidades, resultados, auditoría y archivos retenidos. **`down --volumes` borra los datos del proyecto seleccionado**; no usarlo para apagar normalmente. Cambiar de carpeta, nombre de proyecto o archivo `.env` no migra bases existentes. Las claves y certificados quedan en las carpetas locales `secrets`; también son necesarios al volver a iniciar.

## Pruebas reproducibles

### AUTH

Desde **`App/ms-users`**, después de `npm ci` y la inicialización:

```sh
npm run build
npm run lint
npm test -- --runInBand
npm run test:docker
npm audit --omit=dev
```

`test:docker` crea un PostgreSQL temporal para pruebas funcionales y de seguridad, sin utilizar la base normal. `npm audit` consulta avisos conocidos; no certifica la seguridad de la aplicación.

### STEGO e integración en Windows

Con Python 3.12 instalado, desde **`App`**:

```powershell
py -3.12 -m venv ms-stego/.venv
.\ms-stego\.venv\Scripts\python.exe -m pip install -r ms-stego/requirements-dev.txt
.\ms-stego\.venv\Scripts\python.exe -m pytest -q ms-stego/tests -c ms-stego/pytest.ini
.\ms-stego\.venv\Scripts\python.exe tests/run.py
```

Si no existe el lanzador `py`, usar `python -m venv ms-stego/.venv` después de comprobar que `python --version` es 3.12.

### STEGO e integración en Linux/macOS

Desde **`App`**, con Python 3.12 disponible:

```sh
python3.12 -m venv ms-stego/.venv
ms-stego/.venv/bin/python -m pip install -r ms-stego/requirements-dev.txt
ms-stego/.venv/bin/python -m pytest -q ms-stego/tests -c ms-stego/pytest.ini
ms-stego/.venv/bin/python tests/run.py
```

Las pruebas integradas requieren Docker y la configuración inicial. Crean nombre de proyecto, puertos y volúmenes temporales, ejecutan ambos servicios reales y limpian únicamente sus recursos. No usan los volúmenes `seguro-u1-v1`. Regeneran resultados bajo `App/docs/pruebas/resultados`.

Con Postman CLI instalado y disponible en PATH, añadir `--postman` al comando de `tests/run.py` para ejecutar también el ensayo de la colección.

Para regenerar las imágenes desde **`App`**:

```powershell
cd ms-stego
.\.venv\Scripts\python.exe -m scripts.generate_samples
cd ..
```

En Linux/macOS, sustituir el ejecutable por `.venv/bin/python`. Las muestras existentes ya se incluyen: no es necesario regenerarlas para utilizar la API.

La [guía de pruebas y evidencias](App/docs/04_pruebas_y_evidencias.md) relaciona los casos con T01–T12. Las evidencias incluidas documentan ejecuciones anteriores; al repetirlas cambian fechas y UUID. No equivalen a una prueba de carga, una certificación ni una evaluación independiente de precisión.

## Solución de problemas

| Síntoma | Comprobación y solución |
|---|---|
| No conecta con Docker | Abrir Docker Desktop, usar contenedores Linux y comprobar `docker version`. |
| `Cannot find module ... dotenv` | Ejecutar `npm ci` dentro de `App/ms-users` y volver a `App` antes de inicializar. |
| Falta `.env` o una clave | Desde `App`, ejecutar `node scripts/init-env.cjs` antes de Compose. No crear archivos vacíos como sustituto. |
| OpenSSL no encontrado | Instalarlo o indicar su ejecutable en `OPENSSL_PATH`; repetir el inicializador. |
| Puerto ocupado | Detener los contenedores o procesos de la instalación anterior desde Docker Desktop. No ejecutar simultáneamente dos conjuntos con los mismos puertos. |
| PostgreSQL rechaza la contraseña tras cambiar `.env` | Cambiar `DB_PASSWORD` no modifica la contraseña de una base ya inicializada. Recuperar la configuración correspondiente o migrar/rotar las credenciales de forma controlada; no borrar volúmenes para corregirlo. |
| Login responde 401 | Comprobar la siembra, la contraseña local y el proyecto Compose. Repetir el seed no cambia contraseñas existentes. |
| STEGO responde 401 | Renovar el JWT y comprobar que el token pertenece a este AUTH. |
| STEGO responde 403 | Revisar rol vigente, asignación activa y propietario del análisis. |
| STEGO responde 503 | Comprobar AUTH, certificados, credencial sincronizada y registros; también puede indicar capacidad o procesamiento no disponibles. |
| Advertencia de certificado | Confiar en el `ca.crt` de esta instalación y comprobar vigencia. Los certificados de desarrollo duran 30 días. No desactivar TLS. |
| Postman muestra `EISDIR` | Seleccionar un archivo PNG real en `file`, no una carpeta; revisar la carpeta de trabajo. El error ocurre en el cliente antes de llegar a la API. |
| 413 o 415 al subir imagen | Revisar tamaño, dimensiones y variante PNG. Renombrar una extensión no convierte el archivo. |
| AUTH devuelve 429 | Esperar el período de bloqueo. El login admite inicialmente cinco intentos por minuto por origen. |
| Un contenedor no queda saludable | Consultar `ps` y `logs --tail 100 auth`, `stego` o `postgres` con el mismo prefijo `node scripts/compose.cjs -p seguro-u1-v1`. |

Para renovar TLS de desarrollo, detener los servicios, respaldar el material existente y retirar solo el certificado/clave TLS de servidor vencidos; si también venció la CA, retirar además su certificado, clave y serial para regenerarla. Ejecutar el inicializador, volver a distribuir la nueva CA a Postman si cambió y recrear los servicios. Conservar el par JWT y los volúmenes. La rotación de la clave JWT es un procedimiento distinto y exige distribuir la nueva pública e iniciar sesión de nuevo.

## Documentación técnica

```text
.
├── README.md                    Guía de instalación y uso de este repositorio
└── App/
    ├── ms-users/                AUTH: NestJS, migraciones y siembra
    ├── ms-stego/                STEGO: FastAPI, detector y SQLite
    ├── docker-compose.yml      Despliegue conjunto
    ├── scripts/                Inicialización y acceso a Compose
    ├── tests/                  Pruebas de integración y Postman
    └── docs/
        ├── contratos/          OpenAPI exportados
        ├── diagramas/          Arquitectura y modelo N:M en SVG
        ├── pruebas/            Colección REST, imágenes y evidencias
        └── historico/          Checklists y registros de fases anteriores
```

| Documento | Contenido |
|---|---|
| [Arquitectura y modelo de datos](App/docs/01_arquitectura_y_modelo_datos.md) | Responsabilidades, comunicación, límites de confianza y relación N:M. |
| [Seguridad, contratos y método](App/docs/02_seguridad_contratos_y_metodo.md) | JWT, RBAC, decisiones vigentes, parámetros y limitaciones del detector. |
| [Matriz de riesgos](App/docs/03_matriz_riesgos.md) | Riesgos, controles y enfoques de desarrollo seguro. |
| [Pruebas y evidencias](App/docs/04_pruebas_y_evidencias.md) | Ejecuciones reproducibles y trazabilidad de escenarios. |
| [Guion Postman](App/docs/05_guion_sustentacion_postman.md) | Pasos detallados para la demostración. |
| [OpenAPI AUTH](App/docs/contratos/openapi-auth.json) y [OpenAPI STEGO](App/docs/contratos/openapi-stego.json) | Contratos de las APIs. |
| [README de AUTH](App/ms-users/README.md) y [README de STEGO](App/ms-stego/README.md) | Detalles de cada servicio y ejecución individual. |

Para compartir el proyecto, versionar código, lockfiles, ejemplos de configuración y documentación. Cada persona genera sus credenciales al instalar. No agregar `.env`, claves, certificados privados, bases locales ni dependencias descargadas. Los datos de demostración y los resultados incluidos no sustituyen la gestión de secretos, la retención de datos ni la operación necesaria para un entorno de producción.
