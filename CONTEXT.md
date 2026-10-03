# Project Context: Video Streaming Load & Stress Testing Platform

## 1. System Vision & Purpose
A demonstration-grade, evidence-first SRE & performance engineering platform.
The core deliverable is **not** just a streaming site, but a complete **break-observe-diagnose-fix-prove** lifecycle demonstrated through reproducible load tests, multi-layer observability, and documented architectural trade-offs.

---

## 2. Core Architecture

### Data Plane vs. Control Plane Separation
* **Edge / Static Media Plane (Nginx):**
  * Sits in front of the application.
  * Serves HLS `.ts` video segments directly from `media_data` Docker named volume.
  * Handles static caching, headers, and media file transport without touching Python/App worker runtimes.
* **Application / Control Plane (Backend API — FastAPI, multi-worker Uvicorn):**
  * Manages catalog metadata, search, playback session initialization, dynamic live HLS manifest (`.m3u8`) rendering, and viewer heartbeats.
  * Stateless with respect to live sequence — reads sequence pointer from Redis only.
* **Live Clock Sidecar (`services/live-clock/`):**
  * Single isolated Python process. The only writer of the live manifest sequence pointer in Redis.
  * Advances `#EXT-X-MEDIA-SEQUENCE` every `TARGET_SEGMENT_DURATION` seconds.
* **Storage & State Plane:**
  * **PostgreSQL:** Video metadata, catalog (10,000 seed records distributed evenly across 4–5 distinct VOD assets), historical playback session records.
  * **Redis:** Live sequence pointer (written only by live-clock sidecar), active viewer sessions/heartbeats, manifest micro-cache.
* **Synthetic Client & Telemetry Plane (run on host, outside Docker):**
  * **k6:** Raw HTTP load generation simulating concurrent viewers (manifest polling, segment downloads, heartbeats). Runs on host to avoid CPU contention with stack under test.
  * **Playwright:** Canary UX probe measuring real browser playback metrics (TTFF, stalls, rebuffering frequency). Runs on host, launched only after k6 signals VU plateau.
  * **Observability:** Prometheus + Grafana + `cAdvisor` / `node_exporter` (inside Docker, within resource caps).

---

## 3. Architectural Rules & Invariants
1. **Never serve raw video binary segments from application runtimes.** Static media streaming is offloaded to Nginx from a named Docker volume.
2. **Permanent Flaw Toggles:** Bottlenecks and concurrency bugs are never temporary code hacks. Each flaw is controlled by a permanent `FLAW_*` environment variable so any experiment can be reproduced on demand without git-reverting.
3. **Dual Client Telemetry:** Server-side metrics (latency, error rates, pool saturation) must always be correlated against client-side UX metrics (rebuffer ratio, startup latency) and concurrency correctness metrics (counter drift).
4. **Live clock is always a single isolated process.** No API worker may ever write the live sequence pointer.
5. **k6 and Playwright always run on the host.** They must never run inside Docker alongside the stack under test during the same experiment.
6. **All Prometheus metrics in experiment reports are deltas.** `run_experiment.py` snapshots metrics at test start and reports `end − start` values. Absolute cumulative values are never used in generated reports.
7. **Container restart before every experiment.** `run_experiment.py` always stops and restarts affected services when changing flaw flags, then polls the health endpoint before starting load.

---

## 4. Domain Glossary
* **HLS (HTTP Live Streaming):** Video delivered as a sequence of small HTTP-based file chunks (`.ts`) indexed by a playlist manifest (`.m3u8`).
* **VOD (Video On Demand):** Static playlist where all segments exist prior to playback and viewers can seek anywhere.
* **Simulated Live:** A sliding-window HLS playlist where the live-clock sidecar advances `#EXT-X-MEDIA-SEQUENCE` and segment pointers to mimic broadcast television without real-time transcoding.
* **Thundering Herd:** Hundreds of concurrent clients requesting the exact same live segment within the same narrow time window.
* **Canary Probe:** A single isolated real browser (Playwright) running alongside synthetic load (k6) to measure true human UX degradation.
* **Viewer Heartbeat:** Periodic lightweight HTTP ping (`POST /api/live/heartbeat`) from clients every 5 seconds to indicate an active viewing session in stateless HTTP streaming.
* **Viewer Drift:** The discrepancy between actual connected virtual users and the server's reported live viewer count caused by race conditions and lost updates.
* **VU Plateau Signal:** A JSON file (`results/k6_plateau.json`) written by k6 when it reaches its target virtual user count. Polled by `run_experiment.py` to time Playwright launch.
* **Metric Delta:** The difference between a Prometheus counter's value at test end and its value at test start, used to isolate one experiment's contribution from cumulative historical totals.
* **Live-Clock Sidecar:** The single-process container (`services/live-clock/`) that is the sole writer of the live manifest sequence state in Redis.

---

