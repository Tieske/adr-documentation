# ADR Documentation — Process Design

This document defines how Architecture Decision Records (ADRs) are created,
numbered, reviewed, and maintained in this project.

## Purpose

ADRs capture significant technical decisions — the context, the decision
itself, the alternatives considered, and the consequences — so future
contributors understand *why* the system looks the way it does, not just
*what* it looks like.

## What warrants an ADR

Write an ADR for:

- Anything likely to be questioned later ("why don't we just use X instead?"), _when in doubt check with your team!_
- Technology or tool choices (database, framework, language, library)
- Architectural patterns (monolith vs. microservices, sync vs. async, event-driven design)
- Decisions with real trade-offs (performance vs. simplicity, build vs. buy)
- Decisions that are hard or expensive to reverse (data formats, API contracts, auth strategy)

Skip an ADR for routine implementation details, or anything trivially reversible.

## File location

All ADRs live in `docs/adr/`.

## Naming convention

```
NNNN-title-in-kebab-case.md
```

- `NNNN` — zero-padded sequential number (`0001`, `0002`, …)
- `title` — short, lowercase, hyphen-separated, phrased as the decision
  itself, not the topic/question. **Must be unique** across every ADR
  ever written, past or present — it's the ADR's real identity, not the
  number.

Example: `0003-use-jwt-for-authentication.md`, not
`0003-authentication-strategy.md`.

Numbers are **never reused or resequenced**. If a decision is reversed,
write a *new* ADR and mark the old one as superseded (see Front matter
below) — don't delete or renumber it.

## Front matter

Every ADR starts with a YAML front matter block, so the whole `docs/adr/`
directory can be machine-parsed without touching Markdown. This would allow
using it to generate overviews in for example `docs/adr/README.md` or a
static website (see Appendix: Automation opportunities — none of this is
built on day one).

```yaml
---
status: Proposed
proposed_date: YYYY-MM-DD
supersedes: []
superseded_by: []   # this one is not in the template, since it's added later
tags: []
---
```

- `status` — one of:

  | Status | Meaning |
  |---|---|
  | Proposed | Under discussion, not yet merged/accepted |
  | Accepted | Merged and in effect |
  | Superseded | Replaced by one or more later decisions (see `superseded_by`) |

  The `Proposed` state can only exist while a PR is open. Once merged it should be
  changed to `Accepted`.

  There's no standalone "Deprecated" status: if a decision is
  reversed, that reversal is itself a decision and gets its own ADR (see
  Workflow below), making the old one `Superseded`. If nothing formally
  reverses it, it's still `Accepted`, however outdated it feels.
- `proposed_date` — the date the ADR was proposed (not updated on later
  edits)
- `supersedes` — list of references (e.g. `[0042]`) to the ADR(s) this one
  replaces, empty if it doesn't reverse anything. **Author-filled,
  required.** A reference can be either the `NNNN` number or the `title`
  slug — both are unambiguous, since `title` is guaranteed unique (see
  Naming convention above), and a number may not be final yet if the
  referenced ADR is itself still mid-review. The schema deliberately
  doesn't constrain the format beyond "a string" for this reason; see
  Appendix: Automation opportunities for normalizing references to one
  fixed format later. It's a list rather than a single value to allow one
  ADR to replace several earlier ones at once.
- `superseded_by` — the inverse: list of references to the ADR(s) that
  replace *this* one, same reference format as `supersedes`. **Absent
  from every ADR at creation** — the block above is what a new ADR
  actually starts with, and a brand-new ADR can't yet have been superseded
  by anything. It only appears later, added to an *older* ADR by whoever
  opens the PR that supersedes it (see Workflow, step 3), and it always
  mirrors `status: Superseded`. Optional in the schema for exactly this
  reason.
- `tags` — freeform list of short labels (e.g. `[database, security]`) for
  filtering/grouping ADRs by area. Optional; leave as `[]` if none apply.
  There's no fixed tag vocabulary to maintain — pick whatever's useful at
  the time.

