# Tema 3: Desarrollo de Microservicios con .NET Core 8

## Índice
- [3.1 Estructura y Organización del Proyecto](#31-estructura-y-organización-del-proyecto)
- [3.2 Patrones de Diseño Comunes](#32-patrones-de-diseño-comunes)
- [3.3 Domain-Driven Design (DDD)](#33-domain-driven-design-ddd)
- [3.4 Validación y Manejo de Errores](#34-validación-y-manejo-de-errores)
- [3.5 Health Checks](#35-health-checks)
- [3.6 Documentación de APIs](#36-documentación-de-apis)
- [3.7 Testing de Microservicios](#37-testing-de-microservicios)

---

## 3.1 Estructura y Organización del Proyecto

### Clean Architecture

La Clean Architecture separa el código en capas con dependencias hacia el centro.

**Estructura de proyecto:**
```
ProductService/
├── ProductService.Domain/          # Entidades, interfaces del dominio
├── ProductService.Application/     # Lógica de negocio, casos de uso
├── ProductService.Infrastructure/  # Implementaciones de BD, APIs externas
└── ProductService.API/            # Controladores, configuración
```

**Ejemplo - Entidad en Domain:**
```csharp
// ProductService.Domain/Entities/Product.cs
namespace ProductService.Domain.Entities;

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int Stock { get; set; }
    
    public bool IsAvailable() => Stock > 0;
}
```

### Arquitectura en Capas

**1. Domain Layer - Entidad**
```csharp
public class Order
{
    public int Id { get; set; }
    public DateTime OrderDate { get; set; }
    public decimal Total { get; set; }
    public List<OrderItem> Items { get; set; } = new();
}
```

**2. Application Layer - Interface**
```csharp
public interface IOrderService
{
    Task<Order> CreateOrderAsync(Order order);
    Task<Order?> GetByIdAsync(int id);
}
```

**3. Application Layer - Implementación**
```csharp
public class OrderService : IOrderService
{
    private readonly IOrderRepository _repository;
    
    public OrderService(IOrderRepository repository)
    {
        _repository = repository;
    }
    
    public async Task<Order> CreateOrderAsync(Order order)
    {
        order.OrderDate = DateTime.UtcNow;
        return await _repository.AddAsync(order);
    }
    
    public async Task<Order?> GetByIdAsync(int id)
    {
        return await _repository.GetByIdAsync(id);
    }
}
```

**4. Infrastructure Layer - Repositorio**
```csharp
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;
    
    public OrderRepository(AppDbContext context)
    {
        _context = context;
    }
    
    public async Task<Order> AddAsync(Order order)
    {
        _context.Orders.Add(order);
        await _context.SaveChangesAsync();
        return order;
    }
    
    public async Task<Order?> GetByIdAsync(int id)
    {
        return await _context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id);
    }
}
```

**5. API Layer - Controlador**
```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;
    
    public OrdersController(IOrderService orderService)
    {
        _orderService = orderService;
    }
    
    [HttpPost]
    public async Task<IActionResult> Create(Order order)
    {
        var result = await _orderService.CreateOrderAsync(order);
        return CreatedAtAction(nameof(GetById), new { id = result.Id }, result);
    }
    
    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(int id)
    {
        var order = await _orderService.GetByIdAsync(id);
        if (order == null) return NotFound();
        return Ok(order);
    }
}
```

### Organización de Carpetas

**Proyecto único (servicios pequeños):**
```
OrderService/
├── Controllers/
├── Models/
├── Services/
├── Repositories/
└── Program.cs
```

**Múltiples proyectos (servicios grandes):**
```
OrderService/
├── OrderService.API/
│   ├── Controllers/
│   └── Program.cs
├── OrderService.Domain/
│   ├── Entities/
│   └── Interfaces/
├── OrderService.Application/
│   └── Services/
└── OrderService.Infrastructure/
    ├── Data/
    └── Repositories/
```

---

## 3.2 Patrones de Diseño Comunes

### Repository Pattern

**Interface:**
```csharp
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(int id);
    Task<IEnumerable<Product>> GetAllAsync();
    Task<Product> AddAsync(Product product);
    Task UpdateAsync(Product product);
    Task DeleteAsync(int id);
}
```

**Implementación:**
```csharp
public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _context;
    
    public ProductRepository(AppDbContext context)
    {
        _context = context;
    }
    
    public async Task<Product?> GetByIdAsync(int id)
    {
        return await _context.Products.FindAsync(id);
    }
    
    public async Task<IEnumerable<Product>> GetAllAsync()
    {
        return await _context.Products.ToListAsync();
    }
    
    public async Task<Product> AddAsync(Product product)
    {
        _context.Products.Add(product);
        await _context.SaveChangesAsync();
        return product;
    }
    
    public async Task UpdateAsync(Product product)
    {
        _context.Products.Update(product);
        await _context.SaveChangesAsync();
    }
    
    public async Task DeleteAsync(int id)
    {
        var product = await GetByIdAsync(id);
        if (product != null)
        {
            _context.Products.Remove(product);
            await _context.SaveChangesAsync();
        }
    }
}
```

**Registro en Program.cs:**
```csharp
builder.Services.AddScoped<IProductRepository, ProductRepository>();
```

### Unit of Work Pattern

```csharp
public interface IUnitOfWork : IDisposable
{
    IProductRepository Products { get; }
    IOrderRepository Orders { get; }
    Task<int> SaveChangesAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;
    
    public UnitOfWork(AppDbContext context)
    {
        _context = context;
        Products = new ProductRepository(_context);
        Orders = new OrderRepository(_context);
    }
    
    public IProductRepository Products { get; }
    public IOrderRepository Orders { get; }
    
    public async Task<int> SaveChangesAsync()
    {
        return await _context.SaveChangesAsync();
    }
    
    public void Dispose()
    {
        _context.Dispose();
    }
}
```

**Uso:**
```csharp
public class OrderService
{
    private readonly IUnitOfWork _unitOfWork;
    
    public OrderService(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }
    
    public async Task ProcessOrderAsync(Order order)
    {
        // Agregar orden
        await _unitOfWork.Orders.AddAsync(order);
        
        // Actualizar stock de productos
        foreach (var item in order.Items)
        {
            var product = await _unitOfWork.Products.GetByIdAsync(item.ProductId);
            if (product != null)
            {
                product.Stock -= item.Quantity;
                await _unitOfWork.Products.UpdateAsync(product);
            }
        }
        
        // Guardar todos los cambios en una transacción
        await _unitOfWork.SaveChangesAsync();
    }
}
```

### Factory Pattern

```csharp
public interface INotificationFactory
{
    INotification CreateNotification(string type);
}

public class NotificationFactory : INotificationFactory
{
    public INotification CreateNotification(string type)
    {
        return type.ToLower() switch
        {
            "email" => new EmailNotification(),
            "sms" => new SmsNotification(),
            "push" => new PushNotification(),
            _ => throw new ArgumentException("Tipo de notificación no válido")
        };
    }
}

public interface INotification
{
    Task SendAsync(string message);
}

public class EmailNotification : INotification
{
    public async Task SendAsync(string message)
    {
        await Task.Run(() => Console.WriteLine($"Email: {message}"));
    }
}

public class SmsNotification : INotification
{
    public async Task SendAsync(string message)
    {
        await Task.Run(() => Console.WriteLine($"SMS: {message}"));
    }
}
```

**Uso:**
```csharp
var factory = new NotificationFactory();
var notification = factory.CreateNotification("email");
await notification.SendAsync("Tu pedido ha sido confirmado");
```

### Strategy Pattern

```csharp
public interface IShippingStrategy
{
    decimal Calculate(decimal weight, decimal distance);
}

public class StandardShipping : IShippingStrategy
{
    public decimal Calculate(decimal weight, decimal distance)
    {
        return weight * 0.5m + distance * 0.1m;
    }
}

public class ExpressShipping : IShippingStrategy
{
    public decimal Calculate(decimal weight, decimal distance)
    {
        return weight * 1.0m + distance * 0.2m + 10m;
    }
}

public class ShippingCalculator
{
    private IShippingStrategy _strategy;
    
    public void SetStrategy(IShippingStrategy strategy)
    {
        _strategy = strategy;
    }
    
    public decimal CalculateCost(decimal weight, decimal distance)
    {
        return _strategy.Calculate(weight, distance);
    }
}
```

**Uso:**
```csharp
var calculator = new ShippingCalculator();

calculator.SetStrategy(new StandardShipping());
var standardCost = calculator.CalculateCost(5, 100); // $55

calculator.SetStrategy(new ExpressShipping());
var expressCost = calculator.CalculateCost(5, 100); // $35
```

### CQRS (Command Query Responsibility Segregation)

**Commands (Escritura):**
```csharp
public record CreateProductCommand(string Name, decimal Price, int Stock);

public class CreateProductHandler
{
    private readonly IProductRepository _repository;
    
    public CreateProductHandler(IProductRepository repository)
    {
        _repository = repository;
    }
    
    public async Task<int> HandleAsync(CreateProductCommand command)
    {
        var product = new Product
        {
            Name = command.Name,
            Price = command.Price,
            Stock = command.Stock
        };
        
        var result = await _repository.AddAsync(product);
        return result.Id;
    }
}
```

**Queries (Lectura):**
```csharp
public record GetProductQuery(int Id);

public class GetProductHandler
{
    private readonly IProductRepository _repository;
    
    public GetProductHandler(IProductRepository repository)
    {
        _repository = repository;
    }
    
    public async Task<Product?> HandleAsync(GetProductQuery query)
    {
        return await _repository.GetByIdAsync(query.Id);
    }
}
```

**Controlador con CQRS:**
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly CreateProductHandler _createHandler;
    private readonly GetProductHandler _getHandler;
    
    public ProductsController(
        CreateProductHandler createHandler, 
        GetProductHandler getHandler)
    {
        _createHandler = createHandler;
        _getHandler = getHandler;
    }
    
    [HttpPost]
    public async Task<IActionResult> Create(CreateProductCommand command)
    {
        var id = await _createHandler.HandleAsync(command);
        return CreatedAtAction(nameof(GetById), new { id }, null);
    }
    
    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(int id)
    {
        var product = await _getHandler.HandleAsync(new GetProductQuery(id));
        if (product == null) return NotFound();
        return Ok(product);
    }
}
```

---

## 3.3 Domain-Driven Design (DDD)

### Entidades y Value Objects

**Entidad (tiene identidad única):**
```csharp
public class Customer
{
    public int Id { get; set; } // Identidad única
    public string Name { get; set; } = string.Empty;
    public Email Email { get; set; } = null!;
    public Address Address { get; set; } = null!;
}
```

**Value Object (sin identidad, inmutable):**
```csharp
public class Email
{
    public string Value { get; }
    
    public Email(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("Email no puede estar vacío");
            
        if (!value.Contains("@"))
            throw new ArgumentException("Email inválido");
            
        Value = value;
    }
    
    public override bool Equals(object? obj)
    {
        if (obj is not Email other) return false;
        return Value == other.Value;
    }
    
    public override int GetHashCode() => Value.GetHashCode();
}

public class Address
{
    public string Street { get; }
    public string City { get; }
    public string ZipCode { get; }
    
    public Address(string street, string city, string zipCode)
    {
        Street = street;
        City = city;
        ZipCode = zipCode;
    }
}
```

### Agregados y Aggregate Roots

```csharp
// Aggregate Root
public class Order
{
    private readonly List<OrderItem> _items = new();
    
    public int Id { get; set; }
    public DateTime OrderDate { get; private set; }
    public decimal Total { get; private set; }
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    
    public Order()
    {
        OrderDate = DateTime.UtcNow;
    }
    
    // Método de negocio que mantiene consistencia
    public void AddItem(int productId, string productName, decimal price, int quantity)
    {
        var existingItem = _items.FirstOrDefault(i => i.ProductId == productId);
        
        if (existingItem != null)
        {
            existingItem.IncreaseQuantity(quantity);
        }
        else
        {
            _items.Add(new OrderItem(productId, productName, price, quantity));
        }
        
        CalculateTotal();
    }
    
    public void RemoveItem(int productId)
    {
        var item = _items.FirstOrDefault(i => i.ProductId == productId);
        if (item != null)
        {
            _items.Remove(item);
            CalculateTotal();
        }
    }
    
    private void CalculateTotal()
    {
        Total = _items.Sum(i => i.Subtotal);
    }
}

// Entidad hija (no se accede directamente, solo a través del Aggregate Root)
public class OrderItem
{
    public int Id { get; set; }
    public int ProductId { get; private set; }
    public string ProductName { get; private set; }
    public decimal Price { get; private set; }
    public int Quantity { get; private set; }
    public decimal Subtotal => Price * Quantity;
    
    private OrderItem() { } // Para EF Core
    
    public OrderItem(int productId, string productName, decimal price, int quantity)
    {
        ProductId = productId;
        ProductName = productName;
        Price = price;
        Quantity = quantity;
    }
    
    public void IncreaseQuantity(int amount)
    {
        if (amount <= 0)
            throw new ArgumentException("La cantidad debe ser mayor a cero");
            
        Quantity += amount;
    }
}
```

### Domain Events

```csharp
// Evento del dominio
public record OrderCreatedEvent(int OrderId, DateTime CreatedAt, decimal Total);

// Interface para eventos
public interface IDomainEvent
{
    DateTime OccurredOn { get; }
}

// Implementación
public class OrderCreatedEvent : IDomainEvent
{
    public int OrderId { get; }
    public decimal Total { get; }
    public DateTime OccurredOn { get; }
    
    public OrderCreatedEvent(int orderId, decimal total)
    {
        OrderId = orderId;
        Total = total;
        OccurredOn = DateTime.UtcNow;
    }
}

// Handler del evento
public class OrderCreatedEventHandler
{
    private readonly IEmailService _emailService;
    
    public OrderCreatedEventHandler(IEmailService emailService)
    {
        _emailService = emailService;
    }
    
    public async Task HandleAsync(OrderCreatedEvent @event)
    {
        await _emailService.SendOrderConfirmationAsync(@event.OrderId);
        Console.WriteLine($"Orden {@event.OrderId} creada con total: {@event.Total}");
    }
}
```

### Domain Services

```csharp
public interface IOrderDomainService
{
    bool CanPlaceOrder(Customer customer, Order order);
    decimal ApplyDiscount(Order order, Customer customer);
}

public class OrderDomainService : IOrderDomainService
{
    public bool CanPlaceOrder(Customer customer, Order order)
    {
        // Lógica de negocio compleja que involucra múltiples agregados
        if (customer.CreditLimit < order.Total)
            return false;
            
        if (order.Items.Count == 0)
            return false;
            
        return true;
    }
    
    public decimal ApplyDiscount(Order order, Customer customer)
    {
        // Lógica de descuento basada en reglas de negocio
        if (customer.IsVip && order.Total > 1000)
            return order.Total * 0.10m; // 10% descuento
            
        if (order.Total > 500)
            return order.Total * 0.05m; // 5% descuento
            
        return 0;
    }
}
```

### Bounded Contexts

```csharp
// Contexto de Ventas
namespace Sales.Domain
{
    public class Product
    {
        public int Id { get; set; }
        public string Name { get; set; } = string.Empty;
        public decimal Price { get; set; }
        public bool IsAvailable { get; set; }
    }
}

// Contexto de Inventario
namespace Inventory.Domain
{
    public class Product
    {
        public int Id { get; set; }
        public string Sku { get; set; } = string.Empty;
        public int Stock { get; set; }
        public string Warehouse { get; set; } = string.Empty;
        public DateTime LastRestockDate { get; set; }
    }
}

// Son el mismo concepto pero diferentes modelos según el contexto
```

---

## 3.4 Validación y Manejo de Errores

### Data Annotations

```csharp
public class CreateProductDto
{
    [Required(ErrorMessage = "El nombre es obligatorio")]
    [StringLength(100, MinimumLength = 3)]
    public string Name { get; set; } = string.Empty;
    
    [Required]
    [Range(0.01, 999999.99)]
    public decimal Price { get; set; }
    
    [Range(0, int.MaxValue)]
    public int Stock { get; set; }
    
    [EmailAddress]
    public string? SupplierEmail { get; set; }
    
    [Url]
    public string? ProductUrl { get; set; }
}
```

**Controlador:**
```csharp
[HttpPost]
public async Task<IActionResult> Create([FromBody] CreateProductDto dto)
{
    if (!ModelState.IsValid)
        return BadRequest(ModelState);
        
    // Procesar...
    return Ok();
}
```

### FluentValidation

**Instalación:**
```bash
dotnet add package FluentValidation.AspNetCore
```

**Validator:**
```csharp
using FluentValidation;

public class CreateProductValidator : AbstractValidator<CreateProductDto>
{
    public CreateProductValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("El nombre es obligatorio")
            .Length(3, 100).WithMessage("El nombre debe tener entre 3 y 100 caracteres");
            
        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("El precio debe ser mayor a 0")
            .LessThan(1000000).WithMessage("El precio es demasiado alto");
            
        RuleFor(x => x.Stock)
            .GreaterThanOrEqualTo(0).WithMessage("El stock no puede ser negativo");
            
        RuleFor(x => x.SupplierEmail)
            .EmailAddress().When(x => !string.IsNullOrEmpty(x.SupplierEmail))
            .WithMessage("Email inválido");
    }
}
```

**Registro en Program.cs:**
```csharp
using FluentValidation;

builder.Services.AddValidatorsFromAssemblyContaining<CreateProductValidator>();
```

**Uso en controlador:**
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IValidator<CreateProductDto> _validator;
    
    public ProductsController(IValidator<CreateProductDto> validator)
    {
        _validator = validator;
    }
    
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateProductDto dto)
    {
        var validationResult = await _validator.ValidateAsync(dto);
        
        if (!validationResult.IsValid)
            return BadRequest(validationResult.Errors);
            
        // Procesar...
        return Ok();
    }
}
```

### Problem Details (RFC 7807)

**Configuración en Program.cs:**
```csharp
builder.Services.AddProblemDetails();

var app = builder.Build();

app.UseExceptionHandler();
app.UseStatusCodePages();
```

**Clase de respuesta:**
```csharp
public class ValidationProblemDetails : Microsoft.AspNetCore.Mvc.ValidationProblemDetails
{
    public ValidationProblemDetails(IEnumerable<ValidationFailure> errors)
    {
        Title = "Errores de validación";
        Status = StatusCodes.Status400BadRequest;
        Type = "https://tools.ietf.org/html/rfc7807";
        
        foreach (var error in errors)
        {
            if (Errors.ContainsKey(error.PropertyName))
            {
                Errors[error.PropertyName] = Errors[error.PropertyName]
                    .Concat(new[] { error.ErrorMessage }).ToArray();
            }
            else
            {
                Errors.Add(error.PropertyName, new[] { error.ErrorMessage });
            }
        }
    }
}
```

### Manejo Centralizado de Excepciones

**Middleware personalizado:**
```csharp
public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;
    
    public GlobalExceptionMiddleware(
        RequestDelegate next, 
        ILogger<GlobalExceptionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error no controlado");
            await HandleExceptionAsync(context, ex);
        }
    }
    
    private static Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";
        
        var response = exception switch
        {
            NotFoundException => (StatusCodes.Status404NotFound, "Recurso no encontrado"),
            ValidationException => (StatusCodes.Status400BadRequest, "Error de validación"),
            UnauthorizedAccessException => (StatusCodes.Status401Unauthorized, "No autorizado"),
            _ => (StatusCodes.Status500InternalServerError, "Error interno del servidor")
        };
        
        context.Response.StatusCode = response.Item1;
        
        var result = JsonSerializer.Serialize(new
        {
            error = response.Item2,
            message = exception.Message
        });
        
        return context.Response.WriteAsync(result);
    }
}

// Excepciones personalizadas
public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }
}
```

**Registro:**
```csharp
app.UseMiddleware<GlobalExceptionMiddleware>();
```

### Códigos de Estado HTTP

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetById(int id)
{
    var product = await _repository.GetByIdAsync(id);
    
    if (product == null)
        return NotFound(new { message = "Producto no encontrado" }); // 404
        
    return Ok(product); // 200
}

[HttpPost]
public async Task<IActionResult> Create(Product product)
{
    var result = await _repository.AddAsync(product);
    return CreatedAtAction(nameof(GetById), new { id = result.Id }, result); // 201
}

[HttpPut("{id}")]
public async Task<IActionResult> Update(int id, Product product)
{
    if (id != product.Id)
        return BadRequest(); // 400
        
    await _repository.UpdateAsync(product);
    return NoContent(); // 204
}

[HttpDelete("{id}")]
public async Task<IActionResult> Delete(int id)
{
    await _repository.DeleteAsync(id);
    return NoContent(); // 204
}
```

---

## 3.5 Health Checks

### Health Checks Básicos

**Instalación:**
```bash
dotnet add package Microsoft.Extensions.Diagnostics.HealthChecks
```

**Configuración básica:**
```csharp
// Program.cs
builder.Services.AddHealthChecks();

var app = builder.Build();

app.MapHealthChecks("/health");
```

**Health Check personalizado:**
```csharp
public class CustomHealthCheck : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, 
        CancellationToken cancellationToken = default)
    {
        var isHealthy = true; // Tu lógica aquí
        
        if (isHealthy)
        {
            return Task.FromResult(
                HealthCheckResult.Healthy("El servicio está funcionando"));
        }
        
        return Task.FromResult(
            HealthCheckResult.Unhealthy("El servicio tiene problemas"));
    }
}
```

**Registro:**
```csharp
builder.Services.AddHealthChecks()
    .AddCheck<CustomHealthCheck>("custom_check");
```

### Health Checks de Dependencias

**Base de datos:**
```csharp
dotnet add package AspNetCore.HealthChecks.SqlServer
```

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("Default")!,
        name: "sql_server",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "db", "sql" });
```

**API externa:**
```csharp
dotnet add package AspNetCore.HealthChecks.Uris
```

```csharp
builder.Services.AddHealthChecks()
    .AddUrlGroup(
        uri: new Uri("https://api.example.com/health"),
        name: "external_api",
        failureStatus: HealthStatus.Degraded);
```

**Redis:**
```csharp
dotnet add package AspNetCore.HealthChecks.Redis
```

```csharp
builder.Services.AddHealthChecks()
    .AddRedis(
        redisConnectionString: "localhost:6379",
        name: "redis_cache");
```

### Liveness vs Readiness Probes

```csharp
// Program.cs
var app = builder.Build();

// Liveness: ¿El servicio está vivo?
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live")
});

// Readiness: ¿El servicio está listo para recibir tráfico?
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

**Configuración:**
```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: new[] { "live" })
    .AddSqlServer(
        connectionString!,
        tags: new[] { "ready", "db" })
    .AddRedis(
        "localhost:6379",
        tags: new[] { "ready", "cache" });
```

### Health Checks UI

**Instalación:**
```bash
dotnet add package AspNetCore.HealthChecks.UI
dotnet add package AspNetCore.HealthChecks.UI.InMemory.Storage
```

**Configuración:**
```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString!);

builder.Services
    .AddHealthChecksUI()
    .AddInMemoryStorage();

var app = builder.Build();

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecksUI(options => options.UIPath = "/health-ui");
```

**appsettings.json:**
```json
{
  "HealthChecksUI": {
    "HealthChecks": [
      {
        "Name": "Product Service",
        "Uri": "http://localhost:5001/health"
      },
      {
        "Name": "Order Service",
        "Uri": "http://localhost:5002/health"
      }
    ],
    "EvaluationTimeInSeconds": 10,
    "MinimumSecondsBetweenFailureNotifications": 60
  }
}
```

### Respuesta Detallada

```csharp
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        
        var result = JsonSerializer.Serialize(new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                description = e.Value.Description,
                duration = e.Value.Duration.TotalMilliseconds
            }),
            totalDuration = report.TotalDuration.TotalMilliseconds
        });
        
        await context.Response.WriteAsync(result);
    }
});
```

---

## 3.6 Documentación de APIs

### Swagger/OpenAPI Básico

**Instalación (incluido por defecto en .NET 8):**
```csharp
// Program.cs
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

