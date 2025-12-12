> **Autor:** Edgar (Homz) Macias
> **Última actualización:** Diciembre 2025  
> **Nivel:** Principiante  
> **Tiempo estimado:** 45-60 minutos

---

## 📋 Índice

1. [Introducción](#introducción)
2. [¿Por qué este tutorial existe?](#por-qué-este-tutorial-existe)
3. [Conceptos Fundamentales](#conceptos-fundamentales)
4. [Paso 1: Asegurar Root User](#paso-1-asegurar-root-user)
5. [Paso 2: Crear Usuario IAM](#paso-2-crear-usuario-iam)
6. [Paso 3: Configurar Alertas de Facturación](#paso-3-configurar-alertas-de-facturación)
7. [Paso 4: Instalar AWS CLI](#paso-4-instalar-aws-cli)
8. [Paso 5: Blindar tu Repositorio](#paso-5-blindar-tu-repositorio)
9. [Verificación Final](#verificación-final)
10. [Recursos Adicionales](#recursos-adicionales)

---

## Introducción

Este tutorial te guía a través del proceso de configuración **segura** de una cuenta AWS desde cero, con énfasis especial en:

- 🔒 **Seguridad:** Proteger tu cuenta contra accesos no autorizados
- 💰 **Control de costos:** Evitar sorpresas en facturación
- 🎯 **Mejores prácticas:** Configuración profesional desde el día 1

**⚠️ IMPORTANTE:** Este tutorial asume que trabajarás con repositorios **públicos**. Toda la configuración está diseñada para **NUNCA exponer secretos**.

---

## ¿Por qué este tutorial existe?

### El problema común

Muchos desarrolladores comienzan en AWS y cometen estos errores:

1. Usan el Root User para trabajo diario → **Riesgo de seguridad crítico**
2. No configuran alertas de facturación → **Facturas de $1,000+ en 24 horas**
3. Suben credenciales a GitHub → **Bots las encuentran en 5 minutos**

### Historias de terror reales

- **Desarrollador A:** Subió `aws_credentials.txt` a GitHub público → Bots crearon 50 instancias GPU → **Factura: $15,000**
- **Desarrollador B:** Usó Root sin MFA → Phishing comprometió la cuenta → Perdió acceso completo
- **Desarrollador C:** No configuró alarmas → Dejó una Lambda en loop infinito → **Factura: $3,200**

**Este tutorial previene estos escenarios.**

---

## Conceptos Fundamentales

### Root User vs IAM User

| Característica | Root User | IAM User |
|---------------|-----------|----------|
| **Poder** | Control absoluto sobre TODO | Permisos limitados que tú defines |
| **Uso recomendado** | Solo setup inicial | Trabajo diario |
| **MFA** | Obligatorio | Altamente recomendado |
| **Riesgo si se compromete** | Crítico (pérdida total) | Limitado al scope del usuario |
| **Analogía** | Usuario `root` en Linux | Usuario con `sudo` limitado |

### ¿Qué es MFA?

**Multi-Factor Authentication (Autenticación de Dos Factores)**

- **Sin MFA:** Solo necesitas contraseña → Un keylogger te compromete
- **Con MFA:** Necesitas contraseña + código temporal (celular o USB) → Mucho más seguro

**Apps recomendadas:**
- Google Authenticator (iOS/Android)
- Authy (multi-dispositivo)
- Microsoft Authenticator

---

## Paso 1: Asegurar Root User

### 1.1 - Acceder a la Consola

1. Ve a [https://console.aws.amazon.com](https://console.aws.amazon.com)
2. Inicia sesión como **Root user** (usarás el email con el que creaste la cuenta)

### 1.2 - Activar MFA en Root

1. Clic en tu nombre (arriba derecha) → **Security credentials**
2. Sección **"Multi-factor authentication (MFA)"**
3. **Assign MFA device** → Selecciona:
   - **Authenticator app** (recomendado para empezar)
   - **Security key** (si tienes YubiKey o similar)
4. Escanea el código QR con tu app (Google Authenticator/Authy)
5. Introduce **2 códigos MFA consecutivos** (espera 30 segundos entre cada uno)
6. **¡IMPORTANTE!** Guarda los códigos de recuperación en un lugar seguro físico

### 1.3 - Crear Alias de Cuenta (Opcional)

1. Misma pantalla → Busca **"Account Alias"**
2. Crea un alias memorable (ej: `mi-startup-prod`, `personal-aws`)
3. **Beneficio:** URL de login más amigable

### 1.4 - Cerrar sesión de Root

**A partir de ahora, NUNCA uses Root para trabajo diario.**

---

## Paso 2: Crear Usuario IAM

### 2.1 - Ir a IAM

1. En la consola, busca **"IAM"** en la barra superior
2. Menú izquierdo → **Users** → **Create user**

### 2.2 - Configurar Usuario

**User name:** Tu nombre (ej: `juan-admin`, `maria-dev`)

**Access type:**
- ✅ Marca: **"Provide user access to the AWS Management Console"**

**Console password:**
- Opción 1: **Custom password** (crea una contraseña fuerte)
- Opción 2: **Auto-generated** (AWS genera una aleatoria)

**IMPORTANTE:** Desmarca "Users must create a new password at next sign-in" (para simplificar)

### 2.3 - Asignar Permisos

**Para aprendizaje:**
- Selecciona **"Attach policies directly"**
- Busca y marca: **`AdministratorAccess`**

**Para producción (después):**
- Usa permisos más granulares según el principio de "Least Privilege"
- Ejemplo: `AmazonS3FullAccess`, `AWSLambdaFullAccess`, etc.

### 2.4 - Revisar y Crear

1. Clic **Next** → **Create user**
2. **DESCARGA** el archivo CSV con credenciales
3. Guarda este CSV en un **gestor de contraseñas** (1Password, Bitwarden, etc.)

### 2.5 - Activar MFA para IAM User

1. En la lista de usuarios → Clic en tu usuario
2. Pestaña **"Security credentials"**
3. Sección **"Multi-factor authentication (MFA)"**
4. **Assign MFA device** → Mismo proceso que Root

### 2.6 - Iniciar sesión como IAM User

1. Cierra sesión de Root
2. Ve a: `https://TU-ALIAS.signin.aws.amazon.com` (o usa el Account ID)
3. **IAM user name:** `juan-admin`
4. **Password:** La que configuraste
5. **MFA code:** Código actual de tu app

---

## Paso 3: Configurar Alertas de Facturación

### 3.1 - Habilitar Alertas (Requiere Root)

**⚠️ Solo este paso requiere Root temporalmente**

1. Inicia sesión como **Root User**
2. Clic en tu nombre → **Account**
3. Baja hasta **"Billing preferences"** → **Edit**
4. Activa:
   - ✅ **"Receive Free Tier Usage Alerts"**
   - ✅ **"Receive Billing Alerts"**
5. Introduce tu email → **Save preferences**
6. **Cierra sesión de Root**

### 3.2 - Crear Presupuesto ($0.01)

**Ahora como IAM User:**

1. Busca **"AWS Budgets"** (o Billing → Budgets)
2. **Create budget**
3. **Template:** Selecciona **"Zero spend budget"** (recomendado para empezar)
   - O usa **"Customize"** para $0.01 exactos

**Si usas Customize:**
- **Budget name:** `Alerta-Costo-Minimo`
- **Period:** Monthly
- **Budgeted amount:** `0.01` USD
- **Budget scope:** All AWS services

### 3.3 - Configurar Alertas del Presupuesto

**Alert 1 - Costo Real:**
- **Threshold:** Actual costs - 100% (cuando alcances $0.01)
- **Email:** Tu email principal

**Alert 2 - Pronóstico (Opcional):**
- **Threshold:** Forecasted costs - 80%
- **Email:** Tu email

### 3.4 - Free Tier Alert (Adicional)

AWS enviará emails automáticamente cuando:
- Estés cerca de superar límites del Free Tier
- Hayas superado un servicio del Free Tier

---

## Paso 4: Instalar AWS CLI

### 4.1 - Instalación según OS

**macOS:**
```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```

**Linux:**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**Windows:**
1. Descarga: [https://awscli.amazonaws.com/AWSCLIV2.msi](https://awscli.amazonaws.com/AWSCLIV2.msi)
2. Ejecuta el instalador

**Verificar:**
```bash
aws --version
# Output esperado: aws-cli/2.x.x Python/3.x.x ...
```

### 4.2 - Crear Access Keys

**⚠️ SOLO para tu usuario IAM, NUNCA para Root**

1. Consola AWS (como IAM User) → **IAM** → **Users** → Tu usuario
2. Pestaña **"Security credentials"**
3. Sección **"Access keys"** → **Create access key**
4. **Use case:** **Command Line Interface (CLI)**
5. Marca "I understand..." → **Next**
6. (Opcional) Description: `CLI-Local-DevMachine`
7. **Create access key**
8. **¡DESCARGA EL CSV!** (Solo se muestra UNA vez)

### 4.3 - Configurar Perfil Local

```bash
aws configure --profile mi-proyecto
```

**Introduce:**
1. **AWS Access Key ID:** `AKIA...` (del CSV)
2. **AWS Secret Access Key:** `wJalr...` (del CSV)
3. **Default region:** `us-east-1` (recomendado para empezar)
4. **Output format:** `json` (o vacío)

**¿Dónde se guarda?**
- Linux/macOS: `~/.aws/credentials` y `~/.aws/config`
- Windows: `C:\Users\TuUsuario\.aws\credentials`

### 4.4 - Configurar Variable de Entorno

**Bash/Zsh:**
```bash
echo 'export AWS_PROFILE=mi-proyecto' >> ~/.zshrc
source ~/.zshrc
```

**Fish:**
```fish
echo 'set -gx AWS_PROFILE mi-proyecto' >> ~/.config/fish/config.fish
source ~/.config/fish/config.fish
```

**Windows PowerShell:**
```powershell
[System.Environment]::SetEnvironmentVariable('AWS_PROFILE', 'mi-proyecto', 'User')
```

### 4.5 - Probar Configuración

```bash
aws sts get-caller-identity
```

**Output esperado:**
```json
{
    "UserId": "AIDAXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/juan-admin"
}
```

**✅ Si el `Arn` contiene `:user/TuNombre` → Todo correcto**  
**❌ Si contiene `:root` → Estás usando Root (MAL)**

---

## Paso 5: Blindar tu Repositorio

### 5.1 - Crear `.gitignore` (CRÍTICO)

En la raíz de tu proyecto:

```bash
touch .gitignore
```

**Contenido mínimo obligatorio:**

```gitignore
# === SECRETOS AWS Y TERRAFORM ===
*.tfvars
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl

# === CREDENCIALES ===
.env
.env.*
*.pem
*.key
aws_credentials.txt
credentials.json
config.json

# === GOLANG ===
*.exe
*.dll
*.so
*.dylib
*.test
*.out
vendor/
go.work

# === SISTEMA ===
.DS_Store
Thumbs.db

# === EDITORES ===
.vscode/
.idea/
*.swp
*.swo
*~
```

### 5.2 - Verificar ANTES de commit

```bash
git status
```

**NUNCA debes ver:**
- Archivos `.tfstate`
- Archivos `.env`
- Archivos con extensión `.pem` o `.key`
- Directorios `.terraform/`

### 5.3 - Agregar Pre-Commit Check (Opcional pero recomendado)

Instala `pre-commit`:

```bash
# macOS
brew install pre-commit

# Linux
pip install pre-commit

# Windows
pip install pre-commit
```

Crea `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-added-large-files
      - id: detect-aws-credentials
      - id: detect-private-key
      
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.5
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
```

Activar:

```bash
pre-commit install
```

---

## Verificación Final

### Checklist de Seguridad

- [ ] Root User tiene MFA activado
- [ ] Root User NO se usa para trabajo diario
- [ ] IAM User creado con permisos AdministratorAccess
- [ ] IAM User tiene MFA activado
- [ ] AWS Budgets configurado ($0.01 o Zero Spend)
- [ ] Free Tier alerts habilitadas
- [ ] AWS CLI instalado y verificado
- [ ] Perfil de AWS CLI configurado (NO usa Root)
- [ ] `.gitignore` creado y validado
- [ ] Ningún archivo sensible en `git status`

### Test de Comando

```bash
# Debe mostrar tu usuario IAM
aws sts get-caller-identity

# Debe listar buckets (vacío si no tienes ninguno)
aws s3 ls

# Debe mostrar tu región configurada
aws configure get region
```

---

## Recursos Adicionales

### Documentación Oficial

- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS Free Tier](https://aws.amazon.com/free/)
- [AWS CLI User Guide](https://docs.aws.amazon.com/cli/latest/userguide/)

### Herramientas Recomendadas

- **Gestores de contraseñas:** 1Password, Bitwarden, LastPass
- **MFA Apps:** Google Authenticator, Authy, Microsoft Authenticator
- **Git Security:** git-secrets, pre-commit

### Límites del Free Tier (Primeros 12 meses)

| Servicio | Límite Mensual |
|----------|----------------|
| EC2 | 750 horas de t2.micro |
| S3 | 5GB de almacenamiento |
| Lambda | 1M de invocaciones |
| DynamoDB | 25GB de almacenamiento |
| RDS | 750 horas de db.t2.micro |

**⚠️ Servicios SIN Free Tier:**
- NAT Gateway (~$32/mes)
- Load Balancers (~$16/mes)
- Elastic IP no asignada ($3.6/mes)

---

## Próximos Pasos

Una vez completado este módulo:

1. **Módulo 1:** Terraform y Remote Backend (S3 + DynamoDB)
2. **Módulo 2:** IAM Roles y Políticas avanzadas
3. **Módulo 3:** Desplegar aplicación Go con Lambda

---

## Preguntas Frecuentes

**Q: ¿Puedo usar Root User ocasionalmente?**  
A: Solo para tareas que REQUIERAN Root (cambiar plan de soporte, cerrar cuenta). Día a día: NO.

**Q: ¿Qué hago si pierdo acceso a MFA?**  
A: Usa los códigos de recuperación que guardaste. Si los perdiste, contacta a AWS Support (puede tomar días).

**Q: ¿Puedo tener múltiples usuarios IAM?**  
A: Sí. Crea uno por persona/servicio con permisos específicos.

**Q: ¿Qué hago si subí credenciales a GitHub por error?**  
A: 1) Borra el commit inmediatamente, 2) Rota las credenciales en IAM, 3) Revisa CloudTrail por actividad sospechosa.

---

## Contribuciones

Si encuentras errores o mejoras, abre un Issue o Pull Request.

---

## Licencia

MIT License - Úsalo, compártelo, mejóralo.

---

**Creado por:** Edgar (Homz) Macias
**Repositorio:** https://github.com/edgar-macias-se/aws_road