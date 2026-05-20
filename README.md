# TP13 — Docker Evaluation

## Project Structure

```
tp13/
├── .github/
│   └── workflows/
│       └── docker.yml
├── api/
│   ├── app.js
│   ├── package.json
│   ├── Dockerfile
│   └── .dockerignore
├── nginx/
│   └── default.conf
├── monitoring/
│   ├── prometheus.yml
│   └── grafana/
│       ├── provisioning/
│       │   ├── datasources/prometheus.yml
│       │   └── dashboards/dashboard.yml
│       └── dashboards/
│           └── api-dashboard.json
├── captures/
├── docker-compose.yml
├── docker-compose.registry.yml
├── docker-compose.prod.yml
├── .env
└── README.md
```

---

## Part 1 — API & Dockerfile

The Node.js Express API lives in `api/`. It exposes three routes:

- `GET /` — returns the container hostname (`os.hostname()`), the `PET` environment variable value, and a request counter incremented on each call
- `GET /healthz` — returns `{ "status": "ok" }` with HTTP 200, used by the Docker HEALTHCHECK
- `GET /metrics` — Prometheus metrics exposed via `prom-client` (request counter + default Node.js metrics)

### Dockerfile highlights

- Base image: `node:20-alpine`
- `COPY package*.json ./` first to leverage Docker layer cache for `npm install`
- `RUN npm install --only=production` — no dev dependencies in the image
- `COPY . .` after installing dependencies
- Non-root user: creates `appuser` in `appgroup` and switches to it with `USER appuser`
- `HEALTHCHECK` targets `/healthz` with `--interval=10s --timeout=5s --start-period=15s --retries=3`
- `.dockerignore` excludes `node_modules`, `.env`, `.git`

---

## Part 2 — Private Registry

The private registry stack is defined in `docker-compose.registry.yml` and runs independently from the main stack.

**Start the registry:**
```bash
docker compose -f docker-compose.registry.yml --env-file .env up -d
```

**Build and push the API image:**
```bash
docker build -t localhost:5001/mon-api:1.0.0 ./api
docker push localhost:5001/mon-api:1.0.0
```

The `image:` field in `docker-compose.yml` references `localhost:${REGISTRY_PORT:-5001}/mon-api:1.0.0`.

