# Laboratorio: Configuración de CustomProblemDetails en ASP.NET Core

## Objetivo
Configurar un manejo global de errores usando `CustomProblemDetails` siguiendo la RFC 7807, que se aplique automáticamente a todos los endpoints de la API sin necesidad de configuración individual. Este laboratorio usa **FluentValidation** para la validación de modelos.

---

## Paso 1: Crear la clase CustomProblemDetails

Crea el archivo `CustomProblemDetails.cs` en la carpeta `common`:

**Ruta:** `webapi/common/CustomProblemDetails.cs`

```csharp
namespace webapi.common;

public class CustomProblemDetails
{
    public string Type { get; set; } = "about:blank";
    public string Title { get; set; } = string.Empty;
    public int Status { get; set; }
    public string Detail { get; set; } = string.Empty;
    public string Instance { get; set; } = string.Empty;
    public Dictionary<string, object>? Extensions { get; set; }
}
```

**Descripción:** Esta clase representa la estructura estándar de errores según RFC 7807, con un diccionario de extensiones para información adicional como `traceId` y `timestamp`.

---

## Paso 2: Crear el GlobalExceptionHandler

Crea el archivo `GlobalExceptionHandler.cs` en la carpeta `common`:

**Ruta:** `webapi/common/GlobalExceptionHandler.cs`

```csharp
using Microsoft.AspNetCore.Diagnostics;
using FluentValidation;

namespace webapi.common;

public class GlobalExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        var problemDetails = new CustomProblemDetails
        {
            Instance = httpContext.Request.Path,
            Extensions = new Dictionary<string, object>
            {
                ["traceId"] = httpContext.TraceIdentifier,
                ["timestamp"] = DateTime.UtcNow
            }
        };

        switch (exception)
        {
            case KeyNotFoundException:
                problemDetails.Status = StatusCodes.Status404NotFound;
                problemDetails.Title = "Resource Not Found";
                problemDetails.Detail = exception.Message;
                problemDetails.Type = "https://tools.ietf.org/html/rfc7231#section-6.5.4";
                break;

            case ValidationException validationException:
                problemDetails.Status = StatusCodes.Status400BadRequest;
                problemDetails.Title = "Validation Error";
                problemDetails.Detail = "One or more validation errors occurred.";
                problemDetails.Type = "https://tools.ietf.org/html/rfc7231#section-6.5.1";
                
                // Agregar los errores de validación al diccionario de extensiones
                var errors = validationException.Errors
                    .GroupBy(e => e.PropertyName)
                    .ToDictionary(
                        g => g.Key,
                        g => g.Select(e => e.ErrorMessage).ToArray()
                    );
                
                problemDetails.Extensions["errors"] = errors;
                break;

            case ArgumentException:
                problemDetails.Status = StatusCodes.Status400BadRequest;
                problemDetails.Title = "Bad Request";
                problemDetails.Detail = exception.Message;
                problemDetails.Type = "https://tools.ietf.org/html/rfc7231#section-6.5.1";
                break;

            default:
                problemDetails.Status = StatusCodes.Status500InternalServerError;
                problemDetails.Title = "Internal Server Error";
                problemDetails.Detail = "An unexpected error occurred";
                problemDetails.Type = "https://tools.ietf.org/html/rfc7231#section-6.6.1";
                break;
        }

        httpContext.Response.StatusCode = problemDetails.Status;
        httpContext.Response.ContentType = "application/problem+json";
        await httpContext.Response.WriteAsJsonAsync(problemDetails, cancellationToken);

        return true;
    }
}
```

**Descripción:** Este handler intercepta todas las excepciones no manejadas y las convierte en respuestas `CustomProblemDetails` con el código HTTP apropiado. Para las excepciones de FluentValidation, agrupa los errores por propiedad y los incluye en el diccionario `Extensions`.

---

## Paso 3: Crear el GlobalErrorResponsesOperationFilter

Crea el archivo `GlobalErrorResponsesOperationFilter.cs` en la carpeta `common`:

**Ruta:** `webapi/common/GlobalErrorResponsesOperationFilter.cs`

