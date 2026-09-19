---
name: emgr-performance
description: "How EmgR sheds load and how fast it actually is — three independent concurrency bounds and the distinct errors each produces, the single-flight leader/follower registry and what happens when a leader panics, the honest imgproxy comparison (emgr is slower cold, faster warm, and the README refuses to spin it), the criterion suite and the 15%-threshold CI regression gate. Load before changing a concurrency limit, touching the resize orchestrator, or quoting a benchmark number."
---

# emgr-performance

> **Verified against emgr `446c4df6` (2026-09-19).** Version-sensitive claims
> below carry the date they became true. On an older or newer emgr, trust the
> repository over this page. See
> [VERSIONING.md](https://github.com/vaam-apps/emgr-skills/blob/main/VERSIONING.md).

## Start here: emgr is slower than imgproxy, and that is the documented position

**Cold cache, against imgproxy v4.0.13: roughly 3.48x slower on p50 and about
2.86x less throughput.** The README states this in those words and declines to
present anything else. The ratio moves with storage backend and format mix,
but has stayed in the 3–4x (p50) / 2.6–2.9x (throughput) band across every
measured configuration.

**Warm cache: ~0.39 ms vs imgproxy's ~21 ms.** This is **not** a
processing-speed win and must never be quoted as one. imgproxy has no result
cache and reprocesses every request; emgr redirects a repeat request to its own
storage before the pipeline runs. The cold cost and the warm win are the _same
trade-off_ seen from two sides — emgr's path is process → store → `301` →
client re-fetches, which is exactly why a miss costs an extra round trip and a
hit is nearly free. You cannot remove one without losing the other, and which
matters depends on a production cache-hit rate these benchmarks do not
measure.

Keep that register. Overstating this service's speed is the specific failure
its own documentation is written to avoid.

## Three independent concurrency bounds

They are separate on purpose, and produce distinguishable failures.

| Bound                       | Default   | Where                                              | On exhaustion                                           |
| --------------------------- | --------- | -------------------------------------------------- | ------------------------------------------------------- |
| `MAX_CONCURRENT_REQUESTS`   | 512       | Router middleware, `try_acquire_owned`             | immediate `503`, `Cache-Control: no-store`              |
| `max_concurrent_downloads`  | 20        | `ImageService::download_image`                     | **queues** — a blocking `.acquire().await`, no shedding |
| `max_concurrent_processing` | CPU count | `ImageService::process_image`, `try_acquire_owned` | immediate `503`                                         |

**Note the asymmetry**: downloads queue, processing sheds. The two `503`s
reach the client through `classify_resize_error`'s substring match on
`"permit"` — so **an error message on either path must keep that word**, or
the response silently becomes a `502`.

`REQUEST_TIMEOUT_SECS` (30) wraps the handler in `tokio::time::timeout`, also
`503`. All four values go through `env_positive`, which treats `0` as unset so
an override cannot wedge the router.

The saturation and timeout layers are hand-rolled rather than
`tower::limit::ConcurrencyLimitLayer` + `LoadShedLayer` +
`tower_http::timeout::TimeoutLayer` because none of those `tower`/`tower-http`
features are enabled in `Cargo.toml`. The layer order, outer to inner: CORS →
rate limiter → conditional-download (`304`) → saturation+timeout →
compression. A `304` therefore resolves **without** taking a permit.

The governor rate limiter's per-key state never shrinks on its own, so a
background task calls `retain_recent()` every 60s.

## Single-flight (`src/services/resize/handler.rs`)

Concurrent requests for the **same cache key** are coalesced: a
`DashMap<String, broadcast::Sender<InFlightResult>>` keyed by cache key. The
first caller becomes leader; the rest subscribe and await the broadcast. No
`.await` happens while the shard lock is held.

**The part that matters is failure.** `InFlightGuard`'s `Drop` always runs —
on panic-unwind and on future cancellation alike — removes the map entry, and
if `finish` was never called sends a synthetic
`"…was cancelled or panicked before completing"` error. So followers get an
error instead of hanging forever. `futures::future::Shared` and `OnceCell`
were both considered and rejected precisely because neither signals "the
original future was dropped without completing".

Both properties are tested: 100 concurrent callers produce exactly one origin
fetch and one upload; and against a dead origin, 20 followers all resolve
within 5s with the map cleaned up afterwards.

Note this is **above** the storage layer, which is why
`local_fs_handler` still treats concurrent writers to one key as expected.

## Micro-benchmarks

criterion, `cargo bench --features local_fs`. Figures below are
darwin/arm64 on a synthetic fixture — real photographs measure differently,
often _faster_ for the same operation, which is why `benches/fixtures.rs`
provides both kinds.

| Operation                                             |     Time |
| ----------------------------------------------------- | -------: |
| JPEG decode, 1920×1080                                |  7.32 ms |
| JPEG encode (baseline)                                |   930 µs |
| JPEG encode (progressive)                             | 17.58 ms |
| PNG encode (production path, `CompressionType::Best`) | 98.93 ms |
| WebP encode                                           | 23.66 ms |
| WebP decode, 1920×1080 (libwebp)                      | 32.27 ms |
| AVIF encode (`DEFAULT_AVIF_SPEED = 6`)                | 65.89 ms |
| AVIF decode, 1920×1080 (dav1d)                        | 55.13 ms |
| Resize, downscale, Lanczos3                           |  3.43 ms |
| Resize, downscale, Triangle→Bilinear                  |  1.15 ms |
| Full pipeline, photo → thumbnail JPEG                 |  6.15 ms |
| Full pipeline, 4K → large downscale                   | 19.59 ms |

> ~~PNG encode costs 1.71 ms.~~ **Corrected:** that measured the `image`
> crate's default `CompressionType::Fast`. Production builds an explicit
> `CompressionType::Best` — about **56x** more on the same fixture. The bench
> file now carries the trap in a comment; `.bench-baseline/BASELINE.md` has
> the full story in its "PNG encode correction" section.
>
> The AVIF/JPEG **encode** figures above come from criterion on a synthetic
> fixture; `adr/0005`'s medians (119.2 ms / 1.14 ms) come from the Kodak
> corpus. They measure different things — do not reconcile them by picking
> one.

## The CI regression gate

`.github/scripts/bench_gate.py`, run only on pull requests, against a
`main` baseline persisted by a separate push-to-main job.

- Compares `mean.point_estimate` from
  `target/criterion/<id>/{new,main}/estimates.json`.
- **Fails when `pct_change > 15` _and_ the new measurement is at or above the
  floor.** The floor is `--floor-ns`, default **100 µs**: anything faster is
  reported but can never fail the build, because sub-100µs criterion noise is
  not signal.
- A bench with no baseline is listed with no delta, never counted as a
  regression.
- The report is posted as a PR comment whether or not it passes; a separate
  step turns a non-zero exit into a failed job.

## Where the measured record lives

- **`.bench-baseline/BASELINE.md`** — every run, backend and `FORMATS`
  configuration the headline ratios are drawn from, plus the traps that
  produced misleading intermediate readings: encoder-profile defaults,
  redirect-blended metrics, DCT-scale thresholds, the PNG compression level.
- **`.bench-baseline/P0-COMPARISON.md`** — the before/after for the first
  optimisation wave.
- **`bench-imgproxy/`** — the three-way end-to-end harness (k6 driver, nginx
  origin, compose stack) behind the cold/warm numbers.
- **`PERFORMANCE_OPTIMIZATIONS.md`** — the mechanism-level writeup. **Its
  `ENABLE_HTTP2` paragraph is stale**; see `emgr-config`.

## Build tuning that was silently doing nothing

`.cargo/config.toml` carries the AVX2/FMA `rustflags`. It used to live in
`Cargo.toml`, where Cargo ignored it with `warning: unused manifest key` — so
the optimisation this project documented was **never applied to any build**
([#28](https://github.com/vaam-apps/image-resizer/issues/28)).

It is deliberately **not** `target-cpu=native`: that tunes for whichever
machine ran the build and can `SIGILL` on an older CPU in a distributed image.

The `benchmark` binary no longer fetches over the network — `BENCHMARK_TEST_URLS`
is gone and in-process fixtures replace it, so a run needs no network at all.
Redirects are still followed end to end, to keep the shape realistic.
