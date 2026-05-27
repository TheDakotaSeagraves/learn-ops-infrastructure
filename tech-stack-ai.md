# Tech Stack (AI)

## 1. Run Questions

### 1a. Config Files

| Config File | Location | Config Value | What it's for | How it's used |
|---|---|---|---|---|
| `.env` | `/` | `POSTGRES_DB` | Name of the PostgreSQL database | Used by the `database` and `postgres_exporter` services via `env_file` |
| `.env` | `/` | `POSTGRES_USER` | PostgreSQL login username | Referenced in healthcheck, connection strings, and exporter auth |
| `.env` | `/` | `POSTGRES_PASSWORD` | PostgreSQL login password | Used to authenticate the database connection |
| `.env` | `/` | `DATA_SOURCE_NAME` | Full DB connection URI for postgres_exporter | Tells the exporter how to connect to PostgreSQL |
| `docker-compose.yml` | `/` | `env_file` (per service) | Points each service to its own `.env` file | Injects environment variables into the container at runtime |
| `docker-compose.yml` | `/` | `depends_on` (per service) | Defines startup order between services | Ensures database is healthy before API starts; API is up before Prometheus scrapes |
| `docker-compose.yml` | `/` | `networks: learningplatform` | Shared Docker network name | Allows all services to communicate by container name |
| `prometheus.yml` | `/` | `scrape_interval` | How often Prometheus collects metrics | Controls polling frequency for all scrape jobs |
| `prometheus.yml` | `/` | `evaluation_interval` | How often Prometheus evaluates alert rules | Controls alerting responsiveness |
| `prometheus.yml` | `/` | `metrics_path` | URL path for the Django metrics endpoint | Tells Prometheus where to fetch app metrics on the `api` service |
| `valkey/docker-compose.yml` | `/valkey` | `--save 900 1` | RDB persistence config (save if 1 write in 900s) | Determines how often Valkey snapshots data to disk |
| `valkey/docker-compose.yml` | `/valkey` | `--loglevel` | Valkey log verbosity | Controls what gets written to the container log |
| `valkey/docker-compose.yml` | `/valkey` | `restart: unless-stopped` | Container restart policy | Keeps Valkey running automatically after crashes or reboots |

### 1b. How to Start It

Run targets from the repo root with `make <target>`.

| Target | Command | What it does | When to use it |
|---|---|---|---|
| `setup` | `./scripts/setup.sh` | Runs first-time setup (creates network, env files, etc.) | First time only, before any `up` target |
| `doctor` | `./scripts/setup.sh --doctor` | Checks that prerequisites and config are in place | Troubleshooting; verify environment before starting |
| `up` | `docker compose up --build -d` | Builds and starts **all** services in detached mode | Normal full-stack start |
| `up-api` | `docker compose up --build -d api` | Builds and starts only the **API** service | Working on the backend only; skips client build |
| `up-client-api` | `docker compose up --build -d api client` | Builds and starts the **API and client** together | Frontend + backend work; skips monitoring services |
| `restart` | `down` then `up --build -d` | Tears down all containers then rebuilds and restarts everything | Picking up config changes that require a full cycle |

**How they differ:** `up`, `up-api`, and `up-client-api` all build before starting, but target different subsets of services — use the narrowest one for faster iteration. `restart` is equivalent to `down` + `up` and forces a clean container restart. `setup` and `doctor` are not start commands — `setup` must run once before the first `up`, and `doctor` is for diagnostics only.

### 1c. Where to Access It

| Service | Port | URL |
|---|---|---|
| API | 8000 | http://localhost:8000 |
| API Debugger | 5678 | http://localhost:5678 |
| Client | 3000 | http://localhost:3000 |
| Grafana | 3001 | http://localhost:3001 |
| Prometheus | 9090 | http://localhost:9090 |
| Postgres Exporter | 9187 | http://localhost:9187/metrics |
| Database (PostgreSQL) | 5432 | TCP only — no browser URL |
| Valkey | 6379 | TCP only — no browser URL |

### 1d. Service Dependencies

| Service | Depends On | Why |
|---|---|---|
| `api` | `database` | API requires a healthy PostgreSQL connection before it can start; enforced with a healthcheck condition |
| `prometheus` | `api` | Prometheus scrapes metrics from the API's `/metrics/metrics` endpoint — API must be running first |
| `grafana` | `prometheus` | Grafana reads time-series data from Prometheus; Prometheus must be up to serve as a data source |
| `postgres_exporter` | `database` | Exporter connects directly to PostgreSQL to collect DB metrics; database must be up first |
| `valkey-monitor` | `valkey` | Runs `valkey-cli monitor` against the Valkey container; Valkey must be running to connect |
| `database` | _(none)_ | Base service with no upstream dependencies |
| `client` | _(none declared)_ | No `depends_on` in compose, but functionally depends on `api` being reachable at runtime |
| `valkey` | _(none)_ | Base service with no upstream dependencies |

### 1e. Main Entry Points

| Service | Startup File | Routes / URL Config File |
|---|---|---|
| | | |
| | | |
| | | |

## 2. Services

| Service Name | Tech Stack (including version) | Purpose |
|---|---|---|
| | | |

## 3. System Overview
