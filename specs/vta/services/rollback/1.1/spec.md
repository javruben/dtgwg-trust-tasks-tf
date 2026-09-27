---
slug: vta/services/rollback
version: "1.1"
title: VTA Services — Rollback
summary: An operator reverts a transport to its previous settings by writing a new log entry.
status: draft
targetFrameworkVersion: "0.5.0"
category: did-management
keywords:
  - vta
  - services
  - transport
  - did-document
authors:
  - Glenn Gore (https://github.com/stormer78)
parties:
  - role: Operator
    requirement: REQUIRED
    member: issuer
  - role: VTA
    requirement: REQUIRED
    member: recipient
proofRequirement:
  requirement: REQUIRED
  rationale: Changes what the world believes about reaching this agent, exactly as enable and update do.
issuedAtRequirement:
  requirement: REQUIRED
  rationale: A rollback reverts the service to a previous release, so what it discards is decided by when it runs. Replayed, it discards a release deployed since the first execution.
sideEffects:
  level: mutating
  rationale: "Writes a new DID-document log entry restoring earlier settings."
exposure:
  discloses: metadata
  actsAsSubject: false
errorCodes:
  - code: vta/services/rollback:notFound
    meaning: No such transport is configured on this agent.
    retryable: false
  - code: vta/services/rollback:conflict
    meaning: There is no previous state to restore.
    retryable: false
  - code: vta/services/rollback:notAuthorized
    meaning: The caller is not a super-admin.
    retryable: false
related:
  - vta/services/update
  - vta/services/enable
---

## Abstract

**VTA Services — Rollback** reverts a transport to its previous advertised
settings.

It does so by writing a **new** log entry. The did:webvh log is append-only, so
a rollback is a forward step that restores an earlier state — never an erasure
of what happened. Anyone auditing the document afterwards sees the mistake and
the correction, which is the intended behaviour and not a limitation to work
around.

## Changes from 1.0

1.1 adds one optional request member, `drainTtlSecs`, for the **mediated**
transports — `didcomm` and `tsp`, the two whose settings name a mediator. It is
the same member, with the same rules, that
[`vta/services/update/1.1`](../../update/1.1/) added.

A rollback of a mediated transport can move it off its current mediator:
restoring a previous `mediatorDid`, or re-disabling a transport whose enable is
being undone. The mediator left behind then drains — it keeps accepting delivery
for correspondents still holding the current DID document, and a DIDComm sender
and a TSP sender with a cached document are stranded in exactly the same way
when it stops. 1.0 gave the operator no say in how long that drain lasts, so a
recipient could only apply its own default, even where the same operator could
choose the window when making the change the rollback undoes. An operator
rolling back a mediator it no longer trusts needs the shortest window it is
allowed; one whose correspondents refresh slowly needs a longer one. Nothing
else changes; a 1.0 request is a valid 1.1 request, and the response is
unchanged.

The drain rules are those of update 1.1:

- `drainTtlSecs` is refused for a non-mediated transport (`rest`, `webauthn`):
  a member that does not apply to the named service makes the request malformed
  (`malformedRequest`) rather than being ignored.
- Absent, the recipient applies its default window.
- A recipient **MAY** raise a value below its floor to the floor — over a
  request that arrived through the mediator being replaced it **MUST**, since
  tearing that mediator down discards the reply — and reports the window it
  applied in the result's `drainUntil`.
- Where one mediator carries both mediated transports, replacing it for either
  drains it for both: the old mediator keeps accepting DIDComm and TSP delivery
  until the one `drainUntil` the result reports. A recipient **MUST NOT** run two
  drains of different lengths on one mediator.

A rollback of a mediated transport that leaves no mediator draining — the
previous state names the mediator already in use, or the rollback is a
`noOp` — starts no drain. `drainTtlSecs` is then accepted and has no effect, and the result carries
neither `drainingMediator` nor `drainUntil`. That is not a refusal: whether a
rollback drains depends on stored state the operator may not have in view, and
refusing a well-formed window for it would make the member unusable in exactly
the case it exists for.

## When there is nothing to write

If the previous state already equals the current one, the rollback writes
nothing and answers `kind: "noOp"` with no `logEntryVersionId`. That is a
**success**: the requested state holds. Reporting it as a failure would be
wrong, and reporting it as an ordinary success would name a log entry that does
not exist — which is why `kind` is required and `logEntryVersionId` is not.

## Status of this Document

This is a **draft** *Trust Task specification* per [SPEC.md §5.3](/SPEC.md#53-maturity-levels); the schema **MAY** change without notice.

## Conformance

[[RFC2119]](https://www.rfc-editor.org/rfc/rfc2119) and [[RFC8174]](https://www.rfc-editor.org/rfc/rfc8174) apply.

A conforming **consumer** (the VTA) **MUST** reject a payload carrying `drainTtlSecs` for a non-mediated `service` (`rest`, `webauthn`) with `malformedRequest`, rather than ignoring the surplus member. It **MUST** write a did:webvh log entry for every accepted change, and **MUST** report `serverless: true` when that entry was not published. When the rollback leaves a mediator draining it **MUST** report `drainingMediator` and the `drainUntil` it actually applied, which **MAY** be later than `drainTtlSecs` asks for but **MUST NOT** be earlier than the floor it enforces.

## Authorization

Authority is **super-admin**. Every task in this family edits — or reads — what
the agent tells the world about reaching it, and the ACL has no finer capability
for a subset of transports.

A `proof`, where present, establishes that the producer authored the request. It
is not the authorization; the caller's super-admin role is.

## Request

The Operator sends the request to the VTA; the payload is described by
`payload.schema.json`.

### Rolling back a REST endpoint

```json
{
  "id": "00000006-0000-4000-8000-000000000001",
  "type": "https://trusttasks.org/spec/vta/services/rollback/1.1",
  "issuer": "did:key:z6MkOperator",
  "recipient": "did:web:vta.example",
  "issuedAt": "2026-08-19T09:10:00Z",
  "payload": {
    "service": "rest"
  }
}
```

### Rolling back a mediator change with a one-hour drain

Restoring the previous DIDComm mediator, and giving the one being left an hour
to drain:

```json
{
  "id": "00000006-0000-4000-8000-000000000003",
  "type": "https://trusttasks.org/spec/vta/services/rollback/1.1",
  "issuer": "did:key:z6MkOperator",
  "recipient": "did:web:vta.example",
  "issuedAt": "2026-08-19T09:12:00Z",
  "payload": {
    "service": "didcomm",
    "drainTtlSecs": 3600
  }
}
```

## Response

The VTA answers with a `#response` document whose payload is the sub-schema
anchored at `response` in `payload.schema.json`: a `result` describing what the
rollback did. Failures use `trust-task-error`, not a `#response` document.

### A rollback that restored earlier settings

```json
{
  "id": "00000006-0000-4000-8000-000000000002",
  "type": "https://trusttasks.org/spec/vta/services/rollback/1.1#response",
  "issuer": "did:web:vta.example",
  "recipient": "did:key:z6MkOperator",
  "issuedAt": "2026-08-19T09:10:01Z",
  "threadId": "00000006-0000-4000-8000-000000000001",
  "payload": {
    "result": {
      "kind": "updated",
      "logEntryVersionId": "5-zQmLogEntry",
      "effectiveAt": "2026-08-19T09:10:01Z",
      "vtaDid": "did:webvh:QmAgent:vta.example",
      "serverless": false
    }
  }
}
```

### A mediator rollback that left the old mediator draining

```json
{
  "id": "00000006-0000-4000-8000-000000000004",
  "type": "https://trusttasks.org/spec/vta/services/rollback/1.1#response",
  "issuer": "did:web:vta.example",
  "recipient": "did:key:z6MkOperator",
  "issuedAt": "2026-08-19T09:12:01Z",
  "threadId": "00000006-0000-4000-8000-000000000003",
  "payload": {
    "result": {
      "kind": "updated",
      "logEntryVersionId": "6-zQmLogEntry",
      "effectiveAt": "2026-08-19T09:12:01Z",
      "drainingMediator": "did:web:new-mediator.example",
      "drainUntil": "2026-08-19T10:12:01Z",
      "vtaDid": "did:webvh:QmAgent:vta.example",
      "serverless": false
    }
  }
}
```

## Security & Privacy

### Data carried

The request carries the transport to roll back and, for a mediated transport,
the drain window. The response carries the log entry that made the change and,
when one was started, the draining mediator and when its drain ends. Everything
here ends up in, or describes, the agent's public DID document.

### Correlation

The advertised surface is public by construction: anyone may fetch the agent's
DID document. The rollback restores settings that were published before and
introduces no identifier that was not about to be published again.

### Retention

The signed log entry is the durable record, kept for as long as the DID's log
is; the exchange itself carries nothing beyond it.

### Consent/purpose

The operator changes how its own agent is reached. No other party's data is
involved; correspondents learn of the change by resolving the DID, which the
drain gives them time to do.

### Threats

The disclosure risk here is not the *content* of a service entry but the
*ability to change it*: an attacker who can roll a transport back can point the
agent's traffic at infrastructure it was previously using — possibly one the
operator moved away from because it was compromised — and every client that
resolves the DID afterwards will believe it. That is why every mutation in this
family is super-admin only, and why each writes a signed log entry rather than
flipping a runtime flag. The log is what makes a change attributable after the
fact.

Two failure modes are worth stating plainly:

- **`serverless: true` means nobody else can see the change yet.** The entry is
  written locally but not published; a consumer that reports success without
  surfacing this tells the operator a change is live when no verifier can
  observe it.
- **A drain is not a completed rollback.** While `drainUntil` is in the future
  the mediator being left is still accepting delivery. A short `drainTtlSecs`
  strands senders that have not re-resolved the DID; the floor exists so that at
  least the request's own reply is not one of them. Conversely, a long drain on
  a mediator being abandoned because it is distrusted keeps it in the delivery
  path for that long — the operator chooses, and the recipient reports what it
  applied.
