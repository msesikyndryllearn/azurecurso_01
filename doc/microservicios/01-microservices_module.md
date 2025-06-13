# Módulo 8: Introducción a Microservicios con ASP.NET Core

**Duración:** 3 horas  
**Objetivo:** Comprender los fundamentos de la arquitectura de microservicios y aprender a crear microservicios básicos utilizando ASP.NET Core.

---

## 1. Fundamentos de Microservicios

### 1.1 ¿Qué son los Microservicios?

Los microservicios son un patrón de arquitectura donde una aplicación se estructura como una colección de servicios pequeños, independientes y débilmente acoplados. Cada microservicio es responsable de una funcionalidad específica del negocio y puede ser desarrollado, desplegado y escalado de forma independiente.

### 1.2 Características Principales

**Autonomía:** Cada microservicio opera de manera independiente con su propia base de datos y lógica de negocio.

**Responsabilidad única:** Cada servicio se enfoca en una capacidad de negocio específica.

**Comunicación a través de APIs:** Los servicios se comunican mediante protocolos bien definidos, típicamente HTTP/REST o mensajería asíncrona.

**Despliegue independiente:** Cada microservicio puede ser desplegado sin afectar otros servicios.

**Diversidad tecnológica:** Diferentes servicios pueden usar diferentes tecnologías y lenguajes de programación.

### 1.3 Ventajas de los Microservicios

**Escalabilidad independiente:** Permite escalar solo los servicios que lo necesiten, optimizando recursos.

**Flexibilidad tecnológica:** Cada equipo puede elegir la tecnología más adecuada para su servicio.

**Resistencia a fallos:** Un fallo en un servicio no necesariamente afecta a toda la aplicación.

**Desarrollo paralelo:** Diferentes equipos pueden trabajar simultáneamente en distintos servicios.

**Facilidad de mantenimiento:** Servicios más pequeños son más fáciles de entender y mantener.

### 1.4 Desafíos de los Microservicios

**Complejidad de red:** Mayor latencia y posibles fallos de comunicación entre servicios.

**Gestión de datos:** Mantener la consistencia de datos a través de múltiples servicios.

**Monitoreo y debugging:** Más difícil rastrear problemas a través de múltiples servicios.

**Overhead operacional:** Requiere más infraestructura y herramientas de gestión.

**Transacciones distribuidas:** Manejar transacciones que abarcan múltiples servicios.

### 1.5 Cuándo usar Microservicios

**Aplicaciones complejas:** Cuando la aplicación monolítica se vuelve difícil de mantener.

**Equipos grandes:** Cuando múltiples equipos necesitan trabajar independientemente.

**Escalabilidad diferenciada:** Cuando diferentes partes de la aplicación tienen diferentes requisitos de escalabilidad.

**Tecnologías diversas:** Cuando se necesita usar diferentes tecnologías para diferentes funcionalidades.

### 1.6 Patrones Fundamentales

**API Gateway:** Punto de entrada único que enruta requests a los microservicios apropiados.

**Service Discovery:** Mecanismo para que los servicios se encuentren y comuniquen entre sí.

**Circuit Breaker:** Patrón para prevenir cascadas de fallos entre servicios.

**Event Sourcing:** Almacenar eventos en lugar de estados para mantener consistencia.

**CQRS (Command Query Responsibility Segregation):** Separar operaciones de lectura y escritura.

---

## 2. Creación de Microservicios Simples con ASP.NET Core

### 2.1 Preparación del Entorno

Antes de comenzar, asegúrate de tener instalado:
- .NET 8 SDK o superior
- Visual Studio 2022 o Visual Studio Code
- Docker Desktop (opcional, para containerización)

### 2.2 Estructura de un Microservicio Básico

Un microservicio típico en ASP.NET Core incluye:
- Controladores para exponer APIs
- Servicios para lógica de negocio
- Repositorios para acceso a datos
- Modelos de dominio
- Configuración y middleware

### 2.3 Creando el Primer Microservicio: Servicio de Productos

**Paso 1: Crear el proyecto**
```bash
dotnet new webapi -n ProductService
cd ProductService
```

**Paso 2: Definir el modelo de dominio**
```csharp
// Models/Product.cs
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

**Paso 3: Crear el repositorio**
```csharp
// Repositories/IProductRepository.cs
public interface IProductRepository
{
    Task<IEnumerable<Product>> GetAllAsync();
    Task<Product> GetByIdAsync(int id);
    Task<Product> CreateAsync(Product product);
    Task<Product> UpdateAsync(Product product);
    Task<bool> DeleteAsync(int id);
}
```

**Paso 4: Implementar el repositorio en memoria**
```csharp
// Repositories/InMemoryProductRepository.cs
public class InMemoryProductRepository : IProductRepository
{
    private readonly List<Product> _products = new();
    private int _nextId = 1;

    public Task<IEnumerable<Product>> GetAllAsync()
    {
        return Task.FromResult(_products.AsEnumerable());
    }

    public Task<Product> GetByIdAsync(int id)
    {
        var product = _products.FirstOrDefault(p => p.Id == id);
        return Task.FromResult(product);
    }

    public Task<Product> CreateAsync(Product product)
    {
        product.Id = _nextId++;
        product.CreatedAt = DateTime.UtcNow;
        _products.Add(product);
        return Task.FromResult(product);
    }

