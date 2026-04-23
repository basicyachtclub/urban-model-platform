# Urban Model Platform — local development setup findings

This note records what blocked `make start-dev` on a typical developer machine (macOS, Docker, Poetry/conda-style `.venv`), what was changed, and how to verify the fix.

---

## Non-technical summary

**What was going wrong**

The project’s “start development” command assumed two things that often are not true on a real laptop: that a tool called **conda** is installed and on the command line, and that the database setup step can be run the same way every time. In practice, the environment folder (`.venv`) was there and had the right programs inside it, but the Makefile did not use those programs reliably. Separately, a small bug in configuration code could crash the app as soon as Python loaded settings. The example environment file also pointed authentication and map-server addresses at **Docker-internal names** that only work *inside* containers, not when you run the API on your Mac next to Docker.

**What we changed**

- The Makefile was updated so development commands call **Flask directly from `./.venv/bin`**, without requiring `conda` on your PATH.
- Database initialization (`flask db init`) is **skipped when the project already ships migration files**, so repeat runs do not fail.
- Configuration code was fixed so **Keycloak URLs** are normalized safely with current Python libraries.
- Example and local `.env` values were aligned so that when the API runs **on the host**, it talks to Keycloak and GeoServer via **localhost and the ports published by Docker** (not `keycloak` / `geoserver` hostnames).
- Contributing docs were corrected so the **Docker network name** matches what Compose expects (`ump_dev`, not `dev`).

**What you should still know**

- `make start-dev` starts **Postgres (API DB), Keycloak, and Keycloak’s DB**. It does **not** start GeoServer by default. If you need GeoServer locally, start it explicitly with Docker Compose.
- After `make start-dev`, run the Flask app from your IDE or the command line as described in the project’s contributing guide.
- **After a machine reboot:** start **Docker Desktop** before any `make` target that uses Docker. Otherwise you see *Cannot connect to the Docker daemon*.
- The **`ump_dev` Docker network** (and your containers) **persist** across reboots. If `make initiate-dev` fails with *network with name ump_dev already exists*, the network is fine; you can ignore that or skip re-running `initiate-dev` when everything is already configured.
- On **Apple Silicon**, the **example `modelserver`** image must build as **`linux/amd64`** so `rasterio` installs from a prebuilt wheel (see `platform` in `docker-compose-dev.yaml` and the submodule `Dockerfile`). Otherwise the image build can fail with Poetry/rasterio errors.
- If **`POST` to `/api/processes/.../execution`** returns **500** with a **PostgreSQL syntax error** around `encode` / `sha512` / `check_for_cache`, the **`check_for_cache` SQL in `Process`** must match the hash used when inserting jobs (see `job.py`); this was corrected in `src/ump/api/models/process.py` so the same `curl` example in the docs succeeds.

---

## Technical summary

### Symptoms observed

1. **`make start-dev` exited early with `conda: command not found`**  
   The recipe relied on `conda activate` and a conda `profile.d` script. The project’s `./.venv` was a **conda-prefix environment** (e.g. `conda-meta` present) but **without** a classic `bin/activate` script, and the `conda` CLI was not available in the non-interactive shell used by `make`.

2. **Flask / import failures (`flask db init` / `flask db upgrade`)**  
   Importing the app triggered `UmpSettings` validation. The field validator `ensure_trailing_slash` for `UMP_KEYCLOAK_URL` called `str.endswith` on a value that could already be a **Pydantic `HttpUrl` instance** (Pydantic v2), causing:  
   `AttributeError: 'HttpUrl' object has no attribute 'endswith'`.

3. **`flask db init` on a repo that already contains `migrations/env.py`**  
   The repository ships Alembic/Flask-Migrate metadata. Unconditional `flask db init` is **not idempotent** and fails on subsequent runs.

4. **`export PATH=...` did not reliably affect later recipe lines**  
   Even with `.ONESHELL`, relying on a separate `export PATH` line before bare `flask` led to **`flask: command not found`** in practice. Using **`$(DEV_VENV_BIN)/flask`** explicitly avoids that.

5. **Host vs container service URLs in `.env`**  
   `.env.example` used `UMP_KEYCLOAK_URL=http://keycloak:8080/auth` and GeoServer URLs aimed at the `geoserver` hostname. Those resolve **inside** the Compose network. For **host-run** Flask, they must use **published ports** (e.g. `localhost:8282`, `localhost:8181`) and the API DB port on the host (e.g. `5433` when mapped from Compose).

6. **Documentation mismatch**  
   `CONTRIBUTING.md` referred to `docker network create dev` while `DOCKER_NETWORK` in `.env` defaults to **`ump_dev`**, causing “external network not found” if the user followed the doc literally.

7. **Docker not running or network already exists**  
   After a reboot, **Docker Desktop** may not be started yet → `initiate-dev` fails when creating the network. If **`ump_dev` already exists** from an earlier run, `docker network create ump_dev` fails with *already exists*; the network is still valid for Compose.

8. **Silent wait / “hang” after `UmpSettings` during migrations**  
   `db_handler` opens a **Postgres connection pool at import** (`minconn=1`) before `flask db upgrade` prints much else. If Postgres is slow to accept connections, or TCP hangs on **`localhost`**, the terminal looks frozen. **Mitigation:** `UMP_DATABASE_CONNECT_TIMEOUT` in settings; optional `UMP_DATABASE_HOST=127.0.0.1`; slightly longer sleep in `make start-dev` after `docker compose up`.

9. **`modelserver` image build: `rasterio` + Poetry on `linux/arm64`**  
   **Rasterio** 1.3.9 has no `manylinux` wheel for **Linux aarch64**; Docker on Apple Silicon defaults to `arm64` builds, so Poetry tries to build from sdist and fails (`pkg_resources` / build errors). **Mitigation:** build/run **`modelserver` as `linux/amd64`** (same idea as other services in this repo).

