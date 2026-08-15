# Contributing to the ecosystem

The `@interop/*` / BYOE repos run a deliberately asymmetric process with two
tiers. The boundary between them sits at merge time: everything
ecosystem-shaped (roadmaps, cross-repo impact walks, decision records,
learnings, releases) is bookkeeping that needs the whole-ecosystem view, and
it is done by core maintainers at or after merge (and so shouldn't be demanded
of a casual contributor at PR time).

## Tier 1: contributors

Your concern is per-repo and manageable. For a bug fix, a test, a doc
correction, or a focused feature:

- Build and test locally; match the repo's code style (its CONTRIBUTING.md).
- Open a PR with tests and a one- or two-line summary of what changed and
  why.

That is the whole list. You do not need to:

- file or reference a roadmap item;
- fill in a `touches:` impact list;
- write or update a decision record, LEARNINGS.md, or ECOSYSTEM.md;
- write the CHANGELOG entry (changelogs here are curated prose; the merging
  maintainer writes the entry from your PR summary);
- read the repo's full ARCHITECTURE.md, unless your change touches the
  invariants it describes -- the reviewer will point you at the relevant
  section if it does.

One thing worth knowing exists: cross-repo design decisions are recorded in
the owning repo's `decisions/` directory, each with its rationale, the
alternatives that were rejected, and the criteria for revisiting. If review
feedback cites a record, that is not a brush-off -- it is the recorded
answer to exactly your question. And if you believe your change genuinely
meets a record's Revisit Criteria, say so in the PR; that is the mechanism
working as designed.

## Tier 2: core maintainers

Core maintainers and their coding agents handle the merge-time and post-merge
bookkeeping:

- roadmap items in the affected repo, following the item schema canonical in
  [isomorphic-lib-template `AGENTS.md`](https://github.com/interop-alliance/isomorphic-lib-template/blob/main/AGENTS.md),
  including `touches:` for anything that changes a spec, a wire contract, or a
  shared `@interop/*` API;
- the "Parties to this contract" walk for normative changes, and the
  breaking-release doc-vs-code audit (isomorphic-lib-template AGENTS.md,
  "Releasing");
- decision records (`decisions/NNNN-slug.md` in the owning repo) when a
  cross-repo decision is made, per the convention in isomorphic-lib-template
  `decisions/`;
- CHANGELOG entries, versioning, and publishing;
- writing cross-repo lessons into [LEARNINGS.md](LEARNINGS.md) in the same
  working session, and keeping [ECOSYSTEM.md](ECOSYSTEM.md) current.

Onboarding read for this tier: ECOSYSTEM.md, then LEARNINGS.md, then the
ARCHITECTURE.md of the repos you will touch.

Review convention: answer decision-shaped PR feedback by citing the decision
record rather than re-arguing it, and treat a contributor argument that
meets a record's Revisit Criteria as a reason to reopen, not as a nuisance.

Process gates stay maintainer-side: do not add CI enforcement of
maintainer-tier artifacts (roadmap references, changelog entries, decision
records) to contributor PRs. That trade-off is revisited only if multiple
core contributors start colliding.

## Not a tier: BYOE app developers

Developers building apps on was-react against these wallets are consumers,
but don't need to be direct contributors. Their documentation surface is
[was-react's](https://github.com/interop-alliance/was-react/) docs and the spec
texts (which address them directly, e.g. the ecosystem expectations in the App
Connect spec); the material in this file does not apply to them.

## The per-repo pointer (fan-out snippet)

Each active repo's CONTRIBUTING.md carries this short paragraph so
contributors learn the split without leaving the repo; this is the canonical
wording:

> PRs are welcome: tests plus a short summary of what changed is enough.
> You do not need to touch roadmaps, changelogs, or any cross-repo
> bookkeeping -- maintainers handle those at merge. The ecosystem-wide
> conventions (for maintainers) live in the
> [byoe-ecosystem](https://github.com/interop-alliance/byoe-ecosystem) repo.
