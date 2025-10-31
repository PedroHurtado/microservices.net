# Tema 4: Comunicación entre Microservicios

## 4.1 Patrones de Comunicación

### Comunicación Síncrona vs Asíncrona

**Síncrona:** El cliente espera la respuesta inmediata del servidor.

```csharp
// Ejemplo: Llamada HTTP síncrona
var response = await httpClient.GetAsync("https://api.pedidos.com/orders/123");
var order = await response.Content.ReadFromJsonAsync<Order>();
```

**Asíncrona:** El cliente envía el mensaje y continúa sin esperar respuesta inmediata.

```csharp
// Ejemplo: Publicar evento
await messageBus.Publish(new OrderCreatedEvent 
{ 
    OrderId = 123, 
    Total = 100.50m 
});
```

### Request/Response

Patrón donde un servicio hace una petición y espera una respuesta específica.

```csharp
// Servicio de Pedidos solicita información de un producto
public class OrderService
{
    private readonly HttpClient _httpClient;
    
    public async Task<decimal> GetProductPrice(int productId)
    {
        var response = await _httpClient.GetAsync($"api/products/{productId}");
        var product = await response.Content.ReadFromJsonAsync<Product>();
        return product.Price;
    }
}
```

### Publish/Subscribe

Un servicio publica eventos y múltiples servicios pueden suscribirse.

```csharp
// Publisher
public class OrderService
{
    private readonly IEventBus _eventBus;
    
    public async Task CreateOrder(Order order)
    {
        // Guardar pedido
        await _repository.Save(order);
        
        // Publicar evento
        await _eventBus.Publish(new OrderCreatedEvent(order.Id));
    }
}

// Subscriber 1: Servicio de Inventario
public class InventoryEventHandler : IEventHandler<OrderCreatedEvent>
{
    public async Task Handle(OrderCreatedEvent @event)
    {
        // Reducir stock
        await _inventory.DecreaseStock(@event.OrderId);
    }
}

// Subscriber 2: Servicio de Notificaciones
public class NotificationEventHandler : IEventHandler<OrderCreatedEvent>
{
    public async Task Handle(OrderCreatedEvent @event)
    {
        // Enviar email
        await _emailService.SendOrderConfirmation(@event.OrderId);
    }
}
```

### Event-Driven Architecture

Los servicios reaccionan a eventos que ocurren en el sistema.

```csharp
// Evento
public record ProductPriceChangedEvent(int ProductId, decimal OldPrice, decimal NewPrice);

// Servicio que escucha el evento
public class PriceHistoryService
{
    public async Task OnPriceChanged(ProductPriceChangedEvent @event)
    {
        await _database.SaveAsync(new PriceHistory
        {
            ProductId = @event.ProductId,
            OldPrice = @event.OldPrice,
            NewPrice = @event.NewPrice,
            ChangedAt = DateTime.UtcNow
        });
    }
}
```

### Choreography vs Orchestration

**Choreography:** Cada servicio sabe qué hacer cuando ocurre un evento.

```csharp
// Cada servicio actúa de forma independiente
OrderService -> Publica "OrderCreated"
    -> InventoryService escucha y reduce stock
    -> PaymentService escucha y procesa pago
    -> ShippingService escucha y crea envío
```

**Orchestration:** Un coordinador central dirige el flujo.

```csharp
// Orquestador
public class OrderOrchestrator
{
    public async Task ProcessOrder(int orderId)
    {
        // Paso 1: Validar inventario
        var hasStock = await _inventoryService.CheckStock(orderId);
        if (!hasStock) throw new Exception("Sin stock");
        
        // Paso 2: Procesar pago
        var paymentOk = await _paymentService.ProcessPayment(orderId);
        if (!paymentOk) throw new Exception("Pago fallido");
        
        // Paso 3: Crear envío
        await _shippingService.CreateShipment(orderId);
    }
}
```

---

## 4.2 Comunicación Síncrona con HTTP/REST

### RESTful APIs

