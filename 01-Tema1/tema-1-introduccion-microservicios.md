# Tema 1: Introducción a los Microservicios

## 1.1 ¿Qué son los Microservicios?

### Definición y conceptos fundamentales

Los **microservicios** son un estilo arquitectónico donde una aplicación se construye como un conjunto de **servicios pequeños e independientes**, cada uno ejecutándose en su propio proceso y comunicándose mediante protocolos ligeros (generalmente HTTP/REST o mensajería).

**Características clave:**
- Cada servicio implementa una capacidad de negocio específica
- Pueden desplegarse independientemente
- Son mantenidos por equipos pequeños
- Pueden estar escritos en diferentes tecnologías

**Ejemplo simple:**

Imagina una tienda online como un solo sistema grande (monolito):
```
[Tienda Online Completa]
- Gestión de usuarios
- Catálogo de productos
- Carrito de compras
- Procesamiento de pagos
- Envíos
- Notificaciones
```

Con microservicios, se dividiría en:
```
[Servicio Usuarios] [Servicio Productos] [Servicio Carrito]
[Servicio Pagos] [Servicio Envíos] [Servicio Notificaciones]
```

Cada uno funciona independientemente y se comunica con los demás cuando es necesario.

### Historia y evolución de la arquitectura de software

**Línea temporal:**

1. **Años 60-80: Arquitectura Mainframe**
   - Aplicaciones monolíticas centralizadas
   - Todo el código en un único sistema

2. **Años 90-2000: Arquitectura Cliente-Servidor**
   - Separación entre interfaz de usuario y lógica de negocio
   - Bases de datos centralizadas

3. **Años 2000-2010: Service-Oriented Architecture (SOA)**
   - Servicios empresariales más grandes
   - ESB (Enterprise Service Bus) como intermediario
   - Servicios acoplados entre sí

4. **2011-presente: Microservicios**
   - Netflix, Amazon, Uber popularizaron el enfoque
   - Servicios pequeños y desacoplados
   - DevOps y cultura de automatización
   - Contenedores (Docker) y orquestación (Kubernetes)

**Ejemplo de evolución:**

Sistema de Netflix a lo largo del tiempo:
- **2000s**: Aplicación monolítica en servidores propios
- **2008**: Problema de base de datos corrupta → 3 días sin servicio
- **2009-2012**: Migración gradual a AWS y microservicios
- **Actualidad**: +700 microservicios independientes

### Principios de diseño de microservicios

#### 1. **Responsabilidad única**
Cada servicio hace una cosa y la hace bien.

```csharp
// ❌ MAL: Servicio que hace demasiado
public class UserService
{
    void CreateUser() { }
    void SendEmail() { }
    void ProcessPayment() { }
    void GenerateReport() { }
}

// ✅ BIEN: Servicios separados
UserService       → CreateUser()
EmailService      → SendEmail()
PaymentService    → ProcessPayment()
ReportingService  → GenerateReport()
```

#### 2. **Autonomía**
Cada servicio puede desarrollarse, desplegarse y escalarse independientemente.

**Ejemplo:**
El Servicio de Productos puede actualizarse sin afectar al Servicio de Pagos.

#### 3. **Descentralización**
No hay un único punto de control. Cada equipo toma decisiones sobre su servicio.

**Ejemplo:**
- Equipo A usa SQL Server para el Servicio de Usuarios
- Equipo B usa MongoDB para el Servicio de Productos
- Ambas decisiones son válidas según sus necesidades

#### 4. **Resistencia a fallos**
Los servicios deben diseñarse asumiendo que otros servicios pueden fallar.

**Ejemplo:**
```csharp
public async Task<Order> GetOrderDetails(int orderId)
{
    var order = await _orderRepository.GetById(orderId);
    
    try
    {
        order.UserInfo = await _userService.GetUser(order.UserId);
    }
    catch (Exception)
    {
        // Si el servicio de usuarios falla, continuamos sin esa info
        order.UserInfo = new UserInfo { Name = "Usuario no disponible" };
    }
    
    return order;
}
```

#### 5. **Orientación al negocio**
Los servicios se organizan alrededor de capacidades de negocio, no de capas técnicas.

