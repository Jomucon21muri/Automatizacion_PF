# Sesión 5: Fundamentos de APIs REST

## Objetivos de aprendizaje

Al finalizar esta sesión, serás capaz de:

- Comprender la arquitectura REST y sus principios
- Realizar peticiones HTTP (GET, POST, PUT, DELETE)
- Autenticar requests con API keys, OAuth, JWT
- Manejar respuestas JSON y errores
- Documentar y testear APIs

## ¿Qué es una API REST?

**API** (Application Programming Interface) es un conjunto de reglas que permite que diferentes aplicaciones se comuniquen entre sí.

**REST** (Representational State Transfer) es un estilo arquitectónico para diseñar APIs basadas en HTTP.

!!! quote "Definición Técnica"
    Una API REST permite acceder y manipular recursos mediante URLs estándar usando los métodos HTTP.

### Analogía del Restaurante

```mermaid
graph LR
    A[Cliente<br/>Tu aplicación] -->|Orden| B[Camarero<br/>API]
    B -->|Solicitud| C[Cocina<br/>Servidor/Base de Datos]
    C -->|Datos| B
    B -->|Respuesta| A
    
    style A fill:#e3f2fd
    style B fill:#fff9c4
    style C fill:#e8f5e9
```

- **Cliente**: Tu aplicación (n8n, Make, código)
- **API (Camarero)**: Interfaz que toma pedidos
- **Servidor (Cocina)**: Procesa la solicitud y devuelve datos

## Principios de REST

### 1. Cliente-Servidor

Separación de responsabilidades:
- Cliente maneja UI/UX
- Servidor maneja datos y lógica de negocio

### 2. Stateless (Sin Estado)

Cada request es independiente, contiene toda la información necesaria.

```javascript
// ❌ MAL (stateful)
POST /api/login → servidor guarda sesión
GET /api/transactions → usa sesión guardada

// ✅ BIEN (stateless)
GET /api/transactions
Headers: Authorization: Bearer TOKEN_XYZ123
```

### 3. Cacheable

Respuestas pueden ser cacheadas para mejor performance.

### 4. Interface uniforme

URLs y métodos HTTP estandarizados.

## Métodos HTTP

### Verbos CRUD

| Método | Operación | Uso | Idempotente |
|--------|-----------|-----|-------------|
| **GET** | Read (Leer) | Obtener datos | Sí |
| **POST** | Create (Crear) | Crear nuevo recurso | No |
| **PUT** | Update (Actualizar) | Actualizar recurso completo | Sí |
| **PATCH** | Update parcial | Actualizar campos específicos | No |
| **DELETE** | Delete (Eliminar) | Eliminar recurso | Sí |

**Idempotente**: Múltiples requests idénticos tienen el mismo efecto que uno solo.

### Ejemplos financieros

#### GET - Obtener transacciones

```http
GET /api/v1/transactions?from=2024-01-01&to=2024-03-27
Host: api.banco.com
Authorization: Bearer abc123xyz
```

**Respuesta (200 OK)**:

```json
{
  "data": [
    {
      "id": "tx_001",
      "amount": 150.50,
      "currency": "USD",
      "date": "2024-03-26",
      "description": "Amazon purchase"
    },
    {
      "id": "tx_002",
      "amount": -75.00,
      "currency": "USD",
      "date": "2024-03-27",
      "description": "ATM withdrawal"
    }
  ],
  "total": 2,
  "page": 1
}
```

#### POST - Crear transferencia

```http
POST /api/v1/transfers
Host: api.banco.com
Authorization: Bearer abc123xyz
Content-Type: application/json

{
  "from_account": "ACC_123",
  "to_account": "ACC_456",
  "amount": 500.00,
  "currency": "EUR",
  "description": "Payment to supplier"
}
```

**Respuesta (201 Created)**:

```json
{
  "transfer_id": "trf_789",
  "status": "pending",
  "created_at": "2024-03-27T10:30:00Z",
  "estimated_completion": "2024-03-27T12:00:00Z"
}
```

#### PUT - Actualizar cuenta

```http
PUT /api/v1/accounts/ACC_123
Host: api.banco.com
Authorization: Bearer abc123xyz
Content-Type: application/json

{
  "nickname": "Cuenta Principal",
  "alert_threshold": 1000.00,
  "notifications": true
}
```

#### DELETE - Cancelar transferencia programada

```http
DELETE /api/v1/scheduled_transfers/sch_456
Host: api.banco.com
Authorization: Bearer abc123xyz
```

**Respuesta (204 No Content)** (sin body)

## Anatomía de una Request HTTP

### Estructura completa

```http
POST /api/v1/payments HTTP/1.1
Host: api.stripe.com
Authorization: Bearer sk_live_XXX
Content-Type: application/json
User-Agent: MyApp/1.0
Accept: application/json
X-Idempotency-Key: unique-key-123

{
  "amount": 2000,
  "currency": "usd",
  "source": "tok_visa",
  "description": "Charge for customer@example.com"
}
```

**Componentes**:

1. **Request Line**: Método + Ruta + Versión HTTP
2. **Headers**: Metadata (autorización, tipo contenido)
3. **Body**: Datos enviados (solo POST/PUT/PATCH)

