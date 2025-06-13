# Módulo 9: Despliegue de Microservicios en Azure Container Apps

**Duración:** 3 horas  
**Objetivo:** Aprender a desplegar y gestionar microservicios en Azure Container Apps, implementando autenticación y manejo de revisiones para un entorno de producción escalable.

---

## 1. Introducción a Azure Container Apps

### 1.1 ¿Qué son Azure Container Apps?

Azure Container Apps es un servicio de aplicaciones de contenedores totalmente administrado que permite ejecutar microservicios y aplicaciones containerizadas sin gestionar la infraestructura subyacente. Está construido sobre Kubernetes pero abstrae la complejidad de la orquestación.

### 1.2 Características Principales

**Serverless:** Escala automáticamente desde cero según la demanda.

**Multi-container:** Soporte para aplicaciones con múltiples contenedores.

**Traffic splitting:** Divide el tráfico entre diferentes revisiones de la aplicación.

**Autenticación integrada:** Soporte nativo para proveedores de identidad.

**Event-driven scaling:** Escala basado en eventos de Azure Service Bus, Storage Queue, etc.

**Networking avanzado:** VNET integration, ingress personalizado, y comunicación entre servicios.

### 1.3 Componentes Clave

**Container App:** La aplicación que contiene uno o más contenedores.

**Container App Environment:** Entorno seguro donde se ejecutan las Container Apps.

**Revision:** Versión inmutable de una Container App.

**Traffic Rules:** Reglas que determinan cómo se distribuye el tráfico entre revisiones.

**Ingress:** Configuración de entrada HTTP/HTTPS para las aplicaciones.

### 1.4 Ventajas sobre otras opciones

**Simplicidad:** Menos configuración que AKS, más potente que Azure Web Apps.

**Costo-efectivo:** Paga solo por recursos consumidos.

**Integración nativa:** Perfecta integración con servicios de Azure.

**Developer-friendly:** Enfocado en la experiencia del desarrollador.

---

## 2. Preparación del Entorno

### 2.1 Requisitos Previos

- Suscripción activa de Azure
- Azure CLI instalado y configurado
- Docker Desktop para crear imágenes de contenedores
- Visual Studio Code con extensiones de Azure
- Conocimiento básico de contenedores y Docker

### 2.2 Instalación de Herramientas

**Azure CLI:**
```bash
# Verificar instalación
az version

# Instalar extensión de Container Apps
az extension add --name containerapp --upgrade

# Login a Azure
az login
```

**Docker:**
```bash
# Verificar instalación
docker --version
docker-compose --version
```

### 2.3 Configuración inicial de Azure

**Crear grupo de recursos:**
```bash
# Variables de configuración
RESOURCE_GROUP="rg-microservices-demo"
LOCATION="eastus"
ENVIRONMENT_NAME="env-microservices"

# Crear grupo de recursos
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

**Crear Container Apps Environment:**
```bash
# Crear el entorno
az containerapp env create \
  --name $ENVIRONMENT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION
```

---

## 3. Despliegue y Gestión de Microservicios

### 3.1 Preparación de la Aplicación para Container Apps

**Dockerfile optimizado para producción:**
```dockerfile
# Dockerfile para ProductService
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS base
WORKDIR /app
EXPOSE 8080

# Crear usuario no-root para seguridad
RUN addgroup -g 1001 -S appuser && adduser -S appuser -G appuser -u 1001
USER appuser

FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src

# Copiar archivos del proyecto
COPY ["ProductService.csproj", "."]
RUN dotnet restore "ProductService.csproj"

COPY . .
RUN dotnet build "ProductService.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "ProductService.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Configurar variables de entorno
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production

ENTRYPOINT ["dotnet", "ProductService.dll"]
```

**appsettings.Production.json:**
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "Services": {
    "UserService": "https://userservice.internal",
    "ProductService": "https://productservice.internal"
  },
  "ConnectionStrings": {
    "DefaultConnection": ""
  }
}
```

