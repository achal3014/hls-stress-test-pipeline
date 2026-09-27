# ADR-0005: Red-Team Architectural Resolutions

## Status
Accepted

## Context
A principal-engineer red-team review of ADRs 0001–0004 and CONTEXT.md identified nine architectural hazards across the runner orchestration, live manifest design, observability pipeline, and local resource management. All findings are resolved here before the project spec is written. Issues are grouped by original severity.

---

## Resolved Critical Issues

### C1: Container Restart Orchestration in `run_experiment.py`
**Problem:** Environment variables cannot be injected into already-running containers. The runner must fully cycle affected services when changing flaw toggles.

**Resolution:** `run_experiment.py` executes the following sequence for every experiment run:
1. Write target flaw flags to `.env`.
2. `docker compose up -d --force-recreate <affected-services>` to apply env changes.
3. Poll a designated health endpoint (e.g., `GET /health`) until the service returns `200` (max 30s timeout, 2s interval).
4. Only then launch k6 and the Playwright canary.

---

### C2: Live Viewer Counter — Contradiction Resolved, Logic Corrected
**Problem (1):** ADR-0002 (Redis read-modify-write) and ADR-0003 ("in-memory") contradicted each other on where the flawed counter's state lives.
**Problem (2):** Incrementing on every heartbeat with no decrement causes the counter to inflate infinitely rather than drift from the true value.

**Resolution:**
- **State location:** Standardized on **Redis** for both flawed and fixed modes. This makes the race condition testable, reproducible, and observable via Grafana.
- **Corrected counter model:**
  - **Flawed Mode (`FLAW_VIEWER_COUNTER_RACE=true`):** On heartbeat: Python reads counter value from Redis, adds 1 in application memory, writes back via `SET`. This is a classic read-modify-write race under concurrent load — multiple workers overwrite each other's increments. On viewer leave/timeout: no reliable decrement (clients may disconnect without notice), so a background cleanup task prunes stale sessions with naive integer subtraction, introducing further drift.
  - **Fixed Mode (`FLAW_VIEWER_COUNTER_RACE=false`):** Redis Sorted Set (`ZADD stream:viewers <timestamp> <user_id>`) per heartbeat. Expired entries pruned via `ZREMRANGEBYSCORE`. Count via atomic `ZCARD`. Accurate and TTL-self-healing.

---

### C3: Prometheus Post-Test Query Timing Hazard
**Problem:** Querying Prometheus immediately after k6 stops misses the final scrape window (pull model, ~15s interval). Also, Prometheus histogram bucket approximations produce imprecise percentiles.

**Resolution:**
- `run_experiment.py` sleeps for `scrape_interval + 5s` (i.e., 20s default) after k6 exits before querying Prometheus.
- p50/p95/p99 latency stats are sourced from **k6's local JSON output file** (`--out json=results/run.json`), not from Prometheus histograms.
- Prometheus is used for time-series overlays and visual dashboards only, not for authoritative percentile calculations.

---

## Resolved High Priority Issues

### H1: Live Manifest Clock — Single-Process Isolation
**Problem:** If FastAPI background tasks advance the live sequence clock, multiple Uvicorn workers race to update Redis, producing inconsistent or backwards-going `#EXT-X-MEDIA-SEQUENCE` numbers that break HLS clients.

**Resolution:** The live sequence clock is extracted into a **dedicated sidecar container** (`services/live-clock/`) — a minimal Python process running a single `while True` loop that advances the manifest pointer in Redis every `TARGET_SEGMENT_DURATION` seconds. FastAPI workers are stateless with respect to manifest generation: they only read the current sequence state from Redis and render the `.m3u8` text response.

---

### H2: Local CPU Starvation — Docker Resource Caps
**Problem:** Running k6, Playwright/Chromium, and the full Docker stack on one Windows machine causes CPU steal. Playwright will report degraded UX metrics caused by Chromium starvation, not backend behavior.

**Resolution:**
- Docker Compose applies explicit CPU and memory caps to server-side containers:
  - `api`: `cpus: '1.0'`, `mem_limit: '512m'`
  - `nginx`: `cpus: '0.5'`, `mem_limit: '256m'`
  - `postgres`: `cpus: '1.0'`, `mem_limit: '512m'`
  - `redis`: `cpus: '0.5'`, `mem_limit: '256m'`
  - `live-clock`: `cpus: '0.25'`, `mem_limit: '64m'`
