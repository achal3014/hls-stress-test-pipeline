# 01: Procedural Synthetic HLS Video Generator

**What to build:** An automated media generator script that procedurally synthesizes 8 minutes of dual-rendition HLS video streams with embedded millisecond timecode clocks and audio test tones directly into the target media volume, without downloading external media files or committing binaries to Git.

**Blocked by:** None (can start immediately)

**Status:** ready-for-agent

- [ ] A runnable Python/FFmpeg script (`scripts/generate_media.py`) generates HLS content using procedural video filters (`testsrc2`, `sine`).
- [ ] Generates two quality renditions: 360p @ 400kbps and 720p @ 1.2Mbps, chunked into 2-second `.ts` segments (approx. 240 segments per rendition).
- [ ] Generates master and variant HLS playlist files (`master.m3u8`, `360p.m3u8`, `720p.m3u8`) with valid HLS tags (`#EXT-X-VERSION:3`, `#EXT-X-TARGETDURATION:2`).
- [ ] Each frame displays an on-screen running millisecond clock for visual verification of stream latency and playback synchronization.
- [ ] Generates media into the target directory in under 30 seconds with 0 bytes committed to version control.