### Headers comunes

| Header | Propósito | Ejemplo |
|--------|-----------|---------|
| **Authorization** | Autenticación | `Bearer token123` |
| **Content-Type** | Formato del body | `application/json` |
| **Accept** | Formato esperado de respuesta | `application/json` |
| **User-Agent** | Identificación del cliente | `n8n/1.0` |
| **X-API-Key** | API key alternativa | `abc-123-xyz-789` |

## Códigos de Estado HTTP

### Categorías

```mermaid
mindmap
  root((Status Codes))
    2xx Success
      200 OK
      201 Created
      204 No Content
    3xx Redirection
      301 Moved Permanently
      304 Not Modified
    4xx Client Error
      400 Bad Request
      401 Unauthorized
      403 Forbidden
      404 Not Found
      429 Too Many Requests
    5xx Server Error
      500 Internal Server Error
      502 Bad Gateway
      503 Service Unavailable
```

### Códigos críticos en finanzas

| Código | Nombre | Significado | Acción |
|--------|--------|-------------|--------|
| **200** | OK | Éxito | Procesar datos |
| **201** | Created | Recurso creado | Confirmar creación |
| **400** | Bad Request | Datos inválidos | Validar inputs |
| **401** | Unauthorized | No autenticado | Refresh token |
| **403** | Forbidden | Sin permisos | Verificar scopes |
| **404** | Not Found | Recurso no existe | Manejar ausencia |
| **429** | Too Many Requests | Rate limit excedido | Implementar retry con backoff |
| **500** | Server Error | Error del servidor | Reintentar más tarde |

## Formatos de datos

### JSON (JavaScript Object Notation)

Estándar de facto para APIs modernas.

```json
{
  "customer": {
    "id": "cust_123",
    "name": "María García",
    "email": "maria@example.com",
    "accounts": [
      {
        "type": "checking",
        "balance": 5430.25,
        "currency": "EUR"
      },
      {
        "type": "savings",
        "balance": 15000.00,
        "currency": "EUR"
      }
    ],
    "status": "active",
    "vip": true,
    "created_at": "2020-05-15T14:30:00Z"
  }
}
```

**Tipos de datos**:
- String: `"texto"`
- Number: `123`, `45.67`
- Boolean: `true`, `false`
- Null: `null`
- Array: `[1, 2, 3]`
- Object: `{"key": "value"}`

### XML (Legacy)

```xml
<customer>
  <id>cust_123</id>
  <name>María García</name>
  <email>maria@example.com</email>
  <accounts>
    <account>
      <type>checking</type>
      <balance>5430.25</balance>
      <currency>EUR</currency>
    </account>
  </accounts>
</customer>
```

Menos común en APIs modernas, pero usado en sistemas legacy bancarios.

## Autenticación y Seguridad

### 1. API Keys

Más simple, menos seguro.

```http
GET /api/data
X-API-Key: abc123xyz789
```

**Pros**: Fácil implementación  
**Cons**: Si se filtra, acceso total

### 2. Bearer Tokens

Token de acceso en header Authorization.

```http
GET /api/data
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### 3. OAuth 2.0

Estándar para delegación de acceso.

```mermaid
sequenceDiagram
    participant User
    participant App
    participant AuthServer
    participant API
    
    User->>App: Quiero conectar mi banco
    App->>AuthServer: Redirect a página de login
    User->>AuthServer: Login + Autoriza acceso
    AuthServer->>App: Authorization Code
    App->>AuthServer: Exchange code por Access Token
    AuthServer->>App: Access Token + Refresh Token
    App->>API: Request con Access Token
    API->>App: Datos protegidos
```

**Ventajas**:
- No se comparten credenciales
- Permisos granulares (scopes)
- Tokens expirables

**Ejemplo Plaid**:

```javascript
// Obtener access token
POST https://production.plaid.com/item/public_token/exchange
{
  "client_id": "YOUR_CLIENT_ID",
  "secret": "YOUR_SECRET",
  "public_token": "public-sandbox-xxx"
}

// Respuesta
{
  "access_token": "access-sandbox-xxx",
  "item_id": "item-xxx"
}

// Usar access token
POST https://production.plaid.com/transactions/get
{
  "access_token": "access-sandbox-xxx",
  "start_date": "2024-01-01",
  "end_date": "2024-03-27"
}
```

### 4. JWT (JSON Web Tokens)

Token auto-contenido con información codificada.

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**Estructura**: Header.Payload.Signature

**Decodificado**:

```json
{
  "header": {
    "alg": "HS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "1234567890",
    "name": "John Doe",
    "iat": 1516239022,
    "exp": 1516242622,
    "scopes": ["read:transactions", "write:transfers"]
  }
}
```

## Herramientas de testing

### Postman

Interface gráfica para testear APIs.

```
1. Crear Collection "Banking API"
2. Add Request GET /transactions
3. Set Headers: Authorization: Bearer XXX
4. Send → Ver respuesta
5. Save examples
6. Generate code snippets
```

### cURL (Command Line)

```bash
# GET request
curl -X GET "https://api.example.com/transactions" \
  -H "Authorization: Bearer abc123" \
  -H "Accept: application/json"