```csharp
using Microsoft.OpenApi.Models;
using Swashbuckle.AspNetCore.SwaggerGen;

namespace webapi.common;

public class GlobalErrorResponsesOperationFilter : IOperationFilter
{
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        // Agregar respuesta 400 si no existe
        if (!operation.Responses.ContainsKey("400"))
        {
            operation.Responses.Add("400", new OpenApiResponse
            {
                Description = "Bad Request",
                Content = new Dictionary<string, OpenApiMediaType>
                {
                    ["application/problem+json"] = new OpenApiMediaType
                    {
                        Schema = context.SchemaGenerator.GenerateSchema(
                            typeof(CustomProblemDetails), 
                            context.SchemaRepository)
                    }
                }
            });
        }

        // Agregar respuesta 404 si no existe (excepto para POST)
        if (!operation.Responses.ContainsKey("404") && 
            context.ApiDescription.HttpMethod?.ToUpper() != "POST")
        {
            operation.Responses.Add("404", new OpenApiResponse
            {
                Description = "Not Found",
                Content = new Dictionary<string, OpenApiMediaType>
                {
                    ["application/problem+json"] = new OpenApiMediaType
                    {
                        Schema = context.SchemaGenerator.GenerateSchema(
                            typeof(CustomProblemDetails), 
                            context.SchemaRepository)
                    }
                }
            });
        }

        // Agregar respuesta 500 si no existe
        if (!operation.Responses.ContainsKey("500"))
        {
            operation.Responses.Add("500", new OpenApiResponse
            {
                Description = "Internal Server Error",
                Content = new Dictionary<string, OpenApiMediaType>
                {
                    ["application/problem+json"] = new OpenApiMediaType
                    {
                        Schema = context.SchemaGenerator.GenerateSchema(
                            typeof(CustomProblemDetails), 
                            context.SchemaRepository)
                    }
                }
            });
        }
    }
}
```

**Descripción:** Este filtro agrega automáticamente las respuestas de error (400, 404, 500) a la documentación OpenAPI de todos los endpoints.

---

## Paso 4: Configurar Program.cs

Modifica tu archivo `Program.cs` para registrar los servicios:

**Ruta:** `webapi/Program.cs`

```csharp
using webapi.common;
using Microsoft.OpenApi.Models;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);

// Registrar el exception handler global
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

// Configurar controladores y endpoints
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();

// Configurar Swagger/OpenAPI
builder.Services.AddSwaggerGen(options =>
{
    // Título y descripción de la API
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Pizza API",
        Version = "v1",
        Description = "API para gestión de pizzas e ingredientes"
    });

    // Personalizar el nombre del esquema para CustomProblemDetails
    options.CustomSchemaIds(type => 
    {
        if (type == typeof(CustomProblemDetails))
            return "CustomProblemDetails";
        
        return type.Name;
    });

    // Mapear ProblemDetails de Microsoft a CustomProblemDetails
    options.MapType<ProblemDetails>(() => new OpenApiSchema
    {
        Type = "object",
        Properties = new Dictionary<string, OpenApiSchema>
        {
            ["type"] = new OpenApiSchema 
            { 
                Type = "string", 
                Example = new Microsoft.OpenApi.Any.OpenApiString("about:blank") 
            },
            ["title"] = new OpenApiSchema { Type = "string" },
            ["status"] = new OpenApiSchema { Type = "integer", Format = "int32" },
            ["detail"] = new OpenApiSchema { Type = "string" },
            ["instance"] = new OpenApiSchema { Type = "string" },
            ["extensions"] = new OpenApiSchema 
            { 
                Type = "object",
                AdditionalPropertiesAllowed = true,
                Example = new Microsoft.OpenApi.Any.OpenApiObject
                {
                    ["traceId"] = new Microsoft.OpenApi.Any.OpenApiString("00-abc123-def456-01"),
                    ["timestamp"] = new Microsoft.OpenApi.Any.OpenApiString("2025-10-30T10:30:00Z")
                }
            }
        },
        AdditionalPropertiesAllowed = false
    });

    // Aplicar el filtro global para respuestas de error
    options.OperationFilter<GlobalErrorResponsesOperationFilter>();
});

var app = builder.Build();

// Configurar el pipeline HTTP
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// Habilitar el exception handler (DEBE ir antes de UseRouting)
app.UseExceptionHandler();
app.UseStatusCodePages();

app.UseHttpsRedirection();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

**Puntos clave:**
- `AddExceptionHandler<GlobalExceptionHandler>()`: Registra el manejador de excepciones
- `MapType<ProblemDetails>()`: Mapea el esquema de OpenAPI para usar nuestra estructura
- `CustomSchemaIds()`: Personaliza el nombre del esquema en OpenAPI (aparecerá como `CustomProblemDetails` en lugar de `webapicommonopenapiCustomProblemDetails`)
- `OperationFilter<GlobalErrorResponsesOperationFilter>()`: Aplica las respuestas de error a todos los endpoints

---

## Paso 5: Verificar los endpoints existentes

Tus endpoints **NO necesitan** configuración adicional. El código quedará limpio:

```csharp
public void AddRoutes(IEndpointRouteBuilder app)
{
    app.MapPut("/ingredientes/{id:guid}", async (IService service, Guid id, [FromBody] Request request) =>
    {
        await service.Handler(id, request);
        return Results.NoContent();
    })        
    .WithOpenApi()
    .WithName("UpdateIngredient")
    .WithSummary("Actualizar un ingrediente existente")
    .WithDescription("Endpoint para actualizar el nombre y costo de un ingrediente")
    .WithTags("Ingredientes")
    .Produces(StatusCodes.Status204NoContent);
    // ¡No necesitas agregar .Produces<CustomProblemDetails>!
}
```

---

## Paso 6: Probar la configuración

### 6.1 Ejecutar la aplicación

```bash
dotnet run
```

### 6.2 Abrir Swagger UI

Navega a: `https://localhost:5001/swagger`

