# Guía de Escalado Automático para App Service en Azure

## Tabla de Contenidos
1. [Introducción](#introducción)
2. [Prerrequisitos](#prerrequisitos)
3. [Configuración Paso a Paso](#configuración-paso-a-paso)
4. [Métricas Específicas de App Service](#métricas-específicas-de-app-service)
5. [Configuraciones Recomendadas](#configuraciones-recomendadas)
6. [Monitoreo y Troubleshooting](#monitoreo-y-troubleshooting)
7. [Mejores Prácticas](#mejores-prácticas)
8. [Costos y Optimización](#costos-y-optimización)

## Introducción

El escalado automático en Azure App Service permite ajustar automáticamente el número de instancias de tu aplicación web basándose en métricas como CPU, memoria, peticiones HTTP y longitud de cola, garantizando rendimiento óptimo y control de costos.

### Beneficios para App Service
- **Respuesta automática**: Escala según demanda real de tu aplicación web
- **Optimización de costos**: Paga solo por las instancias que necesitas
- **Alta disponibilidad**: Mantiene la aplicación responsive durante picos de tráfico
- **Sin intervención manual**: Funciona 24/7 sin supervisión

## Prerrequisitos

### Planes de App Service Compatibles
- **Standard (S1, S2, S3)**
- **Premium (P1, P2, P3)**
- **Premium V2 (P1v2, P2v2, P3v2)**
- **Premium V3 (P1v3, P2v3, P3v3)**

> **Nota**: Los planes Basic y Free NO soportan escalado automático.

### Permisos Necesarios
- **Contributor** o **Owner** en el App Service Plan
- **Monitoring Contributor** para configurar métricas y alertas

## Configuración Paso a Paso

### Paso 1: Acceder a tu App Service

1. Inicia sesión en [Azure Portal](https://portal.azure.com)
2. Ve a **"Servicios de aplicaciones"** o busca tu App Service
3. Selecciona tu aplicación web

### Paso 2: Navegar al App Service Plan

1. En el menú lateral de tu App Service, busca **"App Service Plan"**
2. Haz clic en el nombre de tu plan (ej: "ASP-myapp-12345")
3. Esto te llevará a la configuración del plan

### Paso 3: Habilitar Escalado Automático

1. En el menú lateral del App Service Plan, selecciona **"Escalado horizontal (App Service Plan)"**
2. Haz clic en **"Escalado automático personalizado"**
3. Asigna un nombre a tu configuración (ej: "Autoscale-WebApp-Prod")

### Paso 4: Configurar Límites de Instancias

En la sección **"Límites de instancia"**:
- **Mínimo**: 1-2 instancias (para apps críticas mínimo 2)
- **Máximo**: Basado en tu presupuesto y carga esperada (ej: 10)
- **Por defecto**: Número de instancias cuando no hay reglas activas (ej: 2)

### Paso 5: Crear Reglas de Escalado

#### Regla 1: Escalado Hacia Arriba por CPU
1. Haz clic en **"Agregar una regla"**
2. Configura:
   - **Origen de métrica**: Recurso actual
   - **Métrica**: CPU Percentage
   - **Estadística de tiempo**: Average
   - **Operador**: Greater than
   - **Umbral**: 70
   - **Duración**: 10 minutos
3. Acción:
   - **Operación**: Increase count by
   - **Recuento de instancias**: 1
   - **Tiempo de enfriamiento**: 5 minutos

#### Regla 2: Escalado Hacia Arriba por Peticiones HTTP
1. Haz clic en **"Agregar una regla"**
2. Configura:
   - **Origen de métrica**: Recurso actual
   - **Métrica**: Requests
   - **Estadística de tiempo**: Total
   - **Operador**: Greater than
   - **Umbral**: 1000 (peticiones en 5 minutos)
   - **Duración**: 5 minutos
3. Acción:
   - **Operación**: Increase count by
   - **Recuento de instancias**: 1
   - **Tiempo de enfriamiento**: 3 minutos

#### Regla 3: Escalado Hacia Abajo por CPU
1. Haz clic en **"Agregar una regla"**
2. Configura:
   - **Origen de métrica**: Recurso actual
   - **Métrica**: CPU Percentage
   - **Estadística de tiempo**: Average
   - **Operador**: Less than
   - **Umbral**: