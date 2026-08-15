# Ecosystem Learnings

Durable cross-repo knowledge for the `@interop/*` ecosystem: invariants,
gotchas, and process recipes that are not the property of any single repo.
This file is rehydration memory, not audit history -- short enough to reread
at the start of any cross-repo task.

## What belongs here (and what does not)

- Belongs: a lesson that applies across two or more repos, or that would be
  lost when a repo-local doc is rewritten. Ownership rules ("X is canonical
  in Y"), pinned constraints, portable gotchas, process recipes.
- Does not belong: per-repo facts (that repo's AGENTS.md / ARCHITECTURE.md
  owns them); task history (the repos' archived roadmaps own it); anything
  derivable from code or a CHANGELOG. Never a second archive.

## Maintenance policy

- Prefer rewrite-in-place over append-only growth. Most updates edit an
  existing entry or the follow-ups list; do not add chronology.
- At the end of a task that produced a cross-repo lesson, record it here in
  the same working session -- one entry: the rule, why it holds, and one
  concrete example or pointer. Convert relative dates to absolute.
- Delete entries that stop being true. Compact the file when it grows noisy;
  split by topic only if a section outgrows the rest, and keep this file as
  the routing index if that happens.

## Where the canonical conventions live (routing)

- Roadmap item schema (`<PREFIX>-N`, field block, `touches:`, verbatim
  archival) and the publish/releasing recipe (TBD-dated top CHANGELOG entry,
  breaking-release doc-vs-code audit): isomorphic-lib-template `AGENTS.md`.
  Downstream repos defer there; do not re-derive the schema from examples.
- Code-style conventions: the marked block in isomorphic-lib-template
  `CONTRIBUTING.md` is the shared core copied across repos; edit it there,
  never in downstream copies.
- Contract consumers: the "Parties to this contract" tables in the AGENTS.md
  of wallet-attached-storage-spec, app-connect-spec, and
  encrypted-collections-spec. A normative change's checklist is a walk of
  the owning table.
- Decision records: cross-repo decisions get a `decisions/NNNN-slug.md`
  record in the owning repo; the template and convention live in
  isomorphic-lib-template `decisions/`.
- Terminology: `clientId` / `writerId`, never "device" (defined in the
  wallet repos' AGENTS.md files).

## Cross-repo invariants and gotchas

### No `instanceof` across package boundaries (the WC-64 rule)

Never use `instanceof` on a class or error type imported from another
`@interop/*` package. Under pnpm `link:` overrides (and any dual-package
situation) each package resolves its own copy of the dependency, so the
check fails even though the object is "really" that type. Use an `err.name`
check for errors, or a structural check (constructor.name plus expected
methods) in tests. Examples: freewallet's `OnboardingExchangeGoneError`
catch is a name check (2026-08-13); the linked-mode `instanceof ZcapClient`
test failure fixed structurally (2026-08-01).

### `pnpm test` in the lib repos is lint + tsc + vitest

In the template-derived repos (wallet-core, was-client, ...) the `test`
script runs lint, `tsc` over the dev tsconfig (which covers `test/`), and
vitest. Running vitest alone under-reports: type errors in test files pass
silently. A "green" verdict requires the full `pnpm test`. Learned
2026-08-01 when a vitest-only run missed tsc errors in two test files.

### Publish in dependency order, then re-verify off the registry

When a change spans layered packages (e.g. was-client then wallet-core then
the wallets), publish upstream first, drop every `link:`/`workspace:`
override, and re-run the consumer's full suite against the registry
versions before calling the train shipped. Linked-mode green does not prove
registry-mode green (see the instanceof rule above; also export-map and
peer-range mistakes only surface off the registry).

### Playwright: the first fill after a navigation needs settling

`page.goto()` (and popup loads) resolve before React commits a lazy route;
under load, a `.fill()` issued in that window lands on the outgoing route's
still-attached input and dies with it, leaving the fresh field empty --
which surfaces as an unreproducible flake. Any fill that is the first
interaction after a goto or popup load must fill-then-verify-then-retry
(freewallet's `fillSettled` helper in `tests/e2e-was/helpers.ts`:
fill, `toHaveValue`, `toPass`). Root-caused 2026-08-01; the pattern applies
to every React e2e suite in the ecosystem.

### vitest: key derivation and VP signing need the node environment

`CapabilityAgent.fromSecret()` and VP/DIDAuth signing throw under vitest's
jsdom environment (`"data" must be a string or Uint8Array`) even though the
same code works in node and in real browsers. In repos whose vitest default
is jsdom (needed for React component tests), any test file that derives a
key agent or signs must start with `// @vitest-environment node`.

### ezcap convenience methods send the wrong WAS actions

`ZcapClient.read()` / `.write()` send `action: 'read'` / `'write'`, but the
WAS action vocabulary is HTTP verbs. For node-side WAS invocations use
`request({ method: 'GET', action: 'GET' })` (etc.). Learned 2026-08-03
writing server-side e2e assertions.

### grep silently skips files containing NUL bytes

A source file with a literal NUL byte (e.g. was-react `src/config.ts`, a
`.join('\0')` written as a raw byte) is treated as binary: plain `grep`
skips the whole file with no warning, so a "no matches" sweep can lie. Use
`grep -a` when a sweep must be exhaustive.

## Current follow-ups

- Seed further entries from the older per-repo lessons as they resurface;
  this file was created 2026-08-15, modeled on the keri-ts project's
  compaction-governed learnings layer.
