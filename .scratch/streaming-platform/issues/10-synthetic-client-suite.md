# 10: Host-Based k6 Scenarios & Synchronized Playwright Canary Probe

**What to build:** Host-side load generation scripts using k6 for synthetic traffic patterns (Baseline VOD, Match Kickoff Spike, Ramping Viewers, Soak), emitting plateau signal files, paired with a headless Playwright canary probe that launches on the plateau signal to capture real browser UX degradation during peak load.

**Blocked by:** 06: Live Viewer Counter with Flaw Toggle (Lost Updates vs. Redis ZSET), 07: Gateway Saturation & Token-Bucket Backpressure Flaws, 09: HTML5 Web Video Player & Telemetry Beacon Client

**Status:** ready-for-agent

- [ ] Host-executable k6 scripts in `load-tests/` simulating realistic viewer workflows:
  - Catalog browsing and search (`search.js`)
  - VOD playback session initiation and segment pulling (`vod_playback.js`)
  - Live stream thundering herd with manifest polling, segment downloading, and heartbeats (`match_kickoff.js`)
- [ ] k6 writes a plateau signal file (`results/k6_plateau.json`) when the configured Virtual User target is reached.
- [ ] k6 outputs raw run statistics to JSON (`--out json=results/k6_metrics.json`) for authoritative percentile calculations.
- [ ] Headless Playwright script (`canaries/playback_canary.py`) polls for the plateau signal, launches Chromium against `http://localhost:8080/player/`, and executes continuous playback throughout the peak stress window.
- [ ] Verification smoke test executes k6 with 5 VUs, verifies the plateau signal is created, and confirms the Playwright canary launches and completes successfully.
