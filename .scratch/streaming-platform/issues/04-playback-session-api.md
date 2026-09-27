# 04: Playback Session API with Flaw Toggle (Async Event Loop Blocking)

**What to build:** An endpoint for initializing playback sessions (`POST /api/sessions`) that records viewer sessions in the database, featuring an environment-toggled flaw (`FLAW_EVENT_LOOP_BLOCKING`) that executes synchronous CPU-heavy work directly on the asynchronous event loop thread.

**Blocked by:** 03: Catalog Metadata Service with Flaw Toggle (Slow Search & Pool Constriction)

**Status:** ready-for-agent

- [ ] Database schema defines `playback_sessions` table (`id`, `video_id`, `client_ip`, `user_agent`, `started_at`, `is_active`).
- [ ] FastAPI exposes `POST /api/sessions` accepting video ID and client metadata, returning a session token and stream playback manifest URL.
- [ ] When `FLAW_EVENT_LOOP_BLOCKING=true`, session creation runs a synchronous CPU-bound hashing loop (e.g. repeated SHA-256) directly within the async route handler, blocking the single-threaded event loop and starving concurrent requests on unrelated routes.
- [ ] When `FLAW_EVENT_LOOP_BLOCKING=false`, CPU-bound tasks are offloaded to an `asyncio.to_thread` executor pool or bypassed, keeping the event loop free for concurrent I/O.
- [ ] Verification test proves that under concurrent session starts, unrelated `GET /api/catalog` request latencies spike when the flaw is active and remain flat when disabled.
