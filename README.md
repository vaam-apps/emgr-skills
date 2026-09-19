# emgr-skills

Agent skills for [EmgR](https://github.com/vaam-apps/image-resizer) — an
imgproxy-compatible, on-the-fly image resizing service in Rust.

**The project is `emgr`; the repository is `image-resizer`.** Both names are
correct and neither substitutes for the other in a path.

EmgR is small but unusually sharp-edged. A plain `cargo build` produces a
binary that refuses to start. Two independent checks fail the **process** at
startup rather than the request at runtime. An `axum` helper that looks like
the right one issues `308` where the wire contract is `301`. Its own README
states that the service is roughly 3.48x slower than imgproxy on a cold cache
and declines to spin the warm-cache number as a speed win. An agent dropped
into that repository without context does not fail slowly — it produces
something plausible, goes green, and ships a control that is not there.

These skills are the context. Install the one that matches the work:

```bash
npx skills add https://github.com/vaam-apps/emgr-skills --skill emgr
```

Start with `emgr`. It is the orientation skill and it routes to the rest.
Adding more later is the same command with a different `--skill`;
`--skill '*'` takes all nine.

## Upgrading

```bash
npx skills update                    # every installed skill, from every source
npx skills update emgr emgr-config   # just these
npx skills ls                        # what is installed, and from where
```

`update` (alias `upgrade`) re-fetches from the default branch and rewrites the
`computedHash` in `skills-lock.json`. **Commit that lockfile** — it, not the
install command, is what pins you: a project keeps the exact content it
installed until someone runs `update`. `npx skills experimental_install`
restores a checkout from the lockfile, which is what a fresh clone or a CI job
wants.

**Before you upgrade, read [CHANGELOG.md](CHANGELOG.md)** — specifically the
entries naming a claim that **stopped being true**, which is the thing that
will break an integration written against the old page.

**Do not upgrade blindly if you are pinned to an older emgr.** These skills
track `main`. Pulling the latest ones onto a six-week-old checkout is exactly
the case where a skill will confidently describe an option code or an
environment variable your tree does not accept.
[VERSIONING.md](VERSIONING.md) has the full policy.

**Do not hand-edit an installed skill.** The lockfile hashes it, so a local
edit reads as drift rather than as an intended change — and the next `update`
silently overwrites it. Send a PR here instead.

## The skills

| Skill                                                | Load it when                                                          |
| ---------------------------------------------------- | --------------------------------------------------------------------- |
| [`emgr`](skills/emgr/)                               | Anything in the repo. Orientation, the startup traps, the map         |
| [`emgr-url-api`](skills/emgr-url-api/)               | The signed-path grammar, option codes, negotiation, presets, statuses |
| [`emgr-security`](skills/emgr-security/)             | Signing, `/metrics` auth, SSRF, cache-key validation                  |
| [`emgr-image-pipeline`](skills/emgr-image-pipeline/) | Codecs, formats, quality constants, the ADRs, licence obligations     |
| [`emgr-storage-cache`](skills/emgr-storage-cache/)   | Backends, the cache key, ETags, atomicity, error mapping              |
| [`emgr-config`](skills/emgr-config/)                 | Any environment variable, the profiles, cgroups, the drift gate       |
| [`emgr-performance`](skills/emgr-performance/)       | Concurrency, single-flight, the benchmarks and the bench gate         |
| [`emgr-ops`](skills/emgr-ops/)                       | Docker, Helm, compose, health, tracing                                |
| [`emgr-tooling`](skills/emgr-tooling/)               | CI, tests, cargo-deny, release-please, commit messages                |

## Why a separate repository

Two reasons, and the second is the load-bearing one.

A skill is **installed, not cloned**. `npx skills add` fetches one directory
into `.agents/skills/` and pins its hash in `skills-lock.json`. That works for
any project that _calls_ emgr — someone building signed URLs from a CDN edge
worker gets `emgr-url-api` and `emgr-security` without vendoring the whole
tree.

And skills must be able to **move at a different speed from the code**. A
skill is not documentation-of-record; it is a briefing, and a briefing that has
to clear the full CI gate to be corrected is a briefing nobody corrects.

The cost of that separation is drift, and drift is what the gate below exists
to refuse.

## The parity rule

> **Every feature lands in three places or it has not landed: the code, the
> docs, and the skills.**

A document that lags is worse than none, because people trust it. A skill that
lags is worse still, because an agent does not merely trust it — it **acts** on
it, at machine speed, across every session that loads it.

So this repository ships a gate, and it fails in **both** directions:

```bash
node tools/verify-coverage.mjs /path/to/image-resizer
```

- **docs → skills.** An emgr documentation page — a page in the mkdocs tree
  under `docs/`, an `adr/*.md`, a `helm/*/Chart.yaml`, or a `.roo/rules/*.md`
  — that no skill claims fails the gate. That is the half that catches a
  feature shipping with no briefing.
- **skills → docs.** A path claimed in `coverage.json` that no longer exists
  in image-resizer fails the gate. That is the half that catches a skill still
  describing something that moved or was deleted.

A one-directional gate rots in the direction nobody looks.

`coverage.json` is the map. An entry earns its place by prose that actually
covers the path — **the gate checks the claim exists; a reviewer checks it is
true.** Green means "every documentation page is claimed by some skill and no
skill cites a dead path". It cannot read prose, so it does **not** mean the
claiming skill says anything true about that page.

CI runs the gate against image-resizer's `main` daily and on every push, so a
merge there that outruns this repository shows up here as a red build rather
than as a confidently wrong agent three weeks later. The checkout is
full-depth, deliberately — the gate asks git _when_ a page was added.

## Versioning — read this before you trust a skill

> **A skill is true of _an_ emgr, not of emgr.** A feature on `main` may be
> absent from the tree you are editing.

Every `SKILL.md` is stamped, under its title, with the emgr commit it was
verified against:

> **Verified against emgr `446c4df6` (2026-09-19).**

That stamp is machine-enforced — the gate refuses a skill that carries none, or
one whose stamp is not the baseline or a descendant of it. And
`verify-coverage` reports how far the checkout you point it at has drifted.

Inside the prose, the mechanism is **dated claims** and struck-through
corrections, matching the house style image-resizer's own
`docs/about/changelog.md` and `.bench-baseline/BASELINE.md` already use.

Full policy: [VERSIONING.md](VERSIONING.md). What changed between releases,
including any claim that **stopped** being true: [CHANGELOG.md](CHANGELOG.md).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). The short version: a skill is judged on
whether an agent that read it does the right thing, not on whether it is
complete. **Prefer the caveat over the tour**, and date anything that could go
stale.

## Licence

MIT, matching image-resizer.
