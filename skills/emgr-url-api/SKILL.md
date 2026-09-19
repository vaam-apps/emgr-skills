---
name: emgr-url-api
description: "EmgR's wire contract — the imgproxy-compatible signed-path grammar, every processing-option code it accepts and the several it deliberately only half-implements, the four resize modes, Accept-header content negotiation for .auto, named presets and the option allowlist, and every response status. Load before building or debugging a request URL, adding a processing option, or reasoning about what a caller can and cannot ask for."
---

# emgr-url-api

> **Verified against emgr `446c4df6` (2026-09-19).** Version-sensitive claims
> below carry the date they became true. On an older or newer emgr, trust the
> repository over this page. See
> [VERSIONING.md](https://github.com/vaam-apps/emgr-skills/blob/main/VERSIONING.md).

```
GET /{signature}/{processing_options}/{plain|base64url source}.{extension}
```

Path segments, not query parameters — the reasoning is in
[ADR 0002](https://github.com/vaam-apps/image-resizer/blob/main/adr/0002-url-api-shape.md).
Signature verification is `emgr-security`'s subject; this page is the grammar
after it — except for [`references/signing-a-url.md`](references/signing-a-url.md),
which walks building one end to end against a vector the source repository
pins in its own tests.

## How the parser tells an option from the source

**A processing-option segment always contains a `:`; a base64url source
segment never can**, because `:` is not in the base64url alphabet. That single
check (`looks_like_option`, `src/modules/url/options.rs`) is the whole
boundary. It is worth knowing because it explains why a _plain_ (unencoded)
source URL — which does contain `://` — must be the last segment and is
recognised positionally, not by shape.

## The option codes

Parsed in `ProcessingOptions::parse` (`src/modules/url/options.rs`). An
unknown code is a hard `UrlParseError::UnknownOption`, never ignored.

| Code        | Args                                           | Effect                                                     |
| ----------- | ---------------------------------------------- | ---------------------------------------------------------- |
| `rs`        | `{type}:{width}:{height}` (`0` = unset)        | Resize — see the four modes below                          |
| `q`         | `0`–`100`                                      | Global quality                                             |
| `fq`        | `{format}:{quality}` pairs                     | Per-format quality. **`png` is rejected**                  |
| `webpo`     | `lossy` \| `lossless`                          | WebP mode — **partial**, see below                         |
| `jpgo`      | `{progressive}[:{no_subsample}]`               | **Partial** — 2 of imgproxy's 6 slots                      |
| `mb`        | `{bytes}`                                      | `max_bytes` cap; applied to JPEG output only               |
| `bl`        | `{sigma}`                                      | Blur                                                       |
| `g`         | `{bool}`                                       | Grayscale                                                  |
| `el`        | `{bool}`                                       | Enlarge — permission to upscale                            |
| `bg`        | `R:G:B` or hex                                 | Background colour                                          |
| `ar`        | `{bool}`, default **`true`**                   | EXIF auto-rotate                                           |
| `sm`        | `{bool}`, default **`true`**                   | Strip metadata                                             |
| `c`         | `{w}:{h}[:{gravity…}]`                         | Explicit crop                                              |
| `gr`        | `{type}[:{x}:{y}]`                             | Gravity — **`gr`, not `g`**, which is grayscale            |
| `rot`       | `{angle}`                                      | Rotate; must be a multiple of 90                           |
| `fl`        | `{h}:{v}`                                      | Flip                                                       |
| `t`         | `{threshold}[:{color}:{eq_hor}:{eq_ver}]`      | Trim                                                       |
| `ex`        | `{bool}`                                       | Extend — **1 arg only**, imgproxy's `:gravity` is rejected |
| `pd`        | `{top}[:{right}:{bottom}:{left}]`              | Padding, CSS-shorthand cascade                             |
| `z`         | `{zoom}` or `{zx}:{zy}`                        | Zoom multiplier                                            |
| `dpr`       | `{value}`                                      | DPR multiplier                                             |
| `mw` / `mh` | `{px}`                                         | Minimum width / height floor                               |
| `wm`        | `{opacity}[:{position}[:{x}[:{y}[:{scale}]]]]` | Watermark; opacity enables it                              |
| `wmu`       | base64url URL                                  | Per-request watermark source                               |
| `wms`       | `{w}:{h}` (`0` = derive)                       | Watermark size                                             |
| `wmr`       | `{angle}`                                      | Watermark rotation                                         |
| `wmsh`      | `{sigma}`                                      | Watermark shadow blur                                      |
| `pr`        | `{name}[:{name}…]`                             | Preset expansion — see below                               |

**`sm` defaults to stripping.** EXIF is removed from output unless a caller
opts in with `sm:0` — a privacy-motivated default
([#5](https://github.com/vaam-apps/image-resizer/issues/5)), not a format
limitation.

### Partial by design, and it refuses rather than ignores

`webpo` and `jpgo` implement a strict subset of imgproxy's slots, and **extra
arguments are rejected, not silently dropped**. That is the right behaviour
and worth preserving: a silently-ignored option is the "flag that does
nothing" failure this project has already shipped once (see the JPEG
progressive bug in `.bench-baseline/BASELINE.md`). Gravity's watermark
positions likewise exclude `re` (repeat) and `ch` (chessboard), documented and
unimplemented.

## The four resize modes (`src/models/params.rs`)

- **`fit`** — scale to fit _inside_ the box, aspect preserved; neither
  dimension exceeds the request. **This is the default** when the type slot is
  empty.
- **`fill`** — scale to _cover_ the box, then crop the overflow.
- **`force`** — stretch to exactly `w`×`h`, ignoring aspect ratio.
- **`auto`** — `fill` when source and target share orientation, else `fit`.

> ~~Every resize type crops.~~ **Corrected by
> [#59](https://github.com/vaam-apps/image-resizer/issues/59)/[#1](https://github.com/vaam-apps/image-resizer/issues/1):**
> the service used to crop regardless of the requested type, so
> `rs:fit:800:600` on a 1920×1080 source returned 800×600. It now returns
> 800×450. **A deployment upgrading across that change gets different bytes
> for the same URL** — `docs/about/changelog.md` lists it as a breaking change.

## Content negotiation — only for `.auto`

`src/modules/negotiation.rs`. Every explicit extension passes through
untouched; only the `.auto` extension consults `Accept`.

Preference order **AVIF > WebP > JPEG**, weighted by the highest matching `q`
(exact type, `image/*`, or `*/*`). Ties — including both defaulting to `1.0` —
go to AVIF. `q=0` is an explicit rejection, not a default. No `Accept` header,
an unparseable one, or one accepting neither AVIF nor WebP falls back to JPEG,
still marked negotiated.

**`Vary: Accept` is stamped only on a negotiated response**, because only a
`.auto` request's `Location` actually depends on the header.

`.auto` is opt-in per URL and stays that way deliberately: AVIF encode costs
roughly **105x** a JPEG encode at this service's own defaults
([ADR 0005](https://github.com/vaam-apps/image-resizer/blob/main/adr/0005-avif-measurement-libavif-mozjpeg.md)).
Making it the service-wide default is not justified on those numbers.

## Presets and the option allowlist

`src/modules/url/presets.rs`, configured by `PRESETS` and
`ALLOWED_PROCESSING_OPTIONS`.

- A preset containing a `pr:` segment is **rejected at config-load time** — no
  recursion.
- A preset named `default` is **prepended to every request**, whether or not
  the caller writes `pr:`.
- Segments apply **in order**, so an explicit option after `pr:` overrides the
  preset's value for that field.
- **Only directly-present segments are checked against
  `ALLOWED_PROCESSING_OPTIONS`.** A preset's expansion is not re-checked. That
  is the point: it lets an operator hand out a restricted set of presets while
  forbidding the raw options they are built from.
- Unset or empty `ALLOWED_PROCESSING_OPTIONS` means unrestricted — distinct
  from an explicitly empty set, which would allow nothing.

## Responses

| Status                    | When                                                             |
| ------------------------- | ---------------------------------------------------------------- |
| `301 Moved Permanently`   | Success. `Location: /api/images/files/{key}`                     |
| `400 Bad Request`         | Malformed path; source too large; decode failure; SSRF rejection |
| `403 Forbidden`           | Missing, malformed, or wrong signature; `unsigned` when disabled |
| `502 Bad Gateway`         | The origin fetch failed                                          |
| `503 Service Unavailable` | A concurrency limit or the request timeout is shedding load      |
| `304 Not Modified`        | `If-None-Match` matched on the download route                    |

**It is `301`, hand-built with `StatusCode::MOVED_PERMANENTLY`.**
`axum::response::Redirect::permanent` issues `308`; the wire contract and its
tests are specifically `301`. Do not "simplify" that call site.

Every error response carries **`Cache-Control: no-store`**. Before
[#41](https://github.com/vaam-apps/image-resizer/issues/41) every failure was
swallowed into a bare `200 OK` with an empty body — indistinguishable from a
real empty image to both a CDN and a client checking `response.ok`.

## The second route

`GET /api/images/files/{key}` is unsigned by design: it serves only
content-addressed bytes this service already produced, and the key is
validated before any backend is touched (`emgr-security`). It derives
`Content-Type` from the key's own extension — the generated router it replaced
hardcoded `image/png` for every format, so a downloaded `.jpg` was served with
a lying content type.
