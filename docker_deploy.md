# Gapps — Production Deployment Guide

> Deploy this app to the shared app server at `https://ptswebapps/gapps/`.

---

## Server Overview

| Detail     | Value                                 |
|------------|---------------------------------------|
| Hostname   | `PTSCORPVS0WAPP01`                   |
| IP address | `10.69.69.10`                         |
| DNS name   | `ptswebapps`                          |
| OS         | Ubuntu 24.04 LTS                      |
| Docker     | 29.2.1 (Compose v5.0.2 plugin)       |
| Proxy      | Traefik (latest), HTTPS with TLS      |
| SSH user   | `wapp01admin` (key-based auth, passwordless sudo) |

---

## App Details

| Detail          | Value                                                    |
|-----------------|----------------------------------------------------------|
| App name        | `gapps`                                                  |
| Path prefix     | `/gapps`                                                 |
| Container port  | 5000                                                     |
| Routing pattern | **StripPrefix + APP_PREFIX** — Traefik strips `/gapps`, Flask WSGI middleware re-adds it via SCRIPT_NAME for URL generation |
| Database        | PostgreSQL 16 (separate container, Docker named volume)  |
| Auth            | Local credentials (email/password)                       |

---

## Prerequisites

- SSH access to server: `ssh wapp01admin@10.69.69.10`
- GitHub repo: `https://github.com/msp-vibe-coder/gapps.git`
- No SSO or external auth providers required for the base deployment

---

## Docker Compose Configuration

The production `docker-compose.yml` is created directly on the server (not the one in the repo, which is for local dev).

```yaml
services:
  gapps:
    build: .
    container_name: gapps
    restart: unless-stopped
    depends_on:
      - gapps-db
    env_file:
      - .env.docker
    environment:
      - FLASK_CONFIG=default  # maps to ProductionConfig in config.py
      - SQLALCHEMY_DATABASE_URI=postgresql://${POSTGRES_USER:-db1}:${POSTGRES_PASSWORD:-db1}@gapps-db/${POSTGRES_DB:-db1}
      - APP_PREFIX=/gapps
      - HOST_NAME=https://ptswebapps/gapps
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.gapps.rule=PathPrefix(`/gapps`)"
      - "traefik.http.routers.gapps.entrypoints=websecure"
      - "traefik.http.routers.gapps.tls=true"
      - "traefik.http.middlewares.gapps-strip.stripprefix.prefixes=/gapps"
      - "traefik.http.routers.gapps.middlewares=gapps-strip"
      - "traefik.http.services.gapps.loadbalancer.server.port=5000"
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:5000/api/v1/health')"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    networks:
      - traefik-net
      - gapps-internal

  gapps-db:
    image: postgres:16
    container_name: gapps-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-db1}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-db1}
      POSTGRES_DB: ${POSTGRES_DB:-db1}
      PGDATA: /data/postgres
    volumes:
      - gapps-pgdata:/data/postgres
    networks:
      - gapps-internal

networks:
  traefik-net:
    external: true
  gapps-internal:
    driver: bridge

volumes:
  gapps-pgdata:
```

> **Why StripPrefix + APP_PREFIX?** Flask doesn't natively handle path prefixes like Next.js `basePath`. Traefik's `StripPrefix` removes `/gapps` so Flask sees clean paths (`/login`, `/api/v1/...`). The `APP_PREFIX=/gapps` env var triggers WSGI middleware that sets WSGI `SCRIPT_NAME`, making `url_for()` generate `/gapps/...` URLs, and a JavaScript fetch override prepends the prefix to all API calls. We use `APP_PREFIX` instead of `SCRIPT_NAME` because Gunicorn reads `SCRIPT_NAME` from the process environment and conflicts with Traefik's StripPrefix.

> **Why a separate internal network?** The PostgreSQL container (`gapps-db`) is on `gapps-internal` only — it is not exposed to Traefik or the host. Only the `gapps` app container bridges both networks.

---

## `.env.docker` Template

Create this file on the server at `/home/wapp01admin/apps/gapps/.env.docker`:

```env
# ============================================================
# Gapps — Production Environment Variables
# ============================================================

# Flask secret key (REQUIRED — change from default)
# Generate: python -c "import secrets; print(secrets.token_hex(32))"
SECRET_KEY=

# PostgreSQL credentials (must match docker-compose environment)
POSTGRES_USER=db1
POSTGRES_PASSWORD=db1
POSTGRES_DB=db1

# Gunicorn workers (2-4 recommended for production)
GUNICORN_WORKERS=2

# Default admin account (created on first init_db)
DEFAULT_EMAIL=admin@example.com
DEFAULT_PASSWORD=admin1234567
```

> **Note:** `SQLALCHEMY_DATABASE_URI`, `FLASK_CONFIG`, `APP_PREFIX`, and `HOST_NAME` are set in docker-compose.yml's `environment:` block — do not duplicate them here.

Upload to server:
```bash
scp .env.docker wapp01admin@10.69.69.10:/home/wapp01admin/apps/gapps/.env.docker
```

---

## Deployment Steps

### First Deploy

```bash
# 1. Clone the repo on the server
ssh wapp01admin@10.69.69.10 "cd /home/wapp01admin/apps && git clone https://github.com/msp-vibe-coder/gapps.git gapps"

# 2. Checkout the correct branch
ssh wapp01admin@10.69.69.10 "cd /home/wapp01admin/apps/gapps && git checkout feature/reverse-proxy-support"

# 3. Fix CRLF line endings in shell scripts (repo is from Windows)
ssh wapp01admin@10.69.69.10 "cd /home/wapp01admin/apps/gapps && find . -name '*.sh' -exec sed -i 's/\r$//' {} +"

# 4. Create .env.docker on the server (see template above)

# 5. Replace docker-compose.yml with production version (see config above)

# 6. Build and start
ssh wapp01admin@10.69.69.10 "cd /home/wapp01admin/apps/gapps && docker compose up -d --build"

# 7. Verify containers are running
ssh wapp01admin@10.69.69.10 "docker ps | grep gapps"
```

