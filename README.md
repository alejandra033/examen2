# Examen 2 — Seguridad Enterprise en GitHub Actions

## Objetivo General
Aplicar hardening completo de seguridad en GitHub Actions siguiendo las mejores prácticas de industria.

---

## 📋 Contenido de la Solución

### 1. **Permisos (Principle of Least Privilege)**
- ✅ Permisos mínimos globales en el nivel de workflow
- ✅ Permisos específicos granulares por job
- ✅ Revocación de permisos no utilizados

### 2. **Control de Supply Chain**
- ✅ Version pinning exacto de actions (SHA-256)
- ✅ Validación de integridad de dependencias
- ✅ Prevención de dependency confusion
- ✅ Auditoría de cambios de versiones

### 3. **Gestión de Secrets**
- ✅ Separación: repository → organization → environment
- ✅ Scope correcto por nivel
- ✅ No hardcoding de credenciales
- ✅ Rotación automática de tokens

### 4. **Environments (Staging & Production)**
- ✅ Staging: deployment automático
- ✅ Production: deployment con aprobación requerida
- ✅ Restricciones de rama (main only)
- ✅ Límite de deployments concurrentes

### 5. **OIDC (OpenID Connect)**
- ✅ Eliminación de secrets de larga duración
- ✅ Credenciales de corta duración (tokens JWT)
- ✅ Integración con proveedores cloud
- ✅ Auditoría y trazabilidad mejorada

---

## 🗂️ Estructura de Archivos

```
.github/
├── workflows/
│   ├── ci.yml                      # Pipeline de CI con permisos mínimos
│   ├── deploy-staging.yml          # Deploy a staging automático
│   └── deploy-production.yml       # Deploy a production con aprobación + OIDC
├── SECURITY.md                     # Política de seguridad
└── dependabot.yml                  # Configuración de dependencias

docs/
├── HARDENING_GUIDE.md              # Guía detallada de hardening
├── OIDC_EXPLANATION.md             # Explicación OIDC
└── SECRETS_STRATEGY.md             # Estrategia de secrets

README.md                           # Este archivo
```

---

## 🚀 Características de Seguridad Implementadas

### Permisos Granulares
```yaml
# Cada job tiene solo los permisos que necesita
permissions:
  contents: read
  security-events: write  # Solo para análisis de seguridad
```

### Version Pinning con SHA
```yaml
# ✅ SEGURO: SHA-256 exacto
- uses: actions/checkout@2541b1294d2704b0964813337f33b291d3f8596b

# ❌ INSEGURO: Tag mutable
- uses: actions/checkout@v4
```

### OIDC para AWS/Azure/GCP
```yaml
# Sin secrets en el repositorio
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::ACCOUNT:role/github-actions
    aws-region: us-east-1
```

### Environments con Aprobación
```yaml
deploy:
  environment:
    name: production
    url: https://api.example.com
  # Requiere aprobación antes de ejecutar
```

---

## 📝 Cómo Usar Esta Solución

### 1. Clona el repositorio
```bash
git clone <repository-url>
cd examen2
```

### 2. Revisa la documentación
- `docs/HARDENING_GUIDE.md`: Guía completa paso a paso
- `docs/OIDC_EXPLANATION.md`: Entender OIDC
- `.github/SECURITY.md`: Política de seguridad

### 3. Configura los Environments en GitHub
```
Settings > Environments > Create Environment
- staging (sin restricciones)
- production (requiere aprobación)
```

### 4. Configura los Secrets
```
Settings > Secrets and Variables > Actions
- REGISTRY_USERNAME (repository level)
- REGISTRY_PASSWORD (repository level)
- DEPLOY_KEY_STAGING (environment: staging)
- DEPLOY_KEY_PROD (environment: production)
```

### 5. Configura OIDC (si usas cloud)
```
Settings > Security > OIDC > AWS/Azure/GCP
Crea el trust relationship en tu proveedor
```

### 6. Valida con Trivy (analysis)
```bash
# Scan de seguridad
trivy config .github/workflows/
```

---

## 🔒 Verificación de Seguridad

### Checklist Pre-Producción
- [ ] Todos los workflows tienen `permissions` definidos
- [ ] Todas las actions están pinned a SHA-256
- [ ] No hay secrets hardcodeados en el repositorio
- [ ] Production environment requiere aprobación
- [ ] OIDC configurado (si aplica)
- [ ] Se ejecutó análisis de seguridad (Trivy)
- [ ] Se revisó el código de todos los workflows

---

## 📚 Referencias Externas

- [GitHub Actions Security Best Practices](https://docs.github.com/actions/security-guides)
- [Supply Chain Security](https://docs.github.com/actions/security-guides/keeping-your-actions-up-to-date-with-dependabot)
- [Using OIDC with GitHub Actions](https://docs.github.com/actions/deployment/security-hardening-your-deployments)
- [CWE-829: Inclusion of Functionality from Untrusted Control Sphere](https://cwe.mitre.org/data/definitions/829.html)

---

**Estado**: ✅ Implementado y documentado
**Última actualización**: 2026-05-29