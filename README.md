# exosnap-ffmpeg-build

Pinned, minimal, LGPL-only FFmpeg builds for [ExoSnap](https://github.com/Exoridus/exosnap).

## Purpose

ExoSnap uses FFmpeg's `libavformat`, `libavcodec`, `libavutil`, and `libswresample` for MKV-to-MP4
stream-copy remux (and future Quick Trim). This repository provides a controlled replacement for the
BtbN prebuilt archives: same upstream FFmpeg tag, but built from source with only the components
ExoSnap actually needs, so the artifact footprint is smaller and every configure flag is explicit and
auditable.

**Builds run on demand only** — triggered manually via `workflow_dispatch` or automatically when a
`r*` tag is pushed. There is no continuous nightly schedule.

## Versioning scheme

Release tags follow the pattern `r<N>` (e.g. `r1`, `r2`). The FFmpeg upstream tag being built is
recorded inside the archive in `BUILD-INFO.txt` and in the release title.

| Release tag | FFmpeg upstream ref | Notes |
|-------------|---------------------|-------|
| r1          | n8.1.1              | Initial controlled build; baseline parity with BtbN autobuild-2026-06-11 |
| r2          | n8.1.1              | Adds `mp4` muxer; r1 omitted it so `avformat_alloc_output_context2("mp4", …)` returned AVERROR(EINVAL) |

## What is built

A Windows x86-64 shared (LGPL) build cross-compiled from Linux (mingw-w64), containing only the
components ExoSnap links and uses:

| Component | Type | Reason included |
|-----------|------|-----------------|
| `avformat` | muxer + demuxer | Container I/O — MP4 (mov), Matroska (mkv/webm) |
| `avcodec` | codec parameters | `avcodec_parameters_copy`; no decode/encode |
| `avutil` | utility | Required by avformat and avcodec |
| `swresample` | resampler | Linked as dependency of avformat |
| Demuxer: `matroska` | built-in to avformat | Input format for MKV remux |
| Muxer: `mp4` | built-in to avformat | Output format for MP4 (required by `avformat_alloc_output_context2("mp4", …)`) |
| Muxer: `mov` | built-in to avformat | Output format for MP4 faststart (shares movenc backend with mp4) |
| Muxer: `matroska` | built-in to avformat | Output format for recovery MKV remux |
| Protocol: `file` | built-in | Local file I/O |
| Parser: `h264` | built-in | Stream parameter parsing |
| Parser: `hevc` | built-in | Stream parameter parsing |
| Parser: `av1` | built-in | Stream parameter parsing |
| Parser: `aac` | built-in | Stream parameter parsing |
| Parser: `opus` | built-in | Stream parameter parsing |
| Parser: `vorbis` | built-in | Stream parameter parsing |
| Parser: `mpegaudio` | built-in | Stream parameter parsing |
| BSF: `aac_adtstoasc` | built-in | AAC framing for MP4 (avformat uses it internally) |
| BSF: `extract_extradata` | built-in | Codec extradata extraction (avformat internal use) |
| BSF: `h264_mp4toannexb` | built-in | H.264 format conversion (avformat internal use) |
| BSF: `hevc_mp4toannexb` | built-in | HEVC format conversion (avformat internal use) |
| BSF: `av1_metadata` | built-in | AV1 metadata handling (avformat internal use) |
| BSF: `null` | built-in | No-op passthrough (avformat internal use) |

Components explicitly **not** built: encoders, decoders, avfilter, avdevice, swscale, all programs
(ffmpeg/ffprobe/ffplay), documentation, avresample.

## Artifact layout

The release archive mirrors the BtbN win64-lgpl-shared layout that ExoSnap's CMake expects:

```
ffmpeg-<ref>-win64-lgpl-shared/
  bin/
    avformat-62.dll
    avcodec-62.dll
    avutil-60.dll
    swresample-6.dll
  include/
    libavformat/
    libavcodec/
    libavutil/
    libswresample/
  lib/
    avformat.lib       # MSVC-compatible import lib (generated via llvm-dlltool)
    avcodec.lib
    avutil.lib
    swresample.lib
  LICENSE.md           # FFmpeg LGPL-2.1+ license
  BUILD-INFO.txt       # upstream commit, configure line, toolchain versions
```

The archive is a ZIP. A `SHA256SUMS.txt` file accompanies each release.

## How to trigger a build

### Manual (workflow_dispatch)

```
gh workflow run build.yml \
  --repo Exoridus/exosnap-ffmpeg-build \
  --field ffmpeg_ref=n8.1.1
```

### Tag push (automated release)

```
git tag r2 && git push origin r2
```

The workflow runs automatically, builds, and creates a GitHub Release with the archive attached.

## How ExoSnap consumes the artifacts

See [consume.md](consume.md) for the one-CMake-line switch from BtbN to this repo's releases.

## LGPL §4 compliance

FFmpeg is licensed under **LGPL-2.1-or-later**. ExoSnap distributes FFmpeg as shared DLLs
(dynamic linking), which satisfies LGPL §4:

- **Shared-only**: only `avformat-62.dll`, `avcodec-62.dll`, `avutil-60.dll`,
  `swresample-6.dll` are shipped — no static FFmpeg code is incorporated into `exosnap.exe`.
- **Unmodified upstream**: FFmpeg is built from an unmodified upstream tag. No patches are
  applied. Users can verify this by checking `BUILD-INFO.txt` in the archive against the
  pinned upstream commit.
- **Source offer**: the upstream source is available at the pinned tag on
  <https://github.com/FFmpeg/FFmpeg>. The exact configure flags used are recorded in
  `BUILD-INFO.txt`, enabling anyone to reproduce the build.
- **License shipped**: `LICENSE.md` from the FFmpeg source tree is included in every artifact
  archive. ExoSnap's portable ZIP also stages it next to `exosnap.exe`.
- **User replaceability**: because the DLLs are separate files next to the executable, users
  can substitute their own FFmpeg build without relinking ExoSnap.

The build scripts in this repository are MIT-licensed (see `LICENSE`). FFmpeg itself is
LGPL-2.1+ and unmodified.