**Ejemplo:**
```
❌ Organización por capas técnicas:
- ServicioUI
- ServicioLógicaNegocio
- ServicioAccesoDatos

✅ Organización por capacidades de negocio:
- ServicioGestiónPedidos
- ServicioGestiónInventario
- ServicioGestiónClientes
```

## 1.2 Arquitectura Monolítica vs Microservicios

### Características de la arquitectura monolítica

Una aplicación monolítica es un **único bloque de código** donde todos los componentes están interconectados y son interdependientes.

**Estructura típica:**

```
Aplicación Monolítica
│
├── Capa de Presentación (UI)
│   ├── Controllers
│   └── Views
│
├── Capa de Lógica de Negocio
│   ├── Módulo Usuarios
│   ├── Módulo Productos
│   ├── Módulo Pedidos
│   └── Módulo Pagos
│
└── Capa de Acceso a Datos
    └── Base de Datos única
```

**Características:**
- Todo se despliega junto
- Una única base de código
- Comunicación interna (llamadas a métodos)
- Una base de datos compartida
- Escalado vertical (servidor más potente)

**Ejemplo de código monolítico:**

```csharp
// Toda la lógica en una sola aplicación
public class OrderController : Controller
{
    private readonly AppDbContext _db;
    
    public OrderController(AppDbContext db)
    {
        _db = db;
    }
    
    public IActionResult CreateOrder(OrderDto dto)
    {
        // Validar usuario
        var user = _db.Users.Find(dto.UserId);
        
        // Validar productos
        var product = _db.Products.Find(dto.ProductId);
        
        // Crear pedido
        var order = new Order { /*...*/ };
        _db.Orders.Add(order);
        
        // Procesar pago
        var payment = new Payment { /*...*/ };
        _db.Payments.Add(payment);
        
        // Enviar notificación
        SendEmail(user.Email, "Pedido confirmado");
        
        _db.SaveChanges();
        return Ok(order);
    }
}
```

### Ventajas y desventajas del monolito

#### Ventajas ✅

1. **Simplicidad inicial**
   - Fácil de desarrollar al principio
   - Un solo proyecto para gestionar

2. **Despliegue sencillo**
   - Un único artefacto (DLL, ejecutable)
   - No necesitas coordinar múltiples servicios

3. **Debugging más fácil**
   - Todo el código está en un lugar
   - Puedes depurar de principio a fin

4. **Testing más simple**
   - Tests de integración en un solo proceso
   - No necesitas levantar múltiples servicios

5. **Sin latencia de red**
   - Las llamadas entre módulos son directas en memoria

**Ejemplo:**
```csharp
// En un monolito, esto es instantáneo
var user = userService.GetUser(userId);
var orders = orderService.GetUserOrders(userId);
```

#### Desventajas ❌

1. **Escalabilidad limitada**
   - Tienes que escalar toda la aplicación, aunque solo una parte necesite más recursos

**Ejemplo:**
```
Monolito: Si el módulo de reportes consume muchos recursos,
debes escalar toda la aplicación (incluyendo usuarios, productos, etc.)

Microservicios: Solo escalas el servicio de reportes
```

2. **Despliegues riesgosos**
   - Un pequeño cambio requiere redesplegar toda la aplicación
   - Mayor riesgo de afectar todo el sistema

3. **Acoplamiento alto**
   - Los módulos están fuertemente conectados
   - Cambiar una parte puede romper otras

4. **Difícil de mantener a largo plazo**
   - El código crece y se vuelve complejo
   - Los nuevos desarrolladores tardan en entender todo el sistema

5. **Limitación tecnológica**
   - Todo debe estar en la misma tecnología
   - Difícil actualizar frameworks

**Ejemplo de acoplamiento:**
```csharp
// Cambiar la clase User afecta a múltiples módulos
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    // Si añades un campo nuevo, puede romper:
    // - Módulo de pedidos
    // - Módulo de reportes
    // - Módulo de notificaciones
}
```

### Transición de monolito a microservicios

La migración debe ser **gradual**, no de golpe.

**Estrategia recomendada: Strangler Fig Pattern**

Rodeas el monolito con nuevos servicios que van "estrangulando" poco a poco la funcionalidad antigua.

