# Quill — Local Library

Single source of truth: `quill_corpus.xlsx` (gitignored). Reviewers edit
in Excel; Quill reads it on startup.

## The workbook — 4 sheets

| Sheet | Purpose |
|---|---|
| `Prior Answers` | Past approved client answers. **Top priority** — reflects how we actually answer. `client_name = REDACTED`; client identity goes in `tags` as `source:<client>`. |
| `Policies` | Internal policies + public legal docs, one row per excerpt. |
| `FAQs` | Generic canonical Q&A. Fallback when no prior answer or policy applies. |
| `Evidence` | Pointers to attachments (pen test, SOC 2, ISO 27001). Never quoted — listed in the attachment summary. |

## Recency rule

Newest `last_updated` wins. On ties:
**`Prior Answers > Policies > FAQs > Evidence`**.

## Sensitivity

- `public` → quote freely.
- `nda-gated` → quote only after Aarushi confirms client NDA on file.
- `internal-only` → reviewer context only, never sent to client.

## Tags (comma-separated)

Always tag with what applies:
`source:<client>`, `framework:<sig-lite|caiq|vsa-core|...>`,
`date:<YYYY-MM>`, plus topical tags (`encryption`, `gdpr`, etc.).

## Review workflow — Quill never auto-sends

1. Quill drafts into a **working copy** of the client's questionnaire,
   citing rows from `quill_corpus.xlsx`.
2. **Aarushi + Julie review** (two-reviewer rule).
3. Durable corrections get written back into `quill_corpus.xlsx` so
   future drafts inherit them.
4. **Only after both reviewers approve**, Quill uploads the reviewed
   workbook back to the questionnaire (portal / Jira / email).

If approval is missing, Quill stops and asks.

## Git policy

- **Committed:** this README and per-sheet `_example.md` schema files.
- **Gitignored:** `quill_corpus.xlsx`, `_raw/`, `library.db*`.
