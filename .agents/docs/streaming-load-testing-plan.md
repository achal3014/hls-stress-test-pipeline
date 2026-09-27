# Video Streaming Platform — Load & Stress Testing Project
### Planning / Brainstorm Document (v1)

---

## 1. Problem Statement

Every large-scale consumer platform eventually faces the same question: *how much traffic can our system survive before it slows down or breaks — and what exactly breaks first?* This is especially acute for **live video streaming**, where the failure mode is unlike ordinary web traffic: thousands of viewers don't send independent, staggered requests — they all request the *same* live segment within the *same* couple of seconds (a match kickoff, a viral moment). This "thundering herd" pattern has visibly taken down real streaming platforms during major live events.

Answering "where will it break, and why" *before* it happens in production — instead of during the actual event, in front of real users — is the discipline of **load/stress testing combined with observability**. It is a real, named job function (SRE / performance engineering) at every company operating at scale, and it is meaningfully different from just "building an app that works."

This project exists to practice and demonstrate that discipline specifically, using video streaming as the domain because it produces failure modes (hot-key cache misses, concurrency-correctness bugs, resource exhaustion under synchronized load) that are more distinctive and more interview-worthy than a generic CRUD app would produce.

## 2. Pitched Solution

Build two things, in sequence:

1. **A realistically-architected HLS video streaming platform** (VOD + a simulated "live" channel), built with the same architectural components a real one would have (API layer, metadata DB, cache, segment-serving layer, a shared live-viewer-count) — deliberately engineered with a small number of known, realistic weak points.
2. **A reusable load-testing and observability pipeline** (k6 for traffic generation, a headless-browser probe for real playback UX, Prometheus + Grafana for multi-layer metrics) that simulates flash-crowd traffic against the platform, diagnoses *why* it breaks using live dashboards, and proves a fix works via a before/after comparison.

The deliverable isn't "a streaming site" — it's the evidence trail: build → break → observe → diagnose → fix → prove.

## 3. Goals and Non-Goals

**Goals**
- A working streaming platform with a handful of real user flows (browse catalog, play VOD, watch a simulated live channel)
- Multi-layer instrumentation: business-logic metrics from the app itself, infra metrics from exporters, real playback-UX metrics from a headless browser
- A repeatable load-testing pipeline covering multiple traffic patterns (not just "hit it hard once")
- At least one **correctness-under-concurrency** bug (not just a speed problem) — e.g. a live viewer counter that drifts from the true value under concurrent load
- A clean, documented before/after story for at least 2–3 of the planted bottlenecks

**Non-Goals (explicitly out of scope for v1)**
- No real adaptive multi-bitrate live transcoding — content is segmented **offline**, once, ahead of time
- No real CDN / multi-region edge network — a single-origin cache layer stands in for "CDN behavior"
- No production-grade auth, payments, or DRM — at most a stub user/session concept
- No mobile apps — a single web player is enough to drive and measure
- No internet-facing deployment or security hardening — this is a controlled local/lab environment
- No Kubernetes / autoscaling for v1 (candidate stretch goal only, see §14)

## 4. Architecture Overview

Four layers, talking to each other over one Docker Compose network:

- **Synthetic client layer** — k6 (raw HTTP load) and a headless browser probe (Playwright or Selenium — real playback UX)
- **Application layer** — API/backend service(s): catalog, playback-session, manifest/segment serving, live-viewer-count
- **Data layer** — Postgres (video/session metadata), Redis (segment/manifest cache, live viewer counters)
- **Observability layer** — app-level Prometheus instrumentation + infra exporters (`node_exporter`, `cAdvisor`) → Prometheus (storage) → Grafana (dashboards)

Everything is one `docker-compose up` away from running, so a test run is fully reproducible.

---

# PART 1 — The Application

## 5. Features Planned (MVP)

| Feature | Purpose |
|---|---|
| Content catalog (list + search) | Basic browsing; search is where a slow-query bottleneck lives |
| VOD playback | A user picks a video, plays it via HLS (manifest + segments) |
| Simulated live channel(s) | Pre-segmented content looped on a rolling window, so many viewers request the *same current segment* — this is what creates the thundering-herd scenario |
| Manifest endpoint (`.m3u8`) & segment endpoint (`.ts`) | The actual data-plane every viewer hits repeatedly |
| "Start playback" / session endpoint | Records a viewer session, looks up metadata — deliberately slow lookup lives here |
| Live concurrent-viewer counter | Updated by a heartbeat from every connected viewer, shown as a number — the shared, mutable state prone to a race condition |
| Minimal web player (HTML5 `<video>` + hls.js) | Just enough UI for a headless browser to drive and extract real telemetry |
| *(Stretch)* 2-rendition quality switch | Exercises adaptive bitrate logic; gives the browser probe something extra to observe |

## 6. Tech Stack (App)

- **Backend:** FastAPI (Python) — built-in OpenAPI spec, clean Prometheus client integration. *(Node/Express is an equally fine alternative if you're more comfortable there — the rest of the plan doesn't depend on the choice.)*
- **Database:** Postgres — video/session metadata
- **Cache / shared state:** Redis — segment & manifest cache, live viewer counters
- **Video packaging:** ffmpeg — offline HLS segmenting
- **Player:** HTML5 `<video>` + hls.js
- **Containerization:** Docker + Docker Compose

## 7. Data & Video Prep

