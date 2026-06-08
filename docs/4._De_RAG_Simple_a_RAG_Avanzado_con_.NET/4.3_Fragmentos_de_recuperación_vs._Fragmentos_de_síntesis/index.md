# 4.3 Fragmentos de recuperación vs. Fragmentos de síntesis

En el diseño de sistemas RAG tradicionales, solemos utilizar el mismo fragmento de texto (*chunk*) tanto para la búsqueda en la base de datos vectorial como para alimentar el contexto del LLM. Sin embargo, este enfoque presenta un compromiso técnico difícil de resolver: fragmentos pequeños son excelentes para una recuperación precisa, pero fragmentos grandes son necesarios para una síntesis coherente.

En esta sección, exploraremos la estrategia de desacoplar los **Fragmentos de Recuperación** de los **Fragmentos de Síntesis**, una técnica avanzada para optimizar la relevancia y la calidad de las respuestas en .NET.

---

### 1. El Dilema del Tamaño del Fragmento

Para entender por qué necesitamos separar estos conceptos, debemos analizar los objetivos contrapuestos de cada fase del pipeline:

*   **Fragmentos de Recuperación (Retrieval Chunks):** Su objetivo es la **precisión semántica**. Un fragmento pequeño (p. ej., 100-300 tokens) suele contener una idea atómica. Esto facilita que el modelo de *embedding* capture el significado exacto de una oración o párrafo corto, minimizando el "ruido" de temas secundarios.
*   **Fragmentos de Síntesis (Synthesis Chunks):** Su objetivo es la **comprensión del contexto**. El LLM necesita entender las relaciones de causa y efecto, las referencias a conceptos anteriores y el flujo narrativo. Un fragmento demasiado pequeño puede omitir información crítica que rodea a la respuesta, provocando que el modelo genere respuestas incompletas o alucinaciones por falta de contexto.

### 2. Definición y Diferencias

| Característica | Fragmentos de Recuperación | Fragmentos de Síntesis |
| :--- | :--- | :--- |
| **Tamaño Típico** | Pequeño (128 - 256 tokens) | Grande (512 - 2048 tokens o más) |
| **Uso Principal** | Búsqueda por similitud de coseno / Producto punto | Ventana de contexto para el Prompt del LLM |
| **Almacenamiento** | Base de datos vectorial (Vectores + Metadata) | Base de datos documental o almacenamiento de objetos |
| **Meta** | Minimizar la distancia vectorial | Maximizar la coherencia y fidelidad |

### 3. Implementación del Desacoplamiento en .NET

La estrategia técnica consiste en crear una jerarquía donde múltiples "hijos" (fragmentos de recuperación) apuntan a un "padre" (fragmento de síntesis). 

En C#, podemos representar esta estructura mediante una relación de metadatos. Al realizar la búsqueda vectorial, recuperamos el fragmento pequeño, pero en lugar de enviar ese texto al LLM, consultamos su ID de referencia para obtener el fragmento mayor o el contexto circundante.

#### Ejemplo de Estructura de Datos

```csharp
public class DocumentChunk
{
    // ID único del fragmento pequeño (hijo)
    public string Id { get; set; } = Guid.NewGuid().ToString();

    // El vector generado para este fragmento pequeño
    public float[] Vector { get; set; }

    // El texto optimizado para la búsqueda
    public string RetrievalText { get; set; }

    // Referencia al ID del fragmento de síntesis (padre)
    public string ParentContextId { get; set; }

    // Metadatos adicionales (pág, documento_id, etc.)
    public Dictionary<string, object> Metadata { get; set; }
}

public class ParentContext
{
    public string Id { get; set; }
    public string FullText { get; set; } // El contexto extendido para el LLM
}
```

### 4. Flujo de Trabajo en el Pipeline

Cuando implementamos esta técnica en .NET (por ejemplo, usando `Microsoft.SemanticKernel` o el SDK de `Azure AI Search`), el flujo varía sensiblemente:

1.  **Ingesta:**
    *   Se toma un documento PDF y se divide en fragmentos grandes (Padres).
    *   Cada fragmento grande se subdivide en fragmentos pequeños (Hijos).
    *   Se generan *embeddings* **solo** para los fragmentos pequeños.
    *   Se almacenan los vectores de los hijos junto con el `ParentContextId`.

2.  **Consulta:**
    *   El usuario hace una pregunta.
    *   El sistema busca los $N$ fragmentos pequeños más similares vectorialmente.
    *   **Paso Crítico:** El sistema recolecta los `ParentContextId` únicos de esos resultados y recupera los fragmentos de síntesis correspondientes desde un almacén de datos (como Azure Blob Storage o una tabla de SQL).
    *   Se construye el prompt del LLM utilizando los fragmentos de síntesis.

### 5. Ventajas Técnicas en entornos .NET

*   **Eficiencia de Costos:** Los modelos de *embeddings* suelen tener límites de tokens. Procesar fragmentos pequeños es más rápido y predecible.
*   **Mejor Recall:** Al tener fragmentos más granulares, es más probable que el motor de búsqueda vectorial (como Qdrant o pgvector) encuentre la sección exacta del PDF que responde a la consulta, evitando que la relevancia se diluya en un fragmento de 2000 palabras.
*   **Contexto Enriquecido:** El LLM recibe suficiente información para no perder el hilo de la respuesta, lo que resulta fundamental en manuales técnicos o documentos legales donde una cláusula (hijo) depende del encabezado de la sección (padre).

Este enfoque sirve de base para la técnica de **Parent Document Retriever**, que profundizaremos en el siguiente punto, donde veremos cómo automatizar este proceso de recuperación jerárquica de forma eficiente en C#.