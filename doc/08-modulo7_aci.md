# Módulo 7: Aplicaciones Containerizadas con Azure Container Instances
*Duración: 4 horas*

## Tabla de Contenidos
1. [Introducción a Docker](#1-introducción-a-docker)
2. [Azure Container Instances (ACI)](#2-azure-container-instances-aci)
3. [Creación de Contenedores Docker](#3-creación-de-contenedores-docker)
4. [Configuración de Contenedores](#4-configuración-de-contenedores)
5. [Despliegue en Azure Container Instances](#5-despliegue-en-azure-container-instances)
6. [Laboratorios Prácticos](#6-laboratorios-prácticos)
7. [Mejores Prácticas](#7-mejores-prácticas)

---

## 1. Introducción a Docker

### 1.1 ¿Qué es Docker?
Docker es una plataforma de containerización que permite empaquetar aplicaciones y sus dependencias en contenedores ligeros y portables. Los contenedores proporcionan:

- **Portabilidad**: Ejecuta en cualquier lugar donde Docker esté instalado
- **Aislamiento**: Separación completa entre aplicaciones
- **Eficiencia**: Menor overhead que las máquinas virtuales
- **Escalabilidad**: Fácil escalado horizontal y vertical

### 1.2 Conceptos Fundamentales

#### Imágenes Docker
Una imagen Docker es una plantilla de solo lectura que contiene:
- Sistema operativo base
- Runtime de la aplicación
- Bibliotecas y dependencias
- Código de la aplicación
- Variables de entorno y configuración

#### Contenedores Docker
Un contenedor es una instancia ejecutable de una imagen Docker que incluye:
- La imagen Docker base
- Un entorno de ejecución aislado
- Un sistema de archivos de escritura
- Configuración de red

#### Dockerfile
Archivo de texto que contiene instrucciones para construir una imagen Docker:

```dockerfile
# Ejemplo de Dockerfile para aplicación Node.js
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000
CMD ["npm", "start"]
```

### 1.3 Arquitectura de Docker

#### Componentes Principales
- **Docker Engine**: Motor de containerización
- **Docker Client**: Interfaz de línea de comandos
- **Docker Registry**: Repositorio de imágenes (Docker Hub, ACR)
- **Docker Compose**: Herramienta para aplicaciones multi-contenedor

#### Flujo de Trabajo
1. Desarrollar aplicación
2. Crear Dockerfile
3. Construir imagen (`docker build`)
4. Ejecutar contenedor (`docker run`)
5. Publicar imagen en registro

---

## 2. Azure Container Instances (ACI)

### 2.1 ¿Qué es Azure Container Instances?
Azure Container Instances es un servicio serverless que permite ejecutar contenedores Docker sin gestionar infraestructura. Características principales:

- **Sin servidor**: No gestión de VMs o clusters
- **Facturación por segundo**: Pago solo por tiempo de ejecución
- **Inicio rápido**: Contenedores listos en segundos
- **Aislamiento**: Cada contenedor en su propia VM

### 2.2 Casos de Uso Ideales

#### Aplicaciones Batch
- Procesamiento de datos por lotes
- Trabajos de ETL (Extract, Transform, Load)
- Tareas de machine learning

#### APIs y Microservicios
- APIs REST ligeras
- Microservicios con tráfico variable
- Aplicaciones de prueba y desarrollo

#### Automatización y CI/CD
- Agentes de build
- Tareas de deployment
- Scripts de automatización

### 2.3 Ventajas de ACI

#### Simplicidad
- No configuración de infraestructura
- Modelo de programación simple
- Integración nativa con Azure

#### Escalabilidad
- Escalado automático basado en demanda
- Sin límites de pre-provisioning
- Facturación granular

#### Seguridad
- Aislamiento a nivel de hypervisor
- Integración con Azure AD
- Redes virtuales privadas

---

## 3. Creación de Contenedores Docker

### 3.1 Preparación del Entorno

#### Instalación de Docker Desktop
```bash
# Verificar instalación
docker --version
docker-compose --version
```

#### Configuración Inicial
```bash
# Verificar que Docker está funcionando
docker run hello-world

# Ver imágenes locales
docker images

# Ver contenedores en ejecución
docker ps
```

### 3.2 Construcción de Imágenes

#### Dockerfile Básico
```dockerfile
# Imagen base
FROM ubuntu:20.04

# Metadatos
LABEL maintainer="tu-email@ejemplo.com"
LABEL version="1.0"

# Instalar dependencias
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    && rm -rf /var/lib/apt/lists/*

# Crear directorio de trabajo
WORKDIR /app

# Copiar archivos
COPY src/ ./src/
COPY config/ ./config/

# Exponer puerto
EXPOSE 8080

# Comando por defecto
CMD ["./src/app"]
```

#### Construcción de la Imagen
```bash
# Construir imagen
docker build -t mi-aplicacion:v1.0 .

# Construir con argumentos
docker build --build-arg VERSION=1.2.3 -t mi-app:latest .

# Ver historial de capas
docker history mi-aplicacion:v1.0
```

### 3.3 Mejores Prácticas para Dockerfiles

#### Optimización de Capas
```dockerfile
# ❌ Malo: múltiples capas RUN
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget

# ✅ Bueno: una sola capa RUN
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    && rm -rf /var/lib/apt/lists/*
```

#### Multi-stage Builds
```dockerfile
# Stage 1: Build
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

---

## 4. Configuración de Contenedores

### 4.1 Variables de Entorno

#### Definición en Dockerfile
```dockerfile
# Variables de entorno por defecto
ENV NODE_ENV=production
ENV PORT=3000
ENV DATABASE_URL=postgresql://localhost:5432/mydb
```

#### Uso en tiempo de ejecución
```bash
# Pasar variables individuales
docker run -e NODE_ENV=development -e PORT=8080 mi-app

# Usar archivo de variables
docker run --env-file .env mi-app
```

### 4.2 Volúmenes y Almacenamiento

#### Tipos de Volúmenes
```bash
# Volumen nombrado
docker run -v mi-volumen:/data mi-app

# Mount bind (carpeta host)
docker run -v /host/path:/container/path mi-app

# Volumen temporal
docker run --tmpfs /tmp mi-app
```

#### Persistencia de Datos
```dockerfile
# Definir volumen en Dockerfile
VOLUME ["/data", "/logs"]
```

### 4.3 Redes Docker

#### Tipos de Redes
- **Bridge**: Red por defecto, aislada
- **Host**: Usa la red del host
- **None**: Sin conectividad de red
- **Custom**: Redes definidas por usuario

```bash
# Crear red personalizada
docker network create mi-red

# Ejecutar contenedor en red específica
docker run --network mi-red mi-app

# Conectar contenedor existente
docker network connect mi-red mi-contenedor
```

---

## 5. Despliegue en Azure Container Instances

### 5.1 Preparación del Entorno Azure

#### Instalación de Azure CLI
```bash
# Instalar Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Verificar instalación
az --version

# Iniciar sesión
az login
```

#### Configuración de Recursos
```bash
# Crear grupo de recursos
az group create --name rg-containers --location eastus

# Verificar subscripción activa
az account show
```

### 5.2 Despliegue Básico con CLI

#### Desde Docker Hub
```bash
# Desplegar contenedor público
az container create \
    --resource-group rg-containers \
    --name mi-contenedor \
    --image nginx:latest \
    --dns-name-label mi-app-unica \
    --ports 80
```

#### Desde Azure Container Registry
```bash
# Crear ACR
az acr create \
    --resource-group rg-containers \
    --name miregistro \
    --sku Basic

# Subir imagen a ACR
az acr build --registry miregistro --image mi-app:v1 .

# Desplegar desde ACR
az container create \
    --resource-group rg-containers \
    --name mi-app-acr \
    --image miregistro.azurecr.io/mi-app:v1 \
    --registry-login-server miregistro.azurecr.io \
    --registry-username $(az acr credential show --name miregistro --query username -o tsv) \
    --registry-password $(az acr credential show --name miregistro --query passwords[0].value -o tsv)
```

### 5.3 Configuración Avanzada

#### Archivo YAML de Configuración
```yaml
# aci-deploy.yaml
apiVersion: 2019-12-01
location: eastus
name: mi-aplicacion-grupo
properties:
  containers:
  - name: mi-app
    properties:
      image: mi-registro.azurecr.io/mi-app:latest
      ports:
      - port: 80
        protocol: TCP
      resources:
        requests:
          cpu: 1.0
          memoryInGB: 1.5
      environmentVariables:
      - name: NODE_ENV
        value: production
      - name: DATABASE_URL
        secureValue: postgresql://user:pass@server:5432/db
  - name: sidecar-logging
    properties:
      image: fluent/fluent-bit:latest
      resources:
        requests:
          cpu: 0.5
          memoryInGB: 0.5
  imageRegistryCredentials:
  - server: mi-registro.azurecr.io
    username: mi-usuario
    password: mi-password
  osType: Linux
  restartPolicy: Always
  ipAddress:
    type: Public
    ports:
    - protocol: TCP
      port: 80
    dnsNameLabel: mi-app-unica
tags:
  Environment: Production
  Team: DevOps
```

#### Despliegue con YAML
```bash
# Desplegar usando archivo YAML
az container create --resource-group rg-containers --file aci-deploy.yaml
```

### 5.4 Monitoreo y Logs

#### Visualización de Logs
```bash
# Ver logs en tiempo real
az container logs --resource-group rg-containers --name mi-contenedor --follow

# Ver logs de contenedor específico en grupo
az container logs --resource-group rg-containers --name mi-grupo --container-name mi-app
```

#### Métricas y Alertas
```bash
# Ver métricas de CPU y memoria
az monitor metrics list \
    --resource /subscriptions/{subscription-id}/resourceGroups/rg-containers/providers/Microsoft.ContainerInstance/containerGroups/mi-contenedor \
    --metric "CpuUsage,MemoryUsage"
```

---

## 6. Laboratorios Prácticos

### Laboratorio 1: Aplicación Web Simple

#### Objetivo
Crear y desplegar una aplicación Node.js simple en ACI

#### Archivos del Laboratorio

**package.json**
```json
{
  "name": "mi-app-web",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.0"
  }
}
```

**server.js**
```javascript
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hola desde Azure Container Instances!',
    timestamp: new Date().toISOString(),
    hostname: require('os').hostname()
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'OK' });
});

app.listen(port, () => {
  console.log(`Servidor ejecutándose en puerto ${port}`);
});
```

**Dockerfile**
```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --only=production

COPY server.js ./

EXPOSE 3000

USER node

CMD ["npm", "start"]
```

#### Pasos del Laboratorio
```bash
# 1. Construir imagen
docker build -t mi-app-web:v1.0 .

# 2. Probar localmente
docker run -p 3000:3000 mi-app-web:v1.0

# 3. Subir a ACR
az acr build --registry miregistro --image mi-app-web:v1.0 .

# 4. Desplegar en ACI
az container create \
    --resource-group rg-containers \
    --name mi-app-web \
    --image miregistro.azurecr.io/mi-app-web:v1.0 \
    --dns-name-label mi-web-app-$(date +%s) \
    --ports 3000 \
    --environment-variables NODE_ENV=production \
    --registry-login-server miregistro.azurecr.io \
    --registry-username $(az acr credential show --name miregistro --query username -o tsv) \
    --registry-password $(az acr credential show --name miregistro --query passwords[0].value -o tsv)
```

### Laboratorio 2: Aplicación Multi-Contenedor

#### Objetivo
Desplegar una aplicación con frontend y base de datos

#### Configuración YAML
```yaml
# multi-container.yaml
apiVersion: 2019-12-01
location: eastus
name: app-completa
properties:
  containers:
  - name: frontend
    properties:
      image: nginx:alpine
      ports:
      - port: 80
      resources:
        requests:
          cpu: 0.5
          memoryInGB: 0.5
      volumeMounts:
      - name: nginx-config
        mountPath: /etc/nginx/conf.d
  - name: api
    properties:
      image: node:18-alpine
      ports:
      - port: 3000
      resources:
        requests:
          cpu: 1.0
          memoryInGB: 1.0
      environmentVariables:
      - name: DATABASE_URL
        value: mongodb://mongo:27017/miapp
      command: ["node", "server.js"]
  volumes:
  - name: nginx-config
    azureFile:
      shareName: nginx-config
      storageAccountName: mistorage
      storageAccountKey: "clave-secreta"
  osType: Linux
  ipAddress:
    type: Public
    ports:
    - port: 80
    dnsNameLabel: app-completa-demo
```

---

## 7. Mejores Prácticas

### 7.1 Seguridad

#### Gestión de Secretos
```bash
# Usar Azure Key Vault para secretos
az keyvault create --name mi-keyvault --resource-group rg-containers

# Almacenar secreto
az keyvault secret set --vault-name mi-keyvault --name db-password --value "mi-password-seguro"

# Usar secreto en ACI (requiere identidad administrada)
az container create \
    --resource-group rg-containers \
    --name mi-app-segura \
    --image mi-app:latest \
    --assign-identity \
    --environment-variables DB_HOST=mi-db.com \
    --secure-environment-variables DB_PASSWORD="$(az keyvault secret show --vault-name mi-keyvault --name db-password --query value -o tsv)"
```

#### Imágenes Seguras
- Usar imágenes base oficiales y actualizadas
- Escanear vulnerabilidades con Azure Security Center
- No incluir secretos en imágenes
- Ejecutar como usuario no-root

### 7.2 Rendimiento y Costo

#### Dimensionamiento Adecuado
```bash
# Configurar recursos específicos
az container create \
    --resource-group rg-containers \
    --name mi-app-optimizada \
    --image mi-app:latest \
    --cpu 0.5 \
    --memory 1.0 \
    --restart-policy OnFailure
```

#### Monitoreo de Costos
- Usar etiquetas para tracking de costos
- Implementar políticas de apagado automático
- Monitorear métricas de utilización

### 7.3 Observabilidad

#### Logging Estructurado
```javascript
// Ejemplo de logging estructurado
const winston = require('winston');

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console()
  ]
});

