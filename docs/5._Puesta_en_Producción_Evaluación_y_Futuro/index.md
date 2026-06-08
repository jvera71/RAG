# 5. Puesta en Producción, Evaluación y Futuro

## 5. Puesta en Producción, Evaluación y Futuro

Tras haber diseñado la arquitectura, seleccionado los modelos de embeddings y optimizado las estrategias de recuperación avanzada, el paso crítico es la transición de un prototipo funcional (PoC) a un sistema de grado de producción. Esta fase se centra en la fiabilidad, la observabilidad y la capacidad de escala dentro del ecosistema .NET.

### 5.1 Marcos de evaluación automatizados para .NET

A diferencia del software tradicional, un sistema RAG es estocástico. Evaluarlo requiere métricas que validen tanto la **Recuperación** (¿es relevante el contexto?) como la **Generación** (¿es veraz la respuesta?).

#### Pruebas unitarias de LLM con xUnit
Podemos utilizar **xUnit** o **NUnit** para implementar "Evals" (evaluaciones). En lugar de comparar cadenas de texto exactas, usamos un "LLM Juez" para validar la semántica de la respuesta.

```csharp
[Fact]
public async Task RAG_Response_ShouldBeFaithfulToContext()
{
    // Arrange
    var query = "¿Cuál es la política de vacaciones de la empresa?";
    var result = await _ragService.AskQuestionAsync(query);

    // Act - Usamos un prompt de evaluación (LLM Judge)
    var evaluationPrompt = $@"
        Analiza si la siguiente 'Respuesta' se basa únicamente en el 'Contexto' proporcionado.
        Responde solo 'SÍ' o 'NO'.
        Contexto: {result.Context}
        Respuesta: {result.Answer}";

    var evaluation = await _kernel.InvokePromptAsync(evaluationPrompt);
    
    // Assert
    Assert.Contains("SÍ", evaluation.ToString().ToUpper());
}
```

#### Azure AI Studio Prompt Flow
Para pipelines complejos, **Azure AI Studio Prompt Flow** permite visualizar y medir el rendimiento del flujo RAG utilizando métricas predefinidas como *Groundedness* (fundamentación) y *Relevance*. En .NET, esto se integra consumiendo los flujos mediante SDKs de Azure para orquestar la evaluación a gran escala.

---

### 5.2 Despliegue en producción: APIs RAG con ASP.NET Core

El estándar para exponer servicios RAG en .NET son las **Minimal APIs** debido a su baja sobrecarga. Un aspecto crucial es el **Streaming**, que mejora la percepción de latencia para el usuario final.

```csharp
app.MapPost("/ask", async (RAGRequest request, Kernel kernel) =>
{
    var chatService = kernel.GetRequiredService<IChatCompletionService>();
    
    // Invocación con Streaming para una experiencia fluida
    var result = chatService.GetStreamingChatMessageContentsAsync(
        request.Prompt, 
        new OpenAIPromptExecutionSettings { MaxTokens = 500 }
    );

    return Results.Ok(result); // En Blazor o clientes JS, esto se consume como un stream de eventos
});
```

*   **gRPC:** Recomendado para comunicaciones internas entre microservicios (por ejemplo, entre un servicio de procesamiento de documentos y el motor de búsqueda).
*   **Blazor:** Permite construir interfaces interactivas que consumen `IAsyncEnumerable<T>` para mostrar la respuesta del LLM palabra por palabra.

---

### 5.3 Monitoreo y Observabilidad

En un sistema RAG, no basta con saber si el servidor está "vivo"; necesitamos trazar el camino desde la pregunta del usuario hasta el fragmento de texto recuperado de la base de datos vectorial.

#### Integración con OpenTelemetry y Application Insights
.NET tiene un soporte nativo excepcional para **OpenTelemetry**. El SDK de **Semantic Kernel** emite trazas de diagnóstico que permiten ver exactamente qué prompts se enviaron y cuántos tokens se consumieron.

```csharp
// Configuración en Program.cs
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddSource("Microsoft.SemanticKernel*") // Captura trazas del Kernel
        .AddAzureMonitorTraceExporter(o => o.ConnectionString = "...")
    );
```

**Métricas clave a monitorear:**
1.  **Latencia de Recuperación:** Tiempo invertido en la búsqueda vectorial.
2.  **Tokens por segundo:** Velocidad de generación del LLM.
3.  **Costo estimado:** Basado en el consumo de tokens de entrada/salida por usuario.

---

### 5.4 Arquitecturas escalables: Ingesta asíncrona

El procesamiento de PDFs (OCR, fragmentación, embeddings) es una tarea intensiva en CPU y tiempo. Nunca debe realizarse de forma síncrona dentro de una solicitud HTTP.

**Patrón de Ingesta Desacoplada:**
1.  **Web API:** Recibe el PDF, lo almacena en un Blob Storage y publica un mensaje en **Azure Service Bus** o **RabbitMQ**.
2.  **Worker Service (o Azure Function):** Se activa con el mensaje, descarga el PDF, realiza el *chunking* y lo inserta en la base de datos vectorial (ej. Qdrant o Azure AI Search).

```csharp
// Ejemplo de Azure Function con trigger de Service Bus para procesamiento RAG
public class IngestionWorker
{
    [FunctionName("ProcessDocument")]
    public async Task Run([ServiceBusTrigger("pdf-queue")] string message)
    {
        // 1. Extraer texto del PDF
        // 2. Fragmentar (Chunking)
        // 3. Generar Embeddings
        // 4. Upsert en Vector DB
    }
}
```

---

### 5.5 El futuro: RAG vs. LLMs de ventana de contexto ultra larga

Modelos como Gemini 1.5 Pro o GPT-4o ofrecen ventanas de contexto de hasta 1M o 2M de tokens, lo que plantea la pregunta: *¿Sigue siendo necesario RAG?*

1.  **Costo:** Pasar 1 millón de tokens en cada consulta es prohibitivo económicamente comparado con recuperar solo los 3 fragmentos más relevantes vía RAG.
2.  **Rendimiento:** La latencia de procesar ventanas de contexto masivas es significativamente mayor.
3.  **Actualización:** RAG permite actualizar el conocimiento del sistema instantáneamente añadiendo documentos a la base de datos vectorial, sin necesidad de re-procesar todo el contexto en cada llamada.

El futuro apunta a un enfoque **híbrido**: usar RAG para filtrar información masiva y ventanas de contexto amplias para permitir razonamientos complejos sobre los datos recuperados.

---

### 5.6 Conclusiones finales y próximos pasos

Implementar RAG en .NET va más allá de conectar un PDF con una API de OpenAI. Requiere una ingeniería robusta en la preparación de datos, una selección cuidadosa de la estrategia de recuperación y, sobre todo, un ciclo de evaluación constante.

**Próximos pasos recomendados:**
*   Explorar el uso de **.NET Aspire** para simplificar la orquestación de bases de datos vectoriales y servicios de IA en entornos locales.
*   Implementar **Cache Semántica** para reducir costos en preguntas recurrentes.
*   Investigar el uso de **Modelos Locales (Phi-3, Llama 3)** mediante **ONNX Runtime** para escenarios con restricciones estrictas de privacidad.