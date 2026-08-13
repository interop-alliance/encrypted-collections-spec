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
| [[WAS]]                                                        | The storage substrate: Spaces, Collections, Resources, the `encryption` descriptor slot, the Encryption Scheme Registry's `edv` envelope validation, conditional writes, the `changes` feed. This profile's server-visible halves ([descriptor shape rails](https://w3c-ccg.github.io/wallet-attached-storage-spec/#epoch-data-model), the epoch stamp surfaces, the [`blinded-index` query profile](https://w3c-ccg.github.io/wallet-attached-storage-spec/#query-profile-blinded-index) with its unique-attribute enforcement) are defined there.                                                                                |
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

  <dt><dfn data-lt="blinding keys">blinding key</dfn></dt>
  <dd>The key that makes queries over an encrypted collection possible
  without telling the server what is being searched for. In a blinded
  query, the client never sends a plaintext attribute name or value.
  It HMACs each one into an opaque token, and the server matches
  stored tokens against queried tokens. The server learns which
  [=envelopes=] matched, but not what the query meant.
  Concretely, the blinding key is the single HMAC-SHA-256 key under
  which an [=indexable=] collection's index tokens are blinded: a
  32-byte secret carried by the descriptor's `hmac` member, wrapped to
  each [=recipient=] like an [=epoch secret=]. Installed when the
  collection is provisioned and never rotated
  ([[[#blinded-index]]]).</dd>

  <dt><dfn data-lt="indexable">indexable collection</dfn></dt>
  <dd>An [=encrypted collection=] whose descriptor carries a
  [=blinding key=], so its envelopes may carry blinded index tokens
  and its readers may query by them ([[[#blinded-index]]]).</dd>

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
* `hmac` (OPTIONAL) - The collection's [=blinding key=]: its permanent
  id and type, and the wrap of the blinding secret to each recipient
  ([[[#blinding-key]]]). Present from the descriptor's creation or
  never, and never rotated ([[[#blinding-key-lifecycle]]]).

The descriptor's `scheme` and `version` are set-once for the life of the
collection, per [[WAS]], whose [Epoch data
model](https://w3c-ccg.github.io/wallet-attached-storage-spec/#epoch-data-model)
and
[Encryption Scheme Registry](https://w3c-ccg.github.io/wallet-attached-storage-spec/#encryption-scheme-registry)
define the server-visible rails over these members (shape validation, `epochs`
append-only enforcement, `currentEpoch` monotonicity, and the structural
fail-closed rejection of plaintext writes). The rails do not cover `hmac`:
the server stores and returns that member opaquely, with no shape
validation of its own ([[[#blinding-key]]]). A descriptor MUST NOT be
combined with a server-side plaintext-index (`indexes`) declaration --
the server cannot extract attributes from an opaque envelope; an
encrypted collection indexes by blinded tokens instead
([[[#blinded-index]]]).

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

On an [=indexable=] collection, each of these operations also
maintains the [=blinding key=]'s wraps ([[[#blinding-key-lifecycle]]]).

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
"jwe": ..., "indexed": ... }` (with `indexed` optional;
[[[#indexed-entries]]]), whose `jwe` is a
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
outside the ciphertext. The one plaintext-derived exception is the
opaque blinded tokens of an [=indexable=] collection's `indexed`
member ([[[#indexed-entries]]]), from which no attribute name or
value is recoverable.

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

## The blinded index {#blinded-index}

An [=encrypted collection=] MAY be [=indexable=]: selected attributes
of its documents are blinded into opaque HMAC tokens, so the server
can match equality queries without learning attribute names or
values. The construction is the [[EDV]] blinded-index model. Three
parts are client territory and defined here: the [=blinding key=] the
descriptor distributes ([[[#blinding-key]]]), the `indexed` token
entries envelopes carry ([[[#indexed-entries]]],
[[[#blinded-tokens]]]), and the encrypted index schema
([[[#index-schema]]]). The query wire shape and the server's matching
are the server-visible half, defined by [[WAS]]'s [`blinded-index`
query
profile](https://w3c-ccg.github.io/wallet-attached-storage-spec/#query-profile-blinded-index).
Serving queries is a server feature (the `blinded-index-query`
backend feature of [[WAS]]); an indexable collection stored on a
server without it is fully functional, merely unqueryable remotely.

### The blinding key {#blinding-key}

An [=indexable=] collection's descriptor carries the OPTIONAL `hmac`
member, a JSON object:

* `id` (REQUIRED) - A non-empty string identifying the blinding key,
  opaque and permanent for the life of the collection (the reference
  construction mints a `urn:uuid:` URN). Every `indexed` entry and
  every query names the key by this id.
* `type` (REQUIRED) - The string `Sha256HmacKey2019`. Any other value
  MUST be refused; a reader never infers the blinding algorithm from
  anything but this member.
* `recipients` (REQUIRED) - A non-empty array of recipient entries in
  exactly the shape of an epoch's ([[[#recipient-entry]]]): the raw
  32-byte blinding secret, key-wrapped to each recipient's
  [=key-agreement key=]. One wrap construction serves both kinds of
  secret; there is no separate blinding-key wrap format to review.

The blinding secret MUST be 32 freshly generated random bytes. Like
an [=epoch secret=], it MUST NOT be, or be derived from, any
longer-lived key or any other collection's key ([[[#first-epoch]]]),
and nothing secret appears in the descriptor.

Recipiency of the blinding key is granted by the same roster
operations that grant epoch recipiency
([[[#blinding-key-lifecycle]]]). A declared `hmac` member from which
a client can resolve no secret -- no entry names its
[=key-agreement key=], or the entry that does fails to unwrap -- is a
typed failure, exactly as for an epoch wrap ([[[#reads]]]). Indexing
and querying fail closed: the client MUST NOT downgrade to writing
un-indexed envelopes on an [=indexable=] collection, which would
write envelopes no query can find. Plain reads do not involve the
blinding key.

### Fixed at provisioning {#blinding-key-lifecycle}

Indexability is a property fixed at the collection's birth, and the
blinding key never changes afterward:

* The key MUST be minted in the same provisioning act as the first
  epoch ([[[#first-epoch]]]) and wrapped to the same initial
  recipients. It MUST NOT be added to a descriptor that already
  carries an epoch roster without one. A retro-fitted key would leave
  every envelope written before it without tokens, silently absent
  from every query, so mid-life addition is refused rather than
  half-honored.
* The install shares the first epoch's adoption rule: an `hmac`
  member already present in the descriptor is adopted as-is, never
  overwritten. Exactly one blinding key ever exists per collection.
* The key never rotates -- not on epoch rotation, not on recipient
  removal. Blinded tokens MUST compare across the collection's whole
  history; a successor key would split every query at the
  switchover, with a re-blind of the whole history the only repair.
* The roster operations of [[[#roster-operations]]] maintain the
  blinding key's `recipients` alongside the epochs'. Adding a
  recipient MUST wrap the blinding secret to it, idempotently.
  Removing a recipient drops the leaver's wrap entry as housekeeping
  only: the key itself is unchanged, and the operation MUST NOT mint
  a replacement key ([[[#blinding-key-removal]]] states the
  asymmetry this accepts). Replacing a recipient's key composes the
  two, as for epochs.

A descriptor without an `hmac` member describes a collection that is
not indexable. That is a conforming state, not an error; only a
request to make such a collection indexable after the fact is
refused.

<div class="note" title="Why mid-life indexing is refused">
In a plaintext document store an index is a server-side optimization:
the server can backfill it over data it can read, and an incomplete
index degrades to a slower scan, never to wrong answers. Neither
property holds here. Tokens are computed client-side over plaintext
the server never sees, so retro-fitting a blinding key would require
a client-side sweep re-encrypting the collection's entire history.
And the blinded index is the only query mechanism over ciphertext:
there is no scan fallback, so a partially indexed collection answers
queries with a silent subset that looks complete. Allowing the key
with forward-only indexing would bake that correctness failure in
permanently; allowing it with a backfill sweep would demand a
completion guarantee no party can give, since writers on cached
descriptors, including local-first writers offline for long periods,
keep producing un-indexed envelopes during and after the sweep. The
refusal trades that machinery for an invariant a verifier can
actually check.
</div>

Under the log form ([[[#log-form]]]) each entry's `state` carries the
full descriptor, `hmac` included, so the entry proof covers the
member exactly as it covers the epoch configuration -- and
fixed-at-provisioning becomes a claim a verifier can
check across the entry chain: one permanent `id`, present from the
first entry or absent from all. On pure point state the member has no
client-side guard of its own (the epoch pin does not cover it), and a
host serving a substituted `hmac` member knows its secret, so it can
test guessed attribute values against the reader's subsequent query
tokens. The log form is what closes that, as with any configuration
substitution ([[[#epoch-integrity]]]).

### Indexed entries {#indexed-entries}

A content envelope of an [=indexable=] collection MAY carry the
EDV `indexed` member ([[[#stored-envelope]]]): the blinded tokens the
server matches queries against. The structure is the [[EDV]]
Encrypted Document's `indexed` array, reused verbatim, with this
profile's cardinalities:

* At most one entry, a JSON object whose `hmac.id` and `hmac.type`
  MUST equal the descriptor `hmac` member's `id` and `type` -- one
  blinding key per collection, so one entry per envelope.
* The entry's `sequence` is the envelope's own `sequence`.
* The entry's `attributes` is an array of `{ "name": ..., "value":
  ..., "unique": ... }` objects (`unique` optional), one per blinded
  token pair, computed as [[[#blinded-tokens]]] defines. An attribute
  with `"unique": true` claims per-collection uniqueness of its token
  pair; enforcing the claim is the server-visible half, the
  unique-blinded-attributes rule of [[WAS]]'s `blinded-index`
  profile.

An envelope of a collection with no blinding key MUST NOT carry
`indexed`. Neither does any metadata envelope -- a resource metadata
envelope and the Collection Metadata envelope alike MUST NOT carry
the member. Only content envelopes are indexed; queries match
content, and the index schema itself rides the Collection Metadata
envelope ([[[#index-schema]]]).

`indexed` lives outside the JWE, in the clear, and that is its
function: the server stores and matches it without being able to
invert a token. It is consequently outside the AEAD and the `was`
binding. A reader MUST NOT treat `indexed` as authenticated: query
results are candidate envelopes, and integrity rests where it always
does, on each envelope's decrypt and binding verification
([[[#binding-verification]]]). A tampering server can suppress or
misreport matches -- a fetch-axis failure ([[[#two-axes]]]) no
per-envelope construction detects -- but cannot forge an envelope
that decrypts.

### Token computation {#blinded-tokens}

Every writer of a collection MUST blind identically, or queries stop
matching history. The choices below are therefore permanent
byte-level values, baked into every stored token. Throughout, `HMAC`
is HMAC-SHA-256 [[RFC2104]] under the collection's 32-byte blinding
secret, `||` is byte concatenation, and `base64url` is unpadded
[[RFC4648]] (section 5).

An indexed attribute is named by a dotted path into the sealed
plaintext document, rooted at `content` (the document's content) or
`meta` (the document's own inner type descriptor: `contentType` and
`encoding`). The path never addresses the WAS resource metadata
slot, which is a separate envelope ([[[#was-binding]]]). A literal
`.` inside a path segment is escaped as `\.`, and the name digest
below is taken over the full path string as written, escapes
included. For a simple attribute with path `name` holding plaintext
value `value`:

* The name digest is `SHA-256(UTF-8(name))` -- the path string
  itself, uncanonicalized.
* The value digest is `SHA-256(UTF-8(JCS(value)))`, where JCS is the
  JSON Canonicalization Scheme [[RFC8785]]. The value is
  canonicalized as a JSON value: a string digests with its quotes, a
  number in its canonical JSON form.
* The name token is `base64url(HMAC(name digest))`.
* The value token is `base64url(HMAC(SHA-256(name digest || value
  digest)))`. Folding the name digest in salts the value, so equal
  values under different attribute names produce unrelated tokens.

An attribute whose value is an array blinds each element as its own
token pair.

A compound index -- an ordered list of paths declared as one index
([[[#index-schema]]]) -- blinds one token pair per leading prefix of
the list, so a query may constrain any prefix. The prefix's name
digest is `SHA-256(name digest 1 || ... || name digest k)`, and its
value digest `SHA-256(value digest 1 || ... || value digest k)`;
both feed the same HMAC-and-salt step as a simple attribute's
digests. Two boundary rules keep writer and querier aligned:

* A one-member prefix is blinded in this compound
  (digest-of-digests) form -- unless the same path is also declared
  as a simple index, in which case the simple form already covers it
  and the compound form is not emitted.
* `unique` is carried only by the full-length prefix. A shorter one
  cannot claim the whole combination's uniqueness.

Multi-valued members spread combinatorially, one token pair per
combination of element values.

### The index schema {#index-schema}

Which attributes a collection indexes -- and with which `unique` and
compound declarations -- is configuration every writer and every
later-added reader must agree on: a writer blinds exactly the
declared attributes at write time, and a reader can only query what
it knows is declared. This profile persists that schema encrypted,
inside the Collection Metadata envelope ([[[#was-binding]]]), as the
`indexSchema` member of the collection metadata's `custom` object:

* `revision` (REQUIRED) - A non-negative integer, incremented by each
  declaration write; the empty schema is revision `0`.
* `indexes` (REQUIRED) - An array of declarations, each a JSON
  object:
  * `attribute` (REQUIRED) - A dotted path (a simple index) or an
    array of dotted paths (a compound index), per
    [[[#blinded-tokens]]]. A one-member array MUST be normalized to
    the bare path: it declares the same simple index, never a
    one-member compound (the two spellings blind differently, so
    only one may exist).
  * `unique` (OPTIONAL) - `true` to declare the index unique.
  * `addedIn` (REQUIRED) - The `revision` at which the declaration
    was added.

Declaring an index appends an entry and increments `revision`,
through the conditional-write mechanics of the Collection Metadata
surface (its validator and `If-Match`, per [[WAS]]), so concurrent
declarations serialize. Declarations are add-only in this profile.
Re-declaring an existing index on identical terms is a no-op;
changing a declaration's `unique` in place MUST be refused -- the
server has been enforcing, or not enforcing, the claim for every
envelope already written.

`addedIn` is the partial-coverage marker: envelopes written before a
declaration carry no tokens for it, so matches on a later-added
attribute MAY be partial over the collection's earlier history. A
reader that needs full coverage backfills by re-writing each earlier
envelope's `indexed` entries under the current schema -- a
client-side sweep of the whole history, the same cost class that
makes rotating the blinding key untenable
([[[#blinding-key-lifecycle]]]).

A reader MUST NOT blind a query for an attribute the persisted schema
does not declare: there is nothing for the tokens to match, and
failing closed keeps a typo from reading as an empty collection. An
absent or empty `indexSchema` on an indexable collection means
nothing is declared yet -- indexable, but nothing to query. A
persisted `indexSchema` that does not conform to this shape, or a
non-conforming entry within one, yields no declarations rather than
a refusal: the schema is discovery metadata riding a shared `custom`
object, and the fail-closed guard is the undeclared-attribute
refusal above.

The schema rides the encrypted metadata envelope, never the plaintext
Collection Description, because attribute names and index structure
reveal the collection's data model ([[[#index-schema-sensitivity]]]).

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

When a grant admits a party as a [=recipient=], that party's
[=key-agreement key=] is not taken from the request. It is derived from
the controller DID the grant already names: an Ed25519 `did:key` DID
[[DID-KEY]] has a canonical X25519 twin, and that twin is the recipient
key. The wallet and the grantee run the same derivation independently
over the same DID and land on byte-identical results -- the
key-agreement key itself never transits -- which is what makes the
`kid` the wallet writes into the
roster ([[[#recipient-entry]]]) match the `kid` the grantee computes for
itself (the grantee alone additionally derives the matching private
half, from its own Ed25519 secret).

The derivation is defined over a bare Ed25519 `did:key` DID
(`did:key:z6Mk...`) and nothing else. An input outside that domain fails
the derivation, and the failure is a refusal: the operation admitting
the recipient MUST NOT proceed ([[APP-CONNECT]] terms such a grant
unsatisfiable). Out-of-domain inputs include a key id (a DID with a
fragment -- the derivation is defined over the DID itself), a
non-Ed25519 `did:key` (an X25519 `did:key:z6LS...` carries no Edwards
point to convert), and a DID of any other method (a `did:web` does not
carry its key material in the identifier).

The conversion, byte-exactly:

1. Strip the `did:key:` prefix. The remainder is the controller's
   `publicKeyMultibase`: `"z"` followed by base58btc [[BASE58]] of the
   multicodec header `0xed 0x01` followed by the 32-byte Ed25519 public
   key. Decode it; a missing `z`, a wrong multicodec header, or bytes
   that do not decode to a valid Ed25519 curve point fail the
   derivation (a refusal, as above).
2. Map the Ed25519 point to its Montgomery (X25519) twin by the
   [[RFC7748]] birational map: `u = (1 + y) / (1 - y) mod 2^255 - 19`,
   where `y` is the point's Edwards y-coordinate; encode `u` as 32
   little-endian bytes.
3. Re-encode under the X25519 multicodec header:
   `"z" + base58btc(0xec 0x01 || u)`, where `||` is byte concatenation
   -- a `z6LS...` string, the recipient key's `publicKeyMultibase`.

The derived key is identified as:

* `id` - The controller DID with the derived `publicKeyMultibase` as
  its fragment: `did:key:z6Mk...#z6LS...`. This key id is the roster
  `kid` of every recipient entry naming the party
  ([[[#recipient-entry]]]).
* `type` - `X25519KeyAgreementKey2020`.

Both members describe the derived key object; a recipient entry names
the key by its `kid` alone and carries no `type` member
([[[#recipient-entry]]]).

<div class="note" title="Two key-id shapes in one descriptor">
A descriptor carries two `did:key...#...` shapes that are easy to
confuse. An epoch key id is self-referential, `did:key:z6LS...#z6LS...`
-- the DID part is the X25519 [=epoch key=] itself ([[[#epoch-id]]]). A
recipient `kid` is cross-curve, `did:key:z6Mk...#z6LS...` -- the DID
part is the Ed25519 controller, the fragment its derived X25519 twin.
The DID part of a key id therefore always states which entity the key
belongs to.
</div>

An implementation MUST NOT accept a recipient key supplied on the wire,
in any form; the key-agreement key derives from the named controller
DID and from nothing else. An explicitly supplied key would let a
request pair controller DID A with recipient key B, and the granting
side would have to verify the binding anyway -- which for a `did:key`
means performing this exact derivation. Deriving makes key substitution
impossible by construction and keeps both axes of a share -- the
capability and the roster entry -- pointing at the same entity.
[[APP-CONNECT]] states this invariant at its consent surface and defers
the derivation itself to this section.

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

### The blinding key survives removal {#blinding-key-removal}

Removing a recipient rotates the epoch but never the
[=blinding key=] ([[[#blinding-key-lifecycle]]]), so a removed recipient keeps
the ability to compute blinded tokens forever. Against the server
alone this is harmless: the server gates the query profile behind a
capability the removed reader no longer holds. The residual is
collusion. A server willing to run the removed recipient's queries,
or to hand it the stored `indexed` entries, lets it confirm guessed
attribute values against any envelope in the collection, past and
future writes alike. The asymmetry is accepted by design -- the
alternative, rotating the blinding key on removal, would orphan the
tokens of the collection's whole history -- and it grants no
content, only equality tests against values the remover already
guessed. Documentation and libraries built on this profile MUST
state the asymmetry alongside the limitations of
[[[#rotation-limitations]]].

### What the index reveals {#index-schema-sensitivity}

Blinded tokens hide names and values but not structure. The server
learns, by design, which envelopes carry equal values for equal
attributes (matching is token equality), how many attributes each
envelope indexes, and which tokens a querying reader asks about;
`unique` declarations additionally mark which attributes claim
collection-wide uniqueness. Deployments should index the few
attributes they query, not everything they store.

The schema itself is sensitive for the same reason: attribute names
and compound structure describe the collection's data model. That is
why `indexSchema` lives inside the encrypted Collection Metadata
envelope ([[[#index-schema]]]) rather than in the plaintext
Collection Description, and why the descriptor's `hmac` member
carries a key id and wrapped bytes only, never attribute names.
