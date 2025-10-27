# Tema 2: Fundamentos de .NET para Microservicios

## 2.1 Introducción a .NET y .NET Core

### Historia de .NET Framework vs .NET Core

**.NET Framework** fue lanzado en 2002 y era exclusivo de Windows. Tenía limitaciones para arquitecturas modernas.

**.NET Core** surgió en 2016 como una versión multiplataforma, de código abierto y optimizada.

**Ejemplo de diferencia:**
```csharp
// .NET Framework - Solo Windows
// Instalado en C:\Windows\Microsoft.NET\Framework

// .NET Core - Multiplataforma
// Se ejecuta en Windows, Linux, macOS
dotnet run
```

### .NET Core 3.1 - Última versión LTS

- **.NET Core 1.0**: Primera versión (2016)
- **.NET Core 2.0**: Mejoras de rendimiento y APIs
- **.NET Core 3.0**: Soporte para Windows Desktop
- **.NET Core 3.1 (LTS)**: Última versión con soporte hasta diciembre 2022 (extendido)

**Ejemplo de versión:**
```bash
dotnet --version
# Salida: 3.1.x
```

### Ventajas de .NET Core para microservicios

**Multiplataforma y portabilidad:**
```bash
# Compilar en Windows
dotnet build

# Ejecutar en Linux
dotnet MyApi.dll
```

**Rendimiento y optimizaciones:**
- Alto rendimiento (hasta 7x más rápido que .NET Framework)
- Menor uso de memoria
- Startup más rápido
- Ideal para contenedores Docker

---

## 2.2 ASP.NET Core Web API

### Creación de un proyecto Web API

```bash
# Crear nueva API con .NET Core 3.1
dotnet new webapi -n ProductsApi -f netcoreapp3.1

# Cambiar al directorio
cd ProductsApi

# Ejecutar
dotnet run
```

### Estructura de un proyecto ASP.NET Core

```
ProductsApi/
├── Controllers/         # Controladores de API
├── Models/             # Modelos de datos
├── Properties/
│   └── launchSettings.json
├── Program.cs          # Punto de entrada
├── Startup.cs          # Configuración de servicios y middleware
├── appsettings.json    # Configuración
└── ProductsApi.csproj  # Definición del proyecto
```

**Program.cs:**
```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Hosting;

namespace ProductsApi
{
    public class Program
    {
        public static void Main(string[] args)
        {
            CreateHostBuilder(args).Build().Run();
        }

        public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder.UseStartup<Startup>();
                });
    }
}
```

**Startup.cs:**
```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

namespace ProductsApi
{
    public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        // ConfigureServices: registrar servicios en el contenedor DI
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddControllers();
        }

        // Configure: configurar el pipeline de HTTP request
        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseHttpsRedirection();
            app.UseRouting();
            app.UseAuthorization();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
            });
        }
    }
}
```

### Controladores y routing

**Ejemplo básico:**
```csharp
using Microsoft.AspNetCore.Mvc;

namespace ProductsApi.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class ProductsController : ControllerBase
    {
        [HttpGet]
        public IActionResult GetAll()
        {
            var products = new[] { "Laptop", "Mouse", "Keyboard" };
            return Ok(products);
        }

        [HttpGet("{id}")]
        public IActionResult GetById(int id)
        {
            return Ok($"Producto con ID: {id}");
        }

        [HttpPost]
        public IActionResult Create([FromBody] string name)
        {
            return CreatedAtAction(nameof(GetById), new { id = 1 }, name);
        }
    }
}
```

### Model binding y validación

**Modelo con validación:**
```csharp
using System.ComponentModel.DataAnnotations;

namespace ProductsApi.Models
{
    public class Product
    {
        [Required(ErrorMessage = "El nombre es obligatorio")]
        [StringLength(100)]
        public string Name { get; set; }

        [Range(0.01, 10000)]
        public decimal Price { get; set; }
    }
}
```

**Controller con validación:**
```csharp
[HttpPost]
public IActionResult Create([FromBody] Product product)
{
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState);
    }

    return Ok(product);
}
```

### Serialización JSON

