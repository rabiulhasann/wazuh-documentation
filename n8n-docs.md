**Tags:** Docker, PostgreSQL, Production

# Deploy *n8n* in Production

A complete Docker Compose setup with PostgreSQL, environment management, health checks, and everything you need to run n8n reliably in production.

**Stats:**

- **2** Services
- **3** Config Files
- **5** Health Checks

## Quick Start

Get n8n running in under 5 minutes.

### Clone your files

Place all three config files in a project directory.

```bash
mkdir n8n-prod && cd n8n-prod
```

### Create your .env file

Copy the example and fill in your values.

```bash
cp .env.example .env
```

### Generate secret keys

Run this twice — once for each secret key in your .env.

```bash
openssl rand -hex 32
```

### Create local-files directory

```bash
mkdir -p local-files
```

### Launch!

```bash
docker compose up -d
```

## File Structure

Your project directory should look like this:

```
📁 n8n-prod/
├── docker-compose.yaml — service definitions
├── .env — your secrets (never commit!)
├── .env.example — template (safe to commit)
├── .gitignore — protects sensitive files
└── 📁 local-files/ — workflow file access
```

## docker-compose.yaml

Two services: **postgres** (database) and **n8n** (app). n8n waits for Postgres to be healthy before starting.

Architecture:

- **postgres** — postgres:16-alpine (healthcheck, restart)
- **n8n** — n8nio/n8n:latest (:5678, healthcheck)

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: n8n_postgres
    restart: unless-stopped
    environment:
      - POSTGRES_DB=${DB_POSTGRESDB_DATABASE}
      - POSTGRES_USER=${DB_POSTGRESDB_USER}
      - POSTGRES_PASSWORD=${DB_POSTGRESDB_PASSWORD}
		volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_POSTGRESDB_USER} -d ${DB_POSTGRESDB_DATABASE}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s
    networks:
      - n8n_network

  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "${N8N_PORT:-5678}:5678"
    environment:
      # Timezone
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - TZ=${TZ}
# n8n Host & Network
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=${N8N_PROTOCOL:-https}
      - WEBHOOK_URL=${WEBHOOK_URL}
# n8n Settings
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_RUNNERS_ENABLED=true
      - N8N_SECURE_COOKIE=false
      # Security
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_USER_MANAGEMENT_JWT_SECRET=${N8N_USER_MANAGEMENT_JWT_SECRET}
# Database
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${DB_POSTGRESDB_DATABASE}
      - DB_POSTGRESDB_USER=${DB_POSTGRESDB_USER}
      - DB_POSTGRESDB_PASSWORD=${DB_POSTGRESDB_PASSWORD}
      - DB_POSTGRESDB_SCHEMA=${DB_POSTGRESDB_SCHEMA:-public}
# Logging
      - N8N_LOG_LEVEL=${N8N_LOG_LEVEL:-info}
      - N8N_LOG_OUTPUT=${N8N_LOG_OUTPUT:-console}
		volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/files
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- <http://localhost:5678/healthz> || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    networks:
      - n8n_network

volumes:
  postgres_data:
    driver: local
  n8n_data:
    driver: local

networks:
  n8n_network:
    driver: bridge
```

## .env.example

Copy to `.env` and replace all placeholder values. Never commit `.env` to version control.

> ⚠
**Security Note:** The `.env` file contains secrets. Only `.env.example` should ever be committed to Git.
> 

```
# ============================================================
# n8n Production Environment Variables
# ============================================================
# 1. Copy this file:   cp .env.example .env
# 2. Fill in values below
# 3. NEVER commit .env to version control
# Timezone
GENERIC_TIMEZONE=Asia/Dhaka
TZ=Asia/Dhaka

# n8n Host & Network
N8N_HOST=n8n.yourdomain.com
N8N_PORT=5678
N8N_PROTOCOL=https
WEBHOOK_URL=https://n8n.yourdomain.com/

# Security — generate with: openssl rand -hex 32
N8N_ENCRYPTION_KEY=change_me_use_openssl_rand_hex_32
N8N_USER_MANAGEMENT_JWT_SECRET=change_me_use_openssl_rand_hex_32
# PostgreSQL
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n_user
DB_POSTGRESDB_PASSWORD=change_me_strong_password
DB_POSTGRESDB_SCHEMA=public

