# 12: Unified Experiment Runner CLI (`run_experiment.py`) & Markdown Report Pipeline

**What to build:** The core single-command pipeline orchestrator (`experiments/run_experiment.py`) that applies flaw configurations, recreates containers, polls health, executes k6 and Playwright, computes Prometheus metric deltas, and generates structured before/after Markdown comparison reports in `reports/`.

**Blocked by:** 11: Multi-Layer Prometheus Metrics & Correlated Grafana Dashboards

**Status:** ready-for-agent

- [ ] CLI script `python experiments/run_experiment.py --scenario <name> --flaw <name> [--baseline]` implements end-to-end experiment execution.
- [ ] Automatically updates configuration flags in `.env` and issues `docker compose up -d --force-recreate <services>` to ensure clean flag application.
- [ ] Polls the application `/health` endpoint until the system is healthy before launching load generation.
- [ ] Executes host-side k6 load test and coordinates Playwright canary launch upon detection of the `k6_plateau.json` signal file.
- [ ] Pauses for Prometheus scrape interval flush (`scrape_interval + 5s`) after load completion.
- [ ] Snapshots Prometheus metrics before and after the test to compute metric deltas, and extracts authoritative latency percentiles directly from k6 JSON output.
- [ ] Generates a formatted Markdown report in `reports/<scenario>_<flaw>_<timestamp>.md` showing side-by-side comparison tables (Baseline vs Flawed) of p50/p95/p99 latency, error rates, rebuffer stall counts, and viewer count drift.
- [ ] Full end-to-end verification demonstrates executing an automated test run for all 6 flaws with zero manual intervention.
