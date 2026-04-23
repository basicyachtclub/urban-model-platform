# Urban Model Platform — useful local development commands

Quick reference for setup, Docker, Flask, database migrations, and smoke tests.  
Default ports below match [`.env.example`](../.env.example); adjust if your `.env` differs.

| Service        | Typical URL / port |
|----------------|--------------------|
| UMP API (Flask)| `http://127.0.0.1:5000` |
| API prefix     | `/api` |
| API Postgres   | `127.0.0.1:5433` |
| Keycloak       | `http://localhost:8282` |
| GeoServer      | `http://localhost:8181/geoserver` |

**HTTP API (curl, processes, jobs):** see [api-basic-usage.md](api-basic-usage.md).

---

## One-time / occasional setup

```bash
# From repository root
make initiate-dev
```

Creates `./.venv` (if missing), copies `providers.yaml` / `.env` from examples, creates Docker network `ump_dev`, runs `poetry install`.

If the network already exists, `docker network create ump_dev` will error — that is safe to ignore.

**Git submodule (example model server only):**

```bash
git submodule update --init --recursive
```

---

## Daily workflow

### Start backing services + run migrations

```bash
make start-dev
```

Starts `api-db`, `keycloak`, and `kc-db`; runs `flask db upgrade` (and `db init` only when `migrations/env.py` is missing).

### Start GeoServer (optional)

```bash
docker compose -f docker-compose-dev.yaml up -d geoserver
```

### Start the UMP API on your machine

```bash
flask -A src/ump/main.py --debug run
```

Or with explicit Python env:

```bash
./.venv/bin/flask -A src/ump/main.py --debug run
```

Another port:

```bash
flask -A src/ump/main.py --debug run --port 5003
```

### Start example model server stack (after `start-dev`)

```bash
make start-dev-example
```

---

## Docker Compose

```bash
# Service status
docker compose -f docker-compose-dev.yaml ps

# Stop containers (keep volumes)
make stop-dev

# Restart
make restart-dev

# Remove containers and volumes (data loss)
make clean-dev
```

---

## Database migrations (manual)

Same as Makefile uses:

```bash
export FLASK_APP=src.ump.main
./.venv/bin/flask db current
./.venv/bin/flask db upgrade
```

Or with Alembic directly (see project docs):

```bash
alembic upgrade head
```

---

## Postgres

**From host** (default credentials in `.env.example`):

```bash
psql "postgresql://ump:ump@127.0.0.1:5433/ump" -c "SELECT 1 AS ok;"
```

**Inside the `api-db` container:**

```bash
docker compose -f docker-compose-dev.yaml exec api-db psql -U ump -d ump -c "SELECT 1 AS ok;"
```

---

## API smoke tests (`curl`)

Assume Flask on `127.0.0.1:5000` and `UMP_API_SERVER_URL_PREFIX=/api`.

**Readiness (DB reachable):**

```bash
curl -sS -i http://127.0.0.1:5000/api/health/ready
```

**Process catalog:**

```bash
curl -sS http://127.0.0.1:5000/api/processes/
```

**Sanitized provider config:**

```bash
curl -sS http://127.0.0.1:5000/api/processes/providers
```

**Single process** (replace with an ID from the catalog):

```bash
curl -sS "http://127.0.0.1:5000/api/processes/PREFIX/process-id"
```

**Jobs list:**

```bash
curl -sS "http://127.0.0.1:5000/api/jobs/"
```

**Authenticated request** (after obtaining an access token from Keycloak):

```bash
curl -sS -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  http://127.0.0.1:5000/api/jobs/
```

---

## Keycloak

**OpenID well-known (realm from `.env`):**

```bash
curl -sS -o /dev/null -w "%{http_code}\n" \
  http://localhost:8282/auth/realms/UrbanModelPlatform/.well-known/openid-configuration
```

**Fetch metadata (first lines):**

```bash
curl -sS http://localhost:8282/auth/realms/UrbanModelPlatform/.well-known/openid-configuration | head -c 1500
```

**Admin console (browser):**  
`http://localhost:8282/auth/admin/`  
(Use dev admin credentials from your `docker-compose-dev.yaml` / `.env`.)

---

## GeoServer

**HTTP check:**

```bash
curl -sS -o /dev/null -w "%{http_code}\n" http://localhost:8181/geoserver/web/
```

**Web UI:**  
`http://localhost:8181/geoserver/web/`

---

## Documentation build

```bash
make build-docs
make clean-docs
```

---

## See also

- [Test the example model (beginner, macOS)](test-example-model-beginner.md)
- [Local setup findings / troubleshooting](local-dev-setup-findings.md)
- [Contributing](../CONTRIBUTING.md)
