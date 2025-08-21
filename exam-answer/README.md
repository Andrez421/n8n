## AI Exam Answer (n8n)

Blueprint del workflow para analizar imágenes de cuestionarios, extraer texto (OCR), seleccionar respuestas con IA y enviar/guardar el listado final.

### Archivos
- `AI_Exam_Answer.json`: workflow listo para importar en n8n.
- `AI_Exam_Answer_Flow_MCP_n8n.md`: descripción funcional del flujo.

### Requisitos
- **Credenciales en n8n**:
  - Google Drive OAuth2
  - OpenAI API
  - Telegram Bot
  - Gmail OAuth2
- **Variables en n8n** (Settings → Variables):
  - `OCR_SPACE_API_KEY`: API key de OCR.Space para el nodo HTTP Request.

### Importación
1. En n8n, ve a Workflows → Import from File y selecciona `AI_Exam_Answer.json`.
2. Abre el workflow importado y asigna las credenciales en cada nodo que lo requiera.

### Configuración requerida en el workflow
- Google Drive Trigger: selecciona la carpeta (reemplaza `FOLDER_ID_PLACEHOLDER`).
- Telegram - Enviar: define `chatId` de destino.
- Gmail - Enviar: define `sendTo` (correo destino).
- Guardar a archivo: ajusta la ruta, por ejemplo `/data/ai-exam-answers.txt`.
- OCR Space (HTTP Request): header `apikey` con `{{$env.OCR_SPACE_API_KEY}}` o la clave directamente.

### Flujo de trabajo (resumen)
1. Google Drive Trigger detecta nuevas imágenes en una carpeta.
2. Descarga el archivo (Google Drive Download).
3. Convierte binario a base64 (Binary → Base64) y prepara payload (Prepare OCR Payload).
4. Llama a OCR.Space vía HTTP (OCR Space) y extrae texto (Extract OCR Text).
5. OpenAI analiza preguntas y genera el listado (OpenAI - Analizar preguntas).
6. Se formatea y consolida el listado (Formatear listado → Consolidar listados).
7. Se envía por Telegram y Gmail y se guarda a archivo (Telegram - Enviar, Gmail - Enviar, Guardar a archivo).

### Notas y consideraciones
- OCR: se usa OCR.Space. Verifica límites de uso y costos. Ajusta `language` si necesitas otro idioma.
- LLM: el prompt devuelve solo el listado `n. letra`. Ajusta `maxTokens` o el prompt según tus exámenes.
- Consolidación: el nodo Consolidar listados reúne respuestas de múltiples imágenes.
- Privacidad: evita subir contenido sensible a OCR/LLM externos si la política lo requiere.

### Solución de problemas
- Error al importar JSON: usa el archivo `AI_Exam_Answer.json` sin comentarios/markdown.
- Falta de permisos/credenciales: vincula credenciales en cada nodo correspondiente.
- Variable `OCR_SPACE_API_KEY` no definida: añade la variable en n8n o coloca la clave en el header del nodo HTTP.


