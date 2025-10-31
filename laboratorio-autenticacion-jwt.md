# Laboratorio: Implementación de Autenticación JWT en ASP.NET Core

## Introducción

Este laboratorio te guiará paso a paso para implementar un sistema completo de autenticación basado en JWT (JSON Web Tokens) en tu API, siguiendo la arquitectura de slices vertical que ya tienes implementada.

Al finalizar, tendrás:
- Un endpoint de login que genera tokens JWT
- Protección de endpoints existentes con autorización
- Validación automática de tokens en cada petición

---

## Paso 1: Instalar Paquetes NuGet Necesarios

Abre tu terminal en la raíz del proyecto y ejecuta:

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package System.IdentityModel.Tokens.Jwt
```

---

## Paso 2: Configurar JWT en appsettings.json

Añade la configuración JWT en tu archivo `appsettings.json`:

```json
{
  "Jwt": {
    "Secret": "tu-super-secreto-de-al-menos-32-caracteres-para-mayor-seguridad",
    "Issuer": "webapi",
    "Audience": "webapi-client",
    "ExpirationMinutes": 60
  },
  "ConnectionStrings": {
    // ... tu configuración existente
  }
}
```

> **⚠️ IMPORTANTE**: En producción, almacena el `Secret` en variables de entorno o Azure Key Vault, nunca en el código fuente.

---

## Paso 3: Crear la Configuración JWT

Crea el archivo `webapi/common/configuration/JwtSettings.cs`:

```csharp
namespace webapi.common.configuration;

public class JwtSettings
{
    public string Secret { get; set; } = string.Empty;
    public string Issuer { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;
    public int ExpirationMinutes { get; set; } = 60;
}
```

---

## Paso 4: Crear el Servicio de Generación de Tokens

Crea el archivo `webapi/common/security/JwtTokenGenerator.cs`:

```csharp
using Microsoft.IdentityModel.Tokens;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using webapi.common.configuration;
using webapi.common.dependencyinjection;

namespace webapi.common.security;

public interface IJwtTokenGenerator
{
    string GenerateToken(string username, string userId);
}

[Injectable]
public class JwtTokenGenerator : IJwtTokenGenerator
{
    private readonly JwtSettings _jwtSettings;

    public JwtTokenGenerator(JwtSettings jwtSettings)
    {
        _jwtSettings = jwtSettings;
    }

    public string GenerateToken(string username, string userId)
    {
        var securityKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtSettings.Secret));
        var credentials = new SigningCredentials(securityKey, SecurityAlgorithms.HmacSha256);

        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, userId),
            new Claim(ClaimTypes.Name, username),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new Claim(JwtRegisteredClaimNames.Iat, DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(), ClaimValueTypes.Integer64)
        };

        var token = new JwtSecurityToken(
            issuer: _jwtSettings.Issuer,
            audience: _jwtSettings.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_jwtSettings.ExpirationMinutes),
            signingCredentials: credentials
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

---

## Paso 5: Crear el Slice de Login