### Configuración Avanzada

```csharp
using Microsoft.OpenApi.Models;

builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Version = "v1",
        Title = "Product API",
        Description = "API para gestión de productos",
        Contact = new OpenApiContact
        {
            Name = "Equipo de Desarrollo",
            Email = "dev@example.com"
        }
    });
    
    // Incluir comentarios XML
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);
});
```

**Habilitar XML en .csproj:**
```xml
<PropertyGroup>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <NoWarn>$(NoWarn);1591</NoWarn>
</PropertyGroup>
```

### Documentación de Endpoints

```csharp
/// <summary>
/// Obtiene un producto por su ID
/// </summary>
/// <param name="id">ID del producto</param>
/// <returns>El producto solicitado</returns>
/// <response code="200">Producto encontrado</response>
/// <response code="404">Producto no encontrado</response>
[HttpGet("{id}")]
[ProducesResponseType(typeof(Product), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public async Task<IActionResult> GetById(int id)
{
    var product = await _repository.GetByIdAsync(id);
    if (product == null)
        return NotFound();
    return Ok(product);
}

/// <summary>
/// Crea un nuevo producto
/// </summary>
/// <param name="product">Datos del producto</param>
/// <returns>Producto creado</returns>
/// <response code="201">Producto creado exitosamente</response>
/// <response code="400">Datos inválidos</response>
[HttpPost]
[ProducesResponseType(typeof(Product), StatusCodes.Status201Created)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
public async Task<IActionResult> Create([FromBody] Product product)
{
    var result = await _repository.AddAsync(product);
    return CreatedAtAction(nameof(GetById), new { id = result.Id }, result);
}
```

