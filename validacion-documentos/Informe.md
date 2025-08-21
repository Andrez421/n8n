# Informe técnico — Automatización de Validación de Documentos de Matrícula (n8n)

> **Propósito:** Este documento explica, de forma precisa y ejecutable por IA (p. ej. Cursor), el **objetivo**, **arquitectura**, **flujo**, **datos**, **criterios de validación** y **puntos de configuración** del workflow de n8n que valida documentos de matrícula de aprendices del SENA, actualiza el estado en Google Sheets y notifica por correo las incidencias.

---

## 1) Objetivo del flujo

* **Validar automáticamente** que cada aprendiz subió **todos** los documentos requeridos y que **son legibles**.
* **Actualizar** el registro del aprendiz en Google Sheets con el **Estado** y **Observaciones**.
* **Notificar por correo (Gmail)** al aprendiz (con CC a coordinación) cuando falten documentos o estén ilegibles.
* **Escalar** a 150–200 casos/día con reintentos y espera para cargas completas.

---

## 2) Disparador y alcance

* **Trigger:** creación de **subcarpeta** en la carpeta raíz de Drive.
* **Carpeta raíz (ID):** `1s8k8d8mfbBlpNu_e2zPEY8P5123isQFV`.
* **Nombre de subcarpeta (clave de unión):** `Apellidos, Nombres`.
* **Delay:** 30 segundos tras detectar la subcarpeta (permite que terminen de subir archivos) - **ACTUALMENTE DESHABILITADO**.

---

## 3) Google Sheets (registro maestro)

* **Spreadsheet ID:** `1irpTXy8D2oysdMdzdaxf9Zk9SSEDtENSFxDAum6F1wM`
* **Hoja (tab):** `Registro de Matrículas`
* **Columnas relevantes (extracto):**
  `Nombres`, `Apellidos`, `Correo`, `Tipo de Programa`, `Fecha de Nacimiento`,
  `Enlace Acta`, `Enlace Identidad`, `Enlace Salud`, `Enlace Diploma`,
  `Enlace Foto`, `Enlace ICFES`, `Enlace Registro Civil`, `Enlace Trat. Datos`,
  `Estado `, `Observaciones`.

> **Unión con carpeta:** se usa `Apellidos` (tomados del nombre de la subcarpeta y buscados en la columna `Apellidos` de Sheets).

---

## 4) Conjunto de documentos y formatos

| Clave interna            | Columna en Sheets     | Formato esperado         | Keywords OCR                                                              | Requerimiento |
| ------------------------ | --------------------- | ------------------------ | ------------------------------------------------------------------------- | ------------- |
| ACTA\_DE\_COMPROMISO     | Enlace Acta           | PDF                      | ACTA, COMPROMISO, APRENDIZ, FICHA, FIRMA                                | Always        |
| DOCUMENTO\_DE\_IDENTIDAD | Enlace Identidad      | PDF                      | CEDULA, TARJETA DE IDENTIDAD, IDENTIDAD, REPUBLICA DE COLOMBIA, COLOMBIA | Always        |
| CERTIFICADO\_DE\_SALUD   | Enlace Salud          | PDF                      | ADRES, EPS, SALUD, AFILIADO                                             | Always        |
| CERTIFICADO\_DE\_ESTUDIO | Enlace Diploma        | PDF                      | DIPLOMA, BACHILLER, INSTITUCIÓN, ACTA DE GRADO                         | Always        |
| PRUEBAS\_ICFES           | Enlace ICFES          | PDF                      | ICFES, SABER, RESULTADOS, PUNTAJE                                       | Solo Tecnólogo |
| FOTOGRAFIA               | Enlace Foto           | **JPG/PNG**              | **Sin keywords** - validación por **tamaño ≥ 15 KB** + IA               | Always        |
| REGISTRO\_CIVIL          | Enlace Registro Civil | PDF                      | REGISTRO CIVIL, NACIMIENTO, NOTARÍA                                     | Solo Menores  |
| TRATAMIENTO\_DE\_DATOS   | Enlace Trat. Datos    | PDF                      | TRATAMIENTO DE DATOS, HÁBEAS DATA, LEY 1581                            | Solo Menores  |

**Idioma OCR:** `es` (español)
**Umbrales de legibilidad:** `≥ 20` palabras **y** `≥ 100` caracteres.
**Fotografía:** mínimo `15 KB` (15000 bytes) + validación IA con Gemini.
**Document AI:** Procesador `f7439ed85d11a95b` en proyecto `478244911399`.

> **Nota:** Los nombres de archivo en Drive pueden cambiar; la validación no depende del nombre, sino del **enlace** en Sheets + OCR. Las keywords se evalúan case-insensitive y sin acentos.

---

## 5) Arquitectura y nodos (alto nivel)

