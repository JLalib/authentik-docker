# 🔐 Authentik Docker - SSO/IdP Moderna Autohospedada

[![GitHub Stars](https://img.shields.io/github/stars/goauthentik/authentik?style=flat-square&logo=github)](https://github.com/goauthentik/authentik)
[![Docker Pulls](https://img.shields.io/docker/pulls/goauthentik/server?style=flat-square&logo=docker)](https://hub.docker.com/r/goauthentik/server)
[![License](https://img.shields.io/github/license/goauthentik/authentik?style=flat-square)](https://github.com/goauthentik/authentik/blob/main/LICENSE)
[![Version](https://img.shields.io/docker/v/goauthentik/server?style=flat-square&logo=docker)](https://hub.docker.com/r/goauthentik/server/tags)

## 📋 Descripción general

**Authentik** es un **Identity Provider (IdP) open source moderno** que proporciona **SSO (Single Sign-On) profesional autohospedado** compatible con **SAML, OAuth2/OIDC, LDAP, RADIUS**. Arquitectura **multi-tenant**, **flows visuales** para lógica de autenticación, **access policies**, **blueprints** (automatización), **reverse proxy integrado** (outpost), **auditing completo**, escalable en **Docker/Kubernetes**, todo bajo tu control sin **vendor lock-in**.

El reemplazo **Okta/Auth0/Entra ID/Ping Identity** que pedías para autohospedado. **24.7k estrellas en GitHub**, **MIT open source**, **production-ready**.

## ✨ Características principales

- 🔐 **Multi-protocolo SSO**: SAML 2.0, OAuth2, OIDC, LDAP, RADIUS - un IdP para todo
- 🌐 **Reverse proxy integrado**: Outpost proxy protege cualquier app legacy sin cambios de código
- 🏢 **Multi-tenant**: Múltiples organizations en una instalación, branding customizable
- 🎨 **Visual flow editor**: Drag-drop auth logic (conditional, MFA, risk-based) sin código
- 🛡️ **Access policies**: Conditional auth, geo-blocking, device trust, MFA enforcement
- ⚙️ **Blueprints**: Automation, bulk config, IaC - deployment YAML-based
- 📂 **Directory sync**: LDAP, Active Directory, Google Workspace sync (users + groups)
- 🔑 **Social login**: GitHub, Google, Apple, Discord, proveedores OIDC custom
- 🔐 **MFA + WebAuthn**: TOTP, backup codes, WebAuthn (FIDO2) - seguridad moderna
- 📊 **Audit logging**: Todos los events tracked (user actions, auth, policy changes)
- 📱 **App library**: 500+ pre-configured app integrations - instant SSO setup
- ☸️ **Kubernetes-native**: Helm chart oficial, scalable, cloud-ready, production

## 📋 Requisitos del sistema

- **Docker & Docker Compose v2+**
- **RAM**: 2 GB - 4 GB mínimo (Python + Go app) — **4 GB recomendado**
- **Disco**: 10 GB - 50+ GB espacio (según users, audit logs)
- **Puertos TCP**: 
  - `9000` (web UI)
  - `9300` (LDAP outpost)
  - `9400` (RADIUS outpost)
- **PostgreSQL 14+** (bundled o externo)
- **Redis** (sessions, caching, optional pero recomendado)
- **Python 3.11+** (bundled en imagen)
- **Go 1.20+** (outpost proxy, bundled)
- **Opcional**: LDAP/AD servidor para directory sync
- **Opcional**: External PostgreSQL (para HA production)

> ⚠️ **Compute requirements**: Authentik requiere bastante recursos (Python backend + Go outpost). 2GB RAM mínimo pero 4GB recomendado. Audit logging puede llenar disco rápido, plan retención.
>
> 🏭 **HA/Production**: Para producción usar external PostgreSQL (replicación), Redis externo, múltiples replicas. Kubernetes + Helm recomendado para escala.

## 🐳 Instalación

### Paso 1: docker-compose.yml

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

# Copiar y reemplazar AUTHENTIK_SECRET_KEY en compose
```

### Paso 3: Iniciar Authentik

```bash
# Guardar como docker-compose.yml
docker compose up -d

# Espera ~20 segundos para migrations
docker compose logs -f authentik_server

# Debería ver "Starting application server" cuando listo
```

### Acceder a Authentik

| Servicio | URL |
|----------|-----|
| 🔐 **Authentik Admin UI** | `http://localhost:9000` |
| 🆔 **Authentik Login** | `http://localhost:9000/auth/login/` |

### Setup inicial (primer acceso)

1. Abre `http://localhost:9000`
2. Setup wizard automático → crea admin user
3. Ingresa email + password admin
4. Dashboard admin aparece
5. Configura LDAP sync (opcional), social logins, apps
6. ¡Listo SSO! 🎉

> 💡 **Desde otros dispositivos**: Usa la IP de tu servidor: `http://192.168.1.100:9000`
> Para obtener tu IP: `hostname -I`

## ⚙️ Configuración

1. **Variables de entorno críticas**: `AUTHENTIK_SECRET_KEY` (generar con `openssl rand -hex 32`), `AUTHENTIK_POSTGRESQL__PASSWORD`, `AUTHENTIK_REDIS__HOST`
2. **PostgreSQL**: Usar external DB en producción (replicación, backups gestionados)
3. **Redis**: Recomendado para sessions/cache; external en HA
4. **Outposts**: Configurar Proxy Outpost para proteger apps legacy (puerto 9000/9443)
5. **Email/SMTP**: Configurar en Admin → System → Email para password reset, notificaciones
6. **Certificados TLS**: Usar reverse proxy (Traefik, Nginx Proxy Manager) con Let's Encrypt para HTTPS
7. **Backup strategy**: pg_dump programado + volume snapshots
8. **Log retention**: Configurar rotation para evitar llenar disco (audit logs crecen rápido)

## 🚀 Primeros pasos

1. **Admin login**
   - Abre `http://localhost:9000`
   - Click "Administration" → login con admin credentials
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

5. **Setup LDAP directory sync (opcional)**
   - Admin → Directory Sync → LDAP
   - Configure LDAP server connection
   - Map usuario + groups
   - Enable sync → auto-sync usuarios

6. **Setup social login (opcional)**
   - Admin → Sources → Create
   - Type: GitHub, Google, etc
   - Configure OAuth app credentials
   - Users pueden loguear via social

7. **Crear flow personalizado (avanzado)**
   - Admin → Flows & Stages → Create Flow
   - Drag-drop stages (login form, MFA, policy check)
   - Lógica autenticación customizable
   - Asigna a applications/providers

## 💡 Casos de uso

- 🏢 **Reemplazo Okta/Auth0**: SSO enterprise self-hosted, multi-protocolo, no vendor lock-in
- 🛡️ **Proteger apps legacy**: Reverse proxy outpost, agrega SSO sin cambios código
- 🏠 **Homelabs multi-usuario**: Gestión usuarios + SSO, apps protegidas
- 🔄 **LDAP/AD integration**: Sync usuarios Active Directory, access control integrado
- ☁️ **Multi-tenant SaaS**: Blueprints, organizations separadas, branding custom
- 🔐 **Zero-trust security**: Access policies condicionales, risk-based auth, MFA

## 🔒 Acceso remoto seguro

Para exponer Authentik de forma segura a Internet:

1. **Reverse proxy** (Traefik/Nginx Proxy Manager/Caddy) con **TLS automático** (Let's Encrypt)
2. **Authelia/Cloudflare Tunnel** como capa adicional
3. **Restringir Admin UI** a VPN (WireGuard/Tailscale) o IP allowlist
4. **Configurar `AUTHENTIK_HOST`** y `AUTHENTIK_PROTOCOL=https` en environment
5. **HSTS, CSP headers** via reverse proxy
6. **Rate limiting** en proxy (Authentik tiene rate limiting interno también)

> ⚠️ **NUNCA** expongas puerto 9000 directamente a Internet sin TLS y autenticación adicional.

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

## 📝 Licencia

**MIT License** (core) + **Enterprise edition** disponible con features avanzadas y soporte comercial.

Copyright (c) 2024+ goauthentik/authentik contributors

---

> 📖 **Guía completa**: [Cómo instalar Authentik en Docker - SSO/IdP moderna autohospedada](https://genbyte.blogspot.com/2026/09/como-instalar-authentik-en-docker.html)