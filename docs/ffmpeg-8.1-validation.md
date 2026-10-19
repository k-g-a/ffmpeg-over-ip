# FFmpeg 8.1 upgrade validation

This fork pins Jellyfin-FFmpeg **v8.1.3-1**, commit
`253db2a7b0a8045c54ce68ce33d7f601229b1822`. The port is backported from
[steelbrain/ffmpeg-over-ip#22](https://github.com/steelbrain/ffmpeg-over-ip/pull/22),
commit `f6f0dcbb4e0e7cef390c9d41ba0959a2dbc7ac7a` by Anees Iqbal.
Its original 8.1.1 source pin is advanced to 8.1.3-1 here. The upstream fio
patches and MP4-enabled minimal configuration are retained. The Go transport
and wire protocol are unchanged; compatibility with FFmpeg 7-generated
commands is not an acceptance requirement for this fork.

## Build and automated checks

Use a fresh checkout with submodules:

```bash
git clone --branch upgrade/jellyfin-ffmpeg-8.1 --recurse-submodules https://github.com/k-g-a/ffmpeg-over-ip.git
cd ffmpeg-over-ip
go test ./internal/... ./cmd/... -race -count=1 -timeout 120s
make -C fio test
bash scripts/build-ffmpeg.sh --minimal
python3 tests/static-analysis/scan-raw-syscalls.py
```

The minimal build is for transport tests, not Jellyfin deployment. It requires
a C compiler, make, patch and nasm (or `EXTRA_CONFIGURE=--disable-x86asm` for
local portability testing). The Integration workflow builds the fio helper
binaries and runs every `tests/integration/test-*.sh` script.

The Release workflow builds FFmpeg from the pinned source with Jellyfin's
hardware-acceleration dependencies. It packages both client and server ZIPs
as Actions artifacts on PRs and manual runs; only a `v*` tag publishes a release.
It no longer consumes the historical 7.1.3 archives in `artifacts/`.
Both minimal and full-build caches include the actual submodule commit and
relevant build scripts, patches and fio sources.

For a local full Windows build on a Linux Docker host:

```bash
bash scripts/build-release.sh win64
```

The archive appears in `third_party/jellyfin-ffmpeg/builder/artifacts/`.
The existing release builder resets and cleans the submodule before building;
use a disposable checkout if working on FFmpeg source itself. Full builds
compile toolchains and dependencies and can take substantially longer than
minimal builds. Other supported targets remain `linux64`, `linuxarm64`,
`mac64` and `macarm64` (macOS targets require a macOS host).

## Before merging

Keep this PR in the fork and in draft until deployment testing is complete.
A source build or listed encoder does not prove hardware execution works.

- [ ] Full Windows x64 build completes and both executables start on Windows.
- [ ] `ffmpeg.exe -version` and `ffprobe.exe -version` report 8.1.3.
- [ ] `ffmpeg.exe -encoders` lists `h264_qsv` and `hevc_qsv`.
- [ ] Inspect `-hwaccels` and `-filters` for the required QSV/D3D11, hardware
      upload/download, scaling, tone-mapping and subtitle capabilities.
- [ ] Linux client to Windows server: ffprobe, software transcode, remote input
      reads/seeks, output creation, HLS segment rename/deletion, stop/restart,
      concurrent sessions and unavailable-worker/error behaviour.
- [ ] Intel QSV transcode using the actual Windows adapter; confirm GPU use.
- [ ] Capture real Jellyfin Linux argv and validate explicit Windows QSV/D3D11
      rewrites before enabling Jellyfin hardware transcoding.
- [ ] Test 1080p/4K H.264/HEVC, HDR-to-SDR and subtitle burn-in; measure startup,
      seeking and sustained throughput against the current local setup.

Use isolated test configuration and output directories. No Proxmox, GPU
passthrough or production Jellyfin changes are part of this PR. Keep the current
working binaries and configuration for rollback. Official unpatched FFmpeg
binaries cannot replace these patched binaries: they do not tunnel file I/O.

## Local validation results

Recorded during preparation of this draft:

- Baseline 7.1.3 minimal Linux build succeeded.
- 8.1.3 minimal Linux FFmpeg and FFprobe build succeeded, including all 100
  Jellyfin patches; fio patches apply without fuzz.
- Raw syscall scan passes with the existing nine-file audit allowlist.
- Go unit tests pass with the race detector.
- All 11 integration scripts pass with the 8.1.3 build.
- Socket-based fio copy/operation integration passes under ASan/UBSan.
- C fio unit tests: 73/73 pass normally and under ASan/UBSan. Leak detection
  was disabled locally because LeakSanitizer cannot run in this environment.

Windows hardware execution and the full release builds are separate gates;
these local results do not certify them.