**Configuración en Startup.cs:**
```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers()
        .AddJsonOptions(options =>
        {
            options.JsonSerializerOptions.PropertyNamingPolicy = null; // PascalCase
            options.JsonSerializerOptions.WriteIndented = true; // JSON formateado
        });
}
```

### Versionado de APIs

**Instalar paquete:**
```bash
dotnet add package Microsoft.AspNetCore.Mvc.Versioning
```

**Configuración en Startup.cs:**
```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddApiVersioning(options =>
    {
        options.ReportApiVersions = true;
        options.AssumeDefaultVersionWhenUnspecified = true;
        options.DefaultApiVersion = new ApiVersion(1, 0);
    });

    services.AddControllers();
}
```

**Ejemplo de versionado por URL:**
```csharp
using Microsoft.AspNetCore.Mvc;

namespace ProductsApi.Controllers.V1
{
    [ApiController]
    [ApiVersion("1.0")]
    [Route("api/v{version:apiVersion}/[controller]")]
    public class ProductsController : ControllerBase
    {
        [HttpGet]
        public IActionResult Get() => Ok("Versión 1");
    }
}
```

```csharp
namespace ProductsApi.Controllers.V2
{
    [ApiController]
    [ApiVersion("2.0")]
    [Route("api/v{version:apiVersion}/[controller]")]
    public class ProductsController : ControllerBase
    {
        [HttpGet]
        public IActionResult Get() => Ok("Versión 2 con nuevos campos");
    }
}
```

---

## 2.3 Dependency Injection en .NET Core

### Principios de Inversión de Control (IoC)

En lugar de crear dependencias directamente, las recibimos desde fuera.

**Sin DI (❌ Mal):**
```csharp
public class OrderService
{
    private readonly EmailService _emailService = new EmailService(); // Fuertemente acoplado
}
```

**Con DI (✅ Bien):**
```csharp
public class OrderService
{
    private readonly IEmailService _emailService;

    public OrderService(IEmailService emailService)
    {
        _emailService = emailService; // Inyectado
    }
}
```

### Contenedor de DI en .NET Core

**Registro en Startup.cs:**
```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Registrar servicios
    services.AddScoped<IProductRepository, ProductRepository>();
    services.AddSingleton<IConfiguration>(Configuration);
    
    services.AddControllers();
}
```

### Ciclos de vida: Transient, Scoped, Singleton

**Transient:** Nueva instancia cada vez que se solicita
```csharp
services.AddTransient<IEmailService, EmailService>();
// Cada inyección = nueva instancia
```

**Scoped:** Una instancia por request HTTP
```csharp
services.AddScoped<IOrderService, OrderService>();
// Misma instancia durante todo el request
```

**Singleton:** Una única instancia para toda la aplicación
```csharp
services.AddSingleton<ICacheService, CacheService>();
// Siempre la misma instancia
```

### Registro de servicios

**Ejemplo completo en Startup.cs:**
```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Servicios de infraestructura (Singleton)
    services.AddSingleton<ILogger, Logger>();

    // Servicios de negocio (Scoped)
    services.AddScoped<IProductService, ProductService>();
    services.AddScoped<IOrderService, OrderService>();

    // Servicios de repositorio (Scoped)
    services.AddScoped<IProductRepository, ProductRepository>();

    services.AddControllers();
}
```

### Buenas prácticas de DI

1. **Usar interfaces:**
```csharp
services.AddScoped<IProductService, ProductService>();
// No: AddScoped<ProductService, ProductService>();
```

2. **Constructor injection:**
```csharp
public class ProductController : ControllerBase
{
    private readonly IProductService _productService;

    public ProductController(IProductService productService)
    {
        _productService = productService;
    }
}
```

3. **Evitar Service Locator:**
```csharp
// ❌ Malo
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    var service = app.ApplicationServices.GetService<IProductService>();
}

// ✅ Bueno: inyectar en el constructor
```

---

## 2.4 Configuración y Options Pattern

### appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=ProductsDb;"
  },
  "ApiSettings": {
    "MaxPageSize": 100,
    "Timeout": 30
  }
}
```

### Variables de entorno

```bash
# Linux/macOS
export ConnectionStrings__DefaultConnection="Server=prod;Database=Prod;"

# Windows
set ConnectionStrings__DefaultConnection=Server=prod;Database=Prod;

# Docker
docker run -e ConnectionStrings__DefaultConnection="..." myapi
```

### User Secrets para desarrollo

```bash
# Inicializar secrets
dotnet user-secrets init

# Agregar un secret
dotnet user-secrets set "ApiKey" "mi-clave-secreta-123"

# Listar secrets
dotnet user-secrets list
```

**Uso en código:**
```csharp
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        var apiKey = Configuration["ApiKey"];
        // Usar apiKey...
        
        services.AddControllers();
    }
}
```

### Options Pattern

**Configuración:**
```json
// appsettings.json
{
  "EmailSettings": {
    "SmtpServer": "smtp.gmail.com",
    "Port": 587,
    "SenderEmail": "noreply@example.com"
  }
}
```

**Clase de opciones:**
```csharp
namespace ProductsApi.Configuration
{
    public class EmailSettings
    {
        public string SmtpServer { get; set; }
        public int Port { get; set; }
        public string SenderEmail { get; set; }
    }
}
```

**Registro en Startup.cs:**
```csharp
using Microsoft.Extensions.Options;

public void ConfigureServices(IServiceCollection services)
{
    // Configurar options
    services.Configure<EmailSettings>(
        Configuration.GetSection("EmailSettings"));

    // Registrar servicio que usa options
    services.AddScoped<IEmailService, EmailService>();
    
    services.AddControllers();
}
```

**Uso en el servicio:**
```csharp
using Microsoft.Extensions.Options;

namespace ProductsApi.Services
{
    public class EmailService : IEmailService
    {
        private readonly EmailSettings _settings;

        public EmailService(IOptions<EmailSettings> options)
        {
            _settings = options.Value;
        }

        public void SendEmail()
        {
            Console.WriteLine($"Enviando desde: {_settings.SenderEmail}");
            Console.WriteLine($"Servidor SMTP: {_settings.SmtpServer}:{_settings.Port}");
        }
    }
}
```

### Configuración por ambiente

**Archivos:**
```
appsettings.json              # Base
appsettings.Development.json  # Desarrollo
appsettings.Staging.json      # Pruebas
appsettings.Production.json   # Producción
```

**Ejemplo Development:**
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=DevDb;"
  }
}
```

**Ejemplo Production:**
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=prod-server;Database=ProdDb;"
  }
}
```

**Usar en Startup.cs:**
```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }
    else if (env.IsProduction())
    {
        app.UseExceptionHandler("/error");
        app.UseHsts();
    }

    app.UseHttpsRedirection();
    app.UseRouting();
    app.UseAuthorization();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

**Establecer ambiente:**
```bash
# Linux/macOS
export ASPNETCORE_ENVIRONMENT=Production

# Windows
set ASPNETCORE_ENVIRONMENT=Production
```

**En launchSettings.json:**
```json
{
  "profiles": {
    "ProductsApi": {
      "commandName": "Project",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

---

## 2.5 Middleware y Pipeline de Requests

### Concepto de middleware

Middleware son componentes que procesan requests y responses en un pipeline.

**Flujo:**
```
Request → Middleware 1 → Middleware 2 → Middleware 3 → Endpoint
         ↓               ↓               ↓
Response ← Middleware 1 ← Middleware 2 ← Middleware 3 ←
```

### Orden de ejecución del pipeline

**Ejemplo en Startup.cs:**
```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }

    app.UseHttpsRedirection();  // 1. Redirección HTTPS
    app.UseCors();              // 2. CORS
    app.UseRouting();           // 3. Routing
    app.UseAuthentication();    // 4. Autenticación
    app.UseAuthorization();     // 5. Autorización

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers(); // 6. Endpoints
    });
}
```

### Middleware personalizado

**Ejemplo simple:**
```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Http;

namespace ProductsApi.Middleware
{
    public class RequestLoggingMiddleware
    {
        private readonly RequestDelegate _next;

