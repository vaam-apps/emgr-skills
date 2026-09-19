# Changelog

Per release: the emgr range covered, which skills changed, and — most
importantly — **any claim that stopped being true**, which is the entry someone
upgrading actually needs.

## v2026-09-19-446c4df6 — initial release

Verified against emgr
[`446c4df6`](https://github.com/vaam-apps/image-resizer/commit/446c4df61669fc1c05fbeef7ee4d0cf6c7957ead)
(2026-09-19).

Nine skills, covering the service as of that commit: orientation and the
startup traps, the URL grammar and wire contract, the four security controls,
the codec pipeline, storage and the cache key, the full environment-variable
surface, concurrency and benchmarks, deployment, and CI/tooling.

**No claims retired**, there being no previous release. The claims most likely
to go stale first, and therefore the ones to re-verify soonest:

- **Counts and constants.** `CACHE_KEY_VERSION = 11`, `DEFAULT_AVIF_QUALITY =
80`, `DEFAULT_AVIF_SPEED = 6`, `DEFAULT_WEBP_QUALITY = 82.0`,
  `SHARD_PREFIX_LEN = 2`, `HASH_LEN = 64`, the 15% bench threshold and its
  100 µs floor, 23 documentation pages, nine skills. Every one is dated; none
  is stable.
- **Every default in `emgr-config`.** The whole configuration surface is
  environment variables, so a changed default is a changed deployment.
- **The measured numbers.** `adr/0005`'s medians, the criterion table, and the
  3.48x / 2.86x / ~0.39 ms vs ~21 ms imgproxy comparison. Benchmarks move with
  the codecs.
- **"Not supported" claims.** HEIC, `fq:png`, `webpo`'s and `jpgo`'s omitted
  slots, gravity's `re`/`ch` watermark positions. Each becomes wrong the moment
  it is built.
- **`CPU_THREAD_POOL_SIZE` has no effect.** True as of this commit; it stops
  being true the moment anything calls `get_cpu_thread_pool_size()`.
- **The latent AVIF `imageSizeLimit` bug** (`MAX_SRC_RESOLUTION_MP` above ~268
  breaks all AVIF decoding). Recorded in `adr/0005` as a follow-up not taken.
- **`clippy` is non-blocking in CI.** If
  [#46](https://github.com/vaam-apps/image-resizer/issues/46) is closed, that
  sentence in `emgr-tooling` is wrong.

### Three doc-vs-code disagreements this release records rather than inherits

These are stated in `skills/emgr/references/reading-the-repo.md` and
cross-referenced from the skills that would otherwise repeat them. They are
facts about image-resizer at `446c4df6`, not about these skills:

1. **`PERFORMANCE_OPTIMIZATIONS.md` is stale on `ENABLE_HTTP2`.** It says the
   no-profile path defaults it to `false`; the code reads `.unwrap_or(true)`,
   and `docs/configuration/performance.md` was updated to match.
   [#83](https://github.com/vaam-apps/image-resizer/issues/83) fixed one page
   and not the other.
2. **`docs/about/changelog.md` is stale on versioning.** It says emgr does not
   cut tagged releases and that `Cargo.toml` "has stayed `0.1.2`";
   `Cargo.toml` is `0.2.1` with a release-please marker and the workflow
   exists.
3. **Every `#NN` issue URL in the repository is a dead 404.** They are written
   against `vaam-store/image-resizer`, the previous owner. Measured
   2026-09-19: the repository root `301`s to `vaam-apps`, but
   `…/vaam-store/image-resizer/issues/25` returns `404` with no redirect while
   the `vaam-apps` path returns `200`. The issue numbers are accurate; only
   the owner in the URL is wrong. **Skills here rewrite the owner rather than
   copying the link.**

### Notes on this repository's own gate

`tools/verify-coverage.mjs` is adapted from `vaam-apps/vpay-skills` and
`vaam-apps/vsms-skills`. Four things differ, each commented at the point of
divergence in the file itself:

- **The sentinel is not `AGENTS.md`.** image-resizer has neither `AGENTS.md`
  nor `CLAUDE.md`; its agent rules live in `.roo/rules/`. The gate checks for
  a `Cargo.toml` declaring `name = "emgr"` instead — the manifest alone would
  match any Rust repository on the machine.
- **There is no `docs/flows/`.** vpay's gate walks a feature index that, per
  vsms-skills' own changelog, never existed in vsms and had to be replaced
  there. Nothing of the sort exists here either. The four surfaces walked are
  real and were enumerated: the mkdocs tree, `adr/`, `helm/*/Chart.yaml`, and
  `.roo/rules/`.
- **`ancestors()` is generalised.** vsms hardcodes
  `backends/crates|apps/<name>`. Here a directory claim is valid for any
  directory strictly below a surface root, so `docs/architecture` is a claim
  and bare `docs` is not — the same rule vsms arrived at by hand when it
  removed two blanket directory entries, expressed once instead of
  per-surface.
- **The `fetch-depth: 0` comment is inherited, not rediscovered.**
  vpay-skills measured that failure for real after `vaam-apps/vpay#182`
  merged; the reasoning is carried over with attribution rather than
  presented as this repository's own incident.

The gate was proven to fail before it was trusted — removing a `covers` entry
named the uncovered page; renaming a claimed path named the dead claim;
deleting a version stamp named the skill.
