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
2. **Jira auto-comment fires.** The ticket gets a templated comment within
   seconds: *"Thanks — Security is on this. Meanwhile, please share our
   SafeBase trust profile at [link] with the requester; many questions are
   already answered there."* This is the headline near-term win, even before
   the rest of Quill is built.
3. **Ingestion.** Quill pulls the attached file(s) or, for portals, opens
   the questionnaire via API/scraper.
4. **Question extraction.** A separate LLM call (the *question-extractor
   prompt*) turns the raw file into a structured list of question objects:
   text, expected format (yes_no, yes_no_maybe, short_text, long_text,
   single_select, evidence_only), framework if known (SIG, CAIQ, SOC2-CC,
   HECVAT, VSAQ, custom), section headings.
5. **Retrieval per question.** For each question, retrieve the most-relevant
   N chunks from four sources, in priority order:
   - `<faqs>` — curated, reviewer-approved canonical Q&A.
   - `<policies>` — current Remerge policies (DPA, ISMS, vuln mgmt, BCP, IR,
     access, encryption, T&Cs).
   - `<prior_answers>` — past questionnaire answers we sent, dated.
   - `<trust_profile>` — SafeBase public claims.
6. **Drafting.** The *answer-drafter prompt* (lives in
   `prompts/answer_drafter.md`) returns a structured JSON draft per
   question: `answer`, `remark`, `answer_format`, `confidence`, `citations`,
   `review_flags`, `library_suggestion`, `reviewer_notes`.
7. **Review dashboard.** All drafts for the questionnaire surface in a
   single view. Reviewer (Aarushi) sees the question, the draft, the cited
   source quotes side-by-side, confidence and flags, and can accept / edit /
   reject. Bulk-accept for HIGH-confidence FAQ-grounded drafts; per-item
   review for everything else.
8. **Approval and export.** Approved answers are exported back into the
   original format (filled PDF / Excel / Word) or written back to the
   vendor portal.
9. **Library write-back.** Approved drafts feed the library — the FAQ-worthy
   ones get promoted (one-click "promote to FAQ"), and every answer becomes
   a dated `prior_answer` for future retrieval.
10. **Trust-profile diff queue.** When Quill drafts something that the
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

**Policies** — Internal policy documents (DPA, T&Cs, vuln mgmt, IR, BCP,
access control, encryption, sub-processors, etc.). Ingested from
Confluence/Drive on a schedule. Chunked, embedded, and stored with
`last_updated` and `effective_from/to`. When a policy is updated, prior
chunks are kept but marked `superseded_by` so the recency rule has
something to point at.

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
       │  Per-question      │  ───▶   │  Review dashboard  │
       │  JSON drafts       │         │  (Next.js, RBAC)   │
       └────────────────────┘         └────────┬───────────┘
                ▲                              │ approve / edit
                │                              ▼
       ┌────────┴───────────┐         ┌────────────────────┐
       │  Question          │         │  Export & write-   │
       │  extractor         │ ◀───────│  back to original  │
       │  (Claude)          │         │  format / portal   │
       └────────┬───────────┘         └────────────────────┘
                ▲
                │
       ┌────────┴───────────┐
       │  Ingestion         │  ◀── PDF / Word / Excel / portal scraper
       │  adapters          │
       └────────────────────┘
                ▲
                │
       ┌────────┴───────────┐
       │  Jira intake +     │  ◀── new ticket on Security support board
       │  auto-comment      │
       └────────────────────┘
```

**Stack proposal (open to push-back):**
- Backend: Python (FastAPI) — best ecosystem for document parsing.
- DB: Postgres + pgvector. Simple, ops-known.
- LLM: Claude via Anthropic API. Prompt caching for the system prompt.
- Doc parsing: `unstructured.io` / `Docling` for PDF/Word, `openpyxl` for
  Excel, Playwright for portal scrapers.
- Frontend: Next.js dashboard.
- Auth: Okta/Google SSO (whatever Remerge already uses).
- Hosting: AWS (whatever account Security already operates in).

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

### Phase 1 — Drafter + minimal dashboard (weeks 2–6)
- Lock prompt v1 (this repo).
- Build retrieval layer (hybrid BM25 + embeddings + date-aware ranking).
- Build minimal review dashboard: question / draft / sources / accept-edit.
- Library write-back on approval.
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
- RBAC (Security / Legal / Sales view levels).
- Audit log.
- Slack/email notifications.
- Sales-team `/quill` self-serve query.
- Multi-language (German first).

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

1. **Build vs buy on the dashboard.** Lightweight Next.js dashboard
   (4–6 weeks engineering) vs buying Loopio/Conveyor and integrating our
   library. Lean: build, because the library is the moat.
2. **Where does the library live?** Internal Postgres (recommended), or
   ride on Confluence/Notion as the storage layer (cheaper, messier)?
3. **Who reviews?** Aarushi solo for v1; multi-reviewer RBAC in Phase 4.
   Confirm this is OK.
4. **How public is this project internally?** Right now: invisible until
   ready. When does it become visible to revenue team / leadership?
5. **Naming.** "Quill" is fine internally. If it ever becomes a product
   surface customers see, name needs a re-think.
6. **Budget posture.** Claude API costs scale with questionnaire volume
   and library size. Order-of-magnitude estimate to do once Phase 0 is
   real, but plan for it before announcing.

---

## 11. What to do next, concretely

In priority order:

1. Decide on the open questions in §10, or at least the top 3.
2. Approve (or push back on) the prompt v0.1 in
   `prompts/answer_drafter.md`.
3. Identify ~10 past questionnaires to use as the Phase 0 backfill set.
4. Confirm Confluence/Drive locations for current policies so ingestion
   can begin.
5. When ready to show Bene: this doc + the prompt are the two artifacts.