### Ejemplos y Esquemas

```csharp
public class CreateProductDto
{
    /// <summary>
    /// Nombre del producto
    /// </summary>
    /// <example>Laptop Dell XPS 15</example>
    [Required]
    public string Name { get; set; } = string.Empty;
    
    /// <summary>
    /// Precio del producto en USD
    /// </summary>
    /// <example>1299.99</example>
    [Required]
    [Range(0.01, 999999.99)]
    public decimal Price { get; set; }
    
    /// <summary>
    /// Cantidad en stock
    /// </summary>
    /// <example>50</example>
    [Range(0, int.MaxValue)]
    public int Stock { get; set; }
}
```

### Versionado de APIs

**Instalación:**
```bash
dotnet add package Asp.Versioning.Mvc
```

**Configuración:**
```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
}).AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});
```

**Controladores versionados:**
```csharp
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll()
    {
        return Ok("Version 1.0");
    }
}

[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("2.0")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll()
    {
        return Ok("Version 2.0 con nuevas características");
    }
}
```

**Swagger con versiones:**
```csharp
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo { Title = "API V1", Version = "v1" });
    options.SwaggerDoc("v2", new OpenApiInfo { Title = "API V2", Version = "v2" });
});

app.UseSwaggerUI(options =>
{
    options.SwaggerEndpoint("/swagger/v1/swagger.json", "API V1");
    options.SwaggerEndpoint("/swagger/v2/swagger.json", "API V2");
});
```