**Screenshot — Registry UI (http://localhost:8080) showing the image listed:**

![Registry UI](captures/registry-ui.png)

---

## Part 3 — Compose Stack & Nginx

The main `docker-compose.yml` orchestrates three services on a custom bridge network `internal`:

| Service | Image | PET | Exposed port |
|---------|-------|-----|-------------|
| `cat` | `localhost:5001/mon-api:1.0.0` | `cat` | none (internal only) |
| `dog` | `localhost:5001/mon-api:1.0.0` | `dog` | none (internal only) |
| `nginx` | `nginx:alpine` | — | `${NGINX_PORT}:80` |

`nginx` depends on `cat` and `dog` with `condition: service_healthy`, so it only starts once both API containers pass their healthcheck.

**Nginx routing (`nginx/default.conf`):**
- `GET /` — round-robin between `cat:3000` and `dog:3000`
- `GET /cat` — exclusively to `cat:3000` (path stripped via trailing slash on `proxy_pass`)
- `GET /dog` — exclusively to `dog:3000` (path stripped via trailing slash on `proxy_pass`)

**Start the main stack:**
```bash
docker compose up -d
```

---

## Part 4 — Security

### Environment variables

All configurable values are defined in `.env` — no hardcoded values in `docker-compose.yml` or the Dockerfile:

```env
API_PORT=3000
PET_CAT=cat
PET_DOG=dog
NGINX_PORT=8090
GRAFANA_PORT=3001
PROMETHEUS_PORT=9091
REGISTRY_PORT=5001
REGISTRY_UI_PORT=8080
PORTAINER_PORT=9001
```

### Trivy scan

```bash
trivy image localhost:5001/mon-api:1.0.0
```

**Screenshot — Trivy scan output:**

![Trivy Scan](captures/trivy-scan.png)

### Why `node:20-alpine` and not `node:latest`?

`node:latest` is based on Debian Bookworm (~950 MB). It ships hundreds of system packages (compilers, utilities, libraries) that are completely unnecessary at runtime. Each one is a potential CVE surface.

`node:20-alpine` is based on Alpine Linux (~130 MB total image size). It contains only the OS packages required to run Node.js. The result: far fewer CVEs reported by Trivy. In practice, `node:latest` may report dozens of HIGH/CRITICAL CVEs from system-level packages, while `node:20-alpine` reports zero CRITICAL CVEs on this project.

Additionally, a smaller image means faster pulls, less storage, and a reduced attack surface in production.

---

## Part 5 — Stack Validation

**Screenshot — `docker compose ps` showing all services Up (healthy):**

![Compose PS](captures/compose-ps.png)

**Screenshot — Load balancing: consecutive calls to `GET /` alternate between `cat` and `dog` hostnames:**

![Load Balancing](captures/load-balancing.png)

**Screenshot — `/cat` returns `pet: cat`, `/dog` returns `pet: dog`, counters differ between the two:**

![Cat Dog Routes](captures/cat-dog-routes.png)

---

## Part 6 — Theoretical Questions

### Question: Docker Swarm — `docker compose up` vs `docker stack deploy`

`docker compose up` is a local development tool. It reads a `docker-compose.yml`, creates containers directly on the local Docker engine, and manages their lifecycle (start, stop, logs) on a single machine.

`docker stack deploy` deploys a stack onto a **Docker Swarm cluster** (one or more nodes). It distributes services across nodes with built-in features like automatic replication, rolling updates, and failure recovery across the cluster.

**Why can't `build:` be used with Swarm?**

When deploying a stack with `docker stack deploy`, the services are scheduled across multiple Swarm worker nodes. Those workers only know how to **pull** images from a registry — they do not have access to the source code on the manager node. The `build:` directive requires the source code to be present locally and Docker to build the image on-the-fly. This is incompatible with distributed scheduling. The correct workflow for Swarm is: build the image locally → push to a registry → reference the image via `image:` in the stack file.

---

### Question: Docker Secrets vs environment variables

**Environment variable (via `.env` / `environment:`):**
The value is stored in plain text in the container configuration, visible via `docker inspect`, in shell history, and in logs if printed. It is convenient but insecure for sensitive data like passwords or tokens.

**Docker Secret:**
A Swarm-native mechanism where the secret value is encrypted in transit and at rest (stored in the Raft consensus log). It is never exposed as an environment variable.

**Where is the secret accessible inside the container?**
It is mounted as a file at `/run/secrets/<secret-name>`.

**How to read it from Node.js:**
```js
const fs = require('fs');
const dbPassword = fs.readFileSync('/run/secrets/db_password', 'utf8').trim();
```

---

### Question: Production Backup Strategy

**What must absolutely be backed up:**

| Category | Examples | Why |
|----------|---------|-----|
| **Persistent volumes** (irreplaceable) | Database data, Grafana dashboards, Registry images | Created at runtime; cannot be rebuilt from code |
| **Secret config** (irreplaceable) | `.env` files with passwords/tokens, TLS certificates and private keys | Not in Git; lost if the server dies |

**What is automatically recreatable:**

| Category | Examples | Why |
|----------|---------|-----|
| Docker images | All images | Rebuilt from Dockerfiles via CI/CD |
| Containers | All containers | Ephemeral by design |
| Versioned config | `docker-compose.yml`, `nginx/default.conf`, `prometheus.yml` | Already in Git |

**Recommended backup commands for volumes:**
```bash
# Backup a named volume to a tar archive
docker run --rm -v tp13_grafana-data:/data alpine tar czf - /data > grafana-backup.tar.gz

# Restore
docker run --rm -v tp13_grafana-data:/data alpine tar xzf - -C / < grafana-backup.tar.gz
```

Store backups in an external, encrypted location (S3, off-site server).

---

## Part 7 — Observability & Production

The monitoring stack is integrated into `docker-compose.yml`:

| Service | Port | Purpose |
|---------|------|---------|
| **Prometheus** | `9091` | Scrapes metrics from API (`/metrics`), node-exporter, cAdvisor |
| **Grafana** | `3001` | Dashboards — auto-provisioned from `monitoring/grafana/` |
| **node-exporter** | internal | Host system metrics (CPU, memory, disk) |
| **cAdvisor** | internal | Per-container resource metrics |
| **Portainer** | `9001` | Docker management UI |

Grafana is provisioned automatically via:
- `monitoring/grafana/provisioning/datasources/prometheus.yml` — connects to Prometheus
- `monitoring/grafana/provisioning/dashboards/dashboard.yml` — loads dashboards from disk
- `monitoring/grafana/dashboards/api-dashboard.json` — the actual dashboard (requests, memory, CPU)

The `docker-compose.prod.yml` override file adds CPU and RAM limits on every service via `deploy.resources`. Apply it with:
```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Part 8 — Volumes

**Named volumes** (managed by Docker, for persistent data):

| Volume | Service | Purpose |
|--------|---------|---------|
| `prometheus-data` | prometheus | Time-series data |
| `grafana-data` | grafana | Dashboards, users, settings |
| `portainer-data` | portainer | Portainer state |
| `registry-data` | registry | Pushed Docker images |

**Bind mounts** (config files injected from host, versioned in Git):

| Host path | Container path | Service |
|-----------|---------------|---------|
| `./nginx/default.conf` | `/etc/nginx/conf.d/default.conf` | nginx |
| `./monitoring/prometheus.yml` | `/etc/prometheus/prometheus.yml` | prometheus |
| `./monitoring/grafana/provisioning` | `/etc/grafana/provisioning` | grafana |
| `./monitoring/grafana/dashboards` | `/var/lib/grafana/dashboards` | grafana |

**Why this split?** Named volumes are used for data that must survive container restarts and is created at runtime (Grafana state, Prometheus TSDB). Bind mounts are used for configuration that is version-controlled — so any change to a config file is immediately reflected without rebuilding the image.

**Screenshot — `docker volume ls`:**

![Volume LS](captures/volume-ls.png)

**Screenshot — `docker volume inspect tp13_grafana-data`:**

![Volume Inspect](captures/volume-inspect.png)

---

## Part 9 — CI/CD with GitHub Actions

The workflow `.github/workflows/docker.yml` triggers on every push to `main`.

**Steps:**
1. **Checkout** — fetches the repository
2. **Set up Docker Buildx** — enables BuildKit
3. **Login to GHCR** — authenticates to `ghcr.io` using the automatic `GITHUB_TOKEN`
4. **Build image** — builds from `./api`, loaded locally (not pushed yet)
5. **Trivy scan** — scans the built image; the pipeline **fails** if any `CRITICAL` CVE is found
6. **Push** — pushes the image to `ghcr.io/ianaliti/mon-api:git-<SHA>`

No extra secrets required — `GITHUB_TOKEN` is provided automatically by GitHub Actions.

**Screenshot — GitHub Actions pipeline in success:**

![GitHub Actions](captures/github-actions.png)

---

## Part 10 — VPS Deployment

The full stack is deployed on the VPS at **http://78.138.58.14**.

| Service | URL |
|---------|-----|
| API via Nginx | http://78.138.58.14 |
| Prometheus | http://78.138.58.14:9090 |
| Grafana | http://78.138.58.14:3001 |
| Portainer | http://78.138.58.14:9000 |

**Screenshot — `docker compose ps` on the VPS (all services Up healthy):**

![VPS Compose PS](captures/vps-compose-ps.png)

---

## How to run this project

**Prerequisites:** Docker and Docker Compose installed.

```bash
# 1. Clone the repository
git clone https://github.com/ianaliti/tp13.git
cd tp13

# 2. Start the private registry
docker compose -f docker-compose.registry.yml --env-file .env up -d

# 3. Build and push the API image
docker build -t localhost:5001/mon-api:1.0.0 ./api
docker push localhost:5001/mon-api:1.0.0

# 4. Start the main stack
docker compose up -d

# 5. Verify all services are healthy
docker compose ps
```
