# 05: Live Clock Sidecar & Dynamic Manifest with Flaw Toggle (Cache Stampede)

**What to build:** An isolated `live-clock` sidecar container that monotonically advances the live broadcast sequence pointer in Redis every 2 seconds, paired with a dynamic FastAPI live manifest route (`GET /live/live.m3u8`) and an Nginx micro-caching rule with an environment toggle (`FLAW_CACHE_STAMPEDE`).

**Blocked by:** 02: Nginx Reverse Proxy & Static HLS Streaming Gateway

**Status:** ready-for-agent

- [ ] Docker Compose defines a `redis` service on host port `6379`.
- [ ] A dedicated `live-clock` sidecar container runs a single isolated Python process that advances `live:channel:main:sequence` in Redis every 2.0 seconds.
- [ ] FastAPI exposes `GET /live/live.m3u8` which dynamically constructs a sliding-window HLS playlist from the Redis sequence number, pointing to pre-generated synthetic `.ts` segments.
- [ ] Nginx proxies `/live/*` requests to FastAPI with an `X-Cache-Status` header.
- [ ] When `FLAW_CACHE_STAMPEDE=true`, Nginx cache TTL for `/live/*` is 0s (bypassed), causing every concurrent viewer poll to hit FastAPI directly and trigger request amplification.
- [ ] When `FLAW_CACHE_STAMPEDE=false`, Nginx micro-caches the live manifest for 1.0s (`proxy_cache_valid 200 1s`), absorbing concurrent viewer requests within each segment window.
- [ ] Verification test demonstrates sequence numbers advance without split-brain, and Nginx absorbs >90% of manifest traffic in non-flawed mode.
