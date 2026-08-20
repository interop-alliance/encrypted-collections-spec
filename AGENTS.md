# Encrypted Collections for Wallet Attached Storage -- Specification

This repo is the **WAS Encrypted Collections (WAS-EC)** specification, a W3C
CCG-style document authored in **ReSpec + Markdown**. It defines the
client-side encrypted-collection construction over Wallet Attached Storage:
the encryption descriptor with its key-epoch roster (carried from the
collection's creation -- the single epoch era), the EDV envelope format with
the AEAD-bound `was` binding, epoch-configuration integrity (the epoch
pin and the log form), the resource log profile (the hash-linked,
externally authorized log form under which descriptors and key rosters
are co-managed, extracted from did:webvh),
recipient management, and deferred minting for local-first writers.

This is a **companion spec layered on top of [WAS]** (which owns the storage
model, HTTP API, and the server-visible halves: descriptor shape rails and
epoch stamp surfaces) and a sibling of **[APP-CONNECT]** (which defers to
this profile for the encrypted-collection construction and for the
resource log profile, citing only its external-authorization rule for
enrolled clients).

The profile is implemented by **`@interop/was-client`** (`/edv` subpath) and
consumed through **`@interop/wallet-core`** by the **DCW** and **freewallet**
wallets.

This is a **spec repo, not a code repo**. The deliverable is the rendered
HTML document. There is no build/test/lint pipeline -- "correct" means the
prose is accurate, internally consistent, and renders cleanly in ReSpec.

## Files

- `spec.md` -- the entire normative specification (single file). This is what
  you edit 99% of the time.
- `index.html` -- ReSpec shell: config (`specStatus: "unofficial"`, GitHub
  repo, xref to `did-core`, `localBiblio` for `WAS`, `APP-CONNECT`,
  `DID-KEY`, `EDV`, `DID-WEBVH`, `JSON-LINES`) and the
  `<div data-include="spec.md">` include. The
  `<h1>`, abstract, SotD, and conformance sections live here, not in
  `spec.md`.
- `README.md` -- short; overview and local preview instructions.

## Preview locally

```
npx serve .
```

Then open the served `index.html`; ReSpec renders client-side. There is no
static build step.

## ReSpec / Markdown authoring conventions

Match the existing style (same conventions as the app-connect-spec repo):

- **Cross-references to sections:** `[[[#section-id]]]` renders as a live
  link with the section's title. Headings carry explicit ids
  (`### Reads {#reads}`) -- use those.
- **Term references:** `[=term=]` links to a `<dfn>` in the Terminology list
  (e.g. `[=epoch=]`, `[=recipient=]`, `[=envelope=]`, `[=owner=]`). Define
  new terms in the `## Terminology` `<dl>` with `<dfn data-lt="aliases">`.
- **Bibliography refs:** `[[WAS]]`, `[[APP-CONNECT]]`, `[[DID-KEY]]`,
  `[[EDV]]`, `[[RFC...]]` -- ReSpec auto-resolves (custom keys via
  `localBiblio` in `index.html`).
- **Headings map to numbered sections.** `##` = top-level, `###`/`####` nest.
- Honor the global rules: use `--` not an em dash, and `to` not an arrow.

## Design invariants (so edits stay consistent)

- **The single epoch era.** One era only: descriptors carry epochs from
  creation and every envelope binds `was.epoch`. Never introduce a
  legacy/fallback branch anywhere (no descriptor-less reader tolerance, no
  "missing `was` means accept it"). The spec deliberately contains no
  historical asides; do not add any.
- **Byte-level values are permanent.** The epoch-MAC HKDF info
  (`was-epoch-config-mac/v1`), the MAC payload prefix
  (`was-epoch-config/v1.`), the fixed member order of the config JSON,
  every value in the (reserved) pinned-inputs registry, and the resource
  log constants (the `resource-log:0.1` format identifier, the `{SCID}`
  placeholder, the bare-base58btc SHA-256 multihash entry hash, the
  `eddsa-jcs-2022` / `assertionMethod` proof shape) are baked into stored
  artifacts. Verify against code before changing a single character.
- **The resource log profile lives here, not in App Connect.** Its only
  normative dependency is the controller document; the mention of enrolled
  clients is an informative back-reference to [APP-CONNECT]. A normative
  WAS-EC-to-App-Connect reference in the log profile would re-create the
  dependency cycle the re-homing removed (see
  [decisions/0001](decisions/0001-resource-log-profile-home.md)).
- **Altitude rule:** state invariants, not current implementations ("the
  owner is always a recipient", never "recipient zero is the vault KAK").
- **Fail-closed extensibility** throughout: anything unrecognized is
  rejected, not ignored.
- Wallet terminology: follow the global `clientId` / `writerId` rules --
  never "device" for either concept.

## Parties to this contract

Every repo that implements or consumes this profile, with the specific
modules that speak it. **The maintenance rule: a normative change's checklist
is a walk of this table** -- for each row, resolve the impact as shipped
(naming what landed, including the row's ARCHITECTURE/AGENTS docs) or
explicitly waived (`unaffected: <repo> (<why>)`). A breaking change to the
construction is named as such in each implementing package's CHANGELOG.

| Repo | Modules speaking the contract |
| --- | --- |
| storage-core | The wire types (`CollectionEncryption` and friends; the resource log entry types and `RESOURCE_LOG_METHOD` in `resourceLog.ts`). |
| was-client | `src/edv/` is the reference implementation: the EDV envelope codec, epoch construction (`epochCrypto`, `epochKeys`), recipient operations, `x25519RecipientFromDidKey`, and the descriptor-store seam. `src/log/` is the resource log transport: JSON Lines codec and the CAS append / read-back store seam. |
| wallet-core | `/keys` (the user-key wrap-set roster stores a `CollectionEncryption` descriptor verbatim), the ceremonies that rotate epochs (`/clients`, `/recovery`), and `src/resourceLog/` (entry building, hashing, and chain verification against the controller document). |
| freewallet | Per-collection document ciphers over the local replica and remote-direct backends, app-provisioned collection encryption, share grants (`StorageManager.shareCollection`). |
| dcw (private) | The mobile wallet's document cipher and sync-side epoch handling, over the same was-client/wallet-core modules. |
| was-react | `SharedCollectionReader` and `createDocCipher` (epoch-aware reads as a grantee); its `test/node/sharedCollection.test.ts` exercises the real was-client `/edv` roster operations and recipient derivation. |
| was-teaching-server | The server-visible halves: descriptor shape rails and the `Key-Epoch` stamp surfaces. |
| was-conformance-suite | `encryption-descriptor-api` and the key-epoch checks in `client-spaces` validate the server-visible halves. |

## Ecosystem conventions

- Cross-repo lessons (invariants, gotchas, and process recipes that span
  repos) live in the ecosystem learnings file,
  [byoe-ecosystem/LEARNINGS.md](https://github.com/interop-alliance/byoe-ecosystem/blob/main/LEARNINGS.md)
  (usually checked out beside this repo as `../byoe-ecosystem`); read it at
  the start of any cross-repo task.
- Decisions about the contract this spec owns (profile and wire-contract
  decisions) are recorded in this repo's [decisions/](decisions/) directory,
  one `NNNN-slug.md` file per decision; the convention and template are
  canonical in
  [isomorphic-lib-template's `decisions/`](https://github.com/interop-alliance/isomorphic-lib-template/tree/main/decisions).

## Reference material (read-only, outside this repo)

These are separate repositories. Ground spec prose against real behavior --
check with the user before editing anything in them.

- [wallet-attached-storage-spec](https://github.com/w3c-ccg/wallet-attached-storage-spec)
  -- the WAS spec this profile layers on; its `spec.md` is the source of
  truth.
- [app-connect-spec](https://github.com/interop-alliance/app-connect-spec) --
  the sibling companion spec (recipient-derivation invariant, epoch-roster
  recipient term, the enrolled-client consumer of the resource log profile).
- [was-client](https://github.com/interop-alliance/was-client) -- `src/edv/`
  is the reference implementation of this profile (EdvCodec, epochCrypto,
  recipients, epochKeys).
- [wallet-core](https://github.com/interop-alliance/wallet-core),
  [freewallet](https://github.com/interop-alliance/freewallet), dcw (private
  repo) -- the wallet layers consuming it.
- [storage-core](https://github.com/interop-alliance/storage-core) -- the
  wire types (`CollectionEncryption` and friends).
