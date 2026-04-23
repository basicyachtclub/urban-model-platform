# Urban Model Platform — basic API usage

Practical **HTTP** reference: base URL, common endpoints, and where they are implemented. Assumes a local run with Flask on `http://127.0.0.1:5000` and the usual `UMP_API_SERVER_URL_PREFIX=/api` (see [`.env.example`](../.env.example) and [local dev commands](local-dev-commands.md)).

**Convention:** all paths below are **relative to the API base** `http://127.0.0.1:5000` + your prefix, e.g. `http://127.0.0.1:5000/api/...` when the prefix is `/api`.

## Configuration

| Setting | Role |
|--------|------|
| `UMP_API_SERVER_URL` | Public base URL of the API (used in some responses and links, e.g. job metadata). |
| `UMP_API_SERVER_URL_PREFIX` | Path prefix for all API blueprints (default in code is `/`; [`.env.example`](../.env.example) uses `/api`). |
| `providers.yaml` | Declares which **providers** and **processes** exist and how to reach each model server. UMP only proxies to processes that are allow-listed here. |

**Route registration** (prefix + sub-blueprints) lives in [`src/ump/main.py`](../src/ump/main.py) (`api` blueprint with `url_prefix=config.UMP_API_SERVER_URL_PREFIX`, then `/processes`, `/jobs`, `/health`, etc.).

---

## 1. Readiness (database)

- **GET** `…/health/ready`

Returns `{"status": "ok"}` when the database connection works, or **503** with `{"status": "error"}` otherwise.

**Code:** [`src/ump/api/routes/health.py`](../src/ump/api/routes/health.py)

```bash
curl -sS "http://127.0.0.1:5000/api/health/ready"
```

---

## 2. Processes (discover and run models)

### List all processes (aggregated from configured providers)

- **GET** `…/processes/`

**Code:** [`src/ump/api/routes/processes.py`](../src/ump/api/routes/processes.py) → `load_processes()`

```bash
curl -sS "http://127.0.0.1:5000/api/processes/"
```

### List provider metadata (sanitized)

- **GET** `…/processes/providers`

Strips sensitive fields (URLs, auth details, some per-process flags) before returning. Note: the route comment in code indicates this may list processes even when a provider has `exclude: True` in config — see implementation if you depend on that behavior.

**Code:** [`src/ump/api/routes/processes.py`](../src/ump/api/routes/processes.py)

```bash
curl -sS "http://127.0.0.1:5000/api/processes/providers"
```

### Describe one process

- **GET** `…/processes/{providerPrefix}:{processId}`

Process identifiers must match **`provider:process_id`** (colon-separated). Example: `modelserver:hello-world`. If the id is malformed or unknown, the API returns an **OGC API Processes**–style error payload (`application/problem+json`).

**Code:** [`src/ump/api/routes/processes.py`](../src/ump/api/routes/processes.py), [`src/ump/api/models/process.py`](../src/ump/api/models/process.py) (`Process` class)

```bash
curl -sS "http://127.0.0.1:5000/api/processes/modelserver%3Ahello-world"
```

(Encode `:` as `%3A` in the path if your shell or client is picky.)

### Execute a process (create a job)

- **POST** `…/processes/{providerPrefix}:{processId}/execution`  
- **Request body:** JSON, **OGC API Processes**–style. At minimum, include an **`inputs`** object whose keys and value types must match the process description (the server validates required inputs and schema when `inputs` are defined). UMP will set **`mode`** to **`async`** on the way to the provider. Optional: **`job_name`**.

- **Response:** **201** with JSON like `{"jobID": "…", "status": "…"}` (see `Process.execute`).

**Code:** [`src/ump/api/routes/processes.py`](../src/ump/api/routes/processes.py), [`src/ump/api/models/process.py`](../src/ump/api/models/process.py) (`validate_exec_body`, `execute`).

**Authentication:** if you send a **Bearer** token, UMP decodes it (Keycloak) and attaches the user’s `sub` to the job; some deployments allow anonymous runs depending on process configuration.

```bash
# Example: minimal body — replace inputs with what the process actually expects
curl -sS -X POST "http://127.0.0.1:5000/api/processes/modelserver%3Ahello-world/execution" \
  -H "Content-Type: application/json" \
  -d '{"inputs": {}}'
```

With a token:

