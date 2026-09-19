---
name: emgr-config
description: "Every EmgR environment variable — what it does, its default, whether it is required, and which three cause the process to refuse to start. Covers the three PERFORMANCE_PROFILE presets and how per-variable overrides interact with them, cgroup-aware CPU detection, the CI job that fails when the code and the configuration page disagree, and the one documented variable that has no effect. Load before adding, renaming or documenting any env var, or when a deployment will not boot."
---

# emgr-config

> **Verified against emgr `446c4df6` (2026-09-19).** Version-sensitive claims
> below carry the date they became true. On an older or newer emgr, trust the
> repository over this page. See
> [VERSIONING.md](https://github.com/vaam-apps/emgr-skills/blob/main/VERSIONING.md).

**The service is configured entirely by environment variables.** There is no
config file. `src/modules/env/env.rs` is the single declaration point —
`envconfig` derive, one field per variable — and
`docs/getting-started/configuration.md` is the reference.

## The drift gate: change one, change both

`.github/scripts/check_env_docs.py` runs as CI's `docs-env-drift` job
([#47](https://github.com/vaam-apps/image-resizer/issues/47)) and **fails in
both directions**:

- it extracts every `envconfig(from = "NAME")` from `env.rs`;
- it extracts every backtick-wrapped `ALL_CAPS` token of 3+ characters from
  `docs/getting-started/configuration.md`, minus a small allowlist of
  `STORAGE_TYPE`'s accepted _values_ (`LOCAL_FS`, `MINIO`, `IN_MEMORY`, …);
- a name in the docs and not the code, or in the code and not the docs, exits
  `1`.

Two consequences worth knowing: fenced code blocks are **not** scanned (a
`KEY=value` example line is not backticked, so it pollutes neither set), and
the script is not feature-aware — a variable behind `#[cfg(feature = "s3")]`
must still be documented.

This gate exists because the page previously documented `CACHE_ENABLED`,
`CACHE_TTL_SECONDS`, `S3_BUCKET`, `S3_REGION` and `LOCAL_STORAGE_PATH` — none
of which have ever existed.

## The three that refuse to start

All three `bail!` out of `ApiService::create`, before the listener binds.

| Check                      | Refuses unless                                                                                    |
| -------------------------- | ------------------------------------------------------------------------------------------------- |
| Signing                    | `SIGNING_KEY` **and** `SIGNING_SALT` set and hex-decodable, **or** `ALLOW_UNSIGNED_REQUESTS=true` |
| Metrics auth (`otel` only) | `METRICS_AUTH_TOKEN` set and non-blank, **or** `ALLOW_UNAUTHENTICATED_METRICS=true`               |
| Performance profile        | `PERFORMANCE_PROFILE` is unset, empty, or one of the three valid names                            |

A blank or whitespace-only `METRICS_AUTH_TOKEN` counts as **unset** and
therefore fails at startup — rather than being treated as a token nothing can
match, which would produce a confusing `401` on every scrape instead.

The profile check fails closed for the same stated reason:
`PERFORMANCE_PROFILE=hgh_throughput` silently yielding different behaviour
than the operator asked for is exactly the surprise the convention exists to
prevent ([#83](https://github.com/vaam-apps/image-resizer/issues/83)).

## The variables

### Server and storage

| Variable                                          | Default                                                       |
| ------------------------------------------------- | ------------------------------------------------------------- |
| `HOST`                                            | `0.0.0.0`                                                     |
| `PORT`                                            | `3000`                                                        |
| `STORAGE_TYPE`                                    | unset — only consulted when >1 backend feature is compiled in |
| `STORAGE_SUB_PATH`                                | `""` — becomes the cache-key prefix                           |
| `CDN_BASE_URL`                                    | `http://localhost:9000/image-cache`                           |
| `LOCAL_FS_STORAGE_PATH`                           | `./data/images` (feature `local_fs`)                          |
| `MINIO_ENDPOINT_URL`                              | `http://localhost:9000` (feature `s3`)                        |
| `MINIO_ACCESS_KEY_ID` / `MINIO_SECRET_ACCESS_KEY` | `minioadmin` / `minioadmin`                                   |
| `MINIO_BUCKET`                                    | `image-cache`                                                 |
| `MINIO_REGION`                                    | `us-east-1`                                                   |

### Security

`SIGNING_KEY`, `SIGNING_SALT` (hex, no default), `ALLOW_UNSIGNED_REQUESTS`
(`false`), `METRICS_AUTH_TOKEN`, `ALLOW_UNAUTHENTICATED_METRICS` (`false`),
`ALLOWED_SOURCES` (unset), `ALLOW_LOOPBACK_SOURCE_ADDRESSES` (`false`),
`ALLOW_LINK_LOCAL_SOURCE_ADDRESSES` (`false`), `MAX_REDIRECTS` (`5`). See
`emgr-security` for what each actually gates.

### Limits and decode bombs

| Variable                       | Default | Guards                                                                                                 |
| ------------------------------ | ------- | ------------------------------------------------------------------------------------------------------ |
| `MAX_IMAGE_SIZE_MB`            | `50`    | Download size, enforced by streaming — `Content-Length` is only a cheap early check and is not trusted |
| `MAX_SRC_RESOLUTION_MP`        | `50`    | Source megapixels, from header dimensions before full decode                                           |
| `MAX_OUTPUT_WIDTH` / `_HEIGHT` | `4096`  | Output dimensions                                                                                      |
| `MAX_ANIMATION_FRAMES`         | `512`   | Many-tiny-frame animation bombs                                                                        |

> **Do not raise `MAX_SRC_RESOLUTION_MP` above ~268 without reading
> `emgr-image-pipeline` first.** The value is passed straight to libavif's
> `imageSizeLimit`, which rejects anything above its own 16384×16384 default —
> so a higher setting breaks **all** AVIF decoding with an opaque error.

### Router-level shedding (`src/modules/router/middlewares.rs`)

| Variable                  | Default                           |
| ------------------------- | --------------------------------- |
| `REQUEST_TIMEOUT_SECS`    | `30`                              |
| `MAX_CONCURRENT_REQUESTS` | `512`                             |
| `RATE_LIMIT_BURST`        | `20`                              |
| `RATE_LIMIT_PERIOD_MS`    | `100` (10 req/s sustained per IP) |

All four go through `env_positive`, which **discards `0` as if unset** — so a
`0` override cannot permanently saturate the router into `503`s.

### Pipeline concurrency and HTTP client

`MAX_CONCURRENT_DOWNLOADS` (`20`), `MAX_CONCURRENT_PROCESSING`
(`effective_cpu_count()`), `HTTP_TIMEOUT_SECS` (`30`), `ENABLE_HTTP2`
(`true`), `CONNECTION_POOL_SIZE` (`50`), `KEEP_ALIVE_TIMEOUT_SECS` (`60`),
`CPU_THREAD_POOL_SIZE` (see below), `PERFORMANCE_PROFILE` (unset).

### Options, watermarks, JPEG

`PRESETS`, `ALLOWED_PROCESSING_OPTIONS`, `WATERMARK_URL` (all unset),
`JPEG_PROGRESSIVE` (`false`), `JPEG_NO_SUBSAMPLING` (`false`).

### Observability (`otel`)

`LOG_LEVEL` (`debug`), `OTLP_SPAN_ENDPOINT` (`http://localhost:4317`),
`OTLP_METRIC_ENDPOINT` (`http://localhost:4318/v1/metrics`),
`OTLP_SERVICE_NAME` (`rust-app-example`).

### Runtime

`TOKIO_WORKER_THREADS` (falls back to `effective_cpu_count()`),
`SHUTDOWN_TIMEOUT_SECS` (`20`).

## `CPU_THREAD_POOL_SIZE` does nothing

It is read, stored on `PerformanceConfig`, and exposed via
`get_cpu_thread_pool_size()` — **which nothing calls.** The only other
reference to the field is a `debug!` log line in `main.rs`. CPU-bound work
runs on Tokio's blocking pool via `spawn_blocking`, bounded by
`MAX_CONCURRENT_PROCESSING`'s semaphore, not a separately-sized pool.

Setting it is harmless and has no effect. `docs/configuration/performance.md`
says so too, and is accurate.

## The three profiles

`PERFORMANCE_PROFILE` ∈ `high_throughput`, `low_latency`, `memory_efficient`
(case-insensitive). Each is a starting point; **individually-set variables
still override it per field**.

|                             | default | high_throughput | low_latency | memory_efficient |
| --------------------------- | ------: | --------------: | ----------: | ---------------: |
| `max_concurrent_downloads`  |      20 |              50 |          10 |                5 |
| `max_concurrent_processing` |    cpus |        cpus × 2 |        cpus |         cpus ÷ 2 |
| `http_timeout`              |     30s |             15s |         10s |              45s |
| `max_image_size`            |   50 MB |          100 MB |       20 MB |            10 MB |
| `enable_http2`              |    true |            true |        true |        **false** |
| `connection_pool_size`      |      50 |             100 |          25 |               10 |
| `keep_alive_timeout`        |     60s |            120s |         30s |              30s |
| `max_src_resolution_mp`     |      50 |              50 |          50 |           **25** |
| `max_output_width/height`   |    4096 |            4096 |        4096 |         **2048** |
| `max_animation_frames`      |     512 |             512 |         512 |          **128** |

`memory_efficient` is the only profile that tightens the safety limits, and
the only one that turns HTTP/2 off (HTTP/1.1 uses less memory).

## `ENABLE_HTTP2` — one doc page is stale

The no-profile path resolves an unset `ENABLE_HTTP2` to **`true`**
(`env_config.enable_http2.unwrap_or(true)`), matching `Default::default()`.
This was [#83](https://github.com/vaam-apps/image-resizer/issues/83), and
`docs/configuration/performance.md` marks it fixed.

~~`PERFORMANCE_OPTIMIZATIONS.md` says the no-profile path defaults it to
`false`, so a deployment setting neither variable runs with HTTP/2 off.~~
**That page was not updated with the fix.** Trust the code.

## CPU counts come from the cgroup, not the host

`effective_cpu_count()` (`src/modules/utils/cgroup.rs`) reads cgroup v2
`/sys/fs/cgroup/cpu.max`, falling back to v1's
`cpu.cfs_quota_us`/`cpu.cfs_period_us`, and clamps the result to
`[1, num_cpus::get()]`. Unreadable or unlimited (`max`, `-1`) falls back to
the host count.

The reason is stated plainly in the module: `num_cpus::get()` reads the CPU
**affinity mask**, not the cgroup **quota**, so a pod capped at `400m` on a
16-core node still reports 16 — any pool sized from it is about 40x
oversubscribed ([#44](https://github.com/vaam-apps/image-resizer/issues/44)).
A fractional quota rounds **up** (1.5 CPUs reserves 2) rather than truncating
and leaving half a CPU unused.
