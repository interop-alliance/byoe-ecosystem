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
authorization at the head controller version, the admission hook), from the same
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

### A per-entry rule checked per element needs the reduction before the check

When a rule is written for a whole record but the verifier checks it element by
element (each element's signature or membership), compute the reduction from
elements to one record-level value before running the per-element check, and do
it before any per-element loop that has no lookahead -- a call for element 1
cannot see element 2, so the reduction cannot be discovered mid-loop. A related
fact worth checking whenever a record carries an array outside its hash and
outside every signature: a host can reorder, duplicate, or delete elements of
that array undetected. Deletion is the sharper case when each element's
signature covers only that element and the record minus the array: a strict
non-empty subset of the array still verifies, so any count taken from the
verified elements is a lower bound, not the true count. Any code that reduces
or counts over the array must be set-based (order- and
multiplicity-insensitive) and must not treat that count as complete. Learned
closing vh-resource-log's VRL-4 (an entry's proofs
each carried their own controller version, reduced by taking the maximum after
every proof verified, letting a removed member's proof ride on a co-signer's
version); the reduction moved to a pre-pass before the proof-verification
kernel's loop, and the admission hook gained an explicit order-insensitivity
obligation over its per-proof key list. See vh-resource-log's
`decisions/0002-one-controller-version-per-entry.md`.

### Verify a sibling repo consumes a module before designing convergence around it

Freewallet's FW-292 design (the unlock-methods registry's write protocol)
was drafted, and its wallet-core extraction sequenced, on the premise that
dcw restates the registry helpers and would be converged by version bump.
The premise was false: dcw holds no unlock-methods registry at all, a
recorded dcw product decision (its ARCHITECTURE.md defers recovery codes
to web for exactly this reason). The false claim propagated into the
roadmap item's `touches:` dcw line, a consumer-enumeration claim ("dcw's
counterparts of all of these"), a guard's "both callers" clause, and the
extract-before-fixes sequencing rationale -- and it fell only to the
adversarial review's consumer-completeness charter, after which the
sequencing decision had to be re-made (the fixes now land before the
extraction, freewallet FW-293..299 then wallet-core WC-193).

The lesson: "dcw and freewallet share the ceremonies" does not imply dcw
consumes any particular module, and a `touches:` line or shared-code claim
about a sibling repo is a checkable fact, not an assumption -- grep the
sibling's source for the exported names, and read its ARCHITECTURE.md for
a recorded decision NOT to hold the thing, before any convergence or
extraction is sequenced on it. An extraction whose consumer does not exist
is preparation for a possible future, and should be prioritized (and
ordered against fixes) as that, not as convergence of shipped
restatements. Recorded 2026-08-23 closing freewallet FW-292.

A second instance, 2026-09-05: freewallet FW-447 was filed (out of FW-429's
touches) to move the recovery spend's registry drop into wallet-core "so
dcw, which runs the same continuations, gets it". dcw runs no recovery
continuation and holds no registry, the same recorded decision as above.
The item was re-scoped to the freewallet-internal consolidation before any
wallet-core code was written. A `discovered-from` chain carries premises
forward: an item filed from another item's touches list inherits that
list's unchecked claims, so the grep is owed at filing time, not only at
design time.

### A pnpm install under a live vite dev server poisons the dep-optimize cache

During freewallet FW-315's e2e sweep, a `pnpm install` landed (a
version drift re-resolve) while the playwright-managed vite dev server
was up. Two distinct poisonings followed, both surfacing as app
failures with nothing wrong in the source. First, immediately: the
install pruned `.pnpm` paths the running server held, so every page
load got ENOENT on vite-served chunks and rendered an empty shell.
Second, and far subtler, afterwards: `node_modules/.vite` (the
dep-optimize pre-bundle) rebuilt against the mid-churn node_modules
and kept serving that stale bundle across server restarts. The symptom
was a false integrity refusal -- `ResourceLogIntegrityError` from the
bundled `@interop` verifier chunks -- in exactly one e2e spec, while
the same verification replayed clean in node against the same on-disk
bytes with the same installed packages. Deleting `node_modules/.vite`
and rerunning with zero source change fixed it.

The lesson, for every vite app repo in the ecosystem (freewallet, the
was-react examples): a failure that only the bundled app exhibits,
while the installed library verifies the same data clean in node, is a
stale pre-bundle until proven otherwise -- and after any `pnpm
install` that lands while a dev server is running, clear
`node_modules/.vite` before trusting an e2e run (or add the rm to the
playwright webServer command). The replay-outside-the-app step is also
the cheap discriminator between "our ceremony code broke" and
"the bundle is stale". Recorded 2026-08-25 closing freewallet FW-315.

