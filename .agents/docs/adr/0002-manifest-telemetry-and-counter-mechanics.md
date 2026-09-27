# ADR-0002: Live Manifest Generation, Telemetry Ingestion, and Viewer State Mechanics

## Status
Accepted

## Context
Following decisions in ADR-0001, specific mechanics were needed for:
1. How simulated live channels advance their sliding playlist window.
2. How the Playwright canary browser exports player UX metrics (stalls, latency) to Prometheus.
3. How the live viewer counter operates in both flawed and production-grade modes.
4. The deployment and testing lifecycle.

## Decisions

### 1. Dynamic Live Manifest Endpoint (Option B)
* **Decision:** FastAPI dynamically computes and returns the live HLS `.m3u8` playlist on request from a synchronized timestamp/state stored in Redis.
* **Mechanism:** 
  * A background task or virtual clock advances a monotonic sequence index every target chunk duration (e.g., 2.0 seconds).
  * Nginx sits in front with a configurable micro-cache (e.g., 1.0s TTL).
* **Flaw Interaction:** When `FLAW_CACHE_TTL_ZERO=true`, Nginx micro-caching is bypassed, causing every concurrent k6 viewer to hammer FastAPI simultaneously for the live manifest, demonstrating a manifest cache stampede (thundering herd).

### 2. Real-User Telemetry Ingest Pipeline (Option A)
* **Decision:** Telemetry is gathered from the browser client via an in-player instrumentation hook (`hls.js` event listeners) posting to `/api/telemetry/playback`.
* **Mechanism:**
  * Metrics tracked: Time-To-First-Frame (TTFF), rebuffer stall count, total rebuffer duration, and bitrate rendition switches.
  * FastAPI ingests these beacons and increments Prometheus metrics (`streaming_player_rebuffer_stalls_total`, `streaming_player_ttff_seconds`, etc.).
  * Playwright acts as a headless canary viewer executing this code path under full synthetic load.

### 3. Viewer Counter Architecture (Flawed vs. Fixed)
* **Decision:** Viewers send a heartbeat (`POST /api/live/heartbeat`) every 5 seconds.
* **Flawed Mode (`FLAW_VIEWER_COUNTER_RACE=true`):** 
  * Python executes naive read-modify-write: `val = await redis.get("viewers"); await redis.set("viewers", val + 1)`.
  * Demonstrates lost updates under concurrency. Grafana displays a clear divergence between `k6_actual_vus` and `reported_viewers`.
* **Fixed Mode (`FLAW_VIEWER_COUNTER_RACE=false`):**
  * Implemented using a Redis Sorted Set (`ZADD stream:viewers <timestamp> <user_id>`).
  * Expired heartbeats are automatically pruned (`ZREMRANGEBYSCORE stream:viewers 0 <now - 10s>`).
  * Live viewer count is an atomic, accurate cardinal query (`ZCARD stream:viewers`).

### 4. Dual-Stage Testing Lifecycle
* **Stage 1 (Local Lab):** Complete development and automated testing in Docker Compose with a scaled load envelope (100–500 virtual users, 1 Playwright canary, 360p/720p renditions).
* **Stage 2 (Deployed Capstone):** Optional deployment to cloud VMs (e.g. AWS/DigitalOcean) separating the k6 traffic generator from the application server for high-volume public benchmarks and portfolio artifacts.
