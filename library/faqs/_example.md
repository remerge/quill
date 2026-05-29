---
id: encryption-at-rest
title: Data encryption at rest
canonical_question: "Do you encrypt customer data at rest?"
answer_format: yes_no
frameworks: [SIG-Lite, SIG-Core, CAIQ, SOC2-CC]
last_updated: 2025-02-14
effective_from: 2025-02-14
effective_to: null
superseded_by: null
sensitivity: public
approver: aarushi.rathi
tags: [encryption, data-protection, at-rest]
---

# Data encryption at rest

> **NOTE:** This is a placeholder example showing the FAQ schema. The
> contents below are illustrative, not authoritative Remerge claims.
> Replace with reviewer-approved content before relying on it.

## Canonical question

Do you encrypt customer data at rest?

## Answer

Yes

## Remark

All customer data is encrypted at rest using AES-256-GCM with keys
managed by AWS KMS. Keys are rotated annually per policy.

## Notes for the drafter

- Prefer this FAQ over any prior_answer on the same topic.
- If a question asks specifically about key-management cadence or KMS
  details, those are `internal-only` policy material — flag
  `confidentiality` on the draft.