Crea el archivo `webapi/features/auth/commands/Login.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using System.ComponentModel.DataAnnotations;
using webapi.common;
using webapi.common.dependencyinjection;
using webapi.common.security;

namespace webapi.features.auth.commands;

public class Login : IFeatureModule
{
    public record struct Request(
        [Required][property: Required] string Username,
        [Required][property: Required] string Password
    );

    public record struct Response(
        [Required][property: Required] string Token,
        [Required][property: Required] string Username,
        [Required][property: Required] DateTime ExpiresAt
    );

    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapPost("/auth/login", async (IService service, [FromBody] Request request) =>
        {
            var response = await service.Handler(request);
            
            if (response == null)
            {
                return Results.Unauthorized();
            }

            return Results.Ok(response);
        })
        .AllowAnonymous() // Permite acceso sin autenticación
        .WithOpenApi()
        .WithName("Login")
        .WithSummary("Autenticar usuario")
        .WithDescription("Endpoint para autenticar un usuario y obtener un token JWT")
        .WithTags("Autenticación")
        .Produces<Response>(StatusCodes.Status200OK)
        .ProducesProblem(StatusCodes.Status401Unauthorized);
    }

    public interface IService
    {
        Task<Response?> Handler(Request request);
    }

    [Injectable]
    public class Service : IService
    {
        private readonly IJwtTokenGenerator _tokenGenerator;
        private readonly int _expirationMinutes = 60; // Puedes inyectar esto desde configuración

        public Service(IJwtTokenGenerator tokenGenerator)
        {
            _tokenGenerator = tokenGenerator;
        }

        public async Task<Response?> Handler(Request request)
        {
            // Validar credenciales (DEMO - En producción usar hash y base de datos)
            if (!await ValidateCredentials(request.Username, request.Password))
            {
                return null; // Credenciales inválidas
            }

            // Generar token
            var userId = Guid.NewGuid().ToString(); // En producción, obtener de BD
            var token = _tokenGenerator.GenerateToken(request.Username, userId);

            var response = new Response(
                Token: token,
                Username: request.Username,
                ExpiresAt: DateTime.UtcNow.AddMinutes(_expirationMinutes)
            );

            return response;
        }

        private Task<bool> ValidateCredentials(string username, string password)
        {
            // DEMO: Credenciales hardcodeadas
            // En producción: consultar BD y verificar hash con BCrypt o similar
            var isValid = username == "admin" && password == "admin123";
            return Task.FromResult(isValid);
        }
    }
}
```

---

## Paso 6: Configurar la Autenticación en Program.cs

Modifica tu archivo `Program.cs` para configurar JWT:

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;
using webapi.common.configuration;

var builder = WebApplication.CreateBuilder(args);

// ===== CONFIGURACIÓN JWT =====
var jwtSettings = builder.Configuration.GetSection("Jwt").Get<JwtSettings>()
    ?? throw new InvalidOperationException("JWT settings not configured");

builder.Services.AddSingleton(jwtSettings);

builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ValidIssuer = jwtSettings.Issuer,
        ValidAudience = jwtSettings.Audience,
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(jwtSettings.Secret)),
        ClockSkew = TimeSpan.Zero // Elimina los 5 minutos de tolerancia por defecto
    };
});

builder.Services.AddAuthorization(options =>
{
    // Política por defecto: todos los endpoints requieren autenticación
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});

// ... resto de tu configuración existente ...

var app = builder.Build();

// ===== MIDDLEWARE DE AUTENTICACIÓN =====
app.UseAuthentication(); // ⚠️ Debe ir ANTES de UseAuthorization
app.UseAuthorization();

// ... resto de tu configuración ...

app.Run();
```

---

## Paso 7: Proteger Endpoints Existentes

Con la configuración de `FallbackPolicy` que hicimos en el Paso 6, **todos los endpoints ya están protegidos automáticamente**. 

Ya NO necesitas añadir `.RequireAuthorization()` en cada endpoint. Solo necesitas añadir `.AllowAnonymous()` a los endpoints que NO requieran autenticación (como el login).

Tu slice `CreateIngredient.cs` quedará así (sin cambios):

```csharp
public void AddRoutes(IEndpointRouteBuilder app)
{
    app.MapPost("/ingredientes", async (IService service, [FromBody] Request request) =>
    {
        var response = await service.Handler(request);
        return Results.Ok(response);
    })
    // ⬅️ YA NO ES NECESARIO .RequireAuthorization()
    .WithOpenApi()
    .WithName("CreateIngredient")
    .WithSummary("Crear un nuevo ingrediente")
    .WithDescription("Endpoint para crear un nuevo ingrediente con su nombre y costo")
    .WithTags("Ingredientes")
    .Produces<Response>(StatusCodes.Status200OK)
    .ProducesProblem(StatusCodes.Status400BadRequest)
    .ProducesProblem(StatusCodes.Status401Unauthorized); // ⬅️ Documenta el 401
}
```

**Regla simple:**
- ✅ **Todos los endpoints protegidos por defecto** (no añadas nada)
- ✅ **Solo añade** `.AllowAnonymous()` a endpoints públicos (login, registro, etc.)

---

## 🎯 Ventaja de la FallbackPolicy

Con esta configuración:

```csharp
options.FallbackPolicy = new AuthorizationPolicyBuilder()
    .RequireAuthenticatedUser()
    .Build();
