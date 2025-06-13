# Guía de Azure: Blob Storage y Cosmos DB

## Azure Blob Storage - Conceptos Básicos y Gestión

### ¿Qué es Azure Blob Storage?

Azure Blob Storage es el servicio de almacenamiento de objetos de Microsoft Azure, diseñado para almacenar grandes cantidades de datos no estructurados como texto o datos binarios. Es ideal para servir imágenes o documentos directamente a un navegador, almacenar archivos para acceso distribuido, transmitir video y audio, y realizar copias de seguridad.

### Conceptos Fundamentales

#### Estructura Jerárquica
- **Cuenta de Storage**: Contenedor de nivel superior que proporciona un espacio de nombres único
- **Contenedor**: Organiza un conjunto de blobs, similar a una carpeta
- **Blob**: El archivo individual almacenado

#### Tipos de Blobs
1. **Block Blobs**: Optimizados para cargar grandes cantidades de datos de manera eficiente (hasta 190.7 TB)
2. **Append Blobs**: Optimizados para operaciones de anexado, ideales para logging
3. **Page Blobs**: Optimizados para operaciones de lectura/escritura aleatorias, usados para discos VHD

### Niveles de Acceso (Access Tiers)

#### Hot Tier (Caliente)
- Acceso frecuente a los datos
- Costos de almacenamiento más altos, costos de acceso más bajos
- Ideal para datos activos

#### Cool Tier (Frío)
- Acceso infrecuente (al menos 30 días)
- Costos de almacenamiento más bajos, costos de acceso más altos
- Ideal para copias de seguridad a corto plazo

#### Archive Tier (Archivo)
- Acceso muy infrecuente (al menos 180 días)
- Costos de almacenamiento más bajos, pero requiere rehidratación
- Ideal para archivos a largo plazo

### Gestión de Azure Blob Storage

#### Creación de una Cuenta de Storage
```bash
# Azure CLI
az storage account create \
    --name mystorageaccount \
    --resource-group myResourceGroup \
    --location eastus \
    --sku Standard_LRS
```

#### Creación de Contenedores
```bash
# Azure CLI
az storage container create \
    --account-name mystorageaccount \
    --name mycontainer \
    --auth-mode login
```

#### Subida de Archivos
```bash
# Azure CLI
az storage blob upload \
    --account-name mystorageaccount \
    --container-name mycontainer \
    --name myblob.txt \
    --file ~/myfile.txt
```

#### Configuración de Políticas de Ciclo de Vida
Las políticas permiten automatizar la transición entre niveles de acceso:
- Mover blobs a Cool después de 30 días sin acceso
- Mover blobs a Archive después de 90 días sin acceso
- Eliminar blobs después de 365 días

### Seguridad en Blob Storage

#### Métodos de Autenticación
1. **Shared Access Signatures (SAS)**: Tokens con permisos específicos y tiempo limitado
2. **Azure Active Directory**: Integración con identidades corporativas
3. **Account Keys**: Claves de acceso completo (uso no recomendado para producción)

#### Cifrado
- **En reposo**: Automático con claves gestionadas por Microsoft o cliente
- **En tránsito**: HTTPS obligatorio para todas las operaciones

### Casos de Uso Comunes
- Almacenamiento de contenido web estático
- Backup y archivado de datos
- Análisis de big data
- Distribución de contenido multimedia
- Almacenamiento de logs y datos de telemetría

---

## Introducción a Azure Cosmos DB

### ¿Qué es Azure Cosmos DB?

Azure Cosmos DB es un servicio de base de datos NoSQL totalmente gestionado, diseñado para aplicaciones modernas que requieren alta disponibilidad, escalabilidad global y baja latencia. Ofrece distribución global automática, escalado elástico y múltiples modelos de datos.

### Características Principales

#### Distribución Global
- Replicación automática en múltiples regiones de Azure
- Latencia menor a 10ms para lecturas y escrituras
- Failover automático sin tiempo de inactividad

#### Múltiples Modelos de Datos
1. **SQL (Core)**: API familiar de SQL con documentos JSON
2. **MongoDB**: Compatibilidad con aplicaciones MongoDB existentes
3. **Cassandra**: Para aplicaciones que requieren wide-column
4. **Table**: Reemplazo mejorado para Azure Table Storage
5. **Gremlin**: Para bases de datos de grafos

#### Niveles de Consistencia
1. **Strong**: Consistencia linealizable
2. **Bounded Staleness**: Desfase limitado por tiempo o versiones
3. **Session**: Consistencia dentro de una sesión de cliente
4. **Consistent Prefix**: Las lecturas nunca ven escrituras fuera de orden
5. **Eventual**: Eventual convergencia

### Conceptos Fundamentales

#### Jerarquía de Recursos
- **Cuenta de Cosmos DB**: Unidad de distribución global y facturación
- **Base de Datos**: Contenedor lógico de contenedores
- **Contenedor**: Unidad de escalabilidad y distribución
- **Ítem**: Documento individual (JSON para SQL API)

