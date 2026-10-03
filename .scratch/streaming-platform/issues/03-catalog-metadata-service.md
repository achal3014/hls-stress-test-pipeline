# 03: Catalog Metadata Service with Flaw Toggle (Slow Search & Pool Constriction)

**What to build:** A video catalog service with PostgreSQL database persistence and FastAPI REST endpoints (`/api/catalog`, `/api/catalog/search`), pre-seeded with 10,000 video records, featuring an environment-toggled database flaw (`FLAW_DB_SLOW_QUERY`) that drops the search index and constrains the database connection pool.

**Blocked by:** 02: Nginx Reverse Proxy & Static HLS Streaming Gateway

**Status:** ready-for-agent

- [ ] Docker Compose defines a `postgres` service mapped to host port `5433` (container port `5432`) to avoid local host port collisions.
- [ ] Database schema defines `videos` table (`id`, `title`, `slug`, `description`, `category`, `tags`, `duration_seconds`, `hls_manifest_path`, `created_at`).
- [ ] Automated database seeder populates the database with 10,000 synthetic catalog records during initial startup, distributing them evenly across the 5 VOD Blender film assets (`hls_manifest_path = /media/vod/vod-{id % 5}/master.m3u8`, ~2,000 entries per film). No catalog entry references the live asset path (`/media/live/`).
- [ ] FastAPI service exposes `GET /api/catalog` (paginated listing) and `GET /api/catalog/search?q=<query>`.
- [ ] Nginx proxies `/api/*` to the FastAPI backend service.
- [ ] When `FLAW_DB_SLOW_QUERY=true`, the database search index is omitted and pool size is clamped to 5; search queries execute sequential table scans with measurable queuing under load.
- [ ] When `FLAW_DB_SLOW_QUERY=false`, B-tree/GIN index is enabled and connection pool is set to 25; p95 search latency drops to <20ms.
