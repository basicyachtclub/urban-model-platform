# Test the example model on your Mac (beginner guide)

## What this file is for

This document is a **step-by-step checklist** for people who are **not software developers** but want to run the Urban Model Platform **on their own Mac**, everything **local**, and try the **bundled example model** (a small demo “process” that runs on a separate toy server).

You will use:

- **One Terminal window** to start **Docker** services (database, login server, and the example model server). You run commands there, wait until they finish, and sometimes leave Docker running in the background.
- **A second Terminal window** to start the **main platform app** (the API). You **leave this window open** while you test; closing it stops the API.

**Success** means: you can open a link in your browser (or paste a command) and see that the platform is healthy, lists processes, and can start a **hello-world** job.

Unless a step says otherwise, run commands **from the Urban Model Platform project folder** (the folder you cloned from GitHub). In Terminal, you “go there” with the `cd` command (explained below).

**About paths:** replace `/path/to/urban-model-platform` with the real folder path on your Mac. For example, if the project is in your Documents folder, it might look like `cd ~/Documents/urban-model-platform`.

---

## What is the Urban Model Platform?

In simple terms, the **Urban Model Platform (UMP)** is **middleware** for **urban digital twins**: it helps other software discover and run **simulation models** (for example models that turn inputs into results for planning or analysis). It does not replace a full “digital twin” app by itself; it provides a **standard web API** so many tools can plug in.

