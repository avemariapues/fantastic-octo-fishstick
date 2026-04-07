# Desplegar en Render

## 1. Contenido del repositorio en GitHub

En la **raíz** del repo deben estar solo estos archivos (sin carpetas extra):

- `server.js`
- `package.json`
- `render.yaml`

No subas `node_modules`. No pongas `server.js` ni `package.json` dentro de una carpeta (por ejemplo `src/`); tienen que estar en la raíz.

## 2. Crear el servicio en Render

1. Entra a [render.com](https://render.com) e inicia sesión.
2. **New +** → **Web Service**.
3. Conecta tu cuenta de GitHub si no lo has hecho.
4. Elige el repositorio donde subiste `server.js`, `package.json` y `render.yaml`.
5. Render puede rellenar todo solo si detecta `render.yaml`. Si no:
   - **Name:** tricoserver1 (o el que quieras)
   - **Root Directory:** déjalo **vacío**
   - **Environment:** Node
   - **Build Command:** `npm install`
   - **Start Command:** `npm start`
6. **Advanced** (si aparece):
   - **Health Check Path:** `/health`
7. En **Environment Variables** agrega:
   - `TELEGRAM_TOKEN` = token de tu bot
   - `TELEGRAM_CHAT_ID` = ID del canal o chat
8. Clic en **Create Web Service**.

## 3. Si el deploy falla

- Revisa la pestaña **Logs** y copia el mensaje de error.
- Comprueba que en GitHub, en la raíz del repo, estén exactamente:
  - `server.js`
  - `package.json`
  - `render.yaml`
- En Render → **Settings** del servicio:
  - **Root Directory** debe estar vacío.
  - **Build Command:** `npm install`
  - **Start Command:** `npm start`

## 4. Probar que funciona

Cuando el deploy esté en verde:

- Abre: `https://TU-SERVICIO.onrender.com/`
- Deberías ver algo como: `{"ok":true,"service":"tricoserver1"}`
- Luego: `https://TU-SERVICIO.onrender.com/health`
- Deberías ver: `{"ok":true,"status":"healthy"}`

Usa esa misma URL base en `index.html` y `loading.html` donde pongas la URL del servidor.