```mermaid
flowchart LR
  A[Google Drive Trigger\n(nueva subcarpeta)] --> B[Wait 30s\n(DESHABILITADO)]
  B --> C[Config Centralizada\n(parámetros centralizados)]
  C --> D[Parse Folder Name\n(extraer Apellidos, Nombres)]
  D --> E[Sheets - Lookup Student\n(buscar fila completa)]
  E --> F[Process Student Data\n(determinar docs requeridos)]
  F --> G[Get Drive Files\n(listar archivos en carpeta)]
  G --> H[Validate Documents\n(mapear docs vs archivos)]
  H --> I[Item Lists - Split by Document\n(procesar 1 doc por vez)]
  I --> J{IF - Is PDF\n(¿tipo PDF?)}
  I --> K{IF - Is Photo\n(¿tipo imagen?)}
  J --> L[Download PDF\n(descargar archivo)]
  K --> M[Download Photo\n(descargar imagen)]
  L --> N[Build DocAI Request\n(preparar para Document AI)]
  M --> O[Resize Photo\n(redimensionar para IA)]
  N --> P[Document AI - Process PDF\n(OCR asíncrono)]
  O --> Q[Validate Photo with AI\n(Gemini)]
  P --> R[Parse OCR Results\n(evaluar legibilidad + keywords)]
  Q --> S[Parse Photo Results\n(validar criterios IA)]
  R --> T[Merge All Results\n(combinar resultados)]
  S --> T
  T --> U[Generate Final Report\n(consolidar todos los resultados)]
  U --> V[Update Sheets\n(actualizar Estado y Observaciones)]
  V --> W{IF - Send Email\n(¿hay problemas?)}
  W -->|TRUE| X[Send Email Notification\n(email con lista de problemas)]
```

---

## 6) Especificaciones clave para Cursor (estructuras y lógica)

### 6.1 CONFIG (objeto central de parámetros)

**Implementación actual (nodo Function):**

```javascript
const config = {
  sheets: {
    id: '1irpTXy8D2oysdMdzdaxf9Zk9SSEDtENSFxDAum6F1wM',
    tab: 'Registro de Matrículas'
  },
  
  // Documentos con lógica de requerimiento simplificada
  documents: [
    {
      key: 'ACTA_DE_COMPROMISO',
      sheetColumn: 'Enlace Acta',
      required: 'always',
      mimeTypes: ['application/pdf'],
      keywords: ['ACTA', 'COMPROMISO', 'APRENDIZ', 'FICHA', 'FIRMA'],
      needsSignature: true
    },
    {
      key: 'DOCUMENTO_DE_IDENTIDAD', 
      sheetColumn: 'Enlace Identidad',
      required: 'always',
      mimeTypes: ['application/pdf'],
      keywords: ['CEDULA', 'TARJETA DE IDENTIDAD', 'IDENTIDAD', 'REPUBLICA DE COLOMBIA', 'COLOMBIA'],
      needsName: true
    },
    {
      key: 'CERTIFICADO_DE_SALUD',
      sheetColumn: 'Enlace Salud', 
      required: 'always',
      mimeTypes: ['application/pdf'],
      keywords: ['ADRES', 'EPS', 'SALUD', 'AFILIADO'],
      needsName: true
    },
    {
      key: 'CERTIFICADO_DE_ESTUDIO',
      sheetColumn: 'Enlace Diploma',
      required: 'always', 
      mimeTypes: ['application/pdf'],
      keywords: ['DIPLOMA', 'BACHILLER', 'INSTITUCIÓN', 'ACTA DE GRADO']
    },
    {
      key: 'PRUEBAS_ICFES',
      sheetColumn: 'Enlace ICFES',
      required: 'tecnologo',
      mimeTypes: ['application/pdf'], 
      keywords: ['ICFES', 'SABER', 'RESULTADOS', 'PUNTAJE'],
      needsName: true
    },
    {
      key: 'FOTOGRAFIA',
      sheetColumn: 'Enlace Foto',
      required: 'always',
      mimeTypes: ['image/jpeg', 'image/png'],
      minSizeBytes: 15000,
      isPhoto: true
    },
    {
      key: 'REGISTRO_CIVIL', 
      sheetColumn: 'Enlace Registro Civil',
      required: 'minor',
      mimeTypes: ['application/pdf'],
      keywords: ['REGISTRO CIVIL', 'NACIMIENTO', 'NOTARÍA']
    },
    {
      key: 'TRATAMIENTO_DE_DATOS',
      sheetColumn: 'Enlace Trat. Datos',
      required: 'minor', 
      mimeTypes: ['application/pdf'],
      keywords: ['TRATAMIENTO DE DATOS', 'HÁBEAS DATA', 'LEY 1581'],
      needsSignature: true
    }
  ],
  
  validation: {
    minWords: 20,
    minChars: 100,
    documentAI: {
      projectId: '478244911399',
      location: 'us',
      processorId: 'f7439ed85d11a95b'
    }
  },
  
  email: {
    cc: 'abravop73@gmail.com',
    subject: 'SENA - Corrección de documentos de matrícula',
    template: `Estimado(a) {{nombres}} {{apellidos}},\n\nTras revisar los documentos que cargó para el proceso de matrícula, encontramos incidencias que requieren corrección:\n\n{{problemas}}\n\nPor favor, vuelva a cargar los documentos correspondientes mediante el mismo formulario. Le recomendamos verificar que:\n• El archivo sea el solicitado (tipo correcto).\n• El contenido sea legible (sin borrosidad) y completo.\n• El formato sea PDF (salvo la fotografía: JPG o PNG).\n\nAgradecemos realizar el reenvío a la mayor brevedad para no retrasar su matrícula.\n\nSaludos cordiales,\nEquipo de Matrículas SENA`
  }
};

return [{ json: { config } }];
```

