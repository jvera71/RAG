# 3.1 Preprocesamiento de documentos PDF en .NET

El preprocesamiento de documentos PDF es una de las fases más críticas y, a menudo, más subestimadas en la construcción de un pipeline RAG. La calidad de la respuesta generada por el LLM depende directamente de la calidad del texto extraído. En .NET, contamos con un ecosistema robusto para enfrentar los desafíos técnicos que presentan los archivos PDF, los cuales no fueron diseñados originalmente para ser "leídos" por máquinas, sino para ser visualizados de forma consistente.

### 3.1.1 Extracción de texto de PDFs (Librerías C#)

La elección de la librería adecuada depende del equilibrio necesario entre rendimiento, precisión y licenciamiento.

*   **UglyToad.PdfPig**: Es una de las mejores opciones de código abierto (Apache 2.0). Permite un acceso granular a las letras, palabras y bounding boxes, lo cual es vital para reconstruir el orden de lectura.
*   **iText7 (iTextSharp)**: Es el estándar de la industria en cuanto a capacidades, pero su licencia AGPL/Comercial puede ser una limitante. Es extremadamente potente para PDFs complejos o mal formados.
*   **PdfSharp / QuestPDF**: Aunque son excelentes para la **generación** de PDFs, su capacidad para la **extracción** de texto es limitada comparada con PdfPig.

Ejemplo básico de extracción usando **PdfPig**:

```csharp
using UglyToad.PdfPig;
using UglyToad.PdfPig.Content;

public string ExtractRawText(string filePath)
{
    using (PdfDocument document = PdfDocument.Open(filePath))
    {
        var textBuilder = new StringBuilder();
        foreach (Page page in document.GetPages())
        {
            textBuilder.AppendLine(page.Text);
        }
        return textBuilder.ToString();
    }
}
```

### 3.1.2 Manejo de múltiples páginas

Al extraer texto para RAG, es fundamental no tratar el PDF como un bloque monolítico. Debemos mantener la referencia de la página de origen en los metadatos. Esto permitirá que, en la fase de respuesta, el sistema pueda citar la fuente exacta (ej: *"Según la página 12 del manual..."*).

Es recomendable estructurar la extracción en una colección de objetos:

```csharp
public record DocumentPage(int PageNumber, string Content);

public List<DocumentPage> ExtractPages(string filePath)
{
    using var document = PdfDocument.Open(filePath);
    return document.GetPages()
        .Select(p => new DocumentPage(p.Number, p.Text))
        .ToList();
}
```

### 3.1.3 Limpieza y normalización de texto usando LINQ y expresiones regulares

El texto extraído suele contener "ruido" como caracteres de control, saltos de línea innecesarios o espacios dobles que consumen tokens innecesarios y confunden al modelo de embeddings.

```csharp
public string CleanText(string input)
{
    if (string.IsNullOrWhiteSpace(input)) return string.Empty;

    // 1. Reemplazar saltos de línea múltiples por uno solo
    string cleaned = Regex.Replace(input, @"[\r\n]+", " ");

    // 2. Eliminar caracteres no imprimibles/control
    cleaned = new string(cleaned.Where(c => !char.IsControl(c)).ToArray());

    // 3. Normalizar espacios en blanco
    cleaned = Regex.Replace(cleaned, @"\s+", " ").Trim();

    return cleaned;
}
```

### 3.1.4 Detección de idioma

Antes de proceder a la fragmentación (chunking), es vital identificar el idioma. Esto determina qué tokenizador o modelo de embeddings se utilizará posteriormente. En .NET, librerías como `LanguageDetection` o el uso de servicios cognitivos son comunes.

```csharp
// Ejemplo conceptual usando NLanguageDetection
var detector = new LanguageDetector();
detector.AddAllLanguages();
string language = detector.Detect(text); // Retorna "es", "en", etc.
```

### 3.1.5 Extracción avanzada de PDFs y OCR

Muchos PDFs en entornos corporativos son "PDFs escaneados" (imágenes) o tienen capas de texto corruptas. Aquí la extracción simple falla.

*   **Tesseract.NET**: Una implementación wrapper de Tesseract OCR. Es gratuita y corre localmente, pero requiere manejo manual de la segmentación de imagen.
*   **Azure AI Document Intelligence (antes Form Recognizer)**: Es la opción recomendada para RAG empresarial. No solo hace OCR, sino que entiende el **layout**. Devuelve el contenido en formato Markdown, lo cual es ideal para mantener la jerarquía de títulos y tablas.

```csharp
// Uso de Azure SDK para extracción estructurada
var client = new DocumentAnalysisClient(new Uri(endpoint), new AzureKeyCredential(key));
AnalyzeDocumentOperation operation = await client.AnalyzeDocumentAsync(WaitUntil.Completed, "prebuilt-layout", fileStream);

// El resultado incluye tablas, estilos y contenido organizado
foreach (var table in operation.Value.Tables) { /* ... */ }
```

### 3.1.6 Desafíos estructurales: Manejo de tablas, imágenes y encabezados

La extracción lineal de texto rompe la semántica de las tablas. Si un PDF tiene una tabla con columnas de "Producto" y "Precio", una extracción básica podría mezclar los datos: `Producto A Producto B 10$ 20$`.

Para mitigar esto en .NET:
1.  **Tablas**: Si se usa Azure Document Intelligence, se deben convertir las tablas a formato HTML o Markdown antes de enviarlas al pipeline de fragmentación.
2.  **Encabezados y Pies de página**: Deben ser detectados y usualmente eliminados para evitar que el texto repetitivo (como el título del documento en cada página) contamine la relevancia de la búsqueda vectorial.
3.  **Orden de lectura**: En PDFs de múltiples columnas, debemos asegurarnos de que la librería lea de arriba a abajo en la primera columna y luego pase a la segunda, en lugar de leer horizontalmente a través de ambas. PdfPig gestiona esto mediante la agrupación de palabras en `Letters` y `TextBlocks`.

Este preprocesamiento garantiza que los datos que alimentarán la siguiente etapa (el *Chunking*, que veremos en la sección 3.2) sean semánticamente coherentes y libres de ruido técnico.