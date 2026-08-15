# Ecosystem Map

The one-page view of the `@interop/*` / BYOE ecosystem: what exists, the
dependency direction, which document is authoritative for what, and where a
given kind of change goes. This map routes; the per-repo ARCHITECTURE.md and
AGENTS.md files hold the depth. Kept current by hand -- when a repo's role
or a dependency edge changes, update this map in the same working session.

Private repos are marked; everything else is public on GitHub
(interop-alliance, or w3c-ccg for the WAS spec). If you have a sibling repo
checked out locally, read it there instead of fetching -- but never assume a
particular checkout layout, and check with the maintainer before editing a
repo other than the one you are working in.

## Layer map (dependency direction, apps at the top)

```
Apps        freewallet (browser wallet)   dcw (mobile wallet, private)
            byoe-react-examples, life-advisor (private)  -- BYOE demo apps
            was-teaching-server (companion WAS + KMS server)
                 |                                |
App-side lib     |          was-react  (BYOE app half of App Connect,
                 |                      shared-collection reading)
                 |                                |
Shared wallet    +---------- wallet-core ---------+
layer                (ceremonies, keys, keyring, enrollment, recovery,
                      clients, request/App Connect, webvh, genesis)
                                  |
Storage/wire            was-client (+ /edv, /sync)
                     (WAS HTTP client, EDV cipher, key epochs, sync wire)
                                  |
                            storage-core
                     (shared WAS data-model types)

Side libraries (consumed where needed, no knowledge of the layers above):
  social-core (contacts specs + LWW), verifier-core (VC verification),
  webkms-client (CapabilityAgent, KMS), ezcap (ZcapClient),
  did-method-webvh, data-integrity-core (shape guards, VPR vocabulary),
  byoe-context (hosted JSON-LD contexts)

Foundation forks (low churn; ESM/isomorphism maintenance of upstream code):
  vc, jsonld-signatures, zcap, security-document-loader, did-method-key,
  did-web-resolver, ed25519-*, x25519-*, ecdsa-*, http-*, edv-client,
  minimal-cipher, lru-memoize, ...
```

Rules of the arrows: apps depend down, never sideways into another app;
wallet-core never imports from an app; was-client knows nothing of wallets
or ceremonies. was-react sits beside the wallets (it is the app-side
counterpart, not a wallet dependency). The server shares storage-core and
the zcap stack with the clients but consumes neither was-client nor
wallet-core.

## Specs and their implementations

| Spec repo                                              | Owns                                                                                     | Implemented by                                                                                      |
|--------------------------------------------------------|------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| wallet-attached-storage-spec (W3C CCG; co-edited here) | Space / Collection / Resource model, HTTP API, zcap profile                              | was-teaching-server (authoritative protocol description in its AGENTS.md), was-client, storage-core |
| app-connect-spec                                       | AppConnectQuery, app-key credential, descriptor vocabulary, action ceilings, response VP | wallet-core `/request`, was-react, both wallets                                                     |
| encrypted-collections-spec                             | Envelope cryptography, key epochs, recipient derivation                                  | was-client `/edv`, wallet-core `/keys`                                                              |

Each spec repo's AGENTS.md carries the "Parties to this contract" table --
the enforced consumer registry. A normative change's checklist is a walk of
the owning table; conformance-suite tests servers against the WAS spec.

## Which document is authoritative for what

- WAS protocol and authorization model: was-teaching-server AGENTS.md.
- The shared wallet layer (module layers, key hierarchy, ceremonies,
  cascades, permanent wire constants): wallet-core ARCHITECTURE.md.
- The browser wallet (session/auth flow, storage model, CHAPI, App Connect
  wallet-side, the pins and continuity guards): freewallet ARCHITECTURE.md.
- The BYOE app side (what an app runs against the wallet's grants):
  was-react README/docs.
- zcap mechanics (delegation, invocation, verification): the
  zcap-developer-guide repo.
- House conventions (roadmap schema, releasing, decision records, code
  style): isomorphic-lib-template AGENTS.md and CONTRIBUTING.md.
- Cross-repo lessons and this map: this repo (LEARNINGS.md, ECOSYSTEM.md).

## Ownership heuristics (where does this change go)

- A wallet ceremony, key-custody rule, or anything both wallets must do
  identically: wallet-core. "Changes DCW too" is a point in favor of
  putting it there, never app-side.
- Envelope/cipher/epoch mechanics, the sync wire contract, or WAS HTTP
  client behavior: was-client.
- A new WAS endpoint or authorization rule: the WAS spec first, then
  was-teaching-server, then was-client -- with the parties-table walk.
- App Connect wire shapes (query, credential, descriptors, ceilings):
  app-connect-spec, then wallet-core `/request` and was-react together;
  consent UI stays per-wallet.
- Contacts semantics shared across wallets: social-core (kept team-neutral;
  branded seeds stay app-side).
- Encoding/crypto primitives: never hand-rolled anywhere -- `@scure/base`
  for baseline codecs, the owning `@interop/*` package for algorithm
  building blocks. "Not reachable through the export map" means export it
  upstream, not vendor it.
- A fix needed inside any `@interop/*` package is an in-house change: edit
  the checkout, publish, consume from the registry (package.json `author`
  fields crediting upstream do not indicate current ownership).

## Standing cross-repo rules (pointers, not restatements)

- Cross-repo change discipline: the driving roadmap item carries `touches:`;
  breaking releases run the doc-vs-code audit (isomorphic-lib-template
  AGENTS.md "Releasing"); counterpart tests (e.g. was-react's
  `walletCoreCounterpart.test.ts`) stay green.
- Consume `@interop/*` packages from the npm registry; `link:`/`workspace:`
  overrides are temporary and end with a publish (see LEARNINGS.md on
  dependency-order publishing and the no-instanceof-across-packages rule).
- Cross-repo decisions get a `decisions/NNNN-slug.md` record in the owning
  repo (convention in isomorphic-lib-template `decisions/`).

## Caveats

- dcw and life-advisor are private; their rows in the parties tables are
  flagged as such, and this map's claims about them come from their
  AGENTS.md files, not public code.
- The foundation-fork list is illustrative, not exhaustive; the full fork
  inventory is considerably larger, and most of it is low-churn code that
  does not participate in the conventions rollout.
- This map states current state only; intent and open work live in the
  repos' own roadmaps, not here.
