# Controllers

> **Archivos:** [`src/utils/crud.controller.js`](../../Navaja-Style/backend/src/utils/crud.controller.js), [`src/controllers/*.controller.js`](../../Navaja-Style/backend/src/controllers/)
> **Depende de:** [06-modelos-y-base-de-datos.md](06-modelos-y-base-de-datos.md), [07-errores.md](07-errores.md) · **Lo usa:** [04-rutas.md](04-rutas.md)
> **Conceptos previos:** [controller](glosario.md#controller-controlador), [factory](glosario.md#factory-fábrica), [async / await](glosario.md#async--await), [status code](glosario.md#status-code)

## En una frase

El controller es el que **toma decisiones** con cada request: lee lo que mandó el cliente, valida, le pide datos al modelo y elige qué status code y qué JSON devolver. Es la única capa que habla HTTP (`req`, `res`) y a la vez conoce a los modelos.

## Dos familias de controllers

| Familia | Archivos | Cómo están hechos |
|---|---|---|
| **CRUD genérico** | `categoria`, `producto`, `talle`, `color`, `variante`, `metodoPago` | Se fabrican con `crearControllerCRUD`. Sin validaciones propias. |
| **Escritos a mano** | `usuario`, `auth` | Validan cada campo, normalizan datos y manejan casos especiales (email repetido, usuario inactivo). |

## La fábrica `crearControllerCRUD`

Las seis entidades del catálogo necesitaban exactamente los mismos cinco endpoints. En vez de escribir cinco veces lo mismo, se escribió **una función que fabrica controllers** (commit `01afeac`).

```js
// src/controllers/producto.controller.js:1-8
const productoModel = require("../models/producto.model");
const { crearControllerCRUD } = require("../utils/crud.controller");

module.exports = crearControllerCRUD({
  modelo: productoModel,
  nombreEntidad: "Producto",
  campoNombre: "nombre",
});
```

- Cada controller del catálogo es **solo configuración**: qué modelo usar, cómo se llama la entidad (para los mensajes) y qué campo usar para nombrarla al borrarla.
- **Por qué una fábrica**: si hay que corregir un bug en `eliminar`, se corrige en un solo lugar y queda corregido para las seis entidades. **Si cada controller estuviera copiado**, sería fácil arreglarlo en uno y olvidarse de los otros cinco.

### Cómo funciona la fábrica por dentro

```js
// src/utils/crud.controller.js:3-7 (simplificado)
function crearControllerCRUD({ modelo, nombreEntidad, campoNombre }) {
  async function crear(req, res) {
    const item = await modelo.crear(req.body);
    res.status(201).json(item);
  }
  // ... listar, obtenerPorId, actualizar, eliminar
  return { crear, listar, obtenerPorId, actualizar, eliminar };
}
```

- `{ modelo, nombreEntidad, campoNombre }` en los parámetros: es **desestructuración**. La función recibe un objeto y saca esas tres propiedades como variables. Así, al llamarla, los argumentos tienen nombre y el orden no importa.
- Las funciones de adentro (`crear`, `listar`, ...) **usan `modelo` aunque no lo reciban como parámetro**. Esto funciona por una característica de JavaScript llamada **closure** (clausura): una función definida dentro de otra "recuerda" las variables de la función que la creó, incluso después de que esta terminó. Cada llamada a `crearControllerCRUD` crea un juego nuevo de funciones, cada uno recordando *su propio* `modelo`.
- `return { crear, ... }`: devuelve un objeto con las cinco funciones. Ese objeto es lo que exporta, por ejemplo, `producto.controller.js`, y lo que usan las rutas (`productoController.crear`).

### Cada operación

```js
// src/utils/crud.controller.js:4-12
async function crear(req, res) {
  const item = await modelo.crear(req.body);
  res.status(201).json(item);
}

async function listar(req, res) {
  const items = await modelo.listar();
  res.json(items);
}
```

- `async` + `await`: la consulta a MySQL tarda; `await` espera el resultado **sin bloquear** el servidor (mientras tanto, Node atiende otros requests).
- `res.status(201)`: **201 Created** es el código estándar para "se creó algo". **Si se dejara el default (200)** funcionaría igual, pero el cliente no podría distinguir "creé" de "leí" mirando solo el código.
- `res.json(item)`: convierte el objeto a texto JSON, pone el header `Content-Type: application/json` y envía la respuesta. Sin `.status(...)` antes, el código es 200.
- Se devuelve **el registro completo recién creado** (con su `id` y sus valores por defecto), no solo lo que mandó el cliente: así el frontend recibe el `id` sin hacer otra consulta.

```js
// src/utils/crud.controller.js:14-23
async function obtenerPorId(req, res) {
  const id = Number(req.params.id);
  const item = await modelo.buscarPorId(id);

  if (!item) {
    throw new ApiError(404, `${nombreEntidad} con id ${id} no encontrado`);
  }

  res.json(item);
}
```

- `Number(req.params.id)`: los parámetros de ruta llegan como texto (`"5"`). Se convierte a número porque la columna `id` es `INT`.
- `if (!item)`: si no existe la fila, el modelo devuelve `undefined`, que JavaScript considera "falso".
- `throw new ApiError(404, ...)`: en lugar de responder acá, **lanza** un error. Fijate que la función ni siquiera recibe `next`. **Esto funciona solo porque usamos Express 5**: Express detecta que la función `async` devolvió una promesa rechazada y llama a `next(error)` automáticamente, que lleva el error al `errorHandler`. **En Express 4**, nadie atraparía esa promesa rechazada: el request quedaría colgado y, desde Node 15, una promesa rechazada sin manejar (*unhandled rejection*) **termina el proceso**, es decir, se cae el servidor entero. Detalle en [07-errores.md](07-errores.md#express-5-atrapa-los-errores-de-las-funciones-async).
- `${nombreEntidad}`: por eso la fábrica recibe el nombre: el mensaje dice "Producto con id 99 no encontrado" o "Talle con id 99 no encontrado" según quién lo use.

`actualizar` es igual, pero llama a `modelo.actualizarPorId(id, req.body)`.

```js
// src/utils/crud.controller.js:36-49
async function eliminar(req, res) {
  const id = Number(req.params.id);
  const item = await modelo.eliminarPorId(id);

  if (!item) {
    throw new ApiError(404, `${nombreEntidad} con id ${id} no encontrado`);
  }

  const mensaje = campoNombre
    ? `${item[campoNombre]} eliminado correctamente`
    : `${nombreEntidad} #${id} eliminado correctamente`;

  res.json({ mensaje });
}
```

- El modelo devuelve **el registro que borró** (lo lee antes de borrarlo), así se puede armar un mensaje como "Remeras eliminado correctamente".
- `item[campoNombre]`: acceso a una propiedad **con nombre variable**. Si `campoNombre` es `"nombre_visible"`, equivale a `item.nombre_visible`.
- `campoNombre ? ... : ...`: las variantes no tienen un nombre legible (su "nombre" es la combinación producto + color + talle), así que `variante.controller.js` no pasa `campoNombre` y el mensaje cae en "Variante #3 eliminado correctamente".
- `res.json({ mensaje })`: abreviatura de `{ mensaje: mensaje }`.

## `usuario.controller.js`: validación a mano

Los usuarios no usan la fábrica porque tienen reglas propias: la contraseña se hashea, el email no se puede repetir y la respuesta **nunca** debe incluir `password_hash`.

### `crear` (registro, `POST /api/auth/register`)

```js
// src/controllers/usuario.controller.js:4-6
function bodyEsObjeto(body) {
  return body !== null && typeof body === "object" && !Array.isArray(body);
}
```

- **Por qué hace falta**: `req.body` puede ser `undefined` (el cliente no mandó `Content-Type: application/json`), un array (`[1,2]`) o incluso `null`, y todos pasan el parser de Express. Si después se hiciera `const { email } = req.body` con `undefined`, explotaría con un `TypeError` → 500. Con este chequeo el cliente recibe un 400 con un mensaje claro.
- `typeof null === "object"` en JavaScript (es un error histórico del lenguaje), por eso hay que excluir `null` explícitamente. Y los arrays también son `"object"`, por eso el `Array.isArray`.

```js
// src/controllers/usuario.controller.js:22-31
if (
  typeof nombre !== "string" ||
  typeof apellido !== "string" ||
  typeof email !== "string" ||
  typeof password !== "string"
) {
  return res.status(400).json({
    error: "Nombre, apellido, email y contraseña deben ser texto",
  });
}
```

- **Validar el tipo antes de usar métodos de texto**: más abajo se llama a `email.trim()`. Si `email` fuera un número (`{"email": 123}`), `.trim()` no existe y explotaría con 500. Validar primero convierte un 500 confuso en un 400 explicativo.
- `return res.status(400).json(...)`: el `return` es clave. Corta la función ahí. **Sin el `return`**, el código seguiría ejecutándose e intentaría responder de nuevo más abajo → error `Cannot set headers after they are sent`.

```js
// src/controllers/usuario.controller.js:43-45
const nombreNormalizado = nombre.trim();
const apellidoNormalizado = apellido.trim();
const emailNormalizado = email.trim().toLowerCase();
```

- **Normalizar**: `trim()` saca espacios al principio y al final (`"  Ana "` → `"Ana"`); `toLowerCase()` pasa el email a minúsculas. **Por qué el email en minúsculas**: `Ana@Mail.com` y `ana@mail.com` son la misma casilla. Si se guardaran tal cual, el login tendría que acordarse de cómo lo escribió el usuario al registrarse. El login hace la misma normalización antes de buscar.
- Después vienen los límites de largo (100 para nombre y apellido, 255 para email, 30 para teléfono): **coinciden con los `VARCHAR(...)` de la tabla `usuarios`** en [`sql/schema.sql`](../../Navaja-Style/backend/sql/schema.sql). Validarlos acá permite responder 400 con un mensaje específico en vez de dejar que MySQL rechace el dato.
- Contraseña de 6 a 128 caracteres: el mínimo es una regla de seguridad básica; el máximo evita que alguien mande un texto gigante para que el cálculo del hash consuma CPU.
- `emailEsValido` usa una **expresión regular** (un patrón de texto): `^[^\s@]+@[^\s@]+\.[^\s@]+$` significa "algo sin espacios ni @, una @, algo, un punto, algo". Es una validación de formato aproximada, no garantiza que la casilla exista.

```js
// src/controllers/usuario.controller.js:94-98 y 121-124
const usuarioExistente = await usuarioModel.buscarPorEmail(emailNormalizado);

if (usuarioExistente) {
  return res.status(409).json({ error: "El email ya está registrado" });
}
// ...
} catch (error) {
  if (error.code === "ER_DUP_ENTRY") {
    return res.status(409).json({ error: "El email ya está registrado" });
  }
  next(error);
}
```

- **Por qué se chequea dos veces**: el `buscarPorEmail` previo cubre el caso normal. Pero si llegan **dos registros con el mismo email casi al mismo tiempo**, los dos pueden pasar el chequeo antes de que cualquiera inserte. Ahí actúa la restricción `UNIQUE KEY uq_usuarios_email` de la base: el segundo `INSERT` falla con el código de MySQL `ER_DUP_ENTRY`, y el `catch` lo traduce al mismo 409 (commit `63982fe`). Esta situación se llama **condición de carrera** (*race condition*). **Si solo existiera el primer chequeo**, en ese caso el cliente recibiría un 500.
- **409 Conflict**: el dato es válido, pero choca con algo que ya existe.
- `const passwordHash = await generarHash(password)`: la contraseña nunca se guarda tal cual. Ver [08-autenticacion.md](08-autenticacion.md#hash-de-contraseñas-passwordjs).
- La respuesta arma el objeto `usuario` **campo por campo** y deja afuera `password_hash`. Además pone `rol: "cliente"` a mano: es el valor por defecto de la columna en la base, pero el controller no vuelve a leer el registro, así que lo repite.

### `obtenerMe` y `actualizarMe` (`/api/usuarios/me`)

- Usan `req.usuario.id`, que dejó el middleware [`verificarAutenticacion`](08-autenticacion.md#el-middleware-verificarautenticacion). El id **no lo manda el cliente**: sale del token firmado, así nadie puede consultar o editar el perfil de otro.
- `actualizarMe` hace una **actualización parcial**: solo valida y cambia los campos que vinieron (`if (nombre !== undefined)`). Va armando un objeto `campos` con lo permitido. Si no vino ningún campo válido, responde 400 en vez de ejecutar un `UPDATE` vacío.
- `email` no se puede cambiar desde acá (no está entre los campos aceptados), y si viene `password`, se hashea antes de guardarse como `password_hash`.

### Dos estilos de manejo de errores

Los controllers de usuario y auth usan `try { ... } catch (error) { next(error); }`, mientras que los CRUD hacen `throw` y dejan que Express 5 lo atrape. **Los dos funcionan**; en Express 5 el `try/catch` + `next(error)` es redundante para errores inesperados, pero es útil cuando hay que **interceptar** un error específico, como `ER_DUP_ENTRY`. Ver [07-errores.md](07-errores.md#dos-estilos-conviviendo).

## `auth.controller.js`

Solo contiene `login`. Se explica en [08-autenticacion.md](08-autenticacion.md#login-authcontrollerjs).

## Regla vs. convención

| Aspecto | ¿Regla o convención? | Por qué |
|---|---|---|
| `throw` en `async` sin `try/catch` | Funciona por Express 5 | En Express 4 tiraría abajo el servidor. |
| `return res...` al responder antes de tiempo | Regla práctica | Si no, se intenta responder dos veces. |
| `201` al crear, `409` al duplicar | Convención HTTP | Estándar que entienden todos los clientes. |
| Fábrica para el CRUD | Decisión de diseño | Un solo lugar para corregir. |
| No devolver `password_hash` | Regla de seguridad | El hash no debe salir del servidor. |

## Probalo

```bash
# Crear → 201
curl -i -X POST http://localhost:3000/api/talles \
  -H "Content-Type: application/json" -d '{"nombre": "XL", "orden": 5}'

