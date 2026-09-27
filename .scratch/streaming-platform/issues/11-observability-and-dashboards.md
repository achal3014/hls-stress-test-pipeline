# 11: Multi-Layer Prometheus Metrics & Correlated Grafana Dashboards

**What to build:** Provisioned Prometheus scraping across all system tiers, container resource exporters (`cAdvisor`, `node_exporter`), and four comprehensive Grafana dashboards providing correlated timelines between server metrics, client playback UX, and viewer count correctness.

**Blocked by:** 08: Backend Telemetry Ingestion API & Multiprocess Prometheus Metrics, 10: Host-Based k6 Scenarios & Synchronized Playwright Canary Probe

**Status:** ready-for-agent

- [ ] Docker Compose configures `prometheus` (port `9090`) scraping Nginx, FastAPI (`/metrics`), PostgreSQL (`postgres_exporter`), Redis (`redis_exporter`), and host/container exporters.
- [ ] Docker Compose configures `grafana` (port `3000`) with automated provisioning of Prometheus datasource and dashboards.
- [ ] Dashboard 1: **Golden Signals Overview** — End-to-end request rates, p50/p90/p95/p99 latencies, error percentages, and throughput across API and media gateways.
- [ ] Dashboard 2: **Playback Experience & Canary Correlation** — Real browser TTFF and buffer stall frequency correlated against concurrent server-side segment latency.
- [ ] Dashboard 3: **Resource Saturation & Bottlenecks** — CPU and memory per container, database connection pool utilization, and Redis cache hit/miss ratio.
- [ ] Dashboard 4: **Viewer Count Correctness** — Time-series comparison plotting actual k6 Virtual Users against API reported live viewers, visually displaying counter drift in flawed mode and alignment in fixed mode.
- [ ] Automated Grafana annotations mark load test phase transitions (ramp-up, plateau, cool-down).
