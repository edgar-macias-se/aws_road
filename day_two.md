# IAM Roles & DynamoDB: Seguridad con Least Privilege (Módulo 2)

> **Serie:** AWS Zero to Architect  
> **Nivel:** Intermedio  
> **Tiempo:** 90 minutos  
> **Prerequisitos:** [Módulo 0 - Setup Seguro](./zero_day.md) y [Módulo 1 - Terraform Backend](./first_day.md)

---

## 📋 Contenido

1. [Introducción](#introducción)
2. [Conceptos de IAM](#conceptos-de-iam)
3. [Anatomía de una Policy](#anatomía-de-una-policy)
4. [Principio de Least Privilege](#principio-de-least-privilege)
5. [Trust Relationships](#trust-relationships)
6. [DynamoDB Fundamentals](#dynamodb-fundamentals)
7. [Implementación con Terraform](#implementación-con-terraform)
8. [Testing y Validación](#testing-y-validación)
9. [Troubleshooting](#troubleshooting)
10. [Conclusiones](#conclusiones)

---

## Introducción

En los módulos anteriores configuramos la cuenta AWS y el backend de Terraform. Ahora viene **la parte más crítica**: **Identity and Access Management (IAM)**.

### ¿Por qué IAM es crítico?

**Estadísticas reales:**
- 60% de los incidentes de seguridad en AWS son por **permisos mal configurados**
- Costo promedio de un rol con permisos excesivos: **$15,000 - $50,000** en daños
- Tiempo promedio para detectar permisos excesivos: **45 días**

### Lo que aprenderás

- ✅ Diferencia entre Users, Roles y Policies
- ✅ Cómo crear políticas con mínimo privilegio
- ✅ Trust Relationships (quién puede asumir un rol)
- ✅ Configurar DynamoDB para producción
- ✅ Implementar todo con Terraform
- ✅ Validar permisos correctamente

---

## Conceptos de IAM

### La Analogía del Edificio de Oficinas

```
🏢 AWS = Edificio de Oficinas

├── 👤 IAM Users (Empleados con credenciales)
│   └── Username + Password + Access Keys
│   └── Ejemplo: juan-admin
│
├── 👔 IAM Roles (Uniformes con permisos)
│   └── NO tienen credenciales permanentes
│   └── Se "asumen" temporalmente
│   └── Ejemplo: lambda-execution-role
│
└── 📜 IAM Policies (Reglas escritas)
    └── JSON que define: "Puedes hacer X en Y"
    └── Ejemplo: "Leer tabla 'users' en DynamoDB"
```

### IAM User vs IAM Role

| IAM User | IAM Role |
|----------|----------|
| **Persona/Aplicación** permanente | **Servicio AWS** temporal |
| Credenciales de largo plazo | Credenciales temporales (15min-12h) |
| Username + Password/Keys | Asumido por servicios |
| Ejemplo: tu usuario admin | Ejemplo: rol para Lambda |

### Flujo de un Role

```
1. Lambda function necesita escribir en DynamoDB
   ↓
2. Creas un IAM Role: "lambda-dynamodb-writer"
   ↓
3. Agregas Policy: "Puede escribir en tabla X"
   ↓
4. Asignas el Role a la Lambda
   ↓
5. Lambda se ejecuta:
   - Asume el role (credenciales por 15 min)
   - Escribe en DynamoDB
   - Credenciales expiran
```

**Ventaja:** Si comprometen tu Lambda, solo tienen acceso temporal y limitado.

---

## Anatomía de una Policy

### Estructura Básica

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/users"
    }
  ]
}
```

### Componentes Explicados

#### 1. Version
```json
"Version": "2012-10-17"
```
- Versión del lenguaje de políticas
- **SIEMPRE** usa `2012-10-17` (es la actual)
- No es la versión de tu policy, sino del formato

#### 2. Statement
```json
"Statement": [ ... ]
```
- Array de declaraciones (reglas)
- Puede contener múltiples statements

#### 3. Effect
```json
"Effect": "Allow"  // o "Deny"
```
- **Allow:** Permitir la acción
- **Deny:** Denegar explícitamente (tiene prioridad)

**Regla de oro:** Sin "Allow" explícito = denegado por defecto.

#### 4. Action
```json
"Action": [
  "dynamodb:GetItem",
  "dynamodb:PutItem"
]
```
- Formato: `servicio:operación`
- Puede usar wildcards: `dynamodb:*` (todos los permisos)

**Ejemplos comunes:**
```json
"dynamodb:GetItem"      // Leer un item
"dynamodb:PutItem"      // Escribir un item
"dynamodb:Query"        // Query en tabla
"s3:GetObject"          // Leer objeto de S3
"s3:PutObject"          // Escribir objeto en S3
"logs:PutLogEvents"     // Escribir logs
```

#### 5. Resource
```json
"Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/users"
```
- ARN (Amazon Resource Name) del recurso
- Formato: `arn:aws:servicio:region:account:recurso`
- Puede usar wildcards: `arn:aws:s3:::bucket/*`

**Ejemplos de ARNs:**
```
DynamoDB table:
arn:aws:dynamodb:us-east-1:123456789012:table/users

S3 bucket:
arn:aws:s3:::my-bucket

S3 objects:
arn:aws:s3:::my-bucket/*

CloudWatch Logs:
arn:aws:logs:us-east-1:123456789012:log-group:/aws/lambda/my-function:*
```

### Policy Completa (Ejemplo Real)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DynamoDBAccess",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/users"
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:us-east-1:123456789012:log-group:/aws/lambda/*"
    }
  ]
}
```

**Traducción:**
- Statement 1: "Permite CRUD en tabla 'users'"
- Statement 2: "Permite crear y escribir logs"

---

## Principio de Least Privilege

> **"Dale a un servicio SOLO los permisos que necesita, nada más."**

### Comparación: Mal vs Bien

#### ❌ Mal Ejemplo (Permisos Excesivos)

```json
{
  "Effect": "Allow",
  "Action": "dynamodb:*",
  "Resource": "*"
}
```

**Problemas:**
- Lambda puede hacer CUALQUIER COSA en CUALQUIER tabla
- Si la hackean, pueden borrar TODAS las tablas
- Puede leer datos sensibles de otros proyectos
- Puede crear tablas nuevas (costos inesperados)

**Impacto de hackeo:**
```
Atacante:
1. Borra tabla de producción → Data loss
2. Crea 100 tablas on-demand → $1,000/día
3. Lee tabla de pagos → Robo de datos
4. Modifica tabla de usuarios → Backdoor
```

#### ✅ Buen Ejemplo (Least Privilege)

```json
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:GetItem",
    "dynamodb:PutItem"
  ],
  "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/auth-sessions"
}
```

**Beneficios:**
- Solo puede leer y escribir en UNA tabla
- No puede borrar ni modificar estructura
- No puede acceder a otras tablas
- Daño limitado si comprometen la Lambda

**Impacto de hackeo:**
```
Atacante:
1. Solo puede leer/escribir en tabla 'auth-sessions'
2. NO puede borrar la tabla
3. NO puede acceder a otras tablas
4. Daño contenido y reversible
```

### Visualización

```
❌ Permisos Excesivos:
Lambda → "dynamodb:*" en "*"
         ↓
   Acceso a:
   - tabla users ✓
   - tabla orders ✓
   - tabla payments ✓
   - tabla admin-data ✓
   - Crear tablas ✓
   - Borrar tablas ✓

✅ Least Privilege:
Lambda → "GetItem, PutItem" en "table/auth-sessions"
         ↓
   Acceso a:
   - tabla auth-sessions (solo lectura/escritura) ✓
   
   NO puede:
   - Acceder a otras tablas ✗
   - Borrar la tabla ✗
   - Modificar estructura ✗
```

---

## Trust Relationships

### ¿Qué es un Trust Relationship?

Define **QUIÉN** puede asumir un rol.

**Analogía:** El rol es un chaleco con permisos. El trust relationship es la regla de quién puede ponerse ese chaleco.

### Trust Policy Básica

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Traducción:** "Solo el servicio Lambda puede asumir este rol"

### Componentes

**Principal:** QUIÉN puede asumir el rol
- `Service`: Un servicio de AWS
- `AWS`: Un usuario/rol específico

**Action:** `sts:AssumeRole` (la acción de asumir)

### Ejemplos de Trust Relationships

#### Para Lambda
```json
{
  "Principal": {
    "Service": "lambda.amazonaws.com"
  },
  "Action": "sts:AssumeRole"
}
```

#### Para EC2
```json
{
  "Principal": {
    "Service": "ec2.amazonaws.com"
  },
  "Action": "sts:AssumeRole"
}
```

#### Para otro rol AWS
```json
{
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/deployer-role"
  },
  "Action": "sts:AssumeRole"
}
```

### Flujo Completo

```
Paso 1: Trust Relationship
        ↓
¿Lambda está en "Principal"?
        ↓
    SÍ → Puede asumir el rol → Continuar
    NO → ERROR → STOP

Paso 2: Permissions
        ↓
¿Qué permisos tiene el rol?
        ↓
Lambda ejecuta con esos permisos
```

**Sin trust relationship válido:**
```bash
Error: The role defined for the function 
cannot be assumed by Lambda.
```

---

## DynamoDB Fundamentals

### ¿Qué es DynamoDB?

Base de datos NoSQL serverless de AWS:
- **Fully managed:** AWS maneja servidores, escalado, backups
- **Performance:** Latencia < 10ms
- **Escalable:** De 0 a millones de requests/segundo
- **Pay-per-request:** Solo pagas por operaciones

### Conceptos Clave

#### Table (Tabla)
- Colección de items (similar a tabla SQL)
- Ejemplo: `auth-sessions`

#### Item (Item)
- Un registro individual (similar a fila SQL)
- Ejemplo: Una sesión de usuario

#### Attributes (Atributos)
- Campos del item (similar a columnas SQL)
- Ejemplo: `session_id`, `user_id`, `expires_at`

#### Primary Key
- **Partition Key (hash key):** Identificador único
- Ejemplo: `session_id`

### Billing Modes

#### On-Demand (Pay-per-request)
```terraform
billing_mode = "PAY_PER_REQUEST"
```
- Pagas por operación (~$1.25 por millón)
- Ideal para: Dev, staging, apps con tráfico variable
- Sin mínimos, sin capacidad reservada

#### Provisioned
```terraform
billing_mode = "PROVISIONED"
read_capacity_units  = 5
write_capacity_units = 5
```
- Pagas por capacidad reservada
- Más barato para tráfico predecible y constante
- Ideal para: Producción con tráfico estable

### TTL (Time to Live)

Auto-elimina items expirados (gratis):

```terraform
ttl {
  attribute_name = "expires_at"
  enabled        = true
}
```

**Uso:** Sesiones, cachés, datos temporales

**Ejemplo:**
```json
{
  "session_id": "abc-123",
  "expires_at": 1735689600  // Unix timestamp
}
```

DynamoDB borrará automáticamente este item después de esa fecha.

---

## Implementación con Terraform

### Estructura del Proyecto

```
terraform/
├── backend.tf          (Módulo 1)
├── provider.tf         (NUEVO)
├── variables.tf        (NUEVO)
├── iam.tf              (NUEVO - roles y policies)
├── dynamodb.tf         (NUEVO - tabla)
└── outputs.tf          (NUEVO)
```

### Código Completo

#### `terraform/variables.tf`

```terraform
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Nombre del proyecto"
  type        = string
  default     = "go-hexagonal-auth"
}

variable "environment" {
  description = "Ambiente (dev, staging, prod)"
  type        = string
  default     = "dev"
}

variable "aws_account_id" {
  description = "AWS Account ID"
  type        = string
}

locals {
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

#### `terraform/iam.tf`

```terraform
# IAM Role para Lambda
resource "aws_iam_role" "lambda_execution" {
  name = "${var.project_name}-${var.environment}-lambda-role"

  # Trust Relationship
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })

  tags = local.common_tags
}

# Policy para DynamoDB (Least Privilege)
resource "aws_iam_policy" "lambda_dynamodb" {
  name = "${var.project_name}-${var.environment}-lambda-dynamodb"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:UpdateItem",
          "dynamodb:DeleteItem",
          "dynamodb:Query"
        ]
        Resource = aws_dynamodb_table.auth_sessions.arn
      }
    ]
  })
}

# Policy para CloudWatch Logs
resource "aws_iam_policy" "lambda_logging" {
  name = "${var.project_name}-${var.environment}-lambda-logging"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:${var.aws_region}:${var.aws_account_id}:log-group:/aws/lambda/${var.project_name}-*:*"
      }
    ]
  })
}

# Attach policies al rol
resource "aws_iam_role_policy_attachment" "lambda_dynamodb" {
  role       = aws_iam_role.lambda_execution.name
  policy_arn = aws_iam_policy.lambda_dynamodb.arn
}

resource "aws_iam_role_policy_attachment" "lambda_logging" {
  role       = aws_iam_role.lambda_execution.name
  policy_arn = aws_iam_policy.lambda_logging.arn
}
```

#### `terraform/dynamodb.tf`

```terraform
resource "aws_dynamodb_table" "auth_sessions" {
  name         = "${var.project_name}-${var.environment}-sessions"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "session_id"

  attribute {
    name = "session_id"
    type = "S"  # String
  }

  # TTL para auto-eliminar sesiones expiradas
  ttl {
    attribute_name = "expires_at"
    enabled        = true
  }

  # Encryption (habilitada por defecto)
  server_side_encryption {
    enabled = true
  }

  # Point-in-time recovery (solo prod)
  point_in_time_recovery {
    enabled = var.environment == "prod"
  }

  tags = local.common_tags
}
```

### Ejecución

```bash
# 1. Exportar variables
export TF_VAR_aws_account_id=$(aws sts get-caller-identity --query Account --output text)
export TF_VAR_environment="dev"

# 2. Inicializar
cd terraform
terraform init

# 3. Plan
terraform plan

# 4. Apply
terraform apply
```

**Recursos creados:** 6
- 1 IAM Role
- 2 IAM Policies
- 2 Policy Attachments
- 1 DynamoDB Table

---

## Testing y Validación

### Test 1: Verificar Recursos

```bash
# Ver outputs
terraform output

# Describir tabla
aws dynamodb describe-table \
  --table-name go-hexagonal-auth-dev-sessions

# Verificar rol
aws iam get-role \
  --role-name go-hexagonal-auth-dev-lambda-role

# Listar policies del rol
aws iam list-attached-role-policies \
  --role-name go-hexagonal-auth-dev-lambda-role
```

### Test 2: Insertar Datos

```bash
# Insertar sesión
aws dynamodb put-item \
  --table-name go-hexagonal-auth-dev-sessions \
  --item '{
    "session_id": {"S": "test-123"},
    "user_id": {"S": "user-456"},
    "created_at": {"S": "2024-12-15T10:00:00Z"}
  }'

# Leer sesión
aws dynamodb get-item \
  --table-name go-hexagonal-auth-dev-sessions \
  --key '{"session_id": {"S": "test-123"}}'
```

### Test 3: Validar Trust Relationship

```bash
# Ver trust policy del rol
aws iam get-role \
  --role-name go-hexagonal-auth-dev-lambda-role \
  --query 'Role.AssumeRolePolicyDocument'
```

**Debe mostrar:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "lambda.amazonaws.com"
    },
    "Action": "sts:AssumeRole"
  }]
}
```

---

## Troubleshooting

### Problema: Tabla no aparece en Console

**Causa:** Console en región incorrecta

**Solución:**
```bash
# Verificar región de la tabla
aws dynamodb describe-table \
  --table-name go-hexagonal-auth-dev-sessions \
  --query 'Table.TableArn'

# Cambiar región en console (arriba derecha)
# Seleccionar: US East (N. Virginia) = us-east-1
```

### Problema: Put-Item falla silenciosamente

**Síntoma:** Comando no da error pero scan está vacío

**Causa:** Problema con formato JSON del archivo

**Solución:** Usar sintaxis inline
```bash
aws dynamodb put-item \
  --table-name go-hexagonal-auth-dev-sessions \
  --item '{"session_id": {"S": "test-456"}}'
```

### Problema: Error "EntityAlreadyExists"

**Causa:** Rol ya existe de apply anterior

**Solución:**
```bash
# Opción 1: Destruir y recrear
terraform destroy -target=aws_iam_role.lambda_execution
terraform apply

# Opción 2: Importar
terraform import aws_iam_role.lambda_execution nombre-del-rol
```

---

## Conclusiones

### Lo que lograste

- ✅ **IAM Role** para Lambda con trust relationship
- ✅ **Policies** con least privilege (DynamoDB + Logs)
- ✅ **DynamoDB Table** configurada para producción
- ✅ **TTL** habilitado para auto-limpieza
- ✅ **Encryption** por defecto
- ✅ Todo versionado en Git con Terraform

### Mejores Prácticas Aplicadas

1. **Least Privilege:** Solo permisos necesarios
2. **Trust Relationships:** Solo Lambda puede asumir el rol
3. **Resource-specific:** Policies limitadas a recursos específicos
4. **Infrastructure as Code:** Todo en Terraform
5. **Tags:** Recursos organizados y rastreables

### Costos

| Recurso | Costo Mensual |
|---------|---------------|
| IAM Role + Policies | $0.00 (gratis) |
| DynamoDB (sin uso) | $0.00 |
| DynamoDB (1M ops/mes) | ~$0.25 |
| **Total** | **< $0.50** |

### Seguridad

**Permisos del rol Lambda:**
- ✅ Leer/escribir en tabla `auth-sessions`
- ✅ Escribir logs en CloudWatch
- ❌ NO puede borrar la tabla
- ❌ NO puede acceder a otras tablas
- ❌ NO puede modificar IAM

**Resultado:** Máxima seguridad, mínimo privilegio.

---

## Recursos Adicionales

- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [DynamoDB Developer Guide](https://docs.aws.amazon.com/dynamodb/latest/developerguide/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [IAM Policy Simulator](https://policysim.aws.amazon.com/)

---

## Próximos Pasos

En el **Módulo 3** crearemos:
- Lambda Function con Go
- Compilar para AWS Lambda (ARM64)
- Deploy con Terraform
- API Gateway para exponer la Lambda
- Testing end-to-end

---

## Código Completo

Todo el código está disponible en:
- [terraform/](https://github.com/edgar-macias-se/go-hexagonal-auth/tree/module-2/terraform) - Configuración de Terraform
- [docs/aws/](https://github.com/edgar-macias-se/aws_road) - Documentación completa

---

## Preguntas Frecuentes

**Q: ¿Puedo usar el mismo rol para múltiples Lambdas?**  
A: Sí, pero no es recomendado. Cada Lambda debería tener su propio rol con permisos específicos.

**Q: ¿Qué pasa si borro un item con TTL antes de que expire?**  
A: Puedes borrarlo manualmente. TTL solo borra automáticamente DESPUÉS de expiración.

**Q: ¿On-Demand o Provisioned para producción?**  
A: On-Demand para tráfico variable. Provisioned si tienes tráfico predecible y constante (más económico).

**Q: ¿Cómo roto las credenciales del rol?**  
A: Las credenciales son temporales (15 min). AWS las rota automáticamente.

---

**Autor:** Edgar (Homz) Macías  
**GitHub:** [edgar-macias-se](https://github.com/edgar-macias-se)  
**LinkedIn:** [edgar-macias-devcybsec](https://www.linkedin.com/in/edgar-macias-devcybsec/)  
**Website:** [edgarmacias.com/es](https://edgarmacias.com/es)  
**Serie:** AWS Zero to Architect  
**Repositorio:** [aws_road](https://github.com/edgar-macias-se/aws_road)

---

⭐ **Si este tutorial te ayudó, considera darle una estrella al [repositorio](https://github.com/edgar-macias-se/aws_road)**
