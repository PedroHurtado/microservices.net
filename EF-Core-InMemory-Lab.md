# Laboratorio: Entity Framework Core 8 con InMemory Database en .NET 8

## 1. Instalación de Paquetes NuGet

Abre la terminal en VSCode y ejecuta los siguientes comandos:

```bash
dotnet add package Microsoft.EntityFrameworkCore --version 8.0.0
dotnet add package Microsoft.EntityFrameworkCore.InMemory --version 8.0.0
dotnet add package Microsoft.EntityFrameworkCore.Relational --version 8.0.0
```

## 2. Configuración de Entidades con Fluent API

### 2.1 Configuración de Ingredient

Crea el archivo `Data/Configurations/IngredientConfiguration.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace TuProyecto.Data.Configurations;

public class IngredientConfiguration : IEntityTypeConfiguration<Ingredient>
{
    public void Configure(EntityTypeBuilder<Ingredient> builder)
    {
        builder.ToTable("Ingredients");

        builder.HasKey(i => i.Id);

        builder.Property(i => i.Id)
            .IsRequired()
            .ValueGeneratedNever();

        builder.Property(i => i.Name)
            .IsRequired()
            .HasMaxLength(200);

        builder.Property(i => i.Cost)
            .IsRequired()
            .HasPrecision(18, 2);

        builder.HasIndex(i => i.Name)
            .IsUnique();
    }
}
```

### 2.2 Configuración de Pizza

Crea el archivo `Data/Configurations/PizzaConfiguration.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace TuProyecto.Data.Configurations;

public class PizzaConfiguration : IEntityTypeConfiguration<Pizza>
{
    public void Configure(EntityTypeBuilder<Pizza> builder)
    {
        builder.ToTable("Pizzas");

        builder.HasKey(p => p.Id);

        builder.Property(p => p.Id)
            .IsRequired()
            .ValueGeneratedNever();

        builder.Property(p => p.Name)
            .IsRequired()
            .HasMaxLength(200);

        builder.Property(p => p.Description)
            .IsRequired()
            .HasMaxLength(500);

        builder.Property(p => p.Url)
            .IsRequired()
            .HasMaxLength(500);

        builder.HasIndex(p => p.Name);

        // Configurar relación Many-to-Many sin propiedades de navegación
        builder.HasMany<Ingredient>()
            .WithMany();
    }
}
```

## 3. Creación del DbContext

Crea el archivo `Data/ApplicationDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using TuProyecto.Data.Configurations;

namespace TuProyecto.Data;

public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) 
        : base(options)
    {
    }

    public DbSet<Pizza> Pizzas { get; set; }
    public DbSet<Ingredient> Ingredients { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Opción 1: Registrar configuraciones manualmente
        modelBuilder.ApplyConfiguration(new IngredientConfiguration());
        modelBuilder.ApplyConfiguration(new PizzaConfiguration());

        // Opción 2: Registrar todas las configuraciones automáticamente con reflexión
        // Descomenta las siguientes líneas para usar este enfoque:
        
        // var assembly = typeof(ApplicationDbContext).Assembly;
        // modelBuilder.ApplyConfigurationsFromAssembly(assembly);

        // Opción 3: Registrar configuraciones específicas de un namespace con LINQ y reflexión
        // Descomenta las siguientes líneas para usar este enfoque:
        
        // var configurationsNamespace = "TuProyecto.Data.Configurations";
        // var configurationTypes = typeof(ApplicationDbContext).Assembly
        //     .GetTypes()
        //     .Where(t => t.Namespace == configurationsNamespace 
        //         && !t.IsAbstract 
        //         && !t.IsInterface
        //         && t.GetInterfaces().Any(i => 
        //             i.IsGenericType && 
        //             i.GetGenericTypeDefinition() == typeof(IEntityTypeConfiguration<>)))
        //     .ToList();
        //
        // foreach (var configurationType in configurationTypes)
        // {
        //     var configurationInstance = Activator.CreateInstance(configurationType);
        //     var applyMethod = typeof(ModelBuilder)
        //         .GetMethods()
        //         .First(m => m.Name == nameof(ModelBuilder.ApplyConfiguration) 
        //             && m.GetParameters().Length == 1
        //             && m.GetParameters()[0].ParameterType.GetGenericTypeDefinition() == typeof(IEntityTypeConfiguration<>));
        //     
        //     var entityType = configurationType.GetInterfaces()
        //         .First(i => i.IsGenericType && i.GetGenericTypeDefinition() == typeof(IEntityTypeConfiguration<>))
        //         .GetGenericArguments()[0];
        //     
        //     var genericApplyMethod = applyMethod.MakeGenericMethod(entityType);
        //     genericApplyMethod.Invoke(modelBuilder, new[] { configurationInstance });
        // }
    }
}
```

