# Laboratorio: Configuración de CORS en ASP.NET Core 8.0

## Objetivo

Aprender a configurar y probar CORS (Cross-Origin Resource Sharing) en una aplicación ASP.NET Core 8.0 para permitir peticiones desde diferentes orígenes.

## Requisitos Previos

- .NET 8.0 SDK instalado
- Visual Studio Code o Visual Studio 2022
- Conocimientos básicos de C# y ASP.NET Core
- Navegador web moderno (Chrome, Firefox, Edge)

---

## Parte 1: Creación del Proyecto Backend

### Paso 1: Crear la API

Abre una terminal y ejecuta los siguientes comandos:

```bash
dotnet new webapi -n CorsLabApi
cd CorsLabApi
```

### Paso 2: Configurar CORS en Program.cs

Abre el archivo `Program.cs` y reemplaza su contenido con el siguiente código:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Agregar servicios al contenedor
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// ========== CONFIGURACIÓN DE CORS ==========

// Opción 1: Política permisiva (Solo para desarrollo)
builder.Services.AddCors(options =>
{
    options.AddPolicy("PermitirTodo", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

// Opción 2: Política restrictiva (Recomendada para producción)
builder.Services.AddCors(options =>
{
    options.AddPolicy("PoliticaRestrictiva", policy =>
    {
        policy.WithOrigins("http://localhost:3000", "http://localhost:5500")
              .WithMethods("GET", "POST", "PUT", "DELETE")
              .WithHeaders("Content-Type", "Authorization")
              .AllowCredentials();
    });
});

// Opción 3: Política personalizada con wildcard
builder.Services.AddCors(options =>
{
    options.AddPolicy("SubdominiosPermitidos", policy =>
    {
        policy.WithOrigins("https://*.midominio.com")
              .SetIsOriginAllowedToAllowWildcardSubdomains()
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

// Opción 4: Política por defecto
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.WithOrigins("http://localhost:3000")
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

var app = builder.Build();

// Configurar el pipeline HTTP
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

// ========== APLICAR CORS ==========
// IMPORTANTE: Debe ir ANTES de UseAuthorization
app.UseCors("PoliticaRestrictiva"); // Aplicar política específica

app.UseAuthorization();

app.MapControllers();

app.Run();
```

### Paso 3: Crear un Controlador de Prueba

Crea un nuevo archivo `Controllers/ProductosController.cs`:

```csharp
using Microsoft.AspNetCore.Cors;
using Microsoft.AspNetCore.Mvc;

namespace CorsLabApi.Controllers;

[ApiController]
[Route("api/[controller]")]
public class ProductosController : ControllerBase
{
    private static List<Producto> productos = new()
    {
        new Producto { Id = 1, Nombre = "Laptop", Precio = 1200.00m },
        new Producto { Id = 2, Nombre = "Mouse", Precio = 25.00m },
        new Producto { Id = 3, Nombre = "Teclado", Precio = 75.00m }
    };

    // Endpoint sin restricción CORS (usa la política global)
    [HttpGet]
    public ActionResult<IEnumerable<Producto>> GetAll()
    {
        return Ok(productos);
    }

    // Endpoint con política CORS específica
    [HttpGet("{id}")]
    [EnableCors("PermitirTodo")]
    public ActionResult<Producto> GetById(int id)
    {
        var producto = productos.FirstOrDefault(p => p.Id == id);
        if (producto == null)
            return NotFound();
        
        return Ok(producto);
    }

    // Endpoint que acepta POST
    [HttpPost]
    public ActionResult<Producto> Create([FromBody] Producto producto)
    {
        producto.Id = productos.Max(p => p.Id) + 1;
        productos.Add(producto);
        return CreatedAtAction(nameof(GetById), new { id = producto.Id }, producto);
    }

    // Endpoint PUT para actualizar
    [HttpPut("{id}")]
    public ActionResult<Producto> Update(int id, [FromBody] Producto producto)
    {
        var productoExistente = productos.FirstOrDefault(p => p.Id == id);
        if (productoExistente == null)
            return NotFound();
        
        productoExistente.Nombre = producto.Nombre;
        productoExistente.Precio = producto.Precio;
        
        return Ok(productoExistente);
    }

    // Endpoint sin CORS (para probar bloqueo)
    [HttpDelete("{id}")]
    [DisableCors]
    public ActionResult Delete(int id)
    {
        var producto = productos.FirstOrDefault(p => p.Id == id);
        if (producto == null)
            return NotFound();
        
        productos.Remove(producto);
        return NoContent();
    }
}

public class Producto
{
    public int Id { get; set; }
    public string Nombre { get; set; } = string.Empty;
    public decimal Precio { get; set; }
}
```

### Paso 4: Ejecutar la API

En la terminal, dentro del directorio del proyecto, ejecuta:

```bash
dotnet run
```

La API estará disponible en:
- HTTP: `http://localhost:5000`
- HTTPS: `https://localhost:5001`

**Nota:** Toma nota de la URL exacta que aparece en la consola.

---

## Parte 2: Crear Cliente HTML/JavaScript

Crea un archivo `cliente.html` en cualquier carpeta (fuera del proyecto de la API):

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Cliente CORS Lab</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 20px;
        }
        
        .container {
            max-width: 900px;
            margin: 0 auto;
            background: white;
            border-radius: 15px;
            box-shadow: 0 20px 60px rgba(0,0,0,0.3);
            padding: 30px;
        }
        
        h1 {
            color: #333;
            margin-bottom: 10px;
            font-size: 28px;
        }
        
        .subtitle {
            color: #666;
            margin-bottom: 30px;
            font-size: 14px;
        }
        
        .config-section {
            background: #f8f9fa;
            padding: 20px;
            border-radius: 10px;
            margin-bottom: 25px;
        }
        
        .config-section h2 {
            color: #495057;
            font-size: 18px;
            margin-bottom: 15px;
        }
        
        label {
            display: block;
            margin-bottom: 8px;
            color: #495057;
            font-weight: 600;
        }
        
        input[type="text"] {
            width: 100%;
            padding: 12px;
            border: 2px solid #dee2e6;
            border-radius: 8px;
            font-size: 14px;
            transition: border-color 0.3s;
        }
        
        input[type="text"]:focus {
            outline: none;
            border-color: #667eea;
        }
        
        .tests-section {
            margin-bottom: 25px;
        }
        
        .tests-section h2 {
            color: #495057;
            font-size: 18px;
            margin-bottom: 15px;
        }
        
        .button-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 10px;
        }
        
        button {
            padding: 15px 20px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-size: 14px;
            font-weight: 600;
            transition: transform 0.2s, box-shadow 0.2s;
        }
        
        button:hover {
            transform: translateY(-2px);
            box-shadow: 0 5px 15px rgba(102, 126, 234, 0.4);
        }
        
        button:active {
            transform: translateY(0);
        }
        
        #resultado {
            display: none;
            margin-top: 25px;
            padding: 20px;
            border-radius: 10px;
            animation: slideIn 0.3s ease-out;
        }
        
        @keyframes slideIn {
            from {
                opacity: 0;
                transform: translateY(-10px);
            }
            to {
                opacity: 1;
                transform: translateY(0);
            }
        }
        
        .resultado {
            background-color: #d1ecf1;
            border-left: 4px solid #0c5460;
        }
        
        .error {
            background-color: #f8d7da;
            border-left: 4px solid #721c24;
        }
        
        .resultado-header {
            font-weight: 600;
            margin-bottom: 10px;
            font-size: 16px;
        }
        
        pre {
            background-color: #272822;
            color: #f8f8f2;
            padding: 15px;
            border-radius: 8px;
            overflow-x: auto;
            font-family: 'Courier New', monospace;
            font-size: 13px;
            line-height: 1.5;
        }
        
        .info-box {
            background: #e7f3ff;
            border-left: 4px solid #2196F3;
            padding: 15px;
            border-radius: 8px;
            margin-top: 20px;
        }
        
        .info-box strong {
            color: #1976D2;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🌐 Laboratorio CORS</h1>
        <p class="subtitle">Cliente de Prueba - ASP.NET Core 8.0</p>
        
        <div class="config-section">
            <h2>⚙️ Configuración</h2>
            <label for="apiUrl">URL de la API:</label>
            <input type="text" id="apiUrl" value="https://localhost:5001/api/productos" 
                   placeholder="https://localhost:5001/api/productos">
        </div>

        <div class="tests-section">
            <h2>🧪 Pruebas de CORS</h2>
            <div class="button-grid">
                <button onclick="obtenerProductos()">📋 GET - Lista Completa</button>
                <button onclick="obtenerProductoPorId()">🔍 GET - Por ID</button>
                <button onclick="crearProducto()">➕ POST - Crear Producto</button>
                <button onclick="actualizarProducto()">✏️ PUT - Actualizar</button>
                <button onclick="eliminarProducto()">❌ DELETE - Sin CORS</button>
                <button onclick="requestConCredenciales()">🔐 GET - Con Credenciales</button>
                <button onclick="requestConHeaderCustom()">📨 GET - Header Custom</button>
                <button onclick="limpiarResultado()">🧹 Limpiar</button>
            </div>
        </div>

        <div id="resultado"></div>

        <div class="info-box">
            <strong>💡 Instrucciones:</strong>
            <ul style="margin-left: 20px; margin-top: 10px; line-height: 1.8;">
                <li>Asegúrate de que la API esté corriendo</li>
                <li>Actualiza la URL si es necesario</li>
                <li>Abre la consola del navegador (F12) para ver detalles de CORS</li>
                <li>El botón DELETE debería fallar por la política [DisableCors]</li>
            </ul>
        </div>
    </div>

    <script>
        function mostrarResultado(titulo, mensaje, esError = false) {
            const div = document.getElementById('resultado');
            div.style.display = 'block';
            div.className = esError ? 'error' : 'resultado';
            div.innerHTML = `
                <div class="resultado-header">${esError ? '❌' : '✅'} ${titulo}</div>
                <pre>${mensaje}</pre>
            `;
        }

        function limpiarResultado() {
            const div = document.getElementById('resultado');
            div.style.display = 'none';
        }

        async function obtenerProductos() {
            try {
                const url = document.getElementById('apiUrl').value;
                console.log('🔄 Realizando petición GET a:', url);
                
                const response = await fetch(url);
                
                console.log('📥 Respuesta recibida:', response.status, response.statusText);
                
                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`);
                }
                
                const data = await response.json();
                mostrarResultado(
                    'Productos Obtenidos',
                    JSON.stringify(data, null, 2)
                );
            } catch (error) {
                console.error('❌ Error:', error);
                mostrarResultado('Error al obtener productos', error.message, true);
            }
        }

        async function obtenerProductoPorId() {
            try {
                const url = document.getElementById('apiUrl').value;
                const urlConId = `${url}/1`;
                
                console.log('🔄 Realizando petición GET a:', urlConId);
                
                const response = await fetch(urlConId);
                
                console.log('📥 Respuesta recibida:', response.status, response.statusText);
                
                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`);
                }
                
                const data = await response.json();
                mostrarResultado(
                    'Producto Individual (usa política PermitirTodo)',
                    JSON.stringify(data, null, 2)
                );
            } catch (error) {
                console.error('❌ Error:', error);
                mostrarResultado('Error al obtener producto', error.message, true);
            }
        }

        async function crearProducto() {
            try {
                const url = document.getElementById('apiUrl').value;
                const nuevoProducto = {
                    nombre: "Webcam HD",
                    precio: 89.99
                };

                console.log('🔄 Realizando petición POST a:', url);
                console.log('📤 Datos enviados:', nuevoProducto);

                const response = await fetch(url, {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json'
                    },
                    body: JSON.stringify(nuevoProducto)
                });
                
                console.log('📥 Respuesta recibida:', response.status, response.statusText);
                
                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`);
                }
                
                const data = await response.json();
                mostrarResultado(
                    'Producto Creado',
                    `Solicitud POST exitosa\n\n${JSON.stringify(data, null, 2)}`
                );
            } catch (error) {
                console.error('❌ Error:', error);
                mostrarResultado('Error al crear producto', error.message, true);
            }
        }

        async function actualizarProducto() {
            try {
                const url = document.getElementById('apiUrl').value;
                const productoActualizado = {
                    nombre: "Laptop Gaming",
                    precio: 1599.99
                };

                console.log('🔄 Realizando petición PUT a:', `${url}/1`);
                console.log('📤 Datos enviados:', productoActualizado);

                const response = await fetch(`${url}/1`, {
                    method: 'PUT',
                    headers: {
                        'Content-Type': 'application/json'
                    },
                    body: JSON.stringify(productoActualizado)
                });
                
                console.log('📥 Respuesta recibida:', response.status, response.statusText);
                
                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`);
                }
                
                const data = await response.json();
                mostrarResultado(
                    'Producto Actualizado',
                    `Solicitud PUT exitosa\n\n${JSON.stringify(data, null, 2)}`
                );
            } catch (error) {
                console.error('❌ Error:', error);
                mostrarResultado('Error al actualizar producto', error.message, true);
            }
        }

        async function eliminarProducto() {
            try {
                const url = document.getElementById('apiUrl').value;
                
                console.log('🔄 Realizando petición DELETE a:', `${url}/1`);
                console.log('⚠️ Este endpoint tiene [DisableCors] - debería fallar');
                
                const response = await fetch(`${url}/1`, {
                    method: 'DELETE'
                });
                
                console.log('📥 Respuesta recibida:', response.status, response.statusText);
                
                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`);
                }
                
                mostrarResultado(
                    'Producto Eliminado',
                    'Si ves esto, el CORS no está bloqueando correctamente'
                );
            } catch (error) {
                console.error('❌ Error CORS esperado:', error);
                mostrarResultado(
                    'Error CORS Esperado',
                    `Este endpoint tiene [DisableCors] aplicado\n\nError: ${error.message}\n\nRevisa la consola del navegador para ver los detalles de CORS`,
                    true
                );
            }
        }

        async function requestConCredenciales() {
            try {
                const url = document.getElementById('apiUrl').value;
                
                console.log('🔄 Realizando petición con credenciales a:', url);
                
                const response = await fetch(url, {
                    method: 'GET',
                    credentials: 'include', // Incluir cookies/credenciales
                    headers: {
                        'Content-Type': 'application/json'
                    }
                });
                
                console.log('📥 Respuesta recibida:', response.status, response.statusText);
                
                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`);
                }
                
                const data = await response.json();
                mostrarResultado(
                    'Petición con Credenciales',
                    `Esta petición incluye credentials: 'include'\n\n${JSON.stringify(data, null, 2)}`
                );
            } catch (error) {
                console.error('❌ Error:', error);
                mostrarResultado(
                    'Error con credenciales',
                    `${error.message}\n\nNota: AllowCredentials() requiere orígenes específicos, no AllowAnyOrigin()`,
                    true
                );
            }
        }

        async function requestConHeaderCustom() {
            try {
                const url = document.getElementById('apiUrl').value;
                
                console.log('🔄 Realizando petición con header personalizado a:', url);
                
                const response = await fetch(url, {
                    method: 'GET',
                    headers: {
                        'X-Custom-Header': 'ValorPersonalizado',
                        'Content-Type': 'application/json'
                    }
                });
                
                console.log('📥 Respuesta recibida:', response.status, response.statusText);
                
                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`);
                }
                
                const data = await response.json();
                mostrarResultado(
                    'Petición con Header Custom',
                    `Header enviado: X-Custom-Header: ValorPersonalizado\n\nSi falla, verifica WithHeaders() en la política CORS\n\n${JSON.stringify(data, null, 2)}`
                );
            } catch (error) {
                console.error('❌ Error:', error);
                mostrarResultado(
                    'Error con header custom',
                    `${error.message}\n\nEl header 'X-Custom-Header' debe estar permitido en WithHeaders()`,
                    true
                );
            }
        }

        // Mensaje inicial
        console.log('%c🌐 Cliente CORS Lab iniciado', 'color: #667eea; font-size: 16px; font-weight: bold;');
        console.log('%cAbre las DevTools Network para ver las peticiones OPTIONS (preflight)', 'color: #666; font-size: 12px;');
    </script>
</body>
</html>
```

---

## Parte 3: Escenarios de Prueba

### Escenario 1: Servidor Local con Live Server

1. Instala la extensión "Live Server" en Visual Studio Code
2. Haz clic derecho en `cliente.html` y selecciona "Open with Live Server"
3. El cliente se abrirá en `http://localhost:5500` (o puerto similar)
4. Prueba las diferentes peticiones y observa los resultados

### Escenario 2: Prueba desde Origen No Permitido

1. Cambia el puerto del Live Server o sirve el HTML desde otro servidor
2. Observa los errores CORS en la consola del navegador
3. Modifica la política CORS en `Program.cs` para agregar el nuevo origen

### Escenario 3: Headers Personalizados

El botón "GET - Header Custom" envía un header no estándar. Para que funcione:

1. Modifica la política en `Program.cs`:

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("PoliticaRestrictiva", policy =>
    {
        policy.WithOrigins("http://localhost:3000", "http://localhost:5500")
              .WithMethods("GET", "POST", "PUT", "DELETE")
              .WithHeaders("Content-Type", "Authorization", "X-Custom-Header")
              .AllowCredentials();
    });
});
```

2. Reinicia la API
3. Prueba nuevamente el botón

### Escenario 4: Preflight Requests

1. Abre las DevTools del navegador (F12)
2. Ve a la pestaña "Network"
3. Realiza una petición POST o PUT
4. Observa que hay una petición OPTIONS previa (preflight)
5. Examina los headers de la petición OPTIONS

---

## Parte 4: Conceptos Clave

### ¿Qué es CORS?

CORS (Cross-Origin Resource Sharing) es un mecanismo de seguridad del navegador que restringe las peticiones HTTP entre diferentes orígenes. Un origen está compuesto por:
- Protocolo (http/https)
- Dominio (localhost, ejemplo.com)
- Puerto (3000, 5500, 80)

### Simple Request vs Preflight Request

**Simple Request** (no requiere preflight):
- Métodos: GET, HEAD, POST
- Headers permitidos por defecto: Accept, Accept-Language, Content-Language, Content-Type
- Content-Type solo: application/x-www-form-urlencoded, multipart/form-data, text/plain

**Preflight Request** (requiere OPTIONS):
- Cualquier otro método (PUT, DELETE, PATCH)
- Headers personalizados
- Content-Type: application/json

### Métodos de Configuración

#### 1. Política Global

```csharp
app.UseCors("NombrePolitica");
```

#### 2. Por Controlador

```csharp
[EnableCors("NombrePolitica")]
public class MiController : ControllerBase
```

#### 3. Por Acción

```csharp
[HttpGet]
[EnableCors("NombrePolitica")]
public ActionResult MiAccion()
```

#### 4. Deshabilitar CORS

```csharp
[DisableCors]
public ActionResult AccionSinCors()
```

### Opciones Importantes de Configuración

| Método | Descripción | Ejemplo |
|--------|-------------|---------|
| `WithOrigins()` | Orígenes específicos permitidos | `.WithOrigins("http://localhost:3000")` |
| `AllowAnyOrigin()` | Permite cualquier origen | `.AllowAnyOrigin()` |
| `WithMethods()` | Métodos HTTP específicos | `.WithMethods("GET", "POST")` |
| `AllowAnyMethod()` | Permite todos los métodos | `.AllowAnyMethod()` |
| `WithHeaders()` | Headers específicos | `.WithHeaders("Content-Type", "Authorization")` |
| `AllowAnyHeader()` | Permite todos los headers | `.AllowAnyHeader()` |
| `AllowCredentials()` | Permite cookies/auth | `.AllowCredentials()` |
| `SetIsOriginAllowed()` | Validación personalizada | `.SetIsOriginAllowed(origin => true)` |

---

## Parte 5: Ejercicios Propuestos

### Ejercicio 1: Política Solo Lectura

Crea una política CORS que solo permita peticiones GET desde cualquier origen.

**Solución:**

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("SoloLectura", policy =>
    {
        policy.AllowAnyOrigin()
              .WithMethods("GET")
              .AllowAnyHeader();
    });
});
```

