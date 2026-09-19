# Reading the repo

Conventions that will mislead you if you do not know them, and the pages that
are known to disagree with the code.

## There is no `AGENTS.md`

Unlike its sibling repositories, image-resizer carries no `AGENTS.md` and no
`CLAUDE.md`. Its agent instructions are two short files under `.roo/rules/`:

- `1-commit-message.md` — conventional commits; scan all changes before writing
  one; use the body as a list.
- `2-rust.md` — run `cargo check` before committing; write or update tests for
  files you edit; propose optimisations; **edition 2024**.

That is the whole of it. Everything else an agent needs is in `README.md`,
`docs/`, and the ADRs. Do not go looking for a long engineering record; the
equivalent material here is distributed across `docs/about/changelog.md`,
`.bench-baseline/BASELINE.md`, and the doc comments in the source, which are
unusually dense and worth reading directly.

## A superseded ADR stays in the tree

`adr/0004-avif-measurement.md` opens with **⚠️ SUPERSEDED** and states that
every number in it describes encoders the service no longer ships. It names the
two figures most likely to be quoted from it anyway (AVIF/JPEG `0.7241x`, AVIF
encode `367.8ms`) as wrong, and points at `adr/0005` for the current ones.

**Read an ADR's header before quoting it.** `0001`'s own decision — "drop WebP
as an output format and adopt AVIF via `ravif`" — is likewise no longer what the
code does: WebP output exists, and AVIF goes through libavif/AOM, not `ravif`.

## Corrections are written in place, not silently patched

`.bench-baseline/BASELINE.md` has a "PNG encode correction" section explaining
that the table's PNG figure used to read `1.71 ms` because the bench measured
the `image` crate's default `CompressionType::Fast`, which production never
uses; the production path builds `CompressionType::Best`, at `98.93 ms` — about
56x more. The README reproduces the correction rather than just the new number.

Follow the same style when you correct a skill.

## Known doc-vs-code disagreements as of `446c4df6` (2026-09-19)

These were each checked against the source, not inferred.

### `PERFORMANCE_OPTIMIZATIONS.md` is stale on `ENABLE_HTTP2`

It states that the no-profile construction path "defaults an unset
`ENABLE_HTTP2` to `false`, not `true`", and that a deployment setting neither
variable "therefore runs with HTTP/2 off".

**That is no longer true.** `src/config/performance.rs`'s no-profile branch
reads `enable_http2: env_config.enable_http2.unwrap_or(true)`, with a comment
naming [#83](https://github.com/vaam-apps/image-resizer/issues/83) and saying
explicitly that the ordinary path must not diverge from `Default::default()`.
`docs/configuration/performance.md` was updated to match and marks #83 fixed;
`PERFORMANCE_OPTIMIZATIONS.md` still carries the pre-fix text, including a
pointer to "the same discrepancy documented against the full settings table" on
a page that no longer documents it.

**Trust the code and `docs/configuration/performance.md`.**

### `docs/about/changelog.md` is stale on versioning

It says emgr "doesn't cut dated, tagged releases yet", that `Cargo.toml`'s
version "has stayed `0.1.2` throughout", and that every published image is a
floating or per-commit tag.

`Cargo.toml` is at `0.2.1` (carrying an `# x-release-please-version` marker),
`.github/workflows/release-please.yml` exists and maintains a standing release
PR, and `.release-please-manifest.json` is present. The page's own instruction
— switch to dated version headings once real releases start — has come due.

### `CPU_THREAD_POOL_SIZE` is documented as inert, and that is accurate

Worth listing here because it reads like drift and is not.
`docs/configuration/performance.md` says the variable is read, stored, exposed
via `get_cpu_thread_pool_size()`, and that nothing calls that getter. Checked:
the only other reference to the field is a `debug!` log line in `main.rs`.
Setting it changes nothing. `MAX_CONCURRENT_PROCESSING` is the variable that
actually bounds CPU concurrency.

### Two modules were reconstructed, and say so

`src/modules/negotiation.rs` and `src/modules/url/presets.rs` both open with a
note that the patch which introduced their callers did not include the module
file, and that the implementation was rebuilt from call-site and test evidence
afterwards. Treat their behaviour as derived-from-tests rather than
derived-from-spec, and check the tests before assuming an imgproxy-compatible
edge case is handled.

## Every `#NN` issue link in the repository is a dead 404

Nearly every issue reference in the source and docs is written as a full
`https://github.com/vaam-store/image-resizer/issues/NN` URL — 13 files carry
one — and `vaam-store` is the repository's **previous** owner. It is
`vaam-apps/image-resizer` now.

The rename does **not** save these links, and the difference is worth knowing
precisely, because the repository-root case looks fine and hides the rest
(measured 2026-09-19):

| URL                                             | Result                                       |
| ----------------------------------------------- | -------------------------------------------- |
| `github.com/vaam-store/image-resizer`           | `301` → `github.com/vaam-apps/image-resizer` |
| `github.com/vaam-store/image-resizer/issues/25` | **`404`** — no redirect                      |
| `github.com/vaam-apps/image-resizer/issues/25`  | `200`                                        |

So the issue numbers themselves are accurate and the issues exist; only the
owner in the URL is wrong. **When you cite one of these in a skill, rewrite the
owner to `vaam-apps`** rather than copying the link out of the source. Do not
read a `vaam-store` URL as evidence of a different repository — and do not
treat one as a working link either.