### 6.2 Implementación real de nodos Function

**Parse Folder Name (implementación actual):**

```js
const folderName = $json.name || '';
const [apellidos, nombres] = folderName.split(',').map(s => (s || '').trim());

return [{
  json: {
    folderId: $json.id,
    folderName,
    apellidos: apellidos || '',
    nombres: nombres || ''
  }
}];
```

**Process Student Data (implementación actual):**

```js
// Buscar estudiante por apellidos y procesar datos
const apellidos = $('Parse Folder Name').first().json.apellidos;
const allRows = $items().map(item => item.json);

// Buscar estudiante
const student = allRows.find(row => {
  if (!row.Apellidos) return false;
  const rowApellidos = row.Apellidos.trim().toLowerCase();
  const searchApellidos = apellidos.trim().toLowerCase();
  return rowApellidos === searchApellidos || 
         rowApellidos.includes(searchApellidos) || 
         searchApellidos.includes(rowApellidos);
});

if (!student) {
  throw new Error(`No se encontró estudiante con apellidos: "${apellidos}"`);
}

// Determinar tipo de programa y edad
const tipoPrograma = (student['Tipo de Programa'] || '').toString().toUpperCase();
const isTecnologo = tipoPrograma.includes('TECNOLOGO');

// Calcular edad
function calcAge(dateStr) {
  if (!dateStr) return null;
  const date = new Date(dateStr);
  if (isNaN(date)) return null;
  const now = new Date();
  let age = now.getFullYear() - date.getFullYear();
  const m = now.getMonth() - date.getMonth();
  if (m < 0 || (m === 0 && now.getDate() < date.getDate())) age--;
  return age;
}

const age = calcAge(student['Fecha de Nacimiento']);
const isMinor = age !== null ? age < 18 : true; // Por defecto menor si no se puede calcular

// Obtener configuración y determinar documentos requeridos
const config = $('Config Centralizada').first().json.config;
const requiredDocs = config.documents.filter(doc => {
  if (doc.required === 'always') return true;
  if (doc.required === 'tecnologo') return isTecnologo;
  if (doc.required === 'minor') return isMinor;
  return false;
});

return [{
  json: {
    student,
    nombres: student.Nombres || '',
    apellidos: student.Apellidos || '',
    correo: student.Correo || '',
    tipoPrograma,
    isTecnologo,
    age,
    isMinor,
    requiredDocs,
    folderId: $('Parse Folder Name').first().json.folderId
  }
}];
```

**Validate Documents (implementación actual):**