### 3.2 Construcción y Push de Imágenes

**Azure Container Registry (ACR):**
```bash
# Variables
ACR_NAME="acrmicroservicesdemo"
IMAGE_NAME="productservice"
IMAGE_TAG="v1.0.0"

# Crear ACR
az acr create \
  --resource-group $RESOURCE_GROUP \
  --name $ACR_NAME \
  --sku Standard \
  --admin-enabled true

# Build y push de imagen
az acr build \
  --registry $ACR_NAME \
  --image $IMAGE_NAME:$IMAGE_TAG \
  --file Dockerfile .
```

**Usando Docker Hub (alternativa):**
```bash
# Build local
docker build -t yourusername/productservice:v1.0.0 .

# Push a Docker Hub
docker push yourusername/productservice:v1.0.0
```

### 3.3 Despliegue del Primer Microservicio

**Container App YAML (product-service.yaml):**
```yaml
location: eastus
resourceGroup: rg-microservices-demo
type: Microsoft.App/containerApps
apiVersion: 2022-10-01
name: productservice
properties:
  environmentId: /subscriptions/{subscription-id}/resourceGroups/rg-microservices-demo/providers/Microsoft.App/managedEnvironments/env-microservices
  configuration:
    activeRevisionsMode: Multiple
    ingress:
      external: true
      targetPort: 8080
      transport: http
      allowInsecure: false
    registries:
    - server: acrmicroservicesdemo.azurecr.io
      username: acrmicroservicesdemo
      passwordSecretRef: registry-password
    secrets:
    - name: registry-password
      value: "{acr-password}"
  template:
    containers:
    - image: acrmicroservicesdemo.azurecr.io/productservice:v1.0.0
      name: productservice
      env:
      - name: ASPNETCORE_ENVIRONMENT
        value: "Production"
      resources:
        cpu: 0.5
        memory: 1.0Gi
    scale:
      minReplicas: 1
      maxReplicas: 10
      rules:
      - name: http-rule
        http:
          metadata:
            concurrentRequests: 100
```

**Despliegue usando Azure CLI:**
```bash
# Obtener credenciales de ACR
ACR_PASSWORD=$(az acr credential show --name $ACR_NAME --query "passwords[0].value" -o tsv)

# Crear Container App
az containerapp create \
  --name productservice \
  --resource-group $RESOURCE_GROUP \
  --environment $ENVIRONMENT_NAME \
  --image $ACR_NAME.azurecr.io/productservice:v1.0.0 \
  --target-port 8080 \
  --ingress external \
  --registry-server $ACR_NAME.azurecr.io \
  --registry-username $ACR_NAME \
  --registry-password $ACR_PASSWORD \
  --env-vars ASPNETCORE_ENVIRONMENT=Production \
  --cpu 0.5 \
  --memory 1.0Gi \
  --min-replicas 1 \
  --max-replicas 10
```

### 3.4 Despliegue de Múltiples Microservicios

**Script de despliegue automatizado (deploy-microservices.sh):**
```bash
#!/bin/bash

# Configuración
RESOURCE_GROUP="rg-microservices-demo"
ENVIRONMENT_NAME="env-microservices"
ACR_NAME="acrmicroservicesdemo"

# Array de servicios
declare -a services=("productservice" "userservice" "orderservice")

# Obtener credenciales de ACR
ACR_PASSWORD=$(az acr credential show --name $ACR_NAME --query "passwords[0].value" -o tsv)

# Desplegar cada servicio
for service in "${services[@]}"
do
    echo "Desplegando $service..."
    
    az containerapp create \
      --name $service \
      --resource-group $RESOURCE_GROUP \
      --environment $ENVIRONMENT_NAME \
      --image $ACR_NAME.azurecr.io/$service:latest \
      --target-port 8080 \
      --ingress external \
      --registry-server $ACR_NAME.azurecr.io \
      --registry-username $ACR_NAME \
      --registry-password $ACR_PASSWORD \
      --env-vars ASPNETCORE_ENVIRONMENT=Production \
      --cpu 0.5 \
      --memory 1.0Gi \
      --min-replicas 1 \
      --max-replicas 5
    
    echo "$service desplegado exitosamente"
done
```

