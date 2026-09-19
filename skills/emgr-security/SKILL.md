---
name: emgr-security
description: "The four controls that stop EmgR being an open image proxy — HMAC-SHA256 signed URLs and the unsigned escape, bearer-token /metrics auth, the SSRF source guard with its disguised-IP and DNS-rebinding defences, and cache-key validation. Load before touching signing, the source fetcher, the download route, an allowlist, or anything that decides whether a request is permitted; and before changing an error message on any of those paths."
---

# emgr-security

> **Verified against emgr `446c4df6` (2026-09-19).** Version-sensitive claims
> below carry the date they became true. On an older or newer emgr, trust the
> repository over this page. See
> [VERSIONING.md](https://github.com/vaam-apps/emgr-skills/blob/main/VERSIONING.md).

Four independent controls. Two fail the **process** closed at startup; two
fail the **request** closed at runtime.

## 1. Signed URLs (`src/modules/signing/`)

```
signature = base64url_nopad(HMAC-SHA256(key, salt || signed_path))
```

`salt` and `signed_path` are concatenated with **no separator**, matching
imgproxy exactly. `signed_path` is the request path as received — still
percent-encoded, leading `/` included — with the signature segment stripped.
`SIGNING_KEY` and `SIGNING_SALT` are **hex-encoded**; an odd-length or
non-hex value is a startup failure, not a runtime one.

`verify_signature` decodes the presented value, compares lengths in the clear
(documented as not a secret leak), then uses `subtle::ConstantTimeEq`. **Never
replace that with `==`.**

`verify.rs` pins the README's own worked example as a test — key
`6d792d7369676e696e672d6b6579`, salt `6d792d73616c74`, a known path, and the
expected signature — so the documentation cannot silently drift from what the
module computes.

### Three ways a request is refused, with three different messages

`verify_or_reject` (`src/modules/api/resize.rs`), all `403 Forbidden`:

1. Signature segment is literally `unsigned` and `ALLOW_UNSIGNED_REQUESTS` is
   not `true` → names the env var in the message.
2. Not `unsigned`, but no key is configured → "no signing key is configured".
3. The HMAC does not match → "invalid signature".

**It runs before `SignedRequest::parse`.** That ordering is deliberate: an
unauthenticated caller cannot use parse-error content as an oracle for the
grammar, and malformed unsigned spam does not pay for a full parse. Keep it.

**`403`, not `401`** — there is no `WWW-Authenticate` challenge to offer, and
imgproxy does the same.

## 2. `/metrics` authentication (`src/modules/metrics_auth/`, `otel` only)

Bearer token, compared with `subtle::ConstantTimeEq`, same shape as signing.
Startup fails unless `METRICS_AUTH_TOKEN` is set or
`ALLOW_UNAUTHENTICATED_METRICS=true`
([#77](https://github.com/vaam-apps/image-resizer/issues/77)).

Two details that are easy to get backwards:

- **An empty or whitespace-only token counts as unset.** It is malformed, not
  a secret. Treating it as configured would make every request fail `401` with
  a token nothing could ever match, instead of failing loudly at startup.
- **This one answers `401`, not `403`**, with
  `WWW-Authenticate: Bearer realm="metrics"` — the deliberate opposite of the
  signing path, because here there genuinely is a credential to negotiate.

`/health` is unauthenticated **on purpose**, not by oversight: Kubernetes
probes cannot carry a bearer token, and the handler returns the literal string
`OK` and nothing else.

## 3. The SSRF source guard (`src/services/image/source_guard.rs`)

This module guards SSRF **only**. Size and resolution limits are elsewhere
(`emgr-config`).

- **Schemes**: `http`/`https` only.
- **Blocked ranges**, IPv4: loopback `127.0.0.0/8`, link-local
  `169.254.0.0/16`, RFC1918, `0.0.0.0`, `255.255.255.255`, CGNAT
  `100.64.0.0/10`. IPv6: `::1`, `::`, `fe80::/10`, `fc00::/7`, and
  IPv4-mapped addresses unwrapped and re-checked.
- **Disguised literals are decoded before checking**: decimal
  (`http://2130706433/`), octal (`0177.0.0.1`), hex (`0x7f.0.0.1`), and short
  `inet_aton` forms (`127.1`).
- **DNS rebinding**: the host is resolved once and the HTTP client is pinned
  to that exact `SocketAddr`.
- **Every redirect hop is re-validated**, not just the original URL
  ([#21](https://github.com/vaam-apps/image-resizer/issues/21)).

### `ALLOWED_SOURCES` is narrower than it looks, deliberately

A match lifts the **private-range** block for that host only
([#57](https://github.com/vaam-apps/image-resizer/issues/57) — before that
fix, the guard silently overrode the allowlist, so listing a private origin
authorised nothing). It does **not** lift loopback or link-local, which keep
their own separate opt-ins (`ALLOW_LOOPBACK_SOURCE_ADDRESSES`,
`ALLOW_LINK_LOCAL_SOURCE_ADDRESSES`, both default `false`). **The cloud
metadata endpoint `169.254.169.254` therefore stays unreachable even from an
allowlisted origin's redirect.**

Matching is structural — scheme, host, port and path prefix compared on the
parsed `Url` — **not** a raw string `starts_with`, which would be defeated by
userinfo spoofing (`https://allowed.example@evil.test/`) and subdomain-boundary
spoofing (`https://allowed.example.evil.test/`).

## 4. Cache-key validation (`src/services/storage/key_validation.rs`)

The download route takes `{key}` from an untrusted path and joins it onto a
backend root. Unvalidated, that is an arbitrary file read on `local_fs`
(`PathBuf::join` **silently discards the base** when the argument is absolute)
and an IDOR across the whole bucket on S3
([#23](https://github.com/vaam-apps/image-resizer/issues/23)).

A valid key is exactly `{STORAGE_SUB_PATH prefix}{64 lowercase hex}.{ext}`
where `ext` ∈ `jpg`, `png`, `webp`, `avif`, `gif`. `auto` is deliberately
excluded — it is a negotiation input, never a stored extension. Control bytes
and NUL anywhere are rejected first.

**`InvalidKeyError`'s `Display` reads "Image not found in storage", not
"invalid key".** A malformed key and a well-formed absent key produce the
identical response, so nothing hands back an oracle distinguishing "that shape
is invalid" from "that key does not exist". **Do not "improve" that message.**

Validation runs in `StorageService` before any backend call, so every backend
inherits it rather than each reimplementing it.

## Where this is proven

`tests/storage_key_validation.rs` plants a real secret file outside the
storage root and asserts absolute paths, `../` traversal, percent-encoded and
decoded traversal forms, wrong-length and uppercase hashes, double extensions
and NUL smuggling are all refused — through the public `StorageService` API,
not the validator in isolation. It is compile-gated on `local_fs` and needs no
Docker.

## Supply chain

`deny.toml` currently has **no active `[advisories] ignore` entries**, and
both historical ones were deleted with an explanation rather than left as dead
config — the `paste` advisory stopped applying when `ravif`/`rav1e` left the
graph, and four `h2`/`rustls-webpki` advisories stopped applying when
`aws-sdk-s3`'s misleadingly-named `rustls` feature (which wires the _legacy_
hyper-0.14 connector, not the modern one) was dropped
([#56](https://github.com/vaam-apps/image-resizer/issues/56)).

CI runs `cargo-deny` with `--all-features`, and that is load-bearing rather
than lazy: `rustls` enters the graph **only** through the `s3` feature, so a
narrower invocation would make its advisories invisible.
