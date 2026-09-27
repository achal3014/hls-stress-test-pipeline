# 06: Live Viewer Counter with Flaw Toggle (Lost Updates vs. Redis ZSET)

**What to build:** Client heartbeat and live viewer query endpoints (`POST /api/live/heartbeat`, `GET /api/live/viewers`) with an environment-toggled concurrency flaw (`FLAW_COUNTER_RACE`) demonstrating lost updates and visual counter drift under load versus atomic, self-healing Redis Sorted Sets.

**Blocked by:** 05: Live Clock Sidecar & Dynamic Manifest with Flaw Toggle (Cache Stampede)

**Status:** ready-for-agent

- [ ] FastAPI exposes `POST /api/live/heartbeat` accepting `{ "user_id": "<id>", "channel_id": "<channel>" }` sent by active stream viewers every 5 seconds.
- [ ] FastAPI exposes `GET /api/live/viewers?channel_id=<channel>` returning `{ "channel_id": "<channel>", "viewers": <int> }`.
- [ ] When `FLAW_COUNTER_RACE=true`, the heartbeat handler performs an unsynchronized read-modify-write in Redis (`current = await redis.get(); await redis.set(int(current) + 1)`), causing severe lost updates under concurrent client traffic.
- [ ] When `FLAW_COUNTER_RACE=false`, the heartbeat handler uses a Redis Sorted Set (`ZADD live:channel:<channel>:active_viewers <now_timestamp> <user_id>`), prunes sessions older than 10 seconds (`ZREMRANGEBYSCORE`), and returns the exact cardinal count (`ZCARD`).
- [ ] Concurrency test proves that under 200 concurrent simulated heartbeat clients, the flawed mode reports significantly fewer viewers than actual connected clients, while the fixed mode maintains 100% counting accuracy.