```
Fase 1: Monolito completo
[MONOLITO]

Fase 2: Extraer primer servicio
[MONOLITO] ← → [Servicio Notificaciones]

Fase 3: Extraer más servicios
[MONOLITO] ← → [Notificaciones]
     ↓ ↑        [Pagos]
     ↓ ↑        [Envíos]

Fase 4: El monolito se reduce
[Core Monolito] ← → [Notificaciones]
                     [Pagos]
                     [Envíos]
                     [Reportes]
```

**Pasos prácticos:**

1. **Identificar límites** (bounded contexts)
2. **Extraer servicios periféricos primero** (notificaciones, reportes)
3. **Extraer servicios críticos al final** (usuarios, pedidos)
4. **Mantener compatibilidad** durante la transición

**Ejemplo de transición:**

```csharp
// ANTES: Todo en el monolito
public class OrderService
{
    public void CreateOrder(Order order)
    {
        SaveOrder(order);
        SendEmailNotification(order);  // Lógica interna
    }
}

// DURANTE LA TRANSICIÓN: Llamada al nuevo servicio
public class OrderService
{
    private readonly INotificationService _notificationService;
    
    public async Task CreateOrder(Order order)
    {
        SaveOrder(order);
        await _notificationService.SendEmail(order);  // Servicio externo
    }
}

// DESPUÉS: El servicio de notificaciones es independiente
// ServicioNotificaciones (otro proyecto/servicio)
[ApiController]
[Route("api/notifications")]
public class NotificationController : ControllerBase
{
    [HttpPost("email")]
    public IActionResult SendEmail([FromBody] EmailRequest request)
    {
        // Lógica de envío de email
        return Ok();
    }
}
```

### Casos de uso apropiados para cada arquitectura

#### Cuándo usar MONOLITO ✅

1. **Proyecto pequeño o startup**
   - Equipo de 1-5 desarrolladores
   - MVP (Minimum Viable Product)
   
2. **Dominio simple**
   - Pocas funcionalidades
   - Lógica de negocio no compleja

3. **Equipo sin experiencia en microservicios**
   - Curva de aprendizaje empinada
   - Mejor empezar simple

4. **Requisitos de rendimiento estrictos**
   - Latencia de red puede ser problema
   - Necesitas máxima velocidad

**Ejemplo:**
Blog personal, aplicación interna de gestión para PYME, prototipo para validar idea de negocio.

#### Cuándo usar MICROSERVICIOS ✅

1. **Aplicación grande y compleja**
   - Múltiples dominios de negocio
   - Equipos de 20+ desarrolladores

2. **Necesidad de escalar selectivamente**
   - Diferentes partes tienen diferentes cargas

**Ejemplo:**
```
Netflix:
- Servicio de streaming: Alta carga, necesita muchas instancias
- Servicio de facturación: Baja carga, pocas instancias
```

3. **Equipos distribuidos**
   - Varios equipos trabajando en paralelo
   - Autonomía en decisiones técnicas

4. **Diversidad tecnológica**
   - Necesitas usar diferentes lenguajes/bases de datos

5. **Ciclos de despliegue frecuentes**
   - Despliegues independientes
   - Continuous Deployment

**Ejemplo:**
E-commerce grande (Amazon), plataforma de streaming (Netflix), aplicación bancaria, sistema de reservas de viajes.

## 1.3 Ventajas de los Microservicios

### Escalabilidad independiente

Cada servicio se escala según sus necesidades específicas.

**Ejemplo práctico:**

```
Sistema de E-commerce en Black Friday:

Servicio de Catálogo: 10 instancias (mucha gente mirando productos)
Servicio de Checkout: 5 instancias (algunos comprando)
Servicio de Administración: 1 instancia (uso interno, poco tráfico)
```

En un monolito, tendrías que escalar toda la aplicación, desperdiciando recursos.

**Código de ejemplo:**

```yaml
# Kubernetes: Escalar solo el servicio de catálogo
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog-service
spec:
  replicas: 10  # 10 instancias solo para este servicio
```

### Despliegue continuo y deployment independiente

Puedes actualizar un servicio sin tocar los demás.

**Ejemplo:**

```
Lunes: Desplegar nueva funcionalidad en Servicio de Notificaciones
       (Los demás servicios siguen funcionando sin cambios)

Miércoles: Corregir bug en Servicio de Pagos
           (No afecta a Notificaciones, Productos, etc.)
```