10. **Process execution `POST` returns 500 (`ProgrammingError` near `sha512`)**  
   The **cache lookup query** in `Process.check_for_cache` used **malformed SQL** (extra `),` and missing `:: bytea` on the `convert_to` result) compared to the **hash expression** used when **inserting** jobs in `Job`. Anonymous **hello-world** execution hit that path and crashed. **Fix:** align the SQL with `src/ump/api/models/job.py` / migrations.

### Changes made (files)

| Area | File | Change |
|------|------|--------|
| Settings validation | `src/ump/config.py` | `ensure_trailing_slash` coerces with `str(value)` so both `str` and `HttpUrl` work. |
| Dev workflow | `Makefile` | `DEV_VENV_BIN`; `start-dev` uses `$(DEV_VENV_BIN)/flask`; conditional `flask db init`; longer DB wait; `initiate-dev` `cp && echo` fixes; clearer GeoServer note. |
| Host-side defaults | `.env.example` | Localhost Keycloak/GeoServer URLs; `UMP_GEOSERVER_DB_PORT` aligned with published API DB port on host. |
| Local overrides | `.env` | Same host-side URL/port fixes for this machine’s setup. |
| Docs | `CONTRIBUTING.md` | Network creation documents `ump_dev` and consistency with `DOCKER_NETWORK`. |
| DB connect / migrations | `src/ump/config.py`, `src/ump/api/db_handler.py`, `src/ump/main.py`, `.env.example` | `UMP_DATABASE_CONNECT_TIMEOUT` and `connect_timeout` / query string so failed DB connects fail fast; reduced long silent hangs. |
| `modelserver` on Apple Silicon | `docker-compose-dev.yaml`, `modelserver_example/Dockerfile` | `platform: linux/amd64` / `FROM --platform=linux/amd64` so `rasterio` uses a prebuilt manylinux x86_64 wheel. |
| Process execution / cache | `src/ump/api/models/process.py` | `check_for_cache` SQL fixed to match `encode(sha512(convert_to(...)::bytea), 'base64')` as in `job.py`. |

The **containerized `api` service** in `docker-compose-dev.yaml` still sets Keycloak (and related) URLs appropriate for **in-container** DNS where relevant; host `.env` is primarily for **local** Flask and Compose variable substitution.

### Quick symptom → cause reference

| Symptom | Likely cause |
|---------|----------------|
| `conda: command not found` | Makefile assumed conda CLI; fix uses `./.venv/bin/flask`. |
| `HttpUrl` / `endswith` | Pydantic v2 validator; fixed in `config.py`. |
| `flask: command not found` after PATH export | Use explicit `$(DEV_VENV_BIN)/flask` in Makefile. |
| `flask db init` “already exists” | Skip init when `migrations/env.py` present. |
| Keycloak / GeoServer unreachable from host API | `.env` must use `localhost` + external ports, not `keycloak` / `geoserver`. |
| Compose “external network not found” | Create `ump_dev` or set `DOCKER_NETWORK` to match an existing network. |
| *Cannot connect to the Docker daemon* | Start **Docker Desktop**; wait until it is ready. |
| *Network ump_dev already exists* | **Normal** if you ran `initiate-dev` before; network persists across reboots. |
| No output a long time after `UmpSettings` during `flask db` | DB not ready or slow connect; set **`UMP_DATABASE_CONNECT_TIMEOUT`**, use **`127.0.0.1`** for host, wait longer after `up`. |
| `modelserver` build fails (rasterio / Poetry) on Apple Silicon | Build **`linux/amd64`** for `modelserver` (see `docker-compose-dev.yaml` + submodule `Dockerfile`). |
| `POST .../execution` → 500, SQL error in `check_for_cache` / `sha512` | Fixed SQL in `process.py`; ensure your clone includes that change. |

### Verification

From the repository root (with Docker running and `ump_dev` present if required by Compose):

```bash
make start-dev
```

Expect: containers for `api-db`, `keycloak`, and `kc-db` up; `flask db upgrade` and `flask db current` succeed; no `conda` requirement for this target.

Optional GeoServer:

```bash
docker compose -f docker-compose-dev.yaml up -d geoserver
```

Example process execution (with **modelserver** up and `providers.yaml` pointing at `http://localhost:5005/` for host-side Flask):

```bash
curl -sS -X POST "http://127.0.0.1:5000/api/processes/modelserver:hello-world/execution" \
  -H "Content-Type: application/json" \
  -d '{"inputs": {"name": "Me", "message": "Hi there"}}'
```

Expect **HTTP 201** and JSON with `jobID` and `status` (not a 500 HTML error page from the Werkzeug debugger).

---

## Environment notes

- **“Hang” after the `UmpSettings` print during `flask db upgrade`:** importing `ump.api.db_handler` creates a `psycopg2` connection pool with `minconn=1` at import time, so a real TCP connection to Postgres is opened as soon as the app loads. If the database is not ready yet, the CLI can show no further output for a long time. Mitigations: wait a bit after `docker compose up`, set `UMP_DATABASE_CONNECT_TIMEOUT` (libpq), and consider `UMP_DATABASE_HOST=127.0.0.1` if `localhost` is slow to resolve.
- **OneDrive / cloud-synced paths** can slow Docker volume I/O; if migrations time out, increase wait time or retry once Postgres is healthy.
- **Apple Silicon**: some images use `platform: linux/amd64`; first pulls may be slower under emulation.

---

*Generated from a local setup troubleshooting session; adjust ports if your `.env` differs from `.env.example`.*