The same staleness runs the other way, and it bites the useful test:
editing an installed or `link:`ed package's dist to reproduce a
pre-fix state changes nothing the pre-bundle serves, so the "before"
run passes and reads as a fix that was never needed. Vite invalidates
the pre-bundle on a lockfile or package.json change, not on a file
edit under `node_modules`. Run that comparison with `vite --force` (or
clear `node_modules/.vite` between the two runs). Recorded 2026-08-28
closing freewallet FW-359, whose pre-fix 404 was only reproducible
that way.

### A ceremony mend entry point must classify from the last sub-stage's artifact

A mend function for a multi-stage ceremony often has to decide which arm to
run: "the whole thing needs a re-run" versus "only the last step needs
finishing". That decision must probe the durable artifact of the LAST
sub-stage the finishing arm assumes is already done, not an earlier stage's
signal. wallet-core's `mendCredentialAnchoredAccount` classified "the
establishment already ran, only the record needs re-binding" by attribution
of the published log -- a signal an early sub-stage sets. A tear after the
log published and its update key revealed, but before the account's
`#DelegatedClients` pointer was written, still satisfied that signal, so the
mend took the re-bind arm and threw on the missing pointer; the full
establishment re-run that would actually have healed the tear never ran,
leaving that tear permanently unmendable. The fix gated the re-bind arm on
the pointer itself being present in the published document, and let every
earlier tear fall through to the full re-run, each of whose stages is an
ensure and converges safely either way.

The general rule: an arm that assumes "the ceremony completed through stage
N" must probe stage N's own artifact. Attribution of an earlier stage's
output is satisfied by every tear at or after that stage, so it misclassifies
every later one. Recorded 2026-08-26 closing freewallet's credential-anchored
mend-entry-point work (paired with wallet-core).

### A delegated capability is tested by invoking it, not by asserting its shape

A test that checks a minted zcap's fields -- parent, `expires`,
`allowedAction`, target -- passes just as happily on a capability no server
will ever accept, because none of those fields carry the delegator's
authority. Only an invocation against a server that resolves the delegator's
document exercises the proof. Freewallet's public-terminal App Connect e2e
asserted exactly that field set on a grant whose delegation proof could never
verify, and shipped; the break surfaced from a BYOE example app in the field,
as a 404 on every request the grantee made. Budget one end-to-end invocation
per capability-minting path, and treat shape assertions as a supplement to it.
Recorded 2026-08-27.

### A vocabulary rule from an app repo needs its neutral form in the shared library

Freewallet's `decisions/0011` settled that "durable" names server-backed
storage only, and named the middle tier "browser-local". Carrying the rule
into wallet-core exposed the limit: wallet-core also serves DCW, where a
browser is not what persists anything. The shared library took
"client-local" instead -- the tier named off the repo's existing `clientId`
axis, so it reads right for a browser profile and a phone alike -- and
freewallet's "browser-local" is documented as its app-side specialization.
Both Glossaries say so, and neither word had to lose.

The rule: when an app repo's domain decision reaches a shared package, check
each coined term against every consumer before copying it across. The app is
free to name a tier for its own runtime; the library needs the term that is
true for all of them, plus an explicit line mapping the two. Copying the app's
word into the library is how a package acquires vocabulary that is wrong for
half its consumers.

A phrase-level bulk rename also needs a pass no toolchain gives you. Renaming
"durable client" to "enrolled client" across 200-odd comment sites left two
classes of damage that tsc, eslint, and prettier all accept: article
agreement ("a enrolled client"), and duplicated words where the old phrase
straddled a line wrap ("last enrolled\n * enrolled client"). Both need their
own greps -- `\ba <vowel-word>` and a backreference for a repeated word
across an optional comment marker -- run after the sed and before the review.
Recorded 2026-08-27 closing freewallet FW-368.

### A relation-asymmetry recognition rule has one owner and many copies -- walk them all

