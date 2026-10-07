# Registro de decisiones técnicas

Cada entrada explica **una decisión que da forma al backend**: en qué contexto se tomó, qué se decidió, qué alternativas había y qué consecuencias trae. Si el motivo no quedó registrado en el código ni en los commits, figura como `❓ Pendiente` para que el equipo lo complete.

Las entradas nuevas se agregan al final con el número siguiente.

---

## D-01 · Express 5 como framework HTTP

- **Fecha:** 2026-09-09 (`38d9797`)
- **Contexto:** hacía falta un servidor HTTP para la API REST.
- **Decisión:** Express 5 (`^5.2.1`).
- **Alternativas:** Express 4 (la versión con más tutoriales y ejemplos en internet), Fastify, NestJS.
- **Consecuencias:**
  - Los errores lanzados en funciones `async` llegan solos al `errorHandler`, sin `try/catch` ([07-errores.md](07-errores.md#express-5-atrapa-los-errores-de-las-funciones-async)). Los controllers CRUD dependen de esto.
  - Ojo con ejemplos de internet escritos para Express 4: algunos detalles cambian (por ejemplo, `req.body` queda `undefined` si no hay parser, y la sintaxis de rutas comodín es distinta).
- ❓ Pendiente: ¿se eligió Express 5 a propósito por el manejo de errores async, o por ser la última versión?

## D-02 · MySQL con `mysql2`, SQL escrito a mano y sin ORM

- **Fecha:** 2026-09-09 (`dfde635`, `440186d`)
- **Contexto:** el esquema se diseñó originalmente en PostgreSQL 16 y se tradujo a MySQL (encabezado de [`sql/schema.sql`](../../Navaja-Style/backend/sql/schema.sql)).
- **Decisión:** MySQL + driver `mysql2/promise` con un pool de conexiones; consultas SQL escritas a mano con placeholders `?`.
- **Alternativas:** seguir con PostgreSQL (`pg`); usar un ORM (Sequelize, Prisma) que genera el SQL automáticamente.
- **Consecuencias:** control total y visibilidad del SQL; la seguridad depende de usar siempre `?` para los valores ([06](06-modelos-y-base-de-datos.md#crear-un-insert-dinámico-y-seguro)). Las migraciones de esquema se hacen a mano.
- ❓ Pendiente: ¿por qué se migró de PostgreSQL a MySQL? ¿Por qué SQL a mano en vez de un ORM (¿fin didáctico?)?

## D-03 · Arquitectura en capas (rutas → controllers → modelos) y nombres `entidad.tipo.js`

- **Fecha:** 2026-09-09 a 2026-09-16 (`01afeac`)
- **Contexto:** los labs iniciales tenían todo en un archivo; al crecer, cada cambio obligaba a tocar todo.
- **Decisión:** separar en `routes/`, `controllers/`, `models/`, `middlewares/`, `utils/`, `config/`, con archivos nombrados `entidad.tipo.js` (`producto.model.js`, `logger.middleware.js`).
- **Consecuencias:** cada capa tiene una sola responsabilidad ([01](01-recorrido-de-un-request.md#las-capas)); el nombre del archivo dice qué es aunque se abra fuera de su carpeta.

## D-04 · Fábricas genéricas para el CRUD

- **Fecha:** 2026-09-16 (`01afeac`, migraciones en `8a0fffe`, `33b38f6`)
- **Contexto:** categorías y productos tenían modelos y controllers casi idénticos; se venían talles, colores, variantes y métodos de pago.
- **Decisión:** `crearModeloCRUD` y `crearControllerCRUD` generan las cinco operaciones a partir de una configuración ([05](05-controllers.md#la-fábrica-crearcontrollercrud), [06](06-modelos-y-base-de-datos.md#la-fábrica-crearmodelocrud)).
- **Alternativas:** copiar el código por entidad; usar un ORM.
- **Consecuencias:** agregar una entidad son ~20 líneas; un bug se corrige en un solo lugar. A cambio, las entidades con reglas propias (validaciones, campos especiales) no encajan y necesitan código a mano, como `usuario`.

## D-05 · Errores centralizados con `ApiError` + `errorHandler`

- **Fecha:** 2026-09-09 (`0efe0dd`)
- **Decisión:** una clase `ApiError(statusCode, message)` para errores esperados y un único middleware que arma todas las respuestas de error con forma `{ "error": "..." }`. Los errores inesperados responden siempre 500 genérico.
- **Consecuencias:** formato de error uniforme para el frontend; no se filtran detalles internos ([07](07-errores.md)).

## D-06 · Autenticación con `crypto` nativo: scrypt + token HMAC propio

- **Fecha:** 2026-09-12 (`c1ef22f`, `b976100`)
- **Decisión:** contraseñas hasheadas con `crypto.scrypt` y salt aleatorio (`salt:hash`); tokens con formato `payload.firma` firmados con HMAC-SHA256 y `AUTH_SECRET`, válidos por 1 hora ([08](08-autenticacion.md)).
- **Alternativas:** `bcrypt`/`argon2` para contraseñas; `jsonwebtoken` (JWT estándar) para tokens; sesiones guardadas en el servidor.
- **Consecuencias:** cero dependencias extra; el equipo entiende cada paso. A cambio, hay que mantener el código de seguridad propio y no es interoperable con herramientas que esperan un JWT estándar. Los tokens no se pueden revocar antes de vencer.
- ❓ Pendiente: motivo de no usar `bcrypt`/`jsonwebtoken`.

## D-07 · Email duplicado: chequeo previo + captura de `ER_DUP_ENTRY`

- **Fecha:** 2026-09-28 (`63982fe`)
- **Decisión:** además de buscar el email antes de insertar, capturar el error de la restricción `UNIQUE` de MySQL y responder 409.
- **Consecuencias:** cubre la condición de carrera de dos registros simultáneos ([05](05-controllers.md#crear-registro-post-apiauthregister)).

## D-08 · Chequear `activo` después de verificar la contraseña

- **Fecha:** 2026-10-03 (`75ddc0a`)
- **Decisión:** en el login, responder "Usuario inactivo" (403) solo si la contraseña es correcta.
- **Consecuencias:** el estado de una cuenta no se revela a quien no conoce la contraseña ([08](08-autenticacion.md#login-authcontrollerjs)).

## D-09 · Separar usuarios de autenticación; perfil en `/usuarios/me`

- **Fecha:** 2026-10-06 (`76c02f0`)
- **Decisión:** `auth.controller` queda solo con `login`; el registro y el perfil pasan a `usuario.controller`. El perfil propio se expone en `/api/usuarios/me` (protegido), y el id se toma del token.
- **Alternativas:** `/usuarios/:id` con un chequeo de que el id coincida con el del token.
- **Consecuencias:** responsabilidades más claras; no hay forma de pedir el perfil de otro usuario. Quedaron desactualizados `README.md` y `api-test.http` (siguen usando `/api/auth/me`).