# Registro con email inválido → 400
curl -i -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Ana","apellido":"Pérez","email":"no-es-email","password":"123456"}'
```

```http
HTTP/1.1 400 Bad Request

{"error":"El formato del email no es válido"}
```

## Observaciones

- **Los controllers CRUD no validan nada.** Toda la validación queda en manos de MySQL (`NOT NULL`, `UNIQUE`, `CHECK`, `FOREIGN KEY`). Cuando la base rechaza un dato, el error no es un `ApiError`, así que el cliente recibe `500 "Error interno del servidor"` en vez de un 400/409 que explique qué mandó mal. Ejemplos: categoría con nombre repetido, producto con `categoria_id` inexistente, precio negativo, borrar una categoría que tiene productos.
- **`POST` sin `Content-Type: application/json` en el CRUD → 500.** `req.body` es `undefined` y `crearModeloCRUD` hace `datos[campo]` sobre `undefined` → `TypeError`. Los controllers de usuario lo resuelven con `bodyEsObjeto`; los CRUD no.
- **Id no numérico → 500.** `GET /api/productos/abc` hace `Number("abc")` → `NaN`, y la consulta queda `WHERE id = NaN` (verificado con el escapador de mysql2), que MySQL rechaza. Debería ser 400 o 404.
- **`PUT` sin campos válidos → 404 engañoso.** Si el body no trae ningún campo actualizable, el modelo devuelve `null` sin consultar, y el controller lo interpreta como "no existe": responde "Producto con id 1 no encontrado" aunque el producto exista.
- **`bodyEsObjeto` está duplicada** en `auth.controller.js` y `usuario.controller.js`. Podría vivir en `utils/`.
- **`actualizarMe`**: si el usuario fue borrado después de emitirse su token, `actualizarPorId` devuelve `undefined` y `usuario.id` explota → 500 (en `obtenerMe` sí se contempla con un 404).