---

## 3.7 Testing de Microservicios

### Unit Testing con xUnit

**Instalación:**
```bash
dotnet new xunit -n ProductService.Tests
dotnet add package Moq
dotnet add package FluentAssertions
```

**Ejemplo básico:**
```csharp
public class ProductServiceTests
{
    [Fact]
    public async Task CreateProduct_ValidProduct_ReturnsProduct()
    {
        // Arrange
        var mockRepo = new Mock<IProductRepository>();
        var product = new Product { Name = "Test", Price = 100 };
        
        mockRepo.Setup(r => r.AddAsync(It.IsAny<Product>()))
            .ReturnsAsync(product);
            
        var service = new ProductService(mockRepo.Object);
        
        // Act
        var result = await service.CreateAsync(product);
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal("Test", result.Name);
    }
    
    [Theory]
    [InlineData("Laptop", 999.99)]
    [InlineData("Mouse", 29.99)]
    [InlineData("Teclado", 79.99)]
    public async Task CreateProduct_DifferentProducts_ReturnsCorrectPrice(
        string name, decimal price)
    {
        // Arrange
        var mockRepo = new Mock<IProductRepository>();
        var product = new Product { Name = name, Price = price };
        
        mockRepo.Setup(r => r.AddAsync(It.IsAny<Product>()))
            .ReturnsAsync(product);
            
        var service = new ProductService(mockRepo.Object);
        
        // Act
        var result = await service.CreateAsync(product);
        
        // Assert
        result.Price.Should().Be(price);
    }
}
```