**Beneficios:**
- Menos riesgo en cada despliegue
- Despliegues más frecuentes
- Rollback más rápido si algo falla

```csharp
// Versiones independientes
Servicio Usuarios: v2.3.1
Servicio Productos: v1.8.5
Servicio Pagos: v3.0.2
// Cada uno con su propio ciclo de vida
```

### Resiliencia y tolerancia a fallos

Si un servicio falla, los demás pueden seguir funcionando.

**Ejemplo:**

```
Escenario: El servicio de Recomendaciones cae

❌ Monolito: Toda la aplicación falla
✅ Microservicios: 
   - Productos: ✓ Funciona
   - Carrito: ✓ Funciona
   - Checkout: ✓ Funciona
   - Recomendaciones: ✗ No funciona (muestra mensaje alternativo)
```

**Implementación con Circuit Breaker:**

```csharp
public async Task<List<Product>> GetRecommendations(int userId)
{
    try
    {
        return await _recommendationService.GetRecommendations(userId);
    }
    catch (Exception)
    {
        // Si el servicio de recomendaciones falla, 
        // mostramos productos populares en su lugar
        return await _productService.GetPopularProducts();
    }
}
```

### Flexibilidad tecnológica

Cada servicio puede usar la tecnología más apropiada.

**Ejemplo:**

```
Servicio de Usuarios: .NET + SQL Server
  (Transacciones, datos estructurados)

Servicio de Catálogo: Node.js + MongoDB
  (Alto rendimiento, datos no estructurados)

Servicio de Búsqueda: Java + Elasticsearch
  (Búsqueda de texto completo)

Servicio de Notificaciones: Python + Redis
  (Cola de mensajes rápida)
```

### Equipos autónomos y organizados por dominio

Cada equipo es dueño de su servicio completo.

**Organización tradicional (monolito):**
```
Equipo Frontend → Todas las pantallas
Equipo Backend → Toda la lógica
Equipo BD → Toda la base de datos
```

**Organización por microservicios:**
```
Equipo Pedidos:
  - Frontend de pedidos
  - Backend de pedidos
  - BD de pedidos
  - Despliegue de pedidos

Equipo Pagos:
  - Frontend de pagos
  - Backend de pagos
  - BD de pagos
  - Despliegue de pagos
```

**Beneficio:** Cada equipo puede tomar decisiones rápidas sin depender de otros.

### Mantenibilidad y evolución del software

El código es más fácil de entender y modificar.

**Comparación:**

```
Monolito:
- 500,000 líneas de código
- 200 archivos
- Entender todo lleva meses

Microservicio individual:
- 5,000 líneas de código
- 20 archivos
- Entender el servicio lleva días
```

**Ejemplo de evolución:**

```csharp
// Puedes reescribir completamente un servicio sin afectar a otros

// Versión 1: Servicio de Pagos en .NET Framework 4.8
// Versión 2: Reescribirlo en .NET 8 gradualmente
// Los demás servicios no se enteran del cambio interno
```

## 1.4 Desafíos y Desventajas

### Complejidad operacional

Gestionar 10 servicios es más complejo que gestionar 1 aplicación.

**Ejemplo:**

```
Monolito:
- 1 aplicación para monitorear
- 1 log para revisar
- 1 despliegue

Microservicios (10 servicios):
- 10 aplicaciones para monitorear
- 10 logs para revisar (y correlacionar)
- 10 despliegues para coordinar
```

**Herramientas necesarias:**
- Kubernetes para orquestación
- Prometheus para métricas
- Elasticsearch para logs centralizados
- Jaeger para tracing distribuido

### Gestión de datos distribuidos

No hay una base de datos central.

**Problema:**

```csharp
// En un monolito, esto es fácil:
using (var transaction = db.BeginTransaction())
{
    db.Users.Add(user);
    db.Orders.Add(order);
    transaction.Commit();  // Ambos se guardan o ninguno
}

// En microservicios, están en diferentes bases de datos:
// ¿Cómo garantizamos consistencia?
await userService.CreateUser(user);      // BD Usuarios
await orderService.CreateOrder(order);   // BD Pedidos
// Si el segundo falla, ¿qué hacemos con el usuario creado?
```

