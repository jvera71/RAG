# 4.1 Introducción

Habiendo explorado en el capítulo anterior cómo construir un pipeline de **Naive RAG** (RAG Básico) —donde extraemos texto, lo fragmentamos, generamos embeddings y consultamos una base de datos vectorial—, es imperativo reconocer que, en entornos de producción reales, este enfoque suele ser insuficiente.

La transición de un prototipo a una solución empresarial requiere abordar fallos críticos que surgen cuando los documentos son complejos, las consultas son ambiguas o el volumen de información escala. A continuación, desglosamos los desafíos que definen la necesidad de evolucionar hacia el **RAG Avanzado**.

### Desafíos de Recuperación (Retrieval)

El componente de recuperación es el "buscador" de nuestro sistema. Su objetivo es encontrar los fragmentos más relevantes, pero en un entorno básico enfrenta varios obstáculos:

1.  **Baja Precisión (Ruido):** No todos los fragmentos recuperados por similitud de coseno son útiles. Un fragmento puede ser semánticamente similar a la consulta pero no contener la respuesta, introduciendo "ruido" que confunde al LLM.
2.  **Baja Recuperación (Recall):** Ocurre cuando el sistema no logra encontrar todos los fragmentos necesarios para responder. Esto sucede a menudo si los términos de búsqueda del usuario no coinciden exactamente con los términos técnicos del PDF.
3.  **Desalineación Semántica:** Los modelos de embeddings a veces ignoran matices importantes (como la negación) o fallan al capturar el contexto global del documento al estar limitados a fragmentos aislados (*chunks*).
4.  **Desactualización:** Si el índice vectorial no se sincroniza correctamente con los cambios en los documentos originales, el sistema recuperará información obsoleta.

### Desafíos de Generación

Incluso si recuperamos la información correcta, el LLM puede fallar al procesarla:

1.  **Alucinaciones con Contexto:** El modelo ignora los fragmentos proporcionados y responde basándose en su conocimiento preentrenado, o inventa datos mezclando la información del PDF con hechos falsos.
2.  **Efecto "Lost in the Middle":** Los LLMs tienden a priorizar la información al principio y al final del prompt. Si la respuesta clave se encuentra en el medio de un bloque de 10 fragmentos recuperados, el modelo puede pasarla por alto.
3.  **Falta de Relevancia o Formato:** El modelo puede generar una respuesta veraz pero que no sigue las instrucciones de formato (JSON, Markdown) o el tono requerido por la aplicación .NET.
4.  **Incoherencia ante Fragmentos Contradictorios:** Si dos documentos PDF contienen información opuesta, un pipeline básico no sabe cómo dirimir la verdad, resultando en respuestas confusas.

### Qué se puede hacer para solucionarlos

Para superar estas limitaciones, el RAG Avanzado introduce capas de inteligencia adicionales antes, durante y después del proceso de recuperación. En los siguientes subtemas de este capítulo, implementaremos estrategias en C# que transforman el pipeline:

*   **Optimización de Fragmentación:** No limitarnos a fragmentos de tamaño fijo, sino usar estrategias semánticas o la técnica de **Parent Document Retriever**.
*   **Recuperación Híbrida:** Combinar la búsqueda vectorial (semántica) con la búsqueda de texto completo (keyword search) usando herramientas como Azure AI Search.
*   **Post-procesamiento (Re-ranking):** Implementar modelos de re-clasificación para ordenar los resultados recuperados por su relevancia real antes de enviarlos al LLM.
*   **Refinamiento de Consultas:** Usar el LLM para "reescribir" la pregunta del usuario, haciéndola más apta para la búsqueda vectorial.

#### Ejemplo conceptual en C#: El problema del Naive RAG

Imagine un sistema que utiliza `Semantic Kernel` para buscar en manuales técnicos. Un enfoque básico se vería así:

```csharp
// Enfoque Naive: Recuperación directa por similitud
var searchResults = await _vectorStore.GetNearestMatchesAsync(
    collectionName: "manuales_tecnicos",
    embedding: userQueryEmbedding,
    limit: 3,
    minRelevanceScore: 0.7);

// Problema: Si la consulta es "instrucciones de seguridad", 
// el sistema podría traer fragmentos genéricos que no responden 
// a la necesidad específica del usuario de un modelo de máquina concreto.
```

En las próximas secciones, veremos cómo mejorar este flujo mediante **Query Rewriting** o **Hybrid Search** para asegurar que `searchResults` contenga exactamente lo que el usuario necesita, eliminando las debilidades del modelo de recuperación simple. El objetivo es pasar de una "búsqueda por cercanía" a una "recuperación de conocimiento contextual".