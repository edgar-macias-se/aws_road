# Terraform & Remote Backend: Infraestructura Segura desde Cero (Módulo 1)

> **Serie:** AWS Zero to Architect  
> **Nivel:** Principiante  
> **Tiempo:** 60 minutos  
> **Prerequisitos:** [Módulo 0 - Setup Seguro de Cuenta AWS](./00-setup-seguro.md)

---

## 📋 Contenido

1. [Introducción](#introducción)
2. [¿Qué es Infrastructure as Code?](#qué-es-infrastructure-as-code)
3. [El Peligro del terraform.tfstate](#el-peligro-del-terraformtfstate)
4. [Remote Backend: La Solución](#remote-backend-la-solución)
5. [Implementación Paso a Paso](#implementación-paso-a-paso)
6. [Migración del State](#migración-del-state)
7. [Verificación y Testing](#verificación-y-testing)
8. [Conclusiones](#conclusiones)

---

## Introducción

En el [Módulo 0](https://github.com/edgar-macias-se/aws_road/blob/main/zero_day.md) aseguramos nuestra cuenta AWS. Ahora viene la pregunta: **¿cómo creamos infraestructura de forma profesional, reproducible y segura?**

La respuesta: **Infrastructure as Code (IaC) con Terraform**.

### Lo que aprenderás

- ✅ Diferencia entre ClickOps e Infrastructure as Code
- ✅ Por qué Terraform sobre otras herramientas
- ✅ El riesgo crítico del archivo `terraform.tfstate`
- ✅ Cómo crear un Remote Backend (S3 + DynamoDB)
- ✅ Migrar el state de forma segura
- ✅ Implementar locking para trabajo en equipo

### ⚠️ Advertencia de Seguridad

Este módulo es **crítico** para la seguridad de tu proyecto. Un error al manejar el state file puede exponer credenciales y recursos de AWS. **No saltarse ningún paso.**

---

## ¿Qué es Infrastructure as Code?

### El Problema: ClickOps

Imagina que necesitas crear un bucket S3 para producción:

**Enfoque tradicional (ClickOps):**
```
1. AWS Console → S3 → Create bucket
2. Clic, clic, clic... (15 pasos)
3. Configurar permisos manualmente
4. Habilitar versionado (¿o se te olvidó?)
5. Configurar CORS (¿cuáles eran los valores?)

2 semanas después...
Desarrollador: "¿Qué configuración usamos?"
Respuesta: "No sé, creo que..."
```

**Problemas:**
- ❌ No reproducible (cada quien hace clics diferentes)
- ❌ Sin historial (no sabes quién cambió qué)
- ❌ Errores humanos (olvidar configuraciones)
- ❌ Sin documentación (la configuración solo existe en AWS)

### La Solución: Infrastructure as Code

**Con Terraform:**

```terraform
resource "aws_s3_bucket" "app_storage" {
  bucket = "my-app-prod-bucket"
  
  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

resource "aws_s3_bucket_versioning" "app_storage" {
  bucket = aws_s3_bucket.app_storage.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_cors_configuration" "app_storage" {
  bucket = aws_s3_bucket.app_storage.id
  
  cors_rule {
    allowed_methods = ["GET", "POST"]
    allowed_origins = ["https://myapp.com"]
  }
}
```

**Beneficios:**
- ✅ **Reproducible:** El mismo código crea la misma infraestructura siempre
- ✅ **Versionable:** En Git, con historial completo (`git log`, `git diff`)
- ✅ **Auditable:** Sabes exactamente quién cambió qué y cuándo
- ✅ **Documentado:** El código ES la documentación
- ✅ **Testeable:** Puedes probar cambios en staging antes de producción

### Analogía con Desarrollo de Software

| Sin IaC (ClickOps) | Con IaC (Terraform) |
|-------------------|---------------------|
| Instrucciones verbales | Código fuente |
| Copiar manualmente binarios | `make build` |
| Sin control de versiones | Git con historial |
| "En mi máquina funciona" | Reproducible en cualquier ambiente |

**IaC es como un Makefile para tu infraestructura.**

---

## El Peligro del terraform.tfstate

### ¿Qué es el State File?

Cuando ejecutas Terraform, crea un archivo JSON llamado `terraform.tfstate`:

```json
{
  "version": 4,
  "terraform_version": "1.6.0",
  "resources": [
    {
      "type": "aws_s3_bucket",
      "name": "app_storage",
      "instances": [{
        "attributes": {
          "id": "my-app-prod-bucket",
          "arn": "arn:aws:s3:::my-app-prod-bucket"
        }
      }]
    },
    {
      "type": "aws_db_instance",
      "name": "main",
      "instances": [{
        "attributes": {
          "endpoint": "prod-db.abc123.us-east-1.rds.amazonaws.com",
          "username": "admin",
          "password": "SUPER_SECRET_PASSWORD"  // ⚠️⚠️⚠️
        }
      }]
    }
  ]
}
```

### 🔥 El Problema Crítico

El state file contiene:
- ❌ **Contraseñas en texto plano**
- ❌ **Access Keys** (si las creas con Terraform)
- ❌ **Endpoints de bases de datos**
- ❌ **IDs de todos tus recursos**
- ❌ **Configuraciones de seguridad**

**Si subes `terraform.tfstate` a GitHub público:**

```
Tiempo hasta detección por bots: 5-8 minutos
Tiempo hasta primer ataque: 10-15 minutos
Daño potencial: $1,000 - $50,000+ en facturas
```

### Historia Real

> *"Hice commit de `terraform.tfstate` sin darme cuenta. Contenía credenciales de RDS. En 10 minutos, alguien accedió a mi DB, la copió, la borró y dejó una nota de rescate. Perdí datos de 500 clientes."*  
> — Desarrollador anónimo, Reddit 2023

### ¿Por qué no simplemente agregarlo a .gitignore?

**Problemas del State Local:**

1. **Robo/pérdida de laptop** → State perdido o comprometido
2. **Trabajo en equipo:**
   ```
   Dev A: terraform apply → Crea DB
   Dev B: git pull (NO incluye state en .gitignore)
   Dev B: terraform apply → No sabe que existe la DB
   Terraform: "Esta DB no está en mi state, la borraré"
   Resultado: DATA LOSS
   ```
3. **Sin backup:** Si pierdes el state, pierdes el control de tu infraestructura
4. **Sin auditoría:** No sabes quién modificó qué

---

## Remote Backend: La Solución

### Arquitectura del Remote Backend

```
┌─────────────────────────────────────────────┐
│           Desarrolladores                    │
│  Dev A        Dev B        CI/CD Pipeline    │
└──────┬───────────┬──────────────┬───────────┘
       │           │              │
       ▼           ▼              ▼
┌──────────────────────────────────────────────┐
│          DynamoDB Table (Locking)            │
│  Previene operaciones concurrentes           │
└──────────────────────────────────────────────┘
       │           │              │
       ▼           ▼              ▼
┌──────────────────────────────────────────────┐
│         S3 Bucket (State Storage)            │
│  • Encriptado (AES-256)                      │
│  • Versionado (rollback si falla algo)       │
│  • Privado (no accesible desde internet)     │
└──────────────────────────────────────────────┘
```

### Componentes

#### 1. S3 Bucket
- **Propósito:** Almacenar el archivo `terraform.tfstate`
- **Características:**
  - Versionado habilitado (historial de cambios)
  - Encriptación AES-256 (seguridad en reposo)
  - Acceso bloqueado públicamente
  - Lifecycle policy `prevent_destroy` (protección contra borrado)

#### 2. DynamoDB Table
- **Propósito:** Implementar locking (bloqueo)
- **Características:**
  - Billing mode: Pay-per-request (~$0.00 en la práctica)
  - Previene race conditions en equipo
  - Atributo `LockID` requerido por Terraform

### Flujo de Locking

```
Dev A ejecuta: terraform apply
  ↓
Terraform escribe en DynamoDB:
  {
    "LockID": "state-file",
    "Owner": "DevA@laptop",
    "Timestamp": "2024-12-12T10:00:00Z"
  }
  ↓
Dev B ejecuta: terraform apply (simultáneamente)
  ↓
Terraform intenta adquirir lock
  ↓
DynamoDB responde: "Error, lock ya existe"
  ↓
Terraform muestra: "State locked by DevA, waiting..."
  ↓
Cuando Dev A termina → Borra el lock
  ↓
Dev B puede continuar
```

---

## Implementación Paso a Paso

### Fase 1: Estructura del Proyecto

```bash
cd go-hexagonal-auth

# Crear estructura
mkdir -p terraform/backend

# Verificar .gitignore
cat terraform/.gitignore | grep tfstate
```

**El `.gitignore` debe incluir:**
```gitignore
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl
*.tfvars
```

### Fase 2: Código del Backend

#### `terraform/backend/main.tf`

```terraform
terraform {
  required_version = ">= 1.6.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Project     = var.project_name
      Environment = "backend"
    }
  }
}

# S3 Bucket para el state
resource "aws_s3_bucket" "terraform_state" {
  bucket = "${var.project_name}-terraform-state-${var.aws_account_id}"
  
  lifecycle {
    prevent_destroy = true
  }
}

# Versionado del bucket
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# Encriptación
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Bloquear acceso público
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# DynamoDB para locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "${var.project_name}-terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
  
  lifecycle {
    prevent_destroy = true
  }
}
```

#### `terraform/backend/variables.tf`

```terraform
variable "aws_region" {
  description = "AWS region donde crear el backend"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Nombre del proyecto"
  type        = string
  default     = "go-hexagonal-auth"
}

variable "aws_account_id" {
  description = "AWS Account ID (para naming único)"
  type        = string
}
```

#### `terraform/backend/outputs.tf`

```terraform
output "s3_bucket_name" {
  description = "Nombre del bucket S3"
  value       = aws_s3_bucket.terraform_state.id
}

output "dynamodb_table_name" {
  description = "Nombre de la tabla DynamoDB"
  value       = aws_dynamodb_table.terraform_locks.name
}

output "backend_config" {
  description = "Configuración para backend.tf"
  value = <<-EOT
  terraform {
    backend "s3" {
      bucket         = "${aws_s3_bucket.terraform_state.id}"
      key            = "terraform.tfstate"
      region         = "${var.aws_region}"
      dynamodb_table = "${aws_dynamodb_table.terraform_locks.name}"
      encrypt        = true
    }
  }
  EOT
}
```

### Fase 3: Ejecución

```bash
# 1. Exportar variables
export TF_VAR_aws_account_id=$(aws sts get-caller-identity --query Account --output text)
export TF_VAR_aws_region="us-east-1"
export TF_VAR_project_name="go-hexagonal-auth"

# 2. Inicializar Terraform
cd terraform/backend
terraform init

# 3. Revisar el plan
terraform plan

# 4. Crear recursos
terraform apply
# Escribir: yes

# 5. Guardar configuración
terraform output -raw backend_config > ../backend-config.txt
```

**Recursos creados:**
- S3 Bucket: `go-hexagonal-auth-terraform-state-XXXXXXXXXXXX`
- DynamoDB Table: `go-hexagonal-auth-terraform-locks`

---

## Migración del State

### Por Qué Migrar

**Estado actual:**
```
terraform/backend/terraform.tfstate (LOCAL)
  ↓ En tu laptop
  ↓ Vulnerable
  ↓ Sin locking
```

**Estado deseado:**
```
S3 Bucket (REMOTO)
  ↓ En AWS
  ↓ Encriptado
  ↓ Versionado
  ↓ Con locking
```

### Proceso de Migración

#### 1. Crear `terraform/backend.tf`

```terraform
terraform {
  backend "s3" {
    bucket         = "go-hexagonal-auth-terraform-state-XXXXXXXXXXXX"
    key            = "terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "go-hexagonal-auth-terraform-locks"
    encrypt        = true
  }
}
```

**⚠️ Importante:** Reemplaza `XXXXXXXXXXXX` con tu Account ID real.

#### 2. Ejecutar migración

```bash
cd terraform
terraform init -migrate-state
```

**Terraform preguntará:**
```
Do you want to copy existing state to the new backend?
  Enter a value:
```

**Responder:** `yes`

#### 3. Verificar

```bash
# El state debe estar en S3
aws s3 ls s3://go-hexagonal-auth-terraform-state-XXXXXXXXXXXX/

# NO debe haber state local
ls terraform.tfstate
# Output: No such file or directory
```

---

## Verificación y Testing

### Test 1: State en S3

```bash
# Obtener nombre del bucket
BUCKET=$(cd terraform/backend && terraform output -raw s3_bucket_name)

# Listar contenido
aws s3 ls s3://${BUCKET}/

# Output esperado:
# 2024-XX-XX XX:XX:XX    XXXX terraform.tfstate
```

### Test 2: Versionado

```bash
# Ver versiones del state
aws s3api list-object-versions \
  --bucket ${BUCKET} \
  --prefix terraform.tfstate \
  --query 'Versions[*].[VersionId,LastModified]' \
  --output table
```

### Test 3: Locking

**Terminal 1:**
```bash
cd terraform/backend
terraform console
# Mantener abierto
```

**Terminal 2:**
```bash
cd terraform/backend
terraform plan
```

**Output esperado en Terminal 2:**
```
Error: Error acquiring the state lock

Lock Info:
  ID:        xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
  Who:       usuario@laptop
```

**✅ El locking funciona correctamente.**

### Test 4: Git Ignore

```bash
git status
```

**NO debe mostrar:**
- `terraform.tfstate`
- `.terraform/`
- `*.tfvars`

---

## Conclusiones

### Lo que lograste

- ✅ **Backend remoto:** State en S3 (nunca más en laptop ni Git)
- ✅ **Encriptación:** AES-256 para datos en reposo
- ✅ **Versionado:** Historial completo, rollback posible
- ✅ **Locking:** Trabajo en equipo sin conflictos
- ✅ **Seguridad:** Protección contra exposición accidental

### Costos

| Recurso | Costo Mensual |
|---------|---------------|
| S3 Bucket (state < 1MB) | $0.00 |
| DynamoDB (pay-per-request, bajo uso) | $0.00 |
| **Total** | **< $0.01 USD** |

### Mejores Prácticas

1. **NUNCA** hacer commit de `terraform.tfstate`
2. **SIEMPRE** usar remote backend para proyectos compartidos
3. **SIEMPRE** habilitar versionado en S3
4. **SIEMPRE** usar locking en trabajo en equipo
5. **Revisar** `git status` antes de cada commit

### Recursos Adicionales

- [Documentación oficial de Terraform Backend](https://www.terraform.io/docs/language/settings/backends/s3.html)
- [AWS S3 Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html)
- [DynamoDB Locking](https://www.terraform.io/docs/language/settings/backends/s3.html#dynamodb-state-locking)

### Próximos Pasos

En el **Módulo 2** crearemos:
- IAM Roles con permisos específicos
- Tu primera Lambda function en Go
- DynamoDB table para la aplicación
- API Gateway para exponer la Lambda

---

## Código Completo

Todo el código de este tutorial está disponible en:
- [terraform/backend/](../../terraform/backend/) - Configuración del backend
---

## Preguntas Frecuentes

**Q: ¿Puedo usar el mismo backend para múltiples proyectos?**  
A: No recomendado. Cada proyecto debe tener su propio bucket y tabla DynamoDB para aislamiento.

**Q: ¿Qué pasa si borro accidentalmente el state de S3?**  
A: Por eso el versionado está habilitado. Puedes restaurar versiones anteriores desde la consola de S3.

**Q: ¿El locking funciona con Terraform Cloud?**  
A: Terraform Cloud tiene su propio sistema de locking. Este tutorial es para self-hosted state.

**Q: ¿Cuánto cuesta si tengo muchos cambios frecuentes?**  
A: El costo de S3 es mínimo (~$0.023/GB). El state típicamente es < 1MB. DynamoDB pay-per-request es prácticamente gratis con <100 operaciones/día.

---

**Autor:** Edgar (Homz) Macías  
**Serie:** AWS Zero to Architect  
**Fecha:** Diciembre 2025  
**Licencia:** MIT

---

⭐ **Si este tutorial te ayudó, considera darle una estrella al [repositorio](https://github.com/edgar-macias-se/aws_road)**