- k6 and Playwright run **directly on the host** (outside Docker) to ensure uncontested CPU access.

---

### H3: Prometheus Multiprocess Metric Aggregation
**Problem:** Python `prometheus_client` does not aggregate metrics across Gunicorn/Uvicorn worker processes. Metrics sent to Worker A are invisible when Prometheus scrapes Worker B.

**Resolution:**
- Set `PROMETHEUS_MULTIPROC_DIR` environment variable in the `api` container, pointing to a `tmpfs` volume.
- The FastAPI service starts with `prometheus_client.multiprocess` mode enabled so all workers write to shared memory-mapped files, which the scrape endpoint aggregates atomically.

---

## Resolved Medium Priority Issues

### M1: Shared Media Volume Between FFmpeg Generator and Nginx
**Problem:** ADR-0004 defined no Docker volume connecting the FFmpeg-generated segment files with Nginx's static serving path.

**Resolution:**
- A named Docker volume `media_data` is defined in `docker-compose.yml`.
- Mounted to Nginx at `/usr/share/nginx/html/media`.
- The `scripts/generate_media.py` FFmpeg wrapper writes all `.ts` and `.m3u8` files into this volume during the `docker compose up` initialization sequence (via a `depends_on` condition or an init container pattern).
- Content consists of **8 minutes of synthetic FFmpeg `testsrc2` video** (on-screen millisecond timecode + audio tone) at two renditions (360p @ 400kbps, 720p @ 1.2Mbps), segmented into 2-second `.ts` chunks — approximately 240 segments per rendition.

---

### M2: Playwright Canary Launch Timing — k6 Plateau Signal
**Problem:** Launching Playwright at the same time as k6 causes the canary to see a healthy UX before the VU plateau is reached, completely missing the thundering-herd event.

**Resolution:** k6 writes a JSON plateau signal file (`results/k6_plateau.json`) when its configured VU target is reached using k6's `exec.vu.iterationInScenario` hooks or a threshold callback. `run_experiment.py` polls this file every 2 seconds and launches Playwright only after the signal appears. Playwright then runs in a continuous loop for the remainder of the k6 scenario duration.

---

## Resolved Low Priority Issues

### L1: Seeder Database Connection Portability
**Problem:** The seed script hardcoding `localhost:5433` breaks when the script runs inside a Docker container (where the port is `postgres:5432`).

**Resolution:** The seed script reads a `DATABASE_URL` environment variable. `docker-compose.yml` passes `DATABASE_URL=postgresql://user:pass@postgres:5432/streaming` for containerized seeding. The host-side `.env` provides `DATABASE_URL=postgresql://user:pass@localhost:5433/streaming` for local debugging.

---

## Open Questions Resolved (from Red-Team Report)

### OQ1: Prometheus Metric Contamination Across Test Runs
**Decision:** `run_experiment.py` automatically records a Prometheus metric **snapshot at test start** and computes the **delta** (end − start) for all counter-type metrics. The generated report presents only deltas, not absolute cumulative values, ensuring back-to-back experiments do not contaminate each other's statistics.

### OQ2: Synthetic Video Clip Length
**Decision:** 8 minutes of synthetic content (≈ 240 × 2-second segments per rendition). This ensures the live channel's sliding window never reaches a playlist re-initialization edge case during any of the defined test scenarios (longest: 30-minute soak).

### OQ3: k6 → Playwright Synchronization Mechanism
**Decision:** Option B — k6 writes a plateau signal file (`results/k6_plateau.json`) when the VU target is reached. `run_experiment.py` polls this file every 2 seconds before launching Playwright. This ensures the canary always enters the stream at peak load conditions, not during ramp-up.

---

## Architectural Invariants Added (Supplements ADR-0001 §3)
- **Live clock is always a single isolated process.** No API worker may write the live sequence pointer.
- **k6 runs on the host.** It must never run inside Docker on the same machine as the stack under test.
- **Playwright runs on the host.** It must never run inside Docker alongside k6 during the same experiment.
- **All Prometheus metrics are reported as deltas per experiment run.** Absolute cumulative values are never used in generated reports.