## 4. Configuración en Program.cs

Modifica tu archivo `Program.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using TuProyecto.Data;

var builder = WebApplication.CreateBuilder(args);

// Registrar el DbContext con In-Memory Database
builder.Services.AddDbContext<ApplicationDbContext>(options =>
{
    options.UseInMemoryDatabase("PizzaDb");
    
    if (builder.Environment.IsDevelopment())
    {
        options.EnableSensitiveDataLogging();
        options.EnableDetailedErrors();
    }
});

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

## 5. Configuración en appsettings.json

Crea o modifica el archivo `appsettings.json`:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Information"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "InMemoryDatabaseName": "PizzaDb"
  }
}
```

## 6. Configuración en appsettings.Development.json

Crea o modifica el archivo `appsettings.Development.json`:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.AspNetCore": "Information",
      "Microsoft.EntityFrameworkCore": "Debug"
    }
  }
}
```

## 7. Verificación de la Configuración

Para verificar que todo está correctamente configurado, ejecuta:

```bash
dotnet build
```

Si no hay errores, tu configuración de Entity Framework Core con InMemory Database está lista para usar.

## 8. Opciones de Registro de Configuraciones

En el `OnModelCreating` del DbContext tienes 3 opciones para registrar las configuraciones:

### Opción 1: Manual (Recomendada para proyectos pequeños)
```csharp
modelBuilder.ApplyConfiguration(new IngredientConfiguration());
modelBuilder.ApplyConfiguration(new PizzaConfiguration());
```
**Ventajas**: Control explícito, fácil de debuggear
**Desventajas**: Debes agregar cada configuración manualmente

### Opción 2: Automática por Ensamblado (Recomendada para proyectos grandes)
```csharp
modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
```
**Ventajas**: Registra automáticamente todas las clases que implementan `IEntityTypeConfiguration<T>` en el ensamblado
**Desventajas**: Menos control explícito

### Opción 3: Por Namespace con LINQ y Reflexión
```csharp
var configurationsNamespace = "TuProyecto.Data.Configurations";
var configurationTypes = typeof(ApplicationDbContext).Assembly
    .GetTypes()
    .Where(t => t.Namespace == configurationsNamespace 
        && !t.IsAbstract 
        && !t.IsInterface
        && t.GetInterfaces().Any(i => 
            i.IsGenericType && 
            i.GetGenericTypeDefinition() == typeof(IEntityTypeConfiguration<>)))
    .ToList();

foreach (var configurationType in configurationTypes)
{
    var configurationInstance = Activator.CreateInstance(configurationType);
    var applyMethod = typeof(ModelBuilder)
        .GetMethods()
        .First(m => m.Name == nameof(ModelBuilder.ApplyConfiguration) 
            && m.GetParameters().Length == 1
            && m.GetParameters()[0].ParameterType.GetGenericTypeDefinition() == typeof(IEntityTypeConfiguration<>));
    
    var entityType = configurationType.GetInterfaces()
        .First(i => i.IsGenericType && i.GetGenericTypeDefinition() == typeof(IEntityTypeConfiguration<>))
        .GetGenericArguments()[0];
    
    var genericApplyMethod = applyMethod.MakeGenericMethod(entityType);
    genericApplyMethod.Invoke(modelBuilder, new[] { configurationInstance });
}
```
**Ventajas**: Control fino sobre qué namespace incluir
**Desventajas**: Más complejo, puede ser overkill para la mayoría de proyectos

## 9. Notas Importantes

- **Sin Propiedades de Navegación**: Las entidades no tienen referencias entre sí, la relación se maneja completamente en la configuración de EF Core
- **Tabla Intermedia**: EF Core crea automáticamente una tabla `IngredientPizza` por convención
- **InMemory Database**: Los datos se pierden cuando la aplicación se detiene
- **Consultas Many-to-Many**: Se realizan mediante SQL directo o queries personalizadas
- **Fluent API**: Toda la configuración de relaciones está en las clases de configuración

## 10. Comandos Útiles

```bash
# Restaurar paquetes
dotnet restore

# Compilar el proyecto
dotnet build

# Ejecutar el proyecto
dotnet run

# Limpiar el proyecto
dotnet clean
```
