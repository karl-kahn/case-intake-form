# Proposal: User-Editable Case Context Document

**From:** Karl
**To:** Lucas
**Date:** 2026-03-02
**Re:** Adding a human-in-the-loop document between document ingestion and response drafting

---

## The Idea

Right now the skill's Stage 2 (Clarifying Questions) works as a back-and-forth conversation: Claude reads the case documents, figures out what it knows and doesn't know, then asks the attorney questions to fill gaps. The answers go into `case_context.json`, which Stage 3 reads as its source of truth for drafting.

The proposal: replace that conversational Q&A with a **user-editable document** that Claude pre-populates from whatever it harvests from the case documents, then hands to the attorney to review, correct, and supplement before processing continues.

The flow becomes:

```
[Case Documents]
      │
      ▼
  Claude ingests & extracts
      │
      ▼
[Pre-filled Intake Document]  ← attorney reviews, edits, fills gaps
      │
      ▼
  Claude reads back as source of truth
      │
      ▼
[Draft Responses]
```

Instead of:

```
[Case Documents]
      │
      ▼
  Claude ingests
      │
      ▼
  Claude asks Q1 → attorney answers → Claude asks Q2 → attorney answers → ...
      │
      ▼
  case_context.json (invisible to attorney)
      │
      ▼
[Draft Responses]
```

I've mocked up what the intake document looks like as both a printable markdown sheet and a web form:
- **Web form (live):** https://karl-kahn.github.io/case-intake-form/
- **Markdown version:** `cases/gsdat-docs/CASE_INTAKE_SHEET.md`

The web form has an "Export as JSON" button that produces a `case_context.json` the engine can already consume. But the format of the editable doc itself (web form, markdown, Word, whatever) is an implementation detail — the point is the pattern.

---

## What We Gain

### 1. Attorney can review everything at once
The conversational Q&A is linear — you answer one question, then the next, and you can't easily go back. A document lets the attorney see the full picture, spot inconsistencies, and edit anything in any order. Jeff is a litigator; he reviews documents, he doesn't do chat workflows.

### 2. Claude's extractions become auditable
Right now if Claude misreads a police report or gets an injury list wrong, the attorney only catches it when reviewing the final drafted responses — at which point it's harder to trace back to the source. With an intermediate document, the attorney sees "Claude thinks the injuries are X, Y, Z" and can correct it *before* 20 responses get drafted wrong.

### 3. Gaps are visible up front
If Claude can't find wage records or doesn't know if there's a Medicare layer, the intermediate document shows those as blank fields. The attorney fills them in or marks them as N/A. No back-and-forth, no "I need more information" loops.

### 4. The document persists and accumulates
`case_context.json` already supports reuse across doc types (do FI first, then SI reuses the same context). The editable document makes this explicit — it's a case cover sheet that grows as the attorney works through different doc types. New sessions read it back, only ask about what's new.

### 5. It works offline
An attorney or paralegal can fill out the intake sheet on paper, on a plane, between court appearances. They don't need Claude running. When they come back, they hand the completed sheet to the skill and say "go."

---

## What We Lose (and whether it matters)

### 1. Adaptive follow-up questions
The conversational Q&A can ask smart follow-ups: "You said no wage loss — but you disclosed employment in 2.6. Should 8.x reference that?" A static form can't do this.

**Mitigation:** The form can be a *first pass*. After Claude reads the completed form, it can still ask a short list of targeted follow-ups based on what the attorney wrote. The difference is those follow-ups are 2-3 questions instead of 15, and they're informed by what's already on the form. Think of it as: form handles the 80% that's mechanical, Claude handles the 20% that needs judgment.

### 2. Cross-reference intelligence
Claude currently does things like "extract body parts from 6.2, search medical records for pre-accident complaints to the same body parts, then draft 10.x." An intake form can ask "prior treatment to same body parts?" but it can't do the cross-reference work.

