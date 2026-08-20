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
| [[WAS]]                                                        | The storage substrate: Spaces, Collections, Resources, the `encryption` descriptor slot, the Encryption Scheme Registry's `edv` envelope validation, conditional writes, the `changes` feed. This profile's server-visible halves ([descriptor shape validation](https://w3c-ccg.github.io/wallet-attached-storage-spec/#epoch-data-model), the epoch stamp surfaces, the [`blinded-index` query profile](https://w3c-ccg.github.io/wallet-attached-storage-spec/#query-profile-blinded-index) with its unique-attribute enforcement) are defined there.                                                                                |
| [[APP-CONNECT]]                                                | The companion profile by which an application obtains capabilities into a wallet's Space. It defers to this profile for the encrypted-collection construction: the epoch roster, the envelope format, recipient-key derivation, the definition of an epoch-roster recipient, and the [=resource log=] profile under which its enrolled clients co-manage key resources ([[[#resource-log-profile]]]). |
| [[DID-WEBVH]]                                                  | The log format the [=resource log=] profile is extracted from ([[[#log-webvh]]]), and one method a Space controller's [=controller document=] may be verified under. Nothing outside that profile depends on it. |
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

  <dt><dfn data-lt="user keys">user key</dfn></dt>
  <dd>The account-level secret the user-key roster delivers
  ([[[#user-key-roster]]]): the roster's current [=epoch secret=], minted
  fresh with each roster epoch, wrapped once per enrolled
  [=wallet client=]. Its derived key-agreement key ([[[#recipient-derivation]]])
  is how the account's [=owner=] is a [=recipient=] of the account's
  encrypted collections. Client-minted, held by no server, and
  derivable from no passphrase or seed.</dd>

  <dt><dfn data-lt="wallet clients|enrolled client|enrolled clients">wallet
  client</dfn></dt>
  <dd>The keyed, revocable identity of an (app, user) pair enrolled in an
  account: an Ed25519 signing key listed in the
  [=account controller document=], with its key-agreement key published
  under the controller
  marker ([[[#keyagreement-controller-marker]]]) and a wrap in the
  user-key roster. One machine hosts many clients (browser profiles,
  several apps, several accounts); a client is not tied to hardware.</dd>

  <dt><dfn data-lt="controller document|controller documents|log controller document">account controller
  document</dfn></dt>
  <dd>The DID document [[DID-CORE]] of the [=Space=]'s controller,
  independently resolved and verified by the client per the controller
  DID's method. The source of record for which [=wallet clients=] an
  account has and which key-agreement keys back the user-key roster's
  recipients ([[[#roster-recipients]]]), and the root of authority for
  every [=resource log=] ([[[#log-authorization]]]).</dd>

  <dt><dfn data-lt="collection owner">owner</dfn></dt>
  <dd>The controller of the [=Space=] a [=collection=] lives in. Always a
  [=recipient=] of every encrypted collection in their own Space; any
  departure is an explicit consent surface rather than a silent
  default.</dd>

  <dt><dfn data-lt="collections">collection</dfn></dt>
  <dd>A WAS
  [Collection](https://w3c-ccg.github.io/wallet-attached-storage-spec/#dfn-collections)
  [[WAS]].</dd>

  <dt><dfn data-lt="resources">resource</dfn></dt>
  <dd>A WAS
  [Resource](https://w3c-ccg.github.io/wallet-attached-storage-spec/#resources-and-blobs)
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
* `type` (OPTIONAL) - The string `WasEpochConfiguration`, the descriptor's
  schema identifier under the log form ([[[#log-form]]]). Inside a log
  entry's `state` the member is REQUIRED ([[[#log-entry]]]). A
  point-state projection served beside a log carries
  the same member, so that the projection equals the verified head's
  `state` once `history` is stripped. A descriptor no log governs MAY
  omit it.
* `epochs` (REQUIRED) - A non-empty array of epoch entries
  ([[[#epoch-entry]]]), append-only across the descriptor's life.
* `currentEpoch` (REQUIRED) - The `id` of the epoch new writes encrypt
  under. MUST name an entry in `epochs` and never moves to an older epoch.
* `history` (OPTIONAL) - The log-form dispatch hint ([[[#log-form]]]): an
  object with `resource`, the URL of the log resource governing this
  descriptor, and `method`, an echo of the log's format identifier,
  named to mirror the in-log `parameters.method`. The member appears only
  on the point-state projection of a log-governed collection, not
  inside a log entry's `state` -- the log resource reserves the
  name there ([[[#log-resource]]]). The hint is never authoritative: the log's
  own genesis commits the format identifier, and a served `method` that
  contradicts it is refused, not dispatched on.
* `hmac` (OPTIONAL) - The collection's [=blinding key=]: its permanent
  id and type, and the wrap of the blinding secret to each recipient
  ([[[#blinding-key]]]). Present from the descriptor's creation or
  never, and never rotated ([[[#blinding-key-lifecycle]]]).

The descriptor's `scheme` and `version` are set-once for the life of the
collection, per [[WAS]], whose [Epoch data
model](https://w3c-ccg.github.io/wallet-attached-storage-spec/#epoch-data-model)
and
[Encryption Scheme Registry](https://w3c-ccg.github.io/wallet-attached-storage-spec/#encryption-scheme-registry)
define the server-visible checks over these members (shape validation, `epochs`
append-only enforcement, `currentEpoch` monotonicity, and the structural
fail-closed rejection of plaintext writes). These checks do not cover `type`,
`history`, or `hmac`: the server stores and returns those members
opaquely, with no shape validation of its own ([[[#blinding-key]]] for
`hmac`). A descriptor MUST NOT be
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
  be adopted as-is and MUST NOT be overwritten. Re-running the install
  against a
  settled collection performs only reads. Exactly one first epoch ever
  exists. (A client library MAY additionally expose an explicit
  initialization operation that refuses, rather than adopts, an existing
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
  resolver. Envelopes are roster-blind -- they name the epoch key, not
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
  key wrap), byte-exactly the derivation of [[[#key-wrap-kdf]]].

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
  revoke-without-rotate variant of removal.
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

A descriptor hosted by a storage server gets its shape checks from the
server, but shape checks cannot stop a malicious host from serving a
fabricated or rolled-back epoch configuration. Two client-side guards
close that gap, neither of them carried in the descriptor itself:

* Client-side monotonic state -- an epoch or head pin -- makes a served
  configuration that rolls back behind the pin refusable
  ([[[#configuration-replay]]]).
* The log form of the descriptor ([[[#log-form]]]) makes the whole
  [=epoch configuration=] proof-carrying: each entry's Data Integrity
  proof establishes that the configuration was written by a writer the
  deployment's root of trust vouches for ([[[#log-authorization]]]) --
  so a host-minted configuration fails verification -- and the hash
  chain with a pinned head makes any rollback a chain break.

A deployment on pure point state keeps the server's shape checks
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
content type and encoding ([[[#plaintext-document]]]).
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

### The plaintext document {#plaintext-document}

The JWE plaintext is the UTF-8 JSON serialization of a document object
with the following members, and no others (a chunked document
additionally seals its `stream` state; see the `"chunked"` row below):

* `content` (REQUIRED) - The sealed content, in the shape `meta.encoding`
  selects (below).
* `meta` (REQUIRED on a content envelope) - A JSON object describing
  `content`:
  * `contentType` (REQUIRED) - The media type of the plaintext content,
    a string (`application/json` for JSON content written without a
    declared type).
  * `encoding` (OPTIONAL) - How `content` carries the plaintext. A
    writer MUST use exactly one of the values below, and a reader MUST
    refuse a value it does not recognize rather than fall back to any
    interpretation of `content`.

`meta.encoding` and `content` are paired as follows:

| `meta.encoding` | `content` |
| --- | --- |
| absent | The JSON value itself, verbatim: the caller's JSON object or array. |
| `"utf-8"` | `{ "text": ... }`: the plaintext as one JSON string. Used for text-family media types whose bytes are valid UTF-8. |
| `"base64"` | `{ "bytes": ... }`: the plaintext bytes in standard base64 ([[RFC4648]] section 4, the `+` and `/` alphabet, padded). This is distinct from the unpadded base64url of the JWE members (`jwe.ciphertext`, `epk.x`, `apu`, `apv`, `encrypted_key`); a reader MUST NOT decode the two with one alphabet. Used for binary content, and for text-family content whose bytes are not valid UTF-8. |
| `"chunked"` | `{}`: the document's bytes are not inline; they live in the resource's chunk resources ([[WAS]]'s chunked streams). The chunked document form is not otherwise specified by this document. |

An absent `meta.encoding` means JSON. A reader MUST select the `content`
shape from `meta.encoding` alone and MUST NOT infer an encoding from the
shape of `content`: a caller object that happens to be `{ "text": "..." }`
or `{ "bytes": "..." }`, written as JSON, round-trips as that JSON object.
The writer's `meta.encoding` is the only in-band discriminator, and it is
inside the AEAD.

A metadata envelope -- the encrypted `custom` object of a resource's or
the Collection's metadata ([[[#was-binding]]]) -- seals the `custom`
object itself as `content`, verbatim, and carries no `meta`: the
`custom` object is always JSON, and the absent-`encoding` rule above
applies. A reader of a metadata envelope MUST treat its `content` as the
`custom` object, whatever shape it has.

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
  decrypted content identity rather than on envelope bytes or resource
  id.
* Content-encryption key length: 32 bytes (256 bits), for `XC20P`, `C20P`,
  and `A256GCM` alike. The wrapped `encrypted_key` of an envelope recipient
  is therefore always a 40-byte AES key wrap output.

#### The key-wrap derivation {#key-wrap-kdf}

The `ECDH-ES+A256KW` wrap is used in two places with one construction:
sealing an envelope's content-encryption key to the [=epoch key=] (the
JWE recipient of [[[#stored-envelope]]]), and wrapping an [=epoch secret=]
to a reader's [=key-agreement key=] (a recipient entry of
[[[#recipient-entry]]]). In both, the shared secret `Z` is the X25519
agreement between a fresh ephemeral key pair and the static recipient
key, and the key-encryption key is derived by the [[RFC7518]] Concat KDF
(section 4.6.2) with SHA-256, in a single round, from exactly these
inputs:

* `AlgorithmID` - The UTF-8 bytes of the string `ECDH-ES+A256KW` (the
  `alg` value, not the content-encryption `enc`), 14 bytes.
* `PartyUInfo` - The raw 32 bytes of the ephemeral X25519 public key.
* `PartyVInfo` - The UTF-8 bytes of the static recipient key's id: the
  string carried as the recipient's `kid` (the epoch key id for an
  envelope, [[[#epoch-id]]]; the reader's key-agreement key id for a
  roster entry).
* `SuppPubInfo` - The key data length, 256, as a 32-bit big-endian
  integer.
* `SuppPrivInfo` - Empty.

Each of `AlgorithmID`, `PartyUInfo`, and `PartyVInfo` is encoded as its
byte length, a 32-bit big-endian integer, followed by the bytes
themselves ([[RFC7518]] section 4.6.2's `Datalen || Data` form). The
hashed input is thus

```
uint32be(1) || Z || uint32be(14) || "ECDH-ES+A256KW"
           || uint32be(32) || ephemeral public key
           || uint32be(len(kid)) || kid
           || uint32be(256)
```

and the 256-bit SHA-256 output is, without truncation or further
derivation, the [[RFC3394]] AES key wrap key.

The inputs travel in the recipient `header`: `epk` is the ephemeral
public key as a JWK (`{ "kty": "OKP", "crv": "X25519", "x": ... }`
[[RFC8037]]), `apu` is the unpadded base64url of `PartyUInfo`, and
`apv` is the unpadded base64url of `PartyVInfo`; every base64url value
in the header, `x` and `encrypted_key` included, is unpadded. A writer
MUST emit `epk`, `apu`, and `apv`. An unwrapper derives `PartyUInfo`
from `epk.x` and `PartyVInfo` from its own knowledge of the key the
`kid` names -- it does not take either from `apu` or `apv`, which are
carried for interoperability with a generic [[RFC7518]] recipient and
are not inputs a reader trusts. A header whose `apu` or `apv` disagrees
with the recomputed values therefore fails as an unwrap failure, not as
a distinct refusal.

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
2. A `was.v` that is absent, or present but not a JSON number, is
   refused like a missing `was`: an envelope that does not declare its
   scheme version is outside the profile. There is no default version to
   assume.
3. A `was.v` greater than the scheme version the reader implements is
   refused: a future-scheme envelope this reader does not understand. The
   version the reader implements is the one the collection's descriptor
   declares (its `version`, [[[#descriptor-members]]]), which the reader
   has already confirmed it supports before building any cipher.
4. When the read targeted a known resource id (a content or resource
   metadata read): a present `was.collection` is refused before any id
   comparison (the Collection's metadata envelope was served in a
   resource slot). A string `was.resource` MUST equal the targeted id (a
   mismatch is a server-side swap of two resources' envelopes); without
   a string `was.resource` the envelope is treated as a content-derived
   write, and the id re-derived from its ciphertext ([[[#content-ids]]])
   MUST equal the targeted id (a mismatch means the envelope was served
   under an id it was not written for).
5. When the read targeted the Collection Metadata slot: a present
   `was.resource` is refused (a resource's envelope was served in the
   Collection's slot), and a string `was.collection` MUST be present and
   equal the id of the Collection the read addressed. A missing
   `was.collection` means an envelope of some other slot -- notably a
   content-derived content envelope, whose member set is otherwise
   identical -- and a mismatched one is one Collection's metadata served
   as another's.
6. A missing or non-string `was.epoch` is refused like a missing `was`.
   Present, it MUST equal the epoch of the key that actually decrypted
   the envelope (the `did:key` before the `#` of that key's id); a
   mismatch is a replay of the envelope under a different epoch's key.
   The check has no epoch-less carve-out: there is no epoch-less envelope
   to admit.

The first three failures, and the absence of `was.epoch` in the last,
are scheme refusals (the envelope is outside the profile); the rest are
integrity failures (the server misrepresented what it stored). A reader
SHOULD keep the two classes distinct in its error taxonomy and MUST NOT
mask either as a routine decryption miss. The four scheme-refusal cases
are thus: no `was`; no usable `was.v`; a future `was.v`; no usable
`was.epoch`.

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
order that names it. The incidental order in which secrets happened to
unwrap plays no part in the selection.

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
  failure rather than a silently absent key: key servers whose unwrap
  resolves
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
  member already present in the descriptor is always adopted as-is (not
  overwritten). Exactly one blinding key ever exists per collection.
* The key never rotates, not on epoch rotation, not on recipient
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
In a plaintext document store an index is a server-side optimization.
The server can backfill it over data it can read, and an incomplete
index degrades to a slower scan, as opposed to wrong answers. Neither
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
    the bare path: it declares the same simple index, not a
    one-member compound (the two forms blind differently, so
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

The schema rides the encrypted metadata envelope rather than the
plaintext Collection Description, because attribute names and index structure
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

## The resource log profile {#resource-log-profile}

This section defines the [=resource log=], a hash-linked log format, derived
from the `did:webvh` log format [[DID-WEBVH]], and used for key resources
co-managed between a wallet's clients and the storage server: the
[=encryption descriptors=] of this profile and the [=user key=] rosters
beside them ([[[#user-key-roster]]]). It covers the entry format, entry
hashing, chain verification, the external-authorization rule, client-side head
pinning, and the terminal handover entry. It deliberately does not contain any
key management of its own: a resource log carries no keys, and every entry is
authorized against the log's [=controller document=] ([[[#log-authorization]]]).

This document's own descriptors are the log's state. [[[#log-form]]]
states how a descriptor is carried as log state. The subsections after it
define the log itself.

### Point state and log form {#log-form}

An [=encryption descriptor=] as defined above is point state: the current
configuration, served whole, its invariants enforced by the server. Under
the log form the same resource is governed as a hash-linked log of
full-state entries, each entry proof-carrying and externally authorized,
and the encryption descriptor is one of the log's state schemas.

Under the log form:

* Each log entry's `state` carries the full descriptor at that version,
  under a `type` member identifying its schema -- for this profile's
  descriptors, the string `WasEpochConfiguration`
  ([[[#descriptor-members]]]). The rules of this profile
  apply to the `state` of the verified head entry exactly as they apply
  to a served point-state descriptor: same members
  ([[[#descriptor-members]]]), same first-epoch and append-only rules,
  same fail-closed refusal of an epoch-less state.
* Epoch history and log history are distinct axes. `epochs` remains
  append-only WITHIN each state (a resource-level invariant a verifier
  can check across consecutive entries), while the log records the
  succession of whole configurations -- so "epochs is append-only" and
  "currentEpoch never moves backwards" become verifiable claims over the
  entry chain rather than server-enforced rules.
* Authorship of a configuration is established by the entry's Data
  Integrity proof, verified against the deployment's root of trust per
  the external-authorization rule ([[[#log-authorization]]]); the entry
  proof covers the full [=epoch configuration=], and the hash chain
  with a pinned head makes any rollback a chain break -- the log form
  is what carries this profile's epoch-configuration integrity
  ([[[#epoch-integrity]]]).
* A point-state descriptor served beside a log is a projection bound to
  the log. The projection's `history` member
  names the governing log ([[[#descriptor-members]]]), and a verifying
  consumer acts only on a projection equal to the verified head's `state`
  after stripping `history` -- the one member the projection carries and
  an entry's `state` never does ([[[#log-resource]]]).

Nothing in this profile's epoch rules depends on which framing carries
the descriptor; a deployment adopts the log form per collection.

### Log model {#log-model}

A <dfn data-lt="resource logs">resource log</dfn> is an append-only sequence
of <dfn data-lt="log entry|log entries">log entries</dfn>, each carrying the
full state of one co-managed [=resource=] at that point in its history. Each
entry is hash-chained to its predecessor, and signed by the client that
appended it. The current state of the resource is the state of the verified head
entry; earlier entries exist so that any reader can verify how the resource got
there, entirely client-side, against a host that (for threat modeling purposes)
can be assumed adversarial.

Three parties appear in this profile:

* The [=controller document=] -- the DID document of the [=Space=]'s
  controller, resolved
  and verified by the reader independently of the storage server (for a
  `did:webvh` controller, by fetching and verifying its own hash-chained log).
  It is the log's root of authority; the set of keys that may authorize appends
  is defined there and only there.
* The **writers** -- [=enrolled clients=] (typically, the account's other
  wallet installations; [[APP-CONNECT]] describes them informatively from
  the connecting application's side), each
  holding a key listed in the controller document. Any number of writers may
  append; the profile assumes no coordination between them beyond the storage
  server's [Conditional Request](https://w3c-ccg.github.io/wallet-attached-storage-spec/#conditional-requests)
  compare-and-swap primitive (see [[[#log-append]]]).
* The **host** -- the storage server holding the log resource. Under [[WAS]]
  the host is trusted to enforce authorization on writes; this profile
  deliberately does not lean on that trust. For log verification the host is
  treated as a minimal store that linearizes concurrent appends and nothing
  more, and every guarantee below must hold even against a host that serves
  stale, forged, truncated, or forked logs.

The log's identifier is self-certifying (the [=SCID=] in its genesis
entry commits to the genesis content, so a host cannot substitute one log for
another under the same identity, see [[[#log-hashing]]]), while its
authority is externalized (no entry is valid on the log's own say-so;
every entry's signer must be found in the externally verified controller
document, see [[[#log-authorization]]]). The rationale for this split is given
in [[[#log-rationale]]].

### Relationship to `did:webvh` {#log-webvh}

<div class="note">
This subsection is non-normative.

The profile is an extraction from `did:webvh` [[DID-WEBVH]], and the
extraction is almost entirely subtractive:

* **Kept verbatim:** the five-member entry shape (`versionId`, `versionTime`,
  `parameters`, `state`, `proof`); JCS canonicalization; the
  SHA-256-multihash, `base58btc` entry hash; the SCID-style self-certifying
  genesis; the `eddsa-jcs-2022`-only proof rule; JSON Lines serialization.
* **Deleted:** the in-log key management (`updateKeys`, `nextKeyHashes`
  prerotation), witnesses, watchers, portability, deactivation, and the DID
  document typing of `state`.
* **Replaced:** the one verification step that consulted `updateKeys` -- the
  authorization predicate -- is replaced by the external rule of
  [[[#log-authorization]]].
* **Added:** the [=entry anchor=] ([[[#log-proof]]]), the [=terminal entry=]
  ([[[#log-handover]]]), and the [=chain-head pin=] semantics
  ([[[#log-pin]]]).

A did:webvh log answers to nobody -- it is a root of trust, so it must carry
its own key state. A resource log answers to the account that owns it, so
carrying its own key state would create a second root with no coherent
precedence rule; see [[[#log-rationale]]].
</div>

### The log resource {#log-resource}

A resource log is stored as a single [=resource=], serialized as JSON Lines
[[JSON-LINES]]: one [=log entry=] as one JSON object per line, in order, first
line first. Normatively, each line is a single JSON text [[RFC8259]]
serialized without embedded newlines, and lines are separated by U+000A LINE
FEED.

The log resource is the only serving of the resource it governs: this
profile defines no companion point-state document, and the governed
resource's current state exists only as the `state` of the verified head
entry ([[[#log-verification]]]).

The member name `history` is reserved in entry state: a [=log entry=]'s
`state` MUST NOT contain a `history` member.

### Entry format {#log-entry}

A [=log entry=] is a JSON object with exactly five members, all REQUIRED:

| Member        | Value                                                                                                                                                                                                                                             |
|---------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `versionId`   | `<n>-<entryHash>`: the entry's ordinal position `n` (decimal, starting at `1`), a single `-`, and the entry hash ([[[#log-hashing]]]).                                                                                                            |
| `versionTime` | The append time as an [[RFC3339]] UTC datetime (`Z` suffix). **Advisory only**; see below.                                                                                                                                                        |
| `parameters`  | A JSON object; see the per-entry rules below.                                                                                                                                                                                                     |
| `state`       | The full resource state at this version: a JSON object that MUST carry a `type` member (a string) identifying its schema. State schemas are defined by the referencing profile, not by the log format; for this profile's descriptors the schema is `WasEpochConfiguration` ([[[#log-form]]]). |
| `proof`       | A non-empty array of Data Integrity proofs ([[[#log-proof]]]).                                                                                                                                                                                    |

A verifier MUST reject an entry carrying members other than these five.

**`parameters` rules.** Unlike `did:webvh`, `parameters` do not evolve: format
changes travel through the handover mechanism ([[[#log-handover]]]), and not
through parameter mutation.

* The genesis entry's `parameters` MUST carry `method` (the format
  identifier, [[[#log-format-ids]]]) and `scid` (the [=SCID=]), and MAY carry
  `previousLog` (successor logs only, [[[#log-handover]]]). No other member
  is permitted.
* Every other entry's `parameters` MUST be `{}`, with one exception: a
  [=terminal entry=]'s `parameters` is exactly
  `{ "nextLog": { "method": ..., "scid": ... } }` ([[[#log-handover]]]).
* A verifier MUST reject an entry whose `parameters` carry any member this
  profile does not define for that entry position. This is deliberate
  fail-closed extensibility: the deleted `did:webvh` key-management
  parameters, in particular, MUST NOT be accepted, so a served log can never
  smuggle authorization semantics this profile removed.

**`versionTime` is advisory.** A writer MUST set it to its best knowledge of
the current time, and a verifier MUST NOT refuse an entry on temporal grounds
-- not for being out of order with respect to its neighbors, and not for being
in the future. Ordering authority rests entirely with the hash chain.

<div class="note">
This is a deliberate departure from `did:webvh`, which enforces strict
monotonicity plus a future-skew bound. Under that rule, one appender with a
fast clock produces an entry that later, correctly clocked appenders can never
legally follow -- a permanently unresolvable log. did:webvh needs enforceable
times for `versionTime`-based resolution; this profile offers no time-based
resolution, so it keeps the member for display and audit and moves all
ordering to the chain.
</div>

Example: a two-entry log (shown pretty-printed; on the wire each entry is
one line; hash, DID, and key values are illustrative):

```json
{
  "versionId": "1-QmUv7Yx3mvL2mLpp9pRgTvNqEcYtT7yiV1sfMx6xEBcCyD",
  "versionTime": "2026-08-06T17:00:00Z",
  "parameters": {
    "method": "resource-log:0.1",
    "scid": "QmXbWpS3fY9uNnQhJcM4kR8vT2eDgAqLx5oPiZsE6wHtUa"
  },
  "state": { "type": "WasEpochConfiguration", "...": "..." },
  "proof": [{ "...": "..." }]
}
{
  "versionId": "2-Qma8fN4kQjTr2vXwYhPzUdE9cLmB5oGsKiR7tD3eVpMnHx",
  "versionTime": "2026-08-07T09:30:00Z",
  "parameters": {},
  "state": { "type": "WasEpochConfiguration", "...": "..." },
  "proof": [{
    "type": "DataIntegrityProof",
    "cryptosuite": "eddsa-jcs-2022",
    "proofPurpose": "assertionMethod",
    "verificationMethod": "did:webvh:QmScid...:wallet-storage.example:space:81246131-69a4-45ab-9bff-9c946b59cf2e:id?versionId=6-QmDocVersion...#z6MkclientKey...",
    "proofValue": "z..."
  }]
}
```

### Entry hashing and the SCID {#log-hashing}

Canonicalization throughout this profile is JCS [[RFC8785]], over strict JSON
values (no `undefined`, no non-finite numbers).

**The hash serialization format.** Every hash this profile mints, including the
entry hash, the [=SCID=], and the `scid` values carried in `nextLog` and
`previousLog`, is serialized the same way: the SHA-256 digest of the
canonicalized input, wrapped as a multihash (the bytes `0x12`, `0x20`, then the
32 digest bytes), encoded with the `base58btc` alphabet, with **no** multibase
prefix. This is byte-for-byte the `did:webvh` entry-hash format. It is
deliberately NOT the `z`-prefixed multibase serialization, and NOT the
`base64url`serialization used for content addressing elsewhere in [[WAS]];
one format, chosen once, so a verifier never has to guess.

**The entry hash.** The `entryHash` of entry `n` is the hash, as above, of the
entry with its `proof` member removed and its `versionId` member replaced by
the **predecessor's** `versionId` -- for the genesis entry, by the [=SCID=].
The entry's own `versionId` is then `<n>-<entryHash>`. This is what makes the
chain a chain: each entry's identifier is a commitment to its predecessor's
identifier and transitively to the entire log back to genesis.

**The genesis entry and the <dfn>SCID</dfn>** (self-certifying identifier).
The genesis entry is built in two passes:

1. Build the preliminary entry with the literal placeholder string `{SCID}`
   as the value of both `versionId` and `parameters.scid` (and anywhere else
   the SCID will appear). The SCID is the hash, as above, of this preliminary
   entry.
2. Replace every `{SCID}` placeholder with the SCID, then compute the entry
   hash normally (`versionId` input value: the SCID). The genesis
   `versionId` is `1-<entryHash>`.

A verifier MUST recompute the SCID by the inverse procedure (substitute the
log's `parameters.scid` value back to `{SCID}`, hash, compare) and MUST
reject a log whose SCID does not verify.

Because `parameters.method` is inside the hashed genesis content, the SCID
commits to the format identifier: a host cannot downgrade a log's format
without changing its identity. And because every later entry hash chains to
the genesis through the `versionId` substitution rule, every entry hash in the
system is transitively bound to that identifier. Together with the fixed
five-member input shape, this is this profile's domain separation: no entry hash
can collide with a hash minted over any other structure.

### The entry proof and its anchor {#log-proof}

Each element of an entry's `proof` array MUST be a Data Integrity proof
[[VC-DATA-INTEGRITY]] with:

* `type`: `DataIntegrityProof`
* `cryptosuite`: `eddsa-jcs-2022` [[VC-DI-EDDSA]] -- no other cryptosuite is
  permitted
* `proofPurpose`: `assertionMethod`
* `verificationMethod`: a DID URL identifying the signing key in the
  [=controller document=], carrying the [=entry anchor=] described below
* `proofValue`: the proof value per [[VC-DI-EDDSA]]

The proof input is the complete entry, including its final `versionId`,
with the `proof` member absent. Because the `versionId` is a commitment to
the whole chain ([[[#log-hashing]]]), the signature covers the chain link;
an entry cannot be re-parented without breaking its proof.

A verifier MUST verify every proof in the array. One failing proof rejects
the entry.

**The <dfn data-lt="entry anchors|anchored version">entry anchor</dfn>.**
Where the controller's DID method provides versioned document resolution (as
`did:webvh` does), the proof's `verificationMethod` MUST carry a `versionId`
DID parameter [[DID-CORE]] naming the controller-document version the entry
was authorized under, e.g.
`did:webvh:...:id?versionId=6-Qm...#z6Mk...`. The anchor is inside the
signed proof options, so it cannot be altered without breaking the proof.
Where the controller's document is unversioned (a static DID such as
`did:key`), the anchor is omitted and every anchor rule below reads "the
controller document".

### Authorization {#log-authorization}

An entry is authorized by exactly one rule:

> Every proof's signing key MUST be listed as a verification method under the
> `assertionMethod` relation of the [=controller document=] **at the
> entry's anchored version**, where the controller document is resolved and
> verified by the reader independently of the host serving the log.

The rule's parts, unpacked, each normative:

1. **The document comes from the reader's own verification, not from the
   channel.** The verifier MUST resolve the controller document through its
   own verified resolution pipeline, cryptographically verifying every step:
   for a `did:webvh` controller, it fetches the controller log itself,
   verifies it end-to-end (SCID, entry hash chain, proofs, and its own head
   pin, where it keeps one), and answers the anchored-version lookup from
   that verified history. A verifier MUST NOT obtain the signing key by
   dereferencing the proof's `verificationMethod` URL as an ordinary
   resource fetch, and MUST NOT accept controller-document material supplied
   by the host outside that independently verified resolution -- the host
   serving the resource log is the same party a doctored controller document
   would come from. (In implementation terms: the document loader handed to
   the proof-verification library answers DID URL lookups from the verified
   controller log, not from the wire.)
2. **The anchored version must be present.** The anchor MUST identify a version
   present in the reader's verified controller log, at or before its head. An
   anchor naming an unknown version rejects the entry.
3. **Anchors are monotone along the log.** Each entry's anchored version MUST
   be the same as, or a descendant of (for a linear controller log: at or
   after), the previous entry's anchored version. An entry anchored behind
   its predecessor rejects the log.
4. **The relation is `assertionMethod`.** The log format's fixed proof shape
   ([[DID-WEBVH]]) sets `proofPurpose: assertionMethod`, and Data Integrity
   verification [[VC-DATA-INTEGRITY]] already requires a proof's verification
   method to be authorized under the controller-document relation matching
   the proof's purpose. The authorization rule uses that same relation, so
   proof verification and authorization are one membership check, enforceable
   with standard proof-verification tooling -- no second relation a signing
   key must be dual-listed under, and no custom verifier overriding the
   purpose-relation match. The flip side is that listing a key under
   `assertionMethod` is precisely what entitles it to append: a controller
   whose logs are governed by this profile MUST NOT list a key under
   `assertionMethod` unless that key is entitled to co-manage the account's
   governed resources. Keys listed under other relations only (a
   `keyAgreement`-only recovery key, an `authentication`-only login key)
   remain structurally excluded.

**Writers anchor at their verified head.** A writer appending an entry MUST
anchor it at the head of the controller document as the writer last verified
it. Together with anchor monotonicity this yields the profile's historical
verification: entries verify **as-of-append**. A client revoked from the
controller document keeps its past entries verifiable (they anchor at
versions where its key was present) while losing the ability to have new
entries accepted under post-revocation anchors.

**The sealing append.** After a controller-document change that removes a
verification method, the controller's clients MUST ensure that each governed
resource log receives at least one subsequent entry anchored at or after the
new document version. Anchor monotonicity then makes the removal effective
for that log: no later entry can anchor behind the seal, so the removed key
can never validly append again. Until a log's sealing append operation lands,
the removed key can still extend that log under a pre-removal anchor -- a
residual window with the same shape, detection, and repair story as any other
torn multi-resource ceremony. The staleness is visible in durable state (a
log whose head anchor predates the membership change), and re-running the
sealing pass will converge on the same state.

<div class="note">
In the wallet ceremonies this profile serves, the sealing append is not an
extra write: revoking a client already rotates every encrypted collection to
a new key epoch, and each rotation is itself a state change appended to the
governed log, anchored at the post-revocation document version.
</div>

### Chain verification {#log-verification}

A verifier MUST run the following checks over the full log, in order, and
MUST treat any failure as rejecting the log (not just the failing entry).
A verifier MUST NOT accept any stated or served head, digest, or count in
place of recomputing from the entries themselves.

1. **Parse.** Parse the resource as JSON Lines ([[[#log-resource]]]).
   Reject on any non-object line, on any entry violating [[[#log-entry]]]
   (member set, `parameters` rules, `state.type`), and on any `versionId` whose
   ordinal is not the entry's 1-based position.
2. **Genesis.** Recompute and check the [=SCID=] ([[[#log-hashing]]]).
   Check that `parameters.method` names a format this verifier implements
   ([[[#log-format-ids]]]); where a [=chain-head pin=] or the referencing
   profile supplied an expected method, check that it matches.
3. **Chain.** For each entry, recompute the entry hash from the
   predecessor-substituted input and check it against the `versionId`.
4. **Proofs.** For each entry, verify every proof per [[[#log-proof]]].
5. **Authorization.** For each entry, apply [[[#log-authorization]]]:
   resolve the anchor against the independently verified controller
   document, check `assertionMethod` membership of every signing key,
   and check anchor monotonicity.
6. **Termination.** If any entry is a [=terminal entry=], check that it is
   the last entry; reject a log with entries after a terminal entry, and
   refuse to append past one ([[[#log-handover]]]).
7. **Continuity.** Compare the verified head against the [=chain-head pin=],
   where one is held ([[[#log-pin]]]).

The resource's current state is the verified head entry's `state`.

<div class="note">
Full verification is a wallet-to-wallet concern: it is run by [=enrolled
clients=] co-managing governed resources against a host they do not trust for
them. An application connecting under [[APP-CONNECT]] never runs it -- the
application verifies the response presentation and invokes its
capabilities; it does not read these logs. In wallet
workflows, verification runs:

* before any append -- a writer extends only a head it has verified
  ([[[#log-append]]]), so every state-changing ceremony (a key-epoch
  rotation, the sealing append after a client revocation, a handover) begins
  with a verification pass;
* before acting on current state -- a client coming back online, or a second
  enrolled wallet syncing, verifies before trusting a served roster or epoch,
  since current state is defined only as the verified head's `state`;
* at first contact -- a newly enrolled client bootstrapping onto the
  account's governed resources verifies to establish its [=chain-head pin=]
  ([[[#log-pin]]]);
* on return visits -- re-verification against the held pin is where host
  rollback, truncation, and forks are detected (step 7).
</div>

### Appending {#log-append}

Appending is linearized by the host's compare-and-swap (CAS) primitive
(conditional writes on the log resource's entity tag, per [[WAS]]); the chain
itself carries no consensus mechanism, and none is needed: concurrent writers
produce a CAS conflict as opposed to a fork.

Conditional writes are an optional feature in [[WAS]]. A host serving
resource logs MUST support them: the backend holding the log MUST advertise
the `conditional-writes` feature, and a writer MUST NOT fall back to an
unconditional write against a backend that does not. Without the
precondition, concurrent appends silently overwrite one another instead of
failing into the retry loop below.

A writer MUST:

1. Read the full log and verify it per [[[#log-verification]]] -- an entry
   is never built on an unverified head.
2. Build the new entry against the verified head (full state, hash, anchor,
   proof) and write the extended log conditionally on the entity tag of the
   read from step 1.
3. On a conditional-write conflict: re-read, re-verify, rebase the change on
   the new head, and retry.
4. **Confirm by reading back.** A write acknowledgement is a promise, not a
   fact: the writer MUST read the log back and verify that the extended,
   verified log contains its entry before treating the append -- or any
   ceremony step gated on it -- as durable.

### Head pinning {#log-pin}

A reader that returns to a log across sessions MUST keep a <dfn
data-lt="chain-head pins">chain-head pin</dfn> per log: the log's format
identifier (`parameters.method`), its [=SCID=], and the `versionId` of the
latest verified head. The pin is client-local durable state; where it is
stored and how it is protected are left up to client implementers.

* The pin is established at **first contact**: the first full verification of
  a log the reader has no pin for. First contact is trust-on-first-use, and
  what it establishes is the log's identity (the SCID); see
  [[[#log-trust-bounds]]] for the limits and implications.
* The pin is updated only after a full verification ([[[#log-verification]]])
  of a log whose head is the pinned head or a descendant of it, or after a
  verified handover ([[[#log-handover]]]), which replaces the pin with the
  successor's (method, SCID, head).
* A served log whose SCID or method differs from the pin, outside a verified
  handover, MUST be refused as a continuity break.
* A served log whose head is behind the pin (the pinned head's ordinal
  exceeds the served head's, with the served log an ancestor-prefix of what
  was pinned) is reconcilable divergence -- possibly replication lag. The
  reader MUST NOT adopt it and MUST NOT regress its pin; it MAY retry.
* A served log that has **forked** -- it shares a prefix with the pinned
  history but diverges at some version, so neither is an ancestor of the
  other -- MUST be refused, and the reader SHOULD retain both the served log
  and its pinned evidence: every entry is signed, so a conflicting pair of
  logs under one SCID is transferable, independently verifiable evidence of
  equivocation.

A pin record MAY carry an extensible set of attached proofs (for example, a
host-signed checkpoint, or witness co-signatures adopted later as policy),
under the rule that a reader ignores proofs it cannot attribute. This lets
stronger continuity evidence be layered on without a format change. A reader
MAY additionally render the pinned head as a short human-comparable
fingerprint for out-of-band comparison between clients.

### The terminal handover entry {#log-handover}

One mechanism serves both log compaction (truncating a long history) and
format migration (moving to a successor format): a signed, in-chain
<dfn data-lt="terminal entries">terminal entry</dfn> that closes the log and
names its successor.

* A terminal entry is an ordinary entry ([[[#log-entry]]], [[[#log-proof]]],
  [[[#log-authorization]]] all apply) whose `parameters` is exactly
  `{ "nextLog": { "method": ..., "scid": ... } }`: the successor log's format
  identifier and its [=SCID=]. Its `state` MUST equal its predecessor's
  `state` -- a handover changes no resource state.
* The successor log's genesis `parameters` additionally carries
  `previousLog: { "scid": ..., "head": ... }`: the prior log's SCID and the
  `versionId` of the prior log's last regular entry -- the terminal entry's
  predecessor. (The reference cannot name the terminal entry itself: the
  terminal entry commits to the successor's SCID, so the successor's genesis
  must exist first.)
* For a compaction, the successor's genesis `state` is the full current
  state, so the successor log stands alone; the prior log may then be
  retained or discarded as evidence policy dictates.

A verifier crossing a handover MUST check the link from both sides: the
terminal entry's `nextLog.scid` equals the successor's SCID and the methods
are consistent with what each log's own genesis declares; the successor's
`previousLog.scid` equals the prior log's SCID; and the successor's
`previousLog.head` equals the `versionId` of the terminal entry's immediate
predecessor -- that is, the terminal entry chains directly off the head the
successor references. A successor served without this verifiable link MUST be
refused as a continuity break.

Every verifier of this profile MUST recognize terminal entries and MUST
refuse to append past one -- even though nothing currently emits them. This
is what makes the mechanism a safe migration path: a v1 verifier meeting a
handed-over log refuses to extend the frozen log, rather than continuing a
history its author has closed.

### Format identifiers {#log-format-ids}

The format identifier for this profile is the byte-significant string
`resource-log:0.1`. The identifier is compared only for byte equality
and never parsed: the `0.1` is part of the opaque identifier, not an
orderable version number. A future revision of this profile is a different
identifier reached only through the handover mechanism
([[[#log-handover]]]), not a "greater version" of this one.

The identifier appears at two seams plus the pin, with a strict
authority ordering:

1. **Authoritative:** `parameters.method` in the genesis entry. It is
   SCID-committed and proof-covered ([[[#log-hashing]]]), which makes it the
   only downgrade-safe location: a host cannot alter it without changing the
   log's identity and breaking its genesis proof.
2. **Payload schema:** the `state` document's own `type` member
   ([[[#log-entry]]]) -- the resource schema, versioned by the referencing
   profile independently of the log format.
3. **The pin:** the [=chain-head pin=] stores the method beside the SCID and
   head, so a served format switch outside the handover mechanism
   ([[[#log-handover]]]) is refused as a continuity break rather than
   dispatched to a different verifier.

### Design rationale: externalized log authorization {#log-rationale}

<div class="note">
This subsection is non-normative.

The obvious challenge to this design: self-sovereignty is the prestige
property of hash-linked logs (it is what makes `did:webvh` and its
relatives valuable), so why is a resource log deliberately NOT its own
root of trust, carrying its own key set the way an identity log does?

Because self-sovereignty is the right design exactly when a log's writers
share no pre-existing root of authority. A resource log is the opposite case:
its writer set is *defined as* the account's enrolled clients, which already
have a self-sovereign home in the controller document. Making each log its
own root would not add sovereignty; it would copy the client roster into N
places and then have to keep the copies consistent.

**Revocation atomicity is what in-log key management would break.** Under the
external rule, revoking a client is one controller-document edit, and that
single edit is the revoked client's pull axis everywhere: every delegation,
every invocation, and every log-append right dies against the same document.
If each log carried its own `updateKeys`, revocation would become a rotation
ceremony per log, and a crash mid-ceremony would leave the revoked client
durably authorized on the remainder -- a drift-detection problem created
purely by the duplication. (The sealing append of [[[#log-authorization]]]
narrows per-log windows too, but its failure mode is detectable staleness
that a re-run repairs -- not standing authorization that must be hunted
down.)

**Two roots of trust have no coherent precedence rule.** If a log's in-log
key set and the controller document could disagree, one of them wins. If the
document wins, the in-log keys are dead weight -- maintenance surface with no
authority. If the in-log keys win, the document's revocation didn't revoke,
and a compromised client can entrench itself in whichever logs it can still
rotate. Every answer makes in-log key management either vestigial or a hole.
A KEL never faces this because it answers to nobody; a log subordinate to an
account cannot be half-sovereign.

**The genuine benefit of full self-certification has no audience here.** A
standalone-verifiable log matters when third parties consume it without
account context -- which is why the identity log is self-certifying. An
encryption descriptor or key roster is meaningless outside its account, and
its only readers already fetch and verify the controller document for other
reasons, so "verify the controller first" adds zero marginal cost. The cheap
half of self-certification is kept anyway: the SCID genesis makes each log's
*identity* self-certifying, so a host cannot swap logs under an id. That
split -- self-certifying identity, externalized authority -- is the design.

The rule generalizes before it breaks: if a future resource's writers span
several accounts, the authorization predicate becomes "the signer is in any
of the named controllers' verified documents" -- still external. Only a log
whose membership must evolve with no controlling document anywhere truly
needs to be its own root, and no such log exists in this system.
</div>

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

An account holds one roster resource that delivers its [=user key=] to
every [=wallet client=]: a descriptor-shaped resource whose current
epoch's secret is the current user key, wrapped once per enrolled
client. This section defines the roster resource, the rule that its
recipients resolve from the [=account controller document=] rather than
from the roster itself, and the controller-marker convention that makes
that resolution unambiguous.

### The roster resource {#roster-resource}

The roster reuses this profile's [=encryption descriptor=] verbatim:
epoch entries ([[[#epoch-entry]]]), epoch ids ([[[#epoch-id]]]),
recipient entries ([[[#recipient-entry]]]), and the roster operations
([[[#roster-operations]]]) apply unchanged, as do the
epoch-configuration integrity guards ([[[#epoch-integrity]]]) and the
log form ([[[#log-form]]]). What differs is what the epoch secret is
for. A content collection's [=epoch secret=] seeds an [=epoch key=]
that seals [=envelopes=]; the roster describes no content and seals
nothing. Its epoch secret IS the payload: the current epoch's secret is
the account's current user key, and holding a wrap in the current epoch
is how an enrolled client receives it. Rotating the user key is the
ordinary removal operation of [[[#roster-operations]]] -- a fresh epoch
(a fresh user key) wrapped to each remaining recipient, `currentEpoch`
repointed.

This composes with the first-epoch freshness rule ([[[#first-epoch]]])
rather than excepting it. The user key is minted fresh with its epoch
-- the two are one generation, so the roster's epoch secret is not, and
does not derive from, any longer-lived key. The rule's account-level
clause then does its work one level down: a content collection's epoch
secret MUST NOT be, or be derived from, the user key, so no
collection-epoch escrow can ever hand an external recipient the account
key. The user key sits above collection epochs by wrap direction alone:
its derived key-agreement key ([[[#recipient-derivation]]]) receives
wraps of collection epoch secrets; the user key itself is wrapped only
in the roster.

The roster is a delivery channel, not a source of authority.
Possession of the roster resource confers nothing: its entries hold
public key ids and wrapped-key ciphertext only ([[[#recipient-entry]]]),
its recipients are resolved from the [=account controller document=]
([[[#roster-recipients]]]), and under the log form authorship of every
configuration anchors in that document's keys. An implementation that
keeps an epoch pin ([[[#epoch-integrity]]]) SHOULD retain it even when
it also pins the log head: the epoch pin still guards a client whose
chain-head state was lost.

### Document-backed recipients {#roster-recipients}

The roster's recipients are the account's enrolled wallet clients, and
their [=key-agreement keys=] resolve from the verified
[=account controller document=] rather than from the roster:

* A wrap is minted only to a key the verified document currently lists
  under `keyAgreement`. A recipient entry whose `kid` matches no such
  verification method MUST be dropped from resolution: it receives no
  wrap of any fresh epoch.
* A removal from the document is what retires a recipient. After an
  edit removes a client's verification methods, one rotation retires
  every current-epoch recipient the post-edit document no longer backs
  -- a converging operation a re-run completes, with no per-client
  pairing state required beyond the document itself.
* Where the roster is log-governed, that post-edit rotation is anchored
  at or past the edit, satisfying the sealing rule imposed after an
  authorized-writer removal ([[[#log-authorization]]]).

### The keyAgreement controller marker {#keyagreement-controller-marker}

Resolving recipients from the document requires pairing each enrolled
client with the `keyAgreement` verification method that is its own. The
document may carry `keyAgreement` methods that belong to no enrolled
client (a recovery key held against loss, for one), and nothing in
[[DID-CORE]] ties a `keyAgreement` method to a sibling signing method.
The controller marker is the write-side convention that records the tie
in the document itself.

On the write side:

* Each enrolled client's `keyAgreement` verification method MUST be
  published with its `controller` set to the client's own `did:key` DID
  -- the DID of the client's Ed25519 signing key -- rather than the
  account DID. The method's `id` is unchanged by the marker.
* No other verification method carries the marker. Signing methods MUST
  keep the account DID as `controller` (a proof verifies against the
  controller the method names, and the account's proofs name the
  account). A `keyAgreement` method that belongs to no enrolled client,
  such as a recovery key, MUST keep the account DID as `controller`, so
  client listings and revocation removals never match it structurally.
* The marked key MUST be the canonical X25519 twin of the client's
  signing key, by the conversion of [[[#recipient-derivation]]]. An
  enrollment offering a key-agreement key that is not that twin MUST be
  refused before anything is published. The marker asserts that the
  key-agreement key belongs to that signing key; the twin check is what
  makes the assertion true of every method a conforming wallet
  publishes.

On the read side:

* A client's key-agreement key is the `keyAgreement` method whose
  `controller` equals the client's `did:key` DID. A client with no
  marked method has no resolvable key-agreement key; a reader MUST NOT
  fall back to deriving the twin. Refusing beats guessing: a guessed
  key would let a removal report success over a method still in the
  document.
* A removal MUST remove every `keyAgreement` method whose `controller`
  matches the removed client's `did:key` DID -- the full set, not a
  first match -- so a client that published more than one is fully
  retired.

The marker makes the document agree with the roster's `kid` shape. A
recipient `kid` is `did:key:z6Mk...#z6LS...`
([[[#recipient-derivation]]]): the controller DID, then its derived
twin as the fragment. Under the marker, the document's `keyAgreement`
method for that client carries exactly that controller DID. The DID
half of a recipient's key id and the `controller` of its published
method state the same fact, so a reader moving between the roster and
the document never needs a pairing table.

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
server's checks bind its own storage, not what it chooses to serve.
Deployments needing rollback protection keep client-side monotonic
state (an epoch pin) or adopt the
log form, whose hash chain and pinned head make any rollback a chain
break ([[[#log-form]]]).

### Resource log trust bounds {#log-trust-bounds}

The [=resource log=] profile ([[[#resource-log-profile]]]) is designed against
a host that serves stale, forged, truncated, or forked logs, and its
guarantees come entirely from client-side recomputation: the chain from the
[=SCID=] forward, every proof, and every authorization check against the
independently verified [=controller document=]. Nothing served -- a stated
head, a digest, a count -- is accepted without the log confirming it.

Three attacks remain outside the model, and stating them is part of the design:

**First-contact substitution.** A reader with no [=chain-head pin=] for a log
accepts whichever verifying log the host serves first. The SCID prevents
substitution *under a known identity*, and the authorization rule means any
substitute must still be signed entirely by the account's own enrolled keys --
so what first contact actually risks is being shown a stale or truncated
history that genuine writers produced, not a forged one. From the pin onward,
rollback and forks are refused.

**Per-client equivocation.** A host can serve different (individually valid)
extensions of one log to different clients that have not compared pins. Each
client's own pin keeps its own view consistent; the gap is cross-client. The
mitigations are layered rather than structural: any two clients that ever
compare heads (or exchange logs) hold transferable, independently verifiable
evidence of the equivocation, since every entry is signed and both logs claim
one SCID ([[[#log-pin]]]); and the pin's extensible proof set leaves room for
witness cosignatures as a policy upgrade without a format change. Writers are
better protected than pure readers: the append procedure's read-back
confirmation ([[[#log-append]]]) means a host that hides one writer's entry
from another forces a visible CAS conflict or a visible missing entry, not a
silent split.

**The revocation window.** Between a membership-removing controller-document
edit and a given log's sealing append ([[[#log-authorization]]]), the removed
key can still extend that log under a pre-removal anchor. The window is the
same one any multi-resource revocation cascade has; what the profile adds is
that the window's state is durably visible (a head anchor predating the
membership change) and closes idempotently by re-running the seal.

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
carries a key id and wrapped bytes only, with no attribute names.
