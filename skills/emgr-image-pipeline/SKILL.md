---
name: emgr-image-pipeline
description: "EmgR's codec and transform pipeline — which library does each decode and encode and why, the DCT-scaled JPEG fast path, AVIF's speed and quality constants and the measurement that set them, what is deliberately unsupported (HEIC, PNG quality), the native-library licence obligations, and which ADR numbers are superseded. Load before touching a codec, changing a quality or speed default, adding a format, or quoting a size or timing figure."
---

# emgr-image-pipeline

> **Verified against emgr `446c4df6` (2026-09-19).** Version-sensitive claims
> below carry the date they became true. On an older or newer emgr, trust the
> repository over this page. See
> [VERSIONING.md](https://github.com/vaam-apps/emgr-skills/blob/main/VERSIONING.md).

## Which library does what

| Operation   | Library                                                  |
| ----------- | -------------------------------------------------------- |
| Resize      | `fast_image_resize` (SIMD)                               |
| JPEG decode | `mozjpeg`/libjpeg-turbo, DCT-scaled when possible        |
| JPEG encode | `mozjpeg`, `JCP_FASTEST` profile by default              |
| WebP encode | `webp` crate — real libwebp, lossy + lossless + animated |
| WebP decode | real libwebp via FFI, falling back to `image-webp`       |
| AVIF encode | `libavif-sys` → **AOM**                                  |
| AVIF decode | `libavif-sys` → **dav1d**                                |
| PNG, GIF    | the `image` crate                                        |

**All four native codec libraries are compiled from source and statically
linked** — the runtime container ships no codec `.so` files. This is why the
builder stage installs `nasm cmake meson ninja-build`.

**HEIC is not supported**, as a source or an output.

## Read this before quoting any number

**`adr/0004` is superseded and says so in its own header.** Its two
most-quoted figures — AVIF/JPEG `0.7241x` and AVIF encode `367.8ms` — describe
encoders this service no longer ships and are explicitly marked void. Use
[`adr/0005`](https://github.com/vaam-apps/image-resizer/blob/main/adr/0005-avif-measurement-libavif-mozjpeg.md):

| Figure                      | Current (0005) |   Was (0004) |
| --------------------------- | -------------: | -----------: |
| AVIF / JPEG, matched DSSIM  |       `0.7612` |     `0.7241` |
| AVIF / WebP, matched DSSIM  |       `0.8301` |     `0.8753` |
| AVIF encode, median         |     `119.2 ms` |   `367.8 ms` |
| JPEG encode, median         |      `1.14 ms` |    `4.51 ms` |
| AVIF decode (dav1d), median |      `8.15 ms` | not measured |

`adr/0001`'s own recorded decision — "drop WebP as an output format and adopt
AVIF via `ravif`" — is likewise **not what the code does**. WebP output
exists; AVIF goes through libavif/AOM. `adr/0003` corrects `0001`'s WebP
figure from a synthetic-fixture artifact (`0.96x`) to a Kodak-corpus,
DSSIM-matched median of `0.8399`.

## The constant you will be tempted to change, and why not to

```rust
pub const DEFAULT_AVIF_QUALITY: u8 = 80;   // src/services/image/handler.rs
pub const DEFAULT_AVIF_SPEED:   u8 = 6;
pub const DEFAULT_WEBP_QUALITY: f32 = 82.0;
```

**At production's actual defaults, AVIF ships _more_ bytes than JPEG** — a
median `1.14x` larger than JPEG q75, larger on 19 of 24 Kodak images. The
byte-neutral AVIF quality would be around `65.7`. `adr/0005` measured this and
**deliberately did not propose changing it**; the constant's own doc comment
opens "**Read this before 'optimising' it.**" The trade was considered and
declined. If you want to revisit it, bring a new measurement, not an
intuition.

`DEFAULT_AVIF_SPEED = 6` is **not** on the same scale as the `ravif`/`cavif`
value it replaced. It was re-derived by sweeping 4/6/8/9/10 on AOM, where
speed 6 came out both smaller _and_ lower-DSSIM than speed 8 — a genuinely
dominated choice, not a speed/quality trade. The old value of `4` cost 900ms+
median for no benefit.

AVIF encoding is single-threaded: `avifEncoder.maxThreads` is never set, so
libavif's default of `1` applies.

## The DCT-scaled JPEG decode

`select_jpeg_dct_scale` tries `scale_num` ∈ `[1, 2, 4]` — libjpeg fractions
1/8, 1/4, 1/2 — and picks the most aggressive whose predicted output is still
at least the target in **both** dimensions, falling through to `8` (full
resolution) otherwise. The prediction reproduces libjpeg's own
`jdiv_round_up(dim * scale_num, 8)` exactly, so it matches what
`Decompress::scale` actually produces.

Measured: 4K source → 200×113 thumbnail, **58.03 ms full decode+resize vs
26.21 ms at 1/8 scale — 2.21x**.

The target dimensions come from the **real** `effective_resize_box` /
`resize_dimensions`, not a parallel copy. That matters: an earlier hand-rolled
version under-estimated the target when `zoom`, `dpr`, `min-width` and
`min-height` were combined.

**Fallbacks differ by format, and one has none.** JPEG falls back to the
`image` crate on any error, including a caught panic from the mozjpeg FFI.
WebP falls back the same way. **AVIF has no fallback** — a decode failure is
returned.

## Things that were measured and turned out to be traps

- **Constructing a `mozjpeg::Compress` without `set_fastest_defaults`
  selects `JCP_MAX_COMPRESSION`.** That very nearly shipped as the default for
  every JPEG encode — a +16% pipeline regression. Progressive output
  (`jpgo:1:…`) opts into that profile deliberately, since its ~19x cost is
  caller-chosen.
- **`fq:png` is rejected on purpose.** PNG has no quality knob here: the
  encoder is a fixed `CompressionType::Best`. A benchmark measuring the
  `image` crate's _default_ `CompressionType::Fast` reported `1.71 ms`; the
  real production path is `98.93 ms`, about 56x more.
- **Latent bug, recorded in `adr/0005` and not yet fixed:** `avif_codec::decode`
  passes `max_src_resolution_mp * 1_000_000` straight to libavif's
  `imageSizeLimit`, and libavif rejects any value above its own
  16384×16384 default with `AVIF_RESULT_NOT_IMPLEMENTED`. **Setting
  `MAX_SRC_RESOLUTION_MP` above ~268 breaks all AVIF decoding**, with an
  opaque error unrelated to the image. Check this before raising that variable.
- **Alpha handling for formats without an alpha channel** had a real
  raw-channel-drop fringe bug
  ([#34](https://github.com/vaam-apps/image-resizer/issues/34),
  [#60](https://github.com/vaam-apps/image-resizer/issues/60)) — compositing
  and transparent-pixel normalisation are deliberate, not incidental.

## Licence obligations that travel with the codecs

`NOTICE` and `docs/about/license.md` are not boilerplate. **mozjpeg vendors
Independent JPEG Group code, and the IJG licence requires distributed
documentation to state that the software is "based in part on the work of the
Independent JPEG Group."** `deny.toml`'s allow-list entry for it carries an
explicit warning that this is not a routine allow-list add: if the entry is
ever removed, remove the notice with it; if mozjpeg stays, the notice must
stay. libwebp is BSD-3-Clause via the `webp` crate.

## Pins that are load-bearing

```toml
sha2 = "0.10"   # NOT "0": 0.11 changed finalize()'s return type and breaks the cache key
hmac = "0.12"   # NOT 0.13: it needs digest 0.11, the sha2 0.11 generation above refuses
```

And the release profile uses `panic = "unwind"`, **not** `abort`: this service
decodes attacker-supplied image bytes, so a codec panic under `abort` would
take the process down — a trivial DoS
([#29](https://github.com/vaam-apps/image-resizer/issues/29)).