#### Request Units (RUs)
- Unidad de medida normalizada para el rendimiento
- Combina CPU, memoria y IOPS en una sola métrica
- Facturación basada en RUs consumidas por segundo

### Configuración y Gestión

#### Creación de una Cuenta Cosmos DB
```bash
# Azure CLI
az cosmosdb create \
    --name mycosmosaccount \
    --resource-group myResourceGroup \
    --kind GlobalDocumentDB \
    --locations regionName=eastus failoverPriority=0 \
    --default-consistency-level Session
```

#### Creación de Base de Datos y Contenedor
```bash
# Crear base de datos
az cosmosdb sql database create \
    --account-name mycosmosaccount \
    --resource-group myResourceGroup \
    --name mydatabase

# Crear contenedor
az cosmosdb sql container create \
    --account-name mycosmosaccount \
    --resource-group myResourceGroup \
    --database-name mydatabase \
    --name mycontainer \
    --partition-key-path "/id" \
    --throughput 400
```

### Estrategias de Particionado

#### Clave de Partición
- Propiedad JSON que determina cómo se distribuyen los datos
- Debe tener alta cardinalidad y distribución uniforme
- Ejemplos buenos: userId, deviceId, timestamp truncado
- Ejemplos malos: país, tipo de documento, estado booleano

#### Mejores Prácticas
- Elegir claves que distribuyan uniformemente las operaciones
- Evitar hot partitions (particiones sobrecargadas)
- Considerar patrones de consulta futuros

### Indexación

#### Política de Indexación por Defecto
- Indexación automática de todas las propiedades
- Índices de rango para strings y números
- Índices espaciales para datos geoespaciales

#### Personalización
```json
{
    "indexingMode": "consistent",
    "automatic": true,
    "includedPaths": [
        {
            "path": "/*"
        }
    ],
    "excludedPaths": [
        {
            "path": "/metadata/*"
        }
    ]
}
```

### Seguridad y Gobernanza

#### Control de Acceso
- **Master Keys**: Acceso completo a la cuenta
- **Resource Tokens**: Acceso granular a recursos específicos
- **Azure Active Directory**: Integración con identidades corporativas

#### Cifrado
- Cifrado en reposo automático
- Cifrado en tránsito con TLS
- Soporte para claves gestionadas por el cliente

### Monitoreo y Diagnóstico

#### Métricas Importantes
- Request Units consumidas
- Latencia de operaciones
- Disponibilidad por región
- Errores de throttling

#### Herramientas de Diagnóstico
- Azure Monitor integration
- Diagnostic logs
- Query performance insights
- Cosmos DB Explorer

### Casos de Uso Ideales

#### Aplicaciones Web y Móviles
- Catálogos de productos
- Gestión de perfiles de usuario
- Sistemas de recomendaciones

#### IoT y Telemetría
- Ingesta de datos de sensores
- Análisis en tiempo real
- Archivado de datos históricos

#### Gaming
- Leaderboards globales
- Perfiles de jugadores
- Estados de juego en tiempo real

#### Retail y E-commerce
- Inventario en tiempo real
- Personalización de contenido
- Análisis de comportamiento

### Migración a Cosmos DB

#### Herramientas de Migración
- **Data Migration Tool**: GUI para migraciones simples
- **Azure Data Factory**: Para migraciones complejas y ETL
- **Spark Connector**: Para big data migrations
- **MongoDB native tools**: Para migraciones desde MongoDB

#### Estrategias de Migración
1. **Lift and Shift**: Migración directa con cambios mínimos
2. **Refactor**: Optimización para aprovechar características de Cosmos DB
3. **Hybrid**: Migración gradual con coexistencia temporal

---

## Comparación y Cuándo Usar Cada Servicio

### Azure Blob Storage - Usar Cuando:
- Necesites almacenar archivos, imágenes, videos
- Requieras backup y archivado de datos
- Implementes contenido web estático
- Manejes grandes volúmenes de datos no estructurados
- El costo sea un factor crítico

### Azure Cosmos DB - Usar Cuando:
- Necesites baja latencia global (< 10ms)
- Requieras alta disponibilidad (99.999%)
- Tengas patrones de acceso impredecibles
- Necesites escalado automático
- Manejes datos semi-estructurados o NoSQL
- La aplicación tenga usuarios globales

---

## Recursos Adicionales

### Documentación Oficial
- [Azure Blob Storage Documentation](https://docs.microsoft.com/azure/storage/blobs/)
- [Azure Cosmos DB Documentation](https://docs.microsoft.com/azure/cosmos-db/)

### Herramientas Útiles
- Azure Storage Explorer
- Azure Cosmos DB Explorer
- Azure CLI
- Azure PowerShell
- SDKs para múltiples lenguajes

### Capacitación y Certificaciones
- AZ-104: Azure Administrator Associate
- AZ-204: Azure Developer Associate
- DP-420: Designing and Implementing Cloud-Native Applications Using Microsoft Azure Cosmos DB