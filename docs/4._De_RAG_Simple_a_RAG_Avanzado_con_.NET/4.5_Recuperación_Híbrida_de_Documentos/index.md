# 4.5 Recuperación Híbrida de Documentos

En los capítulos anteriores, hemos explorado cómo la búsqueda vectorial permite capturar la semántica de las consultas del usuario, superando las limitaciones de la búsqueda por palabras clave tradicionales. Sin embargo, en entornos empresariales, la búsqueda vectorial pura presenta lagunas: puede fallar al buscar términos técnicos específicos, números de serie, acrónimos internos o nombres de productos muy similares.

Aquí es donde entra la **Recuperación Híbrida (Hybrid Retrieval)**. Este enfoque combina lo mejor de dos mundos: la precisión léxica de la búsqueda de texto completo y la comprensión contextual de la búsqueda vectorial.

### 4.5.1 ¿Por qué combinar BM25 y Vectores?

La recuperación híbrida utiliza dos algoritmos en paralelo para puntuar la relevancia de los fragmentos de documentos:

1.  **Búsqueda de Texto Completo (BM25):** Basada en la frecuencia de términos. Es imbatible cuando el usuario busca términos exactos (ej. "Error 0x8004210B" o "Modelo XYZ-2024").
2.  **Búsqueda Vectorial (HNSW/Cosine Similarity):** Basada en la cercanía en el espacio latente. Es ideal para consultas en lenguaje natural donde el usuario no usa las palabras exactas del documento (ej. "¿Cómo configurar el correo?" en lugar de "Protocolos de comunicación SMTP").

Al unirlos, el sistema compensa las debilidades de uno con las fortalezas del otro, incrementando significativamente el *Recall* (la capacidad de encontrar todos los documentos relevantes).

### 4.5.2 Reciprocal Rank Fusion (RRF)

Para combinar los resultados de ambos motores (que usan escalas de puntuación diferentes), se suele utilizar una técnica llamada **Reciprocal Rank Fusion (RRF)**. RRF no suma las puntuaciones directamente, sino que asigna una nueva puntuación basada en la *posición* (ranking) que ocupa el documento en cada una de las listas de resultados.

La fórmula simplificada es:
$score = \sum_{d \in r} \frac{1}{k + rank(d)}$

Donde $k$ es una constante (comúnmente 60) que suaviza el impacto de los documentos que aparecen en posiciones muy bajas.

### 4.5.3 Implementación con Azure AI Search en .NET

Azure AI Search es actualmente el servicio de referencia en el ecosistema .NET para implementar búsqueda híbrida de forma nativa. A continuación, vemos cómo realizar una consulta híbrida utilizando el SDK de `Azure.Search.Documents`.

#### Configuración de la consulta híbrida

En este ejemplo, asumimos que ya tienes un índice con campos vectorizados (`VectorContent`) y campos de texto indexados (`TextContent`).

```csharp
using Azure;
using Azure.Search.Documents;
using Azure.Search.Documents.Models;

public async Task<List<SearchResult>> ExecuteHybridSearch(string userQuery, ReadOnlyMemory<float> queryVector)
{
    var searchClient = new SearchClient(
        new Uri(Environment.GetEnvironmentVariable("AZURE_SEARCH_ENDPOINT")),
        "my-pdf-index",
        new AzureKeyCredential(Environment.GetEnvironmentVariable("AZURE_SEARCH_KEY"))
    );

    var searchOptions = new SearchOptions
    {
        // 1. Configuramos la parte de Texto Completo (BM25)
        Filter = "", 
        Size = 5,
        
        // 2. Configuramos la parte Vectorial
        VectorSearch = new()
        {
            Queries = { 
                new VectorizedQuery(queryVector) 
                { 
                    KNearestNeighborsCount = 5, 
                    Fields = { "VectorContent" } 
                } 
            }
        }
    };

    // La ejecución "Hybrid" ocurre automáticamente al enviar texto y vector
    SearchResults<DocumentModel> response = await searchClient.SearchAsync<DocumentModel>(
        userQuery, // Texto para BM25
        searchOptions
    );

    var results = new List<SearchResult>();
    await foreach (SearchResult<DocumentModel> result in response.GetResultsAsync())
    {
        results.Add(new SearchResult
        {
            Text = result.Document.TextContent,
            Score = result.Score, // Puntuación calculada mediante RRF
            Metadata = result.Document.SourcePage
        });
    }

    return results;
}
```

### 4.5.4 Ventajas competitivas en el Pipeline RAG

Implementar la recuperación híbrida en C# ofrece beneficios tangibles para la calidad de la generación:

*   **Robustez ante OOD (Out-of-Distribution):** Si el modelo de *embeddings* no fue entrenado con la jerga específica de tu empresa, la búsqueda de texto completo actuará como una red de seguridad.
*   **Manejo de consultas cortas:** Las consultas de una sola palabra suelen funcionar mejor con búsqueda léxica, mientras que las preguntas largas se benefician de los vectores.
*   **Menor sensibilidad al ruido:** Al requerir que un documento sea relevante en uno o ambos rankings, se filtran mejor los fragmentos que "parecen" semánticamente cercanos pero que no contienen los términos clave buscados.

Es importante notar que, aunque la búsqueda híbrida mejora drásticamente los resultados iniciales, el siguiente paso crítico para un sistema de grado producción es el **Re-ranking**, que veremos en la sección 4.9, donde aplicaremos modelos más costosos computacionalmente solo a este subconjunto de resultados filtrados.