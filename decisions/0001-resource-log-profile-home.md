# 0001: The resource log profile is normatively homed in WAS-EC

- Status: accepted
- Date: 2026-08-13
- Driving work: re-homing the Resource Log Profile -- the hash-linked,
  externally authorized log format for key resources co-managed between
  a wallet's clients and the storage server -- from the App Connect spec,
  where it was first drafted, into this spec, beside the state schemas it
  governs.
- Affects: encrypted-collections-spec (`#resource-log-profile` and its
  subsections, `#log-trust-bounds`); app-connect-spec (a short hand-off
  `#resource-log-profile` section and the `resource log` / `controller
  document` terms defined by reference); storage-core (`resourceLog.ts`)
  and was-client (`src/log/`) implement the profile wherever its text
  lives and are unaffected by the move.

## Context

The profile governs two state schemas, both defined in this spec: the
collection encryption descriptor (`WasEpochConfiguration`) and the
user-key roster that reuses it. This spec's epoch-configuration integrity
section names the log form as what carries descriptor authorship and
rollback protection, and the descriptor's `type` and `history` members
exist for the log form. Yet the profile's normative text lived in the App
Connect spec, a consent-ceremony profile, only because that is where it
was first drafted.

That placement created a circular normative dependency: App Connect
defers to WAS-EC for every other construction (roster, envelope,
recipient derivation), while WAS-EC deferred back to App Connect for the
log form alone. App Connect's own design goals flag the profile as the
one exception to its no-server-side-support rule (it needs WAS
conditional writes), which marks it as storage-layer material.

## Decision

The resource log profile's normative home is this spec. Its text moved
intact: same normative content, same format identifier
(`resource-log:0.1`), same entry, hashing, proof, authorization,
verification, append, pin, and handover rules.

The profile's only normative root of authority is the account controller
document. Its mention of enrolled clients (the writers of the log model)
is an informative back-reference to App Connect. App Connect keeps a
short hand-off section and cites the external-authorization rule from
WAS-EC for its enrolled clients.

## Rejected Alternatives

- Leaving the profile in App Connect. Keeps the dependency cycle and
  keeps a storage-integrity log format inside a consent-ceremony spec
  that otherwise needs nothing from the server.
- A standalone resource-log spec. Deferred, not rejected on the merits:
  the profile's only referencing state schemas today are this spec's
  descriptors and rosters, so a third document would add a reference hop
  with no second consumer to justify it.

## Consequences

- WAS-EC gains DID-WEBVH and JSON-LINES bibliography entries, and the
  resource log constants join the list of permanent byte-level values
  this spec guards.
- App Connect no longer depends on DID-WEBVH except as a controller
  document method; its enrolled-client term cites WAS-EC.
- A normative WAS-EC-to-App-Connect reference inside the log profile
  must not be reintroduced; it would recreate the cycle.

## Revisit Criteria

Reopen this decision when one or more of the following holds:

1. A second referencing profile (a state-schema family outside this
   spec) adopts the resource log format. At that point a standalone
   profile spec is the right home, reached by extraction from this spec,
   with the format identifier unchanged.
2. The profile acquires normative dependencies on App Connect's consent
   surface that cannot be expressed against the controller document
   alone.
