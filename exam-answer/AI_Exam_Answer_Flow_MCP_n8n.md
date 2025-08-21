# Prompt: AI Exam Answer Flow with MCP for n8n

Estás creando un agente en n8n llamado **"AI Exam Answer"** que recibe una serie de imágenes con cuestionarios (preguntas de distintas áreas como matemáticas, lectura crítica, ciencias sociales, ciencias naturales, etc.) desde una carpeta de Google Drive, usa un MCP con un modelo de IA (como ChatGPT o Gemini) para analizar las imágenes, detectar cada pregunta, sus opciones de respuesta y generar un listado final con el número de pregunta y la letra de la respuesta seleccionada (a, b, c, d).  
No se necesita explicación de las respuestas, solo el listado de resultados.

El flujo se activará cuando se suban nuevas imágenes a la carpeta en Google Drive.  
Cada imagen será descargada, procesada con OCR para extraer el texto y enviada al modelo de IA vía MCP para analizar y responder las preguntas.  
El resultado final será consolidado en un solo listado y enviado por correo electrónico y/o mensaje de Telegram.

El agente usa OpenAI o Gemini como LLM y tendrá acceso a las siguientes herramientas vía MCP:
- Google Drive tool (para detectar y descargar las imágenes nuevas)
- OCR tool (para convertir imagen a texto)
- AI MCP tool (para interpretar preguntas y generar respuestas)
- Gmail/Telegram tool (para enviar los resultados al usuario)

**Estructura del flujo:**
1. **Google Drive Trigger** → Detectar nuevas imágenes en carpeta específica.  
2. **Loop Node** → Iterar sobre cada imagen.  
3. **Google Drive Download** → Descargar la imagen.  
4. **OCR Tool** → Extraer texto de la imagen.  
5. **AI MCP Tool** → Procesar el texto, identificar preguntas y opciones, elegir respuesta.  
6. **Merge Node** → Unir las respuestas de todas las imágenes en un solo listado.  
7. **Send Node** (Gmail o Telegram) → Enviar listado final al usuario.

**Ejemplo de formato de respuesta esperado:**
```
1. c  
2. a  
3. b  
4. d  
...
```

Usa el formato JSON más reciente soportado por n8n para nodos, conexiones y estructura de datos.  
El blueprint debe estar listo para producción, sin nodos desconectados, sin elementos obsoletos y siguiendo buenas prácticas.  
Asegúrate de que el MCP se utilice al máximo en el análisis de texto y generación de respuestas.  
No detenerse hasta tener un JSON perfecto y funcional.  
Guardar el resultado final en un archivo.