```

**Conseguimos:**
- ✅ **Seguridad por defecto**: Todos los endpoints están protegidos automáticamente
- ✅ **Menos código**: No necesitas `.RequireAuthorization()` en cada endpoint
- ✅ **Menos errores**: No puedes olvidarte de proteger un endpoint sensible
- ✅ **Código más limpio**: Solo marcas explícitamente los endpoints públicos

**Antes (sin FallbackPolicy):**
```csharp
app.MapPost("/ingredientes", ...).RequireAuthorization(); // Hay que recordar ponerlo
app.MapPost("/pizzas", ...).RequireAuthorization();       // Hay que recordar ponerlo
app.MapPost("/pedidos", ...).RequireAuthorization();      // Hay que recordar ponerlo
app.MapPost("/login", ...).AllowAnonymous();
```

**Ahora (con FallbackPolicy):**
```csharp
app.MapPost("/ingredientes", ...); // Protegido automáticamente
app.MapPost("/pizzas", ...);       // Protegido automáticamente
app.MapPost("/pedidos", ...);      // Protegido automáticamente
app.MapPost("/login", ...).AllowAnonymous(); // Solo marcas los públicos
```

---

## Paso 8: Probar la Implementación

### 8.1. Ejecutar la Aplicación

```bash
dotnet run
```

### 8.2. Probar Login (Obtener Token)

**Petición:**
```http
POST https://localhost:5001/auth/login
Content-Type: application/json

{
  "username": "admin",
  "password": "admin123"
}
```

**Respuesta Esperada:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "username": "admin",
  "expiresAt": "2025-10-31T15:30:00Z"
}
```

### 8.3. Probar Endpoint Protegido SIN Token (Debe Fallar)

**Petición:**
```http
POST https://localhost:5001/ingredientes
Content-Type: application/json

{
  "name": "Mozzarella",
  "cost": 5.50
}
```

**Respuesta Esperada:**
```
401 Unauthorized
```

### 8.4. Probar Endpoint Protegido CON Token (Debe Funcionar)

**Petición:**
```http
POST https://localhost:5001/ingredientes
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

{
  "name": "Mozzarella",
  "cost": 5.50
}
```

**Respuesta Esperada:**
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "name": "Mozzarella",
  "cost": 5.50
}
```

---

## Paso 9: Probar con cURL (Línea de Comandos)

### Login:
```bash
curl -X POST https://localhost:5001/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}' \
  -k
```

### Guardar el token en variable:
```bash
TOKEN=$(curl -X POST https://localhost:5001/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}' \
  -k -s | jq -r '.token')
```

### Usar el token:
```bash
curl -X POST https://localhost:5001/ingredientes \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name":"Pepperoni","cost":3.50}' \
  -k
```

---

## Paso 10: Probar con Swagger/OpenAPI

1. Navega a `https://localhost:5001/swagger`
2. Haz clic en el endpoint `/auth/login`
3. Haz clic en "Try it out"
4. Ingresa las credenciales:
   ```json
   {
     "username": "admin",
     "password": "admin123"
   }
   ```
5. Copia el token de la respuesta
6. Haz clic en el botón **"Authorize"** (candado) en la parte superior
7. Ingresa: `Bearer {tu-token-aquí}`
8. Haz clic en "Authorize" y luego "Close"
9. Ahora podrás usar cualquier endpoint protegido

---

## Paso 11: Configurar Swagger para JWT (Opcional)

Añade esta configuración en tu `Program.cs` para que Swagger tenga un botón de autorización:

```csharp
builder.Services.AddSwaggerGen(options =>
{
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name = "Authorization",
        Type = SecuritySchemeType.Http,
        Scheme = "Bearer",
        BearerFormat = "JWT",
        In = ParameterLocation.Header,
        Description = "Ingresa 'Bearer' [espacio] y luego tu token JWT"
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });
});
```

No olvides añadir el using:
```csharp
using Microsoft.OpenApi.Models;
```

---

## Paso 12: Extraer Información del Usuario Autenticado

