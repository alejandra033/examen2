# Security Policy

## Resumen de la Estrategia de Seguridad

Este documento describe la política de seguridad implementada en los workflows de GitHub Actions.

---

## 1️⃣ Principios Fundamentales

### Principle of Least Privilege (PoLP)
- Cada job tiene **solo** los permisos que necesita
- Se revoca cualquier permiso no utilizado
- Los tokens tienen el tiempo de vida más corto posible

### Defense in Depth
- Múltiples capas de validación y control
- Fallo seguro: si hay duda, se bloquea

### Auditoría y Trazabilidad
- Todos los eventos se registran
- Imposible negar la responsabilidad (non-repudiation)

---

## 2️⃣ Permisos (Permissions)

### Global Permissions (Workflow Level)

```yaml
permissions:
  contents: read        # Solo lectura del código
  security-events: write  # Solo para reporte de seguridad
  id-token: write      # Necesario para OIDC
```

### Job-Specific Permissions

Cada job define sus propios permisos, **sobrescribiendo** los globales:

```yaml
jobs:
  analysis:
    permissions:
      contents: read
      security-events: write
      # No tiene: packages, deployments, pull-requests, issues, etc.
```

### Permisos Prohibidos en Producción

- ❌ `write` en `contents` (push a repositorio)
- ❌ `admin` en cualquier ámbito
- ❌ `write` en `secrets` (modificar secrets)
- ❌ Permisos de `pull-requests: write` en CI

---

## 3️⃣ Control de Supply Chain

### Version Pinning (Immutable References)

#### ✅ SEGURO: SHA-256 exacto
```yaml
- uses: actions/checkout@2541b1294d2704b0964813337f33b291d3f8596b
```

**Ventajas:**
- La action es idéntica siempre
- Imposible que cambiar sin que lo sepas
- Protege contra compromiso de la account

#### ❌ INSEGURO: Tag mutable
```yaml
- uses: actions/checkout@v4  # ¿QUÉ VERSIÓN ES?
```

**Riesgos:**
- La action puede cambiar sin aviso
- Supply chain attack posible
- No auditabilidad

### Cómo obtener el SHA

```bash
# Opción 1: Usar GitHub CLI
gh api /repos/{owner}/actions/{action}/commits --jq '.[0].commit_id'

# Opción 2: URL de git
git ls-remote https://github.com/actions/checkout v4
```

### Renovación de Versiones

Use **Dependabot** para mantener las actions actualizadas:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    pull-request-branch-name:
      separator: "/"
```

---

## 4️⃣ Gestión de Secrets

### Jerarquía de Secrets (Scope)

```
1. Repository Secrets (solo este repo)
   - REGISTRY_USERNAME
   - REGISTRY_PASSWORD

2. Organization Secrets (todos los repos de la org)
   - SLACK_WEBHOOK_GENERAL

3. Environment Secrets (solo en cierto environment)
   - DEPLOY_KEY_STAGING (env: staging)
   - DEPLOY_KEY_PROD (env: production)
```

### Reglas de Oro

1. **Nunca hardcodees secrets**
   ```yaml
   ❌ password: "my-secret-password"
   ✅ password: ${{ secrets.PASSWORD }}
   ```

2. **Usa variables de entorno para mascarar**
   ```yaml
   env:
     SECRET: ${{ secrets.MY_SECRET }}  # Se oculta en logs
   run: echo $SECRET  # Output será [REDACTED]
   ```

3. **Rotación regular**
   - Cada 30-90 días
   - Después de una fuga
   - Después de que alguien abandone el equipo

4. **Usa service accounts**
   ```yaml
   # Mejor: cuenta de servicio con permisos limitados
   username: github-actions-bot
   ```

### Tipos de Secrets por Nivel

#### Repository Level (REGISTRY_USERNAME, REGISTRY_PASSWORD)
- Accesibles en cualquier workflow del repositorio
- Para credenciales genéricas (registry, external services)

#### Environment Level (DEPLOY_KEY_STAGING, DEPLOY_KEY_PROD)
- Solo disponibles cuando usas `environment:`
- Diferentes credenciales por environment
- Ideales para deploy keys

#### Organization Level (SLACK_WEBHOOK_GENERAL)
- Compartidos entre múltiples repositorios
- Usar con cuidado (impacto alto)

---

## 5️⃣ Environments

### Configuración Recomendada

#### Staging
```
Settings > Environments > Create Environment

Name: staging
Required reviewers: Ninguno (deployment automático)
Deployment branches: develop
Max reviewers: N/A
Auto-inactive deadline: 30 días
```

#### Production
```
Settings > Environments > Create Environment

Name: production
Required reviewers: 2 personas (mínimo)
Deployment branches: main
Max reviewers: N/A
Auto-inactive deadline: 30 días
Restrict who can trigger deployments: main branch only
```

### Integración en Workflows

```yaml
deploy:
  environment:
    name: production
    url: https://api.example.com
