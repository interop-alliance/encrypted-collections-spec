## Introduction {#introduction}

This document defines the Encrypted Collections profile for Wallet Attached
Storage ([[WAS]]): the client-side construction by which a WAS
[collection](https://w3c-ccg.github.io/wallet-attached-storage-spec/#dfn-collections)'s
contents are end-to-end encrypted under rotatable key epochs,
so that several readers can hold per-reader keys, reader removal has a
cryptographic meaning, and a storage server never holds key material.

The profile is a client-side convention layered on the WAS storage substrate.
A conforming WAS server needs nothing beyond the features it already
advertises: it stores opaque envelopes, validates their non-secret shape
(the `edv` scheme of the WAS [Encryption Scheme
Registry](https://w3c-ccg.github.io/wallet-attached-storage-spec/#encryption-scheme-registry)),
and serves the
non-secret descriptor and epoch stamps. Every normative requirement in this
profile therefore binds clients -- writers and readers of encrypted
collections -- except where a requirement is explicitly identified as the
server-visible half defined by [[WAS]].

| Specification                                                  | Relationship                                                                                                                                                                                                                                                                                                                                                                           |
|----------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [[WAS]]                                                        | The storage substrate: Spaces, Collections, Resources, the `encryption` descriptor slot, the Encryption Scheme Registry's `edv` envelope validation, conditional writes, the `changes` feed. This profile's server-visible halves ([descriptor shape rails](https://w3c-ccg.github.io/wallet-attached-storage-spec/#epoch-data-model), the epoch stamp surfaces) are defined there.                                                                                |
| [[APP-CONNECT]]                                                | The companion profile by which an application obtains capabilities into a wallet's Space. It defers to this profile for the encrypted-collection construction: the epoch roster, the envelope format, recipient-key derivation, and the definition of an epoch-roster recipient. Its Resource Log Profile defines the log form of the resources this profile shapes ([[[#log-form]]]). |
| [Encrypted Data Vaults](https://identity.foundation/edv-spec/) | The envelope vocabulary: the Encrypted Document shape, the JWE recipients structure, and the blinded-index model this profile's envelopes reuse verbatim.                                                                                                                                                                                                                              |

### Key epochs and eras {#epochs-and-eras}

A [=key epoch=] is one generation of a collection's encryption key. Rather
than encrypting every resource in a collection under a single long-lived key,
this profile encrypts each resource under the key of whichever epoch was
current when the resource was written. The collection's descriptor carries
the full roster of epochs (an append-only history), and each epoch records
which [=recipients=] hold a wrap of that epoch's secret.

Informally: you can think of the [=current epoch=] as "the set of keys,
held by clients and people, that can decrypt what is written to this
collection today". Each earlier epoch is the same answer for an earlier
stretch of the collection's history, so the epoch roster as a whole reads
as the collection's access history.

Epochs are what give key rotation and reader removal their meaning:

* Rotating a collection's key means appending a new epoch, not re-encrypting
  history. Resources written earlier remain sealed under the epochs of their
  writing; new writes seal under the new [=current epoch=].
* Removing a reader means omitting them from the new epoch's recipients.
  They cryptographically lose access to everything written from that epoch
  onward, while their access to earlier epochs is unchanged: ciphertext
  they could once decrypt is not retroactively protected from them by a
  key change alone.

An <dfn data-lt="eras">era</dfn>, by contrast, is not a per-collection
concept but a design regime: a span of the profile's life during which
stored artifacts follow one key-management construction, such that a reader
would need era-specific logic to tell which construction a given artifact
uses. Multiple eras force every future reader to carry branches for every
past construction -- and each branch is a downgrade surface. This profile
avoids that cost outright by having exactly one era, as the next section
defines.

### The single epoch era {#single-epoch-era}

This profile has exactly one [=era=]. An encrypted collection's
[descriptor](https://w3c-ccg.github.io/wallet-attached-storage-spec/#collection-data-model)
carries a key-epoch roster from the moment the collection exists, and every
envelope seals to an epoch key and binds the epoch it sealed under. There is
no descriptor-less era, no epoch-less envelope, and consequently no
acceptance branch for either anywhere in the profile:

* A descriptor declaring the `edv` scheme without an epoch roster is
  non-conforming ([[[#epoch-less-descriptor]]]).
* An envelope without the `was` binding, or without `was.epoch`, is refused
  ([[[#was-binding]]]).
* A reader's own key-agreement key is never a decryption candidate; reads
  route through resolved epoch keys only ([[[#reads]]]).

## Terminology {#terminology}

<dl class="termlist definitions" data-sort="ascending">
  <dt><dfn data-lt="encrypted collections|encrypted">encrypted
  collection</dfn></dt>
  <dd>A WAS
  [collection](https://w3c-ccg.github.io/wallet-attached-storage-spec/#dfn-collections)
  whose Collection Description declares the `edv`
  encryption scheme and whose resources are stored as [=envelopes=]. Its
  descriptor carries an epoch roster from creation
  ([[[#single-epoch-era]]]).</dd>

  <dt><dfn data-lt="descriptor|encryption descriptors">encryption
  descriptor</dfn></dt>
  <dd>The non-secret `encryption` member of a collection's Collection
  Description: the scheme, the scheme version, and the epoch roster
  ([[[#descriptor]]]).</dd>

  <dt><dfn data-lt="epoch|epochs|key epochs">key epoch</dfn></dt>
  <dd>One generation of a collection's encryption key. Each epoch wraps its
  [=epoch secret=] to every [=recipient=]; rotating appends a new epoch
  rather than re-encrypting history.</dd>

  <dt><dfn data-lt="epoch keys">epoch key</dfn></dt>
  <dd>An X25519 [[RFC7748]] key-agreement key pair derived from an epoch's
  [=epoch secret=]. Envelopes written under the epoch name it as their sole
  JWE recipient.</dd>

  <dt><dfn data-lt="epoch secrets">epoch secret</dfn></dt>
  <dd>The fresh 32-byte symmetric secret shared by an epoch's
  [=recipients=]. It never encrypts content directly: it seeds the epoch's
  [=epoch key=] (a holder reconstructs the full key pair, and with it both
  encrypts and decrypts under the epoch). Freshly generated per epoch,
  never derived from or equal to any longer-lived key
  ([[[#first-epoch]]]).</dd>

  <dt><dfn>current epoch</dfn></dt>
  <dd>The epoch named by the descriptor's `currentEpoch` member: the one new
  writes encrypt under.</dd>

  <dt><dfn>epoch configuration</dfn></dt>
  <dd>The core of a descriptor: its `scheme`, `version`, `currentEpoch`,
  and the ordered list of epoch ids. Its integrity against a tampering
  host is the subject of [[[#epoch-integrity]]].
  </dd>

  <dt><dfn data-lt="recipient|recipients|epoch-roster recipients">epoch-roster recipient</dfn></dt>
  <dd>A party, identified by the `kid` of its own [=key-agreement key=],
  holding a wrap of an epoch's secret to that key: an entry in that epoch's
  `recipients` array. Recipiency is the read
  (decrypt) axis of an encrypted collection, distinct from the fetch axis a
  WAS capability governs.</dd>

  <dt><dfn data-lt="envelopes">envelope</dfn></dt>
  <dd>The stored form of a resource in an encrypted collection: an Encrypted
  Data Vault [Encrypted
  Document](https://identity.foundation/edv-spec/#dfn-encrypteddocument)
  [[EDV]] whose `jwe` seals the plaintext to an
  [=epoch key=] and binds the `was` protected-header parameter
  ([[[#envelope]]]).</dd>

  <dt><dfn data-lt="key-agreement keys|KAK">key-agreement key</dfn></dt>
  <dd>A party's own X25519 key, identified by the `kid` its recipient
  entries carry. Epoch secrets are wrapped to it; envelopes never are
  ([[[#single-epoch-era]]]).</dd>

  <dt><dfn data-lt="collection owner">owner</dfn></dt>
  <dd>The controller of the [=Space=] a [=collection=] lives in. Always a
  [=recipient=] of every encrypted collection in their own Space; any
  departure is an explicit consent surface, never a silent default.</dd>

  <dt><dfn data-lt="collections">collection</dfn></dt>
  <dd>A WAS
  [Collection](https://w3c-ccg.github.io/wallet-attached-storage-spec/#dfn-collections)
  [[WAS]].</dd>

  <dt><dfn data-lt="Spaces">Space</dfn></dt>
  <dd>A WAS
  [Space](https://w3c-ccg.github.io/wallet-attached-storage-spec/#spaces)
  [[WAS]].</dd>
</dl>

## The encryption descriptor {#descriptor}

### Members {#descriptor-members}

An [=encrypted collection=]'s [=encryption descriptor=] is a JSON object with
the following members. All of `scheme`, `epochs`, and `currentEpoch` are
REQUIRED from the descriptor's creation onward; there is no state of a
conforming descriptor in which any of them is absent.

* `scheme` (REQUIRED) - The string `edv`.
* `version` (OPTIONAL) - The positive-integer version of the `edv` envelope
  wire format, per the WAS [Encryption Scheme
  Registry](https://w3c-ccg.github.io/wallet-attached-storage-spec/#encryption-scheme-registry).
  An absent `version` means `1`. This document describes version `1`.
* `epochs` (REQUIRED) - A non-empty array of epoch entries
  ([[[#epoch-entry]]]), append-only across the descriptor's life.
* `currentEpoch` (REQUIRED) - The `id` of the epoch new writes encrypt
  under. MUST name an entry in `epochs` and never moves to an older epoch.

The descriptor's `scheme` and `version` are set-once for the life of the
collection, and a descriptor MUST NOT be combined with a server-side
`indexes` declaration -- both per [[WAS]], whose [Epoch data
model](https://w3c-ccg.github.io/wallet-attached-storage-spec/#epoch-data-model)
and
[Encryption Scheme Registry](https://w3c-ccg.github.io/wallet-attached-storage-spec/#encryption-scheme-registry)
define the server-visible rails over these members (shape validation, `epochs`
append-only enforcement, `currentEpoch` monotonicity, and the structural
fail-closed rejection of plaintext writes).

### The first epoch {#first-epoch}

Declaring a collection encrypted and installing its first epoch are one
provisioning act. A writer that provisions an encrypted collection MUST
ensure, before the collection's first content write, that its descriptor
carries an epoch roster; the reference construction performs this as a
two-step -- a crypto-free container ensure, then an EDV-bearing
create-if-absent epoch install -- with these properties, all REQUIRED:

* The [=epoch secret=] of the first epoch (and of every later epoch) MUST
  be freshly generated for that epoch. It MUST NOT be, or be derived from,
  any longer-lived key -- not the [=owner=]'s key-agreement key, not any
  account-level or wallet-level key, not a key any other collection uses.
  A rotation or an external share can wrap an epoch secret to a new
  recipient at any time; a secret that is also a longer-lived key would
  hand that recipient the longer-lived key.
* The first epoch MUST wrap to at least one recipient, and the [=owner=]
  MUST be among the initial recipients of a collection provisioned in
  their own Space.
* The install MUST be create-if-absent and adopting: a descriptor that
  already carries epochs -- including one written by a concurrent
  provisioner that won the guarded create or compare-and-swap race -- MUST
  be adopted as-is, never overwritten. Re-running the install against a
  settled collection performs only reads. Exactly one first epoch ever
  exists. (A client library MAY additionally expose an explicit
  initialization spelling that refuses, rather than adopts, an existing
  roster; it MUST NOT overwrite one.)
* A torn provisioning run (the collection declared encrypted, the epoch
  install not yet landed) MUST be repairable by re-running the same
  install. No checkpoint state beyond the descriptor itself is required to
  detect completion.

### The descriptor precedes content {#descriptor-first}

An encrypted collection's descriptor, with its epoch roster, MUST be
published before the collection's first content push, so that no envelope
reaches the collection sealed under an epoch the published descriptor does
not carry.

A writer that mints envelopes eagerly against a cached descriptor and then
loses the descriptor create to another provisioner MUST adopt the winner's
descriptor and re-encrypt any pending envelopes the adopted roster cannot
route under the winner's current epoch before pushing them. This is legal
precisely because pending envelopes have no server-side existence: an
envelope that has reached the collection is immutable and is never
re-encrypted in place, but a local, never-pushed envelope may be re-minted
freely ([[[#deferred-minting]]]).

### An epoch-less descriptor is refused {#epoch-less-descriptor}

A descriptor whose `scheme` is `edv` but which carries no `epochs` roster
does not describe a state this profile admits. A client encountering one:

* MUST NOT construct a cipher for the collection -- in particular it MUST
  NOT fall back to sealing envelopes directly to any reader's
  key-agreement key;
* MUST treat the state as a torn provisioning to repair by re-running the
  first-epoch install ([[[#first-epoch]]]), or as a served-descriptor
  integrity failure, whichever its role implies;
* MUST fail closed (no reads, no writes) until an epoch-bearing descriptor
  is obtained.

## Key epochs {#epochs}

### Epoch entries {#epoch-entry}

Each entry of `epochs` is a JSON object:

* `id` (REQUIRED) - The epoch's identifier: the `did:key` DID of the
  epoch's public key-agreement key ([[[#epoch-id]]]).
* `recipients` (REQUIRED) - A non-empty array of recipient entries
  ([[[#recipient-entry]]]), one per [=recipient=] of this epoch.

`epochs` is append-only: an epoch, once written, is never removed and its
`id` never changes, so resources stored under it remain readable. Entries
within an existing epoch's `recipients` MAY change (adding a recipient
wraps existing epoch secrets to it, [[[#roster-operations]]]).

### Epoch identifiers {#epoch-id}

An epoch's `id` is the `did:key` DID [[DID-KEY]] of the epoch's public
X25519 key-agreement key -- a `did:key:z6LS...` string. The epoch key's
full key id is that DID with itself as the fragment,
`did:key:z6LS...#z6LS...`, and it is this key id that appears as the JWE
recipient `kid` of every envelope sealed under the epoch.

Two consequences are load-bearing:

* An envelope names its epoch structurally: the JWE recipient `kid` up to
  the `#` is the epoch `id`, resolvable by any standard `did:key`
  resolver. Envelopes are roster-blind -- they name the epoch key, never
  any per-recipient key.
* Epoch ids are collision-free rather than ordinal. Two independently
  minted epochs are two distinct epochs; no counter coordination exists or
  is needed.

### Recipient entries {#recipient-entry}

Each entry of an epoch's `recipients` array reuses the JWE General
Serialization recipient shape ([[RFC7516]] section 7.2) verbatim:

* `header` (REQUIRED) - A JSON object with at least:
  * `kid` (REQUIRED) - The recipient's key-agreement key id.
  * `alg` (REQUIRED) - `ECDH-ES+A256KW`.
  * `epk`, `apu`, `apv` - The ephemeral public key (`{ "kty": "OKP",
    "crv": "X25519", "x": ... }` [[RFC8037]]) and the agreement info of
    the wrap:
    `apu` is the unpadded base64url ephemeral public key, `apv` the
    unpadded base64url UTF-8 of the recipient's `kid` (all base64url in
    the entry, `x` and `encrypted_key` included, is unpadded).
* `encrypted_key` (REQUIRED) - The wrapped payload: the raw 32-byte
  [=epoch secret=], key-wrapped to the recipient's key-agreement key
  (ephemeral-static ECDH, the [[RFC7518]] Concat KDF, [[RFC3394]] AES
  key wrap).

Nothing secret appears in the descriptor: recipient entries hold public
key identifiers and wrapped-key ciphertext only. The server never holds
key material, never unwraps, and never verifies that a wrap is correct --
that is unverifiable without the keys, by construction.

### Roster operations {#roster-operations}

All descriptor mutation goes through conditional writes on the Collection
Description (`If-Match` compare-and-swap; guarded create), per [[WAS]]. The
operations:

* The first-epoch install, defined in [[[#first-epoch]]].
* Adding a recipient MUST wrap every epoch's secret to it (adding a reader
  admits it to the collection's history -- the escrow decision), MUST be
  idempotent, and leaves the [=epoch configuration=] untouched (recipient
  entries are not part of it).
* Removing a recipient MUST mint a fresh epoch wrapped to each remaining
  recipient of the current epoch (never the union of historical
  recipients), append it, repoint `currentEpoch` -- and only then revoke
  the removed reader's
  capabilities. Rotate first, then revoke, as one indivisible removal: a
  client library SHOULD NOT expose a rotate-without-revoke or
  revoke-without-rotate spelling of removal.
* Replacing a recipient's key -- retiring one key-agreement key in favor
  of its successor, the operation behind key-rotation cascades -- composes
  the two in one conditional write: the successor key is escrowed into
  every epoch like an add, and a fresh epoch excluding the retiring key is
  appended like a removal. History stays readable to the successor; the
  retiring key is sealed out of everything written after.

On a precondition failure (a concurrent descriptor write won), the client
re-reads the descriptor, re-applies its change to the fresh state, and
retries, bounded.

## Epoch-configuration integrity {#epoch-integrity}

A descriptor hosted by a storage server gets its shape rails from the
server, but shape rails cannot stop a malicious host from serving a
fabricated or rolled-back epoch configuration. Two client-side guards
close that gap, neither of them carried in the descriptor itself:

* Client-side monotonic state -- an epoch or head pin -- makes a served
  configuration that rolls back behind the pin refusable
  ([[[#configuration-replay]]]).
* The log form of the descriptor ([[[#log-form]]]) makes the whole
  [=epoch configuration=] proof-carrying: each entry's Data Integrity
  proof establishes that the configuration was written by a writer the
  deployment's root of trust vouches for -- so a host-minted
  configuration fails verification -- and the hash chain with a pinned
  head makes any rollback a chain break.

A deployment on pure point state keeps the server's shape rails
(append-only `epochs`, monotonic `currentEpoch`) and the pin; only the
log form additionally establishes authorship against the host itself.

<div class="note" title="The retired epochsMac member">
An earlier revision of this profile carried a REQUIRED descriptor member,
`epochsMac`: an HMAC over the epoch configuration, keyed via HKDF from
the current epoch secret. It has been retired. On a log-governed
descriptor its coverage is a strict subset of chain verification (the
entry proof covers the full epoch configuration), and its classic gaps
-- whole-configuration replay under an old matching MAC, and fresh
fabrication under a newly minted secret -- were gaps with or without it.
Conforming descriptors do not carry the member.
</div>

## The envelope {#envelope}

### The stored envelope {#stored-envelope}

A resource in an encrypted collection is stored as an Encrypted Data Vault
Encrypted Document: the JSON object `{ "id": ..., "sequence": ...,
"jwe": ..., "indexed": ... }` (with `indexed` optional), whose `jwe` is a
JWE in JSON Serialization carrying the sealed plaintext. The stored
content type is `application/jose+json` where the server parses it, with
`application/json` as the portable default an unmodified WAS server
accepts. The envelope shape and the server's structural fail-closed
validation of it are the `edv` scheme registration of [[WAS]]'s Encryption
Scheme Registry.

The plaintext document carried inside the JWE declares its own inner
content type: `{ "contentType": ... }` for JSON content, plus
`"encoding": "utf-8"` for text or `"encoding": "base64"` for binary.
Nothing user-visible -- content, plaintext type, user metadata -- appears
outside the ciphertext.

Every envelope names exactly one JWE recipient: the [=epoch key=] of the
epoch it sealed under ([[[#epoch-id]]]). A conforming writer MUST NOT
name any other recipient -- per-reader access rides the epoch roster, not
per-envelope wraps -- and MUST NOT seal an envelope to any key that is not
an epoch key ([[[#single-epoch-era]]]).

### Algorithms {#algorithms}

* Key agreement and wrap: `ECDH-ES+A256KW` only, over X25519 [[RFC8037]],
  with the [[RFC7518]] Concat KDF (SHA-256).
* Content encryption: `XC20P` (XChaCha20-Poly1305 [[XCHACHA]], the
  extended-nonce variant of [[RFC8439]]) is the profile
  conforming writers use; readers additionally accept `C20P` [[RFC8439]]
  and `A256GCM` [[RFC7518]] (the upstream EDV FIPS suite) on decrypt
  only. A fresh
  content-encryption
  key is generated per envelope, so envelope bytes are nondeterministic
  across re-encryptions of identical plaintext -- deduplication keys on
  decrypted content identity, never on envelope bytes or resource id.

### Content-derived resource ids {#content-ids}

An immutable (content-addressed) collection derives each resource id from
the envelope's ciphertext after encryption:
`"z" + base58btc(0x00 0x10 || first 16 bytes of SHA-256(base64url-decode(jwe.ciphertext)))`
(base58btc per [[BASE58]]).
Because encryption is randomized, the id identifies the stored envelope,
not the plaintext; two replicas independently encrypting identical
plaintext produce distinct resource ids, and convergence is defined on
decrypted content, not on envelope bytes.

### The `was` binding {#was-binding}

Every envelope MUST carry a `was` parameter in its JWE protected header,
bound into the AEAD (so a successful decrypt proves it authentic):

* `v` (REQUIRED) - The `edv` scheme version the envelope was written
  under (the descriptor's `version`; `1` in this document).
* `epoch` (REQUIRED) - The `id` of the [=epoch=] the envelope sealed
  under: the writer's current epoch at encrypt time.
* `resource` (OPTIONAL) - The resource id the envelope is bound to.
  Present whenever the id is known at encrypt time; absent only when no
  resource id exists to bind: a content-derived id, which does not exist
  until after encryption ([[[#content-ids]]]), or a Collection Metadata
  envelope, which belongs to no resource.
* `collection` (OPTIONAL) - The Collection id the envelope is bound to:
  the Collection's `id`, the `{collection_id}` path segment of [[WAS]]'s
  URL templates. Present on exactly one envelope kind, the Collection
  Metadata envelope; an envelope in any resource slot -- content or
  resource metadata -- MUST NOT bind it.

A resource metadata envelope (the encrypted `custom` object of a
resource's metadata, per [[WAS]]) binds `was` exactly like a content
envelope, with `resource` always present -- bound to the resource's id,
not to the metadata envelope's own internal document id -- and `epoch`
the write epoch like every write.

A Collection Metadata envelope (the encrypted `custom` object of the
Collection's own metadata, per [[WAS]]) belongs to no resource. It MUST
bind `collection` -- the id of the Collection it was written for -- and
MUST NOT bind `resource`; `epoch` is the write epoch like every write.
Each of the profile's slots is thereby declared positively, by the
member set of its envelope: `resource` marks a resource-slot envelope,
`collection` marks the Collection Metadata envelope, and a
content-derived content envelope binds neither -- which is exactly why
the Collection Metadata slot requires a present `collection` rather than
merely a missing `resource` ([[[#binding-verification]]]).

### Binding verification {#binding-verification}

After every successful decrypt, and before the plaintext is used, a reader
MUST verify the `was` binding. The checks, in order, all unconditional:

1. A missing `was` parameter is refused: every envelope of this profile
   binds `was` at encrypt time, so its absence means a writer this scheme
   does not admit. There is no unbound-envelope acceptance
   ([[[#single-epoch-era]]]).
2. A `was.v` greater than the scheme version the reader implements is
   refused: a future-scheme envelope this reader does not understand.
3. When the read targeted a known resource id (a content or resource
   metadata read): a present `was.collection` is refused before any id
   comparison (the Collection's metadata envelope was served in a
   resource slot). A string `was.resource` MUST equal the targeted id (a
   mismatch is a server-side swap of two resources' envelopes); without
   a string `was.resource` the envelope is treated as a content-derived
   write, and the id re-derived from its ciphertext ([[[#content-ids]]])
   MUST equal the targeted id (a mismatch means the envelope was served
   under an id it was not written for).
4. When the read targeted the Collection Metadata slot: a present
   `was.resource` is refused (a resource's envelope was served in the
   Collection's slot), and a string `was.collection` MUST be present and
   equal the id of the Collection the read addressed. A missing
   `was.collection` means an envelope of some other slot -- notably a
   content-derived content envelope, whose member set is otherwise
   identical -- and a mismatched one is one Collection's metadata served
   as another's.
5. A missing or non-string `was.epoch` is refused like a missing `was`.
   Present, it MUST equal the epoch of the key that actually decrypted
   the envelope (the `did:key` before the `#` of that key's id); a
   mismatch is a replay of the envelope under a different epoch's key.
   The check has no epoch-less carve-out: there is no epoch-less envelope
   to admit.

The first two failures are scheme refusals (the envelope is outside the
profile); the rest are integrity failures (the server misrepresented
what it stored). A reader SHOULD keep the two classes distinct in its
error taxonomy and MUST NOT mask either as a routine decryption miss.

## Writes and reads {#writes-reads}

### Writes {#writes}

* A write MUST encrypt under the descriptor's `currentEpoch` and bind it
  as `was.epoch`.
* A content write SHOULD declare its epoch to the server (the
  `Key-Epoch` request header, per [[WAS]]), so listings and the `changes` feed can
  serve the epoch stamp and a replicating reader can select its key
  before fetching the envelope. The server-served stamp is advisory
  routing metadata; the authoritative binding is the envelope's own
  AEAD-bound `was.epoch`.
* Inserts use guarded creates (`If-None-Match: *`) and updates use
  `If-Match`, per the collection's mutability class and [[WAS]]'s
  conditional requests.

### Reads {#reads}

A reader resolves its epoch keys from the descriptor: for each epoch
whose `recipients` name the reader's [=key-agreement key=], unwrapping
the [=epoch secret=] and reconstructing the [=epoch key=]. The write
epoch is `currentEpoch` when the reader holds it, and MUST be unwrapped
eagerly (write readiness: a wrap that fails to unwrap surfaces as its
typed failure at key resolution, not at the first write); other held
epochs MAY unwrap lazily on first use. A reader that does not hold
`currentEpoch` -- a removed or archive reader, whose writes the server
rejects via its revoked capability anyway -- selects as its write epoch,
deterministically, the last epoch in the descriptor's canonical `epochs`
order that names it, never the incidental order in which secrets
happened to unwrap.

Decryption routes by the stored envelope's JWE recipient `kid` against
the resolved epoch keys, and only those:

* The reader's own key-agreement key is NEVER a decryption candidate. An
  envelope sealed to anything other than a resolved epoch key is
  unroutable, whatever key it names ([[[#single-epoch-era]]]).
* An envelope naming an epoch the reader's cached descriptor does not
  carry is the unknown-epoch signal. Because an epoch rotation is a
  Collection Description change and emits no `changes`-feed entry, a
  reader holding a stale cached descriptor can meet envelopes stamped
  with an unseen epoch; the remedy is exactly one descriptor re-read,
  key-resolution rebuild, and retry -- guarded to once per collection per
  session, so a genuinely foreign envelope cannot drive a refetch loop.
  Still unknown after the refresh, the envelope is refused as unroutable.
* A failed unwrap of a recipient entry that names the reader is a typed
  failure, never a silently absent key: key servers whose unwrap resolves
  a null key on mismatch are surfaced as the typed failure, not treated
  as "try the next candidate succeeded".
* An AEAD authentication failure after a successful key unwrap is an
  integrity failure of the stored envelope. It MUST be surfaced as such
  and MUST NOT be masked as a key miss or routine decryption error.

A reader that is not a recipient of any epoch of a collection fails
closed with a recipiency error -- the read axis is absent, whatever its
capability (fetch axis) status.

## Deferred minting {#deferred-minting}

The single epoch era implies one obligation on writers: no envelope
exists before its collection's
descriptor is known. A writer MUST NOT mint an envelope for a collection
whose epoch-bearing descriptor it does not hold -- there is nothing
conforming to seal to.

This is deliberately NOT a constraint on local writes. The local-first
invariant this profile preserves in full is:

> A local write is never blocked by the crypto or sync stack. Envelopes
> are a replication artifact, minted when (and under whichever keys) the
> collection's descriptor is known -- not a precondition of the write.

A conforming local-first writer therefore defers minting: local writes
land in local plaintext storage immediately and unconditionally; envelope
minting happens at replication time, when the descriptor is held (the
unlinked-row, late-mint pattern -- rows carry no envelope until a sweep
mints them under the real descriptor). A writer that mints eagerly
against a cached descriptor follows the adopt-and-re-mint rule of
[[[#descriptor-first]]] instead. Both patterns satisfy the same
invariant: no envelope ever reaches the collection sealed under an epoch
the published descriptor does not carry.

<div class="note">
One consequence of "no envelope before the descriptor" is a reconnect
asymmetry. A writer holding a cached descriptor for a collection can mint
offline and push the moment connectivity returns; a writer without one
(a collection it has never synced, or a cache lost to storage clearing)
must first fetch the descriptor, so its deferred rows replicate on the
following sync cycle rather than the first. This is an ordering
asymmetry in replication latency only -- never a lost write, and
invisible to the local writer.
</div>

## Point state and log form {#log-form}

An [=encryption descriptor=] as defined above is point state: the current
configuration, served whole, its rails enforced by the server. The
companion App Connect profile defines a Resource
Log Profile ([[APP-CONNECT]]) under which the same resources are governed
as hash-linked logs of full-state entries, each entry proof-carrying and
externally authorized -- and encryption descriptors are among its
referencing profiles' state schemas.

Under the log form:

* Each log entry's `state` carries the full descriptor at that version,
  under a `type` member identifying its schema. The rules of this profile
  apply to the `state` of the verified head entry exactly as they apply
  to a served point-state descriptor: same members
  ([[[#descriptor-members]]]), same first-epoch and append-only rules,
  same fail-closed refusal of an epoch-less state.
* Epoch history and log history are distinct axes. `epochs` remains
  append-only WITHIN each state (a resource-level invariant a verifier
  can check across consecutive entries), while the log records the
  succession of whole configurations -- so "epochs is append-only" and
  "currentEpoch never moves backwards" become verifiable claims over the
  entry chain rather than server-enforced rails.
* Authorship of a configuration is established by the entry's Data
  Integrity proof, verified against the deployment's root of trust per
  the Resource Log Profile's external-authorization rule; the entry
  proof covers the full [=epoch configuration=], and the hash chain
  with a pinned head makes any rollback a chain break -- the log form
  is what carries this profile's epoch-configuration integrity
  ([[[#epoch-integrity]]]).
* A point-state descriptor served beside a log is a projection bound to
  the log per the Resource Log Profile: a verifying consumer acts only on
  a projection equal to the verified head's `state`.

Nothing in this profile's epoch rules depends on which framing carries
the descriptor; a deployment adopts the log form per collection, by the
referencing profile's rules.

## Recipient-key derivation {#recipient-derivation}

<div class="note" title="Reserved section">
Reserved. This section will define the byte-level derivation of a
recipient's key-agreement key from its controller DID (the
`did:key:z6Mk...` Ed25519-to-X25519 conversion producing the roster `kid`
`did:key:z6Mk...#z6LS...`), the guard rules, and the refusal of
explicitly supplied recipient keys. [[APP-CONNECT]] states the invariant
(derive, never accept) and defers the derivation itself to this profile.
</div>

## The user-key roster {#user-key-roster}

<div class="note" title="Reserved section">
Reserved. This section will define the account-level user-key roster: a
`CollectionEncryption`-shaped descriptor stored as a resource, whose
current epoch delivers the account's user key wrapped once per enrolled
client -- a delivery channel, never a source of authority -- together
with its epoch pin and document-backed recipient resolution rules. The
epoch-configuration integrity guards ([[[#epoch-integrity]]]) and the
log form ([[[#log-form]]]) apply to it unchanged.
</div>

## Pinned derivation inputs {#pinned-inputs}

<div class="note" title="Reserved section">
Reserved. This section will register the exact derivation inputs wallet
implementations of this profile pin permanently (KDF salts and info
strings, key-name strings),
each with its value and the artifact class it is baked into. Changing any
registered value orphans every artifact derived under it.
</div>

## Security considerations {#security-considerations}

### The limitations of rotation {#rotation-limitations}

Epoch rotation protects resources written after the rotation, and nothing
else. It never claws back data a removed reader already fetched;
resources stored under an earlier epoch remain readable to a removed
reader that obtains their ciphertext (a backup, a colluding reader, a
feed pull made before revocation); and it provides no post-compromise
security for the removed reader's past traffic. Closing those gaps
requires re-encrypting the collection under the new epoch -- a
client-side bulk rewrite outside this profile. Documentation and
libraries built on this profile MUST state these limitations rather than
imply stronger guarantees.

### Fetch and decrypt are separate axes {#two-axes}

The WAS capability governs fetch (server-enforced, immediate on
revocation); epoch recipiency governs decrypt (mathematics, prospective
only). Removing a reader means acting on both, rotate first
([[[#roster-operations]]]); neither alone suffices, and documentation and
error messages should never conflate them.

### No downgrade surface {#no-downgrade}

Because the binding checks of [[[#binding-verification]]] are
unconditional and the profile admits no descriptor-less or epoch-less
state, a tampering server has no fallback path to steer a reader onto: it
cannot serve a stripped descriptor to induce direct-to-key sealing
([[[#epoch-less-descriptor]]]), and it cannot serve an unbound envelope
and have it accepted.

### The scope of the collection binding {#collection-binding-scope}

`was.collection` binds the Collection's id, which [[WAS]] scopes to the
Space. Two collections carrying the same id in different Spaces are
therefore not distinguished by the binding; they are distinguished by
their keys alone (every epoch secret is fresh and per-collection,
[[[#first-epoch]]]), so serving one such collection's metadata envelope
as the other's fails as a key miss rather than as a named integrity
failure. The space id is deliberately not bound: it is server-relative,
and baking it into permanent envelope bytes would break decrypt across
replicas and Space re-homing, for no gain against the attack that
remains -- a server aliasing an entire collection, descriptor included,
which no per-envelope binding can detect and which is
descriptor-authenticity territory (the epoch pin and log form,
[[[#configuration-replay]]]).

### Rotation is feed-invisible {#rotation-feed}

An epoch rotation is a Collection Description change and emits no
`changes`-feed entry. The unknown-epoch refresh of [[[#reads]]] is the
required remedy, and its once-per-collection-per-session guard is what
keeps a foreign envelope from becoming a descriptor-refetch denial of
service.

### Configuration replay {#configuration-replay}

A served point-state descriptor carries nothing that lets a client
detect the replay of an entire prior consistent configuration -- the
server's rails bind its own storage, not what it chooses to serve.
Deployments needing rollback protection keep client-side monotonic
state (an epoch pin) or adopt the
log form, whose hash chain and pinned head make any rollback a chain
break ([[[#log-form]]]).