# POST request
curl -X POST "https://api.example.com/transfers" \
  -H "Authorization: Bearer abc123" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 100,
    "from": "ACC_1",
    "to": "ACC_2"
  }'

# With verbose output
curl -v -X GET "https://api.example.com/data"
```

### HTTPie (Más legible)

```bash
# GET
http GET https://api.example.com/transactions \
  Authorization:"Bearer abc123"

# POST
http POST https://api.example.com/transfers \
  Authorization:"Bearer abc123" \
  amount:=100 \
  from=ACC_1 \
  to=ACC_2
```

## Rate Limiting

APIs financieras limitan número de requests para prevenir abuso.

### Headers de rate limit

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 987
X-RateLimit-Reset: 1617123600
```

### Manejo de rate limits

```javascript
// Pseudo-código
async function callAPIWithRetry(url, maxRetries = 3) {
  let retries = 0;
  
  while (retries < maxRetries) {
    const response = await fetch(url);
    
    if (response.status === 200) {
      return response.json();
    }
    
    if (response.status === 429) {
      // Too Many Requests
      const resetTime = response.headers.get('X-RateLimit-Reset');
      const waitTime = resetTime - Date.now() / 1000;
      
      console.log(`Rate limited. Waiting ${waitTime}s...`);
      await sleep(waitTime * 1000);
      retries++;
    } else {
      throw new Error(`API error: ${response.status}`);
    }
  }
  
  throw new Error('Max retries exceeded');
}
```

## Paginación

APIs limitan resultados por request.

### Tipos de paginación

#### 1. Offset-based

```http
GET /transactions?limit=50&offset=0   # Primeros 50
GET /transactions?limit=50&offset=50  # Siguientes 50
GET /transactions?limit=50&offset=100 # Siguientes 50
```

#### 2. Cursor-based

```http
GET /transactions?limit=50
```

**Respuesta**:

```json
{
  "data": [...],
  "pagination": {
    "next_cursor": "cursor_abc123",
    "has_more": true
  }
}
```

**Siguiente página**:

```http
GET /transactions?limit=50&cursor=cursor_abc123
```

#### 3. Page-based

```http
GET /transactions?page=1&per_page=50
GET /transactions?page=2&per_page=50
```

## Documentación de APIs

### OpenAPI / Swagger

Estándar para documentar APIs REST.

```yaml
openapi: 3.0.0
info:
  title: Banking API
  version: 1.0.0
paths:
  /transactions:
    get:
      summary: List transactions
      parameters:
        - name: from
          in: query
          schema:
            type: string
            format: date
        - name: to
          in: query
          schema:
            type: string
            format: date
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Transaction'
```

### Documentación Interactiva

Herramientas generan UI interactiva desde OpenAPI:

- Swagger UI
- Redoc
- Postman Collections

## Ejercicio práctico

### Tarea: Explorar Alpha Vantage API

**Objetivo**: Familiarizarte con una API financiera real.

**Pasos**:

1. **Registrarte**: [https://www.alphavantage.co/support/#api-key](https://www.alphavantage.co/support/#api-key)

2. **GET Stock Data**:
```bash
curl "https://www.alphavantage.co/query?function=TIME_SERIES_DAILY&symbol=AAPL&apikey=YOUR_API_KEY"
```

3. **Analizar respuesta**:
   - ¿Qué estructura tiene?
   - ¿Qué campos contiene?
   - ¿Cómo está organizado el JSON?

4. **Testear endpoints**:
   - Global Quote (precio actual)
   - Technical Indicators (RSI, SMA)
   - Forex rates

5. **Documentar**:
   - Request examples
   - Response examples
   - Rate limits
   - Error handling

**Entregable**: Documento con 5 requests diferentes y sus respuestas.

## Recursos

### APIs financieras para practicar

| API | Propósito | Gratuito |
|-----|-----------|----------|
| **Alpha Vantage** | Stock data | Sí |
| **CoinGecko** | Crypto prices | Sí |
| **ExchangeRate-API** | Forex | Sí |
| **IEX Cloud** | Financial data | Sandbox gratis |
| **Plaid** | Banking (sandbox) | Sí |

### Documentación

- [HTTP Methods - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods)
- [REST API Tutorial](https://restfulapi.net/)
- [OpenAPI Specification](https://swagger.io/specification/)

## Resumen

En esta sesión aprendimos:

✅ Arquitectura y principios REST  
✅ Métodos HTTP (GET, POST, PUT, DELETE)  
✅ Autenticación (API Keys, OAuth, JWT)  
✅ Manejo de respuestas y errores  
✅ Rate limiting y paginación  
✅ Herramientas de testing (Postman, cURL)  

**Próxima sesión**: Exploraremos **APIs financieras específicas** (Stripe, Plaid, Yahoo Finance) y cómo integrarlas en workflows.

---

!!! tip "Tarea para la Próxima Sesión"
    1. Completa el ejercicio con Alpha Vantage
    2. Instala Postman
    3. Crea cuenta en Stripe (modo test)
    4. Lee documentación de Plaid API
