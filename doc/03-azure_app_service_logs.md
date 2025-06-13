# Gestión de Logs y Diagnóstico en Azure App Service

## Introducción

Azure App Service proporciona capacidades integradas de logging y diagnóstico que permiten monitorear, depurar y mantener aplicaciones web. Este documento cubre las diferentes opciones de logging, herramientas de diagnóstico y mejores prácticas.

## Tipos de Logs en App Service

### 1. Application Logs
- **Descripción**: Logs generados por el código de tu aplicación
- **Formatos soportados**: .NET, Java, PHP, Node.js, Python
- **Ubicación**: `/LogFiles/Application/`

### 2. Web Server Logs
- **Descripción**: Logs del servidor web (IIS)
- **Formato**: W3C Extended Log Format
- **Información incluida**: Requests HTTP, respuestas, tiempos de procesamiento

### 3. Detailed Error Messages
- **Descripción**: Páginas de error detalladas para códigos de estado HTTP 400+
- **Formato**: Archivos HTML con información de debugging

### 4. Failed Request Tracing
- **Descripción**: Información detallada sobre requests fallidos
- **Utilidad**: Debugging de problemas de rendimiento y errores

### 5. Deployment Logs
- **Descripción**: Logs del proceso de deployment
- **Ubicación**: `/LogFiles/Git/` y `/LogFiles/kudu/`

## Configuración de Logging

### Habilitar Application Logging

#### Mediante Azure Portal
1. Navegar a tu App Service
2. Ir a **Monitoring** > **App Service logs**
3. Configurar **Application logging**:
   - **File System**: Temporal (máximo 12 horas)
   - **Blob Storage**: Persistente, ideal para producción

#### Mediante Azure CLI
```bash
# Habilitar application logging al filesystem
az webapp log config --name myapp --resource-group mygroup --application-logging filesystem

# Habilitar application logging a blob storage
az webapp log config --name myapp --resource-group mygroup \
  --application-logging azureblobstorage \
  --level information
```

#### Mediante PowerShell
```powershell
# Configurar application logging
Set-AzWebApp -ResourceGroupName "mygroup" -Name "myapp" `
  -AppSettings @{"WEBSITE_HTTPLOGGING_RETENTION_DAYS"="7"}
```

### Configurar Web Server Logs

```bash
# Habilitar web server logging
az webapp log config --name myapp --resource-group mygroup \
  --web-server-logging filesystem
```

### Niveles de Log

| Nivel | Descripción | Uso recomendado |
|-------|-------------|-----------------|
| Error | Solo errores críticos | Producción |
| Warning | Errores y advertencias | Producción |
| Information | Info general + Warning + Error | Desarrollo/Testing |
| Verbose | Logs detallados | Debugging |

## Herramientas de Diagnóstico

### 1. Kudu Console
- **URL**: `https://<app-name>.scm.azurewebsites.net`
- **Funcionalidades**:
  - Explorador de archivos
  - Console de comandos
  - Información del proceso
  - Variables de entorno

### 2. Application Insights
```javascript
// Integración con Application Insights
const appInsights = require('applicationinsights');
appInsights.setup('YOUR_INSTRUMENTATION_KEY');
appInsights.start();

// Logging personalizado
appInsights.defaultClient.trackEvent({
  name: 'CustomEvent',
  properties: { customProperty: 'value' }
});
```

### 3. Azure Monitor
- Métricas de rendimiento
- Alertas automáticas
- Dashboards personalizados
- Integración con Application Insights

### 4. Log Analytics Workspace
```kusto
// Consulta de ejemplo en KQL
AppServiceHTTPLogs
| where TimeGenerated > ago(1h)
| where ScStatus >= 400
| summarize count() by ScStatus, bin(TimeGenerated, 5m)
| render timechart
```

## Acceso a Logs

### 1. Azure Portal
- **Navegación**: App Service > Monitoring > Log stream
- **Ventajas**: Vista en tiempo real
- **Limitaciones**: Solo logs recientes

### 2. Azure CLI
```bash
# Stream de logs en tiempo real
az webapp log tail --name myapp --resource-group mygroup

# Descargar logs
az webapp log download --name myapp --resource-group mygroup
```

### 3. Visual Studio Code
- Extensión Azure App Service
- Streaming directo desde VS Code
- Integración con debugging

### 4. FTP/FTPS
```bash
# Configuración FTP
az webapp deployment list-publishing-profiles --name myapp --resource-group mygroup
```

## Mejores Prácticas

### 1. Configuración por Ambiente
```json
{
  "development": {
    "logging": {
      "level": "verbose",
      "destination": "filesystem"
    }
  },
  "production": {
    "logging": {
      "level": "warning",
      "destination": "blob",
      "retention": "30 days"
    }
  }
}
```

