# 0003: This profile's identifier is `https://w3id.org/pws/encrypted-collections`, and its two optional affordances are its own entry's tokens

- Status: accepted
- Date: 2026-09-16
- Driving work: ECS-9. PWS core removed the per-Backend `features` array, so
  the two affordances this profile offers needed a home, and a `specs` entry
  needs an identifier to be listed under.
- Affects: encrypted-collections-spec (`#service-description` and its
  subsections, the gating sentences in `#blinded-index`, `#log-form`,
  `#limitations`, `#chunked-resources`, and `#log-append`; the conformance
  section in `index.html`); wallet-attached-storage-spec (its service
  description and its blinded-index and governing-log sections cite this
  entry by name); was-client, was-teaching-server, was-conformance-suite
  (the gates and the served `specs` document read this entry).

## Context

PWS core used to carry a `features` array on each Backend description. A
client read it to decide whether the server would serve a blinded-index
query, or a Collection's governing history log. WASS-40 removed the array:
the client never addresses a storage engine, the server sees every write,
and so no token there genuinely varied per backend. Guarantees a profile
depends on -- conditional writes, key epochs -- became baseline server
requirements. The affordances a profile offers had to move to that profile's
own service-description entry.

Two of them are this profile's: server-side blinded-index query with `unique`
enforcement, and the `meta/log` sub-resource from whose head the server
derives a Collection's governed `encryption` member. Both are genuinely
optional, because this profile defines a way to work without each -- fetch
and filter locally for the index, point state with a client-held epoch pin
for the log form.

A `specs` entry is keyed by the persistent identifier a specification
declares for itself in its own text, and a specification that declares none
cannot be advertised at all. This profile declared none.

## Decision

The persistent identifier of this specification is
`https://w3id.org/pws/encrypted-collections`.

It sits under the `pws/` sub-path, beside
`https://w3id.org/pws/authz-profile` and under the core
`https://w3id.org/pws`, so the PWS family reads as one namespace. The
identifier is not the document's location, carries no version, and is
compared as an opaque string.

A version entry under that key carries `version` and `url` as PWS defines
them for every entry, plus this profile's `features` array. The two tokens
are `blinded-index-query` and `governed-history-logs`, defined here with the
meaning they had in PWS's Backend Data Model.

An entry claims conformance to this profile's required server-visible
surface, which is the chunk endpoints (decision 0002). Everything else this
profile defines binds clients, so a server that serves no chunk endpoints and
neither affordance simply lists no entry -- and still stores an encrypted
collection's envelopes, since the construction is client-side.

## Rejected Alternatives

- `https://w3id.org/encrypted-collections`, the flat form PWS core's worked
  example prints. It breaks the pattern the authorization profile sets, and
  reads as an independent specification rather than a PWS profile.
- `https://w3id.org/byoe/encrypted-collections`, under the namespace this
  ecosystem's vocabulary terms use. It signals who maintains the document
  rather than what it profiles, and the maintainer is not what a client
  matching a `specs` key cares about.
- Keeping both affordances as tokens of the core PWS entry. They are this
  profile's surface, not core's, and core would then enumerate tokens whose
  meaning it does not define.
- Advertising each affordance by a URL member instead of a token. PWS
  reserves a URL member for an optional section with an endpoint of its own
  at a URL the client cannot otherwise derive. Both of these live at paths a
  client already knows from a Collection's URL, so a token is the right
  shape.

## Consequences

- The document has a service description section for the first time, between
  Terminology and the encryption descriptor.
- The `conditional-writes` backend-advertisement sentence in `#log-append`
  is gone; conditional writes are a baseline PWS requirement, so a host
  serving resource logs supports them by definition.
- A writer consults `governed-history-logs` before choosing between point
  state and the log form, since the two are written through different
  endpoints.
- PWS core's worked service-description example prints the flat identifier
  and needs correcting to the string decided here.

## Revisit Criteria

Reopen this decision when one or more of the following holds:

1. The PWS core identifier changes -- it is provisional pending the rename
   -- and the sub-path this one hangs off moves with it.
2. A third affordance appears that is optional for a reason other than "the
   profile defines a way to work without it", which would mean the token
   array is carrying something it was not shaped for.
