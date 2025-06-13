# Azure Application Gateway con Container Apps

## Arquitectura de la Solución

Esta guía explica cómo configurar Azure Application Gateway para exponer solo una Container App públicamente, mientras esta se comunica internamente con otra Container App privada.

```
Internet → Application Gateway → Frontend Container App → Backend Container App (Interno)
```

## Componentes de la Arquitectura

### 1. Application Gateway (Puerta de Enlace)
- **Servicio nativo de Azure** para balanceo de carga de aplicaciones (Layer 7)
- Maneja el tráfico HTTPS/HTTP entrante desde internet
- Enruta únicamente al frontend, manteniendo el backend oculto
- Proporciona terminación SSL, WAF y otras funcionalidades avanzadas

### 2. Frontend Container App (Público)
- **Ingress externo habilitado** (`external: true`)
- Recibe tráfico del Application Gateway
- Se comunica internamente con el backend
- Actúa como proxy/gateway de la aplicación

### 3. Backend Container App (Privado)
- **Ingress interno únicamente** (`external: false`)
- Solo accesible desde dentro del entorno de Container Apps
- Maneja la lógica de negocio y datos sensibles
- URL interna: `https://backend-app.internal.{environment-domain}`

## Configuración Detallada

### Red Virtual y Subredes

```yaml
# Red virtual con dos subredes separadas
vnet: 10.0.0.0/16
├── subnet-appgateway: 10.0.1.0/24    # Para Application Gateway
└── subnet-containerapp: 10.0.2.0/24  # Para Container Apps Environment
```

### Container Apps Environment

```yaml
# Entorno compartido para ambas Container Apps
managedEnvironments:
  properties:
    vnetConfiguration:
      infrastructureSubnetId: subnet-containerapp
    zoneRedundant: false
```

### Backend Container App (Interno)

```yaml
# Configuración clave para mantenerlo privado
configuration:
  ingress:
    external: false          # ← CRÍTICO: Solo acceso interno
    targetPort: 8080
    allowInsecure: false
    traffic:
      - latestRevision: true
        weight: 100
```

**Características del Backend:**
- No tiene IP pública
- Solo accesible mediante FQDN interno
- Comunicación cifrada dentro del entorno
- Escalado automático configurado

### Frontend Container App (Público)

```yaml
# Configuración para exposición externa
configuration:
  ingress:
    external: true           # ← Permite acceso desde Application Gateway
    targetPort: 3000
    allowInsecure: false
    traffic:
      - latestRevision: true
        weight: 100

# Variable de entorno para conectar con backend
template:
  containers:
    env:
      - name: BACKEND_URL
        value: https://backend-app.internal.{environment-domain}
```

### Application Gateway

```yaml
# Configuración del pool de backend
backendAddressPools:
  - name: frontendAppPool
    properties:
      backendAddresses:
        - fqdn: {frontend-app-fqdn}    # Solo apunta al frontend

# Health probe y configuración HTTPS
backendHttpSettingsCollection:
  - name: appGatewayBackendHttpSettings
    properties:
      port: 443
      protocol: Https
      pickHostNameFromBackendAddress: true
      requestTimeout: 20
```

## Flujo de Comunicación

### 1. Tráfico Entrante
```
Cliente → Application Gateway (IP Pública) → Frontend Container App
```

### 2. Comunicación Interna
```
Frontend Container App → Backend Container App (URL interna)
```

### 3. Respuesta
```
Backend → Frontend → Application Gateway → Cliente
```

## Ejemplo de Código

### Frontend (Node.js/Express)

```javascript
const express = require('express');
const axios = require('axios');
const app = express();

// URL del backend interno (desde variable de entorno)
const BACKEND_URL = process.env.BACKEND_URL;

// Endpoint que consume el backend interno
app.get('/api/data', async (req, res) => {
  try {
    // Llamada al servicio interno
    const response = await axios.get(`${BACKEND_URL}/api/internal-data`);
    
    res.json({
      message: 'Datos obtenidos del backend interno',
      data: response.data
    });
  } catch (error) {
    res.status(500).json({
      error: 'Error al obtener datos del backend'
    });
  }
});

app.listen(3000);
```

### Backend (Node.js/Express)

```javascript
const express = require('express');
const app = express();

// Endpoint interno - solo accesible desde el frontend
app.get('/api/internal-data', (req, res) => {
  res.json({
    message: 'Datos sensibles del backend',
    data: {
      users: [/* datos internos */],
      serverInfo: {
        hostname: require('os').hostname(),
        environment: 'internal'
      }
    }
  });
});

app.listen(8080);
```

## Ventajas de esta Arquitectura

### Seguridad
- **Backend protegido**: Sin exposición directa a internet
- **Tráfico cifrado**: Comunicación HTTPS en toda la cadena
- **Firewall integrado**: Application Gateway con WAF opcional

### Escalabilidad
- **Escalado independiente**: Frontend y backend pueden escalar por separado
- **Load balancing**: Application Gateway distribuye carga automáticamente
- **Auto-scaling**: Container Apps escala según demanda

### Mantenibilidad
- **Separación de responsabilidades**: Frontend para UI, backend para lógica
- **Despliegues independientes**: Cada servicio se puede actualizar por separado
- **Monitoreo granular**: Métricas separadas por servicio

## URLs Resultantes

### Acceso Público
```
https://{application-gateway-ip}/api/data
```

### URLs Internas (No accesibles desde internet)
```
https://frontend-app.{environment-domain}       # Container App frontend
https://backend-app.internal.{environment-domain}  # Container App backend
```

## Consideraciones de Seguridad

### Network Security Groups (NSG)
- Configurar reglas para permitir solo tráfico necesario
- Bloquear acceso directo al backend desde internet

### Managed Identity
- Usar identidades administradas para autenticación entre servicios
- Evitar credenciales hardcodeadas

### Container Registry
- Usar Azure Container Registry con acceso restringido
- Escaneo de vulnerabilidades en imágenes

## Monitoreo y Logs

### Application Insights
```yaml
# Configurar para ambos container apps
properties:
  configuration:
    dapr:
      appId: frontend-app
      enabled: true
```

### Métricas Clave
- Latencia de Application Gateway
- CPU/Memory de Container Apps
- Número de requests por segundo
- Errores 4xx/5xx

## Costos Estimados

### Application Gateway Standard_v2
- Costo fijo por hora + costo por datos procesados
- Approximately €0.25/hora + €0.008/GB

### Container Apps
- Pay-per-use: CPU/Memory consumidos
- Approximately €0.000012 per vCPU-second

### Networking
- Tráfico saliente y entre zonas
- Tráfico interno en la misma región: gratuito

---

*Esta arquitectura proporciona una solución robusta, segura y escalable para exponer aplicaciones web manteniendo los servicios backend protegidos.*