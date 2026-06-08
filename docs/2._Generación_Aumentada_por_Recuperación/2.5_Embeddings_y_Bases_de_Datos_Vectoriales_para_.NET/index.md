# 2.5 Embeddings y Bases de Datos Vectoriales para .NET

En el contexto de un sistema RAG, la capacidad de transformar información textual en una representación que una máquina pueda "entender" semánticamente es fundamental. En esta sección, profundizaremos en cómo .NET gestiona los **embeddings** y qué opciones de **bases de datos vectoriales** existen para persistir y consultar estos datos de manera eficiente.

### 2.5.1 Modelos de Embeddings

Un *embedding* es una representación numérica (un vector de números de punto flotante) de un fragmento de texto. A diferencia de una búsqueda por palabras clave tradicional, los embeddings capturan el **significado semántico**. En .NET, tenemos tres vías principales para generar estos vectores:

#### Azure OpenAI y OpenAI SDK para .NET
Es la opción más común en entornos empresariales. Utilizando la librería oficial `Azure.AI.OpenAI` o `OpenAI`, podemos enviar texto a modelos como `text-embedding-3-small` o `text-embedding-3-large`.

```csharp
using Azure.AI.OpenAI;
using Azure;

// Configuración del cliente
OpenAIClient client = new OpenAIClient(new Uri("https://tu-recurso.openai.azure.com/"), new AzureKeyCredential("tu-api-key"));

EmbeddingsOptions options = new EmbeddingsOptions("text-embedding-3-small", new[] { "El framework .NET es ideal para IA" });

Response<Embeddings> response = await client.GetEmbeddingsAsync(options);
ReadOnlyMemory<float> embedding = response.Value.Data[0].Embedding;

// El resultado es un vector de 1536 dimensiones (por defecto para este modelo)
Console.WriteLine($"Dimensiones del vector: {embedding.Length}");
```

#### Ejecución local con ONNX Runtime
Para escenarios donde la privacidad es crítica o se busca reducir costos de API, podemos ejecutar modelos de embeddings (como **BERT** o **BGE**) localmente en C# utilizando **ONNX Runtime**. Esto permite que el procesamiento de datos nunca salga de la infraestructura controlada por la aplicación.

*   **Ventaja:** Latencia cero de red y mayor privacidad.
*   **Librerías:** `Microsoft.ML.OnnxRuntime` y `Microsoft.ML.Tokenizers`.

### 2.5.2 Bases de Datos Vectoriales con soporte en C#

Una vez que el texto del PDF se convierte en vectores, necesitamos un motor de búsqueda que no busque coincidencias exactas, sino la **distancia mínima** entre vectores (similitud del coseno). .NET ofrece una excelente integración con los principales proveedores:

#### 1. Azure AI Search
Es la solución gestionada de Microsoft. No solo es una base de datos vectorial, sino un motor de búsqueda completo que soporta búsqueda híbrida (texto + vector).
*   **SDK:** `Azure.Search.Documents`.
*   **Uso ideal:** Aplicaciones corporativas que requieren escalabilidad automática y seguridad integrada con Azure.

#### 2. Qdrant .NET SDK
Qdrant es una de las bases de datos vectoriales de código abierto más eficientes. Su SDK para .NET es de alta calidad y sigue patrones modernos de C#.
*   **Características:** Soporte nativo para filtrado de metadatos (payloads) y alta velocidad en búsquedas de tipo *Nearest Neighbor*.

#### 3. PostgreSQL con pgvector y Entity Framework Core
Esta es la opción preferida para desarrolladores .NET que ya utilizan bases de datos relacionales. Gracias a la extensión `pgvector`, PostgreSQL puede almacenar columnas de tipo `vector`.

La integración con **EF Core** permite definir modelos de la siguiente manera:

```csharp
public class DocumentChunk
{
    public int Id { get; set; }
    public string Content { get; set; }
    
    // Propiedad que almacena el embedding (usando pgvector)
    public Vector Embedding { get; set; } 
}

// Consulta de similitud en LINQ (vía extensiones de pgvector)
var nearestDocs = await context.Chunks
    .OrderBy(c => c.Embedding.L2Distance(queryVector))
    .Take(5)
    .ToListAsync();
```

#### 4. Milvus
Milvus es una base de datos diseñada para manejar miles de millones de vectores. Su SDK para C# permite interactuar con clusters de alto rendimiento, siendo la opción predilecta para proyectos de Big Data en el ecosistema .NET.

---

### Comparativa técnica para la elección

| Característica | Azure AI Search | Qdrant / Milvus | PostgreSQL (pgvector) |
| :--- | :--- | :--- | :--- |
| **Tipo** | SaaS (Gestionado) | Vector Database Nativa | Relacional con extensión |
| **Curva de aprendizaje** | Baja (si ya usas Azure) | Media | Muy baja (si usas EF Core) |
| **Búsqueda Híbrida** | Excelente (Integrada) | Disponible | Requiere configuración manual |
| **Costo inicial** | Variable (Tiered) | Bajo (Self-hosted/Open source) | Bajo (Existente) |

La elección entre estas opciones dependerá de la escala de los documentos PDF a procesar y de la infraestructura existente en el proyecto .NET. Mientras que `Azure AI Search` ofrece una experiencia "llave en mano", `pgvector` con `Entity Framework` permite mantener la simplicidad arquitectónica al no introducir un nuevo motor de base de datos.