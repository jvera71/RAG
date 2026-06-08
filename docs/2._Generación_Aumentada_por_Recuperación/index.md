# 2. Generación Aumentada por Recuperación

Esta sección profundiza en los fundamentos, la arquitectura y los componentes críticos que componen un sistema de **Generación Aumentada por Recuperación (RAG)**. Mientras que la introducción sentó las bases teóricas, aquí analizaremos cómo se estructuran estos sistemas dentro del ecosistema .NET para resolver las limitaciones inherentes de los modelos de lenguaje.

## 2.1 Las limitaciones de la IA Generativa

A pesar de su potencia, los LLMs (como GPT-4 o Claude) operan bajo restricciones que pueden comprometer su utilidad en entornos corporativos.

### 2.1.1 Alucinaciones y el problema de la fecha de corte de conocimiento
Los LLMs no "saben" cosas en el sentido tradicional; son predictores estadísticos de tokens basados en un conjunto de datos estático.
*   **Fecha de corte (Knowledge Cutoff):** Un modelo entrenado hasta 2023 no conocerá las leyes fiscales de 2024 o el lanzamiento de un producto interno de su empresa ayer.
*   **Alucinaciones:** Cuando un modelo no tiene la información, tiende a generar respuestas que suenan plausibles pero son fácticamente incorrectas. En .NET, esto es crítico cuando desarrollamos software financiero o de salud donde la precisión es innegociable.

### 2.1.2 Ejemplo: Transformando la atención al cliente con RAG
Imagine un bot de atención al cliente para una aseguradora. Sin RAG, el bot respondería generalidades sobre pólizas basadas en su entrenamiento. Con RAG, el sistema busca en el repositorio de PDFs de la empresa la póliza específica del usuario "ID-105" y responde basándose *únicamente* en ese documento.

### 2.1.3 Cómo RAG transforma la atención al cliente
RAG actúa como un examen a "libro abierto". En lugar de confiar en la memoria del modelo, le proporcionamos los fragmentos de texto exactos necesarios para responder, reduciendo drásticamente las alucinaciones y garantizando que la información esté actualizada sin necesidad de reentrenar el modelo.

---

## 2.2 Introducción a la Generación Aumentada por Recuperación

RAG es un patrón de diseño que combina dos fases:
1.  **Recuperación (Retrieval):** Localizar información relevante en una fuente de datos externa (ej. una base de datos vectorial con contenido de PDFs).
2.  **Generación (Generation):** Utilizar un LLM para sintetizar una respuesta basada en la información recuperada.

Este enfoque separa el "conocimiento del mundo" (el LLM) del "conocimiento del dominio" (sus documentos).

---

## 2.3 Arquitectura de RAG en aplicaciones .NET (Visión General)

En el mundo .NET, una arquitectura RAG típica se compone de servicios desacoplados que interactúan mediante interfaces. Un flujo estándar sigue este orden:
1.  El usuario envía una consulta vía **ASP.NET Core Web API**.
2.  Un servicio de **Embeddings** convierte la consulta en un vector numérico.
3.  Se realiza una búsqueda de similitud en una **Base de Datos Vectorial**.
4.  El contexto recuperado se inyecta en un **Prompt** junto con la pregunta original.
5.  El **LLM** genera la respuesta final.

---

## 2.4 Construcción del sistema de recuperación (Retrieval)

El sistema de recuperación es el corazón del RAG. En C#, esto se implementa generalmente mediante un servicio que abstrae la complejidad de la búsqueda semántica.

```csharp
public interface IRetrievalService
{
    // Busca los fragmentos más relevantes basados en una consulta
    Task<IEnumerable<TextChunk>> GetRelevantContextAsync(string query, int limit = 3);
}
```

El objetivo es pasar de una búsqueda por palabras clave (SQL `LIKE` o Full-text search) a una **búsqueda semántica**, donde el sistema entiende que "normativa de vacaciones" y "política de días libres" significan lo mismo.

---

## 2.5 Embeddings y Bases de Datos Vectoriales para .NET

Para que la búsqueda semántica funcione, necesitamos transformar el texto en vectores (listas de números que representan conceptos).

### 2.5.1 Modelos de Embeddings
En .NET, podemos generar estos vectores de tres formas:
*   **Cloud:** Usando `Azure.AI.OpenAI` para llamar al modelo `text-embedding-3-small`.
*   **Local:** Usando **ONNX Runtime** y modelos de HuggingFace (como `all-MiniLM-L6-v2`) para procesar datos sin que salgan del servidor, ideal para cumplimiento de privacidad.

### 2.5.2 Bases de datos vectoriales con soporte en C#
Una vez tenemos los vectores, necesitamos dónde guardarlos. Las opciones más sólidas para .NET son:
*   **Azure AI Search:** La solución empresarial nativa con integración profunda en Semantic Kernel.
*   **Qdrant / Milvus:** Bases de datos vectoriales puras con SDKs para .NET muy eficientes.
*   **pgvector:** Extensión de PostgreSQL que permite usar **Entity Framework Core** para realizar búsquedas vectoriales.

---

## 2.6 Pipeline de ingesta de datos en RAG

La ingesta es el proceso de preparar los PDFs para que sean "buscables". Aunque los detalles técnicos se verán en el capítulo 3, el flujo lógico en .NET es:
1.  **Extract:** Leer el PDF.
2.  **Chunk:** Dividir el texto en piezas pequeñas (ej. 500 caracteres).
3.  **Embed:** Convertir cada pieza en un vector.
4.  **Upsert:** Guardar el texto y el vector en la base de datos.

---

## 2.7 Desafíos de la Generación Aumentada por Recuperación

Implementar RAG no está exento de retos técnicos:
*   **Calidad de los fragmentos:** Si el fragmento recuperado está cortado a la mitad, el LLM perderá el contexto.
*   **Latencia:** La doble llamada (una a la base de datos de vectores y otra al LLM) requiere una gestión asíncrona eficiente mediante `Task.WhenAll` o streaming.
*   **Ruido:** Recuperar documentos irrelevantes puede confundir al modelo.

---

## 2.8 Consideraciones de Seguridad y Privacidad en entornos empresariales

Al trabajar con RAG en .NET para empresas, la seguridad es transversal:
*   **Identidad:** Uso de `Microsoft.Identity.Web` para asegurar que el usuario solo recupere documentos a los que tiene permiso (Seguridad a nivel de documento).
*   **Aislamiento de datos:** En Azure, el uso de **Managed Identities** evita exponer claves de API de OpenAI o de las bases de datos en los archivos de configuración.
*   **Residencia de datos:** Configurar los recursos de Azure en regiones específicas para cumplir con normativas como GDPR.

```csharp
// Ejemplo de configuración segura en Program.cs usando DefaultAzureCredential
builder.Services.AddAzureClients(clientBuilder =>
{
    clientBuilder.AddOpenAIClient(new Uri(builder.Configuration["AzureOpenAI:Endpoint"]), 
        new DefaultAzureCredential());
});
```

Este enfoque garantiza que el sistema RAG sea no solo inteligente, sino también robusto y apto para producción en infraestructuras corporativas.