### 2. Structured Logging
```csharp
// Ejemplo en .NET
logger.LogInformation("Processing order {OrderId} for customer {CustomerId}", 
                     orderId, customerId);
```

```javascript
// Ejemplo en Node.js
const winston = require('winston');
const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'app.log' })
  ]
});
```

### 3. Gestión de Retención
- **Filesystem**: Máximo 35 MB o 7 días
- **Blob Storage**: Configurar políticas de lifecycle
- **Application Insights**: 90 días por defecto

### 4. Filtrado y Sampling
```javascript
// Application Insights sampling
appInsights.setup('INSTRUMENTATION_KEY')
  .setSamplingPercentage(50) // 50% sampling
  .start();
```

## Troubleshooting Común

### 1. Logs No Aparecen
- Verificar que logging está habilitado
- Revisar nivel de log configurado
- Confirmar que la aplicación está escribiendo logs

### 2. Performance Issues
- Reducir nivel de verbosidad en producción
- Implementar sampling en Application Insights
- Usar structured logging para mejor rendimiento

### 3. Storage Quota Exceeded
```bash
# Limpiar logs antiguos
az webapp log config --name myapp --resource-group mygroup \
  --web-server-logging off
```

### 4. Application Insights No Recibe Datos
- Verificar instrumentation key
- Confirmar conectividad de red
- Revisar configuración de sampling

## Monitoreo y Alertas

### 1. Configurar Alertas Básicas
```bash
# Crear alerta para errores 5xx
az monitor metrics alert create \
  --name "High 5xx errors" \
  --resource-group mygroup \
  --scopes /subscriptions/{subscription}/resourceGroups/mygroup/providers/Microsoft.Web/sites/myapp \
  --condition "count Http5xx >= 10" \
  --window-size 5m \
  --evaluation-frequency 1m
```

### 2. Dashboard Personalizado
```json
{
  "dashboard": {
    "tiles": [
      {
        "name": "HTTP Response Codes",
        "query": "requests | summarize count() by resultCode"
      },
      {
        "name": "Average Response Time",
        "query": "requests | summarize avg(duration)"
      }
    ]
  }
}
```

## Automatización

### 1. Script de Backup de Logs
```bash
#!/bin/bash
DATE=$(date +%Y%m%d)
az webapp log download --name $APP_NAME --resource-group $RESOURCE_GROUP
mv logs.zip "logs_backup_$DATE.zip"
```

### 2. Análisis Automatizado
```python
import requests
import json
from datetime import datetime, timedelta

def analyze_logs():
    # Conectar a Application Insights API
    # Realizar consultas automáticas
    # Generar reportes
    pass
```

## Seguridad y Compliance

### 1. Datos Sensibles
- Evitar loggear información personal
- Usar Application Settings para secrets
- Implementar log scrubbing

### 2. Acceso Controlado
```bash
# Configurar RBAC para logs
az role assignment create \
  --assignee user@domain.com \
  --role "Log Analytics Reader" \
  --scope /subscriptions/{subscription}/resourceGroups/mygroup
```

## Costos y Optimización

### 1. Estimación de Costos
- Application Insights: ~$2.30/GB ingerido
- Log Analytics: ~$2.76/GB ingerido
- Blob Storage: ~$0.018/GB/mes

### 2. Optimización
- Configurar retención adecuada
- Implementar sampling inteligente
- Usar queries eficientes en KQL

## Recursos Adicionales

### Documentación Oficial
- [Azure App Service Logging](https://docs.microsoft.com/azure/app-service/troubleshoot-diagnostic-logs)
- [Application Insights](https://docs.microsoft.com/azure/azure-monitor/app/app-insights-overview)
- [Kudu Wiki](https://github.com/projectkudu/kudu/wiki)

### Herramientas Útiles
- Azure Storage Explorer
- Application Insights Analytics
- Visual Studio Code Azure Extensions
- Azure CLI

### Plantillas ARM
```json
{
  "type": "Microsoft.Web/sites/config",
  "apiVersion": "2021-02-01",
  "name": "[concat(parameters('siteName'), '/logs')]",
  "properties": {
    "applicationLogs": {
      "fileSystem": {
        "level": "Information"
      },
      "azureBlobStorage": {
        "level": "Information",
        "sasUrl": "[parameters('logsStorageSasUrl')]"
      }
    },
    "httpLogs": {
      "fileSystem": {
        "retentionInMb": 35,
        "enabled": true
      }
    }
  }
}
```

---

*Documento actualizado: Junio 2025*
*Versión: 1.0*