### 3.5 Configuración de Comunicación Interna

**Service-to-Service Communication:**
```bash
# Habilitar comunicación interna
az containerapp ingress update \
  --name userservice \
  --resource-group $RESOURCE_GROUP \
  --type internal
```

**Configuración en appsettings.json:**
```json
{
  "Services": {
    "UserService": "https://userservice.internal.{environment-unique-id}.eastus.azurecontainerapps.io",
    "ProductService": "https://productservice.{environment-unique-id}.eastus.azurecontainerapps.io"
  }
}
```

### 3.6 Gestión de Secretos y Configuración

**Azure Key Vault Integration:**
```bash
# Crear Key Vault
KEYVAULT_NAME="kv-microservices-demo"

az keyvault create \
  --name $KEYVAULT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

# Agregar secretos
az keyvault secret set \
  --vault-name $KEYVAULT_NAME \
  --name "ConnectionStrings--DefaultConnection" \
  --value "Server=tcp:sqlserver.database.windows.net,1433;Database=ProductsDB;"
```

**Usar secretos en Container App:**
```bash
# Agregar identidad gestionada
az containerapp identity assign \
  --name productservice \
  --resource-group $RESOURCE_GROUP \
  --system-assigned

# Actualizar con secretos
az containerapp update \
  --name productservice \
  --resource-group $RESOURCE_GROUP \
  --set-env-vars \
    "ConnectionStrings__DefaultConnection=secretref:db-connection-string"
```

---

## 4. Autenticación en Container Apps

### 4.1 Configuración de Autenticación Integrada

Azure Container Apps proporciona autenticación integrada con múltiples proveedores de identidad sin necesidad de código adicional.

**Habilitar autenticación con Azure AD:**
```bash
# Variables de configuración
APP_NAME="productservice"
CLIENT_ID="your-azure-ad-client-id"
TENANT_ID="your-tenant-id"

# Habilitar autenticación
az containerapp auth update \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --set-env-vars \
    "MICROSOFT_PROVIDER_AUTHENTICATION_SECRET=your-client-secret" \
  --auth-config-file auth-config.json
```

**Archivo de configuración auth-config.json:**
```json
{
  "platform": {
    "enabled": true,
    "runtimeVersion": "~1"
  },
  "globalValidation": {
    "requireAuthentication": true,
    "unauthenticatedClientAction": "RedirectToLoginPage"
  },
  "identityProviders": {
    "azureActiveDirectory": {
      "enabled": true,
      "registration": {
        "openIdIssuer": "https://login.microsoftonline.com/{tenant-id}/v2.0",
        "clientId": "{client-id}",
        "clientSecretSettingName": "MICROSOFT_PROVIDER_AUTHENTICATION_SECRET"
      }
    }
  },
  "login": {
    "routes": {
      "logoutEndpoint": "/.auth/logout"
    }
  }
}
```

### 4.2 Implementación de JWT Authentication

**Configuración en ASP.NET Core:**
```csharp
// Program.cs - Configuración de JWT
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

// Configuración JWT
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["JwtSettings:SecretKey"])),
            ValidateIssuer = true,
            ValidIssuer = builder.Configuration["JwtSettings:Issuer"],
            ValidateAudience = true,
            ValidAudience = builder.Configuration["JwtSettings:Audience"],
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers().RequireAuthorization();
app.Run();
```

