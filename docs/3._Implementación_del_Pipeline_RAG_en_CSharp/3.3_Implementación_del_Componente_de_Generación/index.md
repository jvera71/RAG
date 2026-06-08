# 3.3 Implementación del Componente de Generación

Una vez que hemos procesado nuestros documentos PDF y recuperado los fragmentos de texto (chunks) más relevantes desde nuestra base de datos vectorial, entramos en la fase final del pipeline RAG básico: la **Generación**.

En esta etapa, el componente de generación actúa como el "cerebro" del sistema, tomando la consulta original del usuario y el contexto recuperado para producir una respuesta coherente, precisa y, sobre todo, fundamentada en los datos proporcionados.

### 3.3.1 Ingeniería de Prompts estructurada usando Semantic Kernel Prompts

La calidad de la respuesta generada depende directamente de cómo estructuramos las instrucciones para el LLM. En el ecosistema .NET, **Semantic Kernel** es la herramienta líder para gestionar esta interacción, permitiendo separar la lógica del código de la definición de los prompts.

#### Uso de plantillas con Handlebars
Semantic Kernel permite utilizar la sintaxis de Handlebars para crear prompts dinámicos que inyectan el contexto recuperado de forma limpia.

```handlebars
## Instrucciones
Eres un asistente virtual experto. Tu tarea es responder a la pregunta del usuario utilizando ÚNICAMENTE la información proporcionada en la sección de "Contexto". Si la respuesta no se encuentra en el contexto, indica amablemente que no tienes suficiente información para responder.

## Contexto
{{#each chunks}}
---
Documento: {{this.Source}}
Contenido: {{this.Text}}
---
{{/each}}

## Pregunta del usuario
{{query}}

## Respuesta
```

#### Definición de Prompts en YAML
Para una mayor profesionalización, es recomendable definir estos prompts en archivos YAML. Esto facilita el versionado y la colaboración entre ingenieros de prompts y desarrolladores.

```yaml
name: ResumenDocumental
description: Genera una respuesta basada en fragmentos de PDF.
template_format: semantic-kernel
template: |
  Responde de forma profesional basándote en:
  {{$context}}
  
  Usuario: {{$input}}
input_variables:
  - name: input
    description: La pregunta del usuario.
  - name: context
    description: Los fragmentos recuperados del PDF.
execution_settings:
  default:
    max_tokens: 1000
    temperature: 0.2
```

### 3.3.2 Integración del modelo generativo (Azure.AI.OpenAI SDK)

La implementación técnica en C# requiere configurar el SDK de Azure OpenAI (o el SDK de OpenAI) dentro del kernel de Semantic Kernel. A continuación, se muestra cómo orquestar el flujo desde que recibimos los fragmentos hasta que generamos la respuesta.

#### Configuración del Kernel

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Connectors.OpenAI;

var builder = Kernel.CreateBuilder();

// Agregamos el servicio de generación (Chat Completion)
builder.AddAzureOpenAIChatCompletion(
    deploymentName: "gpt-4",
    endpoint: "https://tu-recurso.openai.azure.com/",
    apiKey: "tu-api-key"
);

Kernel kernel = builder.Build();
```

#### Implementación del flujo de generación

Supongamos que ya hemos realizado la búsqueda vectorial y tenemos una lista de fragmentos. El siguiente código muestra cómo alimentar el componente de generación:

```csharp
// 1. Preparar el contexto (proveniente del paso 3.2)
var retrievedChunks = new List<string> { 
    "El límite de crédito para empresas tipo A es de $50,000.",
    "Las empresas tipo A deben presentar estados financieros anuales."
};

string contextBlob = string.Join("\n", retrievedChunks);

// 2. Definir la función semántica (Prompt)
string promptTemplate = @"
    Eres un analista financiero. Utiliza el siguiente contexto para responder la pregunta.
    Contexto: {{$context}}
    Pregunta: {{$query}}
";

var generateFunction = kernel.CreateFunctionFromPrompt(promptTemplate);

// 3. Ejecutar la generación
var arguments = new KernelArguments
{
    ["context"] = contextBlob,
    ["query"] = "¿Cuál es el límite de crédito para las empresas tipo A?"
};

var response = await kernel.InvokeAsync(generateFunction, arguments);

Console.WriteLine($"Respuesta del LLM: {response}");
```

### Consideraciones clave en la implementación

Al implementar el componente de generación en .NET, debemos tener en cuenta los siguientes factores críticos:

1.  **Groundedness (Fidelidad):** El prompt debe ser explícito en prohibir al modelo utilizar conocimiento externo ("hallucinations"). Se debe instruir al LLM para que admita cuando no conoce la respuesta basándose solo en el contexto.
2.  **Manejo de Ventana de Contexto:** Aunque en el punto 3.2 definimos el fragmentado (chunking), en la generación debemos asegurar que la suma de tokens del prompt, el contexto recuperado y la respuesta esperada no exceda el límite del modelo (ej. 8k o 128k tokens). En .NET, podemos usar la librería `Microsoft.ML.Tokenizers` para contar tokens antes de enviar la solicitud.
3.  **Parámetros de Configuración:** 
    *   **Temperature:** Para RAG sobre documentos técnicos o legales, se recomienda un valor bajo (entre 0 y 0.3) para garantizar respuestas deterministas y precisas.
    *   **Stop Sequences:** Podemos definir secuencias de parada para evitar que el modelo genere texto irrelevante después de responder.
4.  **Streaming:** Para mejorar la experiencia de usuario (UX) en aplicaciones Blazor o Web APIs, es recomendable usar el método `InvokeStreamingAsync` de Semantic Kernel, permitiendo que la respuesta se muestre palabra por palabra en la interfaz.

Este componente de generación cierra el ciclo del RAG básico, transformando datos crudos recuperados en respuestas estructuradas y accionables para el usuario final. En la siguiente sección (3.4), veremos cómo evaluar si estas respuestas son realmente precisas y fieles al documento original.