### Integration Testing

**Instalación:**
```bash
dotnet add package Microsoft.AspNetCore.Mvc.Testing
```

**WebApplicationFactory:**
```csharp
public class ProductsControllerTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    
    public ProductsControllerTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }
    
    [Fact]
    public async Task GetAllProducts_ReturnsSuccessStatusCode()
    {
        // Act
        var response = await _client.GetAsync("/api/products");
        
        // Assert
        response.EnsureSuccessStatusCode();
        Assert.Equal("application/json; charset=utf-8", 
            response.Content.Headers.ContentType?.ToString());
    }
    
    [Fact]
    public async Task CreateProduct_ValidData_ReturnsCreated()
    {
        // Arrange
        var product = new { Name = "Test Product", Price = 99.99, Stock = 10 };
        var content = new StringContent(
            JsonSerializer.Serialize(product),
            Encoding.UTF8,
            "application/json");
        
        // Act
        var response = await _client.PostAsync("/api/products", content);
        
        // Assert
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    }
}
```

### Mocking y Fakes

**Moq - Repositorio:**
```csharp
public class OrderServiceTests
{
    [Fact]
    public async Task ProcessOrder_ValidOrder_UpdatesStock()
    {
        // Arrange
        var mockProductRepo = new Mock<IProductRepository>();
        var mockOrderRepo = new Mock<IOrderRepository>();
        
        var product = new Product { Id = 1, Stock = 100 };
        mockProductRepo.Setup(r => r.GetByIdAsync(1))
            .ReturnsAsync(product);
        
        var service = new OrderService(mockOrderRepo.Object, mockProductRepo.Object);
        var order = new Order();
        order.AddItem(1, "Product", 10, 5);
        
        // Act
        await service.ProcessAsync(order);
        
        // Assert
        mockProductRepo.Verify(r => r.UpdateAsync(
            It.Is<Product>(p => p.Stock == 95)), Times.Once);
    }
}
```

