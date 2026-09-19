# Versioning

> **A skill is true of an emgr, not of emgr.**

This is the failure mode this page exists to prevent:

> An agent loads `emgr-config`, reads that a deployment refuses to start
> without `SIGNING_KEY`/`SIGNING_SALT`, and writes a runbook around it. The
> claim is true of `main`. The operator is pinned to a commit from before
> [#27](https://github.com/vaam-apps/image-resizer/issues/27), where an
> unsigned deployment starts happily. Nothing in the skill said when it became
> true, so nothing warned anyone.

The inverse is just as bad and harder to spot: a skill that still describes
something the current emgr has removed, which an agent then faithfully
reproduces. emgr has real examples of both — `make init` and the generated
`openapi.yaml` router were load-bearing until [#53](https://github.com/vaam-apps/image-resizer/issues/53)
deleted them, and `BENCHMARK_TEST_URLS` was a real variable until the benchmark
stopped fetching over the network.

## What an emgr "version" actually is

emgr **does** cut releases — `release-please` versions the crate and
`.github/workflows/build.yml` publishes per-commit and floating Docker tags. But
a skill describes the **tree**, not a release, and the tree moves between tags.
A skill correct at one release can be wrong three merges later without any
version number changing.

So the identity a skill carries is **a commit and a date**. That is not a
workaround; it is the convention the source repository already writes in, in
dated sentences like "this page previously documented variables that don't
exist" and "PNG's number above is the one that used to read as 1.71 ms".

**Follow that convention here.** It is the whole mechanism.

## The three rules

### 1. Every `SKILL.md` names the emgr it was verified against

Directly under the title:

```markdown
> **Verified against emgr `446c4df6` (2026-09-19).** Version-sensitive claims
> below carry the date they became true. On an older or newer emgr, trust the
> repository over this page.
```

`coverage.json`'s `baseline` block carries the same ref in machine-readable
form, and `tools/verify-coverage.mjs` prints how far the checkout you gave it
has drifted.

**A stamp may be newer than the baseline; it may never be older or unrelated.**
Re-verifying one skill against a later emgr and stamping just that one is
correct and expected — the gate checks only that the stamped commit _contains_
the baseline. Requiring all nine stamps to move together would make a one-skill
correction cost a full re-verification pass, which is how you get nine
rubber-stamps.

The baseline also governs coverage. A claim on a **directory** covers the pages
that existed when the claim was made; a page added under a claimed directory
_since_ the baseline fails the gate by name, because inheriting the parent's
claim would hide exactly the case the gate exists to catch.

### 2. A version-sensitive claim carries the date it became true

Not "the AVIF encoder is libavif" but "**libavif/AOM since
[ADR 0005](https://github.com/vaam-apps/image-resizer/blob/main/adr/0005-avif-measurement-libavif-mozjpeg.md);
it was `ravif`/`rav1e` before**".

A claim is version-sensitive if a reader on a six-week-old checkout would be
misled by it. In practice that is most claims about:

| Kind of claim       | Write it as                                             |
| ------------------- | ------------------------------------------------------- |
| An env var exists   | "`<NAME>`, added in `#<issue>`"                         |
| A default changed   | "`<N>` — it was `<M>` until `#<issue>`"                 |
| A startup check     | "fails closed since `#<issue>`; before that it started" |
| A codec was swapped | "`<new>` since `#<issue>`; `<old>` before"              |
| A measured number   | "`<N>` ms on `<fixture>`, measured `<date>`"            |
| A count of anything | "N as of `<date>`", never a bare N                      |

The cost of the date is six characters. The cost of omitting it is an agent
confidently generating a config against a surface that does not exist on the
tree it is editing — and for a service whose whole configuration surface is
environment variables, that is a deployment that will not boot.

### 3. Say what it was before

When you correct a skill because emgr changed, **strike the old claim through
and date the correction** rather than overwriting it:

```markdown
~~PNG encode costs 1.71 ms.~~ **Corrected:** that measured the `image` crate's
default `CompressionType::Fast`, which production never uses. The production
path builds an explicit `CompressionType::Best` encoder: **98.93 ms**, ~56x
more expensive on the same fixture.
```

This is the source repository's own house style — its `docs/about/changelog.md`
and `.bench-baseline/BASELINE.md` both do it — and it is not sentimentality. It
tells a reader two things they cannot get any other way: which sentences on the
page have been looked at recently, and what the plausible-but-wrong belief was
— usually the one they were about to form.

## Releases

This repository tags a release whenever a batch of skills is re-verified against
a newer emgr. A tag names the **date of verification and the emgr commit it was
verified against**:

```text
v2026-09-19-446c4df6
```

`CHANGELOG.md` records, per release: the emgr range covered, which skills
changed, and — most importantly — **any claim that stopped being true**, so
someone upgrading can find the thing that will break them.

## Installing, upgrading, and what actually pins you

```bash
npx skills add https://github.com/vaam-apps/emgr-skills --skill emgr
npx skills update                 # re-fetch every installed skill
npx skills update emgr emgr-config
npx skills ls                     # what is installed, and from where
npx skills experimental_install   # restore a checkout from skills-lock.json
```

**`add` and `update` both fetch the default branch.** There is no `--ref` or
`--tag` on either, so a tag in this repository is a _human_ reference point —
something to read `CHANGELOG.md` against — not something the installer can
resolve.

**What pins you is the lockfile, not the command.** `skills-lock.json` records a
`computedHash` of the exact content installed, and a project keeps that content
until someone runs `update`. So:

- **Commit `skills-lock.json`.** It is the only record of which briefing your
  agents are actually running.
- **`experimental_install` is the reproducible path** — a fresh clone or a CI job
  restores exactly what the lockfile names.
- **Do not hand-edit an installed skill.** The hash makes a local edit read as
  drift, and the next `update` silently overwrites it.

### Upgrading deliberately

`update` is a re-fetch, not a merge: it takes whatever `main` says now. Three
things to do before running it, in descending order of how much they matter:

1. **Read `CHANGELOG.md` between your lockfile's release and now**, specifically
   the entries naming a claim that **stopped being true**. A new claim is
   additive; a retired one is what breaks an integration written against the old
   page.
2. **Check how far your emgr has drifted.** Run the gate against your own
   checkout — it reports the distance from the baseline in commits and dates:

   ```text
   baseline: these skills were verified against emgr 446c4df6 (2026-09-19);
   this checkout is 1a2b3c4d (2026-08-02) — 0 commit(s) newer,
   47 commit(s) it does not have.
   ```

   Run against an **older** emgr the gate will legitimately fail on paths that do
   not exist there yet. That is not a bug; it is the tool telling you these
   skills are newer than your tree.

3. **If you are pinned to an old emgr, do not silently take the latest skills.**
   That is precisely the case where a skill will confidently describe an option
   code or an env var your tree does not accept. Check the tag whose emgr commit
   is nearest your own and read forward from there.

### Downgrading

There is no `--ref`, so the honest answer is: check out the tag of this
repository whose emgr commit is nearest yours, and `npx skills add` from that
local path. Rare enough that it has not been made a first-class flow — if you
find yourself doing it often, the real fix is re-verifying a batch of skills
against your emgr and tagging that.

## What the gate can and cannot tell you

`tools/verify-coverage.mjs` checks that every emgr documentation page — a page
in the mkdocs tree, an ADR, a Helm chart, or a `.roo/rules` file — is claimed,
and that every claimed path exists in the checkout you point it at. Run against
an **older** emgr it will legitimately fail on paths that do not exist yet.

**It cannot read prose.** It cannot tell you that a sentence about an option
code is true. Only a dated claim and a reader can do that.
