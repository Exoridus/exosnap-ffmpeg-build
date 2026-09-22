# exosnap-ffmpeg-build

Pinned, minimal FFmpeg builds for [ExoSnap](https://github.com/Exoridus/exosnap). The default branch
builds one profile and only one: LGPL-2.1-or-later, no GPL code, no nonfree code. r6 is the single
release that ever carried `--enable-gpl` with libx264/libx265, and it is not on this line.

## Current ExoSnap pin: r7

ExoSnap's `cmake/VendorFFmpeg.cmake` pins `r7` (upstream n8.1.1) together with the SHA256 of
`ffmpeg-win64-lgpl-shared.zip`. r7 continues the LGPL r5 line and adds the D3D11VA/DXVA2 hardware
decode accelerators the Edit-page player needs.

### Why r6 is not the latest recipe

r6 was built on 2026-07-23 as a feasibility proof that this pipeline could cross-compile libx264 and
libx265. It was reverted the same day on the ExoSnap side after a patent-licensing risk review:
shipping a compiled software H.264/HEVC encoder makes ExoSnap the patent-pool "product manufacturer"
of record, with no legal budget to clear that. See ExoSnap's
`docs/decisions/0007-software-encoding-via-x264.md` ("2026-07-23 revision") for the reasoning.

The `r6` tag and its release stay published, because release tags here are immutable and are never
deleted or rewritten. The recipe that produced it lives on the `feature/gpl-x264-x265` branch. What
changed on 2026-09-22 is that the default branch no longer carries it: for several weeks `main` held
the GPL recipe while every consumed release came from the LGPL line, so anyone reading the default
branch, or triggering a `workflow_dispatch` from it, would have got a GPL artifact by default. The
default branch now is the recipe that r7 was built from.

Any future software-encoder work in ExoSnap is planned as runtime detection of a user-supplied,
independently sourced FFmpeg build, not as a bundled dependency of this repository.

## Purpose

ExoSnap uses FFmpeg's `libavformat`, `libavcodec`, `libavutil`, and `libswresample` for MKV-to-MP4
stream-copy remux (and future Quick Trim). This repository provides a controlled replacement for the
BtbN prebuilt archives: same upstream FFmpeg tag, but built from source with only the components
ExoSnap actually needs, so the artifact footprint is smaller and every configure flag is explicit and
auditable.

**Builds run on demand only** — triggered manually via `workflow_dispatch` or automatically when a
`r*` tag is pushed. There is no continuous nightly schedule.

## Versioning scheme

A release tag is `n<upstream>-exosnap.<revision>`, for example `n9.0.2-exosnap.1`. It answers two
questions and no others: which upstream ref was built, and which revision of our recipe built it.
The revision starts at 1 for each upstream ref and only increases when the recipe changes while the
upstream ref stays put.

| what changed | next tag after `n9.0.2-exosnap.1` |
|---|---|
| the recipe, same upstream | `n9.0.2-exosnap.2` |
| an upstream patch release | `n9.0.3-exosnap.1` |
| an upstream minor or major | `n9.1-exosnap.1`, `n10.0-exosnap.1` |

`upstreamTag` is written exactly as upstream writes it, so a series upstream tags `n9.1` is not
padded to `n9.1.0`.

**The profile is deliberately not in the tag.** Whether a package is LGPL-only is a fact to verify
in the archive, not a string to trust in a file name. `BUILD-INFO.json` states it, the build
self-checks it before packaging, and the consumer refuses a mismatch. A tag can only ever be a
claim.

The tag is also the only place the upstream version comes from on a tag build. Before this scheme
the workflow fell back to a hard-coded `n8.1.1` on any tag push, so a tag and the bytes it named
could disagree without anything noticing.

### Historical releases

`r1` through `r7` stay published exactly as they are. Nothing about them is renamed, rewritten or
deleted. The workflow still accepts an `r*` tag in its trigger so those references keep resolving,
but pushing one now fails with an explanation: an r-tag carries no upstream version, so the workflow
cannot know what it would have to build.

| Release tag | FFmpeg upstream ref | License | Notes |
|-------------|---------------------|---------|-------|
| r1          | n8.1.1              | LGPL-2.1-or-later | Initial controlled build; baseline parity with BtbN autobuild-2026-06-11 |
| r2          | n8.1.1              | LGPL-2.1-or-later | Adds `mp4` muxer; r1 omitted it so `avformat_alloc_output_context2("mp4", …)` returned AVERROR(EINVAL) |
| r3          | n8.1.1              | LGPL-2.1-or-later | Adds `mov` demuxer; r2 could write MP4 but `avformat_open_input` on the output failed (test verification + any read-back of MP4 files) |
| r4          | n8.1.1              | LGPL-2.1-or-later | Adds decoders: h264, hevc, av1, opus, aac, flac, pcm_s16le, pcm_s24le, pcm_s32le, pcm_f32le (Edit-page video player needs real decode; r1-r3 were mux/demux-only) |
| r5          | n8.1.1              | LGPL-2.1-or-later | Adds encoder: aac (FfmpegAacEncoder, ADR 0052) |
| r6          | n8.1.1              | GPL-2.0-or-later | Off the line. Adds `--enable-gpl`, libx264 + libx265 (cross-compiled, static) and their encoders. Reverted on the ExoSnap side the same day; recipe kept on `feature/gpl-x264-x265`, never consumed |
| r7          | n8.1.1              | LGPL-2.1-or-later | Branches from r5, not from r6. Adds the `h264`/`hevc`/`av1` × `d3d11va`/`d3d11va2`/`dxva2` hardware accelerators for the Edit-page player's GPU decode path. **Currently pinned by ExoSnap.** |

## What is built

A Windows x86-64 shared LGPL-2.1-or-later build cross-compiled from Linux (mingw-w64), containing
only the components ExoSnap links and uses:

| Component | Type | Reason included |
|-----------|------|-----------------|
| `avformat` | muxer + demuxer | Container I/O — MP4 (mov), Matroska (mkv/webm) |
| `avcodec` | codec plumbing | `avcodec_parameters_copy`; also carries the built-in decoders, the AAC encoder and the hardware accelerators listed below |
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
| Decoders: `h264`,`hevc`,`av1`,`opus`,`aac`,`flac`,`pcm_s16le`,`pcm_s24le`,`pcm_s32le`,`pcm_f32le` | built-in to avcodec | Edit-page video player decode |
| Hwaccels: `h264`,`hevc`,`av1` × `d3d11va`,`d3d11va2`,`dxva2` | built-in to avcodec | GPU decode for the Edit-page player, vendor-neutral. Frames come back as `ID3D11Texture2D`; no vendor SDK and no CUDA interop is linked in |

Components explicitly **not** built: avfilter, avdevice, swscale, all programs (ffmpeg/ffprobe/ffplay),
documentation, avresample, and every external library, libx264 and libx265 included.

## Artifact layout

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
  LICENSE.md           # FFmpeg LGPL-2.1-or-later license
  BUILD-INFO.json      # the machine-readable identity of this package
```

The archive is a ZIP named `ffmpeg-<upstream>-win64-lgpl-shared.zip`, so two releases no longer
produce identically named downloads. A `SHA256SUMS.txt` file accompanies each release.

`BUILD-INFO.json` is what a consumer verifies against:

```json
{
  "schema": 1,
  "packageTag": "n9.0.2-exosnap.1",
  "upstreamTag": "n9.0.2",
  "upstreamCommit": "...",
  "packageRevision": 1,
  "profile": "lgpl-shared",
  "architecture": "win64",
  "linkage": "shared",
  "libraries": { "avformat": 63, "avcodec": 63, "avutil": 61, "swresample": 7 },
  "target": "x86_64-w64-mingw32",
  "buildDate": "...",
  "buildHost": "...",
  "toolchain": "...",
  "configure": "...",
  "producedBy": "https://github.com/Exoridus/exosnap-ffmpeg-build"
}
```

The library majors are read off the DLLs that were actually produced, never declared, so the file
cannot claim a major the archive does not carry. That is what lets a consumer pin the four numbers
and fail its configure with a named error when an upstream major moves, instead of discovering it at
link time or at runtime.

The `profile` field is backed by a self-check that runs before packaging: the configure line must
not enable `gpl` or `nonfree`, and FFmpeg's own `ffbuild/config.h` must report `CONFIG_GPL 0` and
`CONFIG_NONFREE 0`. Both sides are checked, what was asked for and what the build says it did.

The r6 archive is the one exception to that layout: its directory is named `win64-gpl-shared` and it
carries `LICENSE-x264.md` and `LICENSE-x265.md` next to a GPL `LICENSE.md`.

## How to trigger a build

### Manual (workflow_dispatch)

A dispatch builds and uploads a workflow artifact, and publishes no release.

```
gh workflow run build.yml \
  --repo Exoridus/exosnap-ffmpeg-build \
  --field upstream_ref=n9.0.2 \
  --field package_revision=1
```

### Tag push (automated release)

```
git tag n9.0.2-exosnap.1 && git push origin n9.0.2-exosnap.1
```

The workflow runs automatically, builds, and creates a GitHub Release with the archive attached. A
tag that does not match the scheme fails in the first step, before a toolchain is installed.

## How ExoSnap consumes the artifacts

See [consume.md](consume.md) for the one-CMake-line switch from BtbN to this repo's releases.

## LGPL compliance

The artifact this branch produces is LGPL-2.1-or-later. ExoSnap is GPL-3.0-or-later and links the
DLLs dynamically, so the obligations that matter are the LGPL §4 ones:

- **Unmodified upstream**: FFmpeg is built from unmodified upstream source at the pinned tag. No
  patches are applied. `BUILD-INFO.json` records the exact commit and the full configure line.
- **Source offer**: FFmpeg source at the pinned tag, https://github.com/FFmpeg/FFmpeg.
- **License shipped**: `LICENSE.md` is included in every artifact archive.
- **Relinkable**: the libraries ship as DLLs with their import libraries, so a recipient can replace
  them with their own build.

r6 is the one release built with `--enable-gpl` and statically linked libx264/libx265, which makes
that archive GPL-2.0-or-later. It is not consumed by ExoSnap and is not what this branch builds; its
recipe is on `feature/gpl-x264-x265`.

The build scripts in this repository are MIT-licensed (see `LICENSE`).
