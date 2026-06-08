# 4. De RAG Simple a RAG Avanzado con .NET

En esta fase del curso, abandonamos el modelo lineal de "Ingesta -> Recuperación -> Generación" (conocido como **Naive RAG**) para adentrarnos en arquitecturas que resuelven los problemas de precisión y relevancia que surgen en entornos de producción reales. 

Un sistema RAG básico suele fallar cuando las consultas son ambiguas, cuando el conocimiento está fragmentado en múltiples documentos o cuando el modelo generativo recibe demasiado "ruido" en el contexto. El paso hacia el **RAG Avanzado** implica optimizar cada etapa del pipeline utilizando técnicas de pre-recuperación, recuperación híbrida y post-procesamiento.

### 4.1 Introducción: El salto cualitativo en arquitecturas RAG

El paso de un sistema experimental a uno empresarial requiere identificar dónde se rompe la cadena de valor de la información.

#### Desafíos de Recuperación (Retrieval)
*   **Baja Precisión:** Los fragmentos recuperados son semánticamente similares a la consulta pero no contienen la respuesta.
*   **Baja Recuperación (Recall):** No se encuentran todos los fragmentos necesarios porque la terminología del usuario no coincide exactamente con la del documento.
*   **Desactualización:** Dificultad para priorizar información reciente sobre documentos obsoletos.

#### Desafíos de Generación
*   **Alucinaciones:** El modelo genera una respuesta convincente pero no basada en el contexto recuperado.
*   **Falta de integración:** El LLM no sabe cómo conciliar información contradictoria proveniente de dos fragmentos distintos.
*   **Limitaciones de la ventana de contexto:** Enviar demasiados fragmentos "ruidosos" diluye la capacidad del modelo para encontrar la respuesta correcta.

#### Qué se puede hacer para solucionarlos
Para mitigar esto, implementaremos estrategias en tres niveles:
1.  **Pre-Retrieval:** Optimización de la consulta (Query Rewriting, Expansion).
2.  **Retrieval:** Mejora de la búsqueda mediante técnicas híbridas y gestión de granularidad (Parent Document).
3.  **Post-Retrieval:** Refinamiento de los resultados mediante re-clasificación (Re-ranking) antes de enviarlos al LLM.

---

### 4.3.1 Técnica del Documento Padre (Parent Document Retriever) en C#

Uno de los problemas más comunes en .NET al usar librerías como `UglyToad.PdfPig` es decidir el tamaño del *chunk*. Si el fragmento es muy pequeño, perdemos el contexto; si es muy grande, diluimos el *embedding*.

La técnica del **Parent Document Retriever** resuelve esto dividiendo el documento en:
1.  **Parent Chunks:** Fragmentos grandes (contexto completo).
2.  **Child Chunks:** Fragmentos pequeños (optimizados para búsqueda vectorial).

Cuando el sistema encuentra un *Child Chunk* relevante, no envía ese fragmento al LLM, sino que recupera su *Parent Chunk* asociado.

#### Implementación conceptual en C#

A continuación, se muestra cómo estructuraríamos esta lógica usando un almacén de vectores y un almacén de metadatos (Key-Value) en .NET:

```csharp
public class ParentDocumentRetriever
{
    private readonly IVectorStore _vectorStore; // Ej: Qdrant o Azure AI Search
    private readonly IDocumentStore _parentStore; // Ej: Redis o SQL Server

    public async Task<string> GetContextAsync(string userQuery)
    {
        // 1. Convertir consulta a vector (Embedding)
        var queryVector = await _embeddingService.GenerateVectorAsync(userQuery);

        // 2. Buscar en los fragmentos HIJOS (Child Chunks)
        var searchResults = await _vectorStore.SearchAsync(queryVector, limit: 3);

        var finalContexts = new List<string>();

        foreach (var hit in searchResults)
        {
            // 3. Extraer el ID del documento PADRE desde los metadatos del hijo
            string parentId = hit.Metadata["parent_id"].ToString();

            // 4. Recuperar el fragmento completo (más grande) para el LLM
            var parentContent = await _parentStore.GetAsync(parentId);
            finalContexts.Add(parentContent);
        }

        return string.Join("\n---\n", finalContexts.Distinct());
    }
}
```

---

### 4.5 Recuperación Híbrida de Documentos

No todos los problemas se resuelven con vectores (distancia del coseno). A veces, una búsqueda por palabra clave exacta (BM25) es más efectiva, especialmente para buscar números de parte, nombres propios técnicos o términos legales específicos en los PDFs.

En .NET, al utilizar **Azure AI Search**, podemos implementar consultas híbridas que combinan lo mejor de ambos mundos.

#### Ejemplo de Consulta Híbrida con Azure SDK

```csharp
var searchOptions = new SearchOptions
{
    // Búsqueda vectorial
    VectorSearch = new()
    {
        Queries = { new VectorizedQuery(queryVector) { KNearestNeighborsCount = 3, Fields = { "vectorField" } } }
    },
    // Búsqueda semántica (Re-ranking integrado)
    SemanticSearch = new()
    {
        SemanticConfigurationName = "my-semantic-config",
        QueryCaption = new(QueryCaptionType.Extractive)
    },
    QueryType = SearchQueryType.Semantic // Habilita el re-ranker de Azure
};

// Ejecución sobre el índice de documentos
Response<SearchResults<PdfDocument>> response = await _searchClient.SearchAsync<PdfDocument>(
    searchText: "Normativa ISO 27001 sección 4", 
    options: searchOptions
);
```

---

### 4.10 RAG Basado en Agentes con Semantic Kernel

El enfoque avanzado culmina con la transformación del pipeline estático en uno **agéntico**. Usando **Semantic Kernel**, no solo recuperamos información, sino que permitimos que el LLM decida *qué* herramienta usar para obtener la respuesta.

Al exponer nuestras funciones de búsqueda como `KernelFunctions`, el modelo puede razonar: *"Para responder a esta pregunta sobre el PDF financiero, primero necesito usar la función de búsqueda de tablas y luego la de resumen de texto"*.

```csharp
// Definición de una función nativa que el Agente puede invocar
public class PdfSearchPlugin
{
    [KernelFunction, Description("Busca información técnica detallada en el manual de ingeniería.")]
    public async Task<string> SearchEngineeringManual([Description("El término de búsqueda")] string query)
    {
        // Lógica de recuperación avanzada aquí
        return await _ragService.ExecuteAdvancedQueryAsync(query);
    }
}

// Registro en el Kernel
var kernel = Kernel.CreateBuilder()
    .AddAzureOpenAIChatCompletion(...)
    .Build();

kernel.ImportPluginFromObject(new PdfSearchPlugin());

// El LLM decidirá automáticamente si invoca el plugin basándose en la intención del usuario
var result = await kernel.InvokePromptAsync("¿Cuál es el torque máximo según el manual?");
```

Este enfoque de **RAG Avanzado** asegura que la aplicación .NET sea capaz de manejar la complejidad, la ambigüedad y el volumen de datos de una manera que un simple script de integración no podría lograr. En las siguientes secciones profundizaremos en el ajuste del **Chunk Size** y la integración de **Re-rankers** externos.