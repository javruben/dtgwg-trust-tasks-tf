---
slug: vta/attestation/config-report
version: "0.1"
title: "VTA Attestation — Config Report"
summary: "Ask an agent for fresh attestation evidence that commits to the configuration it booted, so a tenant can check the settings its operator supplied before trusting the agent."
status: draft
targetFrameworkVersion: "0.5.0"
category: provenance
keywords:
  - attestation
  - tee
  - nitro
  - configuration
  - evidence
authors:
  - Glenn Gore (https://github.com/stormer78)
parties:
  - role: verifier
    requirement: REQUIRED
    member: issuer
  - role: verifiable trust agent
    requirement: REQUIRED
    member: recipient
proofRequirement:
  request: OPTIONAL
  response: REQUIRED
  rationale: >-
    As for vta/attestation/report: the request comes from a tenant that does not yet trust the agent and may hold no key it knows, and the verifier's own nonce, bound into the evidence, is what makes the answer its own. The response is REQUIRED so that the agent's DID is tied to the evidence and the configuration view it returns: the evidence authenticates the view by its digest, and the document proof says which agent is presenting it.
issuedAtRequirement:
  requirement: OPTIONAL
  rationale: >-
    Freshness is the nonce's job, and nothing is spent by answering a replay.
sideEffects:
  level: none
  rationale: "The agent asks its platform for a quote over a digest of the configuration it booted, and returns both. No state changes."
exposure:
  discloses: metadata
  ingests: none
  actsAsSubject: false
  rationale: >-
    The configuration view is the secret-free projection the agent booted with — the settings a tenant needs to check, such as which key-management key the agent was pointed at. It carries no secrets by construction; a view that contained one would be a defect of the agent, not a use of this task.
retention:
  class: transient
  rationale: The evidence proves this boot's configuration for this nonce only.
errorCodes:
  - code: vta/attestation/config-report:notAttested
    meaning: This agent has no attestation provider — it does not run in a trusted execution environment, or attestation is disabled.
    retryable: false
  - code: vta/attestation/config-report:noConfigSnapshot
    meaning: This agent captured no configuration snapshot at boot, so it has nothing to attest. Only a deployment that takes its configuration from its operator at launch captures one.
    retryable: false
  - code: vta/attestation/config-report:evidenceUnavailable
    meaning: The platform did not produce a quote. The agent runs in a TEE, but its attestation device refused or failed.
    retryable: true
related:
  - vta/attestation/report
  - vta/attestation/status
---

## Abstract

An agent in a trusted execution environment (TEE) runs an image the verifier can measure, but the image does not fix every setting. Some are supplied by the operator at launch — for example, which key-management key the agent unseals its secrets with. A tenant onboarding onto such an agent needs to know those settings are the ones it expects, and it cannot take the operator's word for them.

This task gives the tenant the evidence. The agent returns a canonical, secret-free **view** of the configuration it booted, and evidence that binds the verifier's nonce and the SHA-384 digest of that view. A verifier that checks the evidence has authenticated the view, and then applies its own policy to it.

## Status of this Document

This specification is a **draft** ([SPEC §5.3](/SPEC.md#53-maturity-levels)). It targets framework version 0.5.0 and may change without a version bump while it remains a draft ([SPEC §5.2](/SPEC.md#52-compatibility-rules)).

## Conformance

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY** and **OPTIONAL** in this document are to be interpreted as described in [BCP 14](https://www.rfc-editor.org/info/bcp14) when, and only when, they appear in all capitals.

A conforming producer and consumer satisfy [SPEC §7.1 and §7.2](/SPEC.md#7-minimum-requirements) in addition to the requirements stated here.

## Authorization

None, as for [`vta/attestation/report`](../../report/0.1/). A recipient **MUST NOT** refuse the request for want of a proof, an ACL entry or a session, and **SHOULD** rate-limit it per transport peer.

## Request

The request payload is the top-level schema of [`payload.schema.json`](payload.schema.json): a **REQUIRED** thirty-two-byte nonce, hex-encoded.

```json
{
  "id": "urn:uuid:2d3e4f5a-6b7c-4d8e-9f0a-1b2c3d4e5f6a",
  "type": "https://trusttasks.org/spec/vta/attestation/config-report/0.1",
  "recipient": "did:webvh:QmExampleScid:vta.example.com",
  "payload": {
    "nonce": "0a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f9"
  }
}
```

## Response

The recipient answers with the `$defs.Response` payload, in a document it signs with its own `authentication` key ([SPEC §4.7](/SPEC.md#47-proof)).

```json
{
  "id": "urn:uuid:7e8f9a0b-1c2d-4e3f-8a4b-5c6d7e8f9a0b",
  "threadId": "urn:uuid:2d3e4f5a-6b7c-4d8e-9f0a-1b2c3d4e5f6a",
  "type": "https://trusttasks.org/spec/vta/attestation/config-report/0.1#response",
  "issuer": "did:webvh:QmExampleScid:vta.example.com",
  "issuedAt": "2026-09-26T12:00:00Z",
  "payload": {
    "configDigestSha384": "OLBgp1GsljhM2TJ+sbHjaiH9txEUvgdDTAzHv2P24donTt6/529l+9Ua0vFImLlb",
    "configView": "eyJ0ZWUiOnsia21zIjp7ImtleV9hcm4iOiJhcm46YXdzOmttczpleGFtcGxlIn19fQ==",
    "nonce": "0a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f9",
    "teeType": "nitro",
    "evidence": "hEShATgioFkRXqlpbW9kdWxlX2lkeCdpLTAxMjM0NTY3ODlhYmNkZWYwLWVuYzAxMjM0NTY3ODlhYmNkZWY",
    "generatedAt": "2026-09-26T12:00:00Z"
  }
}
```

- `configView` is the canonical view, base64 (standard alphabet): the exact bytes whose SHA-384 the evidence binds.
- `configDigestSha384` is that digest, base64. It is a convenience; the verifier recomputes it.
- `nonce` echoes the request; `teeType` and `generatedAt` are informational.

## Verifying

A verifier **MUST**:

1. Check the evidence's signature chains to the platform vendor's root, and that its measured image is one it approved.
2. Check the nonce inside the evidence equals the one it sent.
3. Base64-decode `configView`, compute SHA-384 over those bytes, and check it equals the user data the evidence binds. It **MUST** also equal `configDigestSha384`; a mismatch is a refusal, never a preference for one of the two.
4. Check the response document's proof verifies as its `issuer`.

Only then is the view authenticated, and the verifier applies its own policy to it (for example, that the key-management key named is its own). A verifier does **not** need to reproduce the configuration from any source of its own; the evidence authenticates the view it was given.

## Security & Privacy

### Data carried

The request carries a nonce the verifier generated. The response carries the canonical, secret-free view of the configuration the agent booted, its SHA-384 digest, and the platform's quote binding both the digest and the nonce. The view discloses operator-supplied deployment settings — which is the point, since a tenant must be able to read them — and carries no secrets by construction. A deployment that considers a setting confidential does not put it in the view.

### Correlation

As for [`vta/attestation/report`](../../report/0.1/): the nonce is fresh and meaningless outside the request, and nothing in the response identifies the verifier. The view is the same for every asker.

### Retention

A tenant **MAY** keep the authenticated view as a record of what it onboarded onto. The agent keeps nothing.

### Consent/purpose

The operator's settings are disclosed so that a tenant can check them before it trusts the agent with anything; that is the purpose, and the only one. No subject's personal data is involved.

### Threats

The configuration is attested as the enclave booted it. A setting that can change after boot is outside what this task proves.

`teeType` and `generatedAt` are not covered by the evidence and **MUST NOT** be relied on; the platform is established by verifying the evidence as that platform's.