Diseño de APIs siguiendo principios REST.

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // GET api/products
    [HttpGet]
    public async Task<ActionResult<List<Product>>> GetAll()
    {
        var products = await _repository.GetAllAsync();
        return Ok(products);
    }
    
    // GET api/products/5
    [HttpGet("{id}")]
    public async Task<ActionResult<Product>> GetById(int id)
    {
        var product = await _repository.GetByIdAsync(id);
        if (product == null) return NotFound();
        return Ok(product);
    }
    
    // POST api/products
    [HttpPost]
    public async Task<ActionResult<Product>> Create(Product product)
    {
        await _repository.AddAsync(product);
        return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
    }
    
    // PUT api/products/5
    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, Product product)
    {
        if (id != product.Id) return BadRequest();
        await _repository.UpdateAsync(product);
        return NoContent();
    }
    
    // DELETE api/products/5
    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        await _repository.DeleteAsync(id);
        return NoContent();
    }
}
```

### HttpClient y HttpClientFactory

**Problema:** Crear HttpClient directamente puede causar agotamiento de sockets.

```csharp
// ❌ MAL - No hacer esto
using (var client = new HttpClient())
{
    var response = await client.GetAsync("https://api.example.com");
}
```

**Solución:** Usar HttpClientFactory.

```csharp
// Configuración en Program.cs
builder.Services.AddHttpClient<IProductService, ProductService>(client =>
{
    client.BaseAddress = new Uri("https://api.products.com");
    client.Timeout = TimeSpan.FromSeconds(30);
});

// Uso en el servicio
public class ProductService : IProductService
{
    private readonly HttpClient _httpClient;
    
    public ProductService(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }
    
    public async Task<Product> GetProductAsync(int id)
    {
        var response = await _httpClient.GetAsync($"api/products/{id}");
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<Product>();
    }
}
```

### Resiliencia con Polly

Polly es una librería para implementar patrones de resiliencia.

**Instalación:**
```bash
dotnet add package Microsoft.Extensions.Http.Polly
```

**Retry Pattern:**

```csharp
builder.Services.AddHttpClient<IOrderService, OrderService>()
    .AddTransientHttpErrorPolicy(policy => 
        policy.WaitAndRetryAsync(3, retryAttempt => 
            TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))));
```

**Circuit Breaker:**

```csharp
builder.Services.AddHttpClient<IOrderService, OrderService>()
    .AddTransientHttpErrorPolicy(policy =>
        policy.CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30)));
```

**Combinando Políticas:**

```csharp
builder.Services.AddHttpClient<IOrderService, OrderService>()
    .AddPolicyHandler(GetRetryPolicy())
    .AddPolicyHandler(GetCircuitBreakerPolicy());

static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(retryAttempt));
}

static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(5, TimeSpan.FromSeconds(30));
}
```

### Manejo de Errores

```csharp
public class ProductService
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<ProductService> _logger;
    
    public async Task<Product?> GetProductAsync(int id)
    {
        try
        {
            var response = await _httpClient.GetAsync($"api/products/{id}");
            
            if (response.StatusCode == HttpStatusCode.NotFound)
            {
                return null;
            }
            
            response.EnsureSuccessStatusCode();
            return await response.Content.ReadFromJsonAsync<Product>();
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Error al obtener producto {ProductId}", id);
            throw new ServiceException("Error al comunicar con servicio de productos", ex);
        }
        catch (TaskCanceledException ex)
        {
            _logger.LogError(ex, "Timeout al obtener producto {ProductId}", id);
            throw new ServiceException("Timeout en servicio de productos", ex);
        }
    }
}
```

### Timeouts y Cancelación

```csharp
public class OrderService
{
    private readonly HttpClient _httpClient;
    
    public async Task<Order> GetOrderAsync(int id, CancellationToken cancellationToken)
    {
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
        cts.CancelAfter(TimeSpan.FromSeconds(5)); // Timeout de 5 segundos
        
        var response = await _httpClient.GetAsync($"api/orders/{id}", cts.Token);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<Order>(cancellationToken: cts.Token);
    }
}
```

### Idempotencia en APIs

Asegurar que múltiples llamadas idénticas tengan el mismo efecto que una sola.

```csharp
[ApiController]
[Route("api/[controller]")]
public class PaymentsController : ControllerBase
{
    private readonly IPaymentService _paymentService;
    
