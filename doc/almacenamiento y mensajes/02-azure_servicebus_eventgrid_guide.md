# Guía de Azure: Service Bus y Event Grid

## Comunicación Asíncrona con Azure Service Bus (Service Bus Queues)

### ¿Qué es Azure Service Bus?

Azure Service Bus es un servicio de mensajería empresarial completamente gestionado que proporciona comunicación confiable entre aplicaciones y servicios distribuidos. Actúa como un intermediario de mensajes (message broker) que permite el desacoplamiento de aplicaciones y servicios.

### Conceptos Fundamentales

#### Patrones de Mensajería
- **Desacoplamiento**: Las aplicaciones no necesitan estar activas simultáneamente
- **Confiabilidad**: Garantiza la entrega de mensajes incluso si el receptor está temporalmente no disponible
- **Escalabilidad**: Permite manejar picos de carga mediante buffering de mensajes

#### Entidades de Service Bus
1. **Queues (Colas)**: Comunicación punto a punto (one-to-one)
2. **Topics**: Comunicación publish-subscribe (one-to-many)
3. **Subscriptions**: Filtros para topics específicos

### Service Bus Queues - Características Principales

#### Comunicación Punto a Punto
- Un mensaje enviado a una cola es recibido por un solo consumidor
- Los mensajes se procesan en orden FIFO (First In, First Out)
- Ideal para distribución de trabajo entre múltiples workers

#### Garantías de Entrega
- **At-Least-Once**: Garantiza que cada mensaje se entrega al menos una vez
- **Duplicate Detection**: Detecta y elimina mensajes duplicados
- **Dead Letter Queue**: Almacena mensajes que no pueden ser procesados

#### Características Avanzadas
- **Message Sessions**: Garantiza procesamiento ordenado de mensajes relacionados
- **Message Deferral**: Permite posponer el procesamiento de mensajes
- **Scheduled Messages**: Programar mensajes para entrega futura
- **Auto-forwarding**: Reenvío automático a otra cola o topic

### Configuración y Gestión de Service Bus

#### Creación de un Namespace de Service Bus
```bash
# Azure CLI
az servicebus namespace create \
    --resource-group myResourceGroup \
    --name myServiceBusNamespace \
    --location eastus \
    --sku Standard
```

#### Creación de una Queue
```bash
# Crear una cola
az servicebus queue create \
    --resource-group myResourceGroup \
    --namespace-name myServiceBusNamespace \
    --name myqueue \
    --max-size 1024 \
    --default-message-time-to-live P14D \
    --lock-duration PT30S \
    --max-delivery-count 10
```

#### Parámetros Importantes de las Colas

##### Configuración de Capacidad
- **Max Size**: Tamaño máximo de la cola (1GB - 80GB)
- **Message TTL**: Tiempo de vida del mensaje
- **Max Delivery Count**: Intentos máximos de entrega antes de enviar a DLQ

##### Configuración de Comportamiento
- **Lock Duration**: Tiempo que un mensaje permanece bloqueado
- **Duplicate Detection**: Ventana de tiempo para detectar duplicados
- **Dead Lettering**: Configuración para mensajes no procesables

### Implementación con .NET

#### Envío de Mensajes
```csharp
using Azure.Messaging.ServiceBus;

// Crear cliente de Service Bus
var client = new ServiceBusClient(connectionString);
var sender = client.CreateSender("myqueue");

// Crear y enviar mensaje
var message = new ServiceBusMessage("Hello, Service Bus!");
message.ApplicationProperties["Priority"] = "High";
message.MessageId = Guid.NewGuid().ToString();

await sender.SendMessageAsync(message);
```

#### Recepción de Mensajes
```csharp
// Crear procesador de mensajes
var processor = client.CreateProcessor("myqueue", new ServiceBusProcessorOptions());

// Configurar handlers
processor.ProcessMessageAsync += ProcessMessageHandler;
processor.ProcessErrorAsync += ProcessErrorHandler;

// Iniciar procesamiento
await processor.StartProcessingAsync();

async Task ProcessMessageHandler(ProcessMessageEventArgs args)
{
    string body = args.Message.Body.ToString();
    Console.WriteLine($"Received: {body}");
    
    // Completar el mensaje (removerlo de la cola)
    await args.CompleteMessageAsync(args.Message);
}

async Task ProcessErrorHandler(ProcessErrorEventArgs args)
{
    Console.WriteLine($"Error: {args.Exception.ToString()}");
}
```