    public Task<Product> UpdateAsync(Product product)
    {
        var existingProduct = _products.FirstOrDefault(p => p.Id == product.Id);
        if (existingProduct != null)
        {
            existingProduct.Name = product.Name;
            existingProduct.Description = product.Description;
            existingProduct.Price = product.Price;
            existingProduct.Stock = product.Stock;
        }
        return Task.FromResult(existingProduct);
    }

    public Task<bool> DeleteAsync(int id)
    {
        var product = _products.FirstOrDefault(p => p.Id == id);
        if (product != null)
        {
            _products.Remove(product);
            return Task.FromResult(true);
        }
        return Task.FromResult(false);
    }
}
```

**Paso 5: Crear el controlador**
```csharp
// Controllers/ProductsController.cs
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductRepository _repository;

    public ProductsController(IProductRepository repository)
    {
        _repository = repository;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<Product>>> GetAll()
    {
        var products = await _repository.GetAllAsync();
        return Ok(products);
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<Product>> GetById(int id)
    {
        var product = await _repository.GetByIdAsync(id);
        if (product == null)
            return NotFound();
        
        return Ok(product);
    }

    [HttpPost]
    public async Task<ActionResult<Product>> Create(Product product)
    {
        var createdProduct = await _repository.CreateAsync(product);
        return CreatedAtAction(nameof(GetById), new { id = createdProduct.Id }, createdProduct);
    }

    [HttpPut("{id}")]
    public async Task<ActionResult<Product>> Update(int id, Product product)
    {
        if (id != product.Id)
            return BadRequest();

        var updatedProduct = await _repository.UpdateAsync(product);
        if (updatedProduct == null)
            return NotFound();

        return Ok(updatedProduct);
    }

    [HttpDelete("{id}")]
    public async Task<ActionResult> Delete(int id)
    {
        var deleted = await _repository.DeleteAsync(id);
        if (!deleted)
            return NotFound();

        return NoContent();
    }
}
```

**Paso 6: Configurar los servicios**
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Registrar el repositorio
builder.Services.AddSingleton<IProductRepository, InMemoryProductRepository>();

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

### 2.4 Creando el Segundo Microservicio: Servicio de Usuarios

**Estructura similar al servicio de productos pero con funcionalidad de usuarios:**

```csharp
// Models/User.cs
public class User
{
    public int Id { get; set; }
    public string Username { get; set; }
    public string Email { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime CreatedAt { get; set; }
    public bool IsActive { get; set; }
}
```

### 2.5 Comunicación entre Microservicios

**Usando HttpClient para comunicación síncrona:**

```csharp
// Services/UserService.cs en ProductService
public class UserService
{
    private readonly HttpClient _httpClient;

    public UserService(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<User> GetUserByIdAsync(int userId)
    {
        var response = await _httpClient.GetAsync($"api/users/{userId}");
        if (response.IsSuccessStatusCode)
        {
            var json = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<User>(json);
        }
        return null;
    }
}
```

### 2.6 Configuración para Múltiples Microservicios

**appsettings.json para configurar URLs de servicios:**
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "Services": {
    "UserService": "https://localhost:7001",
    "ProductService": "https://localhost:7002"
  },
  "AllowedHosts": "*"
}
```

### 2.7 Mejores Prácticas

**Versionado de APIs:** Siempre versiona tus APIs para mantener compatibilidad hacia atrás.

**Health Checks:** Implementa endpoints de salud para monitoreo.
```csharp
builder.Services.AddHealthChecks();
app.MapHealthChecks("/health");
```

**Logging estructurado:** Usa logging consistente para facilitar el debugging.

**Manejo de errores:** Implementa manejo de errores global.

**Documentación:** Mantén la documentación actualizada usando Swagger/OpenAPI.

### 2.8 Containerización con Docker

**Dockerfile para el microservicio:**
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["ProductService.csproj", "."]
RUN dotnet restore "ProductService.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "ProductService.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "ProductService.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "ProductService.dll"]
```

**docker-compose.yml para orquestar múltiples servicios:**
```yaml
version: '3.8'
services:
  product-service:
    build: ./ProductService
    ports:
      - "7002:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
  
  user-service:
    build: ./UserService
    ports:
      - "7001:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
```

---

## 3. Ejercicios Prácticos

### Ejercicio 1: Crear un Microservicio de Órdenes
Crea un microservicio que maneje órdenes de compra que se comunique con los servicios de productos y usuarios.

### Ejercicio 2: Implementar un API Gateway Simple
Usa YARP (Yet Another Reverse Proxy) para crear un gateway que enrute requests a los diferentes microservicios.

### Ejercicio 3: Agregar Autenticación JWT
Implementa autenticación JWT que funcione across múltiples microservicios.

---

## 4. Recursos Adicionales

- **Documentación oficial de ASP.NET Core:** https://docs.microsoft.com/aspnet/core
- **Microservices.NET:** Guías y patrones específicos para .NET
- **Dapr:** Framework para microservicios que simplifica el desarrollo
- **Docker:** Para containerización y orquestación
- **Kubernetes:** Para deployment y gestión de contenedores

---

## 5. Próximos Pasos

Una vez dominados estos conceptos básicos, considera explorar:
- Event-driven architectures con Message Brokers (RabbitMQ, Azure Service Bus)
- Service Mesh con Istio o Linkerd
- Observabilidad con OpenTelemetry
- Deployment avanzado con Kubernetes
- Patrones avanzados como Saga y Event Sourcing

---

*Este módulo proporciona una base sólida para entender y comenzar a trabajar con microservicios en ASP.NET Core. La práctica y la experimentación son clave para dominar estos conceptos.*