# 3.2 Implementación del Pipeline de Ingesta de Datos

Una vez que hemos extraído y normalizado el texto de nuestros documentos PDF (como se detalló en la sección 3.1), el siguiente paso crítico es la **Ingesta de Datos**. Este proceso transforma el texto bruto en unidades de información manejables y recuperables que alimentarán nuestra base de datos vectorial.

### 3.2 Implementación del Pipeline de Ingesta de Datos

El pipeline de ingesta no es simplemente un volcado de texto; es una arquitectura de transformación que determina qué tan bien podrá el LLM "entender" el contexto recuperado más adelante.

#### 3.2.1 Impacto de la división de texto (Chunking) en la calidad de RAG

El *Chunking* es el proceso de dividir el texto en fragmentos más pequeños. Su importancia radica en tres factores:

1.  **Límites de Contexto:** Los LLMs tienen una ventana de contexto finita (tokens). No podemos enviar un PDF de 50 páginas como contexto para una sola pregunta.
2.  **Relevancia de la Recuperación:** Las bases de datos vectoriales buscan similitud semántica. Si un fragmento es demasiado grande, el "ruido" de temas secundarios diluye el vector de embedding, dificultando que el motor de búsqueda encuentre la respuesta específica.
3.  **Costo y Latencia:** Fragmentos más pequeños y precisos reducen el consumo de tokens en la llamada al modelo generativo.

Un buen fragmento debe ser lo suficientemente grande para contener una idea completa, pero lo suficientemente pequeño para ser específico. Generalmente, se utiliza un **Overlap** (solapamiento) entre fragmentos para asegurar que la información que queda en los cortes no pierda su significado.

#### 3.2.2 Implementación de estrategias de fragmentación en C#

A continuación, exploramos cómo implementar las estrategias de fragmentación más comunes utilizando el ecosistema de .NET.

##### A. Fragmentación basada en Tokens (`Microsoft.ML.Tokenizers`)
Dado que los LLMs cobran y "piensan" en tokens, esta es la forma más precisa de controlar el tamaño del contexto. Utilizaremos la librería oficial de Microsoft para asegurar la compatibilidad con modelos como GPT-4.

```csharp
using Microsoft.ML.Tokenizers;

public class TokenChunker
{
    private readonly Tokenizer _tokenizer;

    public TokenChunker(string modelName = "gpt-4")
    {
        // Usamos Tiktoken para modelos de OpenAI
        _tokenizer = Tokenizer.CreateTiktokenForModel(modelName);
    }

    public List<string> CreateChunks(string text, int maxTokensPerChunk, int overlapTokens)
    {
        var tokens = _tokenizer.Encode(text).Tokens;
        var chunks = new List<string>();

        for (int i = 0; i < tokens.Count; i += (maxTokensPerChunk - overlapTokens))
        {
            var chunkTokens = tokens.Skip(i).Take(maxTokensPerChunk).ToList();
            chunks.Add(_tokenizer.Decode(chunkTokens));

            if (i + maxTokensPerChunk >= tokens.Count) break;
        }

        return chunks;
    }
}
```

##### B. Fragmentación Recursiva por Caracteres
Esta estrategia intenta mantener la estructura lógica del documento. Busca primero dividir por párrafos (`\n\n`), luego por líneas (`\n`), y finalmente por espacios, hasta alcanzar el tamaño deseado. Aunque existen implementaciones en Semantic Kernel, aquí vemos una lógica simplificada para entender su funcionamiento:

```csharp
public class RecursiveCharacterChunker
{
    private readonly int _maxChunkSize;
    private readonly char[] _separators = { '\n', '.', ' ', "" };

    public RecursiveCharacterChunker(int maxChunkSize) => _maxChunkSize = maxChunkSize;

    public List<string> SplitText(string text)
    {
        var result = new List<string>();
        
        if (text.Length <= _maxChunkSize)
        {
            result.Add(text);
            return result;
        }

        // Lógica de división recursiva basada en el primer separador que reduzca el tamaño
        foreach (var separator in _separators)
        {
            var parts = text.Split(separator, StringSplitOptions.RemoveEmptyEntries);
            if (parts.Length > 1)
            {
                // Re-ensamblar partes respetando el límite (simplificado)
                // En una implementación real, aquí se gestionaría el overlap
                foreach (var part in parts)
                {
                    result.AddRange(SplitText(part));
                }
                break;
            }
        }
        return result;
    }
}
```

##### C. Fragmentación Semántica
A diferencia de las anteriores, esta técnica utiliza modelos de IA para identificar dónde termina una idea y comienza otra. En .NET, esto suele implementarse calculando la similitud de coseno entre las oraciones adyacentes y rompiendo el fragmento cuando la similitud cae por debajo de un umbral.

#### 3.2.3 Impacto de los metadatos en la base de datos vectorial

Inyectar texto plano en una base de datos vectorial es un error común. Los **metadatos** son esenciales para la fase de recuperación avanzada y la trazabilidad.

Al diseñar nuestra estructura de datos en C#, debemos considerar una clase que represente el "Documento de Ingesta":

```csharp
public record VectorDocument
{
    public string Id { get; init; } = Guid.NewGuid().ToString();
    public string Text { get; init; }
    public float[] Embedding { get; set; } // Se generará en el siguiente paso
    
    // Metadatos cruciales
    public Dictionary<string, object> Metadata { get; init; } = new();
}
```

**Metadatos recomendados para aplicaciones RAG en .NET:**

1.  **Source/FileName:** Permite al LLM citar la fuente (ej: "Según el Manual_Operaciones.pdf...").
2.  **PageNumber:** Crucial para que el usuario final pueda verificar la respuesta en el PDF original.
3.  **DocumentType/Category:** Permite realizar **filtros de metadatos** antes de la búsqueda vectorial (ej: buscar solo en documentos de la categoría "Finanzas"), lo que mejora drásticamente la precisión y velocidad.
4.  **ChunkIndex:** Mantiene el orden secuencial, útil si necesitamos recuperar el fragmento anterior o posterior para dar más contexto (técnica que se profundizará en el capítulo 4).
5.  **Timestamp:** Permite descartar información obsoleta en documentos que se actualizan frecuentemente.

La correcta implementación de este pipeline garantiza que los datos no solo estén "disponibles", sino que estén estructurados de forma que el proceso de recuperación (Retrieval) sea lo más eficiente posible.