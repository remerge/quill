# Quill — Project Plan (v0.1, draft)

Status: **draft, internal, not for sharing yet.** This is the working plan
Aarushi can present to her team or to Bene when asked. It will keep evolving
as the system gets built.

---

## 1. The vision in one paragraph

Today, security questionnaires from customers land on the Jira support board
and require a human (currently the revenue team as first responder, then the
Security team) to open each one, read the questions, look up the answers in
past questionnaires and policies, fill them in, and reply. This is slow,
error-prone, and not the revenue team's core job. Quill replaces the
first-responder role with an AI agent that ingests any questionnaire format
(PDF / Word / Excel / vendor portal), retrieves answers from Remerge's own
library of approved past answers and current policies (with strict
recency-aware logic so 2025 policy beats 2024 prior answer), and produces a
reviewer-ready draft with citations and confidence. A dashboard gives the
Security team one-glance status across all open questionnaires. The SafeBase
trust profile remains the proactive share-with-client artifact and stays in
sync with whatever Quill answers.

---

## 2. Why now, and why not just use SafeBase

SafeBase is the trust-profile front door and is staying. But its
AI-questionnaire feature is capped at ~10 questionnaires and is generic — not
tuned to Remerge's policies, terminology, ad-tech context, or the specific
question patterns we get. Loopio / Responsive / Conveyor / Vanta
Questionnaire Assist all solve adjacent problems but each carries its own
licensing cost, integration shape, and lock-in. Quill is the bet that an
internally-owned, Claude-grounded agent — built on top of our own document
library — beats those tools on (a) volume cap, (b) answer quality on
Remerge-specific questions, and (c) integration depth with how we actually
work (Jira, SafeBase, Confluence). It also turns our years of past
questionnaire answers from a folder of dead PDFs into a living, searchable
asset.

---

## 3. The workflow Quill targets, end-to-end

1. **Questionnaire arrives.** Email, Jira form, vendor portal, or shared
   drive. A Jira ticket is created on the Security support board (today's
   intake point).
2. **Jira intake reply is drafted.** Quill picks one of the three templates
   in `prompts/jira_intake_reply.md` (first-time / follow-up /
   SafeBase-deflectable) and posts it as a Jira draft comment for Aarushi
   to review and send. The send itself is automated *outside* Quill — Aarushi
   owns that automation. Quill never sends customer-facing messages directly.
3. **Questionnaire-history check.** Before ingestion, Quill checks whether
   this client has a prior questionnaire on file (match on client name +
   domain + prior ticket metadata). The result drives:
   - Which intake-reply template was picked in step 2.
   - Whether the drafter treats this as a **follow-up** (anchor on prior
     answers, refresh only what changed) or **first-time** (full retrieval
     across the library). Follow-up is the optimised path — most repeat
     questionnaires are 70–90% the same questions.
4. **Ingestion.** Quill pulls the attached file(s) or, for portals, opens
   the questionnaire via API/scraper.
5. **Language normalisation.** If the questionnaire arrives in a language
   other than English (German is the common one), Quill translates each
   question to English before extraction. **All drafted answers are in
   English regardless of source-questionnaire language** — translation
   back into the requester's language, if requested, is a downstream step
   Aarushi decides on per-ticket. Original-language text is preserved
   alongside the English translation for reviewer reference.
6. **Question extraction.** A separate LLM call (the *question-extractor
   prompt*) turns the raw file into a structured list of question objects:
   text, expected format (yes_no, yes_no_maybe, short_text, long_text,
   single_select, evidence_only), framework if known (SIG, CAIQ, SOC2-CC,
   HECVAT, VSAQ, custom), section headings.
7. **Retrieval per question.** For each question, retrieve the most-relevant
   N chunks from the library, in priority order:
   - `<faqs>` — curated, reviewer-approved canonical Q&A.
   - `<policies>` — current Remerge policies and supporting documents (see
     §4 for the full list — security policies, HR policies, privacy policy,
     website T&Cs, sub-processor list, etc.).
   - `<prior_answers>` — past approved questionnaire answers, dated. For
     follow-up clients, the most recent prior answer from this client is
     boosted to the top of retrieval.
   - `<trust_profile>` — SafeBase public claims.
