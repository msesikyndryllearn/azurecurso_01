# Guía Completa API REST - Sistema de Pizzas

## 📋 Índice
1. [Modelo de Datos](#modelo-de-datos)
2. [Estructura de Base de Datos](#estructura-de-base-de-datos)
3. [Buenas Prácticas](#buenas-prácticas)
4. [Versionado de API](#versionado-de-api)
5. [Endpoints de la API](#endpoints-de-la-api)
6. [Códigos de Estado HTTP](#códigos-de-estado-http)
7. [Filtros y Paginación](#filtros-y-paginación)
8. [Ejemplos de Uso](#ejemplos-de-uso)

## 🍕 Modelo de Datos

### Pizza
```json
{
  "id": 1,
  "name": "Pizza Margherita",
  "description": "Pizza clásica italiana con tomate, mozzarella y albahaca",
  "url": "https://example.com/images/pizza-margherita.jpg",
  "price": 12.00,
  "ingredients": [
    {
      "id": 1,
      "name": "Tomate",
      "cost": 2.50
    },
    {
      "id": 2,
      "name": "Mozzarella",
      "cost": 5.00
    }
  ]
}
```

### Ingredient
```json
{
  "id": 1,
  "name": "Tomate",
  "cost": 2.50
}
```

**Nota:** El precio de la pizza se calcula automáticamente como: `sum(cost-ingredients) * 1.20` (margen del 20%)

## 🗄️ Estructura de Base de Datos

### Tablas Principales

#### pizza
- `id` (PK, INT, AUTO_INCREMENT)
- `name` (VARCHAR(100), NOT NULL)
- `description` (TEXT)
- `url` (VARCHAR(255))
- `created_at` (TIMESTAMP)
- `updated_at` (TIMESTAMP)

#### ingredients
- `id` (PK, INT, AUTO_INCREMENT)
- `name` (VARCHAR(50), NOT NULL, UNIQUE)
- `cost` (DECIMAL(10,2), NOT NULL)
- `created_at` (TIMESTAMP)
- `updated_at` (TIMESTAMP)

#### pizza_ingredients (Tabla de Relación N:M)
- `pizza_id` (FK, INT, REFERENCES pizza(id))
- `ingredient_id` (FK, INT, REFERENCES ingredients(id))
- `quantity` (DECIMAL(5,2), DEFAULT 1.00)
- `PRIMARY KEY (pizza_id, ingredient_id)`

## ✅ Buenas Prácticas

### 1. URLs Pluralizadas
```
✅ Correcto: /pizzas
❌ Incorrecto: /pizza
```

### 2. No Expresar Acciones en URLs
```
❌ Incorrecto:
- /pizzas/create
- /pizzas/update
- /pizzas/delete

✅ Correcto:
- POST /pizzas (crear)
- PUT /pizzas/{id} (actualizar completo)
- PATCH /pizzas/{id} (actualizar parcial)
- DELETE /pizzas/{id} (eliminar)
```

### 3. No Expresar Formatos en URLs
```
❌ Incorrecto:
- /pizzas.xml
- /pizzas.json

✅ Correcto:
- /pizzas
- Headers: "Accept: application/json" | "Content-Type: application/json"
```

### 4. Usar Sustantivos, No Verbos
```
❌ Incorrecto: /getPizzas, /createPizza
✅ Correcto: GET /pizzas, POST /pizzas
```

### 5. Consistencia en Nomenclatura
- Usar camelCase para propiedades JSON
- Usar snake_case para parámetros de consulta
- Mantener consistencia en toda la API

## 🔄 Versionado de API

### Estrategias de Versionado

#### 1. Versionado por URL (Recomendado)
```
http://localhost:8080/api/v1/pizzas
http://localhost:8080/api/v2/pizzas
```

#### 2. Versionado por Header
```
Headers: "API-Version: v1"
URL: http://localhost:8080/api/pizzas
```

#### 3. Versionado por Query Parameter
```
http://localhost:8080/api/pizzas?version=v1
```

## 🚀 Endpoints de la API

### Base URL
```
http://localhost:8080/api/v1
```

### 1. 📝 Crear Pizza

**Endpoint:** `POST /pizzas`

**Request:**
```http
POST /api/v1/pizzas
Content-Type: application/json

{
  "name": "Pizza Margherita",
  "description": "Pizza clásica italiana con tomate, mozzarella y albahaca",
  "url": "https://example.com/images/pizza-margherita.jpg",
  "ingredients": [1, 2, 3]
}
```

**Response Success (201 Created):**
```json
{
  "id": 1,
  "name": "Pizza Margherita",
  "description": "Pizza clásica italiana con tomate, mozzarella y albahaca",
  "url": "https://example.com/images/pizza-margherita.jpg",
  "price": 12.00,
  "ingredients": [
    {
      "id": 1,
      "name": "Tomate",
      "cost": 2.50
    }
  ],
  "created_at": "2025-06-12T10:00:00Z"
}
```

### 2. ✏️ Actualizar Pizza

#### Actualización Completa (PUT)

**Endpoint:** `PUT /pizzas/{id}`

**Request:**
```http
PUT /api/v1/pizzas/1
Content-Type: application/json

{
  "name": "Pizza Margherita Actualizada",
  "description": "Nueva descripción",
  "url": "https://example.com/images/nueva-imagen.jpg",
  "ingredients": [1, 2, 4]
}
```

**Response Success (200 OK):**
```json
{
  "id": 1,
  "name": "Pizza Margherita Actualizada",
  "description": "Nueva descripción",
  "url": "https://example.com/images/nueva-imagen.jpg",
  "price": 15.60,
  "ingredients": [...],
  "updated_at": "2025-06-12T11:00:00Z"
}
```

#### Actualización Parcial (PATCH)

**Endpoint:** `PATCH /pizzas/{id}`

**Request:**
```http
PATCH /api/v1/pizzas/1
Content-Type: application/json

{
  "name": "Nuevo Nombre Pizza"
}
```

### 3. 🗑️ Eliminar Pizza

**Endpoint:** `DELETE /pizzas/{id}`

**Request:**
```http
DELETE /api/v1/pizzas/1
```

**Response Success (204 No Content):**
```
(Cuerpo vacío)
```

### 4. 👁️ Obtener Pizza por ID

**Endpoint:** `GET /pizzas/{id}`

**Request:**
```http
GET /api/v1/pizzas/1
Accept: application/json
```

**Response Success (200 OK):**
```json
{
  "id": 1,
  "name": "Pizza Margherita",
  "description": "Pizza clásica italiana",
  "url": "https://example.com/images/pizza-margherita.jpg",
  "price": 12.00,
  "ingredients": [
    {
      "id": 1,
      "name": "Tomate",
      "cost": 2.50
    },
    {
      "id": 2,
      "name": "Mozzarella",
      "cost": 5.00
    }
  ],
  "created_at": "2025-06-12T10:00:00Z",
  "updated_at": "2025-06-12T10:00:00Z"
}
```

### 5. 📋 Obtener Lista de Pizzas

**Endpoint:** `GET /pizzas`

**Request:**
```http
GET /api/v1/pizzas
Accept: application/json
```

**Response Success (200 OK):**
```json
{
  "data": [
    {
      "id": 1,
      "name": "Pizza Margherita",
      "description": "Pizza clásica italiana",
      "price": 12.00,
      "ingredients": [...]
    },
    {
      "id": 2,
      "name": "Pizza Pepperoni",
      "description": "Pizza con pepperoni",
      "price": 14.40,
      "ingredients": [...]
    }
  ],
  "meta": {
    "total": 25,
    "page": 1,
    "per_page": 10,
    "total_pages": 3
  }
}
```

## 📊 Códigos de Estado HTTP

### Códigos de Éxito
- **200 OK** - Solicitud exitosa (GET, PUT, PATCH)
- **201 Created** - Recurso creado exitosamente (POST)
- **204 No Content** - Solicitud exitosa sin contenido (DELETE)

### Códigos de Error del Cliente
- **400 Bad Request** - Datos inválidos en la solicitud
- **401 Unauthorized** - No autenticado
- **403 Forbidden** - Autenticado pero sin permisos
- **404 Not Found** - Recurso no encontrado
- **405 Method Not Allowed** - Método HTTP no permitido
- **409 Conflict** - Conflicto con el estado actual del recurso
- **415 Unsupported Media Type** - Tipo de contenido no soportado
- **422 Unprocessable Entity** - Datos válidos pero no procesables

### Códigos de Error del Servidor
- **500 Internal Server Error** - Error interno del servidor
- **503 Service Unavailable** - Servicio no disponible temporalmente

## 🔍 Filtros y Paginación

### Query Parameters Disponibles

#### Filtros
```http
GET /api/v1/pizzas?name=margherita
GET /api/v1/pizzas?min_price=10&max_price=20
GET /api/v1/pizzas?ingredients=tomate,queso
```

#### Paginación
```http
GET /api/v1/pizzas?page=1&size=25
GET /api/v1/pizzas?offset=0&limit=25
```

#### Ordenamiento
```http
GET /api/v1/pizzas?sort=name&order=asc
GET /api/v1/pizzas?sort=price&order=desc
```

#### Búsqueda
```http
GET /api/v1/pizzas?search=car
GET /api/v1/pizzas?q=margherita
```

### Ejemplos de Combinaciones
```http
# Buscar pizzas que contengan "car", página 1, 25 resultados
GET /api/v1/pizzas?name=car&page=1&size=25

# Pizzas ordenadas por precio descendente
GET /api/v1/pizzas?sort=price&order=desc&page=1&size=10

# Pizzas con precio entre 10 y 20 euros
GET /api/v1/pizzas?min_price=10&max_price=20
```

## 💡 Ejemplos de Uso

### Ejemplo 1: Crear una Nueva Pizza
```bash
curl -X POST http://localhost:8080/api/v1/pizzas \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Pizza Hawaiana",
    "description": "Pizza con piña y jamón",
    "url": "https://example.com/hawaiana.jpg",
    "ingredients": [1, 3, 5]
  }'
```

### Ejemplo 2: Obtener Pizzas con Filtros
```bash
curl -X GET "http://localhost:8080/api/v1/pizzas?name=hawaiana&page=1&size=10" \
  -H "Accept: application/json"
```

### Ejemplo 3: Actualizar Pizza
```bash
curl -X PUT http://localhost:8080/api/v1/pizzas/1 \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Pizza Margherita Premium",
    "description": "Pizza margherita con ingredientes premium",
    "ingredients": [1, 2, 6]
  }'
```

### Ejemplo 4: Eliminar Pizza
```bash
curl -X DELETE http://localhost:8080/api/v1/pizzas/1
```

## 🛡️ Consideraciones de Seguridad

### Autenticación y Autorización
- Implementar JWT tokens para autenticación
- Validar permisos en cada endpoint
- Rate limiting para prevenir abuso

### Validación de Datos
- Validar todos los inputs del cliente
- Sanitizar datos antes de procesarlos
- Implementar validación tanto en frontend como backend

### Headers de Seguridad
```http
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Content-Security-Policy: default-src 'self'
```

## 📚 Recursos Adicionales

### Herramientas Recomendadas
- **Postman** - Testing de APIs
- **Swagger/OpenAPI** - Documentación de APIs
- **Insomnia** - Cliente REST alternativo

### Estándares y Especificaciones
- [RFC 7231 - HTTP/1.1 Semantics](https://tools.ietf.org/html/rfc7231)
- [JSON API Specification](https://jsonapi.org/)
- [REST API Design Best Practices](https://restfulapi.net/)

---

**Nota:** Esta documentación debe mantenerse actualizada con cualquier cambio en la API. Considera implementar versionado semántico (SemVer) para el control de versiones de tu API.