```bash
curl -sS -X POST "http://127.0.0.1:5000/api/processes/modelserver%3Ahello-world/execution" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -d '{"inputs": {}}'
```

Token handling is in `check_jwt` in [`src/ump/main.py`](../src/ump/main.py): if `Authorization: Bearer` is present, the token is verified via Keycloak; if absent, `g.auth_token` is `None` and some routes still work depending on the resource.

---

## 3. Jobs (status and results)

### List jobs

- **GET** `…/jobs/`

Query parameters are passed through to the backend (e.g. filters); see `get_jobs` in [`src/ump/api/jobs.py`](../src/ump/api/jobs.py).

**Code:** [`src/ump/api/routes/jobs.py`](../src/ump/api/routes/jobs.py)

```bash
curl -sS "http://127.0.0.1:5000/api/jobs/"
```

### Get one job

- **GET** `…/jobs/{jobID}`

Optional query: `additionalMetadata=true` to include extra metadata in the `Job.display()` output.

**Code:** [`src/ump/api/routes/jobs.py`](../src/ump/api/routes/jobs.py), [`src/ump/api/models/job.py`](../src/ump/api/models/job.py)

```bash
curl -sS "http://127.0.0.1:5000/api/jobs/YOUR_JOB_ID"
```

### Get job results

- **GET** `…/jobs/{jobID}/results`

Returns JSON with the structured results UMP assembles (including links or payloads depending on the provider and `result-storage` settings).

**Code:** [`src/ump/api/routes/jobs.py`](../src/ump/api/routes/jobs.py) (`asyncio.run(job.results())`).

```bash
curl -sS "http://127.0.0.1:5000/api/jobs/YOUR_JOB_ID/results"
```

### Other job-related routes (optional)

The same [`jobs` blueprint](../src/ump/api/routes/jobs.py) also defines:

- `GET` `…/jobs/{jobID}/users` — users with access to the job  
- `GET` / `POST` `…/jobs/{jobID}/comments` — comments  
- `GET` `…/jobs/{jobID}/share/{email}` — share with another user (requires auth)  

Respectively require or ignore authentication as implemented in each handler.

---

## 4. Quick reference (paths)

Assume prefix `/api` and `BASE=http://127.0.0.1:5000/api`.

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/health/ready` | DB readiness |
| GET | `/processes/` | List processes |
| GET | `/processes/providers` | Provider list (redacted) |
| GET | `/processes/{prefix:processId}` | Process description |
| POST | `/processes/{prefix:processId}/execution` | Start job |
| GET | `/jobs/` | List jobs |
| GET | `/jobs/{jobID}` | Job details |
| GET | `/jobs/{jobID}/results` | Job results |

---

## 5. Where this is implemented

| Area | File |
|------|------|
| App factory, CORS, JWT, blueprint mount | [`src/ump/main.py`](../src/ump/main.py) |
| Process routes | [`src/ump/api/routes/processes.py`](../src/ump/api/routes/processes.py) |
| Job routes | [`src/ump/api/routes/jobs.py`](../src/ump/api/routes/jobs.py) |
| Health | [`src/ump/api/routes/health.py`](../src/ump/api/routes/health.py) |
| Process execution & validation | [`src/ump/api/models/process.py`](../src/ump/api/models/process.py) |
| Job model & result assembly | [`src/ump/api/models/job.py`](../src/ump/api/models/job.py) |
| Config defaults | [`src/ump/config.py`](../src/ump/config.py) |

---

## 6. GeoServer and result storage (heads-up)

If a process in `providers.yaml` uses **`result-storage: geoserver`**, UMP and scheduled cleanup interact with GeoServer (layers/datastores) when old jobs are removed. That does not change the **HTTP** surface above, but it explains some operational requirements (GeoServer URL, credentials, workspace). See cleanup logic referenced from [`src/ump/main.py`](../src/ump/main.py) and provider/process metadata in your config.

---

## 7. OpenAPI

The app uses **APIFlask**; when enabled in your deployment, you may have interactive docs (commonly under `/docs` or similar on the same host). If your build exposes them, you can use that for a full, machine-generated schema. This guide stays aligned with the **Python route modules** as the source of truth.

For a gentle first run with the bundled example process, see [Test the example model on your Mac (beginner guide)](test-example-model-beginner.md).
