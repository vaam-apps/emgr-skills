---
name: emgr-storage-cache
description: "EmgR's persistence layer — the StorageBackend trait and how a backend is chosen from Cargo features at compile time, the local-filesystem atomicity and sharding scheme, the S3 backend and its in-process fake, the SHA-256 cache key with its length-prefixed fields and version byte, ETag and conditional requests, and the AppError-to-status mapping. Load before adding a backend, changing anything that feeds the cache key, or debugging a cache hit that should have been a miss."
---

# emgr-storage-cache

> **Verified against emgr `446c4df6` (2026-09-19).** Version-sensitive claims
> below carry the date they became true. On an older or newer emgr, trust the
> repository over this page. See
> [VERSIONING.md](https://github.com/vaam-apps/emgr-skills/blob/main/VERSIONING.md).

## Backend selection happens at compile time, then at runtime

`StorageBackend` is an `#[async_trait]` trait (`src/services/storage/core.rs`)
with `upload_image`, `upload_image_with_ttl`, `check_cache`, `get_image`,
`delete`. `upload_image` is a provided method forwarding with `ttl = None`, so
adding TTL did not break existing callers
([#40](https://github.com/vaam-apps/image-resizer/issues/40)).

`StorageService` wraps one `Arc<dyn StorageBackend>`, chosen by
`determine_storage_type`:

- **Zero** storage features → `Err("No storage features are enabled")`. This is
  what a plain `cargo build` produces.
- **Exactly one** → used automatically, `STORAGE_TYPE` irrelevant.
- **More than one** → `STORAGE_TYPE` decides (`S3`|`MINIO`,
  `LOCAL_FS`|`LOCALFS`|`LOCAL`, `IN_MEMORY`|`INMEMORY`|`MEMORY`), else a
  priority order of `s3` > `local_fs` > `in_memory`.

**`in_memory` is `#[cfg(all(test, feature = "in_memory"))]`** — the module,
the match arm and the constructor. In a release build with the feature on, the
arm does not exist and `STORAGE_TYPE=IN_MEMORY` falls to a catch-all `Err`
rather than running an unbounded, lock-poisoning cache in production
([#39](https://github.com/vaam-apps/image-resizer/issues/39)). It has no entry
cap, no byte cap and no LRU, and that is why.

## Local filesystem

Path shape: `{prefix}/{first 2 hex chars of hash}/{hash}.{ext}` —
`SHARD_PREFIX_LEN = 2`, so one directory does not accumulate every object.

**Writes are atomic**: full write to a temp file **in the same directory**,
`sync_all()`, then `tokio::fs::rename`. A rename within one directory is
atomic on POSIX, so a concurrent reader sees either the previous complete file
or the new complete one, never a partial. The temp name carries PID, a
nanosecond timestamp and a monotonic counter; a failed write best-effort
removes it, and an orphan never shadows the final path.

This matters more than it looks because **there is no request coalescing at
this layer** — concurrent writers to the same key are expected, not rare.
(Coalescing exists one level up; see `emgr-performance`.)

The TTL sidecar (`<file>.expires`) uses the same stage-then-rename, written
only after the data rename succeeds, and **fails open in every error case** —
a missing, unreadable or malformed sidecar reads as "not expired", never as
"already expired", so a transient read error cannot evict a live entry.

Defence in depth on top of key validation: `.`/`..`/empty segments are
stripped, and both read and write canonicalise and assert the result is still
under the base path, which catches a planted symlink.

`tests/storage_local_fs_atomicity.rs` proves this with 12 concurrent writers
of a 256 KiB single-byte payload against one key while 4 readers poll: every
read is exactly `PAYLOAD_LEN` and entirely one writer's byte. It also covers a
directory planted at the key path being a **miss, not an error**.

## S3

`tests/storage_s3_handler.rs` drives the real `aws_sdk_s3::Client` against an
**in-process axum fake** bound to `127.0.0.1:0`. Both alternatives were
considered and rejected in the file's own doc comment: a trait-level double
would be hermetic but test nothing about `s3_handler.rs`, and a real MinIO
container would need Docker at `cargo test` time plus a new dependency. So
`cargo test --features s3` is fully hermetic. The fake does not validate
SigV4.

`docs/development/testing.md` describes running a real MinIO — that is guidance
for exercising the backend by hand, **not** something `cargo test` requires.

## The cache key

`CacheService::generate_key` (`src/services/cache/handler.rs`) →
`{STORAGE_SUB_PATH}{64 lowercase hex SHA-256}.{format}`, exactly the shape
`key_validation` enforces.

Two properties are load-bearing:

- **Every field is length-prefixed** (4-byte big-endian) before hashing.
  Without that, `w=1,h=23` and `w=12,h=3` both flatten to `123` — and worse,
  the **attacker-controlled URL absorbs a digit from the width field**, so
  `url=".../a.jpg1", w=2` and `url=".../a.jpg", w=12` collide. That is the
  security-relevant case: a caller who controls the URL could pick one landing
  on another request's key
  ([#24](https://github.com/vaam-apps/image-resizer/issues/24)). Regression
  tests exist for both.
- **A version byte leads the hash** — `CACHE_KEY_VERSION`, at **11** as of
  `446c4df6`, bumped whenever a change makes the same inputs produce different
  bytes.

Roughly thirty inputs feed it, including canonicalised blur (`<= 0` or NaN
collapse to the same value), every watermark field, padding, zoom, DPR,
min-width/height, and `strip_metadata`.

**Two of the bump decisions are documented in the constant's own history and
worth understanding before you add a field:**

- **v10 was mandatory**, not courtesy: `strip_metadata` became a hashed input
  when EXIF stripping landed, and every pre-v10 entry may hold EXIF —
  including GPS from a user upload — that a post-fix request would
  deliberately not produce.
- **#67 deliberately did _not_ bump**, and the reasoning is recorded rather
  than left to be re-derived: swapping the JPEG decoder changed output by a
  DSSIM of `<= 0.000014`, invisible, and flushing the whole cache to correct
  an invisible rounding difference is the worse trade. If a future change to
  that path is _perceptible_, the calculus flips.

## ETag and conditional requests

The ETag is **not** a hash of the response body. It is the cache key from the
URL path, quoted — a **strong** validator (no `W/`), which is sound because
`upload_image` writes each key exactly once and the response is served
immutable, so "same key" and "byte-identical response" are equivalent.

`src/modules/utils/etag.rs` implements only the RFC 7232 §3.2 rules for one
strong ETag: split on `,`, trim, match `*` or the value after stripping a
leading `W/`. A `304` is resolved by middleware **before** the concurrency
permit and the request timeout are taken.

## Error mapping (`src/modules/utils/err.rs`)

| Variant               | Status |
| --------------------- | ------ |
| `NotFound`            | 404    |
| `BadRequest`          | 400    |
| `Forbidden`           | 403    |
| `BadGateway`          | 502    |
| `ServiceUnavailable`  | 503    |
| `IoError`, `AnyError` | 500    |

Every error response sets **`Cache-Control: no-store`**. Two classifiers
downcast to a typed error first and only then fall back to string matching:
`classify_download_error` recognises `InvalidKeyError` → `NotFound`;
`classify_resize_error` recognises `SourceRejected` → `BadRequest`, then
matches `"permit"`/`"cancelled"` → 503, `"too large"`/`"decode"` → 400,
`"encode"` → 500, else 502.

**Those substrings are a real coupling.** A semaphore-exhaustion message must
keep the word `permit` in it or the response silently becomes a `502`. If you
reword one of those errors, check `err.rs` in the same change.

## Health

`src/services/health/handler.rs` returns the literal `"OK"` and checks
nothing — no storage probe, no dependency check. Pure liveness, deliberately
unauthenticated, and deliberately leaking no telemetry.