        public RequestLoggingMiddleware(RequestDelegate next)
        {
            _next = next;
        }

        public async Task InvokeAsync(HttpContext context)
        {
            // Antes del request
            Console.WriteLine($"Request: {context.Request.Method} {context.Request.Path}");
            
            await _next(context); // Llamar al siguiente middleware
            
            // Después del response
            Console.WriteLine($"Response: {context.Response.StatusCode}");
        }
    }
}
```

**Extension method:**
```csharp
using Microsoft.AspNetCore.Builder;

namespace ProductsApi.Middleware
{
    public static class MiddlewareExtensions
    {
        public static IApplicationBuilder UseRequestLogging(
            this IApplicationBuilder app)
        {
            return app.UseMiddleware<RequestLoggingMiddleware>();
        }
    }
}
```

**Registrar en Startup.cs:**
```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    app.UseRequestLogging(); // Usar middleware personalizado
    
    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

### Manejo de excepciones

**Middleware de manejo de errores en Startup.cs:**
```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage(); // Página detallada de errores
    }
    else
    {
        app.UseExceptionHandler("/error"); // Endpoint de error personalizado
        app.UseHsts();
    }

    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

**Middleware personalizado de excepciones:**
```csharp
using System;
using System.Net;
using System.Text.Json;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Http;

namespace ProductsApi.Middleware
{
    public class ExceptionHandlingMiddleware
    {
        private readonly RequestDelegate _next;

        public ExceptionHandlingMiddleware(RequestDelegate next)
        {
            _next = next;
        }

        public async Task InvokeAsync(HttpContext context)
        {
            try
            {
                await _next(context);
            }
            catch (Exception ex)
            {
                await HandleExceptionAsync(context, ex);
            }
        }

        private static Task HandleExceptionAsync(HttpContext context, Exception exception)
        {
            context.Response.ContentType = "application/json";
            context.Response.StatusCode = (int)HttpStatusCode.InternalServerError;

            var response = new
            {
                error = "Error interno del servidor",
                message = exception.Message
            };

            var json = JsonSerializer.Serialize(response);
            return context.Response.WriteAsync(json);
        }
    }
}
```

**Usar en Startup.cs:**
```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    app.UseMiddleware<ExceptionHandlingMiddleware>();
    
    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

### CORS (Cross-Origin Resource Sharing)

**Configuración en Startup.cs:**
```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Configurar política de CORS
    services.AddCors(options =>
    {
        options.AddPolicy("AllowFrontend", policy =>
        {
            policy.WithOrigins("http://localhost:3000") // React app
                  .AllowAnyMethod()
                  .AllowAnyHeader();
        });
        
        // Política más permisiva (solo desarrollo)
        options.AddPolicy("AllowAll", policy =>
        {
            policy.AllowAnyOrigin()
                  .AllowAnyMethod()
                  .AllowAnyHeader();
        });
    });

    services.AddControllers();
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    app.UseRouting();
    
    app.UseCors("AllowFrontend"); // Antes de UseEndpoints
    
    app.UseAuthentication();
    app.UseAuthorization();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

**CORS en un controller específico:**
```csharp
using Microsoft.AspNetCore.Cors;
using Microsoft.AspNetCore.Mvc;

namespace ProductsApi.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    [EnableCors("AllowFrontend")]
    public class ProductsController : ControllerBase
    {
        [HttpGet]
        public IActionResult GetAll()
        {
            return Ok(new[] { "Producto 1", "Producto 2" });
        }
    }
}
```

---

## 2.6 Logging en .NET Core

### ILogger interface

**Inyección y uso básico:**
```csharp
using Microsoft.Extensions.Logging;

namespace ProductsApi.Services
{
    public class ProductService
    {
        private readonly ILogger<ProductService> _logger;

        public ProductService(ILogger<ProductService> logger)
        {
            _logger = logger;
        }

