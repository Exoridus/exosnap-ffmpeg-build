# exosnap-ffmpeg-build

Pinned, minimal FFmpeg builds for [ExoSnap](https://github.com/Exoridus/exosnap). GPL-2.0-or-later
since r6 (libx264/libx265); r1-r5 were LGPL-2.1-or-later only.

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
| r3          | n8.1.1              | Adds `mov` demuxer; r2 could write MP4 but `avformat_open_input` on the output failed (test verification + any read-back of MP4 files) |
| r4          | n8.1.1              | Adds decoders: h264, hevc, av1, opus, aac, flac, pcm_s16le, pcm_s24le, pcm_s32le, pcm_f32le (Edit-page video player needs real decode; r1-r3 were mux/demux-only) |
| r5          | n8.1.1              | Adds encoder: aac (FfmpegAacEncoder, ADR 0052) |
| r6          | n8.1.1              | Adds `--enable-gpl`, libx264 + libx265 (cross-compiled, static) and their encoders. **License changes from LGPL-2.1-or-later to GPL-2.0-or-later as of this release.** |

## What is built

A Windows x86-64 shared (GPL-2.0-or-later as of r6) build cross-compiled from Linux (mingw-w64), containing only the
components ExoSnap links and uses:

| Component | Type | Reason included |
|-----------|------|-----------------|
| `avformat` | muxer + demuxer | Container I/O — MP4 (mov), Matroska (mkv/webm) |
| `avcodec` | codec plumbing | `avcodec_parameters_copy`; also carries built-in decoders/encoders listed below (since r4–r6) |
| `avutil` | utility | Required by avformat and avcodec |
| `swresample` | resampler | Linked as dependency of avformat |
| Demuxer: `mov` | built-in to avformat | Input format for MP4/MOV files (required to read back written MP4) |
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
| Encoder: `aac` | built-in to avcodec | Native AAC-LC encode (FfmpegAacEncoder, ADR 0052) |
| Encoder: `libx264` | external, statically linked | Software H.264 encode (ADR 0007; not yet wired into an ExoSnap encoder backend) |
| Encoder: `libx265` | external, statically linked | Software HEVC encode (no ADR yet; build-capability only) |
| Decoders: `h264`,`hevc`,`av1`,`opus`,`aac`,`flac`,`pcm_s16le`,`pcm_s24le`,`pcm_s32le`,`pcm_f32le` | built-in to avcodec | Edit-page video player decode |

Components explicitly **not** built: avfilter, avdevice, swscale, all programs (ffmpeg/ffprobe/ffplay),
documentation, avresample, 10/12-bit x265 multilib.

## Artifact layout

The release archive mirrors the BtbN win64-gpl-shared layout that ExoSnap's CMake expects:

```
ffmpeg-<ref>-win64-gpl-shared/
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
  LICENSE.md           # FFmpeg GPL-2.0-or-later license
  LICENSE-x264.md      # x264 GPL-2.0-or-later license
  LICENSE-x265.md      # x265 license
  BUILD-INFO.txt       # upstream commits (FFmpeg, x264, x265), configure line, toolchain versions
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

## GPL compliance

As of r6, FFmpeg is built with `--enable-gpl` (libx264, libx265), making the combined artifact
**GPL-2.0-or-later**. ExoSnap itself is GPL-3.0-or-later (compatible), so this is not a change in
ExoSnap's own distribution obligations, only in how this specific artifact's license is described:

- **Unmodified upstream**: FFmpeg, x264, and x265 are all built from unmodified upstream source.
  No patches are applied to any of them. `BUILD-INFO.txt` records the exact commit of each.
- **Source offer**: FFmpeg source at the pinned tag (https://github.com/FFmpeg/FFmpeg), x264 and
  x265 source at their pinned commits (recorded in `BUILD-INFO.txt` and the release notes).
- **License shipped**: `LICENSE.md` (FFmpeg), `LICENSE-x264.md`, and `LICENSE-x265.md` are all
  included in every artifact archive.
- r1-r5 remain LGPL-2.1-or-later only (no GPL code); only r6 onward includes libx264/libx265.

The build scripts in this repository are MIT-licensed (see `LICENSE`).
