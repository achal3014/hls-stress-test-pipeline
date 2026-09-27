# 07: Gateway Saturation & Token-Bucket Backpressure Flaws

**What to build:** Gateway worker connection constraint toggles (`FLAW_WORKER_SATURATION`) and API admission control / backpressure middleware (`FLAW_NO_BACKPRESSURE`) demonstrating gateway socket drops (HTTP 502/504) and catastrophic latency collapse under overload versus clean, resilient load shedding (HTTP 429/503).

**Blocked by:** 04: Playback Session API with Flaw Toggle (Async Event Loop Blocking), 05: Live Clock Sidecar & Dynamic Manifest with Flaw Toggle (Cache Stampede)

**Status:** ready-for-agent

- [ ] When `FLAW_WORKER_SATURATION=true`, Nginx `worker_connections` and Uvicorn worker backlog limits are clamped to low thresholds (e.g. 16 connections), causing sudden traffic surges to overflow TCP queues and return `HTTP 502 Bad Gateway` and `HTTP 504 Gateway Timeout`.
- [ ] When `FLAW_WORKER_SATURATION=false`, worker pools and socket backlogs are configured for standard high-concurrency throughput.
- [ ] When `FLAW_NO_BACKPRESSURE=true`, the API attempts to process 100% of incoming requests without rate limiting or concurrency caps, resulting in unbounded latency degradation under heavy load.
- [ ] When `FLAW_NO_BACKPRESSURE=false`, a token-bucket rate limiter / admission control middleware intercepts excess traffic beyond capacity, shedding surplus requests with `HTTP 429 Too Many Requests` or `HTTP 503 (Retry-After)`, while preserving sub-50ms latency for accepted sessions.
- [ ] Verification tests validate the presence of 502/504 gateway responses under worker saturation and structured 429/503 load shedding under backpressure mode.
