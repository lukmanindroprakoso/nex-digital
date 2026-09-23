# Scope Guard — Stage 2 Assignment (AI Solutions Engineer)

Micro AI App that helps Indonesian homeowners and small contractors avoid
disputes over project scope — catching ambiguous or out-of-scope requests
mid-project, before they escalate into a fight.

## Problem

Small residential construction/renovation projects in Indonesia run on
verbal or WhatsApp-based agreements. Recurring conflict points: scope
ambiguity, scope creep ("tambah sedikit saja"), and no structured way to
confirm whether a mid-project request is a genuine change or already
covered.

Full research, evidence, and confidence levels: see [`docs/presentation.pptx`](docs/presentation.pptx).

## How it works

**Stage 1 — Baseline scope.** Either party describes the project in free
text (price, timeline, work items). An AI agent (Gemini) extracts a
structured baseline: scope items, exclusions, price, timeline.

**Stage 2 — Change evaluation.** Any later message is checked against that
baseline and classified as one of:

- `in-scope` — already covered, no cost impact
- `out-of-scope` — a genuinely new item
- `ambiguous` — modifies an existing item in a way that may affect cost
  (triggers a clarifying question instead of guessing)
- `not-scope-related` — general chat, answered naturally

Classification reasons about _substance_, not keyword match — e.g. "ganti
keramik dinding" → "ganti jadi granit" is `ambiguous`, not `in-scope`, even
though the same item is named, because granite materially changes cost.

## Architecture

```
Chat UI (role toggle: Homeowner / Contractor)
        │  POST { session_id, role, message }
        ▼
n8n Webhook
        │
        ▼
Airtable: does a baseline exist for this session_id?
        │
   ┌────┴────┐
   │ No      │ Yes
   ▼         ▼
Stage 1    Stage 2
(Gemini    (Gemini
 extract)   classify)
   │         │
   ▼         ▼
Airtable   out-of-scope / ambiguous →
 write      logged as change request
   │         │
   └────┬────┘
        ▼
Respond to Webhook → Chat UI
```

- **Workflow engine:** n8n
- **AI layer:** Google Gemini (via n8n AI Agent nodes), two system prompts —
  one for extraction, one for classification
- **Storage:** Airtable (`BaselineScope`, `ChangeRequests` tables) — swapped
  in mid-build after a Supabase outage; kept for the rest of the build
  since it was reliable and fast enough for a demo
- **Frontend:** single-page HTML/CSS/JS chat UI, no framework, no build
  step — built for zero-setup risk during a live demo

## Repo structure

```
/docs/presentation.pptx        — presentation deck (problem, approach, architecture, etc.)
/frontend/index.html           — chat UI
/workflow/n8n-export.json      — full n8n workflow export
/prompts/stage1-extraction.md  — Gemini system prompt, baseline extraction
/prompts/stage2-classification.md — Gemini system prompt, change classification
README.md
AI-USAGE.md
```

## Known limitations (by design, not oversight)

- The role-toggle chat UI simulates two parties in one session — a demo
  simplification, not real multi-user infrastructure. Explained further in
  the presentation.
- Stage 1 (baseline intake) is a structured single-turn extraction, not a
  multi-turn conversational intake — deliberate, to protect build time for
  Stage 2's classification logic, which is the actual differentiator.
- No production-grade auth, retry/observability, or WhatsApp integration —
  named as next steps in the presentation, not built for this timebox.

## Running it

1. Import `workflow/n8n-export.json` into n8n, connect Gemini and Airtable
   credentials, activate the webhook.
2. Create the Airtable base with `BaselineScope` and `ChangeRequests`
   tables (fields listed in the workflow's Airtable nodes).
3. Open `frontend/index.html`, set the webhook URL at the top of the file.
4. Send a baseline message first, then any follow-up message to test
   classification.
