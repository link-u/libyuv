# LIBYUV_AVIF_PROFILE

This fork builds a decode-oriented subset for libavif Android slim
(8-bit YUV420/YUV400/YUV444, RGBA Bitmap, bilinear 420).

## Enable

```bash
cmake -S . -B build -DLIBYUV_AVIF_PROFILE=ON
cmake --build build --target yuv
```

Produces a **static** library only (`yuv` / `libyuv.a`). libavif Android JNI
links this archive into `libavif_android.so`.

Android NDK (`Android.mk`): `LIBYUV_AVIF_PROFILE=yes` (default).

## Runtime APIs that remain

These match the Android slim / decode-only call sites:

| API | Role |
|-----|------|
| `I420ToARGBMatrixFilter` | 8-bit YUV420 → RGBA, always bilinear |
| `I420AlphaToARGBMatrixFilter` | 8-bit YUVA420 → RGBA, always bilinear |
| `I444ToARGBMatrix` | 8-bit YUV444 → RGBA (no chroma upsample) |
| `I444AlphaToARGBMatrix` | 8-bit YUVA444 → RGBA |
| `ARGBAttenuate` | premultiply RGB×A/255 |
| `ScalePlane` | 8-bit plane resize (bilinear; YUV400 = Y only) |
| `CopyPlane` | identity scale path inside `ScalePlane` |

Filter entry points ignore `filter` under this profile and always run the
bilinear helpers (`I420ToARGBMatrixBilinear` /
`I420AlphaToARGBMatrixBilinear`). JNI treats libavif RGBA as libyuv ABGR via
YVU matrices (`lutIsYVU[RGBA]=true`).

### Color matrices

Kept (YUV + YVU pair) for Android slim:

- `kYuvH709Constants` / `kYvuH709Constants` — limited + BT.709 (AV1)
- `kYuvJPEGConstants` / `kYvuJPEGConstants` — full + BT.601 (AV2)
- `kYuvF709Constants` / `kYvuF709Constants` — full + BT.709 (AV2)

JNI uses YVU (`lutIsYVU[RGBA]=true`). BT.2020 and limited BT.601 matrices
are omitted under `LIBYUV_AVIF_PROFILE`. `CMAKE_DISABLE_FIND_PACKAGE_JPEG`
only skips libyuv's MJPEG decoder; it does not remove `kYuvJPEGConstants`.

## Kernel trimming

Unused SIMD `HAS_*` macros are `#undef`'d via:

- [`include/libyuv/avif_profile_undef_row_has.h`](../include/libyuv/avif_profile_undef_row_has.h)
- [`include/libyuv/avif_profile_undef_scale_has.h`](../include/libyuv/avif_profile_undef_scale_has.h)

Regenerate with `python tools/gen_avif_profile_undef_has.py`.

Kept row SIMD prefixes: `I444ToARGBRow`, `I444AlphaToARGBRow`, `CopyRow`,
`ARGBAttenuateRow`, `InterpolateRow` (8-bit).

Kept scale SIMD prefixes: `FixedDiv*`, `ScaleCols*`, `ScaleFilterCols*`,
`ScaleRowDown2/34/38`, `ScaleRowUp2_Linear/Bilinear`.

C fallbacks: `python tools/guard_row_common_avif.py --force`,
`python tools/guard_scale_common_avif.py --force`.

`convert_argb.cc` keep-set: `python tools/trim_convert_argb_avif.py`.

NEON / NEON64 / SVE / SME row+scale objects stay enabled for speed.

Non-MSVC builds also enable `-ffunction-sections` and LTO/IPO when available.

## Omitted (non-exhaustive)

- Nearest `I420ToARGBMatrix` / `I420AlphaToARGBMatrix` and convenience wrappers
- `I400*` / `J400*`, RGB24 / RGB565 / RGBA convert paths
- `ARGBUnattenuate`, `ARGBCopy`, encode `ArgbConstants` (+ neon RGB→YUV wrappers that referenced them)
- MJPEG, compare, rotate, RGB→YUV
- `scale_argb` / `scale_rgb` / `scale_uv`, `ScalePlaneBox`, `ScalePlaneDown4`
- `ScalePlane_16` / `I420Scale*` (`ScalePlane_12` is a `-1` stub for libavif link)

`I444ToARGBRow_Any_NEON` is gated on `HAS_I444TOARGBROW_NEON` (not I422), so the
bilinear 420 path and I444 matrix path still link after I422 kernels are trimmed.

## Link notes for libavif

`src/scale.c` may still reference `ScalePlane_12` (depth > 8). This profile
provides a stub that returns `-1`. For 8-bit-only Android builds that is fine.

Point `AVIF_LIBYUV=LOCAL` at this tree (`avif` branch). Keep final
`libavif_android.so` link with `-Wl,--gc-sections` (and LTO when possible);
avoid `LOCAL_WHOLE_STATIC_LIBRARIES` so unused kernels can be GC'd.