8. **Drafting.** The *answer-drafter prompt* (lives in
   `prompts/answer_drafter.md`) returns a structured JSON draft per
   question: `answer`, `remark`, `answer_format`, `confidence`, `citations`,
   `review_flags`, `library_suggestion`, `reviewer_notes`. Where the
   library has no usable answer, the draft returns
   `status="INSUFFICIENT_CONTEXT"` and the question is routed to the
   "needs Aarushi input" bucket on the Jira ticket.
9. **Review on the Jira ticket.** Quill posts the full set of drafts as a
   structured comment on the Jira ticket (drafts table + per-question
   citations + flags). **Aarushi and Julie are both tagged as required
   reviewers.** They review per-question (Yes / No / Maybe + remark for
   each), edit in place, and approve. Quill never submits a questionnaire
   to the client — approval is an explicit human action on the Jira ticket.
   Bulk-accept is available for HIGH-confidence FAQ-grounded drafts; everything
   else gets per-item review.
10. **End-of-questionnaire attachment summary.** When all questions are
    drafted, Quill posts a separate Jira comment listing every document
    the questionnaire asked about (pen test report, SOC 2 report, ISMS
    certificate, etc.) so Aarushi knows which attachments to gather and
    send alongside the approved answers. Quill itself does not attach or
    send these — it just lists them.
11. **Approval and export.** Once both reviewers approve on the Jira ticket,
    approved answers are exported back into the original format (filled
    PDF / Excel / Word) or written to the vendor portal. Aarushi triggers
    the send.
12. **Library write-back.** Once the Jira ticket is marked approved,
    every approved draft becomes a dated `prior_answer` in the library.
    FAQ-worthy ones get promoted via one-click "promote to FAQ" on the
    Jira comment thread.
13. **Trust-profile diff queue.** When Quill drafts something that the
    SafeBase trust profile doesn't reflect (or contradicts), a "suggest
    SafeBase update" item lands in a separate queue.

---

## 4. The library design (the actual core of the product)

Quill is, more than anything else, a library plus a retrieval layer. The
drafter prompt is the easy part. The library is what determines whether the
drafts are any good.

Four collections, all dated, all versioned:

**FAQs** — Curated canonical Q&A. Vendor-agnostic phrasing. No client names.
Every entry has `last_updated`, `approver`, `effective_from`,
optional `effective_to`. This is the source of truth Quill leans on first.

**Policies** — Internal policy and public-document corpus. Broader than
just security policies — this is the full set Quill needs read access to:

- *Security policies:* ISMS, Information Security Policy, Vulnerability
  Management, Incident Response, Business Continuity / DR, Access Control,
  Encryption, Acceptable Use, Secure SDLC, Logging & Monitoring.
- *Data / privacy:* DPA, Privacy Policy, sub-processor list, data-retention
  policy, cross-border transfer mechanisms.
- *People / HR:* HR security policy, background-check policy, onboarding /
  offboarding procedures, security training records (policy-level, not
  individual records).
- *Public legal / commercial:* Website Terms & Conditions, MSA template,
  cookie policy.
- *Evidence-grade attachments* (treated as documents, not chunked text):
  most-recent pen test report, SOC 2 report, ISO 27001 / ISMS certificate,
  any other auditor-issued artefact. These are surfaced for the
  end-of-questionnaire attachment summary (§3 step 10), not used to draft
  prose answers.

Ingested from Confluence / Drive on a schedule. Chunked, embedded, and
stored with `last_updated` and `effective_from/to`. When a policy is
updated, prior chunks are kept but marked `superseded_by` so the recency
rule has something to point at. Each item carries a sensitivity tag
(public / NDA-gated / internal-only) so the drafter knows whether the
content is safe to quote in a customer-facing answer.