**Mitigation:** This is engine logic, not intake logic. The form captures the attorney's knowledge ("yes, prior lower back treatment 2019-2020"). Claude still does the cross-referencing during Stage 3 processing. The form doesn't replace the engine's intelligence — it replaces the *data collection*.

### 3. JSON structure
The current `case_context.json` is structured data with nested objects, arrays, and typed fields. A human-editable document is inherently less structured.

**Mitigation:** The editable document is the *input*. Claude reads it and writes `case_context.json` from it — same as it does today from conversational answers. The schema doesn't change. The form just becomes a friendlier front door to the same data.

### 4. Per-doc-type scoping
The current Q&A only asks questions relevant to the selected document type (FI needs different context than SI). A single intake form asks everything.

**Mitigation:** This is actually fine. The form is short (one page). Asking a few questions that don't apply to this particular doc type costs the attorney 10 seconds of skipping. And the answers persist — so when they come back for the next doc type, those fields are already filled.

---

## What This Means for the Skill Architecture

### Changes needed

1. **New stage between Stage 1 and Stage 2:** After `scan_case.py` inventories the folder and Claude ingests the documents, Claude writes a pre-filled intake document (populating what it can from documents, leaving blanks for what it can't).

2. **Stage 2 shrinks:** Instead of a full Q&A session, Stage 2 becomes: "Read the completed intake form → write `case_context.json` → ask 2-3 targeted follow-ups if needed → proceed."

3. **`case_context_schema.md` needs new fields:** The intake form I mocked up adds fields the current schema doesn't have:
   - `matter_type` (litigation / UIM arbitration / other)
   - `party_designation` (Plaintiff / Claimant / Cross-Complainant)
   - `opposing_designation` (Defendant / Respondent / Cross-Defendant)
   - `insurance.medicare`, `insurance.medi_cal`, `insurance.supplement`
   - `flags.felony_history`, `flags.prior_lawsuits`, `flags.other_counsel`
   - `flags.unrelated_conditions_to_protect` (triggers 2.13 escalation)
   - `wage_loss.disclose_if_not_claiming` (Kobrin's "third path")

   These all came out of the Kobrin case review — see `LOGIC_REVIEW.md` for the full analysis.

4. **Intake form format:** The web form is a mockup. In practice, the "editable document" could be:
   - A markdown file Claude generates and the attorney edits in any text editor
   - A web form (if we build a lightweight UI)
   - A Word document (since Jeff already works in Word)
   - A JSON file with good comments (if the attorney is comfortable with that)

   My recommendation: start with markdown. It's readable, editable anywhere, and Claude can read it back trivially. If Jeff wants something fancier, the web form is ready to go.

### What doesn't change

- `assemble.py` (template-fill engine) — untouched, still reads `responses.json`
- `ingest.py` (PDF ingestion) — untouched
- `tag_responses.py` (completeness validation) — untouched
- Reference files (objections, patterns, interpretation guide) — untouched
- The responses.json schema — untouched
- Stage 3 processing logic — reads `case_context.json` same as before

The intake form is purely a change to how `case_context.json` gets populated. Everything downstream is the same.

---

## Net Assessment

We lose very little. The conversational Q&A's advantages (adaptive follow-ups, per-doc-type scoping) are preserved by keeping a short follow-up step after the form is read back. Everything else about the form approach is strictly better: more auditable, more portable, faster for the attorney, and it makes Claude's document extraction visible and correctable before it compounds into 20+ drafted responses.

The implementation lift is modest — it's mostly a change to SKILL.md's Stage 2 instructions and a schema update to `case_context_schema.md`. The scripts and engine don't change.

---

## Attachments

- `cases/gsdat-docs/CASE_INTAKE_SHEET.md` — the one-page markdown intake form
- `cases/gsdat-docs/case_intake_form.html` — interactive web form mockup (also live at https://karl-kahn.github.io/case-intake-form/)
- `cases/gsdat-docs/LOGIC_REVIEW.md` — the 23-item review that surfaced the new fields needed