- Use royalty-free / permissively-licensed test footage — e.g. the Blender Foundation open movies ("Big Buck Bunny," "Sintel," "Tears of Steel"), which are commonly used for exactly this kind of technical demo and are safe to include in a public GitHub repo. Avoid any commercial/copyrighted content, since this will likely be public-facing on a resume.
- Run ffmpeg **once, offline**, to segment 3–5 short clips into HLS (`.m3u8` + `.ts` files), ideally 2 quality renditions each if you build the stretch bitrate-switch feature.
- For the "live" channel: a small background script maintains a rolling manifest window — drops the oldest segment reference, appends the "next" one — cycling through the same pre-made segment pool. This mimics a real live manifest without needing actual live capture or encoding.

## 8. Planted Architectural Flaws (for later stress testing)

These are ordinary, realistic performance/concurrency oversights — the same category of thing real systems get wrong — not security vulnerabilities. Categories only for now; exact values/config get decided when you build each piece:

1. **Thin connection pool / worker count** on the segment-serving endpoint — the classic saturation point once concurrent viewers pass a threshold
2. **Missing index** on catalog search / metadata lookup — a slow query that degrades under concurrent load
3. **Short or absent cache TTL** on segment/manifest responses — enables a cache-miss storm under a viewer spike
4. **Unsynchronized read-modify-write** on the live viewer counter — a genuine race condition; under concurrency the displayed count drifts from the real value (this is your "correctness," not just "speed," story)
5. **A synchronous, unnecessarily heavy step** on the playback-start path (e.g. redundant serialization work) — a clean CPU-bound bottleneck, distinct in shape from the I/O-bound ones above
6. **No backpressure on segment requests** — under overload the server exhausts a resource and fails messily, rather than rejecting excess load gracefully

---

# PART 2 — The Load Testing & Observability Pipeline

## 9. Pipeline Tooling

| Purpose | Tool |
|---|---|
| Traffic generation | k6 |
| Real playback UX probe | Playwright (or Selenium — same role; Playwright tends to have cleaner async/network-interception support for video) |
| App metrics | Prometheus client library, instrumented directly in the FastAPI app |
| Infra metrics | `node_exporter` (host), `cAdvisor` (containers) |
| Metrics storage | Prometheus |
| Dashboards | Grafana |
| Orchestration | Docker Compose |

## 10. Load Testing Strategies

| Test type | Pattern | Question it answers |
|---|---|---|
| Smoke | 1–5 virtual users, 1–2 min | Does the pipeline itself work before trusting bigger runs? |
| Load (baseline) | Steady traffic at expected normal peak, ~10–15 min | What does "healthy" look like? |
| Spike | Traffic jumps abruptly, near-instantly | Does a sudden "match kickoff" moment cause a pile-up? |
| Ramping stress | Step increases (e.g. 100 → 200 → 300…) | Exactly where's the breaking point, and in what order do things fail? |
| Soak | Moderate load held for 30 min+ | Do things degrade slowly (memory/connection drift) that a short test would miss? |
| *(Stretch)* Chaos/failure | Kill/throttle Redis mid-test | Does the app degrade gracefully or fall over completely? |

## 11. Testing Scenarios / Experiments

1. **Baseline** — steady VOD browsing + playback at expected normal load
2. **Match kickoff spike** — thousands of viewers join the same live channel within seconds (thundering herd on manifest + segment fetch)
3. **Ramping viewers on one live channel** until the viewer-counter or segment server breaks — establishes the ceiling
4. **Long soak on a live channel** — watch for memory/connection drift and cache-eviction thrash over an extended run
5. **Cache-enabled vs. cache-disabled comparison** — directly demonstrates the value of the caching layer with a clean before/after
6. *(Stretch)* **Redis killed mid-test** — observe whether the app degrades gracefully or fails hard

## 12. Planned Metrics

**Client / synthetic (k6)**
- Request rate, latency percentiles (p50/p90/p95/p99) on manifest & segment fetches, error rate

**Real-user (headless browser probe)**
- Time-to-first-frame (startup time), rebuffering count & duration, bitrate switch events, playback stalls — this is the metric set no server-side tool can produce

**App-level (your own instrumentation)**
- Requests/sec per endpoint, DB pool usage, query duration (catalog search, session start), cache hit/miss ratio on segment & manifest requests, segment-serving latency
- **Viewer-count drift**: displayed count vs. actual connected clients — the direct, visual proof of the race condition (and later, of the fix)

**Resource-level (exporters)**
- CPU / memory per container, network throughput (segment bytes/sec — worth watching, it can get large fast), disk I/O

## 13. Observability & Dashboard Design

| Dashboard | Shows |
|---|---|
| Golden signals overview | Combined k6 + app latency/error/throughput — single-glance health |
| Playback experience | Browser-probe startup time & rebuffer events, correlated against server-side segment latency at the same timestamps |
| Resource & bottleneck drilldown | CPU/memory/pool/cache per component — where you point to say "this is the root cause" |
| Viewer-count correctness | Expected vs. actual counter value over the run — visually shows the race condition, then visually shows it fixed |

**One design habit worth setting up early:** add a Grafana annotation whenever k6 transitions between load stages. It makes every before/after and cause/effect comparison trivial to read later, instead of eyeballing timestamps.


## 14. Success Criteria

- Dashboards showing a clear before/after for at least 2–3 distinct bottleneck types (not just one)
- The viewer-count race condition demonstrated and fixed — this is the standout, most differentiated finding
- A short README/write-up: scenario, what broke, why, how it was fixed, proof it worked — this is the artifact that actually gets read by anyone reviewing the project
