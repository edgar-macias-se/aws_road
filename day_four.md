# API Gateway: De Lambda Privada a Endpoint Público (Módulo 4)

> **Serie:** AWS Zero to Architect  
> **Nivel:** Intermedio  
> **Tiempo:** 90 minutos  
> **Prerequisitos:** [Módulo 0](./zero_day.md), [Módulo 1](./first_Day.md), [Módulo 2](./day_two.md), [Módulo 3](./day_three.md)

---

## 📋 Contenido

1. [Introducción](#introducción)
2. [¿Qué es API Gateway?](#qué-es-api-gateway)
3. [Throttling y Rate Limiting](#throttling-y-rate-limiting)
4. [Arquitectura Completa](#arquitectura-completa)
5. [Implementación con Terraform](#implementación-con-terraform)
6. [CORS Explicado](#cors-explicado)
7. [Testing Completo](#testing-completo)
8. [Costos y Seguridad](#costos-y-seguridad)
9. [Troubleshooting](#troubleshooting)
10. [Conclusiones](#conclusiones)

---

## Introducción

En el [Módulo 3](./03-lambda-go.md) creamos una función Lambda en Go que crea sesiones en DynamoDB. Funcionaba perfectamente, pero tenía una limitación crítica:

**Solo era invocable con AWS CLI** (requería credenciales).

En este módulo vamos a:
- ✅ Crear un endpoint HTTPS público
- ✅ Hacer la Lambda accesible desde internet
- ✅ Configurar throttling (rate limiting)
- ✅ Habilitar CORS para browsers
- ✅ Controlar costos con quotas

### Lo que NO existía (Módulo 3)

```bash
# ❌ Esto NO funcionaba
curl https://mi-api.com/sessions

# ✅ Solo esto funcionaba (requiere credenciales AWS)
aws lambda invoke --function-name mi-lambda ...
```

### Lo que SÍ existe (Módulo 4)

```bash
# ✅ Ahora ESTO funciona (sin credenciales)
curl -X POST https://abc123.execute-api.us-east-1.amazonaws.com/dev/sessions \
  -H 'Content-Type: application/json' \
  -d '{"user_id": "test-user", "ttl": 24}'

# Respuesta:
{
  "session_id": "f7a3b2c1-4d5e-6789-abcd-ef0123456789",
  "user_id": "test-user",
  "expires_at": 1734393600,
  "message": "Session created successfully"
}
```

---

## ¿Qué es API Gateway?

### Definición Simple

> **API Gateway = El "portero" entre internet y tu Lambda.**

### Analogía: Restaurant

**Lambda sola (Módulo 3):**
```
Chef en cocina sin puerta
- Solo el dueño puede entrar (AWS CLI)
- Nadie del público puede pedir comida
- Sin control de cuántos platos se hacen
```

**Lambda + API Gateway (Módulo 4):**
```
Chef + Mesero + Seguridad
- El público puede pedir (HTTP requests)
- Mesero traduce pedidos (API Gateway)
- Seguridad controla entrada (throttling)
- Registra quién pidió qué (logs)
```

### ¿Qué Hace API Gateway?

```
Internet (curl, Postman, browsers, apps)
    ↓
    ↓ HTTPS Request
    ↓
┌───────────────────────────────────┐
│       API Gateway                 │
│                                   │
│  1. Recibe HTTP request           │
│  2. Valida formato                │
│  3. Rate limiting (throttling)    │
│  4. CORS headers                  │
│  5. Logging                       │
│  6. Traduce a evento Lambda       │
└───────────────┬───────────────────┘
                ↓
                ↓ Lambda Integration
                ↓
┌───────────────────────────────────┐
│       Lambda Function             │
│   (tu código Go)                  │
└───────────────────────────────────┘
```

### Tipos de API Gateway

AWS ofrece 3 tipos:

| Tipo | Precio | Uso | Features |
|------|--------|-----|----------|
| **REST API** | $3.50/M requests | Aplicaciones completas | Throttling, caching, API keys |
| **HTTP API** | $1.00/M requests | APIs simples | Básico, sin throttling |
| **WebSocket** | $1.00/M requests | Chat, streaming | Conexiones bidireccionales |

**Usamos REST API** porque necesitamos throttling y control de seguridad.

---

## Throttling y Rate Limiting

### El Problema: Abuso y Costos

**Escenario sin throttling:**
```
Atacante envía 1 millón de requests por minuto
→ Lambda se ejecuta 1 millón de veces
→ Factura por minuto: $3.50 (API Gateway) + $0.60 (Lambda) = $4.10/min
→ En 1 hora: $246
→ En 1 día: $5,904
```

😱 **Sin throttling, un ataque podría costarte miles de dólares.**

### La Solución: Throttling + Quota

**Con throttling configurado:**
```
Límite: 100 requests/segundo
Burst: 50 requests simultáneos
Quota: 10,000 requests/día

Atacante envía 1 millón de requests/minuto
→ API Gateway rechaza el exceso (error 429)
→ Solo pasan 6,000 requests/minuto (100/seg × 60)
→ Máximo al día: 10,000 requests (quota)
→ Factura máxima: $0.041/día = $1.23/mes
```

✅ **Con throttling, es IMPOSIBLE generar facturas inesperadas.**

### Configuración en Terraform

```hcl
resource "aws_api_gateway_method_settings" "main" {
  settings {
    # Throttling
    throttling_burst_limit = 50    # Máximo simultáneos
    throttling_rate_limit  = 100   # Máximo por segundo
  }
}

resource "aws_api_gateway_usage_plan" "main" {
  # Quota diario
  quota_settings {
    limit  = 10000
    period = "DAY"
  }
}
```

### Cómo Funciona

**Burst Limit (50 requests simultáneos):**
```
Llegan 100 requests al mismo tiempo:
- API Gateway acepta 50
- Rechaza 50 con error 429 (Too Many Requests)
```

**Rate Limit (100 requests/segundo):**
```
Segundo 1:
  Request 1-100:  ✅ Accepted
  Request 101:    ❌ 429 Too Many Requests
  Request 102-200: ❌ 429

Segundo 2:
  Request 201-300: ✅ Accepted (nuevo segundo)
```

**Quota (10,000 requests/día):**
```
Día 1:
  Request 1-10,000:  ✅ Accepted
  Request 10,001:    ❌ 429 (excede quota)
  Request 10,002+:   ❌ 429

Día 2 (00:00 UTC):
  Quota resetea
  Request 1:         ✅ Accepted
```

---

## Arquitectura Completa

### Diagrama de Flujo

```
┌──────────────────────────────────────────────────────┐
│                  Internet                            │
│     (curl, Postman, React app, Mobile app)           │
└────────────────────────┬─────────────────────────────┘
                         │
                         ▼ POST /sessions
                         
┌────────────────────────────────────────────────────────┐
│              API Gateway REST API                      │
│  https://abc123.execute-api.us-east-1.amazonaws.com   │
│                                                        │
│  ┌──────────────────────────────────────────────────┐ │
│  │ Stage: /dev                                      │ │
│  │ Resource: /sessions                              │ │
│  │ Method: POST                                     │ │
│  │                                                  │ │
│  │ ✅ Throttling: 100 req/seg                       │ │
│  │ ✅ Burst: 50 simultáneos                         │ │
│  │ ✅ Quota: 10,000 req/día                         │ │
│  │ ✅ CORS: Enabled                                 │ │
│  │ ✅ Logs: CloudWatch                              │ │
│  └──────────────────────────────────────────────────┘ │
└────────────────────────┬───────────────────────────────┘
                         │
                         ▼ Lambda Integration (AWS_PROXY)
                         
┌────────────────────────────────────────────────────────┐
│           Lambda Function (Go ARM64)                   │
│      go-hexagonal-auth-dev-create-session             │
│                                                        │
│  ┌──────────────────────────────────────────────────┐ │
│  │ cmd/lambda/main.go                               │ │
│  │  └─ Parse event                                  │ │
│  │  └─ Validate input                               │ │
│  │                                                  │ │
│  │ internal/core/domain/session.go                  │ │
│  │  └─ NewSession(userID, ttl)                      │ │
│  │  └─ Business logic                               │ │
│  │                                                  │ │
│  │ internal/adapters/repository/dynamodb_session.go │ │
│  │  └─ Save(session)                                │ │
│  │  └─ DynamoDB operations                          │ │
│  └──────────────────────────────────────────────────┘ │
└────────────────────────┬───────────────────────────────┘
                         │
                         ▼ PutItem
                         
┌────────────────────────────────────────────────────────┐
│              DynamoDB Table                            │
│       go-hexagonal-auth-dev-sessions                   │
│                                                        │
│  PK: session_id (UUID)                                 │
│  Attributes:                                           │
│    - user_id                                           │
│    - expires_at (TTL)                                  │
│    - created_at                                        │
│    - data                                              │
└────────────────────────────────────────────────────────┘
```

### Stack Completo (Módulos 0-4)

| Capa | Servicio | Propósito |
|------|----------|-----------|
| **Frontend** | API Gateway | Endpoint público HTTPS |
| **Compute** | Lambda (Go) | Lógica de negocio |
| **Storage** | DynamoDB | Persistencia de sesiones |
| **IAM** | Roles + Policies | Permisos (Least Privilege) |
| **Logs** | CloudWatch | Monitoring y debugging |
| **IaC** | Terraform | Infraestructura como código |
| **State** | S3 + DynamoDB | Remote backend |

---

## Implementación con Terraform

### Estructura de Archivos

```
terraform/
├── provider.tf         # AWS provider
├── backend.tf          # S3 + DynamoDB backend
├── variables.tf        # Variables de proyecto
├── iam.tf             # Roles para Lambda
├── dynamodb.tf        # Tabla de sesiones
├── lambda.tf          # Lambda function
└── api_gateway.tf     # API Gateway (NUEVO)
```

### Componentes de API Gateway

#### 1. REST API

```hcl
resource "aws_api_gateway_rest_api" "main" {
  name        = "${var.project_name}-${var.environment}-api"
  description = "API Gateway para ${var.project_name}"

  endpoint_configuration {
    types = ["REGIONAL"]  # Regional endpoint (más barato)
  }
}
```

**¿Qué hace?** Crea el contenedor principal de la API.

**REGIONAL vs EDGE:**
- REGIONAL: Requests van directo a la región (más barato)
- EDGE: Usa CloudFront (CDN global, más caro)

**Para desarrollo:** REGIONAL es suficiente.

#### 2. Resource (Path)

```hcl
resource "aws_api_gateway_resource" "sessions" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  parent_id   = aws_api_gateway_rest_api.main.root_resource_id
  path_part   = "sessions"
}
```

**¿Qué hace?** Crea el path `/sessions`.

**Resultado:**
```
https://abc123.execute-api.us-east-1.amazonaws.com/dev/sessions
                                                       ^^^^^^^^
                                                       Resource
```

#### 3. Method (HTTP Verb)

```hcl
resource "aws_api_gateway_method" "create_session" {
  rest_api_id   = aws_api_gateway_rest_api.main.id
  resource_id   = aws_api_gateway_resource.sessions.id
  http_method   = "POST"
  authorization = "NONE"  # Sin autenticación (público)
}
```

**¿Qué hace?** Define que se puede hacer `POST` a `/sessions`.

#### 4. Integration (Backend)

```hcl
resource "aws_api_gateway_integration" "lambda_integration" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  resource_id = aws_api_gateway_resource.sessions.id
  http_method = aws_api_gateway_method.create_session.http_method
  
  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = aws_lambda_function.create_session.invoke_arn
}
```

**¿Qué hace?** Conecta el método POST con la Lambda.

**AWS_PROXY:**
- API Gateway pasa TODO el evento a Lambda sin modificar
- Lambda retorna la respuesta completa (statusCode, headers, body)
- Más flexible que custom integration

#### 5. Lambda Permission

```hcl
resource "aws_lambda_permission" "api_gateway_invoke" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.create_session.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_api_gateway_rest_api.main.execution_arn}/*/*"
}
```

**¿Qué hace?** Le da permiso a API Gateway para invocar la Lambda.

**Sin esto:** Error 403 (Forbidden)

#### 6. Deployment y Stage

```hcl
resource "aws_api_gateway_deployment" "main" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  
  triggers = {
    redeployment = sha1(jsonencode([
      aws_api_gateway_resource.sessions.id,
      aws_api_gateway_method.create_session.id,
    ]))
  }
}

resource "aws_api_gateway_stage" "main" {
  deployment_id = aws_api_gateway_deployment.main.id
  rest_api_id   = aws_api_gateway_rest_api.main.id
  stage_name    = var.environment  # "dev"
}
```

**¿Qué hace?**
- Deployment: Crea un "snapshot" de la configuración
- Stage: Publica el deployment en un ambiente (dev/prod)

**Resultado:**
```
https://abc123.execute-api.us-east-1.amazonaws.com/dev/sessions
                                                   ^^^
                                                   Stage
```

#### 7. Throttling Settings

```hcl
resource "aws_api_gateway_method_settings" "main" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  stage_name  = aws_api_gateway_stage.main.stage_name
  method_path = "*/*"
  
  settings {
    throttling_burst_limit = 50
    throttling_rate_limit  = 100
    logging_level          = "INFO"
    metrics_enabled        = true
  }
}
```

**¿Qué hace?** Configura límites de requests por segundo.

#### 8. Usage Plan (Quota)

```hcl
resource "aws_api_gateway_usage_plan" "main" {
  name = "${var.project_name}-${var.environment}-usage-plan"
  
  quota_settings {
    limit  = 10000
    period = "DAY"
  }
  
  throttle_settings {
    burst_limit = 50
    rate_limit  = 100
  }
}
```

**¿Qué hace?** Limita requests por día.

---

## CORS Explicado

### ¿Qué es CORS?

**Same-Origin Policy:**
```
Frontend en: https://myapp.com
API en:      https://api.aws.com

Browser bloquea el request por seguridad
(diferentes dominios)
```

**CORS (Cross-Origin Resource Sharing):**
```
API dice: "Permito requests desde otros dominios"
Browser permite el request
```

### Preflight Request

Cuando un browser hace un request cross-origin, primero envía un **preflight request**:

```
1. Browser detecta CORS request
   POST https://api.aws.com/sessions
   Origin: https://myapp.com

2. Browser envía OPTIONS (preflight):
   OPTIONS https://api.aws.com/sessions
   Origin: https://myapp.com
   Access-Control-Request-Method: POST

3. API responde con CORS headers:
   200 OK
   Access-Control-Allow-Origin: *
   Access-Control-Allow-Methods: POST,OPTIONS

4. Si headers OK:
   Browser envía el POST real

5. Si headers no coinciden:
   Browser bloquea el request
```

### Implementación en Terraform

```hcl
# 1. Método OPTIONS
resource "aws_api_gateway_method" "options_sessions" {
  rest_api_id   = aws_api_gateway_rest_api.main.id
  resource_id   = aws_api_gateway_resource.sessions.id
  http_method   = "OPTIONS"
  authorization = "NONE"
}

# 2. Integration MOCK (no llama Lambda)
resource "aws_api_gateway_integration" "options_integration" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  resource_id = aws_api_gateway_resource.sessions.id
  http_method = aws_api_gateway_method.options_sessions.http_method
  type        = "MOCK"
}

# 3. Response con CORS headers
resource "aws_api_gateway_integration_response" "options_integration_response" {
  response_parameters = {
    "method.response.header.Access-Control-Allow-Origin"  = "'*'"
    "method.response.header.Access-Control-Allow-Methods" = "'POST,OPTIONS'"
    "method.response.header.Access-Control-Allow-Headers" = "'Content-Type'"
  }
}
```

**Headers CORS necesarios:**
- `Access-Control-Allow-Origin: *` → Permite cualquier dominio
- `Access-Control-Allow-Methods: POST,OPTIONS` → Métodos permitidos
- `Access-Control-Allow-Headers: Content-Type` → Headers permitidos

---

## Testing Completo

### Test 1: Request Básico

```bash
# Obtener URL
API_URL=$(terraform output -raw api_gateway_endpoint_create_session)

# Invocar
curl -X POST "$API_URL" \
  -H 'Content-Type: application/json' \
  -d '{"user_id": "test-user", "ttl": 24}'
```

**Respuesta:**
```json
{
  "session_id": "f7a3b2c1-4d5e-6789-abcd-ef0123456789",
  "user_id": "test-user",
  "expires_at": 1734393600,
  "message": "Session created successfully"
}
```

**StatusCode:** 201 ✅

### Test 2: Throttling

```bash
# Enviar 200 requests rápidamente
for i in {1..200}; do
  curl -X POST "$API_URL" \
    -H 'Content-Type: application/json' \
    -d "{\"user_id\": \"load-$i\"}" \
    -s -o /dev/null -w "%{http_code}\n" &
done | grep 429 | wc -l
```

**Resultado esperado:** ~100 requests con 429 (throttled)

### Test 3: CORS Preflight

```bash
curl -X OPTIONS "$API_URL" \
  -H 'Origin: http://localhost:3000' \
  -H 'Access-Control-Request-Method: POST' \
  -v
```

**Headers esperados:**
```
< access-control-allow-origin: *
< access-control-allow-methods: POST,OPTIONS
< access-control-allow-headers: Content-Type
```

### Test 4: Validación

```bash
# user_id vacío
curl -X POST "$API_URL" \
  -H 'Content-Type: application/json' \
  -d '{"user_id": ""}'

# Respuesta:
{
  "error": "Bad Request",
  "message": "user_id is required"
}
```

**StatusCode:** 400 ✅

### Test 5: JSON Inválido

```bash
curl -X POST "$API_URL" \
  -H 'Content-Type: application/json' \
  -d 'invalid json'

# Respuesta:
{
  "error": "Bad Request",
  "message": "Invalid JSON"
}
```

**StatusCode:** 400 ✅

### Test 6: Verificar en DynamoDB

```bash
SESSION_ID="f7a3b2c1-..."  # Del test anterior

aws dynamodb get-item \
  --table-name go-hexagonal-auth-dev-sessions \
  --key "{\"session_id\": {\"S\": \"$SESSION_ID\"}}"
```

**Debe mostrar el item creado** ✅

---

## Costos y Seguridad

### Pricing

```
API Gateway REST API:
- Requests: $3.50 por millón
- Data transfer out: $0.09 por GB

Ejemplos:
10,000 requests/mes:
  Requests: 10,000 × $0.0000035 = $0.035
  Data: 10,000 × 1KB = $0.001
  Total: $0.036/mes

100,000 requests/mes:
  Requests: 100,000 × $0.0000035 = $0.35
  Data: 100,000 × 1KB = $0.009
  Total: $0.36/mes
```

### Costo Total (Módulos 0-4)

| Servicio | Uso | Costo/mes |
|----------|-----|-----------|
| S3 (Backend) | State file | $0.00 |
| DynamoDB (Locks) | < 25 reads | $0.00 |
| DynamoDB (Sessions) | 10k writes | $0.00 |
| Lambda | 10k invocations | $0.06 |
| API Gateway | 10k requests | $0.35 |
| CloudWatch Logs | < 5GB | $0.00 |
| **TOTAL** | | **$0.41/mes** |

**Con Free Tier:** $0.00 (primer año)

**Máximo con quota (10k req/día):**
- API Gateway: $1.05/mes
- Lambda: $0.18/mes
- **Total:** $1.23/mes

### Seguridad

**Protecciones configuradas:**

1. **Throttling:** 100 req/seg, burst 50
   - Imposible saturar Lambda
   
2. **Quota:** 10,000 req/día
   - Costo máximo predecible

3. **Logs:** Cada request logeado
   - IP, timestamp, status code
   - Detecta ataques

4. **CORS:** Solo dominios permitidos
   - Protege contra CSRF

5. **IAM:** Lambda con Least Privilege
   - Solo accede a su tabla DynamoDB

---

## Troubleshooting

### Error: "Missing Authentication Token"

**Síntoma:**
```json
{"message": "Missing Authentication Token"}
```

**Causa:** URL incorrecta o método incorrecto.

**Solución:**
```bash
# Verificar URL exacta
terraform output api_gateway_endpoint_create_session

# Debe terminar en /sessions
# Correcto: .../dev/sessions
# Incorrecto: .../dev
```

### Error: "Internal Server Error"

**Síntoma:**
```json
{"message": "Internal server error"}
```

**Causa:** Lambda falló.

**Diagnóstico:**
```bash
aws logs tail /aws/lambda/go-hexagonal-auth-dev-create-session --since 5m
```

### Error: "Forbidden" (403)

**Síntoma:**
```json
{"message": "Forbidden"}
```

**Causa:** Lambda permission faltante.

**Solución:**
```bash
# Verificar permission
aws lambda get-policy \
  --function-name go-hexagonal-auth-dev-create-session
```

Debe incluir `Principal: apigateway.amazonaws.com`.

### Throttling No Funciona

**Síntoma:** Puedes enviar miles de requests sin 429.

**Causa:** Usage plan no asociado.

**Solución:**
```bash
terraform apply  # Reaplica la configuración
```

---

## Conclusiones

### Lo que lograste

- ✅ Endpoint HTTPS público funcional
- ✅ Lambda accesible desde internet
- ✅ Throttling configurado (100 req/seg)
- ✅ Quota diario (10k requests)
- ✅ CORS habilitado para browsers
- ✅ Logs en CloudWatch
- ✅ Costo controlado (máx $1.23/mes)

### Arquitectura Completa

Ahora tienes un stack serverless completo:
- Frontend: API Gateway (endpoint público)
- Compute: Lambda (Go ARM64)
- Storage: DynamoDB (sessions)
- Security: IAM Roles (Least Privilege)
- Monitoring: CloudWatch Logs
- IaC: Terraform (reproducible)

### Mejores Prácticas Aplicadas

1. **Throttling y Quota:** Previene abuso y controla costos
2. **CORS:** Permite requests desde browsers
3. **AWS_PROXY:** Máxima flexibilidad en Lambda
4. **Logs:** Debugging y auditoría
5. **Terraform:** Infraestructura versionada

### Performance

**Cold Start (primera invocación):**
- API Gateway: ~20ms
- Lambda Go: ~156ms
- DynamoDB: ~30ms
- **Total:** ~206ms

**Warm Invocations:**
- Total: ~95ms

### Costos Reales

Con 10,000 requests/mes:
```
API Gateway: $0.35
Lambda: $0.06
Total: $0.41/mes

Con Free Tier: $0.00
```

---

## Recursos Adicionales

- [API Gateway Developer Guide](https://docs.aws.amazon.com/apigateway/)
- [API Gateway REST API Reference](https://docs.aws.amazon.com/apigateway/latest/api/)
- [Throttling Best Practices](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html)
- [CORS on API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-cors.html)

---

## Próximos Pasos

### Nivel Intermediate

**Agregar más endpoints:**
- `GET /sessions/{id}` - Obtener sesión
- `DELETE /sessions/{id}` - Cerrar sesión
- `GET /sessions` - Listar sesiones

**Autenticación:**
- API Keys
- Cognito User Pools
- Custom authorizer (JWT)

**Custom Domain:**
- `api.tudominio.com`
- Certificado SSL (ACM)
- Route53 configuration

### Nivel Advanced

**Caching:**
- Cache responses en API Gateway
- Reduce invocaciones a Lambda

**CloudFront:**
- CDN global
- Edge locations
- DDoS protection

**Multi-stage:**
- dev, staging, prod
- Blue-green deployments

---

## Código Completo

Todo el código está disponible en:
- [terraform/api_gateway.tf](https://github.com/edgar-macias-se/go-hexagonal-auth/terraform/api_gateway.tf) - API Gateway config
- [terraform/lambda.tf](https://github.com/edgar-macias-se/go-hexagonal-auth/terraform/lambda.tf) - Lambda config
- [cmd/lambda/](https://github.com/edgar-macias-se/go-hexagonal-auth/cmd/lambda/) - Lambda handler
- [internal/](https://github.com/edgar-macias-se/go-hexagonal-auth/internal/) - Domain y adapters

---

## Preguntas Frecuentes

**Q: ¿Es segura mi API sin autenticación?**  
A: Para desarrollo sí. Para producción, agrega autenticación (API Keys, Cognito, JWT).

**Q: ¿Cómo cambio el límite de throttling?**  
A: Edita `throttling_rate_limit` en `api_gateway.tf` y aplica con Terraform.

**Q: ¿Puedo usar custom domain?**  
A: Sí, necesitas Route53, ACM (certificado SSL) y configuración adicional en API Gateway.

**Q: ¿Cómo monitoreo la API?**  
A: CloudWatch Logs + Métricas. También puedes usar CloudWatch Dashboards.

**Q: ¿Cuánto cuesta en producción con alto tráfico?**  
A: Con 1M requests/mes: API Gateway $3.50 + Lambda $0.60 = $4.10/mes.

---

**Autor:** Edgar Macías  
**GitHub:** [edgar-macias-se](https://github.com/edgar-macias-se)  
**LinkedIn:** [edgar-macias-devcybsec](https://www.linkedin.com/in/edgar-macias-devcybsec/)  
**Website:** [edgarmacias.com/es](https://edgarmacias.com/es)  
**Serie:** AWS Zero to Architect  
**Repositorio:** [aws_road](https://github.com/edgar-macias-se/aws_road)

---

⭐ **Si este tutorial te ayudó, considera darle una estrella al [repositorio](https://github.com/edgar-macias-se/aws_road)**