        public Product GetProduct(int id)
        {
            _logger.LogInformation("Obteniendo producto con ID: {ProductId}", id);
            
            try
            {
                // Lógica de negocio
                return new Product { Id = id, Name = "Laptop" };
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error al obtener producto {ProductId}", id);
                throw;
            }
        }
    }
}
```

### Niveles de logging

```csharp
_logger.LogTrace("Información de depuración muy detallada");
_logger.LogDebug("Información útil para debugging");
_logger.LogInformation("Flujo normal de la aplicación");
_logger.LogWarning("Situación inesperada pero no crítica");
_logger.LogError(exception, "Error recuperable");
_logger.LogCritical(exception, "Error fatal");
```

**Configuración en appsettings.json:**
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information",
      "ProductsApi.Services.ProductService": "Debug"
    }
  }
}
```

### Proveedores de logging

**Configuración en Program.cs:**
```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureLogging(logging =>
            {
                logging.ClearProviders(); // Limpiar proveedores por defecto
                logging.AddConsole();     // Console provider
                logging.AddDebug();       // Debug provider
                logging.AddEventLog();    // EventLog (Windows)
            })
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

**Logging con Serilog:**
```bash
# Instalar paquetes
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
```

**Configurar Serilog en Program.cs:**
```csharp
using Serilog;
using Serilog.Events;

public class Program
{
    public static void Main(string[] args)
    {
        // Configurar Serilog
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Debug()
            .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
            .Enrich.FromLogContext()
            .WriteTo.Console()
            .WriteTo.File("logs/log-.txt", 
                rollingInterval: RollingInterval.Day,
                retainedFileCountLimit: 7)
            .CreateLogger();

        try
        {
            Log.Information("Iniciando aplicación");
            CreateHostBuilder(args).Build().Run();
        }
        catch (Exception ex)
        {
            Log.Fatal(ex, "La aplicación falló al iniciar");
        }
        finally
        {
            Log.CloseAndFlush();
        }
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .UseSerilog() // Usar Serilog
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

**Configurar Serilog desde appsettings.json:**
```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console"
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/log-.txt",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 7
        }
      }
    ]
  }
}
```

**Program.cs con configuración desde appsettings:**
```csharp
using Serilog;

public class Program
{
    public static void Main(string[] args)
    {
        var configuration = new ConfigurationBuilder()
            .AddJsonFile("appsettings.json")
            .Build();

        Log.Logger = new LoggerConfiguration()
            .ReadFrom.Configuration(configuration)
            .CreateLogger();

        try
        {
            Log.Information("Iniciando aplicación");
            CreateHostBuilder(args).Build().Run();
        }
        finally
        {
            Log.CloseAndFlush();
        }
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .UseSerilog()
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

### Structured logging

**Logging con propiedades:**
```csharp
_logger.LogInformation(
    "Usuario {UserId} realizó pedido {OrderId} por {Amount}",
    userId,
    orderId,
    amount
);

// Salida estructurada:
// Usuario 123 realizó pedido 456 por 99.99
// Con propiedades: UserId=123, OrderId=456, Amount=99.99
```

**Con Serilog (serialización de objetos):**
```csharp
Log.Information("Procesando pedido {@Order}", order);
// @ serializa todo el objeto en JSON
```

### Logging en producción

**appsettings.Production.json:**
```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "File",
        "Args": {
          "path": "/var/log/productsapi/log-.txt",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 30,
          "fileSizeLimitBytes": 10485760,
          "rollOnFileSizeLimit": true
        }
      }
    ]
  }
}
```

**Buenas prácticas:**
```csharp
// ✅ Bueno: Structured logging
_logger.LogInformation("Pedido {OrderId} completado en {Duration}ms", 
    orderId, duration);

// ❌ Malo: Concatenación de strings
_logger.LogInformation("Pedido " + orderId + " completado");

// ✅ Bueno: Loguear contexto
_logger.LogWarning("Stock bajo para producto {ProductId}. Stock actual: {Stock}", 
    productId, currentStock);

// ❌ Malo: Mensaje genérico
_logger.LogWarning("Stock bajo");

// ✅ Bueno: Usar niveles apropiados
_logger.LogError(ex, "Error al procesar pedido {OrderId}", orderId);