### Ejercicio 2: Múltiples Orígenes

Configura una política que permita peticiones desde tu aplicación React (puerto 3000) y Angular (puerto 4200).

**Solución:**

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("FrontendApps", policy =>
    {
        policy.WithOrigins(
                  "http://localhost:3000",  // React
                  "http://localhost:4200"   // Angular
              )
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials();
    });
});
```

### Ejercicio 3: CORS Condicional por Ambiente

Implementa CORS diferente para Development y Production.

**Solución:**

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddCors(options =>
{
    if (builder.Environment.IsDevelopment())
    {
        options.AddDefaultPolicy(policy =>
        {
            policy.AllowAnyOrigin()
                  .AllowAnyMethod()
                  .AllowAnyHeader();
        });
    }
    else
    {
        options.AddDefaultPolicy(policy =>
        {
            policy.WithOrigins("https://miapp.com", "https://www.miapp.com")
                  .WithMethods("GET", "POST", "PUT", "DELETE")
                  .WithHeaders("Content-Type", "Authorization")
                  .AllowCredentials();
        });
    }
});
```

### Ejercicio 4: Validación Dinámica de Orígenes

Crea una política que valide dinámicamente si un origen está permitido usando una lista configurable.

**Solución:**

```csharp
var origenesPermitidos = new[] { "http://localhost:3000", "http://localhost:5500" };

builder.Services.AddCors(options =>
{
    options.AddPolicy("ValidacionDinamica", policy =>
    {
        policy.SetIsOriginAllowed(origin => 
              {
                  return origenesPermitidos.Contains(origin);
              })
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials();
    });
});
```