    [HttpPost]
    public async Task<IActionResult> ProcessPayment([FromBody] PaymentRequest request)
    {
        // Usar un idempotency key
        var idempotencyKey = Request.Headers["Idempotency-Key"].FirstOrDefault();
        
        if (string.IsNullOrEmpty(idempotencyKey))
        {
            return BadRequest("Idempotency-Key header requerido");
        }
        
        // Verificar si ya se procesó
        var existingPayment = await _paymentService.GetByIdempotencyKey(idempotencyKey);
        if (existingPayment != null)
        {
            return Ok(existingPayment); // Retornar resultado existente
        }
        
        // Procesar pago
        var payment = await _paymentService.ProcessAsync(request, idempotencyKey);
        return Ok(payment);
    }
}
```

---

## 4.3 gRPC

### Introducción a gRPC

gRPC es un framework RPC de alto rendimiento que usa HTTP/2 y Protocol Buffers.

**Ventajas:**
- Más rápido que REST/JSON
- Tipado fuerte
- Streaming bidireccional
- Generación automática de código

**Instalación:**
```bash
dotnet add package Grpc.AspNetCore
dotnet add package Google.Protobuf
dotnet add package Grpc.Tools
```

### Protocol Buffers

Definir el contrato en archivo `.proto`:

```protobuf
// protos/product.proto
syntax = "proto3";

option csharp_namespace = "ProductService.Grpc";

service ProductService {
  rpc GetProduct (ProductRequest) returns (ProductResponse);
  rpc ListProducts (EmptyRequest) returns (stream ProductResponse);
}

message ProductRequest {
  int32 id = 1;
}

message ProductResponse {
  int32 id = 1;
  string name = 2;
  double price = 3;
  bool in_stock = 4;
}

message EmptyRequest {}
```

### Implementación del Servidor gRPC

```csharp
public class ProductService : ProductService.ProductServiceBase
{
    private readonly IProductRepository _repository;
    private readonly ILogger<ProductService> _logger;
    
    public ProductService(IProductRepository repository, ILogger<ProductService> logger)
    {
        _repository = repository;
        _logger = logger;
    }
    
    public override async Task<ProductResponse> GetProduct(
        ProductRequest request, 
        ServerCallContext context)
    {
        _logger.LogInformation("Obteniendo producto {ProductId}", request.Id);
        
        var product = await _repository.GetByIdAsync(request.Id);
        
        if (product == null)
        {
            throw new RpcException(new Status(StatusCode.NotFound, "Producto no encontrado"));
        }
        
        return new ProductResponse
        {
            Id = product.Id,
            Name = product.Name,
            Price = product.Price,
            InStock = product.InStock
        };
    }
    
    public override async Task ListProducts(
        EmptyRequest request, 
        IServerStreamWriter<ProductResponse> responseStream, 
        ServerCallContext context)
    {
        var products = await _repository.GetAllAsync();
        
        foreach (var product in products)
        {
            await responseStream.WriteAsync(new ProductResponse
            {
                Id = product.Id,
                Name = product.Name,
                Price = product.Price,
                InStock = product.InStock
            });
        }
    }
}
```

**Configuración en Program.cs:**

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddGrpc();

var app = builder.Build();
app.MapGrpcService<ProductService>();
app.Run();
```

### Cliente gRPC

```csharp
public class ProductClient
{
    private readonly ProductService.ProductServiceClient _client;
    
    public ProductClient(GrpcChannel channel)
    {
        _client = new ProductService.ProductServiceClient(channel);
    }
    
    public async Task<ProductResponse> GetProductAsync(int productId)
    {
        var request = new ProductRequest { Id = productId };
        var response = await _client.GetProductAsync(request);
        return response;
    }
    
    public async Task<List<ProductResponse>> GetAllProductsAsync()
    {
        var products = new List<ProductResponse>();
        var request = new EmptyRequest();
        
        using var call = _client.ListProducts(request);
        
        await foreach (var product in call.ResponseStream.ReadAllAsync())
        {
            products.Add(product);
        }
        
        return products;
    }
}
```