// ❌ Malo: Loguear excepciones como información
_logger.LogInformation("Error: " + ex.Message);
```

---

## Ejemplo Completo: API de Productos con .NET Core

Aquí un ejemplo que integra todos los conceptos del tema:

### Estructura del proyecto

```
ProductsApi/
├── Controllers/
│   └── ProductsController.cs
├── Models/
│   └── Product.cs
├── Services/
│   ├── IProductService.cs
│   └── ProductService.cs
├── Repositories/
│   ├── IProductRepository.cs
│   └── ProductRepository.cs
├── Configuration/
│   └── ProductSettings.cs
├── Middleware/
│   └── RequestLoggingMiddleware.cs
├── Program.cs
├── Startup.cs
└── appsettings.json
```

### Program.cs
```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Hosting;
using Serilog;

namespace ProductsApi
{
    public class Program
    {
        public static void Main(string[] args)
        {
            // Configurar Serilog
            Log.Logger = new LoggerConfiguration()
                .WriteTo.Console()
                .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day)
                .CreateLogger();

            try
            {
                Log.Information("Iniciando ProductsApi");
                CreateHostBuilder(args).Build().Run();
            }
            catch (Exception ex)
            {
                Log.Fatal(ex, "La aplicación falló al iniciar");
            }
            finally
            {
                Log.CloseAndFlush();
            }
        }

        public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .UseSerilog()
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder.UseStartup<Startup>();
                });
    }
}
```

### Startup.cs
```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.OpenApi.Models;
using ProductsApi.Configuration;
using ProductsApi.Middleware;
using ProductsApi.Repositories;
using ProductsApi.Services;

namespace ProductsApi
{
    public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        public void ConfigureServices(IServiceCollection services)
        {
            // Configuración con Options Pattern
            services.Configure<ProductSettings>(
                Configuration.GetSection("ProductSettings"));

            // CORS
            services.AddCors(options =>
            {
                options.AddPolicy("AllowAll", policy =>
                    policy.AllowAnyOrigin()
                          .AllowAnyMethod()
                          .AllowAnyHeader());
            });

            // Dependency Injection
            services.AddScoped<IProductRepository, ProductRepository>();
            services.AddScoped<IProductService, ProductService>();

            // Controllers
            services.AddControllers()
                .AddJsonOptions(options =>
                {
                    options.JsonSerializerOptions.PropertyNamingPolicy = null;
                    options.JsonSerializerOptions.WriteIndented = true;
                });

            // Swagger
            services.AddSwaggerGen(c =>
            {
                c.SwaggerDoc("v1", new OpenApiInfo 
                { 
                    Title = "Products API", 
                    Version = "v1" 
                });
            });
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
                app.UseSwagger();
                app.UseSwaggerUI(c => c.SwaggerEndpoint("/swagger/v1/swagger.json", "Products API v1"));
            }

            // Middleware personalizado
            app.UseMiddleware<RequestLoggingMiddleware>();

            app.UseHttpsRedirection();
            app.UseRouting();
            app.UseCors("AllowAll");
            app.UseAuthorization();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
            });
        }
    }
}
```

### appsettings.json
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning"
    }
  },
  "ProductSettings": {
    "MaxPageSize": 100,
    "CacheDurationMinutes": 10
  },
  "AllowedHosts": "*"
}
```

### Configuration/ProductSettings.cs
```csharp
namespace ProductsApi.Configuration
{
    public class ProductSettings
    {
        public int MaxPageSize { get; set; }
        public int CacheDurationMinutes { get; set; }
    }
}
```

### Models/Product.cs
```csharp
using System.ComponentModel.DataAnnotations;

namespace ProductsApi.Models
{
    public class Product
    {
        public int Id { get; set; }
        
        [Required(ErrorMessage = "El nombre es obligatorio")]
        [StringLength(100, MinimumLength = 3)]
        public string Name { get; set; }
        
        [Range(0.01, 100000, ErrorMessage = "El precio debe estar entre 0.01 y 100000")]
        public decimal Price { get; set; }
        
        [StringLength(500)]
        public string Description { get; set; }
    }
}
```

