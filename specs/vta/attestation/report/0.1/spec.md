---
slug: vta/attestation/report
version: "0.1"
title: "VTA Attestation — Report"
summary: "Ask an agent for fresh hardware attestation evidence, bound to a nonce the verifier chose and to the agent's DID, so the verifier can check what code the agent runs."
status: draft
targetFrameworkVersion: "0.5.0"
category: provenance
keywords:
  - attestation
  - tee
  - nitro
  - sev-snp
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
    The request is made before the verifier trusts the agent, often by a party that holds no key the agent would recognise, and what protects it is not a signature but the nonce: the evidence binds the verifier's own nonce, so a report the verifier did not ask for cannot pass its check, whoever sent the request. The response is REQUIRED because the evidence attests the enclave, not the agent's DID, and the verifier is trying to learn that this DID is that enclave: the agent's signature over the document ties the DID it signs as to the evidence it returns, alongside the DID the evidence itself binds.
issuedAtRequirement:
  requirement: OPTIONAL
  rationale: >-
    Freshness is the nonce's job. A replayed request carries the nonce its verifier already has an answer for; the agent producing that evidence again changes nothing the verifier relies on.
sideEffects:
  level: none
  rationale: "The agent asks its platform for a quote and returns it. No state changes."
exposure:
  discloses: metadata
  ingests: none
  actsAsSubject: false
  rationale: >-
    The evidence names the measured image and the platform, and binds the agent's DID. That is what an attestation discloses by design, and nothing more.
retention:
  class: transient
  rationale: The evidence proves freshness for this nonce only. A verifier that wants to rely on the agent again asks again with a new nonce.
errorCodes:
  - code: vta/attestation/report:notAttested
    meaning: This agent has no attestation provider — it does not run in a trusted execution environment, or attestation is disabled.
    retryable: false
  - code: vta/attestation/report:evidenceUnavailable
    meaning: The platform did not produce a quote. The agent runs in a TEE, but its attestation device refused or failed.
    retryable: true
related:
  - vta/attestation/status
  - vta/attestation/config-report
---

## Abstract

An agent running in a trusted execution environment (TEE) can produce **evidence**: a quote signed by the platform vendor's key that names the code image the enclave was started from. A verifier that checks the evidence against the vendor's root knows what code holds the agent's keys.

This task asks for it. The verifier supplies a **nonce**, and the agent returns evidence that binds that nonce and the agent's DID. Binding the nonce makes the evidence fresh: it was produced for this request, and no earlier report can stand in for it. Binding the DID ties the enclave to the identity the verifier will go on to trust.

## Status of this Document

This specification is a **draft** ([SPEC §5.3](/SPEC.md#53-maturity-levels)). It targets framework version 0.5.0 and may change without a version bump while it remains a draft ([SPEC §5.2](/SPEC.md#52-compatibility-rules)).

## Conformance

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY** and **OPTIONAL** in this document are to be interpreted as described in [BCP 14](https://www.rfc-editor.org/info/bcp14) when, and only when, they appear in all capitals.

A conforming producer and consumer satisfy [SPEC §7.1 and §7.2](/SPEC.md#7-minimum-requirements) in addition to the requirements stated here.

## Authorization

None. Evidence is public by design, and a verifier asks before it trusts the agent. A recipient **MUST NOT** refuse the request for want of a proof, an ACL entry or a session.

Producing a quote costs the platform work, so a recipient **SHOULD** rate-limit the request per transport peer.

## Request

The request payload is the top-level schema of [`payload.schema.json`](payload.schema.json). `nonce` is **REQUIRED**: thirty-two bytes the verifier generated for this request, hex-encoded.

```json
{
  "id": "urn:uuid:1c2d3e4f-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
  "type": "https://trusttasks.org/spec/vta/attestation/report/0.1",
  "recipient": "did:webvh:QmExampleScid:vta.example.com",
  "payload": {
    "nonce": "8f14e45fceea167a5a36dedd4bea2543a1f0b1c2d3e4f5a6b7c8d9e0f1a2b3c4"
  }
}
```

A recipient **MUST** bind exactly this nonce into the evidence, and **MUST NOT** answer with evidence produced for any other nonce — including evidence it cached from an earlier request. There is no nonce-less form of this task: a report nobody asked for is one anybody can replay.

## Response

The recipient answers with the `$defs.Response` payload, in a document it signs with its own `authentication` key ([SPEC §4.7](/SPEC.md#47-proof)).

```json
{
  "id": "urn:uuid:6d7e8f9a-0b1c-4d2e-8f3a-4b5c6d7e8f9a",
  "threadId": "urn:uuid:1c2d3e4f-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
  "type": "https://trusttasks.org/spec/vta/attestation/report/0.1#response",
  "issuer": "did:webvh:QmExampleScid:vta.example.com",
  "issuedAt": "2026-09-26T12:00:00Z",
  "payload": {
    "teeType": "nitro",
    "evidence": "hEShATgioFkRXqlpbW9kdWxlX2lkeCdpLTAxMjM0NTY3ODlhYmNkZWYwLWVuYzAxMjM0NTY3ODlhYmNkZWY",
    "nonce": "8f14e45fceea167a5a36dedd4bea2543a1f0b1c2d3e4f5a6b7c8d9e0f1a2b3c4",
    "vtaDid": "did:webvh:QmExampleScid:vta.example.com",
    "generatedAt": "2026-09-26T12:00:00Z"
  }
}
```

- `evidence` is the platform's quote, base64: a COSE_Sign1 attestation document for `nitro`, an attestation report for `sev-snp`.
- `nonce` echoes the request. `vtaDid` is the DID the evidence binds as its user data.
- `generatedAt` is informational. Freshness comes from the nonce, not from this time.

## Verifying

A verifier **MUST**, before it relies on anything the agent attests:

1. Check the evidence's signature chains to the platform vendor's root (the AWS Nitro root; the AMD root and the chip's endorsement key).
2. Check the measured image (`PCR0` for `nitro`; the launch measurement for `sev-snp`) is one it approved.
3. Check the nonce inside the evidence equals the nonce it sent, byte for byte. The echoed `nonce` is not evidence.
4. Check the DID bound inside the evidence equals the response document's `issuer`, and that the document's proof verifies as that DID.

A `simulated` report **MUST** be refused, except by a development tool told to accept one.

## Security & Privacy

### Data carried

The request carries a nonce the verifier generated. The response carries the platform's quote, which names the measured image and the platform and binds the nonce and the agent's DID, plus an echo of the nonce and informational timing. None of it is personal data.

### Correlation

The nonce is fresh per request and meaningless outside it, and the evidence binds the agent's public DID, which the verifier already knows. A verifier learns nothing that links one request to another. The agent learns only that someone asked, from whatever the transport tells it.

### Retention

Neither side needs to keep the exchange. The evidence proves freshness for one nonce; a verifier that wants to rely on the agent again asks again.

### Consent/purpose

Evidence is public by design: attestation exists so that anyone can check what code an agent runs. The agent speaks for itself; no subject's data is involved.

### Threats

The request is unauthenticated on purpose. What a forged or replayed request can obtain is evidence for a nonce its sender chose — which is no use to anyone else, because every verifier checks its own nonce. The recipient's cost is a quote per request, which rate-limiting bounds.

An intermediary can drop or delay the response but cannot alter it: the document proof covers the payload, and the evidence carries its own vendor signature.