The filename and number stay immutable; only the front matter (`status`
and `superseded_by`, when a later ADR supersedes this one) changes over
time — `supersedes` itself is set once, at authoring time, and doesn't
change afterward. The `id`/number and `title` are deliberately *not*
duplicated into front matter — the filename is already their single
source of truth (see Naming convention above), and duplicating them would
just create another thing to keep in sync.

### Schema

[`adr-frontmatter.schema.json`](adr-frontmatter.schema.json) (JSON Schema
draft-04) captures the above machine-readably: `status`, `proposed_date`,
`supersedes`, and `tags` are required; `superseded_by` is the one optional
field, since it's meant to be filled in later, not by the ADR's own
author. `status` is constrained to its enum, `proposed_date` to
`YYYY-MM-DD`, and `supersedes`/`superseded_by`/`tags` to string arrays —
deliberately unpatterned for the first two, since a reference may be
either a number or a title slug (see Front matter above).
`additionalProperties: false` means a typo'd or stray field fails
validation instead of silently doing nothing. A CI check can validate each
ADR's front matter against it (see Appendix: Automation opportunities).

## Workflow

1. Copy `TEMPLATE.md` to `docs/adr/XXXX-your-decision-title.md`.
2. Fill it in; open a PR with `status: Proposed` in the front matter.
3. If this ADR supersedes an older one, list the old ADR's number in
   `supersedes`. In the *same PR*, manually update the old ADR: set its
   `status` to `Superseded` and add this ADR's number to its
   `superseded_by`. (Automating this update is tracked in Appendix:
   Automation opportunities — not part of the process today.)
4. Discuss in the PR. Update the ADR in place as the discussion clarifies
   the decision (the PR is the discussion record; the ADR is the outcome).
5. On merge:
   - CI/maintainer renames `XXXX-...` to the next sequential number.
   - `status` is updated to `Accepted`.


# Appendices 

## Appendix: Open questions / to decide

- [ ] Who has authority to merge (any maintainer, or a specific role)?
- [ ] Do we want to restrict the list of tags? (adding a tag can be in
      the same PR against the jsonschema, making it explicit and reviewable)
- [ ] Which CI tool/script enforces numbering + title-uniqueness
      (see Appendix: Automation opportunities)?
- [ ] Do we want an auto-generated index page, and if so, via what tool?
- [ ] Do we need linting? (see json-schema for example)

## Appendix: Automation opportunities

Ideas worth building eventually, tracked here rather than acted on now
since none of them ship on day one — this project starts lightweight and
manual, and automation gets added only once the manual version is felt to
be a real burden. When one of these gets built, move it out of this list
and into the relevant section above as the documented, current behavior.

- **Auto-populate `superseded_by`.** On merge (or on a schedule), scan
  every ADR's `supersedes` list and write the reverse mapping into the
  referenced ADRs' `superseded_by`, flipping their `status` to
  `Superseded` at the same time. This is what turns `superseded_by` from
  a manual step (Workflow, step 3) into a fully derived field.
- **CI schema validation.** Validate every ADR's front matter against
  `adr-frontmatter.schema.json` as a required check.
- **CI numbering/title enforcement.** Enforce, in CI, that `NNNN` and
  `title` are each never duplicated across `docs/adr/`, so a bad merge
  fails CI instead of being caught by hand.
- **Normalize `supersedes`/`superseded_by` references.** These fields
  currently accept either a `NNNN` number or a `title` slug (see Front
  matter above), since a number may not be final yet at authoring time.
  Once an ADR is merged and numbered, automation could rewrite all
  references to it — everywhere in `docs/adr/` — to the canonical `NNNN`
  form, so the repo converges on one consistent reference format over
  time without requiring authors to guess it upfront.
- **Auto-generated index.** Generate `docs/adr/README.md` (number, title,
  status, tags) directly from front matter, so the index can't drift from
  the ADRs it lists.
- **Static site.** A small generated site for browsing and searching ADRs
  by status, date, or tag — the fuller realization of what machine-parseable
  front matter is ultimately for (see Front matter above).
