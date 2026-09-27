# 08: Backend Telemetry Ingestion API & Multiprocess Prometheus Metrics

**What to build:** A high-throughput telemetry ingestion endpoint (`POST /api/telemetry/playback`) that collects real-user playback UX beacons and exposes aggregated Prometheus metrics across multi-worker Uvicorn processes using `PROMETHEUS_MULTIPROC_DIR`.

**Blocked by:** 02: Nginx Reverse Proxy & Static HLS Streaming Gateway, 04: Playback Session API with Flaw Toggle (Async Event Loop Blocking)

**Status:** ready-for-agent

- [ ] FastAPI exposes `POST /api/telemetry/playback` accepting client UX payloads: `{ session_id, stream_id, event_type, ttff_ms, stall_count, stall_duration_ms, rendition, timestamp }`.
- [ ] Container environment configures `PROMETHEUS_MULTIPROC_DIR` mapped to a shared `tmpfs` volume for inter-worker metric aggregation.
- [ ] Python `prometheus_client` multiprocess collector is configured to aggregate metrics across all worker processes.
- [ ] Exposes Prometheus metrics on `GET /metrics`:
  - `streaming_player_ttff_seconds` (Histogram/Gauge for Time-To-First-Frame)
  - `streaming_player_rebuffering_stalls_total` (Counter for playback stall events)
  - `streaming_player_rebuffering_duration_seconds_total` (Counter for total stalled playback time)
  - `streaming_player_rendition_switches_total` (Counter for ABR quality changes)
- [ ] Verification test verifies that sending telemetry payloads across multiple workers accurately increments aggregated values exposed at `http://localhost:8000/metrics`.
