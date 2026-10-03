# 09: HTML5 Web Video Player & Telemetry Beacon Client

**What to build:** A lightweight HTML5 web player interface powered by `hls.js`, served statically by Nginx at `/player/`, instrumented with lifecycle event hooks that calculate Time-To-First-Frame (TTFF) and buffer stall metrics and transmit them as background beacons to the telemetry endpoint.

**Blocked by:** 02: Nginx Reverse Proxy & Static HLS Streaming Gateway, 05: Live Clock Sidecar & Dynamic Manifest with Flaw Toggle (Cache Stampede), 08: Backend Telemetry Ingestion API & Multiprocess Prometheus Metrics

**Status:** ready-for-agent

- [ ] HTML5 video player UI served statically by Nginx at `http://localhost:8080/player/index.html`.
- [ ] Integrates `hls.js` client library supporting adaptive bitrate streaming and live manifest polling.
- [ ] Instruments player lifecycle hooks:
  - Measures Time-To-First-Frame (TTFF) from stream attachment to the first rendered frame (`HTMLVideoElement.playing`).
  - Measures buffer stall count and duration by capturing `waiting` and `playing` events.
  - Captures `hls.js` error events (`hlsError`) and quality level changes (`LEVEL_SWITCHED`).
- [ ] Transmits background telemetry beacons via `navigator.sendBeacon` or asynchronous `fetch` to `/api/telemetry/playback`.
- [ ] Transmits client viewer heartbeats every 5 seconds to `POST /api/live/heartbeat`.
- [ ] Verification test verifies that playing both VOD (`/media/vod/vod-0/master.m3u8`) and Live (`/live/live.m3u8`) streams produces telemetry beacons and heartbeats visible in browser network logs.
