---
id: pentest-2025-q1
title: External penetration test report — 2025 Q1
doc_kind: pentest
issued_on: 2025-03-20
expires_on: 2026-03-20
nda_required: true
storage_url: drive://security/pentests/2025-q1/report.pdf
topics: [pentest, vulnerability-management, application-security]
last_updated: 2025-03-20
effective_from: 2025-03-20
effective_to: null
superseded_by: null
sensitivity: nda-gated
approver: aarushi.rathi
tags: [pentest, 2025-q1]
---

# External penetration test report — 2025 Q1

> **NOTE:** This is a placeholder example showing the evidence schema.
> No actual report is stored in this repo — `evidence/` entries are
> *pointers* to the real attachment, not the binary itself.

## What this entry represents

A single piece of evidence-grade attachable content. Quill does **not**
quote from these documents — it surfaces them in the end-of-questionnaire
attachment summary (PLAN.md §3 step 10) so Aarushi knows what to gather
and send alongside the approved answers.

## When the drafter touches this entry

- A question of `expected_format = evidence_only` matches one of the
  `topics` — the drafter responds with `"See attached: <title>"` and
  citations point at this entry's `id`.
- A question is generally about a topic listed here — the entry shows
  up in the attachment summary at the end of the questionnaire.

## NDA gating

`nda_required: true` means this attachment can only be sent after
Aarushi has confirmed the requesting client has a current NDA on file.
The drafter does not block, but it flags `confidentiality` so the
reviewer is reminded.

## Freshness

`expires_on` tells the reviewer when this document goes stale. Past the
expiry, Quill flags evidence references with `stale_source` so the
reviewer knows to either refresh the document or temper the answer.