---

## Parte 6: Problemas Comunes y Soluciones

### Problema 1: "No 'Access-Control-Allow-Origin' header"

**Causa:** CORS no está configurado o el middleware está en el orden incorrecto.

**Solución:**
```csharp
// ✅ Correcto
app.UseCors("MiPolitica");
app.UseAuthorization();

// ❌ Incorrecto
app.UseAuthorization();
app.UseCors("MiPolitica");
```

### Problema 2: Credenciales no Funcionan

**Error:** "The value of the 'Access-Control-Allow-Origin' header cannot be '*' when credentials are included"

**Causa:** No puedes usar `AllowAnyOrigin()` con `AllowCredentials()`.

**Solución:**
```csharp
// ❌ Incorrecto
policy.AllowAnyOrigin()
      .AllowCredentials();

// ✅ Correcto
policy.WithOrigins("http://localhost:3000")
      .AllowCredentials();
```

### Problema 3: Headers Personalizados Bloqueados

**Error:** Header 'X-Custom-Header' is not allowed

**Solución:**
```csharp
policy.WithHeaders("Content-Type", "Authorization", "X-Custom-Header");
```

### Problema 4: OPTIONS Request Falla

**Causa:** El servidor no responde correctamente al preflight.

**Solución:** Asegúrate de que `UseCors()` esté antes de `UseAuthorization()` y que la política permita los métodos necesarios.