**Solución:** Saga Pattern (lo veremos en detalle más adelante)

### Latencia de red y comunicación entre servicios

Las llamadas entre servicios son llamadas HTTP, no llamadas a métodos locales.

**Comparación:**

```csharp
// Monolito: 0.01 ms (en memoria)
var user = _userService.GetUser(userId);

// Microservicios: 50-200 ms (red)
var user = await _httpClient.GetAsync("http://user-service/api/users/" + userId);
```

**Impacto:**
Si una operación necesita llamar a 5 servicios, la latencia se multiplica.

```
Operación que requiere datos de 5 servicios:
- Monolito: 0.05 ms
- Microservicios: 250-1000 ms (5 × 50-200ms)
```

### Testing y debugging distribuido

**Desafío del debugging:**

```
Usuario reporta: "No puedo completar mi pedido"

Monolito:
1. Revisar logs
2. Encontrar el error en un lugar
3. Depurar localmente

Microservicios:
1. ¿Qué servicio falló? (Usuarios, Productos, Carrito, Pagos, Envíos?)
2. Revisar logs de cada servicio
3. Correlacionar IDs entre servicios
4. Reproducir el escenario con múltiples servicios levantados
```

**Solución:** Correlation IDs

```csharp
// Cada request tiene un ID único que se propaga
Request → [CorrelationId: abc-123]
  → Servicio Usuarios [CorrelationId: abc-123]
  → Servicio Pedidos [CorrelationId: abc-123]
  → Servicio Pagos [CorrelationId: abc-123]
// Buscas "abc-123" en todos los logs
```

### Consistencia de datos

Los datos están distribuidos, no hay transacciones ACID tradicionales.

**Ejemplo del problema:**

```
Operación: Reservar producto y crear pedido

1. Servicio Inventario: Reservar producto ✓
2. Servicio Pedidos: Crear pedido ✗ (falla)

Resultado: Producto reservado pero sin pedido creado
```

**Tipos de consistencia:**

**Consistencia fuerte** (monolito):
```csharp
// Todo o nada, inmediatamente
using var transaction = db.BeginTransaction();
db.Products.Update(product);
db.Orders.Add(order);
transaction.Commit();
```

**Consistencia eventual** (microservicios):
```csharp
// Los datos se sincronizan "eventualmente"
await productService.ReserveProduct(productId);
await orderService.CreateOrder(order);
// Puede haber un breve momento donde están desincronizados
```

### Curva de aprendizaje

Nuevos conceptos y tecnologías que aprender.

**Conocimientos necesarios:**
- Contenedores (Docker)
- Orquestación (Kubernetes)
- Message Brokers (RabbitMQ, Kafka)
- Service Discovery
- API Gateways
- Distributed Tracing
- Circuit Breakers
- Event-Driven Architecture

**Comparación:**

```
Desarrollador de monolitos:
- .NET
- SQL Server
- IIS
Total: 3 tecnologías principales

Desarrollador de microservicios:
- .NET, Docker, Kubernetes
- SQL Server, MongoDB, Redis
- RabbitMQ, gRPC, REST
- Prometheus, Grafana, Jaeger
Total: 12+ tecnologías principales
```

## 1.5 Cuándo Usar Microservicios

### Criterios de decisión

**Tabla de decisión:**

| Criterio | Monolito | Microservicios |
|----------|----------|----------------|
| **Tamaño del equipo** | 1-10 personas | 10+ personas |
| **Complejidad del dominio** | Baja/Media | Alta |
| **Necesidad de escalado** | Uniforme | Selectivo |
| **Frecuencia de despliegue** | Semanal/Mensual | Diaria/Continua |
| **Madurez DevOps** | Básica | Avanzada |
| **Presupuesto** | Limitado | Amplio |

### Tamaño y complejidad del proyecto

**Proyecto pequeño:**
```
Aplicación de gestión para 5 personas
- 3 módulos
- 1 base de datos
- Actualizaciones mensuales
→ MONOLITO
```

**Proyecto grande:**
```
Plataforma de e-commerce internacional
- 50 dominios de negocio
- 100+ desarrolladores
- Despliegues diarios
→ MICROSERVICIOS
```

