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
  concrete example or pointer. Convert relative dates to absolute. Describe
  the originating work by what it was, not by a repo's roadmap item id --
  those live in gitignored planning files and mean nothing here.
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
- Terminology: `clientId` / `writerId` as opposed to "device" (defined in the
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

### Talk to WAS through was-client handles, not raw ezcap

For node-side WAS invocations (e2e assertions, scripts, acting as a
grantee), use was-client: `WasClient.fromCapability()` rebuilds a pre-bound
Space/Collection/Resource handle from a received zcap, and the typed
helpers (`get()`, `put()`, ...) send the correct actions. Even was-client's
`request()` escape hatch defaults `action` to `method`. The trap this
avoids: ezcap's `ZcapClient.read()` / `.write()` conveniences send
`action: 'read'` / `'write'`, but the WAS action vocabulary is HTTP verbs,
so raw-ezcap calls against WAS fail authorization unless every call passes
an explicit verb action. Learned 2026-08-03 writing server-side e2e
assertions with raw ezcap.

### was-client's `request()` escape hatch throws unmapped errors

`WasClient.request()` is the raw signed-request primitive: it applies NONE
of was-client's typed-error mapping, so a failed write surfaces as the bare
ky/http-client error (`name: 'HTTPError'`, `status` on the root), never as
`PreconditionFailedError`, `NotFoundError`, etc. Any store or ceremony
seam whose contract is an error NAME (e.g. wallet-core's `WebvhIdStore`,
which maps `PreconditionFailedError` to the log-conflict rebase) must map
the raw status itself when it writes through `request()` -- dispatch on
`err.status === 412` (or 404) and rethrow the typed class. Learned
2026-08-19 building the companion delegated log store, when freewallet's
shipped `unlockLogStore` turned out to rely on a comment claiming the
typed error surfaced from `request()`: its bridge-delegated `did.jsonl`
CAS could never detect a lost race. wallet-core's
`delegatedWebvhLogStore` (0.46.0) is the shared, correctly mapped
implementation.

### grep silently skips files containing NUL bytes

A source file with a literal NUL byte (e.g. was-react `src/config.ts`, a
`.join('\0')` written as a raw byte) is treated as binary: plain `grep`
skips the whole file with no warning, so a "no matches" sweep can lie. Use
`grep -a` when a sweep must be exhaustive.

### Multikey decoding: use data-integrity-core's decodeMultikey, not hand-rolled header compares

`@interop/data-integrity-core`'s `/multihash` subpath (8.7.1) exports
`decodeMultikey({ multikey, expectedCodec })`: `z`-prefix check, base58btc
decode, multicodec varint parse, codec expectation, and per-codec key-length
validation, covering the Ed25519/X25519 and NIST P-curve public AND private
codecs. New decode sites should call it instead of comparing header bytes.
A 2026-08-16 sweep found the hand-rolled versions consistently under-check,
in two recurring ways:

- Missing prefix check: helpers that strip the first character without
  testing for `z` (x25519-key-agreement-key's `multibaseDecode` and its
  copies) silently mis-decode a `u`-base64url string as base58.
- Missing length check: header-only compares (did-method-webvh's
  `decodeEd25519Multikey`) accept a 4-byte payload with an `ed01` prefix.

Two wire facts the decoder encodes, worth knowing when reading key bytes
anywhere: ed25519-priv multikeys come in two lengths -- the Multikey spec's
32-byte seed and the legacy 64-byte seed||pub form that
ed25519-verification-key still emits by default -- so length checks on that
codec must accept both; and P-curve public multikeys are compressed SEC1
points (33/49/67 bytes for P-256/384/521), so a raw or uncompressed point is
malformed. Migrations of existing suites must preserve their pinned error
messages (or result-object contracts, e.g. `verifyFingerprint`) by catching
the primitive's throw and mapping it.

### did:webvh verification methods: extra properties survive, unwired VMs default into `authentication`

Two facts about `@interop/did-method-webvh`'s document assembly
(`normalizeVMs` / `createDIDDoc`) that any repo publishing verification
methods through `createDID` / `updateDID` should know. First, a VM object
is spread into the document verbatim, so custom properties (e.g.
wallet-core's `publicKeyCommitment` entries for low-entropy-derived unlock
keys) survive creation, updates, resolution, and the did:web projection.
Second, a VM with no `purpose` is wired into `authentication` by default --
but explicit relationship arrays passed alongside `verificationMethods`
override the purpose-derived wiring wholesale. Wallet-core's ceremonies
always pass all five relation arrays, which is what keeps a
keyAgreement-only entry out of `authentication`; a new call site that
omits the arrays would silently grant unintended relations. Learned
2026-08-15 building the hash-commitment unlock-key posture entries.

### did:webvh document `@context`: updates preserve it

`@interop/did-method-webvh`'s `updateDID` with the `context` option unset
PRESERVES the prior entry's `@context` (falling back to the base pair only
when the document never had one), so a context added at genesis or in one
update survives every later update without threading it through each call
site. To add a context, pass `additionalContext` (appended after the base
pair or the carried-forward context, deduplicated); the `context` option
is the full-override escape hatch and replaces the `@context` wholesale.
Learned 2026-08-15 adding the `https://w3id.org/byoe` context for
`MultikeyCommitment` entries.