**Configuración del cliente:**

```csharp
builder.Services.AddGrpcClient<ProductService.ProductServiceClient>(options =>
{
    options.Address = new Uri("https://localhost:5001");
});
```

### Tipos de Comunicación gRPC

**1. Unary (Request-Response simple):**

```protobuf
rpc GetProduct (ProductRequest) returns (ProductResponse);
```

**2. Server Streaming:**

```protobuf
rpc ListProducts (EmptyRequest) returns (stream ProductResponse);
```

```csharp
// Cliente
using var call = client.ListProducts(new EmptyRequest());
await foreach (var product in call.ResponseStream.ReadAllAsync())
{
    Console.WriteLine($"Producto: {product.Name}");
}
```

**3. Client Streaming:**

```protobuf
rpc UploadProducts (stream ProductRequest) returns (UploadSummary);
```

```csharp
// Cliente
using var call = client.UploadProducts();
foreach (var product in products)
{
    await call.RequestStream.WriteAsync(product);
}
await call.RequestStream.CompleteAsync();
var summary = await call;
```

**4. Bidirectional Streaming:**

```protobuf
rpc Chat (stream ChatMessage) returns (stream ChatMessage);
```

### gRPC vs REST: Cuándo Usar Cada Uno

**Usar gRPC cuando:**
- Necesitas alto rendimiento
- Comunicación interna entre microservicios
- Streaming de datos en tiempo real
- Contratos fuertemente tipados

**Usar REST cuando:**
- APIs públicas
- Clientes web (navegadores)
- Simplicidad y facilidad de debug
- Compatibilidad con herramientas existentes

---

## 4.4 Comunicación Asíncrona con Message Brokers

### Conceptos de Mensajería

**Ventajas:**
- Desacoplamiento
- Tolerancia a fallos
- Balanceo de carga natural
- Procesamiento asíncrono

**Conceptos clave:**
- **Producer:** Envía mensajes
- **Consumer:** Recibe mensajes
- **Queue:** Cola de mensajes
- **Exchange:** Enruta mensajes
- **Topic:** Canal de publicación/suscripción

### RabbitMQ

**Instalación:**
```bash
dotnet add package RabbitMQ.Client
```

#### Exchanges, Queues y Bindings

**Tipos de Exchange:**
- **Direct:** Routing exacto por routing key
- **Fanout:** Broadcasting a todas las colas
- **Topic:** Routing por patrones
- **Headers:** Routing por headers

**Publisher:**

```csharp
public class RabbitMqPublisher
{
    private readonly IConnection _connection;
    private readonly IModel _channel;
    
    public RabbitMqPublisher()
    {
        var factory = new ConnectionFactory
        {
            HostName = "localhost",
            UserName = "guest",
            Password = "guest"
        };
        
        _connection = factory.CreateConnection();
        _channel = _connection.CreateModel();
        
        // Declarar exchange
        _channel.ExchangeDeclare(
            exchange: "orders",
            type: ExchangeType.Topic,
            durable: true);
    }
    
    public void PublishOrderCreated(OrderCreatedEvent order)
    {
        var message = JsonSerializer.Serialize(order);
        var body = Encoding.UTF8.GetBytes(message);
        
        _channel.BasicPublish(
            exchange: "orders",
            routingKey: "order.created",
            basicProperties: null,
            body: body);
    }
}
```

**Consumer:**

