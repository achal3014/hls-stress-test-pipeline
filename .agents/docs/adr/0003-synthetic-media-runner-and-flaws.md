# ADR-0003: Synthetic Media Generation, Automated Experiment Runner, and 6-Flaw Target Matrix

## Status
Accepted

## Context
Following decisions on media offloading and telemetry architecture, three operational areas required resolution:
1. **Video Asset Strategy:** Storing high-bitrate video clips in Git causes repository bloat, while downloading external assets introduces network dependencies and flakiness during CI/local runs.
2. **Experiment Delivery & Repeatability:** The project's stated mission is an evidence-generating SRE pipeline. Operating containers manually with hand-edited `.env` files and multiple terminal windows is error-prone and does not provide an interviewer-ready or automated evidence trail.
3. **Target Flaw Matrix:** The platform implements a complete 6-flaw matrix covering every major layer of the system (Gateway, Cache, Database, State, Async Runtime, and Resilience).

## Decisions

### 1. Media Preparation Strategy — Hybrid Real + Procedural (Zero Git Bloat)

**Context:** Pure procedural FFmpeg (e.g. `testsrc2`, `smptebars`) produces test patterns that are immediately recognizable as synthetic, undermining the platform's demo realism. Real video was considered too large to commit to Git. A hybrid approach resolves both concerns.

* **Decision:** Two distinct asset classes, each prepared by `scripts/prepare_media.py` at `docker compose up` time. All media paths (`media/`, `*.ts`, `*.m3u8`, `*.mp4`) are listed in `.gitignore` — zero binary bytes are ever committed to the repository.

* **VOD Assets — 5 Real Blender Open Movies:**
  * **Titles:** *Big Buck Bunny*, *Sintel*, *Tears of Steel*, *Elephants Dream*, *Sprite Fright*.
  * **License:** All are Creative Commons licensed and safe for public GitHub repositories.
  * **Acquisition:** `prepare_media.py` checks for a locally cached source MP4; if absent, downloads it from the Blender Foundation's public CDN (one-time internet dependency, ~2–3 minutes on first run, skipped on subsequent runs).
  * **Segmentation:** FFmpeg transcodes each film into dual-rendition HLS (360p @ 400 kbps, 720p @ 1.2 Mbps, 2-second `.ts` segments, static `master.m3u8` / `360p.m3u8` / `720p.m3u8` playlists) written to `/media/vod/vod-{0..4}/`.
  * **Catalog distribution:** The 10,000 PostgreSQL catalog entries are distributed evenly across these 5 titles (~2,000 entries per film). Clicking different catalog titles plays visually distinct, recognizable film content.
  * **Git impact:** 0 bytes — source MP4s and all HLS output are `.gitignore`d.

* **Live Asset — 1 Procedural FFmpeg Video (~8 min):**
  * **Source:** FFmpeg `testsrc` filter (v1, not v2) — no download required, fully offline.
  * **Rationale for `testsrc` (not a Blender film):** The live sliding-window manifest requires that segment `N` is byte-identical across every generator invocation. `testsrc` produces a mathematically fixed, deterministic waveform. Real film content, once re-encoded, may produce subtly different byte sequences across machines due to encoder state — introducing manifest coherency risk when the sidecar cycles back to segment 0 after 8 minutes.
  * **Output:** Written exclusively to `/media/live/`. Never referenced by any VOD catalog entry, preserving complete Nginx cache isolation between VOD and live workloads during `FLAW_CACHE_STAMPEDE` experiments.
  * **Git impact:** 0 bytes — same `.gitignore` rules apply.

* **Shared setup behaviour:** `prepare_media.py` is idempotent — it checks for existing output directories before downloading or generating. Re-running `docker compose up` after the first setup is fast (no redundant downloads or re-encodes).

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
