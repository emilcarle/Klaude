---
name: fluent-writing
description: Run an interactive writing practice session (emails, letters, forms, short texts) with systematic error analysis and compact, severity-tagged feedback. Triggered only when the learner types /fluent-writing. Selects a scenario matched to mastery, lets the learner compose, then analyzes grammar, register, vocabulary, structure, and spelling before updating all databases.
allowed-tools: Read, Write, Bash, Agent
disable-model-invocation: true
---

# Writing Practice Session

## Session Rules (apply to every turn of this skill)

**Language: target language only.** From the greeting to the closing summary, speak the learner's target language (`learner.target_language`). That covers instructions, exercise prompts, feedback, explanations, transitions and the summary. The templates in this file are written in English so they're easy to read. Render them in the target language. Exceptions:
- Source sentences in translation exercises (native → target) stay in the native language.
- If the learner asks for an explanation in English, give it briefly, then continue in the target language.
- At A1–A2, keep the language simple and add a short native-language gloss in parentheses wherever an instruction might not be understood.

Once the closing summary is shown, the session is over: switch back to English.

**Feedback: as short as possible.** Every feedback message must be as brief as it can be while still getting the correction across:

```markdown
❌ 4/10
- ~~Wir muss~~ → **Wir müssen** 🔴 *wir* → Pluralform
- ~~zu nächsten Woche~~ → **auf nächste Woche** 🟡 verschieben auf + Akk.
➡️ Wir müssen die Besprechung auf nächste Woche verschieben.
```

- **First line:** ✅ or ❌, the score out of 10, and at most 6 words of comment.
- **One line per error:** wrong → right, a severity tag (🔴 critical / 🟡 moderate / 🟢 minor), and the reason in 8 words or fewer. Track the category (grammar, prepositions, …) for yourself; don't print it.
- **➡️ Correct version:** add it only when there were errors. If the answer is fully correct, the first line is the whole feedback.
- **Leave out** praise lists, headings, `---` separators, bonus tips and running-accuracy notes. One short ✅ note is allowed when the learner gets right a pattern they missed earlier.
- **After a 🔴 error,** ask for a retype in one line (e.g. `🔁 Bitte abtippen.`).
- **Longer explanations** only when the learner asks for one (*warum?*).
- **Same message:** put the next exercise right after the feedback.

**Speed: no bookkeeping mid-session.** Make no tool calls between the learner's answer and your feedback: no skill loads, no file reads or writes, no DB updates. The rules above are everything you need for feedback. Keep a running tally in your head of scores, errors (with category and severity), review results and new vocabulary. The conversation itself is the record. Everything is saved once, at the end, in the background (see "Session end" below).

**Session end: summary first, then save in the background.**

1. Show the closing summary right away, in the target language. Work out the streak yourself rather than waiting for the DB:
   - last session yesterday → +1
   - already practiced today → unchanged (minimum 1)
   - otherwise → 1
2. In the same response, start **one** background agent with the Agent tool, `subagent_type: "fork"`. A fork inherits this conversation, so it already has every answer. Tell it to:
   - build the `update-db.py` payload as described in the `fluent-db-updater` skill and run it
   - on a validation error, fix the payload and retry
   - write the results file named in this skill, in English
   - reply with one line, and never address the learner

   Then end your turn without waiting for it.
3. When the agent's completion notification arrives, reply with one short line in English (e.g. `💾 Progress saved.`), or tell the learner what failed.
4. If the Agent tool isn't available, do the same steps yourself, after the summary has been shown.
5. If the learner starts a new session before that notification arrives, wait for it before running `read-db.py`.

## Overview

Full-text writing practice with systematic correction. One scenario per session, compact feedback tagged by severity, background DB update at end. Mastery-driven scenario selection keeps the task at the right level — challenging, not frustrating.

## When to Use

Trigger this skill only when the learner types `/fluent-writing`. The skill is gated with `disable-model-invocation: true` — a 15-20 min interactive session with DB writes should never start from an ambiguous prompt.

Skip this skill in favor of `/fluent-vocab` if the learner has not yet hit mastery 2 in basic vocabulary — writing needs a minimum word bank.

## Instructions

### 1. Load context

```bash
python3 "${CLAUDE_PLUGIN_ROOT:-${CLAUDE_PROJECT_DIR:-.}}/.claude/hooks/read-db.py"
```

Need: `learner-profile` (level, target language, focus areas), `mistakes-db` (weak writing patterns), `mastery-db` (writing sub-skills).

### 2. Pick scenario type

From `mastery-db.skills_mastery`:

- Formal email (if `writing_formal_email` mastery < 4)
- Informal email (if `writing_informal_email` < 4)
- Form filling (if `writing_forms` < 4)
- Newsletter / personal text (if overall writing < 3)
- Mixed scenarios (if all ≥ 4)

Scenarios must match the learner's CEFR level — A2 uses everyday situations, B1+ adds opinion / complaint / inquiry.

### 3. Present the task

```markdown
## ✍️ Writing Exercise

**Scenario:** {clear description, in the target language}

**Task:** Write a {type} in {target_language}.

**Requirements:**
- Length: {X-Y} words
- Include: {must-include elements}
- Register: {formal / informal}
- Level: {CEFR}

{Optional: example structure for harder tasks}

**Write your {text_type} below:**
```