### Problema 5: Funciona en Desarrollo pero No en Producción

**Causa:** Probablemente estás usando `AllowAnyOrigin()` en desarrollo.

**Solución:**
```csharp
if (app.Environment.IsDevelopment())
{
    app.UseCors("PermisivaDevelopment");
}
else
{
    app.UseCors("RestrictivaProduction");
}
```

---

## Parte 7: Mejores Prácticas

### ✅ Recomendaciones

1. **Nunca uses `AllowAnyOrigin()` en producción**
2. **Especifica orígenes exactos** con `WithOrigins()`
3. **Limita los métodos** con `WithMethods()`
4. **Restringe headers** con `WithHeaders()`
5. **Usa `AllowCredentials()` solo cuando sea necesario**
6. **Coloca `UseCors()` antes de `UseAuthorization()`**
7. **Aplica políticas específicas por controlador** cuando sea apropiado

### ❌ Evitar

1. ❌ `AllowAnyOrigin()` + `AllowCredentials()`
2. ❌ CORS permisivo en producción
3. ❌ Ignorar los errores CORS sin investigar
4. ❌ Usar wildcards indiscriminadamente
5. ❌ No probar escenarios de preflight

---

## Parte 8: Testing Avanzado

### Usando cURL

```bash
# Preflight request
curl -X OPTIONS https://localhost:5001/api/productos \
  -H "Origin: http://localhost:3000" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: Content-Type" \
  -v

# GET request
curl -X GET https://localhost:5001/api/productos \
  -H "Origin: http://localhost:3000" \
  -v

# POST request
curl -X POST https://localhost:5001/api/productos \
  -H "Origin: http://localhost:3000" \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Test","precio":99.99}' \
  -v
```