**Controlador con autorización:**
```csharp
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class ProductsController : ControllerBase
{
    [HttpGet]
    [Authorize(Roles = "Admin,User")]
    public async Task<ActionResult<IEnumerable<Product>>> GetAll()
    {
        // Obtener información del usuario autenticado
        var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        var userRoles = User.FindAll(ClaimTypes.Role).Select(r => r.Value);
        
        // Lógica del controlador
        return Ok(await _repository.GetAllAsync());
    }

    [HttpPost]
    [Authorize(Roles = "Admin")]
    public async Task<ActionResult<Product>> Create(Product product)
    {
        // Solo administradores pueden crear productos
        return Ok(await _repository.CreateAsync(product));
    }
}
```

### 4.3 Autenticación con Multiple Providers

**Configuración para GitHub, Google, y Azure AD:**
```json
{
  "platform": {
    "enabled": true
  },
  "globalValidation": {
    "requireAuthentication": true,
    "unauthenticatedClientAction": "RedirectToLoginPage"
  },
  "identityProviders": {
    "azureActiveDirectory": {
      "enabled": true,
      "registration": {
        "openIdIssuer": "https://login.microsoftonline.com/{tenant-id}/v2.0",
        "clientId": "{azure-ad-client-id}",
        "clientSecretSettingName": "AZURE_AD_CLIENT_SECRET"
      }
    },
    "gitHub": {
      "enabled": true,
      "registration": {
        "clientId": "{github-client-id}",
        "clientSecretSettingName": "GITHUB_CLIENT_SECRET"
      }
    },
    "google": {
      "enabled": true,
      "registration": {
        "clientId": "{google-client-id}",
        "clientSecretSettingName": "GOOGLE_CLIENT_SECRET"
      }
    }
  }
}
```

### 4.4 Autenticación Custom con Middleware

**Middleware personalizado para validación de API Keys:**
```csharp
public class ApiKeyMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConfiguration _configuration;

    public ApiKeyMiddleware(RequestDelegate next, IConfiguration configuration)
    {
        _next = next;
        _configuration = configuration;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Verificar si la ruta requiere API Key
        if (context.Request.Path.StartsWithSegments("/api"))
        {
            if (!context.Request.Headers.TryGetValue("X-API-Key", out var extractedApiKey))
            {
                context.Response.StatusCode = 401;
                await context.Response.WriteAsync("API Key missing");
                return;
            }

            var validApiKey = _configuration["ApiKey"];
            if (!validApiKey.Equals(extractedApiKey))
            {
                context.Response.StatusCode = 401;
                await context.Response.WriteAsync("Invalid API Key");
                return;
            }
        }

        await _next(context);
    }
}

// Registrar el middleware
app.UseMiddleware<ApiKeyMiddleware>();
```

---

## 5. Gestión de Revisiones en Container Apps

### 5.1 Conceptos de Revisiones

Las revisiones en Azure Container Apps son versiones inmutables de tu aplicación. Cada cambio en la configuración crea una nueva revisión, permitiendo:

- **Blue-Green Deployments**
- **Canary Releases** 
- **A/B Testing**
- **Rollback instantáneo**

### 5.2 Modos de Revisión

**Single Revision Mode:** Solo una revisión activa a la vez.
**Multiple Revision Mode:** Múltiples revisiones pueden coexistir.

```bash
# Cambiar a modo múltiples revisiones
az containerapp revision set-mode \
  --name productservice \
  --resource-group $RESOURCE_GROUP \
  --mode Multiple
```

### 5.3 Despliegue de Nueva Revisión

**Actualización con nueva imagen:**
```bash
# Actualizar con nueva versión
az containerapp update \
  --name productservice \
  --resource-group $RESOURCE_GROUP \
  --image $ACR_NAME.azurecr.io/productservice:v2.0.0 \
  --revision-suffix v2
```

**Usando YAML para configuración avanzada:**
```yaml
# revision-v2.yaml
properties:
  template:
    revisionSuffix: v2
    containers:
    - name: productservice
      image: acrmicroservicesdemo.azurecr.io/productservice:v2.0.0
      env:
      - name: ASPNETCORE_ENVIRONMENT
        value: "Production"
      - name: FEATURE_NEW_API
        value: "true"
    scale:
      minReplicas: 2
      maxReplicas: 15
```