### Patrones de Procesamiento

#### Peek-Lock vs Receive-and-Delete

##### Peek-Lock (Recomendado)
- Bloquea el mensaje temporalmente
- Permite procesamiento seguro con confirmación
- Automáticamente reintenta si el procesamiento falla

```csharp
// El mensaje se bloquea automáticamente
await processor.ProcessMessageAsync += async (args) =>
{
    try
    {
        // Procesar mensaje
        await ProcessBusinessLogic(args.Message);
        
        // Confirmar procesamiento exitoso
        await args.CompleteMessageAsync(args.Message);
    }
    catch (Exception ex)
    {
        // Rechazar mensaje (volverá a estar disponible)
        await args.AbandonMessageAsync(args.Message);
    }
};
```

##### Receive-and-Delete
- Remueve inmediatamente el mensaje de la cola
- Mayor rendimiento, menor confiabilidad
- Riesgo de pérdida de mensajes en caso de falla

#### Manejo de Dead Letter Queue
```csharp
// Procesar mensajes de Dead Letter Queue
var dlqProcessor = client.CreateProcessor("myqueue", "$deadletterqueue");

dlqProcessor.ProcessMessageAsync += async (args) =>
{
    Console.WriteLine($"DLQ Message: {args.Message.Body}");
    Console.WriteLine($"Reason: {args.Message.ApplicationProperties["DeadLetterReason"]}");
    
    // Decidir si reprocessar o log del error
    await args.CompleteMessageAsync(args.Message);
};
```

### Monitoreo y Diagnóstico

#### Métricas Importantes
- **Active Messages**: Mensajes disponibles para procesamiento
- **Dead Letter Messages**: Mensajes en DLQ
- **Incoming/Outgoing Messages**: Tasa de mensajes
- **Server Errors**: Errores del lado del servidor

#### Configuración de Alertas
```bash
# Crear alerta para mensajes en DLQ
az monitor metrics alert create \
    --name "ServiceBus-DLQ-Alert" \
    --resource-group myResourceGroup \
    --resource myServiceBusNamespace \
    --metric "DeadletteredMessages" \
    --operator GreaterThan \
    --threshold 10
```

### Mejores Prácticas para Service Bus Queues

#### Diseño de Mensajes
- Mantener mensajes pequeños (< 256KB recomendado)
- Incluir información de correlación
- Usar propiedades personalizadas para metadatos
- Implementar idempotencia en el procesamiento

#### Gestión de Errores
- Implementar retry logic exponencial
- Monitorear Dead Letter Queues regularmente
- Configurar alertas para anomalías
- Mantener logs detallados de errores

#### Rendimiento
- Usar batch operations para mayor throughput
- Configurar prefetch count apropiadamente
- Implementar processing concurrente cuando sea apropiado
- Monitorear métricas de latencia

---

## Introducción a Azure Event Grid

### ¿Qué es Azure Event Grid?

Azure Event Grid es un servicio de enrutamiento de eventos completamente gestionado que permite una programación reactiva uniforme utilizando un modelo publish-subscribe. Facilita la creación de aplicaciones basadas en eventos con enrutamiento de eventos confiable, masivo y de baja latencia.

### Conceptos Fundamentales

#### Paradigma Event-Driven
- **Reactive Programming**: Las aplicaciones reaccionan a eventos en lugar de polling constante
- **Loose Coupling**: Los productores y consumidores de eventos están desacoplados
- **Scalability**: Maneja millones de eventos con baja latencia

#### Componentes Principales
1. **Event Sources**: Servicios que publican eventos (Storage, IoT Hub, etc.)
2. **Event Grid Topic**: Endpoint donde se envían los eventos
3. **Event Subscriptions**: Configuración de entrega de eventos
4. **Event Handlers**: Servicios que procesan los eventos