```js
// Validar documentos requeridos vs disponibles
const studentData = $('Process Student Data').first().json;
const files = $items().map(item => item.json);
const { requiredDocs, nombres, apellidos, correo } = studentData;

// Función para extraer fileId de URL
function extractFileId(url) {
  if (!url) return '';
  const match1 = url.match(/\/d\/([a-zA-Z0-9_-]+)/);
  if (match1) return match1[1];
  const match2 = url.match(/[?&]id=([a-zA-Z0-9_-]+)/);
  if (match2) return match2[1];
  if (/^[a-zA-Z0-9_-]{20,}$/.test(url)) return url;
  return '';
}

// Normalizar texto
function normalize(text) {
  return (text || '').toString().trim()
    .normalize('NFD').replace(/[\u0300-\u036f]/g, '')
    .toUpperCase();
}

// Procesar cada documento requerido
const results = [];
const missing = [];

for (const doc of requiredDocs) {
  const link = studentData.student[doc.sheetColumn] || '';
  const fileId = extractFileId(link);
  
  // Buscar archivo
  let file = null;
  
  if (fileId) {
    file = files.find(f => f.id === fileId);
  }
  
  if (!file) {
    // Buscar por nombre esperado
    const expectedNames = {
      'ACTA_DE_COMPROMISO': 'ACTA DE COMPROMISO',
      'DOCUMENTO_DE_IDENTIDAD': 'DOCUMENTO DE IDENTIDAD', 
      'CERTIFICADO_DE_SALUD': 'CERTIFICADO DE SALUD',
      'CERTIFICADO_DE_ESTUDIO': 'CERTIFICADO DE ESTUDIO',
      'PRUEBAS_ICFES': 'PRUEBAS ICFES',
      'FOTOGRAFIA': 'FOTOGRAFIA',
      'REGISTRO_CIVIL': 'REGISTRO CIVIL',
      'TRATAMIENTO_DE_DATOS': 'TRATAMIENTO DE DATOS'
    };
    
    const expectedName = normalize(expectedNames[doc.key] || '');
    if (expectedName) {
      file = files.find(f => {
        const fileName = normalize(f.name);
        return fileName === expectedName || fileName.startsWith(expectedName);
      });
    }
    
    // Para fotos, buscar por extensión
    if (!file && doc.isPhoto) {
      file = files.find(f => /\.(jpe?g|png)$/i.test(f.name || ''));
    }
  }
  
  if (!file) {
    missing.push({
      docKey: doc.key,
      reason: `Documento faltante: ${doc.key}`
    });
  } else {
    results.push({
      docKey: doc.key,
      file,
      docConfig: doc,
      nombres,
      apellidos,
      correo
    });
  }
}

// Retornar documentos encontrados para validar
const output = results.map(item => ({ json: item }));

// Log para debugging
console.log(`Documentos encontrados: ${results.length}`);
console.log(`Documentos faltantes: ${missing.length}`);
if (missing.length > 0) {
  console.log('Faltantes:', missing.map(m => m.docKey).join(', '));
}

return output;
```

**Build DocAI Request (implementación actual):**

```js
// Construir request para Document AI - Procesar todos los items
function getBase64FromBinary(itemIndex = 0) {
  const items = $items();
  const currentItem = items[itemIndex];
  
  // Intento 1: Buscar en el binario del item actual
  if (currentItem.binary && typeof currentItem.binary === 'object') {
    const keys = Object.keys(currentItem.binary);
    for (const key of keys) {
      const binary = currentItem.binary[key];
      if (binary && binary.data) {
        console.log(`Item ${itemIndex}: Encontrado binario en binary['${key}']`);
        return binary.data;
      }
    }
  }
  
  // Intento 2: Buscar en todos los items de entrada
  const inputItems = $input.all();
  for (const item of inputItems) {
    if (item.binary && typeof item.binary === 'object') {
      const keys = Object.keys(item.binary);
      for (const key of keys) {
        const binary = item.binary[key];
        if (binary && binary.data) {
          console.log(`Item ${itemIndex}: Encontrado binario en input item binary['${key}']`);
          return binary.data;
        }
      }
    }
  }
  
  // Intento 3: Buscar específicamente en el nodo Download PDF
  try {
    const downloadNodes = $('Download PDF').all();
    if (downloadNodes[itemIndex] && downloadNodes[itemIndex].binary) {
      const keys = Object.keys(downloadNodes[itemIndex].binary);
      for (const key of keys) {
        const binary = downloadNodes[itemIndex].binary[key];
        if (binary && binary.data) {
          console.log(`Item ${itemIndex}: Encontrado binario en Download PDF binary['${key}']`);
          return binary.data;
        }
      }
    }
  } catch (e) {
    console.log(`Item ${itemIndex}: No se pudo acceder al nodo Download PDF:`, e.message);
  }
  
  throw new Error(`Item ${itemIndex}: No se encontró binario base64 del PDF para ${currentItem.json?.docKey || 'documento'}`);
}

// Procesar todos los items de entrada
const items = $items();
const results = [];

for (let i = 0; i < items.length; i++) {
  const item = items[i];
  try {
    const base64 = getBase64FromBinary(i);
    
    results.push({
      json: {
        request: {
          rawDocument: {
            content: base64,
            mimeType: 'application/pdf'
          }
        },
        docKey: item.json.docKey,
        docConfig: item.json.docConfig,
        nombres: item.json.nombres,
        apellidos: item.json.apellidos,
        correo: item.json.correo
      }
    });
    
    console.log(`Procesado exitosamente: ${item.json.docKey}`);
  } catch (error) {
    console.log(`Error procesando ${item.json.docKey}:`, error.message);
    // Opcional: agregar item con error o saltarlo
  }
}

console.log(`Procesados ${results.length} documentos de ${items.length} totales`);
return results;
```

**Parse OCR Results (implementación actual):**