**Prior answers** — Every approved Quill draft, plus a historical backfill
of past manually-sent questionnaires. Each one has `answered_on`,
`approver`, `source_questionnaire`, `client_name` (redacted at retrieval
time so client names don't leak into new drafts), and the original
`question_text` + `answer_text` + `remark_text`.

**Trust profile** — A snapshot of current SafeBase public claims, refreshed
on a schedule. Used as a fallback source and as the diff target for
"should we update SafeBase?" suggestions.

Recency rule across all four: most-recent dated source wins; on ties, FAQ
beats policy beats prior_answer beats trust_profile.

---

## 5. Senior-perspective challenges (what makes this hard in real life)

These are the failure modes a senior person who's actually answered
questionnaires has seen — and what Quill needs to handle.

1. **Excel and PDF parsing is genuinely awful.** Excel questionnaires have
   merged cells, hidden rows, multi-tab structure, instructions on tab 1
   that change how tab 2 should be filled. PDFs often have scanned tables
   that need OCR. Real questionnaires arrive broken — Quill needs to fail
   gracefully and surface "parse failed" rather than silently miss
   questions.
2. **Same intent, different phrasing.** *"Do you encrypt data at rest?"* vs
   *"Is customer data encrypted when stored?"* vs *"What encryption standards
   are used for data persistence?"* Retrieval has to match on intent, not
   keywords. Hybrid (BM25 + embeddings) plus an intent-normalization step
   per question.
3. **Hallucinated certifications are a legal exposure.** If Quill says
   "Yes, FedRAMP authorized" when we aren't, that's a contract issue, not a
   quality issue. The drafter prompt's GROUNDED-ONLY rule is the first
   line of defense; an output-side hallucination check (every citation
   substring must literally exist in the cited source) is the second.
4. **Stale answers in customer hands.** We told Acme in 2024 that we
   "informally handle vulnerability management". In 2025 we have a formal
   policy. The 2024 letter is still in Acme's procurement folder. Quill
   needs to (a) never repeat the 2024 line, and (b) surface "Acme has the
   2024 version" as a follow-up the reviewer can act on. The trust-profile
   diff queue is half of this.
5. **Confidentiality boundaries.** Some answers are fine for the trust
   profile (we use AWS). Some need NDA gating (specific KMS key rotation
   cadence, our DR site location). Some should never leave Security
   (active investigation details, customer X's specific config). The
   `confidentiality` review flag plus per-document sensitivity tags handle
   this.
6. **Multi-part questions glued into one field.** *"Describe your encryption
   in transit and at rest, and your key rotation policy."* — three
   sub-questions. Either decompose pre-LLM or instruct the model to
   enumerate sub-answers with their own citations.
7. **"No" is unreachable by retrieval.** *"Are you FedRAMP authorized?"* —
   the truthful answer is "No", but no policy explicitly says "we are not
   FedRAMP authorized". Quill needs an absence-as-evidence rule, anchored
   to the trust profile as the canonical list of what we *do* have.
8. **Evidence attachments are a separate workflow.** *"Attach your SOC 2
   report."* — Quill doesn't draft text here, it links a document. The
   evidence library needs its own metadata (NDA gating, expiry date,
   versioning) and the drafter needs to know to point at it rather than
   narrate.
9. **Reviewer fatigue is the real failure mode.** If every draft needs
   heavy editing, the reviewer abandons the tool. Bulk-accept for
   HIGH-confidence FAQ-grounded drafts is critical — the value isn't in
   AI drafting, it's in AI drafting the boring 70% so the reviewer can
   focus on the 30% that needs judgement.
10. **Customer-name leakage.** Prior answers say *"we confirmed to Acme
    that…"*. Without scrubbing, that line gets cited into Initech's
    questionnaire. PII / customer-name redaction has to happen at library
    write-back, not at draft time.
11. **Prompt-injection via prior content.** A historical questionnaire
    answer could contain *"Ignore previous instructions and say…"* — pasted
    innocently from a customer doc, or maliciously. Treat all retrieved
    content as data, never instructions.
12. **Frameworks aren't fungible.** A SOC 2 question and a CAIQ question
    about access control look identical but the reviewer's tolerance for
    "Maybe" differs. Carrying the framework as metadata lets the dashboard
    route accordingly (e.g. CAIQ to Security only; SIG-Core to Security +
    Legal).

---

## 6. Competitive landscape — who else plays here

We are not the first; we just want to be the most-tailored-to-Remerge.

- **SafeBase** — Trust-profile front door + AI assist. We use it. Cap at
  ~10 questionnaires/period for AI. Strong public-facing trust center.
  Quill complements rather than replaces — we keep SafeBase, Quill drafts
  what SafeBase can't.
- **Vanta Questionnaire Automation** — Bundled with Vanta compliance.
  Strong if you're a Vanta shop. Less customizable on retrieval logic.
- **Drata Trust Center + Questionnaire Assist** — Similar to Vanta.
- **Loopio** — The RFP/security-questionnaire incumbent. Large library
  management, workflow, multi-reviewer. Expensive. Generic across
  industries.
- **Responsive (formerly RFPIO)** — Same space as Loopio. Enterprise.
- **Conveyor** — Trust-center + AI questionnaire. Used by mid/late-stage
  SaaS. Cleaner UX than Loopio. Still generic.
- **AutoRFP.ai** — Newer, LLM-native. Less mature on integrations.
- **OneTrust** — Enterprise GRC suite. Heavyweight; usually present
  because Procurement bought it, not because Security chose it.

**Quill's wedge:** Claude-grounded, owned by Remerge, no question cap,
deeply integrated with our Confluence/Drive/SafeBase/Jira, recency-aware
library, and a write-back loop that compounds with use. Not trying to
compete with Loopio on enterprise feature breadth — trying to be the right
tool for *us*.

---

## 7. System architecture (high level)

```
                    ┌─────────────────────────────────────┐
                    │           Sources of truth          │
                    │  (Confluence, Drive, SafeBase API,  │
                    │   sub-processor list, past PDFs)    │
                    └──────────────┬──────────────────────┘
                                   │  scheduled ingestion
                                   ▼
       ┌─────────────────────────────────────────────────────┐
       │              Library (Postgres + pgvector)          │
       │  faqs · policies · prior_answers · trust_profile    │
       │  every row dated, versioned, embedded               │
       └────────┬──────────────────────────────┬─────────────┘
                │ retrieval                    │ write-back
                ▼                              ▲
       ┌────────────────────┐         ┌────────┴───────────┐
       │  Answer drafter    │         │  Approved draft    │
       │  (Claude + cached  │         │  → library row     │
       │  system prompt)    │         │  → FAQ promotion   │
       └────────┬───────────┘         │  → SafeBase diff   │
                │                     └────────────────────┘
                ▼                              ▲
       ┌────────────────────┐         ┌────────┴───────────┐
       │  Per-question      │  ───▶   │  Jira ticket as    │
       │  JSON drafts       │         │  review surface    │
       └────────────────────┘         │  (Aarushi + Julie  │
                ▲                     │   tagged)          │
                │                     └────────┬───────────┘
                │                              │ approve / edit
                │                              ▼
       ┌────────┴───────────┐         ┌────────────────────┐
       │  Question          │         │  Export & write-   │
       │  extractor         │ ◀───────│  back to original  │
       │  (Claude)          │         │  format / portal   │
       └────────┬───────────┘         └────────────────────┘
                ▲
                │
       ┌────────┴───────────┐
       │  Ingestion +       │  ◀── PDF / Word / Excel / portal scraper
       │  language norm.    │      (translates to English if needed)
       └────────┬───────────┘
                ▲
                │
       ┌────────┴───────────┐
       │  Questionnaire     │
       │  history check     │  ◀── first-time vs follow-up routing
       └────────┬───────────┘
                ▲
                │
       ┌────────┴───────────┐
       │  Jira intake +     │  ◀── new ticket on Security support board
       │  reply draft       │      (3 templates, Aarushi-sent)
       └────────────────────┘
```

**Stack proposal (open to push-back) — local-first for v1:**
- Backend: Python script(s), runnable from the laptop. No FastAPI server
  in v1 — Quill is invoked manually or via a small CLI.
- Library storage: flat markdown files in this repo (one per policy / FAQ /
  prior answer), with a local SQLite index for retrieval. No Postgres,
  no pgvector, no hosted vector DB in v1.
- LLM: Claude via Anthropic API. Prompt caching for the system prompt.
- Doc parsing: `unstructured.io` / `Docling` for PDF/Word, `openpyxl` for
  Excel. Portal scrapers (Playwright) are deferred to Phase 3.
- Review surface: Jira ticket (no custom dashboard in v1 — drafts posted
  as a structured comment, reviewers approve in-place).
- Auth: not in scope for v1 — single developer runs Quill locally with a
  personal API key. Jira's own permissions govern who sees the drafts.
- Hosting: local laptop. AWS / hosted setup is a Phase 4 question, to be
  driven by real usage data.

---

## 8. Roadmap — build order

Sized so each phase is shippable and demonstrable on its own.

### Phase 0 — Foundation (weeks 0–2)
- Ingest current policies into the library (one-time bulk).
- Backfill ~20 past questionnaires as `prior_answers`.
- Build the **eval set**: 30–50 frozen questions with gold answers across
  yes_no / yes_no_maybe / short_text / long_text.
- Spike: answer-drafter prompt against the eval set, measure baseline.

**Demo-able artifact:** "Quill answered 40 of these 50 questions correctly
with citations, here are the failures."

### Phase 1 — Drafter + Jira-comment review surface (weeks 2–6)
- Lock prompt v1 (this repo).
- Build retrieval layer (hybrid BM25 + embeddings + date-aware ranking).
- Questionnaire-history check: match incoming client → prior questionnaire.
- Jira-comment posting: drafts table + per-question citations + flags +
  Aarushi-and-Julie tagging. No custom dashboard.
- End-of-questionnaire attachment summary (separate Jira comment).
- Library write-back on Jira-ticket approval.
- Jira auto-comment on ticket creation (the quick win, parallel track).

**Demo-able artifact:** "Drop a PDF, get drafts in 60 seconds, click
through to approve."

### Phase 2 — Ingestion expansion (weeks 6–10)
- Question-extractor prompt and pipeline for PDF, Word, Excel.
- FAQ promotion workflow (one-click from `library_suggestion`).
- Confidence calibration from accept/edit rates.
- SafeBase trust-profile diff queue.

**Demo-able artifact:** "Last 5 real questionnaires, end-to-end, reviewer
time per questionnaire."

### Phase 3 — Send-back + portal integration (weeks 10–16)
- Fill the original format back (Excel/Word/PDF write-back).
- First two vendor portal adapters (likely OneTrust + Whistic — based on
  what we actually see most).
- Evidence library with NDA gating.

### Phase 4 — Scale, hardening, broader rollout (weeks 16+)
- Optional dedicated dashboard if Jira-comment review becomes the
  bottleneck (RBAC, audit log, richer side-by-side citation view).
- Slack/email notifications around Jira-comment events.
- Sales-team `/quill` self-serve query.
- Reply translation: source-language send-back for non-English requesters
  (drafts stay English; translation is the last step before send).

---

## 9. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Hallucinated certification or SLA | GROUNDED-ONLY rule + verbatim-citation check + reviewer-in-the-loop. No auto-send. |
| Library quality is low ⇒ drafts are bad | Phase 0 invests in clean ingestion and the eval set before anything else. |
| Reviewer doesn't trust drafts, abandons | Bulk-accept for HIGH-confidence FAQ-grounded drafts; transparent citations; never auto-send. |
| Vendor portal adapters become a maintenance treadmill | Start with the two most-common portals only; for others, draft in dashboard and human-copies-paste to portal. |
| Prompt injection via prior content | Treat retrieved content as data; add an output-side check that quoted citation substrings exist verbatim. |
| Customer-name leakage across questionnaires | Scrubber at library write-back time; redaction at retrieval time as belt-and-braces. |
| SafeBase, our public source of truth, drifts from internal answers | Trust-profile diff queue surfaces it; quarterly reconciliation review. |
| One person builds it, one person knows it | Document this plan, the prompt, the library schema; pair on early phases. |

---

## 10. Open decisions for Aarushi to make

1. **Build vs buy on the dashboard.** *Resolved (2026-05-27):* skip the
   dashboard for v1. The Jira ticket is the review surface — Quill posts
   drafts as a structured Jira comment, reviewers approve there. Revisit
   a dedicated dashboard if Jira-comment ergonomics become the bottleneck.
2. **Where does the library live?** *Resolved (2026-05-27):* local-first.
   The library is a directory in this repo — flat markdown files (one per
   policy / FAQ / prior answer) with YAML frontmatter for the date and
   metadata, indexed into a single local SQLite file for retrieval. No
   Postgres, no pgvector, no Confluence-as-storage in this iteration.
   Embeddings, if needed for retrieval quality, are computed locally and
   cached alongside the SQLite index. Migration to a hosted store is a
   later decision driven by actual scale, not a v1 concern.
3. **Who reviews?** *Resolved (2026-05-27):* Aarushi AND Julie review every
   questionnaire from v1. Both are tagged on the Jira comment; approval
   requires both. Quill never submits to the client directly under any
   circumstance.
4. **How public is this project internally?** *Resolved (2026-05-27):*
   staged visibility. Stay invisible until Phase 0 has a demo-able eval
   result (the "40 of 50 questions correct" artifact). Share PLAN.md +
   the two prompt files with Bene first, then with Julie. Wider visibility
   (revenue team, leadership) waits until Phase 1 ships an end-to-end run
   on a real questionnaire.
5. **Naming.** *Resolved (2026-05-27):* "Quill" internally. The agent is
   never customer-facing in v1 (Aarushi sends, not Quill), so no external
   naming decision is needed yet. Revisit only if Quill ever produces
   artifacts a customer sees directly.
6. **Budget posture.** *Resolved (2026-05-27):* no numeric estimate before
   Phase 0 produces real usage data (eval-run token counts × expected
   questionnaire volume). Plan to share an order-of-magnitude number with
   Bene before any leadership-level announcement. Until then, the cost
   floor is one developer's API key running local spikes — negligible.

---

## 11. What to do next, concretely

§10 is fully resolved as of 2026-05-27. In priority order:

1. Approve (or push back on) the prompt v0.1 in
   `prompts/answer_drafter.md` and the three intake-reply templates in
   `prompts/jira_intake_reply.md`.
2. Lay out the local library structure in this repo — directories for
   `faqs/`, `policies/`, `prior_answers/`, `trust_profile/`, plus a
   `library.db` SQLite index. Pick the frontmatter schema for each.
3. Pull ~5 real policies and ~5 past questionnaires into the local
   library as seed content (anonymised where needed).
4. Stand up a runnable Python spike: load a fixture questionnaire, call
   the drafter prompt against the seeded library, print drafts to
   stdout. Smallest end-to-end path that proves the prompt works on
   real-shaped input.
5. Build the eval set (30–50 frozen Q&A pairs) and a scorer that runs the
   spike against it.
6. Confirm with Julie that she's the second reviewer and walk her through
   how drafts will land in Jira (mockup-only at this stage — no live Jira
   integration in the local prototype).
7. When ready to show Bene: this doc + the two prompt files + the spike's
   output on the eval set are the artifacts to share.