### Arquitectura de Event Grid

#### System Topics vs Custom Topics

##### System Topics
- Creados automáticamente por servicios de Azure
- Publican eventos del ciclo de vida de recursos
- Ejemplos: Storage Account events, Resource Group events

##### Custom Topics
- Creados por el usuario para eventos personalizados
- Útiles para arquitecturas de microservicios
- Permiten definir esquemas de eventos personalizados

#### Event Schema
```json
{
  "id": "unique-event-id",
  "eventType": "Microsoft.Storage.BlobCreated",
  "subject": "/blobServices/default/containers/mycontainer/blobs/myblob.txt",
  "eventTime": "2023-06-15T10:30:00Z",
  "data": {
    "api": "PutBlob",
    "clientRequestId": "client-request-id",
    "requestId": "request-id",
    "eTag": "0x8D4BCC2E4835CD0",
    "contentType": "text/plain",
    "contentLength": 524288,
    "blobType": "BlockBlob",
    "url": "https://example.blob.core.windows.net/mycontainer/myblob.txt"
  },
  "dataVersion": "1.0",
  "metadataVersion": "1",
  "topic": "/subscriptions/subscription-id/resourceGroups/rg/providers/Microsoft.Storage/storageAccounts/myaccount"
}
```

### Configuración y Gestión de Event Grid

#### Creación de un Custom Topic
```bash
# Crear un custom topic
az eventgrid topic create \
    --resource-group myResourceGroup \
    --name myEventGridTopic \
    --location eastus
```

#### Creación de Event Subscription
```bash
# Suscripción a webhook
az eventgrid event-subscription create \
    --source-resource-id "/subscriptions/subscription-id/resourceGroups/myResourceGroup/providers/Microsoft.EventGrid/topics/myEventGridTopic" \
    --name myEventGridSubscription \
    --endpoint https://mywebhook.azurewebsites.net/api/updates

# Suscripción a Service Bus Queue
az eventgrid event-subscription create \
    --source-resource-id "/subscriptions/subscription-id/resourceGroups/myResourceGroup/providers/Microsoft.EventGrid/topics/myEventGridTopic" \
    --name myServiceBusSubscription \
    --endpoint-type servicebusqueue \
    --endpoint "/subscriptions/subscription-id/resourceGroups/myResourceGroup/providers/Microsoft.ServiceBus/namespaces/myNamespace/queues/myQueue"
```

### Event Handlers (Destinos de Eventos)

#### Webhooks
- HTTP endpoints que reciben eventos vía POST
- Deben implementar validation handshake
- Soporte para retry automático y dead lettering

```csharp
// Validación de webhook
[HttpPost]
public IActionResult HandleEvent([FromBody] dynamic eventData)
{
    // Validación de suscripción
    if (eventData.eventType == "Microsoft.EventGrid.SubscriptionValidationEvent")
    {
        return Ok(new { validationResponse = eventData.data.validationCode });
    }
    
    // Procesar eventos
    foreach (var eventItem in eventData)
    {
        Console.WriteLine($"Event Type: {eventItem.eventType}");
        Console.WriteLine($"Subject: {eventItem.subject}");
    }
    
    return Ok();
}
```

#### Azure Functions
- Trigger nativo para Event Grid
- Escalado automático basado en eventos
- Ideal para procesamiento serverless

```csharp
[FunctionName("EventGridTrigger")]
public static void Run([EventGridTrigger] EventGridEvent eventGridEvent, ILogger log)
{
    log.LogInformation($"Event Type: {eventGridEvent.EventType}");
    log.LogInformation($"Subject: {eventGridEvent.Subject}");
    log.LogInformation($"Data: {eventGridEvent.Data}");
}
```

#### Otros Handlers
- **Service Bus Queues/Topics**: Para procesamiento confiable
- **Storage Queues**: Para procesamiento simple y económico
- **Event Hubs**: Para análisis de streaming
- **Logic Apps**: Para workflows sin código
- **Relay Hybrid Connections**: Para endpoints on-premises

### Filtrado de Eventos

