# ADR-0001: Nginx Media Plane Offload and Runtime Flaw Toggles

## Status
Accepted

## Context
During initial architectural review of the Video Streaming Load & Stress Testing Platform, two critical design requirements were identified:
1. **Serving Binary Video Segments:** Serving raw HLS `.ts` video chunks (2–6 MB each) directly through Python/Uvicorn application runtimes creates severe user-space I/O buffer saturation and GIL contention on local developer machines. This would benchmark Python's file I/O limitations rather than the intended database, caching, and concurrency bottlenecks.
2. **Reproducibility of Planted Bottlenecks:** The platform features 6 deliberate performance and concurrency flaws. If fixes are applied as one-off code edits or git commits, re-testing an earlier bottleneck requires rolling back commits, destroying the ability to run automated comparison suites on demand.

## Decisions

### 1. Nginx Media Plane Architecture
* **Decision:** We place Nginx in front of the platform as a reverse proxy and static media server.
* **Responsibilities:**
  * Nginx serves `.ts` video segment files directly from local storage/cache.
  * Nginx proxies API traffic (`/api/*`), live dynamic manifests (`/live/*.m3u8`), and session endpoints to the backend application service.
* **Trade-off:** Adds an extra container to Docker Compose, but ensures realistic network behavior and preserves application runtime resources for business logic, metadata queries, and concurrency testing.

### 2. Permanent Flaw Toggles
* **Decision:** Every planted architectural flaw will be gated behind a permanent configuration toggle (environment variables or config flags), such as:
  * `FLAW_SLOW_SEARCH_INDEX=true|false`
  * `FLAW_CACHE_TTL_SECONDS=0|60`
  * `FLAW_VIEWER_COUNTER_RACE=true|false`
  * `FLAW_CONNECTION_POOL_LIMIT=5|50`
  * `FLAW_SYNCHRONOUS_HEAVY_INIT=true|false`
* **Consequences:** Tests can isolate a single flaw, combine specific flaws, or run a fully optimized baseline without any code changes or git branching.
