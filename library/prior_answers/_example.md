---
id: 2024-08-acme-sig-core__1.4
questionnaire_id: 2024-08-acme-sig-core
client_name: REDACTED
client_domain: REDACTED
framework: SIG-Core
question_id: "1.4"
question_text: "Do you encrypt customer data at rest?"
answer_format: yes_no
answer_text: "Yes"
remark_text: "Customer data is encrypted at rest using AES-256-GCM with AWS KMS-managed keys."
answered_on: 2024-08-10
last_updated: 2024-08-10
effective_from: 2024-08-10
effective_to: null
superseded_by: null
sensitivity: internal-only
approver: aarushi.rathi
tags: [encryption, at-rest]
---

# Prior answer — encryption at rest

> **NOTE:** This is a placeholder example showing the prior_answer
> schema. Client name and domain are stored as `REDACTED` in the file
> body and frontmatter. The un-redacted mapping lives in the
> questionnaire-level metadata only, never in this file.

## Original question (as asked by client)

> Do you encrypt customer data at rest?

## Answer sent

**Yes** — Customer data is encrypted at rest using AES-256-GCM with AWS
KMS-managed keys.

## Reviewer context (internal only — not for customer eyes)

This client (REDACTED) is on the mid-market tier. The encryption answer
is the same across all tiers, so no tier-conditional caveat was added.

## Why this file exists

This is what a single `prior_answer` looks like after a questionnaire
is approved and written back to the library (PLAN.md §3 step 12).
Retrieval ranks prior_answers by `answered_on` and demotes them under
any newer FAQ or policy on the same topic (recency rule, PLAN.md §4).