```js
// Analizar resultados del OCR - Procesar todos los items
const config = $('Config Centralizada').first().json.config;
const buildDocAIItems = $('Build DocAI Request').all();
const currentItems = $items();

const results = [];

for (let i = 0; i < currentItems.length; i++) {
  const currentItem = currentItems[i];
  const docData = buildDocAIItems[i]?.json;
  
  if (!docData) {
    console.log(`No se encontraron datos de Build DocAI Request para item ${i}`);
    continue;
  }
  
  const { docKey, docConfig, nombres, apellidos } = docData;

  // Extraer texto del OCR
  let text = '';
  if (currentItem.json.document && typeof currentItem.json.document.text === 'string') {
    text = currentItem.json.document.text;
  }
  text = (text || '').replace(/\s+/g, ' ').trim();

  const words = text ? text.split(' ').length : 0;
  const chars = text.length;

  // Validaciones
  const issues = [];

  // 1. Legibilidad básica
  const isLegible = words >= config.validation.minWords && chars >= config.validation.minChars;
  if (!isLegible) {
    issues.push(`Documento ilegible (${words} palabras, ${chars} caracteres)`);
  }

  // 2. Palabras clave (normalizadas sin acentos)
  let hasKeywords = true;
  if (docConfig.keywords && docConfig.keywords.length > 0 && text) {
    hasKeywords = docConfig.keywords.some(keyword => 
      text.toUpperCase().normalize('NFD').replace(/[\u0300-\u036f]/g, '').includes(keyword.toUpperCase().normalize('NFD').replace(/[\u0300-\u036f]/g, ''))
    );
    if (!hasKeywords) {
      issues.push('Contenido no corresponde al documento esperado');
    }
  }

  // 3. Validación de nombre (para documentos específicos)
  let hasName = true;
  if (docConfig.needsName && text) {
    const nameTokens = (nombres + ' ' + apellidos).trim().split(/\s+/).filter(Boolean);
    hasName = nameTokens.some(token => 
      text.toUpperCase().normalize('NFD').replace(/[\u0300-\u036f]/g, '').includes(token.toUpperCase().normalize('NFD').replace(/[\u0300-\u036f]/g, ''))
    );
    if (!hasName) {
      issues.push('No se encontró el nombre del estudiante en el documento');
    }
  }

  // 4. Validación de firma (para actas y tratamiento de datos)
  let hasSignature = true;
  if (docConfig.needsSignature && text) {
    const upperText = text.toUpperCase();
    const firmaIndex = upperText.indexOf('FIRMA');
    
    if (firmaIndex === -1) {
      hasSignature = false;
      issues.push('No se detectó área de firma');
    } else {
      // Verificar que hay contenido después de "FIRMA"
      const afterFirma = upperText.slice(firmaIndex + 5, firmaIndex + 200);
      const hasContentAfterFirma = /[A-ZÁÉÍÓÚÑ]{3,}/.test(afterFirma);
      
      if (!hasContentAfterFirma) {
        hasSignature = false;
        issues.push('No se detectó firma válida');
      }
    }
  }

  const isValid = issues.length === 0;

  results.push({
    json: {
      docKey,
      isValid,
      issues,
      details: {
        isLegible,
        hasKeywords,
        hasName,
        hasSignature,
        words,
        chars
      }
    }
  });
  
  console.log(`Procesado OCR para: ${docKey}, válido: ${isValid}`);
}

console.log(`Procesados ${results.length} documentos OCR`);
return results;
```

**Parse Photo Results (implementación actual):**