# Logging: error | warn | info | verbose | debug
N8N_LOG_LEVEL=info
N8N_LOG_OUTPUT=console
```

## .gitignore

Ensures secrets and runtime data never get committed to version control.

```
# Environment — never commit
.env
.env.local
.env.production

# Runtime data
local-files/

# OS
.DS_Store
Thumbs.db
```

## Environment Variables

Complete reference for all supported variables.

| Variable | Default | Required | Description |
| --- | --- | --- | --- |
| 🕐 Timezone |  |  |  |
| `GENERIC_TIMEZONE` | Asia/Dhaka | Yes | n8n internal timezone |
| `TZ` | Asia/Dhaka | Yes | Container system timezone |
| 🌐 Network |  |  |  |
| `N8N_HOST` | — | Yes | Your domain or IP address |
| `N8N_PORT` | 5678 | No | External port mapping |
| `N8N_PROTOCOL` | https | No | Use `http` for bare IP access |
| `WEBHOOK_URL` | — | Yes | Full URL with trailing slash |
| 🔐 Security |  |  |  |
| `N8N_ENCRYPTION_KEY` | — | Yes | Encrypts stored credentials. Generate: `openssl rand -hex 32` |
| `N8N_USER_MANAGEMENT_JWT_SECRET` | — | Yes | Signs auth tokens. Generate: `openssl rand -hex 32` |
| `N8N_SECURE_COOKIE` | true | No | Set `false` when using HTTP (no HTTPS) |
| 🐘 PostgreSQL |  |  |  |
| `DB_POSTGRESDB_DATABASE` | n8n | Yes | Database name |
| `DB_POSTGRESDB_USER` | n8n_user | Yes | Database user |
| `DB_POSTGRESDB_PASSWORD` | — | Yes | Database password — use a strong value |
| `DB_POSTGRESDB_SCHEMA` | public | No | Postgres schema |
| 📋 Logging |  |  |  |
| `N8N_LOG_LEVEL` | info | No | error / warn / info / verbose / debug |
| `N8N_LOG_OUTPUT` | console | No | console / file |

## Troubleshooting

### SSL_ERROR_RX_RECORD_TOO_LONG

Browser is connecting via HTTPS but n8n is serving HTTP.

Fix
Set `N8N_PROTOCOL=http` and access via `http://` URL. Or set up a proper reverse proxy with TLS.

### Secure cookie / insecure URL error

n8n's secure cookie is enabled but you're on HTTP.

Fix
Add `N8N_SECURE_COOKIE=false` to the **environment:** section in docker-compose.yaml. Setting it only in .env is not enough.

### n8n starts before Postgres is ready

Race condition on first boot.

Fix
Already handled via `depends_on: condition: service_healthy`. If still failing, increase `start_period` on the postgres healthcheck.

### .env variable not applied to container

Variable is in .env but container doesn't see it.

Fix
.env only provides substitution for `${VAR}` placeholders. Every variable must also be listed under `environment:` in docker-compose.yaml.

## Nginx Reverse Proxy

For production HTTPS, place n8n behind Nginx with a Let's Encrypt certificate.

> ℹ
Install Certbot: `apt install certbot python3-certbot-nginx` then run `certbot --nginx -d n8n.yourdomain.com`
> 

```
server {
    listen 443 ssl;
    server_name n8n.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/n8n.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.yourdomain.com/privkey.pem;

    location / {
        proxy_pass         <http://localhost:5678>;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "upgrade";
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        chunked_transfer_encoding on;
        proxy_buffering    off;
        proxy_cache        off;
        proxy_read_timeout 3600s;
    }
}

server {
    listen 80;
    server_name n8n.yourdomain.com;
    return 301 https://$host$request_uri;
}
```

## Common Commands

### Start

```bash
docker compose up -d
```

### Stop

```bash
docker compose down
```

### View Logs

```bash
docker compose logs -f n8n
```

### Upgrade

```bash
docker compose pull && docker compose down && docker compose up -d
```

### Container Status

```bash
docker compose ps
```

### n8n Shell

```bash
docker exec -it n8n sh
```