```csharp
public class RabbitMqConsumer
{
    private readonly IConnection _connection;
    private readonly IModel _channel;
    
    public RabbitMqConsumer()
    {
        var factory = new ConnectionFactory { HostName = "localhost" };
        _connection = factory.CreateConnection();
        _channel = _connection.CreateModel();
        
        // Declarar cola
        _channel.QueueDeclare(
            queue: "inventory-service",
            durable: true,
            exclusive: false,
            autoDelete: false);
        
        // Bind cola al exchange
        _channel.QueueBind(
            queue: "inventory-service",
            exchange: "orders",
            routingKey: "order.*");
    }
    
    public void StartConsuming()
    {
        var consumer = new EventingBasicConsumer(_channel);
        
        consumer.Received += (model, ea) =>
        {
            var body = ea.Body.ToArray();
            var message = Encoding.UTF8.GetString(body);
            var order = JsonSerializer.Deserialize<OrderCreatedEvent>(message);
            
            // Procesar mensaje
            ProcessOrder(order);
            
            // Confirmar procesamiento
            _channel.BasicAck(deliveryTag: ea.DeliveryTag, multiple: false);
        };
        
        _channel.BasicConsume(
            queue: "inventory-service",
            autoAck: false,
            consumer: consumer);
    }
    
    private void ProcessOrder(OrderCreatedEvent order)
    {
        Console.WriteLine($"Procesando pedido: {order.OrderId}");
    }
}
```

#### Patrones de Mensajería

**Work Queue (Competidores):**

```csharp
// Múltiples consumers procesan mensajes de la misma cola
_channel.BasicQos(prefetchSize: 0, prefetchCount: 1, global: false);
```

**Publish/Subscribe:**

```csharp
// Exchange tipo fanout - todos los consumers reciben el mensaje
_channel.ExchangeDeclare("notifications", ExchangeType.Fanout);
```

### Azure Service Bus

**Instalación:**
```bash
dotnet add package Azure.Messaging.ServiceBus
```

#### Queues y Topics

**Sender (Queue):**

```csharp
public class ServiceBusSender
{
    private readonly ServiceBusClient _client;
    private readonly ServiceBusSender _sender;
    
    public ServiceBusSender(string connectionString)
    {
        _client = new ServiceBusClient(connectionString);
        _sender = _client.CreateSender("orders");
    }
    
    public async Task SendOrderAsync(Order order)
    {
        var message = new ServiceBusMessage(JsonSerializer.Serialize(order))
        {
            ContentType = "application/json",
            Subject = "order.created",
            MessageId = Guid.NewGuid().ToString()
        };
        
        await _sender.SendMessageAsync(message);
    }
}
```

**Receiver (Queue):**

```csharp
public class ServiceBusReceiver
{
    private readonly ServiceBusProcessor _processor;
    
    public ServiceBusReceiver(string connectionString)
    {
        var client = new ServiceBusClient(connectionString);
        _processor = client.CreateProcessor("orders", new ServiceBusProcessorOptions
        {
            AutoCompleteMessages = false,
            MaxConcurrentCalls = 2
        });
        
        _processor.ProcessMessageAsync += MessageHandler;
        _processor.ProcessErrorAsync += ErrorHandler;
    }
    
    public async Task StartAsync()
    {
        await _processor.StartProcessingAsync();
    }
    
    private async Task MessageHandler(ProcessMessageEventArgs args)
    {
        var body = args.Message.Body.ToString();
        var order = JsonSerializer.Deserialize<Order>(body);
        
        // Procesar pedido
        await ProcessOrderAsync(order);
        
        // Completar mensaje
        await args.CompleteMessageAsync(args.Message);
    }
    
    private Task ErrorHandler(ProcessErrorEventArgs args)
    {
        Console.WriteLine($"Error: {args.Exception.Message}");
        return Task.CompletedTask;
    }
}
```

**Topics y Subscriptions:**

```csharp
// Publisher
var sender = client.CreateSender("order-events");
await sender.SendMessageAsync(new ServiceBusMessage("Order Created"));

// Subscriber 1
var processor1 = client.CreateProcessor("order-events", "inventory-subscription");

// Subscriber 2
var processor2 = client.CreateProcessor("order-events", "notification-subscription");
```

### MassTransit

MassTransit es una abstracción sobre message brokers.