### webvh prerotation: a client can never remove its own update key

Under the ecosystem's prerotation convention a log entry verifies
against its own re-stated `updateKeys` (each hashing into the previous
entry's `nextKeyHashes`), so no entry can both remove a client's update
key and be signed by it -- self-revocation of a fully enrolled client
is structurally impossible for that client, not merely refused by
wallet-core's guard. Any "disconnect myself" ceremony needs a second
authority: another enrolled client, the standing unlock credential's
ladder, or a partial retirement that removes the verification methods
while leaving the (now dead) update key behind. Learned 2026-08-16
implementing freewallet's ephemeral-client logout (FW-156; the ceremony
choice is FW-168).

### A policy check inside a pre-verification callback runs on unverified input

When a callee invokes a caller-supplied callback before its own
verification step (e.g. a kernel's `authorize` callback that runs before
the signature check), a policy or admission check placed inside that
callback sees attacker-chosen input, and its refusal carries the wrong
error class if the input turns out to be forged. Put such checks after
the verifying call returns, on the confirmed-verified result. When a
consumer's error-class predicate gates a hard failure against a soft one,
test the forged-input case against the class it actually surfaces with,
not the class the check was written to raise. Learned 2026-08-22 fixing
vh-resource-log's `admitAppend` hook (VRL-1), which ran inside the
did:webvh kernel's `authorize` and so was consulted on an unverified
proof.

### On an append-only log, the writer runs the reader's full check before the write

Where one refused entry rejects the whole log for every reader and an
appended entry cannot be removed, a writer that builds an entry and
sends it straight away can poison the log it still reads: a key removed
at the controller head, a malformed caller-supplied timestamp, or a
signer seam whose key and signature disagree all write durably and fail
on read-back. The fix is a pre-write pass that runs the reader's full
per-entry verification over the candidate (shape, chain, proofs,
authorization at the head's floor, the admission hook), from the same
code the read loop uses, so the writer refuses exactly what the reader
would. A subset of the checks (policy only, say) leaves the same hazard
open one member over, and a consumer with its own write path must call
the same exported check rather than re-derive part of it. Read-back
stays the only evidence of durability; the pass is self-protection, not
an authorization boundary. Learned 2026-08-22 on vh-resource-log VRL-2
(`verifyResourceLogAppend`), where wallet-core's roster store carried
only the license half of the check inline.

### Scanner error contracts differ by localization, on purpose

The two wallets render a failed QR scan or paste differently, and the
split is deliberate rather than drift. dcw's callbacks throw
`HumanReadableError` with a finished English message and the surface
renders `err.message` as-is (`app/lib/error.ts`, `errorMessageFrom`);
anything else is logged and shown as a fixed fallback. freewallet's
callbacks throw typed error classes carrying a `code`, and the surface
maps each to an i18n key (`src/lib/resolveCredentialsInputErrorMessage.ts`
over `ScanCredentialQrDialog` and the add-credential page). A
pre-rendered message cannot be translated, so a localized app must keep
the mapping at the surface. When lifting scan or paste logic into
wallet-core, throw typed errors with a stable name and code and leave the
message choice to each app; do not port `HumanReadableError`. Recorded
2026-08-22 closing freewallet FW-101.

### Coined terms get retired in favor of ecosystem vocabulary

The resource-log profile called the `versionId` DID parameter on an entry
proof's `verificationMethod` its "anchor". The word is not Data Integrity or
DID Core vocabulary, and it collided with three unrelated senses already in use
(ladder-anchored, credential-anchored genesis, trust anchor). On 2026-08-22 it
was retired for "controller versionId" across encrypted-collections-spec,
vh-resource-log (code, ARCHITECTURE.md glossary, ROADMAP, design doc),
wallet-core's `resourceLog/` module, and app-connect-spec decision 0003.

When naming a profile concept, reuse the term the underlying spec already has
(here DID Core's `versionId`) and qualify it (`controller versionId`) rather
than coining a new noun. Before a rename, grep every sibling repo and tag each
hit by sense; renames that touch a public member (`headAnchorIndex` to
`headControllerVersionIndex`) are breaking and ship with the next major of the
library, with the consumers' range bumps in publish order.

## Current follow-ups

- Seed further entries from the older per-repo lessons as they resurface;
  this file was created 2026-08-15.
