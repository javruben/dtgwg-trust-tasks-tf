---
slug: social-recovery/status/get
version: "0.1"
title: "Social Recovery — Status"
summary: "A device owner checks whether their own device currently needs recovery-buddy-assisted recovery to rejoin the identity — a per-device health read, not a report on any buddy's own standing."
status: draft
targetFrameworkVersion: "0.6.0"
category: key-management
keywords:
  - social-recovery
  - device-health
  - key-recovery
parties:
  - role: device owner
    requirement: REQUIRED
    member: issuer
  - role: recovery maintainer
    requirement: REQUIRED
    member: recipient
proofRequirement:
  requirement: REQUIRED
  rationale: >-
    The status is per-device and per-owner; an unattributed read would let
    a party other than the device's own owner learn whether that device
    currently needs recovery, which is itself a signal about the owner's
    custody state.
sideEffects:
  level: none
  rationale: >-
    A read-only health check (`category: "read"`,
    `requiresApproval: false`); nothing is created or changed.
exposure:
  discloses: metadata
  actsAsSubject: false
  rationale: >-
    The response discloses whether this specific device currently needs
    recovery-buddy-assisted recovery to rejoin the identity — a device
    health fact, not the recovery-buddy roster itself (that is
    `social-recovery/buddies/list`) and not any key material.
errorCodes:
  - code: social-recovery/status/get:deviceNotFound
    meaning: >-
      `localDeviceDid` does not resolve to a device this owner controls.
    retryable: false
related:
  - social-recovery/buddies/list
  - social-recovery/buddies/add
---

## Abstract

The **Social Recovery — Status** Trust Task answers one narrow question:
does this specific device currently need recovery-buddy-assisted recovery
to rejoin the identity? It is a per-device health read, distinct from
[`social-recovery/buddies/list`](../../buddies/list/0.1/spec.md), which
answers a different question (who is enrolled as a recovery buddy for this
identity at all) — a device can need recovery whether or not any buddies
are currently enrolled to help with it, and this task reports only the
former.

## Status of this Document

This specification is a **draft** ([SPEC §5.3](/SPEC.md#53-maturity-levels)).
It targets framework version 0.6.0 and may change without a version bump
while it remains a draft ([SPEC §5.2](/SPEC.md#52-compatibility-rules)).

It documents an operation a maintainer already implements
(`recoveryStatus`, `src/packages/devices/manifest-operations-part-2.ts`),
written down so the shape stops being recoverable only by reading an
implementation.

## Conformance

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD
NOT**, **RECOMMENDED**, **MAY** and **OPTIONAL** in this document are to be
interpreted as described in [BCP 14](https://www.rfc-editor.org/info/bcp14)
when, and only when, they appear in all capitals.

A conforming producer and consumer satisfy
[SPEC §7.1 and §7.2](/SPEC.md#7-minimum-requirements) in addition to the
requirements stated here.

## Definitions

- **`localDeviceDid`** — REQUIRED, the querying device's own DID. This
  task reports on this specific device's own recovery health, not on the
  identity's devices in aggregate.
- **Recovery health** — whether this device, right now, is in a state
  where it would need buddy-assisted recovery to rejoin the identity
  (e.g. it has lost its own key material or standing), as distinct from a
  device that is functioning normally and has no such need.

## Request

The device owner names the querying device. The top-level schema is in
[`payload.schema.json`](payload.schema.json).

### Checking a device's own recovery health

```json
{
  "id": "urn:uuid:00000000-0000-4000-8000-000000000501",
  "type": "https://trusttasks.org/spec/social-recovery/status/get/0.1#request",
  "issuer": "did:example:owner",
  "recipient": "did:example:recovery-maintainer",
  "issuedAt": "2026-01-01T00:00:00Z",
  "threadId": "urn:uuid:00000000-0000-4000-8000-0000000005ff",
  "payload": {
    "localDeviceDid": "did:example:device-a1b2"
  }
}
```

## Response

The maintainer reports the device's current recovery-need status. The
sub-schema is reachable via `$anchor: "response"`. Failures are
`trust-task-error` documents.

### Device healthy, no recovery needed

```json
{
  "id": "urn:uuid:00000000-0000-4000-8000-000000000502",
  "type": "https://trusttasks.org/spec/social-recovery/status/get/0.1#response",
  "issuer": "did:example:recovery-maintainer",
  "recipient": "did:example:owner",
  "issuedAt": "2026-01-01T00:00:01Z",
  "threadId": "urn:uuid:00000000-0000-4000-8000-0000000005ff",
  "payload": {
    "localDeviceDid": "did:example:device-a1b2",
    "needsRecovery": false
  }
}
```

## Security & Privacy

### Data carried

The request carries only a device identifier. The response carries a
single recovery-need flag — no key material, no buddy roster.

### Correlation

This is a self-directed health check: the maintainer already knows this
device's state, and the response is returned only to the device's own
owner. No third party's data is disclosed.

### Retention

`transient` in effect: recovery health is a live, re-checkable state
rather than a record this task itself needs to retain.

### Consent/purpose

Descriptive only, per
[SPEC §7.3 item 13](/SPEC.md#73-specification-requirements) — a plain
health read requires no approval, matching the reference implementation's
`requiresApproval: false`.

### Custody scope

A recovery maintainer typically serves more than one device across
possibly more than one owner. It **MUST** scope this read to devices the
requesting owner actually controls, and **MUST** answer a `localDeviceDid`
belonging to a different owner identically to a nonexistent one, the same
enumeration-refusal discipline `vault/credentials/archive`'s Custody scope
section states for its own boundary.