### 5.4 Gestión de Tráfico entre Revisiones

**Traffic Splitting - Canary Deployment:**
```bash
# 90% tráfico a v1, 10% a v2
az containerapp ingress traffic set \
  --name productservice \
  --resource-group $RESOURCE_GROUP \
  --revision-weight productservice--v1=90 \
  --revision-weight productservice--v2=10
```

**Blue-Green Deployment:**
```bash
# Paso 1: Desplegar nueva revisión sin tráfico
az containerapp update \
  --name productservice \
  --resource-group $RESOURCE_GROUP \
  --image $ACR_NAME.azurecr.io/productservice:v3.0.0 \
  --revision-suffix v3

# Paso 2: Cambiar todo el tráfico a la nueva revisión
az containerapp ingress traffic set \
  --name productservice \
  --resource-group $RESOURCE_GROUP \
  --revision-weight productservice--v3=100
```

### 5.5 Rollback de Revisiones

**Rollback inmediato:**
```bash
# Volver a la revisión anterior
az containerapp ingress traffic set \
  --name productservice \
  --resource-group $RESOURCE_GROUP \
  --revision-weight productservice--v1=100 \
  --revision-weight productservice--v2=0
```

**Script de rollback automatizado:**
```bash
#!/bin/bash

CURRENT_REVISION=$(az containerapp revision list \
  --name productservice \
  --resource-group $RESOURCE_GROUP \
  --query "[?properties.active].name | [0]" -o tsv)

PREVIOUS_REVISION=$(az containerapp revision list \
  --name productservice \
  --resource-group $RESOURCE_GROUP \
  --query "[?!properties.active] | [0].name" -o tsv)

echo "Rolling back from $CURRENT_REVISION to $PREVIOUS_REVISION"

az containerapp ingress traffic set \
  --name productservice \
  --resource-group $RESOURCE_GROUP \
  --revision-weight $PREVIOUS_REVISION=100 \
  --revision-weight $CURRENT_REVISION=0
```

### 5.6 Monitoreo y Health Checks

**Health Probes configuración:**
```yaml
properties:
  template:
    containers:
    - name: productservice
      image: acrmicroservicesdemo.azurecr.io/productservice:v2.0.0
      probes:
      - type: Liveness
        httpGet:
          path: "/health"
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 30
      - type: Readiness
        httpGet:
          path: "/health/ready"
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
```

**Implementación en ASP.NET Core:**
```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck("self", () => Results.Healthy("API is running"))
    .AddCheck("database", () => 
    {
        // Verificar conexión a base de datos
        return Results.Healthy("Database connection OK");
    });

app.MapHealthChecks("/health");
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

---

## 6. Monitoreo y Observabilidad

### 6.1 Application Insights Integration

**Configuración en Container App:**
```bash
# Crear Application Insights
APPINSIGHTS_NAME="ai-microservices-demo"

az monitor app-insights component create \
  --app $APPINSIGHTS_NAME \
  --location $LOCATION \
  --resource-group $RESOURCE_GROUP

# Obtener Instrumentation Key
INSTRUMENTATION_KEY=$(az monitor app-insights component show \
  --app $APPINSIGHTS_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "instrumentationKey" -o tsv)

# Actualizar Container App con Application Insights
az containerapp update \
  --name productservice \
  --resource-group $RESOURCE_GROUP \
  --set-env-vars "APPLICATIONINSIGHTS_CONNECTION_STRING=InstrumentationKey=$INSTRUMENTATION_KEY"
```

### 6.2 Logging Structured

**Configuración en ASP.NET Core:**
```csharp
builder.Services.AddLogging(logging =>
{
    logging.ClearProviders();
    logging.AddConsole();
    logging.AddApplicationInsights();
});