The container automatically checks DB connectivity, initializes models if needed, and starts Gunicorn via `run.sh`. On first start, tables are created and the default admin account is seeded.

**Default admin credentials:** `admin@example.com` / `admin1234567` — change the password after first login.

### Redeploy (Updates)

```bash
ssh wapp01admin@10.69.69.10 "cd /home/wapp01admin/apps/gapps && git pull && docker compose up -d --build"
```

This pulls the latest code, rebuilds the image, and restarts the container. The production `docker-compose.yml` and `.env.docker` are not in git, so they persist across pulls.

---

## Verification Checklist

After deploying, verify everything works:

- [ ] `https://ptswebapps/gapps/` loads the login page
- [ ] Login with `admin@example.com` / `admin1234567` works
- [ ] After login, redirects to `/gapps/`
- [ ] Sidebar navigation links all start with `/gapps/` (home, tenants, projects, integrations)
- [ ] Risk register link works (sidebar → Risk → Risk Register)
- [ ] "More" menu links work (tenant users, admin users, settings, logs)
- [ ] Browser devtools Network tab: all API calls go to `/gapps/api/v1/...` (no 404s)
- [ ] Static assets load from `/gapps/static/...`
- [ ] `window.SCRIPT_ROOT` in browser console equals `/gapps`
- [ ] Create a project → navigate to it → policy center links work
- [ ] Evidence download links include `/gapps/` prefix
- [ ] Logout → login page at `/gapps/login` → login → redirects to `/gapps/`
- [ ] `docker ps` shows both `gapps` and `gapps-db` as `Up` (gapps shows `healthy`)
- [ ] `https://ptswebapps/` does NOT serve this app (Traefik routes only matching paths)

---

## Troubleshooting

### Container won't start

```bash
ssh wapp01admin@10.69.69.10 "docker logs gapps --tail 50"
```

Common causes:
- Missing `.env.docker` — upload it and restart
- DB connection failed — check that `gapps-db` container is running: `docker logs gapps-db`
- Wrong `POSTGRES_PASSWORD` — must match between `.env.docker` and docker-compose environment

### Database connection errors

```bash
# Check if PostgreSQL container is running
ssh wapp01admin@10.69.69.10 "docker ps | grep gapps-db"

# Check PostgreSQL logs
ssh wapp01admin@10.69.69.10 "docker logs gapps-db --tail 20"

# Test connectivity from app container
ssh wapp01admin@10.69.69.10 "docker exec gapps python -c \"import psycopg2; psycopg2.connect('postgresql://db1:db1@gapps-db/db1')\""
```

### Path prefix not working (pages serve at `/` instead of `/gapps/`)

1. Verify `APP_PREFIX=/gapps` is set in docker-compose environment
2. Verify `HOST_NAME=https://ptswebapps/gapps` is set in docker-compose environment
3. Check that Traefik StripPrefix middleware is active:
   ```bash
   ssh wapp01admin@10.69.69.10 "docker inspect gapps --format '{{json .Config.Labels}}' | python -m json.tool | grep strip"
   ```
4. Check browser console — `window.SCRIPT_ROOT` should equal `/gapps`

### CRLF line ending issues

If the container fails with `\r: command not found`, fix on the server:
```bash
ssh wapp01admin@10.69.69.10 "cd /home/wapp01admin/apps/gapps && find . -name '*.sh' -exec sed -i 's/\r$//' {} +"
```

### `.env.docker` lost after redeploy

The deploy script clones/pulls from GitHub — `.env.docker` is not in git. Keep a local backup:
```bash
# Save a backup locally
scp wapp01admin@10.69.69.10:/home/wapp01admin/apps/gapps/.env.docker ./env-docker-backup

# Restore after a fresh clone
scp ./env-docker-backup wapp01admin@10.69.69.10:/home/wapp01admin/apps/gapps/.env.docker
```

---

## Architecture Notes

- **Routing**: StripPrefix + APP_PREFIX. Traefik `PathPrefix(/gapps)` matches requests, `StripPrefix` removes `/gapps` before forwarding to Flask. WSGI middleware reads `APP_PREFIX` and sets WSGI `SCRIPT_NAME=/gapps` so `url_for()` generates prefixed URLs. A global JavaScript `fetch()` override prepends the prefix to all 125+ API calls automatically. We use `APP_PREFIX` instead of `SCRIPT_NAME` because Gunicorn reads `SCRIPT_NAME` from the process environment and expects all incoming URLs to include the prefix — which conflicts with Traefik already having stripped it.
- **Database**: PostgreSQL 16 in a separate container (`gapps-db`) on an internal-only network. Data persisted in Docker named volume `gapps-pgdata`.
- **Auto-init**: `run.sh` entrypoint checks DB connectivity, initializes models/tables if needed, then starts Gunicorn. No manual migration step required for first deploy.
- **Health check**: HTTP check against `/api/v1/health` every 30s. Start period of 60s allows for DB init + model setup.
- **TLS**: Traefik handles TLS termination. The app runs HTTP internally on port 5000.
- **Backward compatible**: Without `APP_PREFIX` set, the app works at root (`/`) exactly as before — the JavaScript prefix helpers are no-ops when `SCRIPT_ROOT` is empty.
