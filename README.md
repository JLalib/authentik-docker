# 🔐 Authentik Docker - SSO/IdP Moderna Autohospedada

[![GitHub Stars](https://img.shields.io/github/stars/goauthentik/authentik?style=for-the-badge&logo=github&label=Stars&color=yellow)](https://github.com/goauthentik/authentik)
[![Docker Pulls](https://img.shields.io/docker/pulls/goauthentik/server?style=for-the-badge&logo=docker&label=Docker%20Pulls&color=blue)](https://hub.docker.com/r/goauthentik/server)
[![License](https://img.shields.io/github/license/goauthentik/authentik?style=for-the-badge&color=green)](https://github.com/goauthentik/authentik/blob/main/LICENSE)
[![Version](https://img.shields.io/github/v/release/goauthentik/authentik?style=for-the-badge&label=Release&color=orange)](https://github.com/goauthentik/authentik/releases)

---

## 📋 Descripción general

**Authentik** es un **Identity Provider (IdP) open source moderno** que proporciona **SSO (Single Sign-On) profesional autohospedado** compatible con **SAML 2.0, OAuth2/OIDC, LDAP, RADIUS**. Es el reemplazo ideal para **Okta, Auth0, Entra ID, Ping Identity** en entornos self-hosted.

Con arquitectura **multi-tenant**, **flows visuales** para lógica de autenticación, **access policies**, **blueprints** (automatización), **reverse proxy integrado (outpost)**, **auditing completo** y escalabilidad nativa en **Docker/Kubernetes**. Todo bajo tu control, sin vendor lock-in.

> 📖 **Artículo original**: [Cómo instalar Authentik en Docker - SSO/IdP moderna autohospedada](https://genbyte.blogspot.com/2026/09/como-instalar-authentik-en-docker.html)

---

## ✨ Características principales

- 🔐 **Multi-protocolo SSO**: SAML 2.0, OAuth2, OIDC, LDAP, RADIUS en un solo IdP
- 🛡️ **Reverse proxy integrado (Outpost)**: Protege cualquier app legacy sin cambios de código
- 🏢 **Multi-tenant**: Múltiples organizations en una instalación, branding personalizable
- 🎨 **Visual Flow Editor**: Drag-drop para lógica de autenticación (condicional, MFA, risk-based)
- 📋 **Access Policies**: Auth condicional, geo-blocking, device trust, MFA enforcement
- ⚙️ **Blueprints**: Automatización, bulk config, Infrastructure-as-Code (YAML)
- 🔄 **Directory Sync**: LDAP, Active Directory, Google Workspace sync (usuarios + grupos)
- 🌐 **Social Login**: GitHub, Google, Apple, Discord, proveedores OIDC personalizados
- 🔑 **MFA + WebAuthn**: TOTP, backup codes, WebAuthn (FIDO2) - seguridad moderna
- 📊 **Audit Logging**: Todos los eventos trackeados (acciones usuario, auth, cambios policy)
- 📱 **App Library**: 500+ integraciones pre-configuradas para SSO instantáneo
- ☸️ **Kubernetes-native**: Helm chart oficial, escalable, cloud-ready, production-grade
- 📈 **24.7k+ GitHub Stars**: Desarrollo activo, 23.6k+ commits, MIT licensed

---

## 📋 Requisitos del sistema

- 🐳 **Docker & Docker Compose v2+**
- 💾 **RAM**: 2 GB mínimo (4 GB recomendado) - Python backend + Go outpost
- 💿 **Disco**: 10 GB - 50+ GB (según usuarios, audit logs)
- 🔌 **Puertos TCP**:
  - `9000` - Web UI
  - `9300` - LDAP outpost
  - `9400` - RADIUS outpost
- 🐘 **PostgreSQL 14+** (bundled o externo)
- 🔴 **Redis** (sesiones, caching - opcional pero recomendado)
- 🐍 **Python 3.11+** (bundled en imagen)
- 🐹 **Go 1.20+** (outpost proxy, bundled)
- 🔧 **Opcional**: Servidor LDAP/AD para directory sync
- 🏭 **Opcional**: PostgreSQL externo para HA production

> ⚠️ **Nota**: Authentik requiere recursos considerables. El audit logging puede llenar disco rápido - planifica retención. Para producción: PostgreSQL externo (replicación), Redis externo, múltiples réplicas. Kubernetes + Helm recomendado para escala.

---

## 🐳 Instalación

### Paso 1: `docker-compose.yml`

```yaml
version: '3.8'

services:
  authentik_postgres:
    image: postgres:16-alpine
    container_name: authentik-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_DB=authentik
      - POSTGRES_USER=authentik
      - POSTGRES_PASSWORD=changeme123
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U authentik"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - authentik_postgres:/var/lib/postgresql/data

  authentik_redis:
    image: redis:8-alpine
    container_name: authentik-redis
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    volumes:
      - authentik_redis:/data

  authentik_server:
    image: ghcr.io/goauthentik/server:latest
    container_name: authentik-server
    restart: unless-stopped
    environment:
      - AUTHENTIK_POSTGRESQL__HOST=authentik_postgres
      - AUTHENTIK_POSTGRESQL__NAME=authentik
      - AUTHENTIK_POSTGRESQL__USER=authentik
      - AUTHENTIK_POSTGRESQL__PASSWORD=changeme123
      - AUTHENTIK_REDIS__HOST=authentik_redis
      - AUTHENTIK_SECRET_KEY=$(openssl rand -hex 32)
    ports:
      - "9000:9000"
    depends_on:
      authentik_postgres:
        condition: service_healthy
      authentik_redis:
        condition: service_healthy
    command: server

  authentik_worker:
    image: ghcr.io/goauthentik/server:latest
    container_name: authentik-worker
    restart: unless-stopped
    environment:
      - AUTHENTIK_POSTGRESQL__HOST=authentik_postgres
      - AUTHENTIK_POSTGRESQL__NAME=authentik
      - AUTHENTIK_POSTGRESQL__USER=authentik
      - AUTHENTIK_POSTGRESQL__PASSWORD=changeme123
      - AUTHENTIK_REDIS__HOST=authentik_redis
      - AUTHENTIK_SECRET_KEY=$(openssl rand -hex 32)
    depends_on:
      authentik_postgres:
        condition: service_healthy
      authentik_redis:
        condition: service_healthy
    command: worker

volumes:
  authentik_postgres:
  authentik_redis:
```

### Paso 2: Generar SECRET_KEY

```bash
# Generar secret aleatorio
openssl rand -hex 32

# Copiar y reemplazar AUTHENTIK_SECRET_KEY en compose (ambos servicios)
```

### Paso 3: Iniciar Authentik

```bash
# Guardar como docker-compose.yml
docker compose up -d

# Espera ~20 segundos para migrations
docker compose logs -f authentik_server

# Debería ver "Starting application server" cuando listo
```

---

## ⚙️ Configuración

1. **Acceder a la UI**: Abre `http://localhost:9000` (o IP del servidor: `http://192.168.1.100:9000`)
2. **Setup Wizard**: El asistente inicial crea el usuario admin (email + password)
3. **Dashboard Admin**: Accede a la administración completa tras login
4. **Configurar LDAP Sync** (opcional): Admin → Directory Sync → LDAP
5. **Configurar Social Logins** (opcional): Admin → Sources → Create (GitHub, Google, etc.)
6. **Crear Applications**: Admin → Applications → Create (SAML, OAuth2, OIDC)
7. **Setup Outpost Proxy** (opcional): Admin → Infrastructure → Outposts → Create (Proxy)

---

## 🚀 Primeros pasos

1. **Admin Login**
   - Abre `http://localhost:9000`
   - Click "Administration" → login con credenciales admin
   - Dashboard admin abre

2. **Crear usuarios**
   - Admin → Users → Create user
   - Ingresa username, email, password
   - Asigna groups si necesario
   - Save → usuario puede loguear

3. **Setup app SAML/OAuth2**
   - Admin → Applications → Create application
   - Nombre app, selecciona provider (SAML, OAuth2, OIDC)
   - Configure redirect URIs (app callback URL)
   - Client ID/Secret generado automático
   - App lista para SSO

4. **Setup reverse proxy outpost**
   - Admin → Infrastructure → Outposts → Create
   - Type: Proxy outpost
   - Configure forwarding (app interno, auth realm)
   - Outpost auto-configura reverse proxy
   - Acceso app protegido via Authentik

5. **Setup LDAP directory sync** (opcional)
   - Admin → Directory Sync → LDAP
   - Configure LDAP server connection
   - Map usuario + groups
   - Enable sync → auto-sync usuarios

6. **Setup social login** (opcional)
   - Admin → Sources → Create
   - Type: GitHub, Google, etc.
   - Configure OAuth app credentials
   - Users pueden loguear via social

7. **Crear flow personalizado** (avanzado)
   - Admin → Flows & Stages → Create Flow
   - Drag-drop stages (login form, MFA, policy check)
   - Lógica autenticación customizable
   - Asigna a applications/providers

---

## 💡 Casos de uso

- 🏢 **Reemplazo Okta/Auth0**: SSO enterprise self-hosted, multi-protocolo, sin vendor lock-in
- 🛡️ **Proteger apps legacy**: Reverse proxy outpost agrega SSO sin cambios de código
- 🏠 **Homelabs multi-usuario**: Gestión usuarios + SSO, apps protegidas centralizadas
- 🔄 **LDAP/AD integration**: Sync usuarios Active Directory, access control integrado
- ☁️ **Multi-tenant SaaS**: Blueprints, organizations separadas, branding custom
- 🔐 **Zero-trust security**: Access policies condicionales, risk-based auth, MFA enforcement

---

## 🔒 Acceso remoto seguro

Para exponer Authentik de forma segura a Internet:

1. **Reverse Proxy** (Traefik, Nginx Proxy Manager, Caddy) con TLS automático (Let's Encrypt)
2. **Authelia/Cloudflare Tunnel** para capa adicional de seguridad
3. **VPN** (WireGuard, Tailscale) para acceso solo red privada
4. **Configurar `AUTHENTIK_HOST`** y `AUTHENTIK_PROTOCOL=https` en variables de entorno
5. **HSTS, CSP, Security Headers** en proxy reverso

> ⚠️ **Nunca expongas puerto 9000 directamente a Internet sin TLS y autenticación adicional**

---

## 🛠️ Gestión y mantenimiento

```bash
# Ver estado
docker compose ps

# Ver logs
docker compose logs -f authentik_server
docker compose logs -f authentik_worker

# Detener Authentik
docker compose down

# Actualizar versión
docker compose pull
docker compose up -d

# Backup database
docker compose exec authentik_postgres pg_dump -U authentik authentik > backup.sql

# Restore database
docker compose exec -T authentik_postgres psql -U authentik authentik < backup.sql

# Monitorear consumo
docker stats authentik_server authentik_worker authentik_postgres authentik_redis

# Típicamente:
# server: 400-800MB RAM
# worker: 300-600MB RAM
# postgres: 300-500MB RAM
```

---

## 📝 Licencia

**MIT License** (core) - [Ver licencia](https://github.com/goauthentik/authentik/blob/main/LICENSE)

Enterprise edition disponible con features avanzadas y soporte comercial.

---

## 📚 Referencias oficiales

- [Authentik Official Website](https://goauthentik.io/)
- [Authentik GitHub Repository](https://github.com/goauthentik/authentik)
- [Authentik Documentation](https://goauthentik.io/docs/)
- [Docker Compose Installation Guide](https://goauthentik.io/docs/installation/docker-compose/)
- [Kubernetes Helm Chart](https://goauthentik.io/docs/installation/kubernetes/)
- [Authentik Helm Chart Repository](https://github.com/goauthentik/helm-charts)
- [Discord Community](https://discord.gg/authentik)
- [Enterprise Support & Pricing](https://goauthentik.io/pricing/)

---

> 📖 **Guía completa**: [Cómo instalar Authentik en Docker - SSO/IdP moderna autohospedada](https://genbyte.blogspot.com/2026/09/como-instalar-authentik-en-docker.html)