app.use((req, res, next) => {
  logger.info('Request received', {
    method: req.method,
    url: req.url,
    ip: req.ip,
    userAgent: req.get('User-Agent')
  });
  next();
});
```

#### Integración con Azure Monitor
```bash
# Habilitar logs de contenedor
az container create \
    --resource-group rg-containers \
    --name mi-app-monitored \
    --image mi-app:latest \
    --log-analytics-workspace /subscriptions/{id}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{workspace}
```

---

## Comandos de Referencia Rápida

### Docker Comandos Esenciales
```bash
# Construcción y gestión de imágenes
docker build -t nombre:tag .
docker images
docker rmi imagen:tag

# Ejecución de contenedores
docker run -d -p 8080:80 --name mi-contenedor nginx
docker ps
docker stop mi-contenedor
docker rm mi-contenedor

# Debugging
docker exec -it mi-contenedor /bin/bash
docker logs mi-contenedor
docker inspect mi-contenedor
```

### Azure CLI para ACI
```bash
# Gestión de contenedores
az container create --help
az container list --resource-group rg-containers
az container show --name mi-contenedor --resource-group rg-containers
az container delete --name mi-contenedor --resource-group rg-containers

# Monitoreo
az container logs --name mi-contenedor --resource-group rg-containers
az container exec --name mi-contenedor --resource-group rg-containers --exec-command "/bin/bash"
```

---

## Recursos Adicionales

### Documentación Oficial
- [Docker Documentation](https://docs.docker.com/)
- [Azure Container Instances Documentation](https://docs.microsoft.com/azure/container-instances/)
- [Azure Container Registry Documentation](https://docs.microsoft.com/azure/container-registry/)

### Herramientas Útiles
- **Docker Desktop**: Entorno de desarrollo local
- **Azure CLI**: Herramienta de línea de comandos
- **Visual Studio Code**: Con extensiones Docker y Azure
- **Azure Storage Explorer**: Gestión de volúmenes

### Próximos Pasos
1. Explorar Azure Kubernetes Service (AKS) para orquestación avanzada
2. Implementar CI/CD con Azure DevOps
3. Aprender sobre Azure Container Apps para aplicaciones serverless
4. Estudiar patrones de microservicios en Azure

---

*Este material cubre los conceptos fundamentales y prácticos necesarios para trabajar con contenedores Docker y Azure Container Instances. La práctica hands-on con los laboratorios proporcionará la experiencia necesaria para implementar soluciones containerizadas en producción.*