### Repositories/IProductRepository.cs
```csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using ProductsApi.Models;

namespace ProductsApi.Repositories
{
    public interface IProductRepository
    {
        Task<IEnumerable<Product>> GetAllAsync();
        Task<Product> GetByIdAsync(int id);
        Task<Product> CreateAsync(Product product);
        Task<bool> UpdateAsync(Product product);
        Task<bool> DeleteAsync(int id);
    }
}
```

### Repositories/ProductRepository.cs
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.Extensions.Logging;
using ProductsApi.Models;

namespace ProductsApi.Repositories
{
    public class ProductRepository : IProductRepository
    {
        private readonly ILogger<ProductRepository> _logger;
        private static readonly List<Product> _products = new List<Product>
        {
            new Product { Id = 1, Name = "Laptop", Price = 999.99m, Description = "Portátil de alto rendimiento" },
            new Product { Id = 2, Name = "Mouse", Price = 29.99m, Description = "Mouse inalámbrico" },
            new Product { Id = 3, Name = "Teclado", Price = 79.99m, Description = "Teclado mecánico" }
        };

        public ProductRepository(ILogger<ProductRepository> logger)
        {
            _logger = logger;
        }

        public Task<IEnumerable<Product>> GetAllAsync()
        {
            _logger.LogInformation("Obteniendo todos los productos. Total: {Count}", _products.Count);
            return Task.FromResult<IEnumerable<Product>>(_products);
        }

        public Task<Product> GetByIdAsync(int id)
        {
            _logger.LogInformation("Buscando producto con ID: {ProductId}", id);
            var product = _products.FirstOrDefault(p => p.Id == id);
            
            if (product == null)
            {
                _logger.LogWarning("Producto {ProductId} no encontrado", id);
            }
            
            return Task.FromResult(product);
        }

        public Task<Product> CreateAsync(Product product)
        {
            product.Id = _products.Any() ? _products.Max(p => p.Id) + 1 : 1;
            _products.Add(product);
            _logger.LogInformation("Producto creado: {ProductId} - {ProductName}", product.Id, product.Name);
            return Task.FromResult(product);
        }

        public Task<bool> UpdateAsync(Product product)
        {
            var existing = _products.FirstOrDefault(p => p.Id == product.Id);
            if (existing == null)
            {
                _logger.LogWarning("Intento de actualizar producto inexistente: {ProductId}", product.Id);
                return Task.FromResult(false);
            }

            existing.Name = product.Name;
            existing.Price = product.Price;
            existing.Description = product.Description;
            
            _logger.LogInformation("Producto actualizado: {ProductId}", product.Id);
            return Task.FromResult(true);
        }

        public Task<bool> DeleteAsync(int id)
        {
            var product = _products.FirstOrDefault(p => p.Id == id);
            if (product == null)
            {
                _logger.LogWarning("Intento de eliminar producto inexistente: {ProductId}", id);
                return Task.FromResult(false);
            }

            _products.Remove(product);
            _logger.LogInformation("Producto eliminado: {ProductId}", id);
            return Task.FromResult(true);
        }
    }
}
```

### Services/IProductService.cs
```csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using ProductsApi.Models;

namespace ProductsApi.Services
{
    public interface IProductService
    {
        Task<IEnumerable<Product>> GetAllProductsAsync();
        Task<Product> GetProductByIdAsync(int id);
        Task<Product> CreateProductAsync(Product product);
    }
}
```

### Services/ProductService.cs
```csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using Microsoft.Extensions.Logging;
using ProductsApi.Models;
using ProductsApi.Repositories;

namespace ProductsApi.Services
{
    public class ProductService : IProductService
    {
        private readonly IProductRepository _repository;
        private readonly ILogger<ProductService> _logger;

        public ProductService(IProductRepository repository, ILogger<ProductService> logger)
        {
            _repository = repository;
            _logger = logger;
        }

        public async Task<IEnumerable<Product>> GetAllProductsAsync()
        {
            _logger.LogInformation("Servicio: Obteniendo todos los productos");
            return await _repository.GetAllAsync();
        }

