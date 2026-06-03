# Systems Map (AI)

## 1. Service Communication Map

| Source | Destination | Protocol | Port | What's Exchanged |
|---|---|---|---|---|
| Browser | `client` | HTTP | 3000 | React app assets, SPA page navigation |
| `client` | `api` | HTTP (REST) | 8000 | JSON API requests and responses |
| `api` | `database` | PostgreSQL wire protocol | 5432 | SQL queries and result sets |
| `api` | `valkey` | RESP (Redis protocol) | 6379 | Cache reads and writes (key/value) |
| `prometheus` | `api` | HTTP scrape | 8000 | `/metrics/metrics` — Django app metrics |
| `prometheus` | `postgres_exporter` | HTTP scrape | 9187 | `/metrics` — PostgreSQL database metrics |
| `postgres_exporter` | `database` | PostgreSQL wire protocol | 5432 | DB stats queries used to build metric payloads |
| `grafana` | `prometheus` | HTTP (PromQL) | 9090 | Time-series queries for dashboard visualizations |
| `valkey-monitor` | `valkey` | RESP | 6379 | `MONITOR` command stream (read-only observation) |
| Developer machine | `api` | HTTP + debugpy | 8000 / 5678 | REST calls + remote debugger attach |

---

## 2. Data Flow Paths

### Standard User Request
```
Browser
  → client (3000)        [React app served, makes API call]
  → api (8000)           [Django processes request, queries DB]
  → database (5432)      [PostgreSQL returns data]
  → api                  [Django serializes response]
  → client               [React updates UI]
  → Browser
```

### Cached Request (Cache Hit)
```
Browser
  → client (3000)
  → api (8000)           [Django checks Valkey first]
  → valkey (6379)        [Cache hit — returns stored value immediately]
  → api                  [Skips database query]
  → client → Browser
```

### Cached Request (Cache Miss)
```
Browser
  → client (3000)
  → api (8000)           [Django checks Valkey — miss]
  → valkey (6379)        [No value found]
  → database (5432)      [Django queries PostgreSQL]
  → api                  [Writes result to Valkey, returns response]
  → client → Browser
```

### Metrics Collection (every 15s)
```
prometheus (9090)
  → scrapes api:8000/metrics/metrics          [Django app metrics]
  → scrapes postgres_exporter:9187/metrics    [DB-level metrics]
      ↑ postgres_exporter queries database:5432 to build those metrics
```

### Observability Query
```
Developer browser
  → grafana (3001)       [Loads dashboard]
  → prometheus (9090)    [Grafana issues PromQL queries]
  → grafana              [Renders time-series charts]
```

---

## 3. Network Topology

All services run on a single shared Docker network named `learningplatform` (declared external — must be created before services start). Within this network, every container is reachable by its service name (e.g., `api`, `database`, `valkey`, `prometheus`). No service needs to know the host IP; Docker's internal DNS resolves names automatically.

```
learningplatform (Docker bridge network)
├── database          (learning-platform-db)
├── api               (learning-platform-api)
├── client            (learning-platform-client)
├── prometheus
├── grafana
├── postgres_exporter
├── valkey
└── valkey-monitor
```

Host machine ports are mapped to container ports with `-p host:container`. Services communicate with each other over the internal network using container ports directly — only the host mappings exist for browser or developer access.

---

## 4. Persistence & Storage

| Volume | Mount Path in Container | Used By | What's Stored |
|---|---|---|---|
| `learning_platform_data` | `/var/lib/postgresql/data` | `database` | All PostgreSQL data files (tables, indexes, WAL) |
| `grafana_data` | `/var/lib/grafana` | `grafana` | Dashboards, users, data source configs |
| `valkey-data` | `/data` | `valkey` | RDB snapshots (written every 900s if ≥1 write) |
| `client_node_modules` | `/app/node_modules` | `client` | Node module cache (survives container restarts without reinstall) |
| `../learn-ops-api` (bind mount) | `/app` | `api` | Live API source code — changes on disk reflect immediately |
| `../learn-ops-client` (bind mount) | `/app` | `client` | Live client source code — changes on disk reflect immediately |

Bind mounts (`../learn-ops-api`, `../learn-ops-client`) link the host working directory into the container, enabling live reload during development. Named volumes (`learning_platform_data`, `grafana_data`, `valkey-data`) are managed by Docker and persist across `docker compose down` — only `docker compose down -v` removes them.

---

## 5. Observability Pipeline

```
Django API  ──────── /metrics/metrics ──────────────────────────┐
                                                                  ▼
PostgreSQL ── postgres_exporter:9187/metrics ──────► prometheus (9090)
                                                                  │
                                                                  ▼
                                                         grafana (3001)
                                                    [dashboards + alerts]
```

| Stage | Service | Role |
|---|---|---|
| Instrumentation | `api` (Django) | Exposes app metrics (request counts, latency, etc.) at `/metrics/metrics` |
| DB instrumentation | `postgres_exporter` | Translates PostgreSQL internal stats into Prometheus-format metrics |
| Collection | `prometheus` | Scrapes both endpoints every 15s; stores time-series data |
| Visualization | `grafana` | Queries Prometheus via PromQL; renders dashboards |

Valkey activity is observed separately via `valkey-monitor`, which streams every command processed by Valkey to its container log in real time (not integrated into Prometheus).
