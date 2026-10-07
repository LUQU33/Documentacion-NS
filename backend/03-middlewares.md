# Middlewares

> **Archivos:** [`src/middlewares/logger.middleware.js`](../../Navaja-Style/backend/src/middlewares/logger.middleware.js), [`src/middlewares/errorHandler.middleware.js`](../../Navaja-Style/backend/src/middlewares/errorHandler.middleware.js), [`src/middlewares/auth.middleware.js`](../../Navaja-Style/backend/src/middlewares/auth.middleware.js)
> **Depende de:** [02-arranque.md](02-arranque.md) · **Conceptos previos:** [middleware](glosario.md#middleware), [request y response](glosario.md#request-y-response)

## En una frase

Un middleware es una función que se mete **en el medio** del camino de un request para hacer algo antes (o en vez) de la respuesta final: registrar, verificar permisos, transformar datos o manejar errores.

## Cómo funciona un middleware

Express llama a cada middleware con tres argumentos:

| Parámetro | Qué es |
|---|---|
| `req` | El request. Se puede **leer** (`req.url`, `req.body`) y también **agregarle** cosas (`req.usuario = ...`) para los middlewares que vienen después. |
| `res` | El response. Si lo usás para responder (`res.json(...)`), el recorrido termina ahí. |
| `next` | Función para pasarle el turno al siguiente middleware de la lista. |

Con esas piezas, cada middleware termina de **una de tres formas**:

1. **Responde** (`res.json(...)`, `res.status(401).json(...)`): el ciclo termina; los middlewares que siguen no se ejecutan.
2. **Llama a `next()`** sin argumentos: "terminé lo mío, que siga el próximo".
3. **Llama a `next(error)`** con un argumento (o hace `throw` / devuelve una promesa rechazada): "algo salió mal". Express **saltea todos los middlewares normales** y busca el próximo middleware de errores.

**Si no hace ninguna de las tres**: el request queda **colgado**. El cliente espera una respuesta que nunca llega hasta que se corta por timeout. Es el error más típico al escribir un middleware a mano.

### Middlewares globales vs. de ruta

- **Globales**: se registran con `app.use(fn)` y se ejecutan para todos los requests (el `logger`, por ejemplo).
- **De ruta**: se pasan como argumentos extra al definir una ruta y solo se ejecutan para esa ruta:

```js
// src/routes/usuario.routes.js:7
router.get("/usuarios/me", verificarAutenticacion, usuarioController.obtenerMe);
```

Express ejecuta las funciones **de izquierda a derecha**: primero `verificarAutenticacion`; solo si este llama a `next()`, se ejecuta `obtenerMe`. Es la misma idea de "lista ordenada" que en `app.js`, pero a nivel de una sola ruta.

## logger

```js
// src/middlewares/logger.middleware.js:1-6
function logger(req, res, next) {
  console.log(`${new Date().toISOString()} ${req.method} ${req.url}`);
  next();
}
```

- `new Date().toISOString()`: fecha y hora actuales en formato estándar (`2026-10-06T14:03:11.512Z`). **Por qué ISO**: se puede ordenar alfabéticamente y es igual en cualquier zona horaria (la `Z` indica UTC).
- `req.method` y `req.url`: qué se pidió (`GET /api/productos/5`).
- `next()`: **imprescindible**. El logger no responde nada; solo observa. **Si faltara**, todos los requests quedarían colgados, porque el logger es el primero de la cadena.
- Usa `req.url` (y no `req.originalUrl`) y funciona bien igual porque está montado con `app.use(logger)` **sin prefijo**: a ese nivel, `req.url` todavía es la URL completa.

> Analogía: es el libro de entradas de un edificio. Anota quién entró y a qué hora, pero no decide nada; lo deja pasar.

Limitación: como se ejecuta **al entrar**, no sabe cómo terminó el request (qué status code se respondió ni cuánto tardó). Para eso habría que engancharse al evento `res.on("finish", ...)` o usar una librería como `morgan`.

## notFound: por qué va después de todas las rutas

```js
// src/middlewares/errorHandler.middleware.js:3-5
function notFound(req, res, next) {
  next(new ApiError(404, `Ruta no encontrada: ${req.method} ${req.originalUrl}`));
}
```

- Este middleware **no mira la URL ni compara nada**. Toda su lógica está en **dónde está ubicado**: último de los middlewares normales en [`app.js`](../../Navaja-Style/backend/src/app.js). Si un request llega hasta acá, es porque **ninguna ruta anterior respondió**. Por eso funciona como "atrapa-todo" de rutas inexistentes.
- **Si estuviera antes de `app.use("/api", routes)`**: respondería 404 a *todos* los requests, incluso a los válidos.
- `next(new ApiError(404, ...))`: en lugar de responder él mismo, **delega** la respuesta en el `errorHandler`. **Por qué**: así hay un solo lugar del código que decide el formato de las respuestas de error. Si mañana se quiere agregar un campo (`{ error, codigo, timestamp }`), se cambia solo el `errorHandler`.
- `req.originalUrl` en vez de `req.url`: dentro de routers montados con prefijo, `req.url` pierde el `/api`; `originalUrl` conserva **siempre** la URL tal como la pidió el cliente. Para un mensaje que ve el cliente, queremos la URL completa.

## errorHandler: el middleware de 4 parámetros

```js
// src/middlewares/errorHandler.middleware.js:7-16
function errorHandler(err, req, res, next) {
  const statusCode = err instanceof ApiError ? err.statusCode : 500;
  const message = err instanceof ApiError ? err.message : "Error interno del servidor";

  if (!(err instanceof ApiError)) {
    console.error(err);
  }

  res.status(statusCode).json({ error: message });
}
```

### Cómo sabe Express que es un manejador de errores

**Es una regla de Express, no una convención**: Express distingue un middleware normal de uno de errores **contando cuántos parámetros declara la función**. En JavaScript, toda función tiene la propiedad `.length` con esa cantidad:

```js
function logger(req, res, next) {}          // logger.length === 3
function errorHandler(err, req, res, next) {} // errorHandler.length === 4
```

Verificado en el código de Express 5 (paquete `router` 2.2.0, `lib/layer.js`):

```js
// router/lib/layer.js:109-112 — cuando HAY un error pendiente
if (fn.length !== 4) {
  return next(error)   // no es manejador de errores: lo salteo
}

// router/lib/layer.js:145-148 — cuando NO hay error
if (fn.length > 3) {
  return next()        // es manejador de errores: ahora no corresponde
}
```

Consecuencias:

- **Sin error**, Express **saltea** el `errorHandler` aunque esté en la lista (por eso no molesta que esté siempre registrado).
- **Con error**, Express **saltea** todos los middlewares de 3 parámetros y va directo al `errorHandler`.
- **`next` no se usa adentro, pero tiene que estar declarado.** Si lo borrás, la función pasa a tener 3 parámetros, Express la trata como middleware normal, y los errores terminan en el manejador por defecto de Express (que responde una página HTML con el stack trace en desarrollo, no nuestro JSON).

### Por qué va último

Cuando ocurre un error, Express busca **el próximo** middleware de 4 parámetros **hacia abajo** en la lista. Si el `errorHandler` estuviera arriba de las rutas, los errores de las rutas nunca lo encontrarían. Y tiene que estar **después de `notFound`**, porque `notFound` le pasa su error con `next(...)`.

### Qué hace con el error

Lo que decide qué ve el cliente está explicado en detalle en [07-errores.md](07-errores.md). En resumen: si es un `ApiError` (un error que lanzamos a propósito) usa su status y su mensaje; si es cualquier otra cosa (un bug, MySQL caído) responde siempre `500 "Error interno del servidor"` y deja el detalle solo en la consola del servidor.

## verificarAutenticacion

Middleware **de ruta** que protege los endpoints que requieren un usuario logueado. Se explica completo en [08-autenticacion.md](08-autenticacion.md#el-middleware-verificarautenticacion). Lo importante acá:

- **Responde directamente** con `res.status(401).json(...)` cuando el token falta o no sirve (opción 1: corta la cadena; el controller nunca se ejecuta).
- Cuando el token es válido, **agrega datos a `req`** (`req.usuario = payload`) y llama a `next()` (opción 2). Los controllers que vienen después leen `req.usuario.id` sin tener que verificar nada de nuevo.

## Regla vs. convención

| Aspecto | ¿Regla o convención? | Por qué |
|---|---|---|
| Llamar a `next()` o responder | Regla | Si no, el request queda colgado. |
| 4 parámetros en `errorHandler` | Regla de Express | Se detecta con `fn.length === 4`. |
| `errorHandler` último, `notFound` justo antes | Regla | Express busca hacia abajo. |
| `notFound` delega en `errorHandler` | Convención | Un solo lugar define el formato de error. |
| Sufijo `.middleware.js` en el nombre | Convención del equipo | Consistencia `entidad.tipo.js` (commit `01afeac`). |

## Probalo

```bash
curl -i http://localhost:3000/api/no-existe
```

```http
HTTP/1.1 404 Not Found
Content-Type: application/json; charset=utf-8

{"error":"Ruta no encontrada: GET /api/no-existe"}
```

```bash
curl -i http://localhost:3000/api/usuarios/me
```

```http
HTTP/1.1 401 Unauthorized

{"error":"Token no enviado"}
```

## Errores comunes

- **El request queda "cargando" para siempre** → un middleware no llamó a `next()` ni respondió.
- **Los errores llegan como HTML en vez de JSON** → el `errorHandler` perdió un parámetro o quedó arriba de las rutas.
- **Todo responde 404** → `notFound` quedó antes de `app.use("/api", routes)`.
- **`Cannot set headers after they are sent`** → se respondió dos veces (por ejemplo, `res.json(...)` sin `return` y después otro `res.json(...)`). Por eso los controllers escriben `return res.status(...).json(...)`.
