# 3.4 Evaluación del Pipeline RAG Básico

Una vez que hemos implementado las fases de ingesta (procesamiento de PDFs y bases de datos vectoriales) y el componente de generación, es crítico medir el rendimiento del sistema. Evaluar un pipeline RAG es complejo porque no solo evaluamos la precisión del modelo de lenguaje (LLM), sino también la calidad de la recuperación de información.

En esta etapa inicial del curso, nos centraremos en los conceptos fundamentales de la evaluación y en cómo estructurar métricas básicas dentro de una aplicación .NET.

### 3.4.1 La necesidad de una evaluación estructurada

A diferencia de las aplicaciones tradicionales, los sistemas RAG tienen salidas no deterministas. Evaluar "a ojo" unas pocas consultas no es suficiente para garantizar la fiabilidad en un entorno empresarial. La evaluación del pipeline básico se divide en dos componentes principales:

1.  **Evaluación del Recuperador (Retrieval):** ¿Son relevantes para la pregunta los fragmentos extraídos de la base de datos vectorial?
2.  **Evaluación del Generador (Generation):** ¿La respuesta generada por el LLM es veraz respecto al contexto y responde realmente a la duda del usuario?

### 3.4.2 Métricas clave de rendimiento

Para cuantificar la calidad, utilizamos principalmente tres métricas (a menudo referidas como la "Tríada de RAG"):

#### A. Relevancia del Contexto (Context Relevance)
Mide si los fragmentos recuperados de nuestra base de datos (por ejemplo, vía *Azure AI Search* o *Qdrant*) contienen la información necesaria para responder a la consulta.
*   **Problema si es baja:** Estamos recuperando "ruido", lo que confunde al LLM o agota su ventana de contexto.

#### B. Fidelidad o Veracidad (Faithfulness/Groundedness)
Mide si la respuesta del LLM se basa **exclusivamente** en los fragmentos recuperados. 
*   **Problema si es baja:** El modelo está "alucinando", utilizando conocimiento externo al PDF o inventando datos que no están en el contexto proporcionado.

#### C. Relevancia de la Respuesta (Answer Relevance)
Mide qué tan pertinente es la respuesta con respecto a la pregunta original.
*   **Problema si es baja:** El modelo da una respuesta veraz basada en el contexto, pero no responde lo que el usuario preguntó.

### 3.4.3 Implementación de un Evaluador Básico en C#

En el ecosistema .NET, podemos implementar una lógica de evaluación utilizando el enfoque **"LLM-as-a-Judge"** (usar un LLM potente como GPT-4 para evaluar el desempeño de nuestro pipeline). 

A continuación, se muestra un ejemplo de cómo estructurar una clase de evaluación técnica utilizando el SDK de **Azure OpenAI** y un registro de resultados:

```csharp
public record RagEvaluationResult(
    float ContextRelevanceScore, 
    float FaithfulnessScore, 
    string Reasoning
);

public class RagEvaluator
{
    private readonly OpenAIClient _client;

    public RagEvaluator(OpenAIClient client)
    {
        _client = client;
    }

    public async Task<RagEvaluationResult> EvaluateBasicMetrics(
        string question, 
        string retrievedContext, 
        string generatedAnswer)
    {
        // Prompt diseñado para que el LLM actúe como juez técnico
        string evalPrompt = $"""
            Actúa como un evaluador experto de sistemas RAG. 
            Califica las siguientes métricas de 0.0 a 1.0:

            1. Relevancia del Contexto: ¿El contexto contiene la respuesta a la pregunta?
            2. Fidelidad: ¿La respuesta se basa estrictamente en el contexto?

            Datos:
            - Pregunta: {question}
            - Contexto: {retrievedContext}
            - Respuesta: {generatedAnswer}

            Responde ÚNICAMENTE en formato JSON:
            {{
                "contextRelevance": 0.0,
                "faithfulness": 0.0,
                "reasoning": "Breve explicación"
            }}
            """;

        var options = new ChatCompletionsOptions();
        options.Messages.Add(new ChatRequestUserMessage(evalPrompt));
        options.ResponseFormat = ChatCompletionsResponseFormat.JsonObject;

        var response = await _client.GetChatCompletionsAsync("gpt-4-turbo", options);
        var jsonContent = response.Value.Choices[0].Message.Content;

        // Deserialización del resultado (usando System.Text.Json)
        var result = JsonSerializer.Deserialize<EvaluationJsonDto>(jsonContent);
        
        return new RagEvaluationResult(
            result.ContextRelevance, 
            result.Faithfulness, 
            result.Reasoning
        );
    }
}
```

### 3.4.4 Preparación de un "Golden Dataset"

Para que esta evaluación tenga sentido, no podemos probarla con una sola pregunta. Es necesario crear un conjunto de datos de prueba (Golden Dataset) que consista en:
1.  Un listado de preguntas frecuentes sobre los PDFs.
2.  (Opcional) La "Respuesta Ideal" (Ground Truth) escrita por un experto humano.

**Estrategia en .NET:**
Podemos utilizar archivos `.json` o archivos de recursos `.resx` para almacenar estas parejas de Pregunta/Respuesta y ejecutar un bucle de evaluación sobre nuestro servicio RAG, promediando los resultados para obtener un KPI de calidad del sistema antes de pasar a la fase de **RAG Avanzado**.

### 3.4.5 Limitaciones de la evaluación básica

Es importante notar que este enfoque manual y programático es útil para la fase de desarrollo inicial. Sin embargo, presenta desafíos de escalabilidad y costo (consumo de tokens para evaluar). En capítulos posteriores (Sección 5), exploraremos cómo automatizar este proceso utilizando herramientas más robustas como **Azure AI Studio Prompt flow** y pruebas unitarias especializadas que se integran en el ciclo de CI/CD de .NET.