```js
// Analizar resultados de validación de foto
function calculateFileSize() {
  // Intento 1: Buscar en el binario del item actual
  if ($binary && typeof $binary === 'object') {
    const keys = Object.keys($binary);
    for (const key of keys) {
      const binary = $binary[key];
      if (binary && binary.data) {
        const base64 = binary.data;
        const padding = base64.endsWith('==') ? 2 : (base64.endsWith('=') ? 1 : 0);
        console.log(`Calculando tamaño desde $binary['${key}']`);
        return Math.floor((base64.length * 3) / 4) - padding;
      }
    }
  }
  
  // Intento 2: Buscar en todos los items de entrada
  const inputItems = $input.all();
  for (const item of inputItems) {
    if (item.binary && typeof item.binary === 'object') {
      const keys = Object.keys(item.binary);
      for (const key of keys) {
        const binary = item.binary[key];
        if (binary && binary.data) {
          const base64 = binary.data;
          const padding = base64.endsWith('==') ? 2 : (base64.endsWith('=') ? 1 : 0);
          console.log(`Calculando tamaño desde input item binary['${key}']`);
          return Math.floor((base64.length * 3) / 4) - padding;
        }
      }
    }
  }
  
  // Intento 3: Buscar en el nodo Download Photo
  try {
    const downloadNode = $('Download Photo').first();
    if (downloadNode && downloadNode.binary) {
      const keys = Object.keys(downloadNode.binary);
      for (const key of keys) {
        const binary = downloadNode.binary[key];
        if (binary && binary.data) {
          const base64 = binary.data;
          const padding = base64.endsWith('==') ? 2 : (base64.endsWith('=') ? 1 : 0);
          console.log(`Calculando tamaño desde Download Photo binary['${key}']`);
          return Math.floor((base64.length * 3) / 4) - padding;
        }
      }
    }
  } catch (e) {
    console.log('No se pudo acceder al nodo Download Photo:', e.message);
  }
  
  console.log('No se encontró imagen binaria para calcular tamaño');
  return 0;
}

// Obtener datos del contexto
const docData = $('Download Photo').first().json;
const docConfig = docData.docConfig;
const size = calculateFileSize();

// Parsear respuesta de Gemini (viene como texto)
let validation = { is_valid: false, description: 'Error parsing AI response', issues: ['Could not parse AI response'] };

try {
  let aiText = '';
  
  // La respuesta puede venir en diferentes formatos
  if (typeof $json === 'string') {
    aiText = $json;
  } else if ($json.text) {
    aiText = $json.text;
  } else if ($json.content) {
    aiText = $json.content;
  } else if ($json.output) {
    aiText = typeof $json.output === 'string' ? $json.output : JSON.stringify($json.output);
  } else {
    aiText = JSON.stringify($json);
  }
  
  console.log('AI Response text:', aiText);
  
  // Buscar JSON en la respuesta (puede estar envuelto en texto)
  const jsonMatch = aiText.match(/\{[\s\S]*\}/);
  if (jsonMatch) {
    validation = JSON.parse(jsonMatch[0]);
    console.log('Parsed validation:', validation);
  } else {
    console.log('No JSON found in AI response');
  }
} catch (e) {
  console.log('Error parsing AI response:', e.message);
  console.log('Raw AI response:', $json);
}

const issues = [];

// Validar tamaño mínimo
const minSize = docConfig.minSizeBytes || 15000;
if (size < minSize) {
  issues.push(`Imagen demasiado pequeña (${size} bytes, mínimo ${minSize})`);
}

// Validar resultado de IA
if (!validation.is_valid) {
  if (validation.issues && Array.isArray(validation.issues)) {
    issues.push(...validation.issues);
  } else {
    issues.push('Imagen no cumple criterios para foto de carnet');
  }
}

const isValid = issues.length === 0;

return [{
  json: {
    docKey: docData.docKey,
    isValid,
    issues,
    details: {
      size,
      minSize,
      aiValidation: validation
    }
  }
}];
```

**Generate Final Report (implementación actual):**

```js
// Generar reporte final consolidado
const validationResults = $items().map(item => item.json);

// Obtener datos del estudiante desde el contexto del flujo
let studentData = {};
try {
  studentData = $('Process Student Data').first().json;
} catch (e) {
  console.log('No se pudo obtener datos del estudiante:', e.message);
  studentData = { nombres: '', apellidos: '', correo: '' };
}

// Deduplicar resultados por docKey (tomar el último resultado para cada documento)
const uniqueResults = [];
const seenDocs = new Set();
for (let i = validationResults.length - 1; i >= 0; i--) {
  const result = validationResults[i];
  if (!seenDocs.has(result.docKey)) {
    uniqueResults.unshift(result);
    seenDocs.add(result.docKey);
  }
}

// Calcular documentos faltantes comparando requeridos vs encontrados
const requiredDocs = studentData.requiredDocs || [];
const foundDocs = uniqueResults.map(r => r.docKey);
const missing = requiredDocs
  .filter(doc => !foundDocs.includes(doc.key))
  .map(doc => ({ docKey: doc.key, reason: `Documento faltante: ${doc.key}` }));

// Procesar resultados únicos
const failed = uniqueResults.filter(result => !result.isValid);
const passed = uniqueResults.filter(result => result.isValid);

// Generar observaciones
const observations = [];

if (missing.length > 0) {
  const missingDocs = missing.map(m => m.docKey).join(', ');
  observations.push(`Documentos faltantes: ${missingDocs}`);
}

if (failed.length > 0) {
  observations.push('Documentos con problemas:');
  failed.forEach(result => {
    const issues = result.issues.join('; ');
    observations.push(`- ${result.docKey}: ${issues}`);
  });
}

// Determinar estado final
const isComplete = missing.length === 0 && failed.length === 0;
const status = isComplete ? 'Completo' : 'Pendiente corrección';
const observationsText = observations.length > 0 ? observations.join('\n') : 'Todos los documentos verificados correctamente';

// Preparar lista de problemas para el correo
const problemsForEmail = [];
if (missing.length > 0) {
  problemsForEmail.push(`• Documentos faltantes: ${missing.map(m => m.docKey).join(', ')}`);
}
failed.forEach(result => {
  problemsForEmail.push(`• ${result.docKey}: ${result.issues.join('; ')}`);
});

console.log(`Resumen: ${uniqueResults.length} documentos procesados, ${passed.length} válidos, ${failed.length} con problemas, ${missing.length} faltantes`);

return [{
  json: {
    status,
    observations: observationsText,
    isComplete,
    nombres: studentData.nombres || '',
    apellidos: studentData.apellidos || '',
    correo: studentData.correo || '',
    problems: problemsForEmail.join('\n'),
    summary: {
      total: uniqueResults.length + missing.length,
      passed: passed.length,
      failed: failed.length,
      missing: missing.length
    }
  }
}];
```