### 6.3 Verificar en Swagger

Para cada endpoint deberías ver automáticamente las respuestas:
- ✅ **200 OK** o **204 No Content** (según el endpoint)
- ✅ **400 Bad Request** → Schema: `CustomProblemDetails`
- ✅ **404 Not Found** → Schema: `CustomProblemDetails`
- ✅ **500 Internal Server Error** → Schema: `CustomProblemDetails`

### 6.4 Probar un error 404

Ejecuta desde terminal o Postman:

```bash
curl -X GET https://localhost:5001/ingredientes/00000000-0000-0000-0000-000000000000
```

**Respuesta esperada:**

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.4",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "Ingredient with ID '00000000-0000-0000-0000-000000000000' not found.",
  "instance": "/ingredientes/00000000-0000-0000-0000-000000000000",
  "extensions": {
    "traceId": "00-abc123def456-xyz789-01",
    "timestamp": "2025-10-30T15:30:00.000Z"
  }
}
```

### 6.5 Probar un error 400 (validación)

Ejecuta:

```bash
curl -X POST https://localhost:5001/ingredientes \
  -H "Content-Type: application/json" \
  -d '{"name": "", "cost": -5}'
```

**Respuesta esperada:**

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "Validation Error",
  "status": 400,
  "detail": "One or more validation errors occurred.",
  "instance": "/ingredientes",
  "extensions": {
    "traceId": "00-def456abc123-qrs456-02",
    "timestamp": "2025-10-30T15:35:00.000Z",
    "errors": {
      "Name": [
        "Name cannot be empty",
        "Name must be at least 3 characters"
      ],
      "Cost": [
        "Cost must be greater than zero"
      ]
    }
  }
}
```

**Nota:** Los errores de FluentValidation se agrupan por propiedad en el objeto `errors` dentro de `extensions`.

---

## Paso 7: Personalizar excepciones adicionales (Opcional)

Si necesitas manejar más tipos de excepciones, agrega casos al `GlobalExceptionHandler`:

```csharp
case UnauthorizedAccessException:
    problemDetails.Status = StatusCodes.Status401Unauthorized;
    problemDetails.Title = "Unauthorized";
    problemDetails.Detail = exception.Message;
    problemDetails.Type = "https://tools.ietf.org/html/rfc7231#section-6.5.1";
    break;

case InvalidOperationException:
    problemDetails.Status = StatusCodes.Status409Conflict;
    problemDetails.Title = "Conflict";
    problemDetails.Detail = exception.Message;
    problemDetails.Type = "https://tools.ietf.org/html/rfc7231#section-6.5.8";
    break;

case NotImplementedException:
    problemDetails.Status = StatusCodes.Status501NotImplemented;
    problemDetails.Title = "Not Implemented";
    problemDetails.Detail = exception.Message;
    problemDetails.Type = "https://tools.ietf.org/html/rfc7231#section-6.6.2";
    break;
```