The ladder-VM recognition rule ("in `capabilityDelegation`, absent from
`capabilityInvocation`") is implemented independently in three places --
wallet-core's `ladderVmIds` (webvh/listClients.ts), wallet-core's
`inventoryOf` (resourceLog/controller.ts, feeding the ceremony-tail
license), and the server's `clientAnnexClause.ts` -- and its normative
home is a fourth: app-connect-spec `decisions/0003`. FW-359's design
review found the doc had enumerated two of the four; the third copy was
the one whose divergence would surface as `ResourceLogLicenseError` far
from the change.

The rule: when a design changes any verification method's published
relation set, grep every repo for the asymmetry predicate itself (both
relation names in one function), not just for readers of the document,
and cite the decision record that owns the rule so the revisit trigger
names it. A negative-space rule ("skipped because it does not match")
is still a consumer, and it is the kind no export-map walk finds.
Recorded 2026-08-28 reviewing freewallet FW-359's design.

### An "ensure" ceremony's completion callback fires on the adopt branch too

wallet-core's `ensureDidWebvh` reports `onDidPublished({ did })` whether it
created the log or adopted one already there. Freewallet's callback
invalidated the session's verified-log memo unconditionally, which was
harmless while nothing read that memo synchronously. FW-344 made the
DIDAuth holder dispatch a memo-only read, and the review found the
ordinary remembered login now emptied the memo between the post-login
gate and the approve-time dispatch, so a webvh-only request was answered
with did:key. The adopt branch publishes nothing, so there was nothing to
drop.

The rule: a callback named for a write ("published", "committed") on an
ensure-shaped ceremony fires on the no-op branch as well, unless the
contract says otherwise. A consumer that clears cached state in it must
compare what the callback reports against what it holds, and clear only
on a change. When adding a synchronous reader over a memo, grep for every
invalidation site and ask which of them fire on the default login path.
Recorded 2026-09-03 closing freewallet FW-344.

### A derived projection is only as fresh as its last authorized writer

`id/did.json` is the did:web projection of a did:webvh log: derived from
`did.jsonl`, and stale the moment an entry publishes without a writer for it.
Whether a ceremony can refresh it is a question about authority, not about
the ceremony. A standing unlock credential's bridge delegation is a PUT on
exactly `did.jsonl`, so every ladder-signed entry writes the log alone, and
the projection kept naming a client the last-client transition removed or a
credential a transient recovery retired. WAS authorization never noticed.
The server resolves a Space's controller from the log and reads the
projection nowhere, which is why the drift survived: the account works
perfectly while did:web verifiers accept revoked keys.

Two placements close a gap like this without widening anything. A ceremony
whose own authority ends at its entry writes the derived resource
immediately BEFORE that entry, under the authority it still holds; a run
torn in between leaves the derived copy under-listing what the log has,
which is the fail-closed direction for a verifier and is re-written by the
re-run. And a standing mender needs some caller with a writer on the
ordinary path: here the client annex's generation delegation, whose target
is the account Space's items subtree, which already covers `id/did.json`,
so a credential-only visit republishes with no server predicate change and
no re-minted bridge. Compare-then-write, so a healthy account pays one GET.

The rule: for every resource a wallet derives from a log, name the writers
of the derived copy separately from the writers of the log, and check the
narrowest one against every ceremony that changes what the log says. A
delegation scoped to one resource is a scope on which resources can be kept
consistent, not only on what can be published. Recorded 2026-09-03 closing
freewallet's stale-projection item (wallet-core `ensureDidWebProjection`,
the removal ceremonies' pre-entry PUT, freewallet `decisions/0018`).

A third placement removes the drift class instead of bounding it: the
SERVER derives the projection from the log head, so the log is the only
thing any writer writes. The server already does this for a did:webvh
Space controller, which it resolves out of `did.jsonl` on every request.
The log-governed collection encryption descriptors (designed 2026-09-07)
take it for the Collection Description's `encryption` member: the wallets
append to a governing log and never PUT a projection, the server serves the
member as the head's `state` plus `history`, and a direct write of the
member is refused. The trade is a server feature and a spec clause against
a per-ceremony PUT, an ensure sweep, and a tear residue in every producer.
Prefer it whenever the server already stores the log and the derived copy
is small; the did:web projection stays producer-written only because it
predates the pattern.

### A verifier-side license must track a ladder's current rung, not its introducing one

wallet-core's ceremony-tail license clause B admits a ladder-signed roster
append only when the entry it anchors at was signed by a rung the log
attributes to the appending ladder. WC-190's first attribution pass
anchored a ladder at the rung that introduced its verification method and
stopped there, which is wrong the moment that rung is spent: a
self-enrollment retires the rung it uses, and every passkey account
self-enrolls at signup by construction, so the very first login of a
passkey account would already be attributing a rung the log no longer
authorizes. The fix climbs the attribution forward with the log, by the
last-position rule `decisions/0007` already states for the reveal/commit
hash order: an entry that reveals exactly one new update key it also
signed is the ladder's next rung when an earlier entry committed that
key's hash last among its own additions, while the ladder's VM still
stands.

The rule: a verifier-side license or attribution keyed to "who controls
X" must track X's CURRENT holder, not the holder at the moment X was
introduced, whenever anything in the system can rotate X on its own --
self-enrollment rotates a ladder's active rung the same way a client
revocation rotates a roster's recipient set. And any new license shape is
a whole-log refusal to a reader that has not shipped it (the admission
hook throws and the verifier propagates it), so its rollout is
verifier-first: every reader of the governed log ships the shape before
any writer emits an append that needs it. dcw's DCW-71 is the pending
consumer of this shape and must land before dcw's own ladder-branch
writers can emit shape-3 appends. Recorded 2026-09-04 closing wallet-core
WC-190.

### The next roadmap id is a counter, not a scan

Every roadmap in the ecosystem moves completed items out of ROADMAP.md
into an archive file, and completed items are the newest ones. So the
highest id almost always sits in the archive, and an agent that derives
the next id from the open roadmap alone reuses one. On 2026-09-04 that
had happened in both freewallet (open max FW-423, archive max FW-425) and
wallet-core (open max WC-186, archive max WC-193). Re-deriving the id
from both files on every filing is also wasted work.

The fix is a `nextAvailableId: <n>` line directly under the H1 of every
ROADMAP.md, the sole source of the next id: filing an item takes `n` and
rewrites the line to `n + 1` in the same edit. Seeded from the archive's
max plus one, never from the open roadmap's. One recovery rule covers
drift (a stale branch, a hand edit): if the counter's id already appears
in either file, reset the counter to one past the highest id across both,
then take it. Seeded 2026-09-04 in freewallet, wallet-core,
isomorphic-lib-template, was-client, was-react, was-teaching-server, and
dcw; the rule text is canonical in isomorphic-lib-template's AGENTS.md and
mirrored in each repo's AGENTS.md and roadmap header.

### Two forks of a stored schema are never "effectively the same file"

An extraction design that merges two forked copies of a module tends to
grade the smallest files as interchangeable. On 2026-09-05 FW-448's draft
called freewallet's and was-react's `syncedDocSchema.ts` (42 and 44 lines)
"effectively the same file, from either copy". The diff was three
hash-affecting declarations (a `createdBy` property on one side, a
`required` list and an `updatedAt` index on the other), both at RxDB
`version: 0`. RxDB hashes a collection's declared schema and refuses
`addCollections` on a database whose stored hash differs at the same
version, so adopting either copy would have failed login on every existing
replica of the other app. The rule: a declaration that a storage engine
persists (an RxDB schema, a Dexie store definition, a SQLite DDL string) is
browser-local stored state, and a merge design diffs it byte for byte and
states the version or invalidation answer before calling it a move.

### On 0.x, a caret pins the minor; a release train lists every range bump

`^0.47.0` admits `0.47.x` and not `0.48.0`. A multi-repo train whose
consumers hold 0.x caret ranges therefore floats nothing: each publish
needs an explicit range bump in every consumer, and a consumer left
behind resolves a second copy of the package beside the one a sibling
dependency pulled (was-react `^0.47.0` against wallet-core `^0.48.0` of
was-client, 2026-09-05). Write the train as a table of publishes AND
bumps, and read the one-resolved-copy lockfile audit as the check that no
bump was missed rather than as what keeps the tree single-copy. A `link:`
held in any consumer at the train's start voids the audit there, so the
train's first step is to publish and drop it.

### A `link:`ed package resolves its peers from its own checkout

A pnpm `link:../pkg` reference symlinks the sibling checkout, and that
checkout resolves its own `node_modules`. So a linked package that declares
a peer (`rxdb`, `@interop/was-client`) loads the copy installed in ITS
directory, not the consumer's, and the consumer's tree holds two physical
copies of the peer. Two things follow. For types, `tsc` sees two identities
for one class (`RxCollection` from each copy) and every value handed
across the boundary fails to assign; the fix while linked is a
`compilerOptions.paths` mapping for the peer in the consumer, commented
as link-only and removed with the link (was-react on `@interop/was-sync`,
2026-09-05). For declaration emit, a consumer whose exported types are
inferred through a linked dependency's transitive package fails with
TS2883 "cannot be named"; the fix is an explicit return-type annotation at
the export, which is correct in registry mode too (wallet-core's
`keyring/kdf.ts`, same day). For runtime, a peer that installs itself onto
a class prototype (RxDB's leader-election plugin) lands on the wrong copy,
and only `resolve.dedupe` in vite keeps the browser bundle single-copy.
None of this is visible off the registry, which is one more reason the
train's last step is to publish, drop every link, and re-run the suite.

### A fake WAS server is a second implementation of the contract

A hand-written in-memory WAS fake ("accepts every write, serves a plausible
feed") is a second implementation of the WAS wire contract, maintained by
the consumer, and it drifts. was-sync's `FakeWasServer` (removed
2026-09-05, WS-11) synthesized a `412` for a header-less `DELETE` that a
real server answers `204`, never assigned `createdBy`, and raised the error
shapes of a `mapAuthErrors: true` port while the main consumer runs the
default port. Four of the ten findings from that day's review were cases
the fake modeled wrongly, and the integration suite passed on all of them.
was-teaching-server exports an in-process `createApp` (filesystem backend
into a `mkdtemp` directory, `listen({ port: 0 })`, then set
`app.serverUrl` to the assigned port), so a driver or client suite runs
against the real server through the real `createWasSyncPort` for the cost
of one devDependency. Two things the real server teaches that a fake does
not: a plaintext collection's `custom` is `{ name, tags }` with `tags` a
string-to-string record (anything else is `400`), and the replication
collection needs the package's LWW conflict handler installed explicitly,
since RxDB's default hands every conflict to the remote.

### A license enforced on the controller adapter binds every log the adapter verifies

wallet-core's ceremony-tail license reads as the roster's rule, and the
clause that states it is titled "the roster axis", but it is enforced as
the mandatory admission hook on the did:webvh controller adapter, so every
resource log verified through that adapter inherits it. The log-governed
collection descriptors (designed 2026-09-07) were nearly built around a
license that was never meant for them: a ladder VM already stands under
`assertionMethod`, so a share from a credential-only session had a valid
signer for a descriptor append, and the only bar was a hook written for
the root key's roster. The resolution scoped the license to the roster
and had the hook take a log class, so a descriptor log passes on
membership alone (app-connect-spec `decisions/0003`, clause B's scope).

The rule: when a policy is enforced at a shared verification seam, a new
artifact verified through that seam states which policy applies to it
when it is introduced, rather than discovering at build time that it
inherited one. And read the enforcement point, not the decision's title,
to learn a policy's real scope.

### A collection-level log lives in a sub-resource, and its placement is chosen by its readers

Two placements for a governing log looked natural and both fail. Beside
the roster in `key-map` is where wallet-core's pin slot id assumed it, and
`key-map` is private: a share grantee holds a read zcap on the shared
collection and an app on its own, so neither could read the log the
pointer named, and the pointer-following reader the log form exists for
would get the masked 404. As a document of the collection it governs, the
server's envelope rule refuses it (an encrypted collection accepts only
envelopes), and a document enumerates in listings and the `changes` feed,
where the sync driver replicates it as a row. The placement that holds is
a sub-resource under the collection URL beside `/meta` (settled
2026-09-07 as `/space/{space_id}/{collection_id}/meta/log`): covered by
any capability whose target covers the collection, outside its document
set for listing, replication, and envelope enforcement.

The rule: before naming where a capability-gated resource lives, list
every reader that must reach it and the capability each one actually
holds. The wallet's own reach is the easy case; the external reader is
what decides placement.

### WAS holds the server surface, the profile holds the semantics

The split for log-governed collections follows the one key epochs already
use. WAS owns what a server does and what a client sends over HTTP: the
sub-resource and its operations, the minimal line contract (JSON Lines,
a `state` member per line, last line is the head), the declaration, the
derivation rule, the problem types, the features flag. The Encrypted
Collections profile owns what an entry is beyond `state`: proofs, chain,
`type`, the reserved `history` rule, verification, external authorization,
and the verifying reader's equality check, none of which the server
checks. Recorded 2026-09-07 when the governing-log spec item was first
written as an encryption feature and had to be regeneralized.

The rule: a mechanism that a non-encryption use would need identically is
WAS text, kept minimal because WAS is a W3C CCG work item and every clause
costs consensus a profile clause does not; anything cryptographic or
verification-side stays in the profile, cited by name from WAS rather than
restated.

### Verify a server claim in source before designing an ordering rule on it

A design pass on 2026-09-07 asserted that the server validates a write's
`Key-Epoch` stamp against the collection descriptor's epochs, and built an
ordering hazard on it: a rotation's projection had to land before the
first write under the new epoch, or the server would refuse the epoch as
unknown. The header is stored opaquely and never checked
(was-teaching-server `src/lib/keyEpoch.ts` says so in its header comment).
The hazard did not exist, and a mender was nearly filed for it.

The rule: a server behavior that a client-side ordering rule depends on is
cited from the server's source or spec text, not from what the behavior
"must" be. The teaching server's module headers state what is and is not
validated; read them before designing around a check.

### Swapping an unsigned write for a signed log append changes two contracts at every caller

When a resource that consumers wrote with a plain conditional PUT becomes
log-governed (the per-collection encryption descriptors, 2026-09-07), every
write site inherits two obligations it never had, and neither shows in the
store seam's type. First, the append's proof key must stand under
`assertionMethod` in the controller document at the version the append
anchors at. A cascade that anchors every collection store at the post-edit
head (correct) while the app still signs with the credential being retired
(the login credential on a passphrase change) has every append refused and
reports the rotation done. Second, the errors a write can raise now include
the verifier's refusal classes (integrity, fork, license), which a fan-out
that collects per-collection failures into a retry bucket turns into an
endless retry with outage copy. The read path already rethrows those by
`isResourceLogRefusal`; the write path has to as well.

The rule: when a seam's backing changes from host-trusted to verified, walk
every writer and state the signer rule on each seam's contract text, and
apply the reader's refusal classification to the writer's failure handling.
The type of the store does not change, so nothing else will surface it.

### A verified read leaves a host-controlled gate standing wherever a membership test still reads the server

Moving the per-collection encryption descriptors under verified logs
(freewallet, 2026-09-08) verified what each rotation wrote and read, and
left the question of WHICH collections a rotation covers on the server's
word: the cascade's `isEncrypted` probe read the Space listing's derived
`encryption` member. A host that omits the member for one collection keeps
it out of every rotation, so it stays keyed to the retired user key
generation, with every append it does make fully verified. The same shape
had been closed in wallet-core's mend arm the day before (a fabricated
absence licensing a fresh roster genesis), and it recurred one layer up
because the gate is not a read of the resource and so was not on the list
of reads to move.

The rule: when a resource moves from host-trusted to verified, enumerate
every predicate over its existence or membership (is it encrypted, does it
have a roster, is it in the set to rotate), not only the reads of its
content, and answer each from the verified artifact (the log's head, or the
absence of a log). The listing may enumerate; it may not decide.

### A self-forget cannot seal the logs it leaves behind

Both forget grades run their collection fan-out before the removal entry,
the inversion the self-forget forces, so every collection log append anchors
at a version that still lists the departing client's key. The departing
client cannot append after the entry (its authority ends there), and on the
last-client transition no remembered login ever runs again to seal. A
document-edit-first ceremony seals for free because its fan-out anchors
post-edit; an inverted one owes an explicit sealing pass from the surviving
signer, and on a client-less account that signer is the ladder branch, so
the pass has to be a transient-login stage (freewallet FW-450, 2026-09-08).

The rule: for every ceremony whose fan-out precedes its strike, name the
sealing pass and who fires it; "the cascade seals" holds for the
edit-first order only.

### Two consumers carrying the same cast-and-probe means the type belongs upstream

was-client typed the sync port's `putMeta` as optional while its own
`createWasSyncPort` always supplied it, and its feed page carried the shared
`unknown` bodies where a consumer needs `Json`. was-sync and was-react each
answered with the same workaround: cast the whole port through `unknown` and
probe for `putMeta` at runtime. The cast silenced every other member too, so
a rename in `query` or `deleteContent` type-checked clean at the seam and
surfaced only inside a push or pull cycle as an `error$` event. Completing
the type upstream (required `putMeta`, `Json` bodies; was-client WCL-39,
was-sync WS-10, 2026-09-08) turned that divergence into a compile error and
deleted both workarounds.

The rule: an `as unknown as` cast at a package seam is a claim about the
upstream type, and the same cast in two consumers is a bug report against
it. Fix the type in the owning `@interop/*` package and alias it downstream,
so the seam is checked by construction.

### A registry that describes menders indexes sites its runner must not run

The mender registry keys every login-time repair by the invariant it makes
true, and its derived-set audit reads one flat index of sites: the two login
chains' registrations, the routing entries that decide whether a session is
built at all, and one entry whose own call site fires it before the chain.
Only the first group is executable. The first build split the index in two
to keep them apart, which immediately drifted: the audit read one array and
the runner another, and an entry could sit in either without failing
anything. The fix was upstream instead -- the shared runner skips a site
carrying no converger, whichever list the block runs from (wallet-core
WC-224, freewallet FW-455, 2026-09-09) -- so one index serves both readers
and a converge-free entry is unrunnable by construction rather than by
bookkeeping.

The same build wanted a settle point partway through one trigger's list, and
took an optional registration-list override on the runner rather than a
second trigger value: the order is one list's order, and a second value
would have hidden the tail from the audit's single total order.

The rule: when a shared executor reads a table that also serves as
documentation, make the non-executable rows unrunnable in the executor, not
in the caller's copy of the table.

### A link: consumer under jest needs the linked package's nested deps transformed

dcw consumed an unpublished wallet-core through `link:../wallet-core` to
build DCW-74 (2026-09-09). Its jest suite then failed to load the package:
the linked tree resolves its own dependencies out of wallet-core's
`node_modules/.pnpm`, which jest's default `transformIgnorePatterns` skips,
and the linked package pulled a second `@babel/runtime` copy. Two config
lines fix it: add the linked package's nested `.pnpm` path to the transform
set, and map `@babel/runtime` back to the consuming project's copy. Both
are inert once the dependency comes from the registry again, so they can
stay until the next `link:` needs them.

The rule: a `link:` reference changes module resolution for the test
runner as well as for tsc, so check the runner's transform and mapping
config before reading a load failure as a code defect.

### A revocation reads the document to explain a refusal, not to skip the POST

Freewallet's connected-apps revocation (deac604, 2026-09-10) skipped the
revocation POST for any grant its verified account document read as dead:
an orphaned signer, a struck parent delegation, a swapped-out generation.
The same day, wallet-core e0cee42 moved the generation delegation's
revocation the other way: the document a login read is a snapshot, a ladder
VM struck by one ceremony can stand again on the server by the time the
revocation runs, and a client clock ahead of the server's reads a live
delegation as dead. So the only local skip is a delegation expired beyond a
ten-minute skew margin; everything else is POSTed, and the document is read
afterward to classify a plain `ValidationError` (expired, orphaned,
signer-gone, generation-swapped) or to rethrow one the client cannot
explain. Moving freewallet's predicate into wallet-core (WC-227) surfaced
the split, and the move took the POST-then-classify policy so both wallets
hold one.

The rule: a revocation POST is the only proof a grant is off the account.
Read the verified document to explain the server's refusal, and skip the
POST only for a grant no server within the skew margin still honors. Keep
that policy in wallet-core, so a wallet cannot drift to its own.

### A canonical-URL change reaches the test fixtures before it reaches the code

WAS v0.5 made the trailing-slash form of a container URL canonical
(was-client 0.61.0), and freewallet's FW-523 sweep onto it (2026-09-13)
touched four source files the roadmap item had named. It also touched
fourteen test files the item had not. A zcap `invocationTarget` is asserted
on all over a suite: unit fixtures carry stored management capabilities that
a target comparison now reads as stale, e2e specs assert
`invocationTarget.endsWith('/collection')`, join a resource URL onto a grant
target with a literal slash, and intercept requests with a Playwright route
pattern (`**/space/*`) whose `*` does not match across the new trailing
slash. Each failure reads as a product defect until the fixture is looked
at.

A second trap sits in the inverse direction. Deriving a server base URL back
out of a Space URL with was-client's `parseSpacePath` drops a sub-path
deployment: the parser sees the deployment's own base path (`/was/`) as a
first path segment and refuses the whole URL, so every grant built from it
goes unsatisfiable on exactly the deployments a bare-origin test suite never
exercises. Split at the `/space/` boundary and hand the prefix back as the
server URL, or use `parseSpaceTarget`, which takes the server URL and does
this itself.

The rule: when a wire-level URL form moves, grep the test tree for the old
form before running anything -- suffix assertions, string joins, and route
glob patterns all encode it -- and treat a sub-path deployment as a case the
suite cannot see, so re-derived base URLs need their own check.

### A server answers every URL under its mount, and none it does not own

Filed 2026-09-14 from freewallet's first signup against a WAS v0.5
freewallet.cloud. The spec's decision 0006 finds the service description
through a `Link: rel="service"` header on every response the server sends,
404s included, so a client may start from any URL it holds. was-client
started from the one URL it held, the configured base, and the origin root
there was a static nginx landing page: no `Link`, no CORS headers, and the
whole signup blocked on the first cross-origin `HEAD`. Every other path on
that host, `/does-not-exist` included, answered from the app with both
headers.

The guarantee is about responses the server sends, and nothing makes the
server the thing that answers its base URL. On a sub-path mount the app owns
its base and the probe works, which is why no test suite saw it. A base URL
at the origin root is the one case where "the server's URL" and "a URL the
server owns" come apart.

The rule: discover from a URL the server itself must answer, never from the
bare base. The wallets take the Spaces Repository URL as their configured
server address, derive the base as its parent, discover once from the
Spaces URL, and hand the description to every client they build through
was-client's `serviceDescription` option. A client that is left to discover
on its own re-derives the base-URL assumption.

### A spec anchor named in an export manifest is permanent wire text

Filed 2026-09-15 from the freewallet backup-bundle design (FW-530). The
Space export archive follows the FEP-6fcd manifest pattern, where every
entry in `contents` carries a `url` pointing at the documentation of
that entry's format. The server fills those in with WAS spec section
URLs (`#spaces`, `#collection-data-model`, `#resource-data-model`,
`#policy`, `#resource-metadata-data-model`), and the wallet bundle
layered over it points at the profile spec's own anchors. Every
archive ever exported carries those strings, and nothing rewrites an
archive on disk. A heading reword that moves an anchor breaks every
existing archive's self-description, silently, since a dangling
fragment still fetches the page.

The fragile part is that a spec editor sees a heading, not a wire
value. ReSpec derives an id from the heading text unless the heading
pins one, so the default is that a reword moves the anchor.

The rule: any spec whose section URLs are written into stored artifacts
(a manifest, a descriptor, a `url` member) pins the id on each such
heading explicitly, the way the WAS spec's `{#authorization}` style
does, and says in one place which of its anchors are depended on this
way. A new artifact format that names spec anchors adds its anchors to
that list in the same change. Name the anchor after the concept, not
the heading wording, so the heading stays free to change.

### A was-client bump past 0.62.0 drags the teaching server with it

Filed 2026-09-16 from was-sync WS-14. was-client 0.62.0 made service
discovery mandatory: every signed request first reads the `rel="service"`
link off an unsigned `HEAD` of the server URL, and a server that carries
none is refused with `IncompatibleServerError` before anything is signed.
was-sync's integration suite pinned `was-teaching-server@^0.29.0`, which
predates WAS v0.5, so the moment its was-client devDependency moved off
0.60.0 all fourteen integration tests failed at client construction, with
an error naming the server rather than the change that caused it.

Any repo whose tests run against an in-process was-teaching-server is
really pinning a pair, not two independent versions. Raising the
was-client devDependency across 0.62.0 raises the server floor to 0.30.0
at the same time, in the same change. The symptom to recognize:
integration tests failing wholesale with `IncompatibleServerError` right
after a client bump, while every unit test stays green -- the unit
suites drive fake ports and never discover a service.

### A refusal raised before the request is permanent, and a retry loop starves on it

Filed 2026-09-16 from was-sync WS-15. The example that taught it is now
historical: was-client 0.66.0 added a fail-closed affordance gate that threw
`NotSupportedError` before any request when a write named `ifMatch` /
`ifNoneMatch` against a collection whose backend advertised no
`conditional-writes`. Nothing about a second attempt changed the backend's
feature list, so every consumer that wrapped WAS writes in a retry-with-backoff
loop had a hole: the batch was re-sent forever and every row behind it was
starved, while the logs showed a generic write failure on repeat. That gate is
being removed under was-client WCL-106 and was-sync WS-16, since conditional
writes became a baseline requirement (see the entry below), and wallet-core
WC-237 was withdrawn 2026-09-16 as obsolete.

Two rules survive, for any client error raised ahead of the request rather
than mapped from a response. First, a retrying consumer needs a give-up branch
for it, and the branch belongs wherever the retry handle lives -- not in the
write path, which usually holds no handle to stop. In was-sync the push handler
only classified and logged the refusal; the controller found the name under
RxDB's error wrapping and released that one collection, leaving its siblings
replicating. Second, the classification goes in the owning package's predicate
set (an `isXError` on the relevant subpath) rather than as a name string
hard-coded in each consumer, so the WC-64 rule stays one contract with one
owner.

The symptom to recognize: a push or write cycle that never advances against one
collection while the rest of the Space syncs. It is invisible in a test setup
where every collection carries the affordance, so no integration suite will
surface it on its own.

### A guarantee a client must build around cannot be an optional affordance

Decided 2026-09-16 (WAS decision 0007, driven by the removal of the Backend
`features` vocabulary). Layering works when the layer is something a server
does or does not serve: listing, a query profile, a change feed. A client that
finds it absent does without it. Layering fails when the layer is a guarantee
the client builds correctness on, such as compare-and-swap, or a stored stamp
whose absence surfaces late as undecryptable rows. The client cannot
substitute a missing guarantee; it can only fall back by silently weakening
what it promised the user. In was-client that fallback cost roughly 525 source
and 730 test lines, including an insert with two implementations and a
metadata patch that dropped its pin, plus a give-up path in every retrying
consumer downstream.

The placement test for any token: could a server honestly support it on one
storage engine and not another? Because the client never addresses an engine
directly, the server is a serializing point that sees every write, so it can
mint its own validator, keep its own change log, or maintain its own index
over any engine. Nothing varies per backend. A token therefore lives
server-wide in the service description, or under the version entry of the
companion spec whose optional affordance it names, and a Backend description
is identity, operator, and persistence only.

## Current follow-ups

- Seed further entries from the older per-repo lessons as they resurface;
  this file was created 2026-08-15.
