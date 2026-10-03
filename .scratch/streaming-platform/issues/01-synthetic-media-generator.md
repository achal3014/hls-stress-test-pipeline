# 01: Media Preparation Script (`scripts/prepare_media.py`)

**What to build:** An idempotent Python setup script that populates the `media_data` Docker volume with all HLS video assets on first run — downloading 5 real Blender open movies from the Blender Foundation's public CDN for VOD content, and procedurally generating 1 dedicated live asset using FFmpeg's `testsrc` filter. No media files are ever committed to Git (`media/`, `*.ts`, `*.m3u8`, `*.mp4` are `.gitignore`d).

**Blocked by:** None (can start immediately)

**Status:** ready-for-agent

- [ ] A runnable Python script (`scripts/prepare_media.py`) orchestrates all asset preparation and is invoked during `docker compose up` initialization (via `depends_on` or an init container pattern).
- [ ] Script is **idempotent**: checks whether output directories already exist before downloading or generating; re-running `docker compose up` after first setup skips all redundant work.
- [ ] **VOD assets — 5 Blender open movies:**
  - Downloads source MP4s from the Blender Foundation's public CDN if not already cached locally (one-time internet dependency, ~2–3 min on first run). Films: *Big Buck Bunny*, *Sintel*, *Tears of Steel*, *Elephants Dream*, *Sprite Fright*.
  - FFmpeg transcodes each film into dual-rendition HLS: **360p @ 400 kbps** and **720p @ 1.2 Mbps**, with **2-second `.ts` segments** and static playlist files (`master.m3u8`, `360p.m3u8`, `720p.m3u8`).
  - Output written to `/media/vod/vod-0/` through `/media/vod/vod-4/` (one directory per film).
- [ ] **Live asset — 1 procedural FFmpeg video (~8 min):**
  - Generated from the FFmpeg `testsrc` source filter (v1) — no internet access required, fully offline, byte-deterministic output across machines and runs.
  - Same dual-rendition HLS spec (360p/720p, 2-second segments, `master.m3u8`, `360p.m3u8`, `720p.m3u8`).
  - Output written exclusively to `/media/live/`. Never referenced by any VOD catalog entry.
- [ ] `media/`, `*.ts`, `*.m3u8`, and `*.mp4` are listed in `.gitignore`. Zero media bytes are committed to the repository.
- [ ] Verification: running the script on a clean clone produces all expected directories and a valid `master.m3u8` in each, playable via `ffprobe` or curl.