### Madurez del equipo

**Señales de que el equipo está listo:**
- Experiencia con Docker y Kubernetes
- Cultura DevOps establecida
- Pipelines CI/CD funcionando
- Conocimiento de arquitecturas distribuidas
- Presupuesto para herramientas de observabilidad

**Señales de que el equipo NO está listo:**
- Primera vez usando .NET
- No hay experiencia con contenedores
- Despliegues manuales
- Sin monitoreo automatizado

### Requisitos de escalabilidad

**Escenario que requiere microservicios:**

```
Aplicación de streaming de video:

Servicio de Video Streaming:
- Millones de requests por segundo
- Alto consumo de ancho de banda
- Necesita 100 instancias

Servicio de Facturación:
- 1000 requests por día
- Lógica compleja pero bajo tráfico
- Necesita 1 instancia

Con monolito: Tendrías que escalar todo a 100 instancias
Con microservicios: Escalas solo lo necesario
```

### Anti-patrones y cuándo NO usar microservicios

#### ❌ Anti-patrón 1: Microservicios distribuidos

```
Cada método de una clase = un microservicio

[ServicioGetUser]
[ServicioUpdateUser]
[ServicioDeleteUser]
[ServicioListUsers]

Problema: Demasiada granularidad, overhead de red excesivo
```

#### ❌ Anti-patrón 2: Base de datos compartida

```
[ServicioUsuarios] ──┐
                      ├──→ [Base de Datos Única]
[ServicioPedidos] ───┘

Problema: No hay independencia real, acoplamiento por BD
```

#### ❌ Anti-patrón 3: Microservicios para proyectos simples

```
Blog personal con:
- 1 desarrollador
- 3 páginas
- 10 usuarios al día

No necesitas microservicios, es sobre-ingeniería
```

#### ❌ Cuándo NO usar microservicios:

1. **Startup en fase inicial**
   - Necesitas velocidad, no escalabilidad
   - El producto puede cambiar completamente

2. **Equipo sin experiencia**
   - Aprende primero con monolitos
   - Después migra gradualmente

3. **Presupuesto limitado**
   - Microservicios requieren más infraestructura
   - Herramientas de monitoreo tienen costo

4. **Requisitos de latencia ultra-baja**
   - Trading de alta frecuencia
   - Sistemas en tiempo real crítico

## 1.6 Principios de Diseño

### Single Responsibility Principle

Cada servicio tiene **una única razón para cambiar**.

**Ejemplo:**

```csharp
// ❌ MAL: Servicio con múltiples responsabilidades
public class CustomerService
{
    void CreateCustomer() { }
    void SendWelcomeEmail() { }
    void ProcessPayment() { }
    void GenerateInvoice() { }
    void UpdateInventory() { }
}
// Si cambian las reglas de email, el servicio cambia
// Si cambian las reglas de facturación, el servicio cambia
// Demasiadas razones para cambiar

// ✅ BIEN: Un servicio, una responsabilidad
CustomerService      → CreateCustomer()
EmailService         → SendWelcomeEmail()
PaymentService       → ProcessPayment()
InvoicingService     → GenerateInvoice()
InventoryService     → UpdateInventory()
```

### Domain-Driven Design (DDD) básico

DDD ayuda a **modelar el software según el negocio**.

**Conceptos clave:**

1. **Dominio:** El área de conocimiento del negocio
2. **Subdominio:** Partes específicas del dominio
3. **Bounded Context:** Límites explícitos de un modelo

**Ejemplo de dominios:**

```
Dominio: E-commerce

Subdominios:
├── Gestión de Catálogo
│   - Productos
│   - Categorías
│   - Inventario
├── Gestión de Ventas
│   - Pedidos
│   - Carrito de compras
│   - Checkout
├── Gestión de Clientes
│   - Usuarios
│   - Perfiles
│   - Historial
└── Gestión de Pagos
    - Transacciones
    - Facturación
    - Reembolsos
```

Cada subdominio puede ser un microservicio.

### Bounded Contexts

Un **Bounded Context** es un límite explícito donde un modelo de dominio aplica.

**Ejemplo:**

La entidad "Cliente" significa cosas diferentes en diferentes contextos:

