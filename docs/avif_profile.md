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
- `ARGBAttenuate` / `ARGBUnattenuate` / `CopyPlane` (other `planar_functions` APIs ifdef’d out)
- NEON / NEON64 / SVE / SME (row+scale) kept for speed

## Kernel trimming (LIBYUV_AVIF_PROFILE)

Unused SIMD `HAS_*` macros are `#undef`'d via:

- [`include/libyuv/avif_profile_undef_row_has.h`](../include/libyuv/avif_profile_undef_row_has.h)
- [`include/libyuv/avif_profile_undef_scale_has.h`](../include/libyuv/avif_profile_undef_scale_has.h)

Regenerate with `python tools/gen_avif_profile_undef_has.py`.

Unused C fallbacks in `row_common.cc` / `scale_common.cc` are wrapped with
`#if !defined(LIBYUV_AVIF_PROFILE)` (see `tools/guard_row_common_avif.py`,
`tools/guard_scale_common_avif.py`).

Non-MSVC builds also enable `-ffunction-sections` and LTO/IPO when available so
the final `libavif_android.so` can GC leftovers after libavif LUT trimming.

## What is omitted

- JPEG / MJPEG, compare, rotate
- RGB→YUV encode paths (`convert.cc`, `convert_from*`, …)
- `scale_argb` / `scale_rgb` / `scale_uv`
- `ScalePlane_16` / `ScalePlane_12` / `I420Scale*`
- 10/12-bit and 422/444 / NV12 convert APIs (ifdef’d in `convert_argb.cc`)
- Unused row/scale SIMD and C kernels (via `HAS_*` undef + source guards)

## Required libavif changes

`reformat_libyuv.c` keeps function pointers for I422/I444/10-bit in LUTs. Those references prevent `--gc-sections` from dropping symbols. For the size win:

1. Narrow YUV→RGB LUTs to 8-bit YUV420 / YUV400 (+ alpha) only; set other entries to `NULL` or shrink tables.
2. Remove or disable RGB→YUV encode tables if encode is unused.
3. Keep `ARGBAttenuate` / `ARGBUnattenuate` for premultiplied alpha.
4. Keep `avifImageScale` → `ScalePlane` for Bitmap size mismatches (Android JNI).
5. Point `AVIF_LIBYUV=LOCAL` at this tree.

Unsupported formats then fall back to libavif’s C conversion path.
