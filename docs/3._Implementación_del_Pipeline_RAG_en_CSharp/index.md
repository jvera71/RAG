# 3. Implementación del Pipeline RAG en C#

La implementación de un pipeline de **Generación Aumentada por Recuperación (RAG)** en C# requiere orquestar la extracción de datos, su transformación y la interacción con modelos de lenguaje. En esta sección, profundizaremos en los componentes técnicos necesarios para construir un pipeline funcional utilizando el ecosistema de .NET.

## 3.1 Preprocesamiento de documentos PDF en .NET

El primer paso crítico es la ingesta de datos. Los documentos PDF no son archivos de texto plano; son contenedores de objetos gráficos, lo que hace que su extracción sea un desafío técnico.

### 3.1.1 Extracción de texto y manejo de páginas
Para extraer contenido en .NET, contamos con librerías robustas. Mientras que `iText7` es un estándar industrial, `UglyToad.PdfPig` destaca por ser de código abierto y permitir un acceso detallado a la estructura del documento.

```csharp
using UglyToad.PdfPig;
using UglyToad.PdfPig.Content;

public string ExtractTextFromPdf(string filePath)
{
    using var document = PdfDocument.Open(filePath);
    var fullText = new StringBuilder();

    foreach (var page in document.GetPages())
    {
        // La extracción simple puede perder el orden lógico (columnas)
        // PdfPig permite iterar por palabras para mantener coherencia
        fullText.AppendLine(page.Text);
    }
    return fullText.ToString();
}
```

### 3.1.3 Limpieza y normalización con LINQ
Una vez extraído el texto, es imperativo limpiarlo para evitar que el ruido (saltos de línea huérfanos, caracteres especiales de control) afecte la calidad de los *embeddings*.

```csharp
public string CleanText(string rawText)
{
    return string.Join(" ", rawText
        .Split(new[] { "\r\n", "\r", "\n" }, StringSplitOptions.RemoveEmptyEntries)
        .Select(line => line.Trim())
        .Where(line => line.Length > 0));
}
```

### 3.1.5 Extracción avanzada y OCR
Para PDFs escaneados o con estructuras complejas (tablas), se recomienda **Azure AI Document Intelligence**. A través de su SDK para .NET (`Azure.AI.FormRecognizer`), podemos obtener no solo texto, sino la estructura semántica (Layout).

---

## 3.2 Implementación del Pipeline de Ingesta de Datos

La ingesta no consiste solo en guardar texto, sino en fragmentarlo de forma que el modelo de recuperación pueda encontrar la información relevante eficientemente.

### 3.2.1 El impacto del Chunking
Si el fragmento de texto es muy pequeño, pierde contexto. Si es muy grande, introduce ruido y excede la ventana de contexto del LLM.

### 3.2.2 Estrategias de fragmentación en C#
Utilizar `Microsoft.ML.Tokenizers` es la forma más precisa de fragmentar, ya que los LLMs no "leen" palabras, sino tokens.

```csharp
using Microsoft.ML.Tokenizers;

public List<string> TokenBasedChunking(string text, int maxTokens)
{
    var tokenizer = TiktokenTokenizer.CreateForModel("gpt-4");
    var tokens = tokenizer.Encode(text);
    var chunks = new List<string>();

    for (int i = 0; i < tokens.Ids.Count; i += maxTokens)
    {
        var chunkIds = tokens.Ids.Skip(i).Take(maxTokens).ToList();
        chunks.Add(tokenizer.Decode(chunkIds));
    }
    return chunks;
}
```

### 3.2.3 Metadatos y Base de Datos Vectorial
Al insertar los fragmentos en una base de datos vectorial (como Qdrant o Azure AI Search), es vital adjuntar metadatos: `source_file`, `page_number` y `timestamp`. Esto permitirá filtrar búsquedas y citar fuentes en la respuesta final.

---

## 3.3 Implementación del Componente de Generación

Una vez recuperados los fragmentos relevantes mediante una búsqueda de similitud de vectores (proceso cubierto conceptualmente en el capítulo 2), debemos enviarlos al LLM junto con la consulta del usuario.

### 3.3.1 Ingeniería de Prompts con Semantic Kernel
**Semantic Kernel** permite definir plantillas de prompts desacopladas del código mediante archivos YAML o Handlebars. Esto facilita la iteración sin recompilar la aplicación.

```yaml
# Prompt Template (config.yaml)
name: RagPrompt
description: Responde preguntas basadas en el contexto proporcionado.
template: |
  Eres un asistente experto. Utiliza únicamente la información de los fragmentos proporcionados para responder.
  Si no conoces la respuesta, indica que no se encuentra en el documento.
  
  Contexto:
  {{$context}}
  
  Pregunta:
  {{$query}}
  
  Respuesta:
```

### 3.3.2 Integración con Azure OpenAI SDK
El componente de generación orquesta la llamada al modelo. En C#, utilizamos el SDK oficial para interactuar con los modelos de completado de chat.

```csharp
var kernel = Kernel.CreateBuilder()
    .AddAzureOpenAIChatCompletion("deployment-name", "endpoint", "api-key")
    .Build();

// Recuperación de fragmentos (Simulado)
string context = string.Join("\n", searchResults.Select(r => r.Text));

var response = await kernel.InvokePromptAsync(promptTemplate, new() {
    { "context", context },
    { "query", userInput }
});

Console.WriteLine(response.ToString());
```

---

## 3.4 Evaluación del Pipeline RAG Básico

Antes de pasar a técnicas avanzadas, debemos medir si nuestro pipeline base es efectivo. En esta etapa inicial, nos centramos en dos métricas fundamentales que pueden automatizarse mediante pruebas unitarias con LLMs (LLM-as-a-judge):

1.  **Relevancia del Contexto (Context Relevance):** Evalúa si los fragmentos recuperados de la base de datos vectorial realmente contienen la respuesta a la pregunta del usuario. Un fallo aquí indica problemas en el *chunking* o en el modelo de *embeddings*.
2.  **Fidelidad de la Respuesta (Faithfulness/Groundedness):** Mide si la respuesta generada por el LLM se basa estrictamente en el contexto recuperado o si el modelo está alucinando información externa.

En .NET, estas métricas se pueden implementar integrando el pipeline con **xUnit**, enviando la tríada (Pregunta, Contexto, Respuesta) a un modelo evaluador (como GPT-4o) para obtener un puntaje numérico de calidad.

Este pipeline básico de Ingesta -> Chunking -> Recuperación -> Generación constituye la base sobre la cual aplicaremos optimizaciones de recuperación híbrida y re-ranking en el siguiente capítulo.