### 4. Wait for the full text

Don't correct mid-composition. Let the learner finish.

### 5. Systematic error analysis

Check every sentence for these categories:

1. **Grammar** — word order, conjugation, clause structure, articles
2. **Formal/informal** — register consistency
3. **Vocabulary** — wrong word, English mixing, register-wrong synonyms
4. **Missing elements** — greeting, closing, required fields
5. **Spelling** — minor at A2, weightier at B2+
6. **Structure** — organization, flow, paragraphing

Tag each finding with a severity: 🔴 critical, 🟡 moderate, 🟢 minor.

### 6. Feedback (compact)

This is the compact Session Rules format, stretched for a multi-sentence text. Write it in the target language:

````markdown
❌ {X}/10 · Grammar {Y} · Vocabulary {Z} · Structure {W} · Communication {V}
- ~~{wrong}~~ → **{correct}** 🔴 {reason, 8 words or fewer}
- ~~{wrong}~~ → **{correct}** 🟡 {reason}
- 🟢 {minor fix} · {minor fix} · {minor fix}
✅ {one line, only if a previously missed pattern is now right}

➡️
```
{fully corrected text}
```
````

- **At most 8 error lines.** 🔴 first, then 🟡. Merge all the 🟢 slips into one line.
- **No** "What you did well" lists, section headings or per-error paragraphs. A longer explanation only when the learner asks.

### 7. Optional rewrite

If the score is below 7, offer a rewrite in one line, in the target language (e.g. `🔁 Nochmal mit den Korrekturen? (nochmal / weiter)`).

### 8. Session summary

```markdown
## 📊 Writing Session Summary

**Text Type:** {type}
**Score:** {X}/10
**Key Takeaways:**
- {learning 1}
- {learning 2}
- {learning 3}

**Next Time:**
- Focus on: {weak pattern}
- Review: {relevant flashcards}

{target-language "well done"}! ✍️
```

### 9. Update all databases

Save in the background, following Session Rules → "Session end". The fork's payload:

- `command_used: "/fluent-writing"`, `skills_practiced: ["writing"]`
- `skill_scores.writing: {exercises: 1, correct: 1_if_score_≥_7_else_0, time_minutes}`
- `errors[]` — one per distinct pattern found (dedupe; the script bumps frequency)
- `focus_next_session[]` — top 2 patterns to drill

Results file (written by the fork): `/results/fluent-writing-session-{NNN}.md`, with the full task, the learner's original text, the corrected version, and the error table. The `fluent-session-analyzer` skill depends on this format.

## Examples

### Example 1 — formal email task (Dutch A2), all in the target language

> ## ✍️ Schrijfopdracht
>
> **Situatie:** Je hebt donderdag een afspraak bij de dokter, maar je kunt niet komen.
>
> **Opdracht:** Schrijf een e-mail aan de huisartsenpraktijk.
>
> - Lengte: 40-60 woorden
> - Noem: begroeting, reden, vraag om een nieuwe afspraak, afsluiting
> - Register: formeel
>
> **Schrijf je e-mail hieronder:**

Learner: "Hallo, Ik schrijf je omdat ik kan niet komen op donderdag. Ik ben ziek. Kunnen we een andere dag afspreken? Bedankt, Mohammad"

> ❌ 6/10 · Grammatica 6 · Woordenschat 8 · Structuur 5 · Communicatie 7
> - ~~Hallo … je~~ → **Beste meneer/mevrouw … u** 🔴 formele mail = u
> - ~~omdat ik kan niet komen~~ → **omdat ik niet kan komen** 🔴 omdat: werkwoord achteraan
> - ~~Bedankt,~~ → **Met vriendelijke groet,** 🟡 formele afsluiting
>
> ➡️
> ```
> Beste meneer/mevrouw,
>
> Ik schrijf u omdat ik donderdag niet kan komen. Ik ben ziek.
> Kunnen we een andere dag afspreken?
>
> Met vriendelijke groet,
> Mohammad
> ```
> 🔁 Nog een keer met de correcties? (*opnieuw* / *verder*)

## Critical Rules

- **One scenario per session.** Don't chain multiple writing tasks — depth over breadth.
- **Wait for the full answer** before correcting.
- **Severity tagging is mandatory.** Fed into `mistakes-db` and drives spaced repetition priority.
- **Always save the session file** in `/results/` (via the background fork) for later analysis by `fluent-session-analyzer`.
- **Never auto-invoke.** This skill is gated; must fire only on explicit `/fluent-writing`.

## Language Reference

### Dutch A2 patterns

**Formal emails:** always "u" (not "je"); open `Beste meneer/mevrouw {NAME},`; closing `Met vriendelijke groet,` + name.

**Informal emails:** "je" not "u"; open `Hallo {NAME},`; closing `Groetjes,` or `Tot snel,`.

**Common mistakes:** mixing formal/informal in one text; word order in `omdat` clauses (verb last); time expressions (`om 10:00 uur`, `op dinsdag`).

Add similar sections for other target languages as the learner needs them.