#### Subject Filtering
```bash
# Filtrar por subject pattern
az eventgrid event-subscription create \
    --source-resource-id $topicid \
    --name myFilteredSubscription \
    --endpoint $endpoint \
    --subject-begins-with "/blobServices/default/containers/images/" \
    --subject-ends-with ".jpg"
```

#### Advanced Filtering
```json
{
  "advancedFilters": [
    {
      "operatorType": "StringContains",
      "key": "data.contentType",
      "values": ["image/"]
    },
    {
      "operatorType": "NumberGreaterThan",
      "key": "data.contentLength",
      "value": 1000000
    }
  ]
}
```

### Publicación de Eventos Personalizados

#### Usando Azure CLI
```bash
# Publicar evento personalizado
az eventgrid event send \
    --topic-name myEventGridTopic \
    --resource-group myResourceGroup \
    --events '[
        {
            "id": "event-1",
            "eventType": "MyApp.OrderCreated",
            "subject": "orders/order-123",
            "eventTime": "2023-06-15T10:30:00Z",
            "data": {
                "orderId": "order-123",
                "customerId": "customer-456",
                "amount": 99.99
            },
            "dataVersion": "1.0"
        }
    ]'
```

#### Usando .NET SDK
```csharp
using Azure.Messaging.EventGrid;

// Crear cliente
var client = new EventGridPublisherClient(new Uri(topicEndpoint), new AzureKeyCredential(topicKey));

// Crear eventos
var events = new List<EventGridEvent>
{
    new EventGridEvent(
        subject: "orders/order-123",
        eventType: "MyApp.OrderCreated",
        dataVersion: "1.0",
        data: new
        {
            OrderId = "order-123",
            CustomerId = "customer-456",
            Amount = 99.99
        })
};

// Publicar eventos
await client.SendEventsAsync(events);
```

### Configuración de Retry y Dead Letter

#### Retry Policy
```bash
# Configurar política de retry
az eventgrid event-subscription create \
    --source-resource-id $topicid \
    --name mySubscription \
    --endpoint $endpoint \
    --max-delivery-attempts 30 \
    --event-ttl 1440  # 24 horas en minutos
```

#### Dead Letter Destination
```bash
# Configurar dead letter storage
az eventgrid event-subscription create \
    --source-resource-id $topicid \
    --name mySubscription \
    --endpoint $endpoint \
    --deadletter-endpoint "/subscriptions/subscription-id/resourceGroups/myResourceGroup/providers/Microsoft.Storage/storageAccounts/mydeadlettersa/blobServices/default/containers/deadletter"
```

### Seguridad en Event Grid

#### Autenticación de Webhooks
- **Access Keys**: Incluidas en query string o headers
- **Azure AD Authentication**: Para endpoints que soportan AAD
- **Webhook Validation**: Handshake obligatorio para nuevas suscripciones

```csharp
// Validar webhook signature
public bool ValidateSignature(string payload, string signature, string key)
{
    var hashBytes = new HMACSHA256(Encoding.UTF8.GetBytes(key))
        .ComputeHash(Encoding.UTF8.GetBytes(payload));
    var hash = Convert.ToBase64String(hashBytes);
    return signature.Equals($"sha256={hash}");
}
```

#### Control de Acceso
- **RBAC**: Control granular sobre topics y subscriptions
- **Private Endpoints**: Comunicación privada vía VNet
- **Managed Identity**: Autenticación sin credenciales hardcodeadas

### Monitoreo y Diagnóstico

#### Métricas Clave
- **Published Events**: Eventos publicados exitosamente
- **Matched Events**: Eventos que coinciden con filtros
- **Delivery Attempts**: Intentos de entrega totales
- **Failed Deliveries**: Entregas fallidas
- **Dead Lettered Events**: Eventos enviados a dead letter

#### Diagnostic Logs
```bash
# Habilitar diagnostic logs
az monitor diagnostic-settings create \
    --name myDiagnostics \
    --resource $topicResourceId \
    --logs '[
        {
            "category": "DeliveryFailures",
            "enabled": true
        },
        {
            "category": "PublishFailures", 
            "enabled": true
        }
    ]' \
    --workspace $logAnalyticsWorkspaceId
```

