# Specification: Video Streaming Load & Stress Testing Platform

## Problem Statement

When high-concurrency consumer platforms experience traffic spikes—particularly in live video streaming where thousands of viewers demand the exact same live media chunk at the exact same second—systems fail in complex, cascading ways across multiple architectural tiers:
1. **Edge / Gateway:** Worker connection pools overflow, dropping TCP sockets and producing HTTP 502/504 gateway errors.
2. **Cache:** Live manifest cache bypasses trigger thundering-herd request stampedes against the backend application.
3. **Database:** Un-indexed queries combined with constrained connection pools cause queries to queue up and exhaust database resources.
4. **Concurrency / State:** Unsynchronized read-modify-write state updates under concurrent viewer heartbeats produce lost updates and severe counter drift.
5. **Runtime / Async:** Synchronous CPU-bound work executed on single-threaded event loops starves concurrent asynchronous I/O across unrelated endpoints.
6. **System Resilience:** In the absence of admission control or backpressure, overloaded servers attempt to process all incoming traffic, cascading into total service collapse rather than shedding load gracefully.

Engineering teams and SREs often struggle to diagnose these failure modes before major live events because typical load testing simply "hits an endpoint hard" without:
- Simulating synchronized, real-world user behaviors (HLS manifest polling, segment streaming, client heartbeats).
- Correlating server-side infrastructure metrics with actual end-user playback degradation (stalls, rebuffering, startup latency).
- Demonstrating concurrency correctness bugs alongside latency degradation and gateway dropouts.
- Providing a fully reproducible, automated evidence trail that proves a bottleneck was identified, isolated, diagnosed, and resolved.

Developers and interviewers evaluating performance engineering skill need a self-contained, reproducible platform that demonstrates the complete **build → break → observe → diagnose → fix → prove** lifecycle across all 6 tiers on standard developer hardware without cloud costs or manual multi-terminal orchestration.

---

## Solution

A fully instrumented, locally deployable video streaming platform with deliberate, toggleable architectural bottlenecks across all 6 architectural tiers, paired with an automated, single-command load testing and observability pipeline.

The solution consists of:
1. **A Realistically Architected Streaming Platform:**
   - **Static Media Plane:** A high-performance reverse proxy (Nginx) that serves HLS `.ts` video chunks directly from a dedicated `media_data` Docker volume, decoupling raw binary transport from application logic.
   - **Application Control Plane:** A high-throughput API service (FastAPI, multi-worker Uvicorn) handling catalog browsing, search, playback session initialization, dynamic live manifest (`.m3u8`) generation, client heartbeat ingestion, and real-user playback telemetry ingestion.
   - **Live Clock Sidecar:** A dedicated, single-process service that monotonically advances the live broadcast sequence pointer in Redis, preventing worker split-brain and manifest drift.
   - **State & Storage Layer:** A relational database (PostgreSQL) seeded with 10,000 video records for catalog queries, and an in-memory data store (Redis) managing the live sliding window, manifest caching, and viewer sessions.
2. **Synthetic Media Generation Engine:**
   - An on-demand generator that produces 8 minutes of multi-rendition HLS video streams with embedded millisecond timecode clocks and audio tones using procedural video synthesis (`testsrc2`, `sine`)—committing 0 MB of media files to version control while ensuring 100% offline reproducibility.