**Instalación:**
```bash
dotnet add package MassTransit
dotnet add package MassTransit.RabbitMQ
```

**Configuración:**

```csharp
builder.Services.AddMassTransit(x =>
{
    // Registrar consumers
    x.AddConsumer<OrderCreatedConsumer>();
    
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("localhost", "/", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });
        
        cfg.ConfigureEndpoints(context);
    });
});
```

**Publisher:**

```csharp
public class OrderService
{
    private readonly IPublishEndpoint _publishEndpoint;
    
    public async Task CreateOrderAsync(Order order)
    {
        await _repository.SaveAsync(order);
        
        await _publishEndpoint.Publish(new OrderCreatedEvent
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            Total = order.Total
        });
    }
}
```

**Consumer:**

```csharp
public class OrderCreatedConsumer : IConsumer<OrderCreatedEvent>
{
    private readonly ILogger<OrderCreatedConsumer> _logger;
    
    public OrderCreatedConsumer(ILogger<OrderCreatedConsumer> logger)
    {
        _logger = logger;
    }
    
    public async Task Consume(ConsumeContext<OrderCreatedEvent> context)
    {
        _logger.LogInformation("Pedido creado: {OrderId}", context.Message.OrderId);
        
        // Procesar evento
        await ProcessOrderAsync(context.Message);
    }
}
```

#### Sagas

Las Sagas coordinan transacciones distribuidas de larga duración.

```csharp
public class OrderStateMachine : MassTransitStateMachine<OrderState>
{
    public OrderStateMachine()
    {
        InstanceState(x => x.CurrentState);
        
        Event(() => OrderSubmitted);
        Event(() => PaymentProcessed);
        Event(() => OrderShipped);
        
        Initially(
            When(OrderSubmitted)
                .Then(context =>
                {
                    context.Instance.OrderId = context.Data.OrderId;
                })
                .TransitionTo(AwaitingPayment));
        
        During(AwaitingPayment,
            When(PaymentProcessed)
                .TransitionTo(AwaitingShipment));
        
        During(AwaitingShipment,
            When(OrderShipped)
                .TransitionTo(Completed));
    }
    
    public State AwaitingPayment { get; private set; }
    public State AwaitingShipment { get; private set; }
    public State Completed { get; private set; }
    
    public Event<OrderSubmittedEvent> OrderSubmitted { get; private set; }
    public Event<PaymentProcessedEvent> PaymentProcessed { get; private set; }
    public Event<OrderShippedEvent> OrderShipped { get; private set; }
}

public class OrderState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; }
    public int OrderId { get; set; }
}
```

---

## 4.5 API Gateway

### Concepto y Propósito

Un API Gateway es un punto de entrada único para los clientes.

**Funciones:**
- Routing de requests
- Agregación de respuestas
- Autenticación y autorización
- Rate limiting
- Transformación de requests/responses
- Cache
- Logging y monitoreo

### Ocelot

**Instalación:**
```bash
dotnet add package Ocelot
```

**Configuración básica (ocelot.json):**

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/products/{everything}",
      "DownstreamScheme": "https",
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5001
        }
      ],
      "UpstreamPathTemplate": "/products/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
    },
    {
      "DownstreamPathTemplate": "/api/orders/{everything}",
      "DownstreamScheme": "https",
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5002
        }
      ],
      "UpstreamPathTemplate": "/orders/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST" ]
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "https://localhost:5000"
  }
}
```

**Program.cs:**

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Configuration.AddJsonFile("ocelot.json", optional: false, reloadOnChange: true);
builder.Services.AddOcelot();

var app = builder.Build();
await app.UseOcelot();
app.Run();
```