Si necesitas obtener información del usuario autenticado en tus endpoints:

```csharp
app.MapGet("/mi-perfil", (ClaimsPrincipal user) =>
{
    var userId = user.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    var username = user.FindFirst(ClaimTypes.Name)?.Value;
    
    return Results.Ok(new { userId, username });
})
// Ya está protegido por la FallbackPolicy, no necesitas .RequireAuthorization()
.WithTags("Perfil");
```

Si quisieras hacerlo público (poco común para un perfil):
```csharp
app.MapGet("/informacion-publica", () =>
{
    return Results.Ok(new { mensaje = "Esta info es pública" });
})
.AllowAnonymous() // Solo si quieres que sea público
.WithTags("Público");
```

---

## Mejoras para Producción

### 1. Hash de Contraseñas
Instala BCrypt:
```bash
dotnet add package BCrypt.Net-Next
```

Ejemplo de uso:
```csharp
// Al registrar usuario
string hashedPassword = BCrypt.Net.BCrypt.HashPassword(password);

// Al validar login
bool isValid = BCrypt.Net.BCrypt.Verify(password, hashedPasswordFromDb);
```

### 2. Refresh Tokens
Implementa refresh tokens para renovar tokens expirados sin requerir login.

### 3. Claims Personalizados
Añade roles, permisos, etc.:
```csharp
new Claim(ClaimTypes.Role, "Admin"),
new Claim("Permission", "CreateIngredient")
```

### 4. Validación de Usuario en Base de Datos
```csharp
private async Task<bool> ValidateCredentials(string username, string password)
{
    var user = await _context.Users
        .FirstOrDefaultAsync(u => u.Username == username);
    
    if (user == null) return false;
    
    return BCrypt.Net.BCrypt.Verify(password, user.PasswordHash);
}
```

### 5. Manejo de Errores Global
Crea un middleware para manejar excepciones de autorización:
```csharp
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        if (context.Response.StatusCode == 401)
        {
            await context.Response.WriteAsJsonAsync(new 
            { 
                error = "No autorizado", 
                message = "Token inválido o expirado" 
            });
        }
    });
});
```

---

## Resolución de Problemas Comunes

### Error: "No authentication handler is configured"
**Solución:** Verifica que `UseAuthentication()` esté antes de `UseAuthorization()` en `Program.cs`.

### Error: "IDX10503: Signature validation failed"
**Solución:** Verifica que el Secret en `appsettings.json` coincida y tenga al menos 32 caracteres.

### Error: "IDX10223: Lifetime validation failed"
**Solución:** El token ha expirado. Solicita uno nuevo mediante login.

### 401 aunque el token es válido
**Solución:** Verifica que estés enviando el header: `Authorization: Bearer {token}` (con espacio después de "Bearer").

---

## Estructura de Archivos Final

```
webapi/
├── common/
│   ├── configuration/
│   │   └── JwtSettings.cs
│   ├── security/
│   │   └── JwtTokenGenerator.cs
│   └── ... (otros archivos comunes)
├── features/
│   ├── auth/
│   │   └── commands/
│   │       └── Login.cs
│   ├── ingredient/
│   │   └── commands/
│   │       └── CreateIngredient.cs (modificado)
│   └── ... (otras features)
├── infrastructure/
│   └── ... (existente)
├── appsettings.json (modificado)
└── Program.cs (modificado)
```

---

## Conclusión

Has implementado con éxito un sistema de autenticación JWT completo siguiendo la arquitectura de slices vertical de tu proyecto. 

**Puntos clave:**
✅ Login sin repositorio, solo validación de credenciales  
✅ Generación de tokens JWT  
✅ Protección de endpoints con `.RequireAuthorization()`  
✅ Retorno automático de 401 para peticiones no autorizadas  
✅ Totalmente compatible con tu arquitectura existente  

**Próximos pasos recomendados:**
1. Implementar hash de contraseñas con BCrypt
2. Crear tabla de usuarios en base de datos
3. Añadir sistema de roles y permisos
4. Implementar refresh tokens
5. Añadir validación de fortaleza de contraseñas

¡Tu API ahora está protegida con autenticación JWT! 🎉
