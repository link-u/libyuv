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
- `ScalePlane_16` / `I420Scale*` ( `ScalePlane_12` is a `-1` stub for libavif link )
- 10/12-bit and 422/444 / NV12 convert APIs (ifdef’d in `convert_argb.cc`)
- Unused row/scale SIMD and C kernels (via `HAS_*` undef + source guards)

## Link notes for libavif

`src/scale.c` always references `ScalePlane_12` (for `depth > 8`). This profile
provides a stub that returns `-1`. For 8-bit-only Android builds that is fine.

Optional libavif-side cleanup (not required to link):

1. In `src/scale.c`, `#if` out the `depth > 8` / `ScalePlane_12` branches when
   building against this libyuv profile.
2. Narrow `reformat_libyuv.c` LUTs to 8-bit YUV420/400 (+ alpha) so LTO/GC can
   drop more convert kernels. Those LUT function pointers otherwise keep I422/
   I444/10-bit convert symbols alive under `--gc-sections`.
3. Remove or disable RGB→YUV encode tables if encode is unused.
4. Keep `ARGBAttenuate` / `ARGBUnattenuate` for premultiplied alpha.
5. Keep `avifImageScale` → `ScalePlane` for Bitmap size mismatches (Android JNI).
6. Point `AVIF_LIBYUV=LOCAL` at this tree (`link-u/libyuv` `avif` branch).

Unsupported formats then fall back to libavif’s C conversion path.
