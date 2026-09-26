---
slug: vta/attestation/status
version: "0.1"
title: "VTA Attestation — Status"
summary: "Ask an agent whether it runs in a trusted execution environment, and which kind, before relying on anything it attests."
status: draft
targetFrameworkVersion: "0.5.0"
category: provenance
keywords:
  - attestation
  - tee
  - nitro
  - sev-snp
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
    The request is asked before the verifier trusts the agent, often by a party the agent has never heard of and which holds no key it would recognise, and it asks for nothing but a public fact about the agent's platform. Requiring a proof would gate a public read on an identity nobody needs to have. The response is REQUIRED because it is the agent's statement about itself: unsigned, an intermediary could report an enclave where there is none and steer the verifier into asking for evidence it then does not check.
issuedAtRequirement:
  requirement: OPTIONAL
  rationale: >-
    Nothing is executed and nothing is spent, so a replayed request is answered with the same public fact; there is no window to bound.
sideEffects:
  level: none
  rationale: "Read-only: the agent reports what it detected at boot."
exposure:
  discloses: metadata
  ingests: none
  actsAsSubject: false
  rationale: >-
    The platform kind and version fingerprint the deployment — which is the point, since a verifier needs to know what evidence to ask for — and nothing else.
retention:
  class: transient
  rationale: The answer is consumed to decide the next request and is worth nothing once the verifier has asked for evidence.
errorCodes:
  - code: vta/attestation/status:notAttested
    meaning: This agent has no attestation provider — it does not run in a trusted execution environment, or attestation is disabled.
    retryable: false
related:
  - vta/attestation/report
  - vta/attestation/config-report
  - vta/attestation/mnemonic-export
---

## Abstract

An agent that runs inside a trusted execution environment (TEE) can prove what code it runs and on what platform. Before a verifier asks for that proof ([`vta/attestation/report`](../../report/0.1/)), it needs to know whether there is one to ask for, and of which kind. This task answers that: the TEE the agent detected at boot, and whether it detected one at all.

The answer is the agent's claim, not evidence. A verifier **MUST NOT** treat `detected: true` as attestation; it tells the verifier which evidence to request and how to verify it.

## Status of this Document

This specification is a **draft** ([SPEC §5.3](/SPEC.md#53-maturity-levels)). It targets framework version 0.5.0 and may change without a version bump while it remains a draft ([SPEC §5.2](/SPEC.md#52-compatibility-rules)).

## Conformance

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY** and **OPTIONAL** in this document are to be interpreted as described in [BCP 14](https://www.rfc-editor.org/info/bcp14) when, and only when, they appear in all capitals.

A conforming producer and consumer satisfy [SPEC §7.1 and §7.2](/SPEC.md#7-minimum-requirements) in addition to the requirements stated here.

## Authorization

None. The request asks for a public fact about the agent and the agent answers anyone who asks, including a party it cannot identify. A recipient **MUST NOT** refuse the request for want of a proof, an ACL entry or a session, and **MUST NOT** vary its answer by who asked.

A recipient **MAY** rate-limit the request per transport peer, as it would any unauthenticated read.

## Request

The payload carries nothing; the request is the question.

```json
{
  "id": "urn:uuid:5f0b6a4e-2c1d-4b8e-9a3f-7d6e5c4b3a21",
  "type": "https://trusttasks.org/spec/vta/attestation/status/0.1",
  "recipient": "did:webvh:QmExampleScid:vta.example.com",
  "payload": {}
}
```

## Response

The recipient answers with the `$defs.Response` payload of [`payload.schema.json`](payload.schema.json), in a document it signs with its own `authentication` key ([SPEC §4.7](/SPEC.md#47-proof)).

```json
{
  "id": "urn:uuid:9a8b7c6d-5e4f-4a3b-8c2d-1e0f9a8b7c6d",
  "threadId": "urn:uuid:5f0b6a4e-2c1d-4b8e-9a3f-7d6e5c4b3a21",
  "type": "https://trusttasks.org/spec/vta/attestation/status/0.1#response",
  "issuer": "did:webvh:QmExampleScid:vta.example.com",
  "issuedAt": "2026-09-26T12:00:00Z",
  "payload": {
    "teeType": "nitro",
    "detected": true,
    "platformVersion": "aws-nitro-enclaves"
  }
}
```

- `teeType` names the platform whose evidence [`vta/attestation/report`](../../report/0.1/) returns: `nitro` (AWS Nitro Enclaves), `sev-snp` (AMD SEV-SNP), or `simulated` (a development build that fabricates evidence no verifier should accept).
- `detected` is `false` when the agent was configured for a platform it did not find at boot. It then produces no evidence.
- A recipient with no attestation provider at all answers with the `vta/attestation/status:notAttested` error rather than a response.

A verifier **MUST** refuse a `simulated` platform unless it is itself a development tool that has been told to accept one.

## Security & Privacy

### Data carried

The request carries nothing. The response carries the platform kind (`nitro`, `sev-snp`, `simulated`), whether the agent detected it at boot, and the platform's version string. None of it is personal data; all of it is about the deployment.

### Correlation

The agent's DID is already public — it is the recipient — and the answer adds only which platform it runs on, the same for every asker. Nothing in it lets a verifier link this request to any other.

### Retention

Neither side needs to keep anything. The answer is consumed to choose the next request, and a verifier asks again when it next needs to know.

### Consent/purpose

The agent discloses its platform to anyone, by design: a verifier cannot ask for the right evidence without it. There is no subject whose consent applies; the agent speaks for itself.

### Threats

The response is a claim, not evidence. Its signature attributes it to the agent's DID; it proves nothing about the hardware. An attacker who controls an agent's host but not an enclave can make the agent say anything here. Only the evidence of [`vta/attestation/report`](../../report/0.1/), verified against the platform vendor's root, says what code is running.
