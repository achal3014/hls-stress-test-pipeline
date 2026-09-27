# ADR-0003: Synthetic Media Generation, Automated Experiment Runner, and 6-Flaw Target Matrix

## Status
Accepted

## Context
Following decisions on media offloading and telemetry architecture, three operational areas required resolution:
1. **Video Asset Strategy:** Storing high-bitrate video clips in Git causes repository bloat, while downloading external assets introduces network dependencies and flakiness during CI/local runs.
2. **Experiment Delivery & Repeatability:** The project's stated mission is an evidence-generating SRE pipeline. Operating containers manually with hand-edited `.env` files and multiple terminal windows is error-prone and does not provide an interviewer-ready or automated evidence trail.
3. **Target Flaw Matrix:** The platform implements a complete 6-flaw matrix covering every major layer of the system (Gateway, Cache, Database, State, Async Runtime, and Resilience).

## Decisions

### 1. Synthetic FFmpeg Video Generation (Zero Downloads)
* **Decision:** All test media (VOD clips and live segment pools) will be generated synthetically on demand using FFmpeg filter sources (`testsrc2`, `sine`).
* **Attributes:**
  * Displays an on-screen running millisecond timecode clock and generates an audio tone.
  * Generated in multiple quality renditions (e.g., 360p @ 400kbps, 720p @ 1.2Mbps) with 2-second HLS segment chunks.
  * 100% reproducible offline, completes in <15 seconds, and commits 0 MB of media files to Git.

### 2. The Single-Command Automated Experiment Runner (`run_experiment.py`)
* **Decision:** The automated single-command runner is defined as a **required core deliverable**, not optional polish.
* **Architecture:**
  * CLI signature: `python run_experiment.py --scenario <name> --flaw <name> [--baseline]`
  * **Responsibilities:**
    1. Reconfigures stack flags/environment variables dynamically.
    2. Cycles and recreates affected containers, polling `/health` before launching load.
    3. Orchestrates k6 synthetic load and Playwright browser canary (launched upon k6 plateau signal).
    4. Queries Prometheus API for the exact test window to extract metric deltas, while pulling authoritative percentiles directly from k6 JSON output.
    5. Outputs structured Markdown before/after summary tables in `reports/` and prints a CLI summary.
* **Build Order Rule:** To prevent compounding bugs, each component (app flags, k6 scripts, Playwright probes, Prometheus, Grafana) is built and manually validated in isolation *first*. The automated runner is then wrapped around the confirmed-working pieces.

### 3. Comprehensive 6-Flaw Target Matrix
The platform implements 6 distinct, toggleable bottlenecks mapping cleanly across each architectural layer:
1. **`FLAW_CACHE_STAMPEDE` (`FLAW_CACHE_TTL_SECONDS=0` vs `2`):**
   * *Layer:* Cache (Nginx/Redis).
   * *Signature:* Live manifest cache bypass triggers thundering-herd backend request amplification and elevated latency under concurrent viewers.
2. **`FLAW_DB_SLOW_QUERY` (`FLAW_SEARCH_INDEX=false` + constrained pool):**
   * *Layer:* Database (PostgreSQL).
   * *Signature:* Missing search index on catalog queries with 10k rows forces full table scans; connection pool queues up and p95 latency spikes >2,000ms.
3. **`FLAW_COUNTER_RACE` (`FLAW_VIEWER_COUNTER_RACE=true` vs `false`):**
   * *Layer:* State / Concurrency Correctness (Redis).
   * *Signature:* Unsynchronized read-modify-write lost updates cause reported live viewers in Grafana to diverge visually below actual connected k6 clients. Fixed mode uses Redis Sorted Sets (`ZSET`).
4. **`FLAW_EVENT_LOOP_BLOCKING` (`FLAW_SYNC_BLOCKING=true`):**
   * *Layer:* Async Runtime (FastAPI / Python event loop).
   * *Signature:* Synchronous CPU-bound hashing inside an async endpoint blocks the event loop thread, causing latency degradation across completely unrelated concurrent API routes.
5. **`FLAW_WORKER_SATURATION` (`FLAW_WORKER_CONSTRAINED=true`):**
   * *Layer:* Gateway / Edge (Nginx / Uvicorn).
   * *Signature:* Artificially restricted worker connection and process limits cause socket backlogs to overflow during traffic spikes, producing `HTTP 502 Bad Gateway` and `HTTP 504 Gateway Timeout` errors.
6. **`FLAW_NO_BACKPRESSURE` (`FLAW_NO_BACKPRESSURE=true`):**
   * *Layer:* System Resilience & Admission Control.
   * *Signature:* Absence of load shedding causes catastrophic latency collapse under overload. Fixed mode activates token-bucket rate limiting/admission control that sheds excess traffic cleanly with `HTTP 429 Too Many Requests` or `HTTP 503 (Retry-After)`, preserving sub-50ms latency for admitted sessions.