3. **Pluggable 6-Tier Runtime Flaw Matrix:**
   - Permanent, environment-controlled flaw toggles representing six distinct failure categories:
     - *Cache Stampede / Thundering Herd (`FLAW_CACHE_STAMPEDE`):* Live manifest caching disabled/bypassed.
     - *Database Slow Query (`FLAW_DB_SLOW_QUERY`):* Un-indexed catalog search paired with constrained connection pools.
     - *Concurrency Correctness Race (`FLAW_COUNTER_RACE`):* Unsynchronized read-modify-write lost updates vs. atomic sliding-window Redis Sorted Sets.
     - *Event Loop Starvation (`FLAW_EVENT_LOOP_BLOCKING`):* Synchronous CPU-bound hashing blocking the asynchronous application runtime.
     - *Worker & Gateway Saturation (`FLAW_WORKER_SATURATION`):* Artificially constrained worker connections and backlogs producing TCP overflows and HTTP 502/504 errors.
     - *Absence of Backpressure / Load Shedding (`FLAW_NO_BACKPRESSURE`):* Uncontrolled request acceptance under overload vs. token-bucket load shedding (`HTTP 429 / 503 (Retry-After)`).
4. **End-to-End Automated Experiment Pipeline:**
   - A single-command orchestrator (`run_experiment.py`) that reconfigures the platform, restarts containers, verifies health, launches synthetic HTTP traffic and a headless browser canary probe (triggered upon k6 plateau signal), extracts Prometheus metric deltas and authoritative percentiles, and outputs a formatted before/after comparative Markdown report.
5. **Multi-Layer Observability:**
   - Prometheus metrics capturing Golden Signals (latency percentiles, error rates, throughput), resource saturation (CPU, memory, connection pools), concurrency correctness (reported viewers vs. actual active clients), and real browser playback UX (Time-To-First-Frame, rebuffering stall counts, rebuffer duration).

---

## User Stories

### Platform Operator & Experiment Runner
1. As an SRE, I want to execute a single command specifying an experiment scenario and flaw toggle, so that the entire benchmark suite runs autonomously without manual intervention.
2. As an SRE, I want the experiment runner to apply configuration changes by restarting affected containers and verifying health before testing begins, so that test runs never execute against stale configurations.
3. As an SRE, I want the experiment runner to delay the launch of the browser canary until the synthetic load generator signals it has reached its target virtual user plateau, so that the canary measures peak stress conditions rather than ramp-up behavior.
4. As an SRE, I want the experiment runner to wait for Prometheus scrape windows to flush after load generation stops, so that tail-end metrics are not lost.
5. As an SRE, I want the experiment runner to compute metric deltas rather than cumulative totals, so that back-to-back experiment runs do not contaminate each other's results.
6. As an SRE, I want the experiment runner to pull authoritative latency percentiles directly from load generator outputs rather than histogram approximations, so that reported p95 and p99 values are mathematically accurate.
7. As an SRE, I want the experiment runner to generate a standardized before/after Markdown comparison table in a reports directory, so that evidence of performance fixes is permanently documented.
8. As a developer, I want to generate synthetic multi-rendition HLS video streams on demand with embedded timecodes, so that the platform functions offline without downloading large external media files.

### Viewer & Synthetic Client Flows
9. As a viewer, I want to search and browse a catalog of video titles, so that I can discover content to watch.
10. As a viewer, I want to initialize a playback session for a selected video, so that playback metadata and stream URLs are established.
11. As a viewer, I want to fetch VOD HLS manifests and video segments smoothly, so that on-demand video streams continuously without interruption.
12. As a viewer, I want to fetch dynamic live HLS manifests that update every segment duration, so that I receive new media chunks as the broadcast progresses.
13. As a viewer, I want my client to transmit periodic heartbeats every 5 seconds, so that my active presence is counted in the live viewer total.
14. As a viewer, I want my client to report playback performance telemetry (startup latency, rebuffering events) to the server, so that real-user experience is monitored.
15. As a synthetic load generator, I want to simulate hundreds of concurrent viewers joining a live broadcast at the exact same second, so that the platform's handling of the thundering-herd effect is stress-tested.
16. As a synthetic load generator, I want to simulate catalog search queries across varied keyword distributions, so that database query performance is stressed under concurrency.

