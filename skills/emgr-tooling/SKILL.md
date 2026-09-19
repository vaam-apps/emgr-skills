---
name: emgr-tooling
description: "Building, testing and shipping EmgR — the feature matrix cargo test actually runs, which CI jobs gate a merge and which only annotate (clippy does not gate), the two separate commit-message checks and why one exists, release-please and the chart-version trap, cargo-deny, and the .roo agent rules. Load before running tests, adding a CI job, writing a commit message, or wondering why a green build still has warnings."
---

# emgr-tooling

> **Verified against emgr `446c4df6` (2026-09-19).** Version-sensitive claims
> below carry the date they became true. On an older or newer emgr, trust the
> repository over this page. See
> [VERSIONING.md](https://github.com/vaam-apps/emgr-skills/blob/main/VERSIONING.md).

## The repo's own agent rules

`.roo/rules/` — **there is no `AGENTS.md` or `CLAUDE.md` here**:

- `1-commit-message.md` — conventional commits; scan **all** changes before
  writing one; use the body as a list.
- `2-rust.md` — run `cargo check` before committing; write or update tests for
  files you edit; propose optimisations; **edition 2024**.

## Building and testing

```bash
cargo build --features local_fs          # at least one storage feature, always
cargo test --no-default-features --features local_fs --all-targets
cargo bench --features local_fs
```

CI's `test` job runs three isolated feature sets: **`local_fs`**, **`s3`**,
**`local_fs,otel`** — each with `--no-default-features`. Nothing in
`Cargo.toml` makes the storage backends mutually exclusive; that they are
chosen one at a time is a convention CI encodes, not a constraint the type
system enforces.

**`cargo test` needs no external service.** Not MinIO, not Docker.
`tests/storage_s3_handler.rs` drives the real AWS SDK against an in-process
axum fake; `src/modules/api/resize.rs`'s tests spin up a one-shot in-process
HTTP origin. `docs/development/testing.md`'s MinIO section is guidance for
exercising the backend by hand.

Integration tests: `storage_key_validation.rs`, `storage_local_fs_atomicity.rs`,
`storage_s3_handler.rs` (each compile-gated on its feature) and
`fixtures_smoke.rs` (ungated — it pins the deterministic bench fixtures'
byte-identity, so a fixture drift cannot silently move every benchmark).

## What gates a merge, and what does not

| Job                       | Workflow            | Gates?                                                          |
| ------------------------- | ------------------- | --------------------------------------------------------------- |
| `test` (×3)               | `ci.yml`            | **yes**                                                         |
| `helm-verify`             | `ci.yml`            | **yes** — `dependency build`, `lint`, `template` on both charts |
| `cargo-deny`              | `ci.yml`            | **yes**                                                         |
| `docs-env-drift`          | `ci.yml`            | **yes**                                                         |
| `bench` (PR only)         | `ci.yml`            | **yes** — 15% threshold, see `emgr-performance`                 |
| `clippy`                  | `ci.yml`            | **no — `continue-on-error: true`**                              |
| `lint` (super-linter)     | `lint-codebase.yml` | yes, for YAML/Actions/gitleaks/hadolint/merge markers           |
| `conventional-title`      | `pr-title.yml`      | **yes**                                                         |
| `commit-message-parses`   | `pr-title.yml`      | **yes**                                                         |
| `sast` / `lint` / `trivy` | `quality.yml`       | reusable org workflows                                          |

> **A green CI does not mean clippy-clean.** The `clippy` job is explicitly
> named non-blocking and carries a standing backlog
> ([#46](https://github.com/vaam-apps/image-resizer/issues/46)) — dead_code,
> collapsible_if, manual_saturating_arithmetic, doc-list indentation and
> naming lints. Do not add to it, and do not assume a warning you see is new.

> **super-linter's `VALIDATE_RUST_CLIPPY` is deliberately off.** Every run of
> it failed: super-linter v7.4.0 bundles Cargo 1.83.0, which cannot parse
> `edition = "2024"` at all. Real clippy lives in `ci.yml` only.

`cargo-deny` runs with `--all-features`, and that is load-bearing rather than
lazy: `rustls` enters the dependency graph **only** through the `s3` feature,
so a narrower invocation would make its advisories invisible. `deny.toml` has
no active `ignore` entries — see `emgr-security`.

## Two commit-message checks, not one

- **`conventional-title`** regex-checks the PR title against
  `^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\(scope\))?!?: .+`.
  The title is passed through `env:`, never interpolated into a `run:` with
  `${{ }}` — a PR title is attacker-controlled text and `${{ }}` in a shell
  body is textual substitution.
- **`commit-message-parses`** runs the **whole squash message** through
  release-please's own parser (`@conventional-commits/parser`, pinned to
  `0.4.1` — the only version satisfying release-please 17.6.0's `^0.4.1`).

The second exists because the title is only line 1. **A body line can make
release-please discard the entire commit — silently, with the workflow
green**: its PEG grammar reads a line beginning `identifier(` as a
type-and-scope header, and a nested `(` inside it is a syntax error. Write a
body line starting with a word, not a function-call-shaped token.

## release-please

`release-type: simple`, driven by `release-please-config.json` and
`.release-please-manifest.json`. Every merge to `main` updates a standing
release PR; merging it cuts the tag.

**It mints a GitHub App token rather than using `GITHUB_TOKEN`, and that is
load-bearing.** GitHub raises no workflow events for anything done with the
default token, so with it the release PR would get **no CI run at all** and
the tag would trigger nothing.

When a release PR is created or updated, the workflow checks the branch out
and runs `cargo metadata` in every directory containing a `Cargo.lock` —
found with `git ls-files '*Cargo.lock'` rather than a hardcoded list —
committing the refreshed lockfile. The Dockerfile builds `--locked`, so a
release PR with a stale lockfile is red at image-build time.

**Only `appVersion` is managed in the Helm charts.** Tying either chart's own
`version:` to the app's release number would silently downgrade it the moment
the numbers diverge — and they already have (`0.1.9` and `0.1.3` against app
`0.2.1`). A human bumps `version:` when they actually change the chart.

## Other workflows

- **`build.yml`** — the four image flavours, with the run-before-push smoke
  test. See `emgr-ops`.
- **`deploy-docs.yml`** — `mkdocs build --clean` to `gh-pages`, then
  `chart-releaser` for `helm/`.
- **`issue-governance.yml`** — validates issue structure against the org
  templates, `enforce: true`.

## The Makefile is Docker-only

Every target wraps `docker compose -p emgr -f compose.yaml`: `pull`, `build`,
`up`, `up-app`, `start`, `stop`, `restart`, `down`, `destroy`, `logs`,
`logs-app`, `ps`, `stats`, `git-pull`, `help`.

**No `make` target runs `cargo test`, `cargo clippy` or `cargo bench.`** If
you are looking for the local equivalent of CI, it is the `cargo` commands at
the top of this page, not a `make` target.