```

**Efectos:**
1. Se requiere aprobación antes de ejecutar
2. Se envía deployment status a GitHub API
3. Se puede rastrear quién aprobó
4. Las credenciales de environment se proporcionan solo si aprobado

---

## 6️⃣ OIDC (OpenID Connect)

### El Problema Tradicional (con Secrets)

```
GitHub Actions → (credenciales hardcodeadas) → AWS
                    ❌ Expuestas en repositorio
                    ❌ Long-lived (válidas años)
                    ❌ Difícil de rotar
```

### La Solución OIDC

```
GitHub Actions → (JWT token firmado por GitHub) → AWS
                    ✅ No almacenado en repositorio
                    ✅ Short-lived (15 minutos)
                    ✅ Ligado a contexto específico
                    ✅ Imposible de reutilizar en otro contexto
```

### Cómo Funciona OIDC en GitHub Actions

1. **GitHub crea un JWT token** con información del contexto:
   ```json
   {
     "iss": "https://token.actions.githubusercontent.com",
     "sub": "repo:owner/repo:ref:refs/heads/main",
     "aud": "https://iam.amazonaws.com/github",
     "iat": 1234567890,
     "exp": 1234568790,
     "ref": "refs/heads/main",
     "sha": "abc123...",
     "repository": "owner/repo",
     "actor": "github-user"
   }
   ```

2. **GitHub Actions intercambia el JWT por credenciales temporales de AWS**

3. **Las credenciales expiran en 15 minutos**

### Configuración en AWS

#### 1. Crear OIDC Provider
```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --thumbprint-list 1234567890abcdef...
```

#### 2. Crear Role de Confianza
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com"
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
```

#### 3. Limitar por Rama
```json
"token.actions.githubusercontent.com:sub": "repo:owner/repo:ref:refs/heads/main"
```

### Uso en Workflow

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::ACCOUNT:role/github-actions
    aws-region: us-east-1
    role-duration-seconds: 900  # 15 minutos
```

### Beneficios de OIDC

| Aspecto | Secrets | OIDC |
|--------|---------|------|
| **Almacenamiento** | Repositorio | Solo JWT en memoria |
| **Duración** | Meses/años | 15 minutos |
| **Revocación** | Manual | Automática |
| **Auditoría** | Básica | Completa (JWT contiene contexto) |
| **Riesgo de fuga** | Alto | Muy bajo |
| **Complejidad** | Baja | Media |

---

## 7️⃣ Análisis de Seguridad Automático

### Herramientas Integradas

1. **Trivy** - Escaneo de configuraciones
   ```yaml
   - uses: aquasecurity/trivy-action@v0.24.0
     with:
       scan-type: 'config'
       scan-ref: '.github/workflows'
   ```

2. **Snyk** - Análisis de dependencias
   ```yaml
   - name: Snyk scan
     env:
       SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
     run: snyk test
   ```

3. **SLSA Framework** - Supply chain security
   ```yaml
   - uses: slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml@v1.10.0
   ```

### Resultados

- Automáticamente subidos a GitHub Security tab
- SARIF format para integración con IDEs
- Bloqueo de merges si hay vulnerabilidades críticas

---

## 8️⃣ Auditoría y Compliance

### Registros Disponibles

1. **GitHub Actions Logs**
   - Settings > Actions > Logs
   - 90 días de retencion

2. **Deployment History**
   - Repository > Deployments
   - Quién aprobó, cuándo, qué versión

3. **Security Logs**
   - Organization > Security > Logs
   - Acceso a secrets, aprobaciones, cambios de permisos

4. **Audit Log**
   - Organization > Settings > Audit log
   - Todas las acciones de administración

### Extracción de Logs

```bash
# Descargar logs de una ejecución
gh run download <run-id> -D ./logs

# Ver logs en tiempo real
gh run view <run-id> --log
```

---

## 9️⃣ Checklist de Seguridad

### Pre-Mergeo
- [ ] Todos los actions tienen SHA pinned
- [ ] `permissions:` definido (mínimos)
- [ ] No hay secrets hardcodeados
- [ ] Análisis de seguridad pasó
- [ ] Cambios revisados por 2 personas

### Pre-Deploy a Staging
- [ ] Tests pasaron
- [ ] Build exitoso
- [ ] OIDC configurado si aplica

### Pre-Deploy a Production
- [ ] Todos los pasos anteriores + staging deploy exitoso
- [ ] 2 aprobaciones requeridas
- [ ] Health checks configurados
- [ ] Rollback automático activado
- [ ] Alertas configuradas

---

## 🔟 Reportar Vulnerabilidades

**NO** abras un issue publicamente si encuentras una vulnerabilidad.

1. Ve a Settings > Security > Report a vulnerability
2. O envía email a: security@example.com
3. Espera confirmación (máximo 48 horas)
4. Coordina la divulgación

---

## Referencias

- [GitHub Actions Security Guides](https://docs.github.com/actions/security-guides)
- [AWS OIDC Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_oidc.html)
- [SLSA Framework](https://slsa.dev/)
- [CWE-829: Supply Chain](https://cwe.mitre.org/data/definitions/829.html)

---

**Última actualización**: 2026-05-29
**Versión**: 2.0 (Enterprise)
