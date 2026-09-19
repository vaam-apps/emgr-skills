# Contributing

A skill is judged on **whether an agent that read it does the right thing**, not
on whether it is complete.

## What a good skill looks like here

**Prefer the caveat over the tour.** An agent can read the code. What it cannot
recover from the code is the trap: that `axum::response::Redirect::permanent`
issues a `308` rather than the `301` this service's tests pin, that the
`in_memory` storage backend is `cfg(test)`-gated so a release build refuses it
even with the feature on, that a cache-key collision was once reachable because
fields were concatenated without length prefixes. Those sentences are the
product.

**Say what is not built, and name it.** emgr's own documents are unusually
willing to do this — the README states plainly that emgr is roughly 3.48x slower
than imgproxy on a cold cache, and `docs/configuration/performance.md` says
`CPU_THREAD_POOL_SIZE` is read, stored, and has no effect. Match that. A skill
that describes a knob without saying it does nothing is worse than no skill.

**Date anything that could go stale.** See [VERSIONING.md](VERSIONING.md).

**Correct, do not overwrite.** Strike the old claim through and date the
correction. The wrong belief is usually the one the reader was about to form.

**Quote the description.** The YAML frontmatter is parsed by a real YAML parser
in `npx skills add`, and an unquoted value containing `": "` is a nested mapping
— the installer **skips the skill outright**. The gate refuses that, but wrap it
anyway.

## The shape

```text
skills/<name>/
  SKILL.md              # frontmatter, the version stamp, then the prose
  references/*.md       # detail pages, each LINKED from SKILL.md
```

- `name:` in the frontmatter **must equal the directory name** — that is what
  `--skill` resolves.
- `description:` is the **only** thing an agent reads when deciding whether to
  load the skill. Say what it covers **and** when to reach for it. Under 80
  characters fails the gate.
- Every `references/*.md` must be linked from `SKILL.md`, and every link must
  resolve. An unreferenced reference page is a page no agent will ever open.
- Keep `SKILL.md` readable in one sitting. Push detail into `references/`.

## The gate

```bash
node tools/verify-coverage.mjs /path/to/image-resizer
```

It fails in both directions, and it also checks frontmatter validity, the
description length, the version stamp, and reference linkage.

**Run it before opening a PR**, and run it against the image-resizer checkout
you actually verified against. It needs a **full-depth** clone, not a shallow
one — it asks git when a page was added.

## Adding coverage for a new emgr feature

1. Read the feature — the code, and its page in `docs/`.
2. Fold it into the **owning** skill's prose. Resist adding a skill: nine is
   already a routing decision an agent has to make, and a routing table that is
   too fine is a routing table nobody follows.
3. Add the docs page (or its directory) **and** the source paths to that
   skill's `covers` in `coverage.json`.
4. Re-stamp that skill with the emgr commit you verified against.
5. Run the gate.

**Adding a path to `covers` without writing the prose passes the gate and
defeats the point.** The gate checks the claim exists; you are the part that
checks it is true.

## Changing a claim

If emgr changed under a skill, the change here is not "edit the sentence". It is:

- correct the sentence, struck through and dated;
- re-stamp the skill;
- add a `CHANGELOG.md` entry **naming the claim that stopped being true**, which
  is the entry someone upgrading actually needs.

## Formatting

Prettier, 80 columns, `proseWrap: preserve`:

```bash
npx --yes prettier@3 --check "**/*.{md,json,yml}"
```

## Commit messages

Conventional Commits, enforced org-wide by the `pr-title` workflow on the squash
title. One extra rule worth knowing, because it fails silently rather than
loudly: **a commit-message body line must not begin with `identifier(` containing
nested parentheses.** release-please parses bodies with a strict PEG grammar that
reads such a line as a type-and-scope header, and a nested `(` inside it is a
syntax error that makes it discard the **whole commit** — no changelog entry, no
version bump, nothing red anywhere. Put a word in front of it, or reword.

## Where the source of truth lives

**image-resizer, always.** When this repository and the source repository
disagree, the source repository is right and this repository has a bug. Fix it
here, and check whether emgr's own docs carried the same wrong claim — a skill's
error is sometimes a faithful copy of a document that was already wrong, not an
error of its own. emgr has a CI job (`check_env_docs.py`) for exactly that class
of drift on its environment variables; it has no equivalent for anything else.