```csharp
// Bounded Context: Ventas
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal CreditLimit { get; set; }
    public List<Order> Orders { get; set; }
}

// Bounded Context: Soporte
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string PreferredContactMethod { get; set; }
    public List<Ticket> SupportTickets { get; set; }
}

// Bounded Context: Marketing
public class Customer
{
    public int Id { get; set; }
    public string Email { get; set; }
    public List<string> Interests { get; set; }
    public DateTime LastCampaignSent { get; set; }
}
```

Son la misma entidad "Cliente" pero con **diferentes propiedades y comportamientos** según el contexto.

**Regla:** Cada Bounded Context puede ser un microservicio.

### Principio de autonomía

Cada servicio debe poder **funcionar independientemente**.

**Ejemplo de servicio NO autónomo (mal):**

```csharp
// Servicio de Pedidos depende directamente del Servicio de Usuarios
public async Task<Order> CreateOrder(CreateOrderDto dto)
{
    // Llamada síncrona - si UserService cae, este servicio falla
    var user = await _userService.GetUser(dto.UserId);
    
    if (user == null)
        throw new Exception("Usuario no encontrado");
    
    var order = new Order
    {
        UserId = user.Id,
        UserName = user.Name  // Dependencia de datos
    };
    
    return order;
}
```

**Ejemplo de servicio autónomo (bien):**

```csharp
// Servicio de Pedidos tiene sus propios datos de usuario
public async Task<Order> CreateOrder(CreateOrderDto dto)
{
    // Copia local de datos del usuario
    var userInfo = await _localUserRepository.GetUserInfo(dto.UserId);
    
    if (userInfo == null)
    {
        // Sincronizar datos si no existen localmente
        userInfo = await SyncUserInfo(dto.UserId);
    }
    
    var order = new Order
    {
        UserId = userInfo.Id,
        UserName = userInfo.Name  // Datos locales, no dependencia en tiempo real
    };
    
    return order;
}
```

### Diseño por capacidades de negocio

Los servicios se organizan por **qué hacen en el negocio**, no por cómo funcionan técnicamente.

**Ejemplo incorrecto (organización técnica):**

```
[Servicio de Base de Datos]
[Servicio de Validación]
[Servicio de API]
[Servicio de UI]
```

**Ejemplo correcto (capacidades de negocio):**

```
[Servicio de Gestión de Inventario]
  - Controlar stock
  - Alertar de productos bajos
  - Proyectar demanda

[Servicio de Cumplimiento de Pedidos]
  - Procesar pedidos
  - Asignar productos
  - Coordinar envíos

[Servicio de Experiencia del Cliente]
  - Gestionar perfiles
  - Rastrear preferencias
  - Personalizar contenido
```

**Pregunta clave:** ¿Qué capacidad del negocio provee este servicio?

---

## Resumen del Tema 1

### Conceptos clave aprendidos:

1. **Microservicios** = Servicios pequeños, independientes, organizados por capacidades de negocio

2. **Ventajas principales:**
   - Escalabilidad independiente
   - Despliegue continuo
   - Resiliencia
   - Flexibilidad tecnológica

3. **Desafíos principales:**
   - Complejidad operacional
   - Gestión de datos distribuidos
   - Latencia de red
   - Testing distribuido

4. **Usar microservicios cuando:**
   - Equipos grandes (10+ personas)
   - Dominio complejo
   - Necesidad de escalado selectivo
   - Cultura DevOps madura

5. **NO usar microservicios cuando:**
   - Proyecto pequeño o startup inicial
   - Equipo sin experiencia
   - Presupuesto limitado
   - Requisitos de latencia ultra-baja

6. **Principios de diseño:**
   - Single Responsibility Principle
   - Domain-Driven Design
   - Bounded Contexts
   - Autonomía de servicios
   - Diseño por capacidades de negocio

### Próximos pasos:

En el **Tema 2** veremos cómo .NET Core nos proporciona las herramientas necesarias para implementar microservicios, incluyendo:
- ASP.NET Core Web API
- Dependency Injection
- Configuración
- Middleware
- Logging

---

**Recursos adicionales recomendados:**
- Libro: "Building Microservices" de Sam Newman
- Documentación oficial de .NET: https://docs.microsoft.com
- Patrones de microservicios: https://microservices.io/patterns/