---

## 7) OCR con Google Document AI (implementación actual)

### 7.1 Configuración de nodos Document AI

**Para PDFs:**
1. **Nodo:** `Build DocAI Request` → prepara request con base64 del PDF
2. **Nodo:** `Document AI - Process PDF` → `https://us-documentai.googleapis.com/v1/projects/478244911399/locations/us/processors/f7439ed85d11a95b:process`
3. **Nodo:** `Parse OCR Results` → evalúa legibilidad + keywords + nombre + firma

**Para Imágenes (JPG/PNG):**
1. **Nodo:** `Resize Photo` → redimensiona a 1024x1024
2. **Nodo:** `Validate Photo with AI` → usa Gemini para validar criterios de foto de carnet
3. **Nodo:** `Parse Photo Results` → evalúa tamaño + resultado de IA

### 7.2 Configuración de credenciales

**Credenciales requeridas:**
- **Google Drive:** `googleDriveOAuth2Api` (ID: `H3fqwTFhSg1quRSR`)
- **Google Sheets:** `googleSheetsOAuth2Api` (ID: `XWBm2gEuEegQv4xE`) 
- **Google API (Document AI):** `googleOAuth2Api` (ID: `ujHKJNgyHmKnqGHa`)
- **Google Gemini:** `googlePalmApi` (ID: `kszLlTPJITHvdOAR`)
- **Gmail:** `gmailOAuth2` (ID: `CniNokNQx65Ut0pO`)

### 7.3 Configuración actual de Document AI Request

```javascript
// Build DocAI Request (Function Node)
const request = {
  "rawDocument": {
    "content": base64,
    "mimeType": "application/pdf"
  }
};

return [{
  json: {
    request,
    docKey: item.json.docKey,
    docConfig: item.json.docConfig,
    nombres: item.json.nombres,
    apellidos: item.json.apellidos,
    correo: item.json.correo
  }
}];
```

**Configuración del nodo Document AI - Process PDF:**
- **Method:** POST
- **URL:** `https://us-documentai.googleapis.com/v1/projects/478244911399/locations/us/processors/f7439ed85d11a95b:process`
- **Authentication:** predefinedCredentialType (googleOAuth2Api)
- **Content Type:** raw
- **Body:** `={{ JSON.stringify($json.request) }}`
- **Timeout:** 30000ms

### 7.4 Validación de fotos con Gemini

**Prompt para Gemini:**
```
Eres un asistente que valida si una imagen sirve como foto para un carnet estudiantil. Responde ÚNICAMENTE con un objeto JSON válido.

CRITERIOS MÍNIMOS:
1) Exactamente UNA persona humana
2) Fondo BLANCO y liso
3) Rostro de frente, ojos abiertos y visibles
4) En color, nítida, sin filtros
5) Encuadre tipo retrato (cabeza y hombros)
6) Iluminación uniforme

Responde SOLO con este formato JSON (sin texto adicional):
{
  "is_valid": true/false,
  "description": "descripción breve de la imagen",
  "issues": ["lista de problemas encontrados"]
}
```

---

## 8) Correo de notificación (Gmail) - Implementación actual

### 8.1 Configuración del nodo Gmail

**Nodo:** `Send Email Notification`
- **Para:** `={{ $('Generate Final Report').item.json.correo }}`
- **Asunto:** `SENA - Corrección de documentos de matrícula`
- **Tipo:** text
- **CC:** `abravop73@gmail.com`
- **Credencial:** `gmailOAuth2` (ID: `CniNokNQx65Ut0pO`)

### 8.2 Plantilla de mensaje (implementación actual)

```javascript
// Configuración actual en nodo Gmail
message: "=Estimado(a) {{ $('Generate Final Report').item.json.nombres }} {{ $('Generate Final Report').item.json.apellidos }},\n\nTras revisar los documentos que cargó para el proceso de matrícula, encontramos incidencias que requieren corrección:\n{{ $('Generate Final Report').item.json.problems }}\n\nPor favor, vuelva a cargar los documentos correspondientes mediante el mismo formulario. Le recomendamos verificar que:\n• El archivo sea el solicitado (tipo correcto).\n• El contenido sea legible (sin borrosidad) y completo.\n• El formato sea PDF (salvo la fotografía: JPG o PNG).\n\nAgradecemos realizar el reenvío a la mayor brevedad para no retrasar su matrícula.\n\nSaludos cordiales,\n\nEquipo de Matrículas SENA."
```