        public async Task<Product> GetProductByIdAsync(int id)
        {
            _logger.LogInformation("Servicio: Obteniendo producto {ProductId}", id);
            return await _repository.GetByIdAsync(id);
        }

        public async Task<Product> CreateProductAsync(Product product)
        {
            _logger.LogInformation("Servicio: Creando nuevo producto {ProductName}", product.Name);
            return await _repository.CreateAsync(product);
        }
    }
}
```

### Middleware/RequestLoggingMiddleware.cs
```csharp
using System;
using System.Diagnostics;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;

namespace ProductsApi.Middleware
{
    public class RequestLoggingMiddleware
    {
        private readonly RequestDelegate _next;
        private readonly ILogger<RequestLoggingMiddleware> _logger;

        public RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
        {
            _next = next;
            _logger = logger;
        }

        public async Task InvokeAsync(HttpContext context)
        {
            var stopwatch = Stopwatch.StartNew();
            
            _logger.LogInformation("Inicio Request: {Method} {Path}", 
                context.Request.Method, 
                context.Request.Path);

            await _next(context);

            stopwatch.Stop();
            
            _logger.LogInformation("Fin Request: {Method} {Path} - Status: {StatusCode} - Duración: {Duration}ms",
                context.Request.Method,
                context.Request.Path,
                context.Response.StatusCode,
                stopwatch.ElapsedMilliseconds);
        }
    }
}
```

### Controllers/ProductsController.cs
```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;
using ProductsApi.Models;
using ProductsApi.Services;

namespace ProductsApi.Controllers
{
    [ApiController]
    [Route("api/v1/[controller]")]
    public class ProductsController : ControllerBase
    {
        private readonly IProductService _productService;
        private readonly ILogger<ProductsController> _logger;

        public ProductsController(IProductService productService, ILogger<ProductsController> logger)
        {
            _productService = productService;
            _logger = logger;
        }

        /// <summary>
        /// Obtiene todos los productos
        /// </summary>
        [HttpGet]
        public async Task<IActionResult> GetAll()
        {
            var products = await _productService.GetAllProductsAsync();
            return Ok(products);
        }

        /// <summary>
        /// Obtiene un producto por ID
        /// </summary>
        [HttpGet("{id}")]
        public async Task<IActionResult> GetById(int id)
        {
            var product = await _productService.GetProductByIdAsync(id);
            
            if (product == null)
            {
                _logger.LogWarning("Controller: Producto {ProductId} no encontrado", id);
                return NotFound(new { message = $"Producto con ID {id} no encontrado" });
            }

            return Ok(product);
        }

        /// <summary>
        /// Crea un nuevo producto
        /// </summary>
        [HttpPost]
        public async Task<IActionResult> Create([FromBody] Product product)
        {
            if (!ModelState.IsValid)
            {
                return BadRequest(ModelState);
            }

            var created = await _productService.CreateProductAsync(product);
            return CreatedAtAction(nameof(GetById), new { id = created.Id }, created);
        }
    }
}
```

### Probar la API

```bash
# Compilar
dotnet build

# Ejecutar
dotnet run

# Probar endpoints
curl http://localhost:5000/api/v1/products
curl http://localhost:5000/api/v1/products/1
curl -X POST http://localhost:5000/api/v1/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Monitor","price":299.99,"description":"Monitor 27 pulgadas"}'
```

---

## Resumen del Tema 2

En este tema hemos cubierto:

1. **Historia y evolución de .NET** - De Framework a Core y .NET moderno
2. **ASP.NET Core Web API** - Creación, estructura, routing y versionado
3. **Dependency Injection** - IoC, ciclos de vida y buenas prácticas
4. **Configuración** - appsettings, variables de entorno y Options Pattern
5. **Middleware** - Pipeline de requests, CORS y manejo de errores
6. **Logging** - Niveles, proveedores y structured logging

Estos son los fundamentos esenciales para construir microservicios robustos con .NET.

---

**Duración estimada:** 3 horas  
**Siguiente tema:** Desarrollo de Microservicios con .NET