**Fake Objects:**
```csharp
public class FakeProductRepository : IProductRepository
{
    private readonly List<Product> _products = new();
    private int _nextId = 1;
    
    public Task<Product?> GetByIdAsync(int id)
    {
        return Task.FromResult(_products.FirstOrDefault(p => p.Id == id));
    }
    
    public Task<IEnumerable<Product>> GetAllAsync()
    {
        return Task.FromResult<IEnumerable<Product>>(_products);
    }
    
    public Task<Product> AddAsync(Product product)
    {
        product.Id = _nextId++;
        _products.Add(product);
        return Task.FromResult(product);
    }
    
    public Task UpdateAsync(Product product)
    {
        var existing = _products.FirstOrDefault(p => p.Id == product.Id);
        if (existing != null)
        {
            _products.Remove(existing);
            _products.Add(product);
        }
        return Task.CompletedTask;
    }
    
    public Task DeleteAsync(int id)
    {
        var product = _products.FirstOrDefault(p => p.Id == id);
        if (product != null)
            _products.Remove(product);
        return Task.CompletedTask;
    }
}
```

### Test Containers

**Instalación:**
```bash
dotnet add package Testcontainers
dotnet add package Testcontainers.MsSql
```

**Ejemplo con SQL Server:**
```csharp
public class ProductRepositoryIntegrationTests : IAsyncLifetime
{
    private MsSqlContainer _sqlContainer = null!;
    private AppDbContext _context = null!;
    
    public async Task InitializeAsync()
    {
        _sqlContainer = new MsSqlBuilder()
            .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
            .Build();
            
        await _sqlContainer.StartAsync();
        
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_sqlContainer.GetConnectionString())
            .Options;
            
        _context = new AppDbContext(options);
        await _context.Database.EnsureCreatedAsync();
    }
    
    [Fact]
    public async Task AddProduct_SavesToDatabase()
    {
        // Arrange
        var repository = new ProductRepository(_context);
        var product = new Product { Name = "Test", Price = 100 };
        
        // Act
        var result = await repository.AddAsync(product);
        
        // Assert
        Assert.True(result.Id > 0);
        
        var saved = await _context.Products.FindAsync(result.Id);
        Assert.NotNull(saved);
        Assert.Equal("Test", saved.Name);
    }
    
    public async Task DisposeAsync()
    {
        await _context.DisposeAsync();
        await _sqlContainer.DisposeAsync();
    }
}
```