### Usando Postman

1. En Postman, las peticiones CORS no se aplican igual que en el navegador
2. Para simular CORS, usa la extensión "Postman Interceptor"
3. O prueba directamente desde el navegador con fetch/axios

---

## Recursos Adicionales

### Documentación Oficial

- [CORS en ASP.NET Core](https://learn.microsoft.com/es-es/aspnet/core/security/cors)
- [MDN Web Docs - CORS](https://developer.mozilla.org/es/docs/Web/HTTP/CORS)

### Headers CORS Importantes

| Header | Descripción |
|--------|-------------|
| `Access-Control-Allow-Origin` | Orígenes permitidos |
| `Access-Control-Allow-Methods` | Métodos HTTP permitidos |
| `Access-Control-Allow-Headers` | Headers personalizados permitidos |
| `Access-Control-Allow-Credentials` | Si se permiten credenciales |
| `Access-Control-Max-Age` | Tiempo de caché del preflight |
| `Access-Control-Expose-Headers` | Headers expuestos al cliente |

---

## Conclusión

CORS es un mecanismo de seguridad esencial para aplicaciones web modernas. Este laboratorio te ha proporcionado:

- ✅ Comprensión de cómo funciona CORS
- ✅ Configuración práctica en ASP.NET Core 8.0
- ✅ Diferentes escenarios y políticas
- ✅ Herramientas para debugging
- ✅ Mejores prácticas de seguridad

**Próximos pasos:**
1. Experimenta con diferentes configuraciones
2. Prueba con aplicaciones frontend reales (React, Angular, Vue)
3. Implementa autenticación JWT con CORS
4. Configura CORS en entornos cloud (Azure, AWS)

---

## Apéndice: Configuración Completa de Producción

```csharp
var builder = WebApplication.CreateBuilder(args);

// Leer orígenes permitidos desde configuración
var origenesPermitidos = builder.Configuration
    .GetSection("CorsOrigins")
    .Get<string[]>() ?? Array.Empty<string>();

builder.Services.AddCors(options =>
{
    options.AddPolicy("ProduccionSegura", policy =>
    {
        policy.WithOrigins(origenesPermitidos)
              .WithMethods("GET", "POST", "PUT", "DELETE")
              .WithHeaders("Content-Type", "Authorization", "X-Api-Key")
              .AllowCredentials()
              .SetPreflightMaxAge(TimeSpan.FromMinutes(10));
    });
});

var app = builder.Build();

app.UseCors("ProduccionSegura");
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

**appsettings.json:**

```json
{
  "CorsOrigins": [
    "https://miapp.com",
    "https://www.miapp.com",
    "https://admin.miapp.com"
  ]
}
```

---

**¡Feliz aprendizaje!** 🚀
