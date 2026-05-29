---
id: data-protection-policy
title: Data Protection Policy
doc_type: security
owner: security-team
source: confluence://security/data-protection-policy
last_updated: 2025-02-14
effective_from: 2025-02-14
effective_to: null
superseded_by: null
sensitivity: internal-only
approver: aarushi.rathi
tags: [encryption, data-protection, kms]
sections:
  - id: "3"
    title: "Scope"
  - id: "4.1"
    title: "Encryption at rest"
  - id: "4.2"
    title: "Encryption in transit"
  - id: "5"
    title: "Key management"
---

# Data Protection Policy

> **NOTE:** This is a placeholder example showing the policy schema. The
> contents below are illustrative, not the authoritative Remerge policy.
> Replace with the real policy markdown before relying on it.

## 3. Scope

This policy applies to all production systems handling customer data.

## 4.1 Encryption at rest

All customer data is encrypted at rest using AES-256-GCM via AWS
KMS-managed keys. Keys are rotated annually.

## 4.2 Encryption in transit

All customer-facing endpoints serve TLS 1.2+ only. Internal service-to-
service traffic in production VPCs is encrypted with mutual TLS.

## 5. Key management

KMS key rotation cadence, dual-control for key deletion, and HSM
backing are detailed in the internal Key Management Standard. That
document is `internal-only` and must not be quoted verbatim to
customers — summarise via this policy's public statements only.
