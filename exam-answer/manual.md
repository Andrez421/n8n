## Manual de configuración (AI Exam Answer en n8n)

Este manual explica, paso a paso y de forma sencilla, cómo configurar el workflow `AI_Exam_Answer.json` en n8n: credenciales, OAuth2, variables de entorno y ajustes de nodos.

### Prerrequisitos
- n8n en ejecución con acceso a la UI (idealmente URL pública o túnel).
- Cuenta de Google con permisos sobre la carpeta de Drive (para leer) y Gmail (para enviar).
- Cuenta en OpenAI con API Key.
- Bot de Telegram y su token.
- API Key de OCR.Space.

---

### 1) Importar el workflow
1. En n8n: Workflows → Import from File → selecciona `AI_Exam_Answer.json`.
2. Abre el workflow importado, no lo actives aún.

---

### 2) Configurar OAuth2 en Google Cloud (Drive y Gmail)
Estos pasos se hacen una sola vez en Google Cloud Console.
1. Ve a Google Cloud Console y crea un Proyecto nuevo.
2. Habilita APIs:
   - Google Drive API
   - Gmail API
3. Configura Pantalla de consentimiento OAuth (Externa) y añade tu correo como probador (Testing) si la app no está verificada.
4. Crea Credenciales → ID de cliente de OAuth 2.0 → Tipo: Aplicación web.
5. URIs de redirección autorizados (añade según tu despliegue):
   - Producción: `https://TU-DOMINIO/rest/oauth2-credential/callback`
   - Local (http): `http://localhost:5678/rest/oauth2-credential/callback`
6. Guarda el Client ID y Client Secret.

Notas de scopes recomendados (los scopes exactos se seleccionarán en n8n):
- Drive lectura: `https://www.googleapis.com/auth/drive.readonly`
- Gmail envío: `https://www.googleapis.com/auth/gmail.send`

---

### 3) Crear credenciales en n8n

#### 3.1 Google Drive OAuth2
1. Credentials → New → busca “Google Drive OAuth2”.
2. Completa Client ID y Client Secret (de Google Cloud).
3. En scopes, utiliza lectura (drive.readonly) si se solicita.
4. Clic “Connect”, finaliza el login y guarda la credencial.

#### 3.2 Gmail OAuth2
1. Credentials → New → “Gmail OAuth2”.
2. Client ID y Secret (puedes reutilizar el mismo cliente OAuth). 
3. Asegúrate de permitir el scope `gmail.send`.
4. Clic “Connect”, acepta permisos y guarda.

#### 3.3 OpenAI API
1. Credentials → New → “OpenAI API”.
2. Pega tu API Key de OpenAI.
3. Guarda. (Opcional: define un modelo por defecto en el nodo al usarlo).

#### 3.4 Telegram
1. En Telegram, con @BotFather crea un bot y copia el token.
2. Credentials → New → “Telegram API”.
3. Pega el token. Guarda.
4. Obtén tu `chatId` con @get_id_bot o mediante `getUpdates` del Bot API.

---

### 4) Variables/entorno para OCR.Space
El workflow usa `{{$env.OCR_SPACE_API_KEY}}`. Debes definir una variable de entorno del sistema para n8n.

- Docker: al ejecutar el contenedor, agrega `-e OCR_SPACE_API_KEY=TU_API_KEY`.
  - Ejemplo: `docker run -e OCR_SPACE_API_KEY=xxxxx -p 5678:5678 n8nio/n8n`

- Windows (servicio/proceso):
  1. Abre PowerShell como administrador.
  2. Ejecuta: `setx OCR_SPACE_API_KEY "TU_API_KEY"`
  3. Reinicia n8n para que lea la variable.

Alternativa: puedes poner la API key directamente en el nodo HTTP (header `apikey`), pero es menos seguro.

---

### 5) Ajustes manuales en los nodos del workflow
Abre el workflow importado y ajusta:

1. Google Drive Trigger (`Google Drive Trigger`):
   - Selecciona la carpeta de Drive a monitorear.
   - Cómo obtener el ID: abre la carpeta en Drive y copia el valor tras `folders/` en la URL.

2. Google Drive Download: usa la misma credencial de Drive.

3. OCR Space (HTTP Request):
   - Verifica que el header `apikey` use `{{$env.OCR_SPACE_API_KEY}}` o pega tu clave.
   - `language`: `spa` (ajústalo si necesitas otro idioma).

4. OpenAI - Analizar preguntas:
   - Selecciona tu credencial de OpenAI.
   - Modelo recomendado: uno de chat actual (por ejemplo, GPT-4o, GPT-4o-mini o equivalente disponible).

5. Telegram - Enviar:
   - Selecciona credencial de Telegram.
   - Define `chatId` de destino.

6. Gmail - Enviar:
   - Selecciona credencial de Gmail.
   - Define `sendTo` (correo destinatario) y asunto si deseas cambiarlo.

7. Guardar a archivo (`Write Binary File`):
   - Ruta por defecto: `/data/ai-exam-answers.txt`.
   - Ajusta según tu entorno:
     - Docker: monta un volumen, por ejemplo `-v ./data:/data`.
     - Windows: puedes cambiar a una ruta local (ej. `C:\\Users\\TU_USUARIO\\ai-exam-answers.txt`) y actualizar el nodo.

---

### 6) Pruebas
1. Activa el workflow.
2. Sube una imagen de examen a la carpeta de Drive configurada.
3. Verifica en n8n la ejecución: 
   - Debe extraer el texto (OCR Space),
   - Analizar con OpenAI, 
   - Consolidar respuestas,
   - Enviar por Telegram/Gmail,
   - Guardar el archivo de resultados.

Si algo falla, revisa el “Execution Log” y el detalle de cada nodo.

---

### 7) Puesta en producción y buenas prácticas
- Usa scopes mínimos necesarios (principio de menor privilegio).
- Considera límites de uso/costo de OCR y OpenAI.
- Monta volúmenes persistentes para archivos.
- Controla errores: puedes añadir nodos de “Error Workflow” o “IF/Branch” para manejar fallos gracefully.

---

### 8) Solución de problemas
- Redirect URI mismatch (Google): confirma que el URI exacto sea `.../rest/oauth2-credential/callback` y coincide con el de n8n.
- 403/insufficientPermissions (Google): revisa scopes y que el usuario tenga acceso a la carpeta/Drive o Gmail.
- Telegram “chat not found”: valida `chatId` y que el bot haya iniciado conversación con el chat/usuario.
- OpenAI 401/insuficiente: verifica API key y cuota.
- OCR.Space 400/401: valida `apikey`, tamaño de imagen y formato; prueba `OCREngine=2` y `language` correctos.
- Error al escribir archivo: comprueba permisos de escritura y ruta mapeada (volumen en Docker o ruta válida en Windows).