### Bottleneck & Observability Scenarios (The 6 Flaws)
17. As an SRE, I want to toggle a cache stampede flaw on the live manifest endpoint (`FLAW_CACHE_STAMPEDE`), so that I can observe backend request amplification and latency spikes when caching is eliminated.
18. As an SRE, I want to toggle a missing database index flaw on catalog search with a constrained connection pool (`FLAW_DB_SLOW_QUERY`), so that I can observe query queuing and pool exhaustion under load.
19. As an SRE, I want to toggle an unsynchronized read-modify-write flaw on the live viewer counter (`FLAW_COUNTER_RACE`), so that I can visually observe the counter drift away from actual active virtual users in Grafana.
20. As an SRE, I want to toggle an atomic Redis Sorted Set viewer counter, so that I can prove the concurrency race condition is fixed and viewer counts match active virtual users exactly.
21. As an SRE, I want to toggle a synchronous CPU-blocking step on playback initialization (`FLAW_EVENT_LOOP_BLOCKING`), so that I can observe event loop lag and response degradation across unrelated concurrent requests.
22. As an SRE, I want to toggle worker connection saturation on the gateway (`FLAW_WORKER_SATURATION`), so that I can observe socket backlog overflows producing HTTP 502/504 gateway errors under traffic spikes.
23. As an SRE, I want to toggle uncontrolled overload acceptance (`FLAW_NO_BACKPRESSURE`), so that I can observe catastrophic failure under excess traffic versus clean load shedding with HTTP 429/503 keeping admitted sessions healthy.
24. As an SRE, I want to view a unified Grafana dashboard displaying Golden Signals, resource saturation, viewer count correctness, and canary rebuffer stalls on correlated timelines, so that I can diagnose root causes in a single pane of glass.
25. As an SRE, I want to see Grafana annotations marking when load stages ramp up, plateau, and cool down, so that cause-and-effect relationships are immediately obvious during visual inspection.

---

## Implementation Decisions

### Subsystem Decomposition & Boundaries

1. **Reverse Proxy & Media Data Plane:**
   - Responsible for terminating incoming client traffic for static media and acting as an edge reverse proxy.
   - Serves `.ts` video segments directly from a shared `media_data` volume without invoking application runtimes.
   - Provides configurable micro-caching for dynamic live playlists (`.m3u8`).
   - Routes API requests, live manifests, telemetry beacons, and player UI requests to appropriate backend services.

2. **Application Control Plane:**
   - High-throughput asynchronous service running multiple worker processes.
   - Exposes REST endpoints for catalog listing, catalog search, playback session creation, dynamic live manifest rendering, viewer heartbeat ingestion, and telemetry beacon ingestion.
   - Integrates multiprocess-aware metric aggregation writing to a shared memory-mapped directory (`PROMETHEUS_MULTIPROC_DIR`), ensuring metrics collected across worker processes are accurately exposed on the Prometheus scrape route.
   - Applies permanent flaw toggles based on environment configuration.

3. **Live Clock Sidecar:**
   - A single, dedicated process that is the sole authority responsible for advancing live broadcast state.
   - Operates a monotonic virtual clock loop advancing the live sequence pointer in Redis at fixed segment intervals.
   - Ensures zero split-brain or sequencing race conditions across application worker processes.

4. **Synthetic Video Generator:**
   - Procedural generation utility utilizing video synthesis filters (`testsrc2`, `sine`).
   - Creates 8 minutes of dual-rendition HLS assets (360p and 720p) with 2-second segment durations, complete with visual millisecond timecodes and audio test tones.
   - Outputs directly into the shared media volume before platform initialization.

5. **Client Telemetry & Canary Probe:**
   - Minimal browser player leveraging HTML5 video and an adaptive streaming client library (`hls.js`).
   - Hooks into player lifecycle events (`MANIFEST_PARSED`, `BUFFER_STALLED`, `LEVEL_SWITCHED`, video `waiting`, video `playing`) to compute startup latency (Time-To-First-Frame) and rebuffer stall count and duration.
   - Dispatches lightweight JSON telemetry beacons to the application backend.
   - A headless browser canary probe runs this player under load conditions, launched by the orchestrator after k6 reaches its VU plateau.

