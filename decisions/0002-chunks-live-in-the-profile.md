# 0002: Chunked resources are defined by PWS-EC, not by PWS core

- Status: accepted
- Date: 2026-09-16
- Driving work: moving the Chunked Resources section out of the core PWS
  specification and into this profile, together with the chunked-stream
  half of the retired EDV-over-PWS appendix.
- Affects: encrypted-collections-spec (`#chunked-resources` and its
  subsections, the `"chunked"` encoding row of `#plaintext-document`, the
  conformance section in `index.html`);
  wallet-attached-storage-spec (the section is removed from core, and the
  `chunks` row of its Reserved Path Segment Registry names this document as
  owner); was-client (`src/edv/`, whose codec keeps its chunk path
  unchanged), was-teaching-server and was-conformance-suite (no wire change;
  their citations re-point here).

## Context

PWS core carried a Chunked Resources section that described itself as
scheme-agnostic. Its only producer and consumer, in every repo speaking the
contract, is this profile's EDV codec.

Chunks exist because of encryption. A JWE is sealed as one unit in memory, so
a document larger than a client can hold must be cut into several bounded
envelopes, each its own request. A plaintext blob has no such constraint: it
is one HTTP body, bounded by the backend's `maxUploadBytes` whether it is
chunked or not. So the substrate belongs beside the envelope construction it
serves.

The counter-argument is that a plaintext deployment may one day need to
upload something larger than one request allows.

## Decision

The Chunked Resources section is normatively homed in this profile. Its wire
behavior moved unchanged: the same paths, verbs, canonical-index rule, `308`
redirect rule, error types, cascade-delete lifecycle, and authorization
scoping. PWS core keeps only the `chunks` row in its Reserved Path Segment
Registry, naming this document as the owner of the segment.

Serving the chunk endpoints is required of a server that implements this
profile rather than advertised by a feature token. Chunk support is blob CRUD
under a sub-path, and an encrypted collection that never stores a large
document never uses it, so a token would cost more than it buys.

A future plaintext large-upload need is a different feature. What such a
deployment wants is resumable upload -- offset-addressed, restartable,
byte-range aware, with a server that knows how to assemble the parts. Chunked
resources are none of those: the server never concatenates anything, indexes
are written in any order, and the framing is the client's. Keeping chunking
in PWS core as a placeholder for resumable upload would keep a section whose
only reason to exist lives here, under a name that misdescribes the feature
it was being held for.

## Rejected Alternatives

- Leaving the section in PWS core. Keeps a section in the substrate spec
  whose only producer and consumer is this profile, and makes every
  implementer read the encryption rationale from the wrong document.
- Keeping it in core and gating it behind a feature token. Adds a discovery
  round trip and a client-side branch for an endpoint group that is cheap to
  serve unconditionally.
- Defining chunked resources in a third document. No second consumer exists
  to justify the reference hop.

## Consequences

- This document defines an HTTP surface for the first time. Its introduction
  and its conformance section say so, and the server-side rows of the parties
  table are no longer only about descriptor shape and epoch stamps.
- PWS-EC gains a PWS-AUTHZ bibliography entry: chunk authorization cites the
  baseline profile's exact-match `invocationTarget` rule and its `Digest`
  request-body integrity requirement.
- Core error types are cited across documents, by absolute link into PWS's
  Error Type Registry.
- The `chunked-streams` feature token is retired. A client gates a chunked
  write on the server implementing this profile, not on a token.

## Revisit Criteria

Reopen this decision when one or more of the following holds:

1. A plaintext (non-encrypted) producer of chunked resources appears with a
   need the chunk endpoints actually serve, rather than a resumable-upload
   need.
2. Resumable upload is specified for PWS core, and its endpoint group
   subsumes what chunked resources do. The chunk endpoints would then be
   restated in terms of it, or retired in its favor.
