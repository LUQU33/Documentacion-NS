# Glosario

Términos que aparecen en la documentación, explicados sin dar nada por sabido. Ordenados alfabéticamente.

## API REST

Una **API** es un conjunto de "puertas" (URLs) que un programa expone para que otros programas le pidan cosas. **REST** es un estilo para diseñarla: cada URL representa un *recurso* (`/api/productos`, `/api/productos/5`) y el *método HTTP* indica qué hacer con él (ver [Método HTTP](#método-http)). El frontend de NavajaStyle habla con este backend a través de su API REST.

## async / await

Forma de escribir en JavaScript código que **espera** algo lento (una consulta a la base, un cálculo de hash) sin bloquear al servidor. Una función marcada `async` siempre devuelve una [promesa](#promesa); adentro, `await` pausa *esa función* hasta que la promesa se resuelva, mientras Node sigue atendiendo otros pedidos. Si dentro de una función `async` se hace `throw`, la promesa que devuelve queda **rechazada**.

## Base64url

Forma de convertir datos binarios o texto a una cadena que solo usa letras, números, `-` y `_`, para que se pueda mandar sin problemas en una URL o un header HTTP. **No es cifrado**: cualquiera puede decodificarla. Se usa en el [token](#token) de autenticación.

## Body (cuerpo)

La parte de un [request](#request-y-response) o response que lleva los datos "grandes". Por ejemplo, al crear un producto, el body es `{"nombre": "Remera", "precio": 5000, ...}`. Los requests `GET` normalmente no tienen body.

## Constraint (restricción de base de datos)

Regla que la propia base de datos hace cumplir, sin importar qué programa intente escribir: `NOT NULL` (no puede estar vacío), `UNIQUE` (no se puede repetir), `CHECK` (tiene que cumplir una condición, ej. `precio >= 0`) y `FOREIGN KEY` (ver [Foreign key](#foreign-key-clave-foránea)). Si una operación viola una constraint, MySQL la rechaza con un error.

## Controller (controlador)

Función que recibe un request ya ruteado, decide qué hacer (validar, llamar al [modelo](#model-modelo), elegir el status code) y arma la respuesta. Es la capa "C" de [MVC](#mvc).

## CRUD

Las cuatro operaciones básicas sobre datos: **C**reate (crear), **R**ead (leer), **U**pdate (actualizar), **D**elete (borrar). En HTTP se mapean a `POST`, `GET`, `PUT`, `DELETE`.

## dotenv / variables de entorno

Las **variables de entorno** son valores de configuración que el sistema operativo le pasa a un programa al arrancar (puerto, contraseña de la base, claves secretas). En Node se leen con `process.env.NOMBRE`. **dotenv** es una librería que lee el archivo `.env` y carga su contenido en `process.env`, para no tener que configurarlas a mano en cada máquina.

## Endpoint

Combinación de [método HTTP](#método-http) + URL que la API atiende. `GET /api/productos` y `POST /api/productos` son dos endpoints distintos aunque compartan URL.

## Express

Librería (framework) de Node.js para construir servidores HTTP. Se encarga de recibir pedidos, decidir qué función los atiende ([ruteo](#router)) y ofrecer ayudas para responder (`res.json`, `res.status`). Este proyecto usa **Express 5**.

## Factory (fábrica)

Función que **construye y devuelve otras funciones u objetos** a partir de una configuración. En el proyecto, `crearModeloCRUD({ tabla: "talles", ... })` devuelve un modelo completo para la tabla `talles`. Evita escribir el mismo código cinco veces.

## Foreign key (clave foránea)

Columna que apunta al `id` de otra tabla y que la base valida: `productos.categoria_id` tiene que existir en `categorias.id`. Define también qué pasa al borrar el registro apuntado: `RESTRICT` (no te deja), `CASCADE` (borra también los que lo apuntan) o `SET NULL` (los deja en `NULL`).

## Hash

Resultado de pasar un dato por una función matemática de **una sola vía**: a partir de la contraseña es fácil calcular el hash, pero a partir del hash es prácticamente imposible obtener la contraseña. Por eso se guarda el hash y no la contraseña. Ver también [Salt](#salt).

## Header

Dato extra que viaja junto a un request o response, con forma `Nombre: valor`. Ejemplos: `Content-Type: application/json` (en qué formato viene el body) y `Authorization: Bearer <token>` (quién sos).

## HMAC

"Firma" calculada mezclando un dato con una **clave secreta** mediante una función de hash. Quien no tenga la clave no puede generar una firma válida, así que sirve para detectar si alguien modificó el dato. Se usa para firmar el [token](#token).

## JSON

Formato de texto para representar datos: `{"nombre": "Remera", "precio": 5000}`. Es el idioma en que el frontend y el backend se mandan información.

## Middleware

Función que Express ejecuta **en el medio** del camino de un request, antes de llegar (o en lugar de llegar) a la respuesta final. Recibe `(req, res, next)` y puede: modificar `req`, responder y cortar el camino, o llamar a `next()` para pasarle el turno al siguiente. Los middlewares de **errores** reciben un parámetro más: `(err, req, res, next)`.

## Model (modelo)

Capa que habla con la base de datos: sabe armar las consultas SQL y devolver los datos como objetos de JavaScript. Es la "M" de [MVC](#mvc).

## Método HTTP

Verbo que indica la intención del request: `GET` (leer), `POST` (crear), `PUT` (reemplazar/actualizar), `DELETE` (borrar).

## Módulo (require / module.exports)

Cada archivo `.js` en Node es un módulo. `module.exports = algo` define qué entrega ese archivo; `require("./archivo")` lo carga y devuelve ese "algo". **Node ejecuta cada módulo una sola vez**, la primera vez que alguien lo pide, y después reutiliza el resultado (queda en caché).

## MVC

Patrón que separa la aplicación en **M**odelo (datos), **V**ista (presentación) y **C**ontrolador (lógica que conecta ambas). En una API, la "vista" es el JSON que se devuelve; en este backend las capas son *rutas → controllers → models*.

## Payload

El "contenido útil" de algo. En el token, el payload es el objeto `{ id, email, rol, exp }` con los datos del usuario.

## Pool de conexiones

Conjunto de conexiones a la base de datos que se abren una vez y se **reutilizan**. Abrir una conexión a MySQL es lento (red, autenticación); con un pool, cada consulta toma una conexión libre, la usa y la devuelve. `mysql2` crea hasta 10 por defecto.

## Promesa

Objeto de JavaScript que representa un resultado que **todavía no está**: va a resolverse (con un valor) o rechazarse (con un error) en el futuro. Ver [async / await](#async--await).

## Query parametrizada (placeholders `?`)

Consulta SQL donde los valores que vienen del usuario no se pegan como texto, sino que se marcan con `?` y se pasan aparte. La librería los **escapa** (neutraliza caracteres peligrosos como `'`) antes de mandarlos. Es la defensa contra [SQL injection](#sql-injection).

## Request y response

El **request** (`req`) es el pedido que manda el cliente: método, URL, headers y body. El **response** (`res`) es la respuesta del servidor: status code, headers y body. En Express, cada middleware recibe ambos objetos.

## Router

Objeto de Express que agrupa rutas. Funciona como una "mini aplicación": se le registran rutas (`router.get(...)`) y después se monta entero en la app con `app.use(router)`.

## Salt

Valor aleatorio que se mezcla con la contraseña antes de calcular el [hash](#hash). Hace que dos usuarios con la misma contraseña tengan hashes distintos, y que no sirvan tablas precalculadas de hashes de contraseñas comunes.

## SQL injection

Ataque en el que alguien manda texto que, si se pega directo en una consulta SQL, la modifica. Ej.: un email `' OR 1=1 --` podría convertir "buscá este usuario" en "traé todos". Se previene con [queries parametrizadas](#query-parametrizada-placeholders-).

## Status code

Número de tres dígitos que resume cómo salió el request. Los que usa el proyecto:

| Código | Significado | Cuándo lo usamos |
|---|---|---|
| 200 | OK | Lectura/actualización exitosa (es el default de `res.json`) |
| 201 | Created | Se creó un recurso |
| 400 | Bad Request | El cliente mandó datos inválidos |
| 401 | Unauthorized | No se identificó o su token no sirve |
| 403 | Forbidden | Se identificó pero no tiene permitido el acceso |
| 404 | Not Found | La ruta o el recurso no existen |
| 409 | Conflict | Choca con algo existente (ej. email repetido) |
| 500 | Internal Server Error | Falló algo del lado del servidor |

## Timing attack / comparación en tiempo constante

Una comparación común (`a === b`) se detiene en el primer carácter distinto, así que tarda un poquito más cuanto más se parecen los valores. Midiendo esos tiempos, un atacante podría adivinar una firma o un hash de a un carácter. `crypto.timingSafeEqual` compara **siempre todos los bytes**, tardando lo mismo sin importar dónde está la diferencia.

## Token

Cadena que el servidor entrega al hacer login y que el cliente reenvía en cada request para demostrar quién es, sin mandar la contraseña de nuevo. En este proyecto tiene la forma `payload.firma` (ver [08-autenticacion.md](08-autenticacion.md)).
