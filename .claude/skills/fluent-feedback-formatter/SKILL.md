---
name: fluent-feedback-formatter
description: Reference for the compact per-answer feedback format in the Fluent system (score line, one line per error with a severity tag (🔴 critical / 🟡 moderate / 🟢 minor), and the correct version). Every practice skill (fluent-learn, fluent-vocab, fluent-writing, fluent-speaking, fluent-reading, fluent-review) already embeds this format in its Session Rules, so do NOT load this skill mid-session, since that costs a round trip while the learner waits. Open it to look up the full severity and category rules, or when editing the format.
---

# Feedback Formatter

## Overview

Feedback after every answer must be **as short as possible while still getting the correction across**. The learner should take it in at a glance and move on to the next exercise. Long explanations slow the session down and bury the one thing to fix.

Every practice skill embeds the compact template in its **Session Rules** section. This skill is the full reference behind it.

## When to Use

- Looking up the severity scale or category labels.
- Changing the feedback format (keep it in sync with the Session Rules blocks in the six practice skills and with `.claude/references/feedback-template.md`).

Don't load this skill during a practice session. The Session Rules are enough.

## Instructions

### 1. Template

Write it in the target language (practice sessions run entirely in the target language):

```markdown
❌ {X}/10 {optional comment, 6 words or fewer}
- ~~{wrong}~~ → **{correct}** {🔴|🟡|🟢} {reason, 8 words or fewer}
- ~~{wrong}~~ → **{correct}** {🔴|🟡|🟢} {reason}
➡️ {full correct version}

{next exercise}
```

- **Fully correct:** `✅ 10/10` (optionally plus a few words). Nothing else, and no correct-version echo.
- **Single-word answers** (vocab, multiple choice): `❌ 3/10 → **{correct}** 🟡 {hint}` on one line.
- **Long texts** (writing tasks): at most 8 error lines (🔴 first, then 🟡), all 🟢 slips merged into one line, then the corrected text. Put the score breakdown on the score line: `❌ 6/10 · Grammatik 6 · Wortschatz 8 · Struktur 5`.
- **Speaking:** at most 2 correction lines, plus an optional `💬 {more natural phrasing}`. Then continue the conversation.

### 2. What to leave out

- Headings ("Corrections:", "Correct version:"), `---` separators
- Lists of things the learner got right. At most one `✅` line, and only when the learner fixed a pattern they missed earlier.
- Multi-sentence explanations, bonus tips, C1 alternatives, running-accuracy notes
- Printing the category label (track it for the DB payload instead)

The learner can always ask *warum?* for a longer explanation. Give it then, in the target language.

### 3. Severity, on every error line

| Symbol | Severity | Meaning | Example |
|--------|----------|---------|---------|
| 🔴 | Critical | Breaks communication or an exam-blocker | du/Sie mix in a formal email; verb-final word order in subordinate clauses |
| 🟡 | Moderate | Noticeable but understandable | Preposition error, wrong plural, missing article |
| 🟢 | Minor | Low priority | Spelling, punctuation, capitalization |

After a 🔴 error, ask for a retype in one line: `🔁 Bitte abtippen.`

### 4. Categories (tracked, not printed)

These feed `errors[].category` in the end-of-session payload:

- `grammar`: word order, conjugation, clause structure
- `formal_informal`: du/Sie, u/je, register mismatch
- `vocabulary`: wrong word, English mixing, register-wrong synonym
- `spelling`: minor
- `prepositions`: auf/für/an/…, om/op/in/…
- `articles`: der/die/das, de/het, definite/indefinite
- `missing`: an omitted greeting, closing or required word

### 5. Score → SM-2

`quality = floor(score / 2)`. `update-db.py` does the SM-2 math at session end, so nothing needs computing mid-session.

## Examples

See `.claude/references/feedback-template.md`.

## Critical Rules

- **Short.** If a line can be cut without losing the correction, cut it.
- **A severity tag on every error line.** It drives spaced-repetition priority.
- **One score per answer,** out of 10.
- **Correct version only when there were errors.**
- **Next exercise in the same message.**
- **Encouraging, in few words.** A warm emoji or 2-3 words is enough. No paragraphs.
