---
name: emgr
description: "Orientation for working in EmgR (the vaam-apps/image-resizer repository) — an imgproxy-compatible, on-the-fly image resizing service in Rust (Axum + Tokio). Load this before any task in the repo: it carries the two things that fail closed at startup, the compile-time storage-feature trap that makes a plain cargo build produce a binary that refuses to run, the request flow, the repository map, and which of the other emgr-* skills to load for the work at hand. Use when reading, planning, reviewing or changing anything in image-resizer."
---

# emgr

> **Verified against emgr `446c4df6` (2026-09-19).** Version-sensitive claims
> below carry the date they became true — a feature on `main` may be absent
> from the tree you are editing. On an older or newer emgr, trust the
> repository over this page. See
> [VERSIONING.md](https://github.com/vaam-apps/emgr-skills/blob/main/VERSIONING.md).

**The project is `emgr`. The repository is `image-resizer`.** `Cargo.toml`
declares `name = "emgr"`, the Helm chart is `helm/emgr`, and the README opens
`# EmgR — Image Resizing Service`. Both names are correct; they are not
interchangeable in a path.

EmgR fetches a source image over HTTP(S), decodes/resizes/encodes it per an
**HMAC-signed URL**, caches the result to a storage backend, and **301-redirects
the caller** to the cached bytes.

## The three things that surprise everyone, all deliberate

### 1. A plain `cargo build` produces a binary that will not start

`Cargo.toml` sets `default = []`. Storage backends are **compile-time Cargo
features**, not runtime configuration:

```bash
cargo build --features local_fs     # or: s3, or "local_fs otel", or "s3 otel"
```

With none of `local_fs`/`s3`/`in_memory` enabled, `StorageService::determine_storage_type`
returns `Err("No storage features are enabled")` before anything else runs.
`otel` is **not** a storage backend — it enables OpenTelemetry export and mounts
an authenticated `/metrics`, independently of the other three.

### 2. Two independent checks fail the process **at startup**, not at first request

Both live in `ApiService::create` (`src/modules/api/handler.rs`) and both
propagate out of `main`:

- **Signing.** `SigningConfig::from_env` `bail!`s unless `SIGNING_KEY` **and**
  `SIGNING_SALT` are set and hex-decodable, **or** `ALLOW_UNSIGNED_REQUESTS=true`.
  Signed URLs are the default, not opt-in ([#27](https://github.com/vaam-apps/image-resizer/issues/27)).
- **Metrics auth** (`otel` builds only). `MetricsAuthConfig::from_env` `bail!`s
  unless `METRICS_AUTH_TOKEN` is set (non-blank) **or**
  `ALLOW_UNAUTHENTICATED_METRICS=true` ([#77](https://github.com/vaam-apps/image-resizer/issues/77)).

A third, smaller one: an unrecognised `PERFORMANCE_PROFILE` value also `bail!`s
rather than silently falling through to defaults. Both are **breaking changes
for deployments that predate them** — see `docs/about/changelog.md`'s own
"Breaking changes for existing deployments" section.

### 3. `in_memory` storage does not exist in a release build

It is gated `#[cfg(all(test, feature = "in_memory"))]` — the module, the match
arm, and the constructor all three. Enabling the feature in a release build and
setting `STORAGE_TYPE=IN_MEMORY` falls through to a catch-all `Err`, not to an
unbounded in-process cache. That is deliberate
([#39](https://github.com/vaam-apps/image-resizer/issues/39)): the backend has
no entry cap, no byte cap and no LRU.

## The request flow, in one paragraph

`GET /{signature}/{processing_options}/{source}.{ext}` →
`verify_or_reject` (403 on a bad/missing signature, **before** the URL grammar
is parsed) → `ResizeService::resize` → cache check → on a miss, a single-flight
leader downloads (bounded by `download_semaphore`), processes on
`spawn_blocking` (bounded by `processing_semaphore`), uploads, and broadcasts to
followers → **`301` to `/api/images/files/{key}`** → the client re-fetches that
unsigned, content-addressed path.

**The `Location` never points at the caller-supplied source URL.** That was an
open redirect from a trusted domain
([#25](https://github.com/vaam-apps/image-resizer/issues/25)), and because a
`301` is cached permanently by browsers regardless of `Cache-Control`, a
transient origin failure would have steered that client away from the resizer
forever.

## Traps that have each cost this project real time

- **`axum::response::Redirect::permanent` issues `308`, not `301`.** The
  redirect is hand-built with an explicit `StatusCode::MOVED_PERMANENTLY`
  because the wire contract and its tests are specifically `301`.
- **`rustflags` is a Cargo _config_ key, never a manifest key.** It sat in
  `Cargo.toml` for a while, where Cargo ignored it with `warning: unused
manifest key` — so the AVX2/FMA tuning this project documented was never
  applied to any build
  ([#28](https://github.com/vaam-apps/image-resizer/issues/28)). It lives in
  `.cargo/config.toml` now, and **not** as `target-cpu=native`, which tunes for
  the build machine and can `SIGILL` on an older CPU.
- **A build-only pipeline cannot catch a startup failure.** The `s3`/`s3_otel`
  images shipped for an unknown period completely unable to start — a glibc
  mismatch between a trixie builder and a bookworm runtime — with CI green
  throughout, because nothing ever ran an image
  ([#62](https://github.com/vaam-apps/image-resizer/issues/62)). `build.yml`
  now runs each image before pushing it.
- **`cargo clippy` is non-blocking in CI** (`continue-on-error: true`,
  [#46](https://github.com/vaam-apps/image-resizer/issues/46)), and there is a
  standing backlog of warnings behind it. Green CI does not mean clippy-clean.
- **`PERFORMANCE_OPTIMIZATIONS.md` and `docs/configuration/performance.md`
  disagree about `ENABLE_HTTP2`'s no-profile default.** The code agrees with the
  docs page (`true`); see `emgr-config`.

## Repository map

| Path                                              | What is in it                                                                                                                                |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/modules/`                                    | HTTP-facing: `router`, `api` (handlers), `url` (the signed-path grammar), `signing`, `metrics_auth`, `env`, `negotiation`, `tracer`, `utils` |
| `src/services/`                                   | The work: `image` (codecs), `resize` (orchestration + single-flight), `storage`, `cache`, `health`, `metrics`                                |
| `src/config/`                                     | `PerformanceConfig` and the three `PERFORMANCE_PROFILE` presets                                                                              |
| `src/bin/`                                        | `healthcheck` (the container `HEALTHCHECK`) and `benchmark`                                                                                  |
| `docs/`                                           | The mkdocs tree, published to GitHub Pages. 14 pages as of `446c4df6`                                                                        |
| `adr/`                                            | Five decision records. **`0004` is superseded by `0005`** and says so in its own header                                                      |
| `benches/`, `bench-imgproxy/`, `.bench-baseline/` | criterion micro-benchmarks, the three-way imgproxy harness, and the measured record                                                          |
| `helm/emgr`, `helm/serverless`                    | Deployment chart and Knative scale-to-zero chart                                                                                             |
| `.roo/rules/`                                     | The repo's agent rules. **There is no `AGENTS.md` and no `CLAUDE.md` here**                                                                  |

## Where the honest record lives

This repository is unusually willing to say what it got wrong, and those
sentences are worth more than the tour:

- **`docs/about/changelog.md`** — grouped by theme, with every breaking change
  named. It also states that releases were untagged when written; the crate is
  on release-please now.
- **`.bench-baseline/BASELINE.md`** — every measured number, and the traps that
  produced misleading intermediate readings (the PNG compression-level trap, the
  JPEG encoder-profile default, redirect-blended metrics).
- **`adr/0004`** — kept in the tree marked superseded, with its numbers voided,
  rather than deleted.

The README does the same at the top level: it states plainly that emgr is
**roughly 3.48x slower on p50 than imgproxy v4.0.13 on a cold cache** and
refuses to present the warm-cache number as a processing-speed win. Match that
register.

## Which skill to load

| The work                                                        | Load                  |
| --------------------------------------------------------------- | --------------------- |
| The URL grammar, processing options, response codes, presets    | `emgr-url-api`        |
| Signing, `/metrics` auth, SSRF, cache-key validation            | `emgr-security`       |
| Codecs, formats, resize kernels, the ADRs, what is unsupported  | `emgr-image-pipeline` |
| Storage backends, the cache key, ETags, atomicity               | `emgr-storage-cache`  |
| Any environment variable, the profiles, cgroups, the drift gate | `emgr-config`         |
| Concurrency, single-flight, the benchmarks and the bench gate   | `emgr-performance`    |
| Docker, Helm, compose, health, tracing, `/metrics` plumbing     | `emgr-ops`            |
| CI, the Makefile, cargo-deny, release-please, tests             | `emgr-tooling`        |

`references/reading-the-repo.md` covers the documentation conventions —
including which pages are known to disagree with the code.