#### Agregación de Requests

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/products/{id}",
      "DownstreamScheme": "https",
      "DownstreamHostAndPorts": [{ "Host": "localhost", "Port": 5001 }],
      "UpstreamPathTemplate": "/products/{id}",
      "Key": "Product"
    },
    {
      "DownstreamPathTemplate": "/api/inventory/{id}",
      "DownstreamScheme": "https",
      "DownstreamHostAndPorts": [{ "Host": "localhost", "Port": 5003 }],
      "UpstreamPathTemplate": "/inventory/{id}",
      "Key": "Inventory"
    }
  ],
  "Aggregates": [
    {
      "RouteKeys": [ "Product", "Inventory" ],
      "UpstreamPathTemplate": "/product-details/{id}"
    }
  ]
}
```

#### Rate Limiting

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/products",
      "UpstreamPathTemplate": "/products",
      "RateLimitOptions": {
        "ClientWhitelist": [],
        "EnableRateLimiting": true,
        "Period": "1m",
        "PeriodTimespan": 60,
        "Limit": 100
      }
    }
  ]
}
```

#### Autenticación

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/orders",
      "UpstreamPathTemplate": "/orders",
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "Bearer",
        "AllowedScopes": []
      }
    }
  ]
}
```

```csharp
// Program.cs
builder.Services
    .AddAuthentication()
    .AddJwtBearer("Bearer", options =>
    {
        options.Authority = "https://identity-server.com";
        options.Audience = "api";
    });

builder.Services.AddOcelot();
```

### YARP (Yet Another Reverse Proxy)

**Instalación:**
```bash
dotnet add package Yarp.ReverseProxy
```

**Configuración (appsettings.json):**

```json
{
  "ReverseProxy": {
    "Routes": {
      "products-route": {
        "ClusterId": "products-cluster",
        "Match": {
          "Path": "/products/{**catch-all}"
        },
        "Transforms": [
          { "PathPattern": "/api/products/{**catch-all}" }
        ]
      },
      "orders-route": {
        "ClusterId": "orders-cluster",
        "Match": {
          "Path": "/orders/{**catch-all}"
        }
      }
    },
    "Clusters": {
      "products-cluster": {
        "Destinations": {
          "products-service": {
            "Address": "https://localhost:5001"
          }
        }
      },
      "orders-cluster": {
        "Destinations": {
          "orders-service": {
            "Address": "https://localhost:5002"
          }
        }
      }
    }
  }
}
```

**Program.cs:**

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();
app.MapReverseProxy();
app.Run();
```

---

## 4.6 Service Discovery

### Conceptos

**Service Registry:** Base de datos de servicios disponibles y sus ubicaciones.

**Service Discovery:** Mecanismo para encontrar servicios dinámicamente.

**Tipos:**
- **Client-side:** El cliente consulta el registry y elige una instancia
- **Server-side:** Un load balancer consulta el registry

### Consul

Consul es una herramienta de service discovery y configuración.

**Instalación del cliente:**
```bash
dotnet add package Consul
```

**Registro de servicio:**

```csharp
public class ConsulServiceRegistration : IHostedService
{
    private readonly IConsulClient _consulClient;
    private readonly IConfiguration _configuration;
    private string _registrationId;
    
    public ConsulServiceRegistration(IConsulClient consulClient, IConfiguration configuration)
    {
        _consulClient = consulClient;
        _configuration = configuration;
    }
    
    public async Task StartAsync(CancellationToken cancellationToken)
    {
        var serviceName = _configuration["ServiceName"];
        var serviceId = $"{serviceName}-{Guid.NewGuid()}";
        var servicePort = int.Parse(_configuration["ServicePort"]);
        
        var registration = new AgentServiceRegistration
        {
            ID = serviceId,
            Name = serviceName,
            Address = "localhost",
            Port = servicePort,
            Check = new AgentServiceCheck
            {
                HTTP = $"http://localhost:{servicePort}/health",
                Interval = TimeSpan.FromSeconds(10),
                Timeout = TimeSpan.FromSeconds(5)
            }
        };
        
        await _consulClient.Agent.ServiceRegister(registration, cancellationToken);
        _registrationId = serviceId;
    }
    
    public async Task StopAsync(CancellationToken cancellationToken)
    {
        await _consulClient.Agent.ServiceDeregister(_registrationId, cancellationToken);
    }
}
```

**Configuración:**

