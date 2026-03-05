# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Gapps is a multi-tenant Security GRC (Governance, Risk, Compliance) platform built with Flask. It tracks compliance progress against frameworks like SOC2, NIST 800-53, ISO 27001, HIPAA, CMMC, PCI DSS, CIS v8, ASVS, NIST CSF, and SSF.

## Build & Run Commands

```bash
# Start the full stack (app + PostgreSQL) via Docker
docker compose up -d

# Rebuild after code changes
docker compose up -d --build

# Run outside Docker (requires a running PostgreSQL)
export FLASK_CONFIG=development
export SQLALCHEMY_DATABASE_URI="postgresql://db1:db1@localhost/db1"
bash run.sh

# Database operations (inside the container or locally)
python manage.py init_db          # Drop all tables and recreate (destructive)
python manage.py create_db        # Create tables without dropping
python manage.py migrate_db       # Run Alembic migrations
python manage.py db migrate       # Generate a new migration
python manage.py db upgrade       # Apply pending migrations
python manage.py db stamp head    # Mark current state as up-to-date

# Startup env flags (set before run.sh or in docker-compose)
RESET_DB=yes           # Reset database on startup
SKIP_INI_CHECKS=yes    # Skip DB connectivity/model checks
INIT_MIGRATE=yes       # Initialize Alembic
MIGRATE=yes            # Run pending migrations
ONESHOT=yes            # Run setup then exit (no Gunicorn)
```

There is no test suite, linter config, or CI/CD pipeline in the repository.

## Architecture

### Application Factory & Entry Points

- **`app/__init__.py`** — Flask application factory (`create_app()`). Registers blueprints, extensions (SQLAlchemy, Flask-Mail, Flask-Login, Flask-Migrate, Authlib OAuth), error handlers, and logging.
- **`flask_app.py`** — Creates the app instance; usable as `python flask_app.py`.
- **`manage.py`** — Flask-Script CLI (`python manage.py <command>`). Commands: `init_db`, `create_db`, `migrate_db`, `force_drop_db`.
- **`run.sh`** — Container entrypoint. Checks DB connectivity, initializes models if needed, then starts Gunicorn.
- **`config.py`** — Three configs: `DevelopmentConfig`, `ProductionConfig`, `TestingConfig`. All settings are env-var-driven.

### Three Blueprints

| Blueprint | Prefix | Source | Purpose |
|-----------|--------|--------|---------|
| `main` | `/` | `app/main/views.py`, `general.py` | Server-rendered HTML pages (Jinja2 templates) |
| `api_v1` | `/api/v1` | `app/api_v1/base.py`, `views.py`, `vendors.py`, `integrations.py` | REST API (~155 endpoints) |
| `auth` | `/` | `app/auth/views.py`, `google.py`, `microsoft.py` | Login, registration, OAuth2/OIDC, magic links |

### API Response Convention

All API endpoints return JSON:
```json
{"ok": true, "message": "success", "code": 200, "extra": { ... }}
```

### Models (`app/models.py`)

Single large file (~5k lines) with ~50 SQLAlchemy models. Key domain groups:

- **Multi-tenancy**: `User`, `Tenant`, `TenantMember`, `TenantMemberRole`, `Role`
- **Compliance core**: `Framework`, `Control`, `SubControl`, `Project`, `ProjectControl`, `ProjectSubControl`
- **Evidence & policy**: `ProjectEvidence`, `EvidenceAssociation`, `Policy`, `PolicyVersion`
- **Risk & audit**: `RiskRegister`, `AuditorFeedback`, `Finding`
- **Vendor management**: `Vendor`, `VendorApp`, `VendorFile`, `Assessment`, `Form`, `FormSection`, `FormItem`
- **Supporting**: `Tag`, `Logs`, `ConfigStore`, `CompletionHistory`

Model mixins live in `app/utils/mixin_models.py` (`DateMixin`, `QueryMixin`, `ControlMixin`, `SubControlMixin`, `AuthorizerMixin`).

### Authorization System

- **`app/utils/authorizer.py`** — `Authorizer` class with 40+ methods for fine-grained RBAC (platform, tenant, project, risk, vendor level). Returns standardized `{status, message, code, object}` dicts.
- **`app/utils/decorators.py`** — `@login_required` (checks auth + email confirmation + password-change enforcement), `@is_logged_in`, `@custom_login()`. API token auth reads from `token` or `Authorization` header.
- **Roles**: `super` (platform admin), `admin`, `viewer`, `vendor`, `riskmanager`, `riskviewer`.

### Frontend

Server-rendered Jinja2 templates using **Alpine.js** for interactivity and **Tailwind CSS + DaisyUI** for styling. Templates are in `app/templates/` with layouts in `app/templates/layouts/`. Static assets in `app/static/`.

### File Storage

`app/utils/file_handler.py` — `FileStorageHandler` abstracts local, AWS S3, and Google Cloud Storage. Selected via `STORAGE_METHOD` env var (`local` | `s3` | `gcs`).

### Framework Data

Pre-built compliance framework definitions live as JSON files in `app/files/base_controls/`. Custom frameworks can be added by dropping a `.json` file in that directory (filename becomes framework name). Policy templates are HTML files in `app/files/base_policies/`.

### Feature Flags

Any environment variable prefixed with `feature_` becomes a boolean feature flag. Accessible in backend via `current_app.config["FEATURE_FLAGS"]` and in frontend templates via `$store.currentUser.featureFlags`.

## Key Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `FLASK_CONFIG` | `development` | Config environment |
| `SQLALCHEMY_DATABASE_URI` | `postgresql://db1:db1@postgres/db1` | Full DB connection string |
| `SECRET_KEY` | `change_secret_key` | Flask session secret |
| `HOST_NAME` | `http://localhost` | Application URL |
| `STORAGE_METHOD` | `local` | File storage backend (`local`, `s3`, `gcs`) |
| `ENABLE_GOOGLE_AUTH` | `false` | Enable Google OAuth |
| `ENABLE_MICROSOFT_AUTH` | `false` | Enable Microsoft OAuth |
| `ENABLE_SELF_REGISTRATION` | `false` | Allow public signup |
| `LLM_ENABLED` | `false` | Enable AI features |
| `GUNICORN_WORKERS` | `1` | Gunicorn worker count |

## Default Credentials

- **PostgreSQL**: user `db1`, password `db1`, database `db1`
- **App admin**: created during `init_db` — check `app/commands/init_db.py` for defaults
- **Docker port mapping**: app on `localhost:8000`, PostgreSQL on `localhost:5432`