### 8.3 Lógica de envío

**Nodo:** `IF - Send Email`
- **Condición:** `={{ $('Generate Final Report').item.json.isComplete }}` equals `false`
- **Solo se envía email cuando hay problemas** (documentos faltantes o ilegibles)

---

## 9) Errores y resiliencia

* **Archivos faltantes:** se marcan en `Observaciones` y gatillan correo.
* **OCR fallido temporalmente:** Document AI maneja reintentos automáticamente.
* **Archivos corruptos o > tamaño límite:** registrar en `Observaciones`; pedir reenvío.
* **Concurrencia:** usar **Item Lists - Split by Document** (1 doc por lote) para controlar memoria y cuota API.
* **Debounce de cargas:** el **delay de 30s** reduce casos de validación antes de que terminen de subir todos los archivos.

---

## 10) Seguridad y privacidad

* **Principio de mínimo privilegio:** credenciales de Drive/Sheets/Gmail/Document AI con permisos estrictamente necesarios.
* **PII:** se almacena solo texto de OCR en memoria de ejecución; no persistir salvo en Observaciones (sin datos sensibles).
* **Document AI:** procesa documentos en Google Cloud con encriptación en tránsito y reposo.
* **Registros (opcional):** crear una segunda hoja "LOG" con docKey, palabras, caracteres, incidencias, timestamp (útil para auditoría).

---

## 11) Rendimiento y cuotas (150–200/día)

* **Document AI:** más eficiente que Vision API para documentos PDF complejos.
* **Gemini:** validación de fotos rápida y precisa.
* **Batching:** procesamiento paralelo con Item Lists para optimizar rendimiento.
* **Paralelismo:** manejo eficiente de múltiples documentos simultáneamente.

---

## 12) Pruebas y criterios de aceptación

**Casos de prueba mínimos:**

1. **Feliz:** todos los documentos correctos y legibles → `Estado=Completo`, sin correo.
2. **Faltante:** ausencia de `Enlace ICFES` → `Estado=Pendiente corrección`, Observaciones con "Faltantes: PRUEBAS_ICFES", correo enviado.
3. **Ilegible:** `Enlace Identidad` con imagen borrosa (OCR < umbral) → "Con incidencias: DOCUMENTO_DE_IDENTIDAD [Ilegible…]", correo.
4. **Foto pequeña:** `Enlace Foto` < 15 KB → incidencia de tamaño.
5. **Keywords:** PDF cargado que no contiene keywords del tipo → "Contenido no corresponde…".
6. **Validación IA:** foto que no cumple criterios de carnet → incidencias específicas de Gemini.

**Aceptación:**

* 100% de triggers de subcarpeta procesados.
* Estados y observaciones consistentes con la evidencia.
* Correos emitidos solo cuando hay problemas.
* Sin reprocesar la misma carpeta si no hay cambios (opcional: bandera "Procesado" en Sheets).

---

## 13) Extensiones futuras (opcionales)

* **Bandera "Procesado"** en Sheets para evitar reprocesar (o metadatos en Drive).
* **Validación de identidad:** buscar el **número de documento** del aprendiz dentro del OCR del "Documento de identidad".
* **Panel de control (Data Studio/Looker):** métricas de avance.
* **Soporte multi-idioma** en OCR si llegan documentos en otro idioma.
* **Validación de firmas:** análisis más avanzado de firmas con IA.

---

## 14) Requisitos de entorno y credenciales (resumen)

* n8n (self-hosted o cloud).
* **Google Drive**, **Google Sheets**, **Gmail**: credenciales OAuth/Service Account con permisos mínimos.
* **Google Document AI API** habilitada.
* **Google Gemini API** habilitada.
* Variables/credenciales seguras en n8n (no embed en código).

---

## 15) Glosario rápido

* **OCR:** Reconocimiento Óptico de Caracteres.
* **Document AI:** Servicio de Google Cloud para procesamiento inteligente de documentos.
* **Gemini:** Modelo de IA de Google para validación de imágenes.
* **Keywords:** Palabras clave esperadas por tipo de documento para validar correspondencia.
* **Estado:** `Completo` o `Pendiente corrección`.
* **Observaciones:** texto con faltantes e incidencias.
* **Item Lists:** Nodo de n8n para procesamiento paralelo de múltiples items.

---

### Fin del informe

Este informe refleja la arquitectura actual del workflow optimizado con Document AI y validación de fotos con Gemini, proporcionando una base sólida para el desarrollo y mantenimiento del sistema de validación automática de documentos.
