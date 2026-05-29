# Quill — Jira Intake Reply Templates (v0.1)

These are the templates Quill drafts as the **first** comment on a new Jira
security-questionnaire ticket. The goal is to set expectations with the
requester and, where possible, route them to the SafeBase trust profile so
they can self-serve before a human gets involved.

**Important:** Quill drafts; Quill does **not** auto-send. Aarushi (and Julie,
where applicable) review the draft on the Jira ticket before it goes out.
The auto-sending of the chosen template is a separate automation Aarushi
owns and will wire up outside of Quill.

The three templates below cover the three intake shapes we actually see:
1. First-time questionnaire from a new or unknown client.
2. Follow-up / repeat client (we have a prior questionnaire on file).
3. SafeBase-deflectable — the questionnaire looks generic enough that the
   trust profile may answer most of it before we even draft.

Quill picks one based on the questionnaire-history check (see PLAN.md §3).

---

## Template A — First-time questionnaire

**When to use:** Client has no prior questionnaire on file, OR client name
is unknown / not yet matched to a prior record.

```text
Subject: Security questionnaire — {{client_name}}

Hi {{requester_first_name}},

Thanks for sending this over — the Remerge Security team has picked it up
and we'll get back to you with a completed response within {{sla_business_days}}
business days.

While we work on the questionnaire, a lot of the standard questions
(certifications, sub-processors, encryption, sub-processor list, data
residency, incident response) are already answered on our public trust
profile:

  {{safebase_url}}

If it's useful to your reviewers, they can request access there directly
and may not need to wait for the full questionnaire.

If anything in the questionnaire is time-critical, please flag it in this
thread and we'll prioritise.

Best,
Remerge Security
```

---

## Template B — Follow-up / repeat client

**When to use:** Quill's questionnaire-history check matched this client to
one or more prior completed questionnaires.

```text
Subject: Security questionnaire — {{client_name}} (refresh)

Hi {{requester_first_name}},

Thanks — we have your previous Remerge security questionnaire on file
(last completed {{prior_answered_on}}). The Security team will refresh
those answers against our current policies and return the updated
response within {{sla_business_days}} business days.

If most of your questions are unchanged from last time, our public trust
profile reflects our current posture and may be enough on its own:

  {{safebase_url}}

If your team has a deadline we should know about, flag it here and we'll
work to that.

Best,
Remerge Security
```

**Notes for the reviewer:** Quill will tag the prior questionnaire ID in
the Jira ticket so the reviewer can open it alongside the new draft.

---

## Template C — SafeBase-deflectable

**When to use:** The intake looks like a short, standard, vendor-agnostic
questionnaire (e.g. a starter SIG-Lite or a generic vendor security
checklist) where >70% of expected questions match items already on the
trust profile. Quill flags this case on the ticket; reviewer decides
whether to deflect or draft.

```text
Subject: Security questionnaire — {{client_name}}

Hi {{requester_first_name}},

Thanks for sending this. Before we draft a full response, our public
trust profile already covers most of what this questionnaire asks about
— certifications, encryption, data handling, sub-processors, and our
incident-response posture:

  {{safebase_url}}

Many of our customers complete their review entirely from the trust
profile. Could you take a look first and let us know which specific
questions (if any) it doesn't answer for you? We'll then draft a focused
response just for those, which usually turns this around much faster
than a full questionnaire pass.

If a full questionnaire response is required regardless (e.g. for
procurement records), reply to this thread and we'll proceed within
{{sla_business_days}} business days.

Best,
Remerge Security
```

---

## Variables Quill fills in

| Variable | Source |
|---|---|
| `{{client_name}}` | Jira ticket field / extracted from intake email |
| `{{requester_first_name}}` | Jira reporter; fallback "there" |
| `{{sla_business_days}}` | Default 5; overridable per-ticket |
| `{{safebase_url}}` | Constant — set once in Quill config |
| `{{prior_answered_on}}` | Date of most recent matched prior questionnaire (Template B only) |

---

## What Quill does NOT do here

- Does not commit to a specific completion date beyond the SLA window.
- Does not name internal staff or routing details.
- Does not promise NDA-gated documents (SOC 2, pen test) in the reply —
  those are sent only after the reviewer approves and verifies NDA status.
- Does not auto-send. Aarushi reviews on the Jira ticket and triggers the
  send via her own automation.