// Logging estructurado
public class ProductsController : ControllerBase
{
    private readonly ILogger<ProductsController> _logger;

    public async Task<ActionResult<Product>> GetById(int id)
    {
        using var scope = _logger.BeginScope(new Dictionary<string, object>
        {
            ["ProductId"] = id,
            ["UserId"] = User.Identity?.Name
        });

        _logger.LogInformation("Fetching product with ID {ProductId}", id);
        
        var product = await _repository.GetByIdAsync(id);
        
        if (product == null)
        {
            _logger.LogWarning("Product with ID {ProductId} not found", id);
            return NotFound();
        }

        _logger.LogInformation("Successfully retrieved product {ProductName}", product.Name);
        return Ok(product);
    }
}
```

---

## 7. Mejores Prácticas y Optimización

### 7.1 Optimización de Recursos

**Configuración de CPU y Memoria:**
```bash
# Configuración optimizada basada en carga
az containerapp update \
  --name productservice \
  --resource-group $RESOURCE_GROUP \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 0 \
  --max-replicas 20
```

### 7.2 Configuración de Escalado

**Event-driven scaling con Azure Service Bus:**
```bash
# Escalado basado en mensajes en cola
az containerapp update \
  --name orderprocessor \
  --resource-group $RESOURCE_GROUP \
  --scale-rule-name azure-servicebus-queue-rule \
  --scale-rule-type azure-servicebus \
  --scale-rule-metadata queueName=orders \
                        namespace=sb-microservices \
                        messageCount=5 \
  --scale-rule-auth secretRef=servicebus-connection-string
```

### 7.3 Security Best Practices

**Network Security:**
```bash
# Crear VNET personalizada
az network vnet create \
  --resource-group $RESOURCE_GROUP \
  --name vnet-microservices \
  --address-prefix 10.0.0.0/16

# Integrar Container Apps Environment con VNET
az containerapp env create \
  --name $ENVIRONMENT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --infrastructure-subnet-resource-id "/subscriptions/{subscription-id}/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Network/virtualNetworks/vnet-microservices/subnets/containerapp-subnet"
```

---

## 8. Troubleshooting Común

### 8.1 Problemas de Conectividad

**Verificar logs de Container App:**
```bash
# Ver logs en tiempo real
az containerapp logs show \
  --name productservice \
  --resource-group $RESOURCE_GROUP \
  --follow

# Ver logs de una revisión específica
az containerapp logs show \
  --name productservice \
  --resource-group $RESOURCE_GROUP \
  --revision productservice--v2
```

### 8.2 Problemas de Autenticación

**Verificar configuración de auth:**
```bash
az containerapp auth show \
  --name productservice \
  --resource-group $RESOURCE_GROUP
```

### 8.3 Problemas de Escalado

**Verificar métricas de escalado:**
```bash
az containerapp revision show \
  --name productservice \
  --resource-group $RESOURCE_GROUP \
  --revision productservice--v2 \
  --query "properties.template.scale"
```

---

## 9. Recursos Adicionales

- **Documentación oficial:** https://docs.microsoft.com/azure/container-apps
- **Azure Container Apps samples:** https://github.com/microsoft/container-apps
- **Bicep templates:** Para infraestructura como código
- **GitHub Actions:** Para CI/CD automatizado
- **Azure Monitor:** Para observabilidad avanzada

---

## 10. Próximos Pasos

- **Service Mesh con Dapr:** Implementar patrones avanzados de microservicios
- **Event-driven Architecture:** Usar Azure Event Grid y Service Bus
- **Advanced Networking:** VNET peering y private endpoints
- **Multi-region Deployment:** Alta disponibilidad geográfica
- **Cost Optimization:** Análisis y optimización de costos

---

*Este módulo proporciona una guía completa para desplegar y gestionar microservicios en Azure Container Apps, desde conceptos básicos hasta implementaciones avanzadas de producción.*