### Agregar información adicional a Extensions

También puedes agregar información contextual adicional al diccionario `Extensions`:

```csharp
case ValidationException validationException:
    problemDetails.Status = StatusCodes.Status400BadRequest;
    problemDetails.Title = "Validation Error";
    problemDetails.Detail = "One or more validation errors occurred.";
    problemDetails.Type = "https://tools.ietf.org/html/rfc7231#section-6.5.1";
    
    var errors = validationException.Errors
        .GroupBy(e => e.PropertyName)
        .ToDictionary(
            g => g.Key,
            g => g.Select(e => e.ErrorMessage).ToArray()
        );
    
    problemDetails.Extensions["errors"] = errors;
    problemDetails.Extensions["errorCount"] = validationException.Errors.Count();
    problemDetails.Extensions["source"] = "FluentValidation";
    break;
```

---

## Resumen

✅ **CustomProblemDetails:** Estructura estándar RFC 7807  
✅ **GlobalExceptionHandler:** Manejo centralizado de excepciones  
✅ **GlobalErrorResponsesOperationFilter:** Documentación automática en OpenAPI  
✅ **Configuración global:** Sin necesidad de configurar cada endpoint  
✅ **Respuestas consistentes:** Mismo formato en toda la API  

---

## Estructura final de archivos

```
webapi/
├── common/
│   ├── CustomProblemDetails.cs
│   ├── GlobalExceptionHandler.cs
│   └── GlobalErrorResponsesOperationFilter.cs
├── Program.cs
└── features/
    └── ingredient/
        ├── commands/
        │   ├── CreateIngredient.cs
        │   ├── UpdateIngredient.cs
        │   └── RemoveIngredient.cs
        └── queries/
            ├── GetIngredient.cs
            └── GetAllIngredients.cs
```

---

## Beneficios de esta configuración

1. **Consistencia:** Todas las respuestas de error siguen el mismo formato
2. **Mantenibilidad:** Cambios en el formato de errores se hacen en un solo lugar
3. **Documentación automática:** OpenAPI/Swagger documenta automáticamente todos los errores posibles
4. **Cumplimiento de estándares:** Sigue la RFC 7807 para Problem Details
5. **Código limpio:** Los endpoints no necesitan configuración adicional de errores
6. **Trazabilidad:** Cada error incluye `traceId` y `timestamp` para debugging

---

## Troubleshooting

### Problema: El esquema aparece con nombre largo `webapicommonopenapiCustomProblemDetails`

**Solución:** Agrega `CustomSchemaIds` en la configuración de Swagger:

```csharp
options.CustomSchemaIds(type => 
{
    if (type == typeof(CustomProblemDetails))
        return "CustomProblemDetails";
    
    return type.Name;
});
```

### Problema: Los errores no se están capturando

**Solución:** Verifica que `app.UseExceptionHandler()` esté antes de `app.UseRouting()` en el pipeline.

### Problema: Swagger muestra `MicrosoftAspNetCoreMvcProblemDetails`

**Solución:** Asegúrate de haber agregado el `MapType<ProblemDetails>()` en la configuración de Swagger.

### Problema: No aparecen las respuestas 400, 404, 500 en Swagger

**Solución:** Verifica que el `OperationFilter<GlobalErrorResponsesOperationFilter>()` esté registrado correctamente.

### Problema: ValidationException no se reconoce

**Solución:** Asegúrate de tener instalado el paquete de FluentValidation y el using correcto:

```bash
dotnet add package FluentValidation
```

```csharp
using FluentValidation;
```

### Problema: Los errores de validación no aparecen en el formato esperado

**Solución:** Verifica que estés lanzando `ValidationException` de FluentValidation en tu código. El validador debe usar `ValidateAndThrow()`:

```csharp
_validator.ValidateAndThrow((name, cost));
```

---

¡Configuración completada! 🎉
