# Estrategia de Secrets en GitHub Actions

## 📌 Índice

1. [Principios Fundamentales](#principios-fundamentales)
2. [Niveles de Scope](#niveles-de-scope)
3. [Matriz de Decisión](#matriz-de-decisión)
4. [Implementación Práctica](#implementación-práctica)
5. [Rotación de Secrets](#rotación-de-secrets)
6. [Auditoría y Compliance](#auditoría-y-compliance)
7. [Seguridad en Logs](#seguridad-en-logs)

---

## Principios Fundamentales

### 1. Principio de Menor Privilegio

Cada secret debería:
- Conceder **solo** los permisos necesarios
- Ser accesible **solo** donde se necesita
- Expirar **lo antes posible**

```yaml
# ❌ MALO: Secret con acceso total
- name: Deploy
  env:
    AWS_SECRET: ${{ secrets.AWS_SECRET_ADMIN }}  # Admin access
  run: deploy.sh

# ✅ BUENO: Secret con permisos específicos
- name: Deploy
  env:
    AWS_SECRET: ${{ secrets.AWS_SECRET_DEPLOY }}  # Deploy only
  run: deploy.sh
```

### 2. Separación de Responsabilidades

Diferentes tipos de secrets para diferentes propósitos:

```
- Registry credentials (Docker, npm)
  └─ Acceso a artifacts

- Deployment credentials (AWS, Azure)
  └─ Acceso a infraestructura

- External APIs (Slack, Sentry)
  └─ Notificaciones y monitoreo

- Database credentials
  └─ Acceso a datos
```

### 3. Seguridad en Capas

```
┌─────────────────────────────────────┐
│  Capa 1: No exista el secret         │
│  (Usar OIDC, variables, etc.)       │
├─────────────────────────────────────┤
│  Capa 2: Si existe, limitar scope   │
│  (Repository → Environment)         │
├─────────────────────────────────────┤
│  Capa 3: Si se comparte, auditar    │
│  (Logs, CloudTrail, etc.)          │
├─────────────────────────────────────┤
│  Capa 4: Si se filtra, rotar rápido │
│  (Revocar en proveedor)            │
└─────────────────────────────────────┘
```

---

## Niveles de Scope

### Nivel 1: Repository Secrets

```
GitHub Settings > Secrets and variables > Actions > Repository secrets
```

**Características:**
- Accesibles en TODOS los workflows del repositorio
- Accesibles en TODAS las ramas
- No protegidos por environment

**Casos de Uso:**
```
✅ Credentials genéricas (registry, APIs comunes)
✅ Que no cambían entre environments
❌ Credenciales de production
❌ Deploy keys específicas de environment
```

**Ejemplo:**
```yaml
# Estos secrets están disponibles globalmente
REGISTRY_USERNAME: "github-bot"
REGISTRY_PASSWORD: "ghp_xxxxxx"
SNYK_TOKEN: "xxxx"
```

### Nivel 2: Organization Secrets

```
GitHub Settings (Org) > Secrets and variables > Actions > Organization secrets
```

**Características:**
- Accesibles en TODOS los repositorios de la organización
- Impacto organizacional alto
- Control centralizado

**Casos de Uso:**
```
✅ Webhooks compartidos (Slack, etc.)
✅ Tokens de servicios transversales
✅ Configuración de organización
❌ Credenciales de repositorio específico
❌ Cualquier secret sensible (usar sparingly)
```

**Ejemplo:**
```
SLACK_WEBHOOK_GENERAL: "https://hooks.slack.com/..."
SENTRY_AUTH_TOKEN: "xxxx"
DATADOG_API_KEY: "xxxx"
```

### Nivel 3: Environment Secrets

```
GitHub Settings > Environments > [environment-name] > Environment secrets
```

**Características:**
- Accesibles SOLO en jobs que especifican `environment:`
- Diferentes valores por environment
- Máximo control y separación

**Casos de Uso:**
```
✅ Deploy credentials (diferentes por environment)
✅ Database credentials (staging vs prod)
✅ API keys específicas por environment
✅ Tokens de servicios por environment
```

**Ejemplo: Staging**
```
environment: staging

Secrets:
- AWS_ROLE_ARN: arn:aws:iam::DEV-ACCOUNT:role/...
- DB_PASSWORD: staging-password
- SLACK_WEBHOOK: staging-webhook
```

**Ejemplo: Production**
```
environment: production

Secrets:
- AWS_ROLE_ARN: arn:aws:iam::PROD-ACCOUNT:role/...
- DB_PASSWORD: prod-password (rotado mensualmente)
- SLACK_WEBHOOK: prod-webhook
```

---

## Matriz de Decisión

### ¿Dónde guardar cada secret?

```
START: ¿Quiero usar OIDC?
│
├─ SÍ → Configurar OIDC (mejor opción)
│        └─ No necesitas guardar credenciales
│
└─ NO → ¿Es específico de un environment?
         │
         ├─ SÍ → Environment Secret
         │        ├─ Staging: environment: staging
         │        └─ Production: environment: production
         │
         └─ NO → ¿Es específico del repositorio?
                  │
                  ├─ SÍ → Repository Secret
                  │        (accesible globalmente en el repo)
                  │
                  └─ NO → ¿Es transversal a múltiples repos?
                           │
                           ├─ SÍ → Organization Secret
                           │        (accesible en toda la org)
                           │
                           └─ NO → ¿Debería existir?
                                    └─ Probablemente no
```

### Ejemplos Prácticos

#### Ejemplo 1: Deploy a AWS

```yaml
# MEJOR: Usar OIDC
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}  # En environment secret
    aws-region: us-east-1

# ALTERNATIVA: Si no OIDC disponible
- name: Deploy
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_KEY }}     # En environment secret
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET }} # En environment secret
  run: deploy.sh
```

**Ubicación:**
```
Production Environment Secret:
- AWS_ROLE_ARN (si OIDC)
- AWS_KEY (si no OIDC)
- AWS_SECRET (si no OIDC)
```

#### Ejemplo 2: Push a Docker Registry

```yaml
- name: Login to Docker
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.REGISTRY_USERNAME }}    # Repository Secret
    password: ${{ secrets.REGISTRY_PASSWORD }}    # Repository Secret
```

**Ubicación:**
```
Repository Secret (acceso global):
- REGISTRY_USERNAME
- REGISTRY_PASSWORD

Razón: Son credenciales genéricas para cualquier push
```

#### Ejemplo 3: Notificación a Slack

```yaml
- name: Notify Slack
  uses: slackapi/slack-github-action@v1
  with:
    payload: ${{ env.SLACK_PAYLOAD }}
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}  # ¿Dónde?
```

**Decisión:**
```
¿Es específico de un environment?
- Staging Slack → environment: staging
- Production Slack → environment: production
- Slack general → Organization Secret

En este ejercicio:
- Staging: environment: staging
- Production: environment: production
```

#### Ejemplo 4: Snyk Token

```yaml
- name: Run Snyk
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}  # ¿Dónde?
```

**Decisión:**
```
¿Es específico de un environment?
- NO: Se usa en todos los workflows (CI, staging, prod)

¿Es específico del repositorio?
- SÍ: Cada repo tiene su propio token de Snyk

Ubicación: Repository Secret
```

---

## Implementación Práctica

### Paso 1: Crear Estructura de Environments

#### GitHub Web UI

```
Settings > Environments > Create Environment

1. Crear "staging"
   - Name: staging
   - Deployment branches: develop, feature/*
   - Required reviewers: None
   - Auto-inactive deadline: 30 days

2. Crear "production"
   - Name: production
   - Deployment branches: main
   - Required reviewers: 2 personas
   - Auto-inactive deadline: 30 days
   - Restrict who can create deployments: main only
```

### Paso 2: Configurar Secrets

#### Via GitHub Web UI

```
Settings > Secrets and variables > Actions

REPOSITORY SECRETS:
  1. REGISTRY_USERNAME
     Value: github-bot
  
  2. REGISTRY_PASSWORD
     Value: ghp_xxxxxxxxxxxx
  
  3. SNYK_TOKEN
     Value: xxxxxxxxxxxx

ENVIRONMENT SECRETS (staging):
  1. AWS_ROLE_ARN
     Value: arn:aws:iam::123456789:role/github-actions-staging
  
  2. DB_PASSWORD
     Value: staging-db-password
  
  3. SLACK_WEBHOOK
     Value: https://hooks.slack.com/staging

ENVIRONMENT SECRETS (production):
  1. AWS_ROLE_ARN
     Value: arn:aws:iam::987654321:role/github-actions-prod
  
  2. DB_PASSWORD
     Value: prod-db-password
  
  3. SLACK_WEBHOOK
     Value: https://hooks.slack.com/production
```

#### Via GitHub CLI

```bash
# Repository Secrets
gh secret set REGISTRY_USERNAME --body "github-bot"
gh secret set REGISTRY_PASSWORD --body "ghp_xxxxxxxxxxxx"

# Environment Secrets
gh secret set AWS_ROLE_ARN --env staging --body "arn:aws:iam::123456789:role/staging"
gh secret set AWS_ROLE_ARN --env production --body "arn:aws:iam::987654321:role/production"
gh secret set DB_PASSWORD --env staging --body "staging-password"
gh secret set DB_PASSWORD --env production --body "prod-password"
```

### Paso 3: Usar en Workflows

```yaml
name: Deploy

on:
  push:
    branches: [ main, develop ]

permissions:
  contents: read
  id-token: write

jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    environment: staging  # ← Acceso a staging secrets
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@2541b1294d2704b0964813337f33b291d3f8596b
      
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}  # ← De environment
          aws-region: us-east-1

      - name: Deploy
        env:
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}  # ← De environment
        run: ./deploy-staging.sh

  deploy-production:
    if: github.ref == 'refs/heads/main'
    environment: production  # ← Acceso a production secrets
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@2541b1294d2704b0964813337f33b291d3f8596b
      
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}  # ← De environment
          aws-region: us-east-1

      - name: Deploy
        env:
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}  # ← De environment
        run: ./deploy-production.sh
```

---

## Rotación de Secrets

### Planificación

| Secret | Frecuencia | Motivo |
|--------|-----------|--------|
| API Keys | 90 días | Standard |
| Database Passwords | 60 días | Compliance |
| SSH Keys | 180 días | Si no está comprometida |
| Tokens de CI/CD | 30 días | Más criticidad |
| Después de leak | Inmediato | Emergencia |

### Proceso de Rotación

```
1. CREAR nuevo secret
   └─ Settings > Secrets > New Secret

2. ACTUALIZAR workflows
   └─ Reemplazar referencias antiguas
   └─ Commit a main (o desarrollar)
   └─ Esperar a que se ejecute correctamente

3. VERIFICAR funcionamiento
   └─ Ejecutar workflow con nuevo secret
   └─ Confirmar que funciona

4. REVOCAR secret antiguo
   └─ En el proveedor externo
   └─ (AWS, Azure, Docker Hub, etc.)

5. ELIMINAR secret en GitHub
   └─ Settings > Secrets > Delete
   └─ Esperar 30 días (GitHub los mantiene 30 días)

6. DOCUMENTAR
   └─ En CHANGELOG o changelog
   └─ Quién rotó, cuándo, por qué
```

### Automatización con GitHub CLI

```bash
#!/bin/bash
# rotate-secrets.sh

NEW_API_KEY="new-api-key-value"
ENVIRONMENT="production"

# Crear nuevo secret
gh secret set API_KEY --env $ENVIRONMENT --body "$NEW_API_KEY"

# Verificar
gh secret list --env $ENVIRONMENT | grep API_KEY

echo "✅ Secret rotated successfully"
echo "⚠️  Revoke old key in AWS/provider"
```

---

## Auditoría y Compliance

### Logs de Acceso a Secrets

GitHub registra:
- Quién accedió al secret
- Cuándo fue accedido
- En qué workflow fue usado
- Si fue exitoso o falló

**Ubicación:**
```
Organization > Settings > Audit log
└─ Filter by: secrets

Events logged:
- org.create_secret
- org.update_secret
- org.delete_secret
- repo.create_secret
- repo.update_secret
- repo.delete_secret
```

### Extracción de Logs

```bash
# Ver últimos 100 eventos de secrets
gh api "orgs/{org}/audit-log?phrase=secret" -q '.[] | 
  "\(.created_at) \(.action) \(.actor.login) \(.org)"'
```

### Compliance Requirements

#### SOC 2

- ✅ Secrets encriptados en reposo
- ✅ Secrets enmascarados en logs
- ✅ Acceso auditado
- ✅ Rotación regular documentada

#### HIPAA

- ✅ Separación por environment
- ✅ Aprobación de deployments a producción
- ✅ Logs con trazabilidad completa
- ✅ Encriptación en tránsito (HTTPS)

#### PCI-DSS

- ✅ No hardcodear secretos
- ✅ Acceso restringido (2FA en GitHub)
- ✅ Rotación cada 90 días
- ✅ Auditoría completa

---

## Seguridad en Logs

### Protección Automática

GitHub automáticamente oculta:

```yaml
# ✅ Estos son ocultados en logs
- run: echo ${{ secrets.API_KEY }}
  # Output: echo ***

- run: curl -H "Authorization: Bearer ${{ secrets.TOKEN }}"
  # Output: curl -H "Authorization: Bearer ***"
```

### Masking Manual

```yaml
# Si necesitas ocultar valores generados
- name: Generate credentials
  run: |
    CRED=$(aws sts get-session-token --query 'Credentials.AccessKeyId' --output text)
    echo "::add-mask::$CRED"  # ← Ocultar en logs
    echo "CRED=$CRED" >> $GITHUB_ENV
```

### Advertencias

❌ **NO hagas esto:**
```yaml
# ❌ Secret en la URL
- run: curl https://api.example.com?key=${{ secrets.API_KEY }}

# ❌ Secret en JSON visible
- run: echo "{ api_key: '${{ secrets.API_KEY }}' }"

# ❌ Secret impreso antes de usar
- run: echo ${{ secrets.API_KEY }}; deploy.sh
```

✅ **Haz esto:**
```yaml
# ✅ Secret en variable de entorno
- name: Deploy
  env:
    API_KEY: ${{ secrets.API_KEY }}
  run: deploy.sh  # Acceso via $API_KEY (oculto)

# ✅ Secret en opción de comando
- run: deploy.sh --key "${{ secrets.API_KEY }}"
  # Output no mostrará el valor
```

---

## Checklist Final

- [ ] Todos los secrets tienen nombres descriptivos
- [ ] Repository secrets solo para credenciales genéricas
- [ ] Environment secrets para deploy credentials
- [ ] Organization secrets para servicios transversales
- [ ] OIDC configurado para cloud providers
- [ ] Staging environment sin aprobación
- [ ] Production environment requiere 2 aprobaciones
- [ ] Rotación de secrets documentada
- [ ] Proceso de rotación automatizado si es posible
- [ ] Logs de auditoría revisados mensualmente
- [ ] Plan de contingencia si hay leak
- [ ] Equipo capacitado en política de secrets

---

**Última actualización**: 2026-05-29