6. **Automated Experiment Runner:**
   - Command-line orchestration tool accepting scenario name, target flaw, and baseline flags.
   - Manages configuration injection, container lifecycle recreation, health verification polling, synchronized load execution, Prometheus metric delta calculation, and markdown report synthesis.

### Data Schemas & State Models

1. **Catalog Video Entity:**
   - Attributes: Unique identifier, title, slug, description, category, tags array, duration in seconds, HLS master manifest path, publication timestamp.
   - Seed scale: 10,000 synthetic records with varied text length and category distribution.
   - Indexing: Conditional B-tree / GIN index on title and description, activated or dropped via the database slow query flaw toggle.

2. **Playback Session Entity:**
   - Attributes: Session identifier, video identifier, client IP, user agent, session start timestamp, active status.
   - Created on playback start; records session metadata in the database.

3. **Live Channel Sequence State (Redis):**
   - Key: `live:channel:<channel_id>:sequence`
   - Value: Integer sequence index representing the current sliding-window position.
   - Read by application workers to dynamically construct valid live `.m3u8` playlists containing the current sequence and corresponding segment URLs.

4. **Live Viewer State (Redis):**
   - *Flawed State:* A single integer key (`live:channel:<channel_id>:viewers`) subjected to unsynchronized read-modify-write operations during client heartbeats.
   - *Fixed State:* A Redis Sorted Set (`live:channel:<channel_id>:active_viewers`) storing client identifiers with heartbeat arrival Unix timestamps as scores. Pruning operations remove members with scores older than 10 seconds, and cardinal queries provide atomic, drift-free viewer counts.

5. **Client Telemetry Beacon Payload:**
   - Attributes: Session identifier, stream identifier, event type (startup, stall_start, stall_end), Time-To-First-Frame in milliseconds, rebuffer stall count, total rebuffer duration in milliseconds, current quality rendition, timestamp.

### Comprehensive 6-Flaw Toggle Specifications

1. **`FLAW_CACHE_STAMPEDE`:**
   - Enabled: Live manifest proxy cache TTL set to 0 seconds. Every viewer poll hits the application backend directly.
   - Disabled: Live manifest proxy cache TTL set to 1 second. Nginx absorbs identical requests within the segment window.

2. **`FLAW_DB_SLOW_QUERY`:**
   - Enabled: Catalog database search index is omitted, and database connection pool maximum is constrained to 5 connections. Concurrent search queries trigger full sequential table scans, causing connection queue starvation and p95 latency spikes (>2,000ms).
   - Disabled: GIN/B-tree index applied to search columns, and connection pool sized appropriately (e.g., 25 connections), reducing latency to <20ms.

3. **`FLAW_COUNTER_RACE`:**
   - Enabled: Heartbeat endpoint executes naive read-modify-write against Redis in application memory, producing lost updates under concurrency.
   - Disabled: Heartbeat endpoint uses Redis Sorted Set atomic timestamp scoring and pruning, maintaining 0% viewer count drift.

4. **`FLAW_EVENT_LOOP_BLOCKING`:**
   - Enabled: Playback session creation executes an intensive synchronous hashing computation directly on the asynchronous event loop thread, blocking all concurrent I/O on that worker.
   - Disabled: Computation is offloaded to a thread pool executor or eliminated.

5. **`FLAW_WORKER_SATURATION`:**
   - Enabled: Worker connections in Nginx and/or worker backlog limits in Uvicorn are severely constrained. Under traffic spikes, connection queues overflow, resulting in TCP socket drops and HTTP 502/504 errors.
   - Disabled: Standard connection pools and worker configurations sized to handle the target traffic volume.

