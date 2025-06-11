
# Módulo 6: Escalado en Azure App Service (2 horas)

## 1. Escalado manual y automático (autoscaling)

### ¿Qué es?
El **escalado** permite ajustar los recursos asignados a tu App Service en Azure para asegurar un rendimiento adecuado ante diferentes cargas de trabajo. Puede ser:

- **Manual**: Cambias tú mismo el número de instancias.
- **Automático (autoscaling)**: Azure ajusta automáticamente el número de instancias basado en métricas como uso de CPU, memoria, etc.

### Desde el Portal de Azure

#### Escalado Manual:
1. Ve al recurso de tu **App Service**.
2. En el menú izquierdo, selecciona **"Escala horizontal (Scale out, App Service plan)"**.
3. Ajusta manualmente el número de instancias.
4. Guarda los cambios.

#### Escalado Automático:
1. Dentro de **"Escala horizontal"**, haz clic en **"Habilitar reglas de escalado automático (Autoscale)"**.
2. Crea un nuevo perfil de escalado.
3. Añade reglas de escalado con condiciones como uso de CPU.
4. Guarda el perfil.

### Desde Azure CLI

#### Escalado Manual:
```bash
az appservice plan update \
  --name NOMBRE_PLAN \
  --resource-group NOMBRE_GRUPO \
  --number-of-workers 3
```

#### Escalado Automático:
```bash
az monitor autoscale create \
  --resource-group NOMBRE_GRUPO \
  --resource NOMBRE_PLAN \
  --resource-type "Microsoft.Web/serverfarms" \
  --name autoScaleConfig \
  --min-count 1 \
  --max-count 5 \
  --count 2

az monitor autoscale rule create \
  --resource-group NOMBRE_GRUPO \
  --autoscale-name autoScaleConfig \
  --condition "Percentage CPU > 70 avg 10m" \
  --scale out 1

az monitor autoscale rule create \
  --resource-group NOMBRE_GRUPO \
  --autoscale-name autoScaleConfig \
  --condition "Percentage CPU < 30 avg 10m" \
  --scale in 1
```

---

## 2. Gestión de slots de despliegue (staging, production)

### ¿Qué son los deployment slots?
Los **slots de despliegue** permiten tener varias versiones de una app en el mismo App Service. Ejemplo:
- **Producción** para usuarios finales.
- **Staging** para pruebas sin afectar producción.

### Desde el Portal de Azure

1. Ve al **App Service**.
2. Selecciona **"Slots de implementación"**.
3. Crea un slot con **"+ Agregar"**.
4. Despliega al slot `staging`.
5. Prueba la aplicación.
6. Usa el botón **"Swap"** para intercambiar con producción.

### Desde Azure CLI

#### Crear un nuevo slot:
```bash
az webapp deployment slot create \
  --name NOMBRE_APP \
  --resource-group NOMBRE_GRUPO \
  --slot staging
```

#### Desplegar a un slot:
```bash
az webapp deployment source config-zip \
  --resource-group NOMBRE_GRUPO \
  --name NOMBRE_APP \
  --slot staging \
  --src ruta/paquete.zip
```

#### Ver la URL del slot:
```bash
az webapp show \
  --name NOMBRE_APP \
  --resource-group NOMBRE_GRUPO \
  --slot staging \
  --query defaultHostName
```

#### Hacer swap entre slots:
```bash
az webapp deployment slot swap \
  --name NOMBRE_APP \
  --resource-group NOMBRE_GRUPO \
  --slot staging \
  --target-slot production
```

---

## Resumen

| Funcionalidad             | Portal Azure                         | Azure CLI                                               |
|--------------------------|--------------------------------------|---------------------------------------------------------|
| Escalado Manual          | Ajuste en "Escala horizontal"        | `az appservice plan update`                            |
| Autoscaling              | Crear reglas en "Escala horizontal" | `az monitor autoscale create` y `az monitor autoscale rule create` |
| Crear Deployment Slot    | "Deployment slots" → "+Agregar"     | `az webapp deployment slot create`                     |
| Swap entre Slots         | Botón "Swap"                         | `az webapp deployment slot swap`                       |
