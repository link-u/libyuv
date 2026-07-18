# LIBYUV_AVIF_PROFILE

This fork can build a decode-oriented subset for libavif on Android.

## Enable

```bash
cmake -S . -B build -DLIBYUV_AVIF_PROFILE=ON
cmake --build build --target yuv
```

Produces a **static** library only (`yuv` / `libyuv.a`). libavif Android JNI links this archive into `libavif_android.so`.

Android NDK (`Android.mk`): `LIBYUV_AVIF_PROFILE=yes` (default).

## What remains

- 8-bit `I420*Matrix` / `I400*Matrix` / `I420Alpha*Matrix` (+ Filter / RGB24 / RGB565 / RGBA)
- `YuvConstants` (BT.601/709/2020 limited and full, including JPEG full range)
- `ScalePlane` (8-bit) for `avifImageScale`
- `ARGBAttenuate` / related planar helpers still compiled; linker GC drops unused ones
- NEON / NEON64 / SVE / SME (row+scale) kept for speed

## What is omitted

- JPEG / MJPEG, compare, rotate
- RGB→YUV encode paths (`convert.cc`, `convert_from*`, …)
- `scale_argb` / `scale_rgb` / `scale_uv`
- `ScalePlane_16` / `ScalePlane_12` / `I420Scale*`
- 10/12-bit and 422/444 / NV12 convert APIs (ifdef’d in `convert_argb.cc`)

## Required libavif changes

`reformat_libyuv.c` keeps function pointers for I422/I444/10-bit in LUTs. Those references prevent `--gc-sections` from dropping symbols. For the size win:

1. Narrow YUV→RGB LUTs to 8-bit YUV420 / YUV400 (+ alpha) only; set other entries to `NULL` or shrink tables.
2. Remove or disable RGB→YUV encode tables if encode is unused.
3. Keep `ARGBAttenuate` / `ARGBUnattenuate` for premultiplied alpha.
4. Keep `avifImageScale` → `ScalePlane` for Bitmap size mismatches (Android JNI).
5. Point `AVIF_LIBYUV=LOCAL` at this tree.

Unsupported formats then fall back to libavif’s C conversion path.