### Testing de APIs

```csharp
public class ProductApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    
    public ProductApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }
    
    [Fact]
    public async Task GetProduct_ExistingId_ReturnsProduct()
    {
        // Act
        var response = await _client.GetAsync("/api/products/1");
        
        // Assert
        response.EnsureSuccessStatusCode();
        
        var content = await response.Content.ReadAsStringAsync();
        var product = JsonSerializer.Deserialize<Product>(content, 
            new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
            
        Assert.NotNull(product);
        Assert.Equal(1, product.Id);
    }
    
    [Fact]
    public async Task GetProduct_NonExistingId_ReturnsNotFound()
    {
        // Act
        var response = await _client.GetAsync("/api/products/99999");
        
        // Assert
        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
    }
    
    [Fact]
    public async Task CreateProduct_InvalidData_ReturnsBadRequest()
    {
        // Arrange
        var invalidProduct = new { Name = "", Price = -10 };
        var content = new StringContent(
            JsonSerializer.Serialize(invalidProduct),
            Encoding.UTF8,
            "application/json");
        
        // Act
        var response = await _client.PostAsync("/api/products", content);
        
        // Assert
        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
    }
}
```

### Coverage de Código

**Instalación:**
```bash
dotnet add package coverlet.collector
```

**Ejecutar con cobertura:**
```bash
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover
```

**Generar reporte HTML:**
```bash
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator -reports:coverage.opencover.xml -targetdir:coveragereport
```

**GitHub Actions ejemplo:**
```yaml
- name: Test with coverage
  run: dotnet test --collect:"XPlat Code Coverage"
  
- name: Generate coverage report
  run: |
    dotnet tool install -g dotnet-reportgenerator-globaltool
    reportgenerator -reports:**/coverage.cobertura.xml -targetdir:coverage
```

---

## Resumen del Tema 3

En este tema hemos cubierto:

1. **Estructura del Proyecto**: Clean Architecture, capas, organización
2. **Patrones de Diseño**: Repository, Unit of Work, Factory, Strategy, CQRS
3. **Domain-Driven Design**: Entidades, Value Objects, Agregados, Domain Events
4. **Validación**: Data Annotations, FluentValidation, Problem Details
5. **Health Checks**: Básicos, dependencias, Liveness/Readiness
6. **Documentación**: Swagger/OpenAPI, versionado de APIs
7. **Testing**: Unit tests, Integration tests, Test Containers

Estos conceptos forman la base sólida para desarrollar microservicios robustos, mantenibles y de alta calidad con .NET Core 8.
