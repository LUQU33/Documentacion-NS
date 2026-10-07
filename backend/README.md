# Documentación técnica del backend de NavajaStyle

Esta carpeta explica **cómo funciona el backend y por qué está hecho así**. Está escrita para alguien que sabe programar un poco pero no conoce en profundidad Express, Node.js, MySQL o HTTP: cada término técnico se define la primera vez que aparece y está en el [glosario](glosario.md).

Para instalar y levantar el proyecto, ver el [`README.md`](../../Navaja-Style/backend/README.md) del backend.

**Última revisión:** `76c02f0` (2026-10-06) · sin cambios sin commitear en `src/`.

## Por dónde empezar

1. [01 · Recorrido de un request](01-recorrido-de-un-request.md): la vista general. **Leelo primero.**
2. [02 · Arranque](02-arranque.md): `package.json`, `server.js`, `app.js` y el orden de los `app.use`.
3. [03 · Middlewares](03-middlewares.md): `next()`, logger, `notFound` y por qué `errorHandler` tiene 4 parámetros.
4. [04 · Rutas](04-rutas.md): cómo una URL llega a un controller.
5. [05 · Controllers](05-controllers.md): la fábrica CRUD y la validación de usuarios.
6. [06 · Modelos y base de datos](06-modelos-y-base-de-datos.md): pool, SQL seguro con `?` y el esquema.
7. [07 · Errores](07-errores.md): `ApiError`, qué ve el cliente y cómo Express 5 atrapa errores `async`.
8. [08 · Autenticación](08-autenticacion.md): hash de contraseñas, tokens firmados y rutas protegidas.

Referencia: [Glosario](glosario.md) · [Registro de decisiones](decisiones.md)

## Mapa: archivo → doc

| Archivo | Doc |
|---|---|
| `package.json`, `.env.example` | [02-arranque](02-arranque.md) |
| `src/server.js` | [02-arranque](02-arranque.md#serverjs-el-punto-de-entrada) |
| `src/app.js` | [02-arranque](02-arranque.md#appjs-el-orden-de-la-cadena) |
| `src/middlewares/logger.middleware.js` | [03-middlewares](03-middlewares.md#logger) |
| `src/middlewares/errorHandler.middleware.js` | [03-middlewares](03-middlewares.md#errorhandler-el-middleware-de-4-parámetros), [07-errores](07-errores.md) |
| `src/middlewares/auth.middleware.js` | [08-autenticacion](08-autenticacion.md#el-middleware-verificarautenticacion) |
| `src/routes/index.js` | [04-rutas](04-rutas.md#indexjs-juntar-todos-los-routers) |
| `src/routes/*.routes.js` | [04-rutas](04-rutas.md) |
| `src/controllers/{categoria,producto,talle,color,variante,metodoPago}.controller.js` | [05-controllers](05-controllers.md#la-fábrica-crearcontrollercrud) |
| `src/controllers/usuario.controller.js` | [05-controllers](05-controllers.md#usuariocontrollerjs-validación-a-mano), [08-autenticacion](08-autenticacion.md) |
| `src/controllers/auth.controller.js` | [08-autenticacion](08-autenticacion.md#login-authcontrollerjs) |
| `src/utils/crud.controller.js` | [05-controllers](05-controllers.md#cómo-funciona-la-fábrica-por-dentro) |
| `src/models/{categoria,producto,talle,color,variante,metodoPago}.model.js` | [06-modelos](06-modelos-y-base-de-datos.md#la-fábrica-crearmodelocrud) |
| `src/models/usuario.model.js` | [06-modelos](06-modelos-y-base-de-datos.md#usuariomodeljs-modelo-a-mano) |
| `src/utils/crud.model.js` | [06-modelos](06-modelos-y-base-de-datos.md#la-fábrica-crearmodelocrud) |
| `src/config/db.config.js` | [06-modelos](06-modelos-y-base-de-datos.md#la-conexión-dbconfigjs) |
| `sql/schema.sql` | [06-modelos](06-modelos-y-base-de-datos.md#el-esquema-sqlschemasql) |
| `src/utils/ApiError.js` | [07-errores](07-errores.md#apierror) |
| `src/utils/password.js` | [08-autenticacion](08-autenticacion.md#hash-de-contraseñas-passwordjs) |
| `src/utils/token.js` | [08-autenticacion](08-autenticacion.md#tokens-tokenjs) |

`lab/` (ignorado por git) contiene los ejercicios de práctica que dieron origen a esta estructura; sus apuntes están en [`../../apuntes/`](../../Navaja-Style/apuntes/).

## Endpoints actuales

| Método | URL | Protegido | Doc |
|---|---|---|---|
| `POST` `GET` | `/api/categorias` | No | [04](04-rutas.md#la-tabla-crud-que-se-repite) |
| `GET` `PUT` `DELETE` | `/api/categorias/:id` | No | [04](04-rutas.md#la-tabla-crud-que-se-repite) |
| ídem | `/api/productos`, `/api/talles`, `/api/colores`, `/api/variantes`, `/api/metodos-pago` | No | [04](04-rutas.md#la-tabla-crud-que-se-repite) |
| `POST` | `/api/auth/register` | No | [05](05-controllers.md#crear-registro-post-apiauthregister) |
| `POST` | `/api/auth/login` | No | [08](08-autenticacion.md#login-authcontrollerjs) |
| `GET` `PUT` | `/api/usuarios/me` | Sí (Bearer) | [08](08-autenticacion.md#el-middleware-verificarautenticacion) |

## Observaciones abiertas

Problemas detectados al documentar (no se modificó el código). El detalle está en la sección "Observaciones" de cada doc.

| # | Observación | Doc |
|---|---|---|
| 1 | Las rutas del catálogo (crear/editar/borrar) no requieren autenticación ni rol `admin`. | [04](04-rutas.md#observaciones) |
| 2 | `AUTH_SECRET` vacío permite fabricar tokens; ausente → login 500. No se valida al arrancar. | [08](08-autenticacion.md#observaciones) |
| 3 | Errores del cliente terminan en 500: JSON mal formado, violaciones de `UNIQUE`/`FOREIGN KEY`/`CHECK`, id no numérico, body ausente en el CRUD. | [07](07-errores.md#observaciones), [05](05-controllers.md#observaciones) |
| 4 | `PUT` sin campos válidos responde 404 "no encontrado" aunque el registro exista. | [05](05-controllers.md#observaciones) |
| 5 | Los tokens no se revocan al desactivar un usuario o cambiar la contraseña. | [08](08-autenticacion.md#observaciones) |
| 6 | `README.md` y `api-test.http` usan `/api/auth/me` (hoy `/api/usuarios/me`). | [04](04-rutas.md#observaciones) |
| 7 | `actualizarMe` da 500 si el usuario del token ya no existe; `bodyEsObjeto` duplicada. | [05](05-controllers.md#observaciones) |

## Cómo mantener esta documentación

Se actualiza con la skill de Claude Code `/documentar-backend` (definida en `Navaja-Style/backend/.claude/skills/documentar-backend/`; se ejecuta desde `Navaja-Style/backend/`):

- `/documentar-backend cambios`: documenta lo que cambió desde la **última revisión** indicada arriba.
- `/documentar-backend src/ruta/archivo.js` o un tema (`/documentar-backend pedidos`): documenta algo puntual.