```csharp
builder.Services.AddSingleton<IConsulClient>(p => new ConsulClient(config =>
{
    config.Address = new Uri("http://localhost:8500");
}));

builder.Services.AddHostedService<ConsulServiceRegistration>();
```

**Descubrimiento de servicios:**

```csharp
public class ServiceDiscovery
{
    private readonly IConsulClient _consulClient;
    
    public ServiceDiscovery(IConsulClient consulClient)
    {
        _consulClient = consulClient;
    }
    
    public async Task<Uri> GetServiceUriAsync(string serviceName)
    {
        var services = await _consulClient.Health.Service(serviceName, tag: null, passingOnly: true);
        
        if (!services.Response.Any())
        {
            throw new Exception($"Servicio {serviceName} no encontrado");
        }
        
        var service = services.Response.First().Service;
        return new Uri($"http://{service.Address}:{service.Port}");
    }
}
```

**Uso con HttpClient:**

```csharp
public class ProductService
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly ServiceDiscovery _serviceDiscovery;
    
    public async Task<Product> GetProductAsync(int id)
    {
        var serviceUri = await _serviceDiscovery.GetServiceUriAsync("product-service");
        var client = _httpClientFactory.CreateClient();
        client.BaseAddress = serviceUri;
        
        var response = await client.GetAsync($"api/products/{id}");
        return await response.Content.ReadFromJsonAsync<Product>();
    }
}
```

### Service Discovery en Kubernetes

En Kubernetes, el service discovery es nativo.

**Definición de Service:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: product-service
spec:
  selector:
    app: product
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

**Acceso desde otro servicio:**

```csharp
// El DNS interno de Kubernetes resuelve automáticamente
var httpClient = new HttpClient();
httpClient.BaseAddress = new Uri("http://product-service");

var response = await httpClient.GetAsync("/api/products/1");
```

**Con namespace:**

```csharp
// formato: http://{service-name}.{namespace}.svc.cluster.local
var uri = "http://product-service.production.svc.cluster.local";
```

### Client-side vs Server-side Discovery

**Client-side:**

```csharp
// Cliente consulta Consul y elige una instancia
public class ClientSideLoadBalancer
{
    private readonly IConsulClient _consul;
    private int _currentIndex = 0;
    
    public async Task<Uri> GetNextServiceInstance(string serviceName)
    {
        var services = await _consul.Health.Service(serviceName, passingOnly: true);
        var instances = services.Response.ToList();
        
        if (!instances.Any())
            throw new Exception("No hay instancias disponibles");
        
        // Round-robin simple
        var instance = instances[_currentIndex % instances.Count];
        _currentIndex++;
        
        return new Uri($"http://{instance.Service.Address}:{instance.Service.Port}");
    }
}
```

**Server-side (con Load Balancer):**

```csharp
// Cliente simplemente llama al load balancer
var httpClient = new HttpClient();
httpClient.BaseAddress = new Uri("http://load-balancer");
var response = await httpClient.GetAsync("/products");
```

---

## Resumen del Tema

En este tema hemos cubierto:

1. **Patrones de Comunicación:** Síncrona vs asíncrona, request/response, pub/sub, event-driven
2. **HTTP/REST:** HttpClient, HttpClientFactory, Polly para resiliencia, manejo de errores
3. **gRPC:** Protocol Buffers, implementación de servicios, tipos de streaming
4. **Message Brokers:** RabbitMQ, Azure Service Bus, MassTransit, Sagas
5. **API Gateway:** Ocelot, YARP, routing, agregación, rate limiting
6. **Service Discovery:** Consul, Kubernetes, client-side vs server-side

**Puntos clave:**
- Elegir comunicación síncrona para operaciones que requieren respuesta inmediata
- Usar comunicación asíncrona para desacoplar servicios y mejorar resiliencia
- Implementar patrones de resiliencia (retry, circuit breaker, timeout)
- Utilizar API Gateway como punto de entrada único
- Implementar service discovery para entornos dinámicos
- gRPC para comunicación interna de alto rendimiento
- Message brokers para eventos y procesamiento asíncrono
