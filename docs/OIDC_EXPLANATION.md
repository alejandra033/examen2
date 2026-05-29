# OIDC (OpenID Connect) - Explicación Completa

## 📌 Tabla de Contenidos

1. [¿Qué es OIDC?](#qué-es-oidc)
2. [El Problema que Resuelve](#el-problema-que-resuelve)
3. [Cómo Funciona](#cómo-funciona)
4. [Beneficios de Seguridad](#beneficios-de-seguridad)
5. [Cuándo Utilizarlo](#cuándo-utilizarlo)
6. [Configuración Paso a Paso](#configuración-paso-a-paso)
7. [Comparativa con Alternatives](#comparativa-con-alternatives)

---

## ¿Qué es OIDC?

**OIDC (OpenID Connect)** es un protocolo abierto de autenticación que se construye sobre **OAuth 2.0**.

### Definición Simple

```
OIDC = Una forma de probar "quién eres" sin compartir contraseñas.
```

En el contexto de GitHub Actions:

```
GitHub Actions dice: "Soy GitHub Actions, ejecutando el job 'deploy'"
AWS responde: "Confío en GitHub Actions, aquí están tus credenciales"
```

### Comparación: OpenID vs OAuth

| Aspecto | OpenID | OAuth |
|--------|--------|-------|
| **Propósito** | Autenticación (¿Quién eres?) | Autorización (¿Qué puedes hacer?) |
| **Salida** | Usuario autenticado | Token de acceso |
| **Ejemplo** | "Login con Google" | "Acceso a tu calendario" |

**OIDC combina ambos**:
- Autenticación: Verifica que eres GitHub Actions
- Autorización: Te da credenciales temporales de AWS

---

## El Problema que Resuelve

### Enfoque Tradicional: Secrets Hardcodeados

```yaml
# ❌ INSEGURO (método antiguo)
env:
  AWS_ACCESS_KEY_ID: AKIAIOSFODNN7EXAMPLE
  AWS_SECRET_ACCESS_KEY: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

**Problemas:**

| Problema | Severidad | Impacto |
|----------|-----------|--------|
| **Largo tiempo de vida** | 🔴 CRÍTICO | Las credenciales son válidas por años |
| **Hardcodeadas en repo** | 🔴 CRÍTICO | Si hay acceso al repo, hay acceso a AWS |
| **Una sola contraseña** | 🔴 CRÍTICO | Todos los workflows usan la misma credencial |
| **Difícil de rotar** | 🟠 ALTO | Cambiarla afecta TODOS los workflows |
| **Sin contexto** | 🟠 ALTO | No se sabe qué workflow usó la credencial |
| **Posible exfiltración** | 🟠 ALTO | El atacante tiene credenciales válidas por años |

### Ejemplo de Ataque

```
1. Atacante obtiene acceso al repositorio (ej: comprometer una cuenta)
2. Lee el archivo .github/workflows/deploy.yml
3. Encuentra AWS_SECRET_ACCESS_KEY
4. Usa la credencial para acceder a AWS
5. Datos comprometidos por MESES/AÑOS antes de ser detectados
6. No hay forma de saber qué fue comprometido
```

---

## Cómo Funciona

### Paso a Paso: Flujo OIDC

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  1. GitHub Actions inicia el workflow                      │
│                                                             │
│     - Job: Deploy                                          │
│     - Branch: main                                         │
│     - Commit SHA: abc123def456...                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  2. GitHub genera un JWT token                             │
│                                                             │
│     JWT = JSON Web Token (firmado con llave privada GitHub)│
│                                                             │
│     {                                                      │
│       "iss": "https://token.actions.githubusercontent.com",│
│       "sub": "repo:owner/repo:ref:refs/heads/main",       │
│       "aud": "sts.amazonaws.com",                          │
│       "iat": 1694765523,                  // issued-at     │
│       "exp": 1694766423,          // expires in 15 min    │
│       "sha": "abc123def456...",                            │
│       "repository": "owner/repo",                          │
│       "environment": "production"                          │
│     }                                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  3. GitHub Actions envía JWT a AWS                         │
│                                                             │
│     POST /sts:AssumeRoleWithWebIdentity                    │
│     Token: eyJhbGciOiJSUzI1NiJ9...                        │
│     RoleArn: arn:aws:iam::123456789012:role/github-actions│
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  4. AWS valida el JWT                                      │
│                                                             │
│     ✓ Firma válida (verificada con llave pública GitHub)  │
│     ✓ No expirado (exp > ahora)                           │
│     ✓ Audience correcto (sts.amazonaws.com)               │
│     ✓ Subject matches trust policy (repo:owner/repo:ref) │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  5. AWS emite credenciales temporales                      │
│                                                             │
│     {                                                      │
│       "AccessKeyId": "ASIA...",                            │
│       "SecretAccessKey": "...",                            │
│       "SessionToken": "...",                               │
│       "Expiration": "2024-01-01T12:30:00Z"  // 15 min    │
│     }                                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  6. GitHub Actions recibe credenciales                     │
│                                                             │
│     - Válidas solo 15 minutos                             │
│     - Vinculadas a este job específico                    │
│     - Incluyen SessionToken (mejor auditoría)             │
│     - No almacenadas en repositorio                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  7. AWS CLI/SDK usa las credenciales                       │
│                                                             │
│     $ aws s3 ls                                           │
│     # Usa credenciales temporales                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  8. Credenciales expiran automáticamente                   │
│                                                             │
│     - Después de 15 minutos                               │
│     - No se pueden reutilizar                             │
│     - Imposible acceder a AWS después del job             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Código en el Workflow

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/github-actions
    aws-region: us-east-1
    role-duration-seconds: 900  # 15 minutos

# Internamente, la action:
# 1. Obtiene JWT de GitHub
# 2. Lo intercambia por credenciales en AWS
# 3. Configura AWS CLI con las credenciales
# 4. Las credenciales expiran en 15 min
```

---

## Beneficios de Seguridad

### 1. **Credenciales de Corta Duración**

```
Secrets (antiguo):        OIDC (nuevo):
├─ 1+ años               ├─ 15 minutos
├─ Hardcodeadas         ├─ Generadas dinámicamente
└─ Reutilizables        └─ De un solo uso
```

**Impacto de Seguridad:**
- Si el JWT es comprometido, es inútil después de 15 minutos
- El atacante no puede usarlo para acceso permanente

### 2. **Contexto en Cada Token**

El JWT incluye información sobre el contexto:

```json
{
  "repository": "owner/repo",
  "ref": "refs/heads/main",      // Solo rama main
  "sha": "abc123...",             // Commit específico
  "environment": "production",    // Environment específico
  "run_id": "12345",              // Este job específico
  "actor": "github-user"          // Quién disparó el workflow
}
```

**Auditoría:**
- Sabes exactamente quién (actor)
- Accedió desde dónde (repo)
- Cuándo (iat, exp)
- A qué ambiente (environment)
- Qué versión del código (sha)

### 3. **No Requiere Rotación Manual**

```
Secrets:
- Cada 30-90 días: ROTAR manual
- Si alguien abandona: ROTAR
- Si hay sospecha de leak: ROTAR
- Proceso manual, propenso a errores

OIDC:
- Automático: cada token es nuevo
- Si hay leak: Tokens ya expirados
- Nada que rotar manualmente
- Seguro por diseño
```

### 4. **Vinculación a Rama**

```yaml
"token.actions.githubusercontent.com:sub": "repo:owner/repo:ref:refs/heads/main"
```

**Efecto:**
- Solo los workflows en rama `main` obtienen credenciales
- Feature branches NO pueden acceder a producción
- Imposible hacer deploy desde una rama no autorizada

### 5. **Imposibilidad de Compartir Credenciales**

```
Secrets: 
- Un secret para todos los workflows
- Alguien puede copiar y usar en otro lugar

OIDC:
- Cada token es único para ese momento
- No se puede copiar y reutilizar
- Vinculado a contexto específico
```

### 6. **Mejor Auditoría en CloudTrail**

```json
// AWS CloudTrail (con OIDC)
{
  "eventName": "AssumeRoleWithWebIdentity",
  "requestParameters": {
    "roleArn": "arn:aws:iam::123456789012:role/github-actions",
    "webIdentityToken": "[JWT token]"  // <-- Contiene contexto
  },
  "userAgent": "aws-cli/2.x",
  "sourceIPAddress": "140.82.112.0/21"  // GitHub actions IP range
}
```

Puedes ver en AWS CloudTrail:
- El token JWT completo (con contexto)
- Desde qué repositorio
- Qué commit
- Quién disparó el workflow

---

## Cuándo Utilizarlo

### ✅ USA OIDC CUANDO:

| Caso de Uso | Razón |
|------------|-------|
| **Deploy a AWS/Azure/GCP** | Soporte nativo de OIDC |
| **Acceso a recursos cloud** | Credenciales dinámicas |
| **Production environment** | Máximo nivel de seguridad |
| **Compliance requirements** | Auditabilidad mejorada |
| **Equipos grandes** | Evita compartir secrets |
| **CI/CD pipeline** | Automatización de rotación |

### ❌ USA SECRETS CUANDO:

| Caso de Uso | Razón |
|------------|-------|
| **Registries privadas** | Mejor soporte (Docker, npm, etc.) |
| **APIs externas** | No todas soportan OIDC |
| **Configuración heredada** | Migración gradual |
| **Herramientas sin OIDC** | Fallback necesario |

### 📊 Matriz de Decisión

```
¿Es acceso a cloud (AWS/Azure/GCP)?
├─ SÍ: ¿Soporta OIDC?
│   ├─ SÍ → USA OIDC ✅
│   └─ NO → USA SECRETS (con rotación manual)
└─ NO: ¿Es producción?
    ├─ SÍ → OIDC si es posible, sino SECRETS
    └─ NO → SECRETS es suficiente
```

---

## Configuración Paso a Paso

### Paso 1: Verificar Proveedores Soportados

GitHub soporta OIDC con:
- ✅ AWS
- ✅ Azure
- ✅ Google Cloud
- ✅ HashiCorp Vault
- ✅ Otros proveedores compatibles

### Paso 2: Configurar en AWS

#### 2A: Crear OIDC Provider

```bash
# Obtener thumbprint de GitHub
THUMBPRINT=$(curl -s https://token.actions.githubusercontent.com/.well-known/openid-configuration | \
  openssl s_client -showcerts -connect token.actions.githubusercontent.com:443 2>/dev/null | \
  openssl x509 -noout -fingerprint | cut -d= -f2 | tr -d ':')

# Crear provider
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list $THUMBPRINT
```

#### 2B: Crear Role

```bash
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:owner/repo:*"
        }
      }
    }
  ]
}
EOF

aws iam create-role \
  --role-name github-actions \
  --assume-role-policy-document file://trust-policy.json
```

#### 2C: Adjuntar Permisos

```bash
# Política mínima para deploy
cat > policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:UpdateService",
        "ecs:DescribeServices",
        "ecs:DescribeTaskDefinition",
        "ecs:DescribeTasks",
        "ecs:ListTasks",
        "iam:PassRole",
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name github-actions \
  --policy-name github-actions-policy \
  --policy-document file://policy.json
```

### Paso 3: Configurar en GitHub Workflow

```yaml
name: Deploy with OIDC

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write    # CRÍTICO: Necesario para OIDC
      contents: read
    steps:
      - uses: actions/checkout@2541b1294d2704b0964813337f33b291d3f8596b

      # La magia sucede aquí
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions
          aws-region: us-east-1
          role-duration-seconds: 900  # 15 minutos

      # Ahora AWS CLI funciona sin secrets
      - name: Deploy
        run: |
          aws ecs update-service \
            --cluster prod \
            --service app \
            --force-new-deployment
```

### Paso 4: Verificar Funcionamiento

```bash
# En el workflow, agregar step de prueba
- name: Verify OIDC
  run: |
    # Si llegamos aquí, OIDC funcionó
    echo "✅ OIDC configuration successful"
    
    # Mostrar información del token (sin exponer el token)
    aws sts get-caller-identity
```

---

## Comparativa con Alternatives

### Método 1: API Keys Hardcodeadas

```yaml
# ❌ MÁS INSEGURO
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

| Aspecto | Calificación |
|--------|-------------|
| Seguridad | 🔴 Baja |
| Auditoría | 🟠 Media |
| Rotación | 🟠 Manual |
| Duración | 🟠 Años |
| Contexto | 🔴 Ninguno |

### Método 2: OIDC

```yaml
# ✅ MÁS SEGURO
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/github-actions
```

| Aspecto | Calificación |
|--------|-------------|
| Seguridad | 🟢 Alta |
| Auditoría | 🟢 Completa |
| Rotación | 🟢 Automática |
| Duración | 🟢 15 min |
| Contexto | 🟢 Completo |

### Tabla Comparativa Completa

```
┌─────────────────────┬──────────────────────┬──────────────────┐
│ Característica      │ Secrets Tradicional  │ OIDC             │
├─────────────────────┼──────────────────────┼──────────────────┤
│ Almacenamiento      │ ❌ Repositorio       │ ✅ No almacenado │
│ Duración            │ ❌ Años              │ ✅ 15 minutos    │
│ Rotación manual     │ ❌ Necesaria         │ ✅ Automática    │
│ Contexto            │ ❌ Ninguno           │ ✅ Completo      │
│ Auditoría           │ ❌ Básica            │ ✅ Detallada     │
│ Riesgo si leak      │ ❌ Crítico (años)    │ ✅ Bajo (expira) │
│ Compartibilidad     │ ❌ Posible           │ ✅ Imposible     │
│ Configuración       │ ✅ Simple            │ ❌ Compleja      │
│ Soporte             │ ✅ Todos (fallback)  │ 🟠 Solo cloud    │
└─────────────────────┴──────────────────────┴──────────────────┘
```

---

## Resumen

### OIDC Resuelve

1. **Credenciales hardcodeadas** → Generadas dinámicamente
2. **Long-lived secrets** → Short-lived tokens
3. **Sin contexto** → Contexto completo
4. **Rotación manual** → Automática
5. **Auditoría pobre** → Completa

### Cuándo Usar

- ✅ Deploy a AWS/Azure/GCP
- ✅ Production environments
- ✅ Equipos grandes
- ✅ Compliance requirements

### Cuándo NO Usar

- ❌ APIs externas sin OIDC
- ❌ Registries privadas
- ❌ Herramientas legacy

---

**Última actualización**: 2026-05-29