### Casos de Uso Comunes

#### Event-Driven Microservices
- Comunicación entre servicios mediante eventos
- Desacoplamiento de componentes de aplicación
- Orquestación de workflows complejos

#### Automatización de Operaciones
- Respuesta automática a cambios en recursos
- Procesamiento de archivos subidos a Storage
- Notificaciones de cambios en bases de datos

#### Integración de Sistemas
- Sincronización entre sistemas on-premises y cloud
- Replicación de datos entre diferentes almacenes
- Auditoría y logging centralizados

#### IoT y Streaming
- Procesamiento de telemetría de dispositivos
- Alertas basadas en umbrales de sensores
- Análisis en tiempo real de datos de streaming

---

## Comparación: Service Bus vs Event Grid

### Cuándo Usar Service Bus Queues

#### Patrones de Comunicación
- **Comando/Solicitud**: Procesamiento de trabajo específico
- **Request-Response**: Comunicación bidireccional
- **Load Leveling**: Suavizar picos de carga

#### Características Necesarias
- Garantías de entrega robustas
- Procesamiento ordenado de mensajes
- Transacciones y sessions
- Alta disponibilidad y durabilidad

#### Ejemplos de Uso
- Procesamiento de órdenes de compra
- Envío de emails transaccionales  
- Procesamiento de pagos
- Integración B2B

### Cuándo Usar Event Grid

#### Patrones de Comunicación
- **Event Notification**: Notificar cambios de estado
- **Event Sourcing**: Reconstituir estado desde eventos
- **CQRS**: Separar comandos de consultas

#### Características Necesarias
- Baja latencia (< 1 segundo)
- Alto throughput (millones de eventos)
- Filtrado avanzado de eventos
- Múltiples suscriptores por evento

#### Ejemplos de Uso
- Notificaciones de cambios en recursos
- Pipelines de procesamiento de medios
- Automatización de DevOps
- Análisis en tiempo real

### Arquitectura Híbrida

#### Combinando Ambos Servicios
```
Event Grid → Service Bus Queue → Function App
     ↓              ↓               ↓
  Routing      Buffering       Processing
```

#### Beneficios de la Combinación
- Event Grid para routing inteligente
- Service Bus para garantías de procesamiento
- Mejor resilencia y escalabilidad
- Separación clara de responsabilidades

---

## Mejores Prácticas Generales

### Diseño de Arquitectura
- Definir claramente los boundaries de contexto
- Implementar idempotencia en todos los handlers
- Usar correlation IDs para trazabilidad
- Considerar eventual consistency

### Manejo de Errores
- Implementar retry policies apropiadas
- Configurar dead letter queues/destinations
- Monitorear métricas de error continuamente
- Mantener logs detallados para debugging

### Seguridad
- Usar Managed Identities cuando sea posible
- Implementar least privilege access
- Cifrar datos sensibles en eventos/mensajes
- Validar todos los inputs de eventos

### Rendimiento
- Batch operations cuando sea apropiado
- Configurar partitioning correctamente
- Monitorear y optimizar throughput
- Usar connection pooling en aplicaciones

---

## Recursos Adicionales

### Documentación Oficial
- [Azure Service Bus Documentation](https://docs.microsoft.com/azure/service-bus-messaging/)
- [Azure Event Grid Documentation](https://docs.microsoft.com/azure/event-grid/)

### SDKs y Herramientas
- Azure SDK for .NET, Java, Python, JavaScript
- Service Bus Explorer
- Event Grid Viewer
- Azure CLI y PowerShell

### Patrones y Arquitecturas
- [Cloud Design Patterns](https://docs.microsoft.com/azure/architecture/patterns/)
- [Event-driven architecture](https://docs.microsoft.com/azure/architecture/guide/architecture-styles/event-driven)
- [Microservices architecture](https://docs.microsoft.com/azure/architecture/guide/architecture-styles/microservices)

### Capacitación
- AZ-204: Developing Solutions for Microsoft Azure
- AZ-304: Microsoft Azure Architect Design
- Microsoft Learn modules sobre messaging patterns