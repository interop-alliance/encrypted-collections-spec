# Encrypted Collections for Wallet Attached Storage -- Specification

This repo is the **WAS Encrypted Collections (WAS-EC)** specification, a W3C
CCG-style document authored in **ReSpec + Markdown**. It defines the
client-side encrypted-collection construction over Wallet Attached Storage:
the encryption descriptor with its key-epoch roster (carried from the
collection's creation -- the single epoch era), the EDV envelope format with
the AEAD-bound `was` binding, epoch-configuration integrity (`epochsMac`),
recipient management, and deferred minting for local-first writers.

This is a **companion spec layered on top of [WAS]** (which owns the storage
model, HTTP API, and the server-visible halves: descriptor shape rails and
epoch stamp surfaces) and a sibling of **[APP-CONNECT]** (which defers to
this profile for the encrypted-collection construction and defines the
Resource Log Profile this spec's log-form section builds on).

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
  `DID-KEY`, `EDV`) and the `<div data-include="spec.md">` include. The
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
  (`was-epoch-config/v1.`), the fixed member order of the config JSON, and
  every value in the (reserved) pinned-inputs registry are baked into
  stored artifacts. Verify against code before changing a single
  character.
- **Altitude rule:** state invariants, not current implementations ("the
  owner is always a recipient", never "recipient zero is the vault KAK").
- **Fail-closed extensibility** throughout: anything unrecognized is
  rejected, not ignored.
- Wallet terminology: follow the global `clientId` / `writerId` rules --
  never "device" for either concept.

## Reference material (read-only, outside this repo)

These are separate repositories. Ground spec prose against real behavior --
check with the user before editing anything in them.

- [wallet-attached-storage-spec](https://github.com/w3c-ccg/wallet-attached-storage-spec)
  -- the WAS spec this profile layers on; its `spec.md` is the source of
  truth.
- [app-connect-spec](https://github.com/interop-alliance/app-connect-spec) --
  the sibling companion spec (Resource Log Profile, recipient-derivation
  invariant, epoch-roster recipient term).
- [was-client](https://github.com/interop-alliance/was-client) -- `src/edv/`
  is the reference implementation of this profile (EdvCodec, epochCrypto,
  epochMac, recipients, epochKeys).
- [wallet-core](https://github.com/interop-alliance/wallet-core),
  [freewallet](https://github.com/interop-alliance/freewallet), dcw (private
  repo) -- the wallet layers consuming it.
- [storage-core](https://github.com/interop-alliance/storage-core) -- the
  wire types (`CollectionEncryption` and friends).
