# 02: Nginx Reverse Proxy & Static HLS Streaming Gateway

**What to build:** An edge reverse proxy and static media gateway container in Docker Compose that serves synthetic HLS playlists and video segments directly from the shared `media_data` volume on host port 8080, with appropriate caching headers, CORS headers, and byte-range support.

**Blocked by:** 01: Procedural Synthetic HLS Video Generator

**Status:** ready-for-agent

- [ ] Docker Compose defines an `nginx` service mapped to host port `8080` (container port `80`).
- [ ] A named Docker volume `media_data` is mounted to Nginx at `/usr/share/nginx/html/media`.
- [ ] Static `.ts` video chunks are served directly from `/media/` with `Content-Type: video/mp2t` and caching headers (`Cache-Control: max-age=86400, public`).
- [ ] Static `.m3u8` playlists are served with `Content-Type: application/vnd.apple.mpegurl` and `Cache-Control: no-cache`.
- [ ] Cross-Origin Resource Sharing (CORS) headers (`Access-Control-Allow-Origin: *`) are enabled for all media endpoints.
- [ ] Client HTTP requests to `http://localhost:8080/media/vod/vod-0/master.m3u8` (and `vod-1` through `vod-4`) return valid playlists and media segments stream smoothly via curl or video player.
