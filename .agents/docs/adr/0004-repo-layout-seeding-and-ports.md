# ADR-0004: Repository Layout, Data Seeding, and Host Port Allocations

## Status
Accepted

## Context
Before beginning implementation, concrete specifications were needed for:
1. Physical repository directory layout.
2. Catalog database seeding scale to guarantee observable query degradation during un-indexed search tests.
3. Host port assignments, accounting for local host port availability.

A port scan on the development machine revealed that port `5432` is actively in use by a local PostgreSQL process (`PID 7340`). All other target ports (`8080`, `8000`, `6379`, `9090`, `3000`) are free.

## Decisions

### 1. Host Port Mappings
To eliminate conflict with the host's existing PostgreSQL service while maintaining clean local access:
* **Nginx (Media & Proxy):** Host `8080` -> Container `80`
* **FastAPI (Application API):** Host `8000` -> Container `8000`
* **PostgreSQL:** Host `5433` -> Container `5432` *(avoiding host 5432 collision; containers communicate via `postgres:5432` on Docker network)*
* **Redis:** Host `6379` -> Container `6379`
* **Prometheus:** Host `9090` -> Container `9090`
* **Grafana:** Host `3000` -> Container `3000`

### 2. Database Seeding Strategy
* **Volume:** 10,000 synthetic video metadata records generated via a fast SQL/Python seed script during initial container setup.
* **Fields:** `id`, `title`, `description`, `category`, `tags`, `duration_seconds`, `hls_manifest_path`, `created_at`.
* **Search Benchmark:** Full-text / LIKE query against `title` and `description`.
* **Bottleneck Mechanics:**
  * When `FLAW_SEARCH_INDEX=false`: Query forces a sequential table scan across 10,000 rows. With pool limit constrained to 5 connections, concurrent search requests from k6 rapidly queue up, driving p95 latency past 2,000ms.
  * When `FLAW_SEARCH_INDEX=true`: Postgres GIN / B-tree index is applied, dropping p95 latency to <15ms.

### 3. Repository Directory Structure
```text
├── services/
│   ├── api/              # FastAPI application (routes, models, flaw configs)
│   ├── nginx/            # Nginx reverse proxy & static media cache configuration
│   └── player/           # HTML5 + hls.js web player with telemetry reporting
├── load-tests/           # k6 load generation scripts (scenarios & stages)
├── canaries/             # Playwright browser canary scripts
├── observability/        # Prometheus configs, alerting rules, and Grafana dashboard JSONs
├── scripts/              # Synthetic video generator (FFmpeg) and DB seeder
├── experiments/          # run_experiment.py orchestrator and report templates
├── reports/              # Generated markdown before/after experiment artifacts
├── docker-compose.yml    # Complete local stack definition
└── CONTEXT.md            # Living system context and glossary
```