Technically, UMP implements the **[OGC API Processes](https://ogcapi.ogc.org/processes/)** standard: models are exposed as **processes** you can list and **execute**. The platform stores jobs and metadata in **PostgreSQL**, uses **Keycloak** for sign-in in real deployments, and can work with **GeoServer** for map-related results. The **example model server** in this repository is a small **pygeoapi** application that pretends to be an external model provider so you can try everything on your laptop.

---

## Note about the public website

While preparing this guide, the main documentation sites (for example URLs under `citysciencelab.github.io` and `urban-model-platform.github.io`) returned **errors or timeouts**. This guide therefore follows the **documentation and configuration inside this GitHub repository** (`README`, `docs/`, `CONTRIBUTING.md`), which match the code you actually cloned.

---

## Before you start (macOS)

1. **Docker Desktop** is installed from [docker.com](https://www.docker.com/products/docker-desktop/) and **running** (you should see a whale icon in the menu bar at the top of the screen).
2. **Terminal** is available (Applications → Utilities → Terminal).
3. **Git** is available (Apple’s developer tools or Git from Homebrew). You already **cloned** the Urban Model Platform repository.
4. **Patience on first run:** the first time, your Mac may download large Docker images and **build** the example model server; this can take **many minutes** and needs free disk space.

---

## Terminal 1 — Dependencies and the example model server

### 1. Open Terminal and go to the project

1. Open **Terminal**.
2. Type `cd ` (with a space after `cd`), then **drag the project folder** from Finder into the Terminal window and press **Enter**.  
   Or type the path by hand, for example:

```bash
cd /path/to/urban-model-platform
```

### 2. One-time: load the example model code (submodule)

The example model lives in a **sub-module** folder. You only need this once per clone (or after a fresh download):

```bash
git submodule update --init --recursive
```

Wait until it finishes. If Git asks for credentials, use the same access you use for GitHub.

### 3. One-time: initial developer setup

This creates configuration files, a Python environment folder (`.venv`), and a Docker network. **First run can take a long time.**

```bash
make initiate-dev
```

If you see an error that the Docker network **already exists**, that is usually fine.

If you **already** ran `make initiate-dev` successfully before, you can skip this step.

### 4. Tell the platform where to find the example model (important)

The file **`providers.yaml`** lists **model servers** UMP may call. The template copied from `providers.yaml.example` uses a Docker-only address (`modelserver`) that **does not work** when the main API runs **on your Mac** outside Docker.

1. Open **`providers.yaml`** in **TextEdit**, **VS Code**, or another text editor (it sits in the project root next to `README.md`).
2. Find the line that looks like:

   `url: "http://modelserver:5000"`

3. Change it to (note **`localhost`** and **`5005`**, and the trailing slash):

   `url: "http://localhost:5005/"`

**Why:** Docker publishes the example model server to your Mac on port **5005** by default (see `PYGEOAPI_SERVER_PORT_EXTERNAL` in `.env.example`). If you changed that port in your `.env`, use the same port here instead of `5005`.

**If you ever run the UMP API inside Docker** on the same Compose network, you would use the Docker service name again; this guide assumes **local Flask on the Mac**, which is what `CONTRIBUTING.md` describes.

### 5. Start databases, Keycloak, and migrations

Still in **Terminal 1**:

```bash
make start-dev
```

Wait until it finishes without errors. It starts **Postgres** and **Keycloak** (and Keycloak’s database) and applies database migrations.

### 6. Start the example model server container

Either run the full “example” target (this runs `start-dev` again, then starts the model server):

```bash
make start-dev-example
```

**Or**, if you **just** ran `make start-dev` and do not want to repeat it:

```bash
docker compose -f docker-compose-dev.yaml up -d modelserver
```

The **first** time, Docker may **build** an image; this can take many minutes.

### 7. Check that containers are running

```bash
docker compose -f docker-compose-dev.yaml ps
```

You want rows such as **api-db**, **keycloak**, **kc-db**, and **modelserver** to show **running** (or “Up”). If **modelserver** is missing or exited, scroll up in Terminal for error messages.

You can leave **Terminal 1** open or close it; **Docker Desktop** keeps the containers running in the background.

---

## Terminal 2 — Start the main API (leave this open)

### 1. Open a new Terminal tab or window

- **New tab:** Shell → New Tab, or press **Cmd–T**.
- **New window:** Shell → New Window.

### 2. Go to the same project folder again

```bash
cd /path/to/urban-model-platform
```

### 3. Start the Flask application

```bash
./.venv/bin/flask -A src/ump/main.py --debug run
```

If that fails with “No such file”, your environment may differ; try:

```bash
flask -A src/ump/main.py --debug run
```

**Leave this Terminal window open.** While it is running, your Mac listens for the API at **`http://127.0.0.1:5000`** by default.

To stop the API later, click that Terminal window and press **Ctrl–C**.

---

## Check that everything works

### 1. Health check (browser)

In **Safari** or **Chrome**, open:

**http://127.0.0.1:5000/api/health/ready**

You should see JSON containing **`"status": "ok"`**. If the page does not load, the API in Terminal 2 is probably not running.

### 2. List processes (browser)

Open:

**http://127.0.0.1:5000/api/processes/**

You should see JSON. If the example model server is reachable, you should be able to find something like **hello-world** in the content.

### 3. Run hello-world with Terminal (optional)

The platform names processes as **`provider_key:process_id`**. With the default `providers.yaml.example` structure, the provider key is **`modelserver`** and the demo process is **`hello-world`**, so the id is:

**`modelserver:hello-world`**

In **another** Terminal tab (Terminal 3 is optional—you can reuse any tab **except** the one where Flask is running), `cd` to the project and run:

```bash
curl -sS -X POST "http://127.0.0.1:5000/api/processes/modelserver:hello-world/execution" \
  -H "Content-Type: application/json" \
  -d '{"inputs": {"name": "Me", "message": "Hi there"}}'
```

You should get JSON back with a **job** identifier and status. (This mirrors the example in `modelserver_example/README.md`, but goes through UMP instead of calling pygeoapi directly.)

**Sign-in:** other processes (for example **squareroot**) may require a **logged-in user** and a **Bearer token** from Keycloak. **hello-world** is configured in the example as **anonymous-friendly** for getting started.

---

## GeoServer and “hello-geo-world”

The example process **hello-geo-world** uses **GeoServer** for results. For your **first** test, stay with **hello-world** only.

If you need GeoServer later, start it with Docker (from the project root):

```bash
docker compose -f docker-compose-dev.yaml up -d geoserver
```

Then open **http://localhost:8181/geoserver/web/** (default dev ports from `.env.example`). Your `.env` should use **localhost** URLs for host-side Flask if you followed the local setup notes in this repository.

---

## If something goes wrong

- **Docker not running** — Start Docker Desktop; wait until it is “ready”.
- **`providers.yaml` still points at `modelserver`** — The API on your Mac cannot resolve that name; use **`http://localhost:5005/`** (or your custom port).
- **Model server container not running** — Run `docker compose -f docker-compose-dev.yaml ps` and check **modelserver**; read logs with `docker compose -f docker-compose-dev.yaml logs modelserver`.
- **Docker build fails when starting `modelserver`** (example: `poetry install` errors mentioning **rasterio** or `ModuleNotFoundError: No module named 'pkg_resources'`) — On **Apple Silicon**, Docker often builds a **linux/arm64** image; **`rasterio` may have no wheel for that**, so Poetry tries to compile from source and fails. This repository fixes that by building/running the **modelserver** image as **`linux/amd64`** (see `platform` in `docker-compose-dev.yaml` and `FROM --platform=linux/amd64` in `modelserver_example/Dockerfile`), which uses the prebuilt **manylinux x86_64** wheel under emulation. If you still see errors: update Docker Desktop; enable **Rosetta** for x86_64/amd64 Linux if your Docker version offers it; share the full build log when asking for help. Until the model server image builds successfully, **`http://127.0.0.1:5000/api/processes/` may return an empty list** `{"processes":[]}` even when the rest of the stack is fine.
- **Flask not started** — Terminal 2 must show the server running; health URL should work.
- **Port already in use** — Another program may be using **5000** or **5005**; stop that program or change ports in `.env` / Flask and update `providers.yaml` accordingly.
- **First run very slow** — Image download and build can take a long time; wait and try again.

For more command references, see [local-dev-commands.md](local-dev-commands.md) and [local-dev-setup-findings.md](local-dev-setup-findings.md).
