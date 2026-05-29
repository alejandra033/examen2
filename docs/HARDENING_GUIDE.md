# Guía Completa de Hardening en GitHub Actions

## 📖 Índice

1. [Introducción](#introducción)
2. [Permisos](#permisos)
3. [Supply Chain](#supply-chain)
4. [Secrets](#secrets)
5. [Environments](#environments)
6. [OIDC](#oidc)
7. [Análisis de Seguridad](#análisis-de-seguridad)
8. [Casos de Uso](#casos-de-uso)
9. [Troubleshooting](#troubleshooting)

---

## Introducción

### ¿Por qué es importante el hardening?

GitHub Actions ejecuta código en máquinas virtuales de GitHub, teniendo acceso a:
- Tu código fuente
- Tus secrets
- Tu infraestructura (vía credenciales)
- Tu identidad de organización

Un ataque exitoso podría resultar en:
- 🔓 Robo de credenciales
- 📊 Exfiltración de datos
- 🚀 Deploy no autorizado
- ⛓️ Compromiso de la cadena de suministro

### Principios de Seguridad

| Principio | Aplicación |
|-----------|-----------|
| **Least Privilege** | Solo permisos necesarios |
| **Immutability** | Code y actions no pueden cambiar |
| **Separation of Concerns** | Cada job hace una cosa |
| **Audit Trail** | Todo se registra |
| **Defense in Depth** | Múltiples capas de protección |

---

## Permisos

### Niveles de Permisos

#### 1. Global (Workflow level)

```yaml
# Mínimos globales
permissions:
  contents: read
  id-token: write

jobs:
  # ...
```

**Aplicación:**
- Se hereda a todos los jobs
- Puede ser sobrescrito por job

#### 2. Job-specific

```yaml
jobs:
  analyze:
    permissions:
      contents: read
      security-events: write
      # ✅ Este job NO tiene acceso a 'id-token'
```

**Comportamiento:**
- Sobrescribe los permisos globales
- Debe especificarse explícitamente

#### 3. Implicit (No especificado)

```yaml
# ❌ INSEGURO: Sin permissions especificado
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Tengo TODOS los permisos!"
```

**Riesgo:**
- Token tiene acceso total (GITHUB_TOKEN scope)
- Puede escribir a repositorio, modificar PRs, etc.

### Permisos Disponibles

```yaml
permissions:
  actions: read|write|admin              # Workflows y runs
  checks: read|write|admin              # Resultados de checks
  contents: read|write|admin            # Código del repositorio
  deployments: read|write|admin         # Deployments
  id-token: read|write                  # OIDC token
  issues: read|write|admin              # Issues
  packages: read|write|admin            # Packages
  pages: read|write|admin               # Pages
  pull-requests: read|write|admin       # PRs
  repository-projects: read|write|admin # Projects
  security-events: read|write           # Code scanning
  statuses: read|write                  # Status checks
```

### Buenas Prácticas

❌ **NUNCA HAGAS ESTO:**
```yaml
permissions: write-all  # ❌ TODO acceso
```

✅ **HAZ ESTO:**
```yaml
permissions:
  contents: read
  id-token: write
  
jobs:
  deploy:
    permissions:
      id-token: write
      contents: read
```

### Matriz de Referencia

| Acción | Permisos Necesarios |
|--------|-------------------|
| Leer código | `contents: read` |
| Subir cambios | `contents: write` |
| Crear issues | `issues: write` |
| Crear PRs | `pull-requests: write` |
| Deploy | `deployments: write` |
| OIDC | `id-token: write` |
| Security scanning | `security-events: write` |

---

## Supply Chain

### Version Pinning

#### El Problema

```yaml
- uses: actions/checkout@v4  # ¿QUÉ ES v4?
```

**¿Qué sucede?**
1. GitHub resuelve `v4` a la última release
2. Si la action se actualiza, tu código cambia
3. El atacante compromete la action → Tu CI está comprometida

#### La Solución

```yaml
# Formato: owner/action@SHA-256
- uses: actions/checkout@2541b1294d2704b0964813337f33b291d3f8596b
```

**Beneficios:**
- El SHA es inmutable
- Siempre ejecutas exactamente el mismo código
- Cambios intencionales vía Dependabot

### Cómo Obtener SHAs

#### Opción 1: GitHub CLI
```bash
$ gh api repos/actions/checkout/commits -q '.[] | select(.message | contains("v4")) | .sha' | head -1
2541b1294d2704b0964813337f33b291d3f8596b
```

#### Opción 2: Git
```bash
$ git ls-remote https://github.com/actions/checkout refs/heads/v4
2541b1294d2704b0964813337f33b291d3f8596b  refs/heads/v4
```

#### Opción 3: Dependabot (Recomendado)
Dependabot automáticamente:
1. Detecta nuevas versiones
2. Crea PRs con SHAs actualizados
3. Te notifica para revisar

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

### Auditoría de SHAs

```bash
# Ver SHAs en uso
grep -r "uses:" .github/workflows/ | grep -o "@[a-f0-9]*"

# Comparar con versiones actuales
for sha in $(grep -o "@[a-f0-9]*" .github/workflows/*.yml | cut -d: -f2 | sort -u); do
  echo "SHA: $sha"
  # Verificar si es válido
done
```

---

## Secrets

### Jerarquía de Scope

```
┌─────────────────────────────────────────────┐
│         Organization Secrets                │
│     (todos los repos pueden leer)           │
│                                             │
│  ┌───────────────────────────────────────┐  │
│  │    Repository Secrets                 │  │
│  │  (solo este repo puede leer)          │  │
│  │                                       │  │
│  │  ┌─────────────────────────────────┐  │  │
│  │  │  Environment Secrets             │  │  │
│  │  │ (solo en cierto environment)    │  │  │
│  │  └─────────────────────────────────┘  │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### Decisiones de Placement

#### Nivel: Repository
```yaml
# Credenciales genéricas (accesibles en cualquier workflow)
- REGISTRY_USERNAME
- REGISTRY_PASSWORD
- EXTERNAL_API_KEY
```

#### Nivel: Environment
```yaml
# Credenciales específicas de environment
- DEPLOY_KEY_STAGING (environment: staging)
- DEPLOY_KEY_PROD (environment: production)
- DB_PASSWORD_STAGING (environment: staging)
- DB_PASSWORD_PROD (environment: production)
```

#### Nivel: Organization
```yaml
# Compartidas entre repositorios (USAR CON CUIDADO)
- SLACK_WEBHOOK_GENERAL
- SENTRY_AUTH_TOKEN
- DATADOG_API_KEY
```

### Uso en Workflows

```yaml
# ✅ CORRECTO: Secret en variable de entorno
steps:
  - name: Deploy
    env:
      DEPLOY_KEY: ${{ secrets.DEPLOY_KEY_PROD }}
    run: deploy.sh  # Accede via $DEPLOY_KEY
    # Log output: *** (está mascara automáticamente)

# ❌ INCORRECTO: Secret en output
- name: Bad practice
  run: echo ${{ secrets.API_KEY }}  # ⚠️ Output será mascarado pero NO es seguro
  
# ❌ MUY MALO: Secret en el código
- name: Very bad
  run: |
    curl -H "Authorization: Bearer secret123" \
      https://api.example.com
    # ❌ Secret en el repositorio SIEMPRE
```

### Masking de Secrets

GitHub automáticamente oculta secrets que:
1. Aparecen en logs como `***`
2. Están en variables de entorno
3. Se pasan a comandos shell

```bash
# Ejemplo
echo ${{ secrets.API_KEY }}

# Output:
# ***
```

### Rotación de Secrets

```bash
# Procedimiento recomendado
1. Crear nuevo secret
   Settings > Secrets > New repository secret
   
2. Actualizar workflows
   Find & replace: OLD_SECRET → NEW_SECRET
   
3. Pruebas en staging
   
4. Deploy a producción
   
5. Revocar secret antiguo en proveedor
   
6. Esperar 90 días y eliminar en GitHub
```

---

## Environments

### Configuración de Staging

```
Settings > Environments > New Environment
│
├─ Name: staging
├─ Deployment branches: develop
├─ Required reviewers: (dejar vacío)
├─ Restrict who can create deployments: (unchecked)
└─ Auto-inactive deadline: 30 days
```

**Resultado:**
- Deployments automáticos a staging
- Sin aprobación requerida
- Logs disponibles para auditoría

### Configuración de Production

```
Settings > Environments > New Environment
│
├─ Name: production
├─ Deployment branches: main
├─ Required reviewers: 2
├─ Restrict who can create deployments: checked
│   ├─ Select deployment branches: main
│   └─ Environment protection rules:
│       └─ Require branch to be up to date before deploying
└─ Auto-inactive deadline: 30 days
```

**Resultado:**
- Aprobación de 2 personas requerida
- Solo rama `main` puede deployar
- Rama debe estar actualizada

### Uso en Workflow

```yaml
deploy:
  environment:
    name: production
    url: https://api.example.com  # URL de deployment
  runs-on: ubuntu-latest
  steps:
    # Estos pasos solo se ejecutan si fue aprobado
    - name: Deploy
      run: ./deploy.sh
```

**Flujo:**
1. Workflow llega a job `deploy`
2. GitHub espera aprobación (máximo 30 días)
3. Reviewer va a: Repository > Deployments > Pending
4. Reviewer verifica y aprueba
5. Job continúa con credenciales de `production` environment

### Aprobadores y Permisos

```yaml
# Configuración automática
Required reviewers:
  - @security-team
  - @devops-lead
```

**Quién puede aprobar:**
- Miembros del repo con permisos `write` o `admin`
- Nombrados explícitamente como reviewers
- NO: el autor del PR/commit

---

## OIDC

### Problema: Secrets Hardcodeados

```
┌──────────────────────────────┐
│   GitHub Repository          │
│                              │
│  [AWS_ACCESS_KEY_ID]  ❌    │  Hardcodeado
│  [AWS_SECRET_ACCESS_KEY]    │  Long-lived
│                              │  Compartido entre workflows
│                              │  Difícil de rotar
└──────────────────────────────┘
         │
         │ Riesgo: Si el repo es comprometido,
         │ la credencial es comprometida PARA SIEMPRE
         ▼
┌──────────────────────────────┐
│   AWS Account                │
│   ❌ Acceso total            │
│   ❌ Por 90 años (expiration)
└──────────────────────────────┘
```

### Solución: OIDC

```
GitHub Actions  ──(create JWT)──>  GitHub
                                    │
                                    │ (JWT signed with GitHub key)
                                    ▼
                              AWS OIDC Provider
                              (Validates JWT)
                                    │
                                    │ (Trust relationship: only main branch)
                                    ▼
                              Generate credentials
                              (STS AssumeRoleWithWebIdentity)
                                    │
                                    │ (short-lived: 15 min)
                                    ▼
                            Return to GitHub Actions
                                    │
                                    ▼ (No secrets in repo!)
                            Execute deploy (AWS CLI)
```

### JWT Payload

```json
{
  "iss": "https://token.actions.githubusercontent.com",
  "sub": "repo:owner/repo:ref:refs/heads/main",
  "aud": "sts.amazonaws.com",
  "iat": 1694765523,
  "exp": 1694766423,
  "ref": "refs/heads/main",
  "sha": "abc123def456...",
  "repository": "owner/repo",
  "repository_owner": "owner",
  "run_id": "4817262966",
  "run_number": "42",
  "runner_environment": "github-hosted",
  "actor": "github-user",
  "environment": "production"
}
```

### Configuración en AWS

#### 1. Crear OIDC Provider

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 1234567890abcdef... \
  --region us-east-1
```

#### 2. Crear Role

```bash
aws iam create-role \
  --role-name github-actions-prod \
  --assume-role-policy-document '{
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
            "token.actions.githubusercontent.com:sub": "repo:owner/repo:ref:refs/heads/main"
          }
        }
      }
    ]
  }'
```

#### 3. Adjuntar Policies

```bash
aws iam attach-role-policy \
  --role-name github-actions-prod \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPushOnly
```

#### 4. Usar en Workflow

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/github-actions-prod
    aws-region: us-east-1
    role-duration-seconds: 900  # 15 minutos
```

### Comparativa: Secrets vs OIDC

| Aspecto | Secrets | OIDC |
|--------|---------|------|
| **Almacenamiento** | Repositorio (encrypted) | No almacenado |
| **Duración** | 1-2 años | 15 minutos |
| **Revocación** | Manual | Automática (expira) |
| **Rotación** | Manual | Automática |
| **Contexto** | Ninguno | Incluido en JWT |
| **Auditoría** | Logs de acceso | JWT contiene contexto |
| **Riesgo si leak** | Comprometido por años | Inofensivo (expirado) |
| **Escalabilidad** | 1 secret = todos los workflows | 1 role = múltiples workflows |

### Casos de Uso

| Caso | Recomendación |
|------|---------------|
| Registry credentials | OIDC + AssumeRole |
| Cloud provider auth | OIDC (AWS, Azure, GCP) |
| Database credentials | Secrets en environment |
| External API keys | Secrets en repository |
| Service-to-service | OIDC si es posible |

---

## Análisis de Seguridad

### Trivy (Config Scanning)

```yaml
- uses: aquasecurity/trivy-action@v0.24.0
  with:
    scan-type: 'config'
    scan-ref: '.github/workflows'
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'HIGH,CRITICAL'

- uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

**Detecta:**
- Permisos excesivos
- Actions sin pin
- Secretos en código
- Configuraciones inseguras

### Snyk (Dependency Scanning)

```yaml
- name: Snyk scan
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high
```

**Detecta:**
- Dependencias vulnerables
- Supply chain vulnerabilities
- Dependency confusion

### SLSA (Supply Chain Security)

```yaml
- uses: slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml@v1.10.0
  with:
    image: ghcr.io/example/app:latest
    registry-username: ${{ github.actor }}
  secrets:
    registry-password: ${{ secrets.GITHUB_TOKEN }}
```

**Proporciona:**
- Provenance (quién construyó, cuándo, dónde)
- Attestations (prueba de compilación)
- Material de construcción (inputs, outputs)

---

## Casos de Uso

### Caso 1: Build & Push a Registry

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
      packages: write
    steps:
      - uses: actions/checkout@2541b1294d2704b0964813337f33b291d3f8596b

      # OIDC para ECR
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions
          aws-region: us-east-1

      - uses: docker/build-push-action@94f8287a70068e39cd60a94a9ace2a56e857e5be
        with:
          push: true
          tags: 123456789012.dkr.ecr.us-east-1.amazonaws.com/app:${{ github.sha }}
```

### Caso 2: Deploy a Kubernetes

```yaml
jobs:
  deploy:
    environment: production
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@2541b1294d2704b0964813337f33b291d3f8596b

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-prod
          aws-region: us-east-1

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig \
            --name prod-cluster \
            --region us-east-1

      - uses: kubernetes-actions/kubectl@v4
        with:
          kubeconfig: ${{ env.KUBECONFIG }}
          args: apply -k overlays/prod/
```

### Caso 3: Approve & Deploy

```yaml
jobs:
  deploy:
    environment:
      name: production
      url: https://api.example.com
    runs-on: ubuntu-latest
    steps:
      - name: Deploy (requires approval)
        run: ./deploy.sh
```

---

## Troubleshooting

### Problema: "OIDC token request failed"

```
Error: OIDC token request failed with status code 403
```

**Causas comunes:**
1. Rol no existe o ARN es incorrecto
2. Trust relationship no incluye tu repositorio
3. Ambiente de runner no está autenticado

**Solución:**
```bash
# Verificar role existe
aws iam get-role --role-name github-actions

# Verificar trust relationship
aws iam get-role --role-name github-actions --query 'Role.AssumeRolePolicyDocument'
```

### Problema: "Permission denied"

```
Error: fatal: could not write to '.git/config'
```

**Causa:**
Permisos `contents: write` pero intenta push

**Solución:**
```yaml
permissions:
  contents: read  # Solo lectura para CI
  # En deploy job: contents: write si es necesario
```

### Problema: "Secret not available"

```
Error: secret not found
```

**Causas:**
1. Secret no configurado en environment
2. Job no especifica environment
3. Nombre del secret es incorrecto

**Solución:**
```yaml
# Asegurar que el job especifica environment
deploy:
  environment: production
  # Ahora acceso a secrets.DEPLOY_KEY_PROD
  steps:
    - run: deploy.sh
      env:
        KEY: ${{ secrets.DEPLOY_KEY_PROD }}
```

---

## Checklist Final

- [ ] Todos los workflows tienen `permissions:` definido
- [ ] Todos los actions están pinned a SHA
- [ ] No hay secrets hardcodeados
- [ ] Secrets están en scope correcto
- [ ] Staging environment sin aprobación
- [ ] Production environment requiere 2 aprobaciones
- [ ] OIDC configurado si usas cloud
- [ ] Trivy/Snyk configurados
- [ ] Logs están disponibles para auditoría
- [ ] Documentación actualizada

---

**Última actualización**: 2026-05-29
