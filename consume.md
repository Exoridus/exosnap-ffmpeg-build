# Switching ExoSnap from BtbN to exosnap-ffmpeg-build

When a new release is available on this repo, update **one block** in
`cmake/VendorFFmpeg.cmake` inside the ExoSnap repository.

## What to change

Find the `FetchContent_Declare` block (currently pointing at BtbN):

```cmake
FetchContent_Declare(
    ffmpeg_prebuilt
    URL      "https://github.com/BtbN/FFmpeg-Builds/releases/download/autobuild-2026-06-11-14-22/ffmpeg-n8.1.1-12-ge0e38acd2f-win64-lgpl-shared-8.1.zip"
    URL_HASH "SHA256=4C9A9CB7D8B4941DA7B0F6B086A2232D3328B1583FC07F9BC2DD79C05A96ED8B"
    DOWNLOAD_EXTRACT_TIMESTAMP TRUE
)
```

Replace it with the URL and SHA256 from the desired release on this repo:

```cmake
FetchContent_Declare(
    ffmpeg_prebuilt
    URL      "https://github.com/Exoridus/exosnap-ffmpeg-build/releases/download/r1/ffmpeg-n8.1.1-win64-lgpl-shared.zip"
    URL_HASH "SHA256=<sha256-from-SHA256SUMS.txt>"
    DOWNLOAD_EXTRACT_TIMESTAMP TRUE
)
```

The archive layout (`bin/`, `include/`, `lib/`) and the DLL filenames
(`avformat-62.dll`, `avcodec-62.dll`, `avutil-60.dll`, `swresample-6.dll`)
and import lib names (`avformat.lib`, etc.) are identical to BtbN, so no
other CMake changes are required.

## Finding the SHA256

Each release on this repo includes a `SHA256SUMS.txt` asset. Download it and
copy the hash for the `.zip` file:

```
gh release download r1 \
  --repo Exoridus/exosnap-ffmpeg-build \
  --pattern SHA256SUMS.txt \
  --output -
```

## Version informational string

Also update the informational cache variable in `VendorFFmpeg.cmake` if you
want to track the exact ref:

```cmake
set(EXOSNAP_FFMPEG_VERSION "n8.1.1"
    CACHE STRING "Pinned FFmpeg build version tag (informational)")
```

## No other changes needed

- `_exosnap_ffmpeg_target()` calls (DLL/lib filenames) stay the same.
- `EXOSNAP_FFMPEG_DLLS` list stays the same.
- The PATH injection and install/license staging blocks are unaffected.