## 5. Tech Stack Choices
* **Language & Framework:** Python 3.11+ / FastAPI (multi-worker Uvicorn with `PROMETHEUS_MULTIPROC_DIR`).
* **Media Proxy:** Nginx (serving static `.ts` chunks from `media_data` volume, micro-caching live manifests).
* **Media Preparation:** `scripts/prepare_media.py` populates the `media_data` Docker volume on first run. Zero media files committed to Git (`media/`, `*.ts`, `*.m3u8`, `*.mp4` are `.gitignore`d):
  * **VOD (5 real Blender open movies):** *Big Buck Bunny*, *Sintel*, *Tears of Steel*, *Elephants Dream*, *Sprite Fright* — CC-licensed, downloaded once from Blender Foundation's public CDN (~2–3 min on first `docker compose up`), cached locally thereafter. FFmpeg segments each into dual-rendition HLS (360p/720p, 2-second `.ts` chunks, static playlists). The 10,000 catalog entries are distributed evenly across these 5 titles (~2,000 per film).
  * **Live (1 procedural FFmpeg video, ~8 min):** Generated from `testsrc` filter — no download, fully offline, deterministic byte output. Stored exclusively at `/media/live/`, never referenced by VOD catalog entries, preserving clean cache isolation for thundering-herd and `FLAW_CACHE_STAMPEDE` experiments.
* **State & Metadata:** PostgreSQL (catalog, sessions) & Redis (live sequence, ZSET heartbeat window).
* **Load Generator:** k6 (runs on host, `--out json=results/run.json` for authoritative percentiles).
* **Canary Browser:** Playwright + `hls.js` in-player telemetry beacon to `/api/telemetry/playback`.
* **Observability:** Prometheus + Grafana + `cAdvisor` / `node_exporter`.
* **Experiment Runner:** `run_experiment.py` (CLI, orchestrates all of the above, generates `reports/` Markdown).

---

## 6. Host Port Allocations
* **Nginx:** `8080:80`
* **FastAPI:** `8000:8000`
* **PostgreSQL:** `5433:5432` *(host 5432 occupied by local Postgres process PID 7340)*
* **Redis:** `6379:6379`
* **Prometheus:** `9090:9090`
* **Grafana:** `3000:3000`

---

## 7. Docker Resource Caps (Server-Side Containers)
Ensures host CPU headroom is available for k6 and Playwright:
* `api`: `cpus: 1.0`, `mem_limit: 512m`
* `nginx`: `cpus: 0.5`, `mem_limit: 256m`
* `postgres`: `cpus: 1.0`, `mem_limit: 512m`
* `redis`: `cpus: 0.5`, `mem_limit: 256m`
* `live-clock`: `cpus: 0.25`, `mem_limit: 64m`

---

## 8. Decision Log Summary
* **ADR-0001:** Nginx Reverse Proxy for Segment Data Plane & Runtime Flaw Toggles.
* **ADR-0002:** Dynamic Live Manifest Generation, In-Player Telemetry Ingestion, and ZSET Viewer Counter Mechanics.
* **ADR-0003:** Synthetic Media Generation, Automated Experiment Runner (`run_experiment.py`), and Core 6-Flaw Target Matrix.
* **ADR-0004:** Repository Directory Layout, Catalog Seeding (10k items), and Host Port Allocations (Postgres → host `5433`).
* **ADR-0005:** Red-Team Resolutions — Container restart lifecycle, counter contradiction fix, Prometheus timing, live-clock isolation, CPU caps, multiprocess metrics, media volume, Playwright timing, seeder portability, metric deltas, 8-minute synthetic video, k6 plateau signal.

---

## 9. Core 6-Flaw Matrix
1. **`FLAW_CACHE_STAMPEDE`** (Cache Layer): Live manifest TTL = 0. Induces thundering-herd request amplification against FastAPI.
2. **`FLAW_DB_SLOW_QUERY`** (Database Layer): Missing catalog search index + pool = 5. Full sequential scan on 10k rows causes connection pool exhaustion and query latency spikes (>2,000ms).
3. **`FLAW_COUNTER_RACE`** (State / Concurrency): Naive read-modify-write in memory vs Redis Sorted Set (`ZSET`). Causes visual counter drift away from actual active virtual users.
4. **`FLAW_EVENT_LOOP_BLOCKING`** (Async Runtime): Synchronous CPU hashing on the async thread. Blocks event loop, degrading latency on unrelated concurrent API routes.
5. **`FLAW_WORKER_SATURATION`** (Gateway / Edge): Constrained worker connection/process limits in Nginx/Uvicorn. Causes TCP socket overflows, producing `HTTP 502/504` errors under traffic spikes.
6. **`FLAW_NO_BACKPRESSURE`** (System Resilience): Uncontrolled request acceptance under overload vs. token-bucket load shedding (`HTTP 429/503`), demonstrating graceful degradation under flash crowds.