6. **`FLAW_NO_BACKPRESSURE`:**
   - Enabled: The API attempts to accept and process 100% of incoming requests without rate limiting or admission control, leading to latency cascading and connection drops under overload.
   - Disabled: Token-bucket admission control middleware actively sheds excess load, rejecting surplus requests with `HTTP 429 Too Many Requests` or `HTTP 503 (Retry-After)` and keeping admitted sessions under 50ms latency.

---

## Testing Decisions & Seams

### Testing Principles
- **External Behavior Verification:** Tests must interact with the system strictly across external network boundaries (HTTP requests, proxy responses, telemetry ingest, database queries). No test may inspect internal application private state directly.
- **Seam Minimization:** Testing is concentrated at the highest possible architectural seams to ensure realistic end-to-end coverage:
  1. *The Primary System Seam (HTTP Reverse Proxy Gateway):* All synthetic traffic (k6) and browser traffic (Playwright) enters through the reverse proxy on host port `8080`.
  2. *The Orchestration Seam (CLI Runner Interface):* The experiment runner (`run_experiment.py`) acts as the top-level test harness, exercising container management, load triggering, and metric gathering.
  3. *The Telemetry Seam (Prometheus Metrics API):* Observability assertions query the standardized Prometheus API on host port `9090` for metric delta evaluations.

### Verification Categories for the 6 Flaws
- **Component Health & Sanity Tests:** Fast automated checks confirming all containers are healthy, synthetic media files exist and are playable, database has 10,000 seed rows, and Redis keys are accessible.
- **Flaw Isolation Regression Tests:** Individual load tests validating that each of the 6 flaw toggles produces its expected signature when enabled, and behaves within normal thresholds when disabled:
  - *1. Manifest Cache Stampede:* Asserts backend request rate drops by >90% when cache is enabled.
  - *2. Catalog Search:* Asserts p95 query latency drops from >1,500ms to <20ms when index is enabled.
  - *3. Viewer Counter:* Asserts displayed viewer count matches active virtual users with 0% drift when ZSET mode is enabled, and displays measurable negative drift when race condition mode is enabled.
  - *4. Event Loop Blocking:* Asserts non-session API route latency remains flat during concurrent session starts when blocking flaw is disabled.
  - *5. Worker Saturation:* Asserts HTTP 502/504 errors drop to 0% when worker connection limits are restored.
  - *6. Backpressure / Load Shedding:* Asserts admitted session latency stays stable (<50ms) and excess traffic receives structured 429/503 responses rather than unhandled socket drops.
- **Canary UX Correlation Tests:** Automated Playwright runs verifying that real browser buffer stalls increase proportionally during simulated server-side saturation.

---

## Out of Scope

1. **Real-time Live Transcoding & Encoding Hardware:** Content is generated synthetically ahead of time; no live video capture cards or RTMP/WebRTC ingest pipelines are included.
2. **Multi-Region Content Delivery Networks (CDNs):** Edge distribution is simulated via a local caching reverse proxy; no real cloud CDN integration (Cloudflare, CloudFront) is configured.
3. **User Authentication, Billing, & DRM:** No user login, OAuth, payment processing, or Widevine/FairPlay DRM encryption is implemented.
4. **Mobile Applications:** No native iOS or Android video streaming clients are built.
5. **Production Kubernetes & Cloud Auto-scaling:** Deployment is strictly targeted at Docker Compose on developer workstations and single cloud virtual machines.

---

## Further Notes

- **Host Port Collision Handling:** The PostgreSQL host port is mapped to `5433` (container `5432`) to avoid collisions with pre-existing local database services. All inter-container communication remains on the internal Docker network.
- **Host Execution for Synthetic Clients:** Both k6 and Playwright execute directly on the host operating system rather than inside Docker containers, preventing CPU starvation and observer-effect distortion on the stack under test.
- **Transition to Implementation:** This specification serves as the direct input to `/to-tickets`, which will decompose these requirements into sequential, tracer-bullet implementation issues with defined blocking edges.
