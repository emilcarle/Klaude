---
name: fluent-reading
description: Run an interactive reading comprehension session with a short target-language text followed by main-idea, detail, vocabulary-in-context, inference, and true/false questions. Triggered only when the learner types /fluent-reading. Presents the text, waits for the learner to read, then asks questions one at a time with immediate feedback, and optionally adds new vocabulary to the spaced-repetition queue.
allowed-tools: Read, Write, Bash, Agent
disable-model-invocation: true
---

# Reading Comprehension Session

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

Present one text (100-500 words depending on level), ask 4-6 comprehension questions, extract vocabulary. Builds passive-to-active bridge: learners decode target-language writing, then answer questions that force recall.

## When to Use

Trigger this skill only when the learner types `/fluent-reading`. The skill is gated with `disable-model-invocation: true` — 15-20 min interactive session with DB writes should never start from an ambiguous prompt.

Skip this skill below A1 mastery 3 — shorter flashcard drills (`/fluent-vocab`) are more appropriate for very early learners.

## Instructions

### 1. Load context

```bash
python3 "${CLAUDE_PLUGIN_ROOT:-${CLAUDE_PROJECT_DIR:-.}}/.claude/hooks/read-db.py"
```

Need: `learner-profile` (level, target language, interests), `mastery-db.skills_mastery.reading`.

### 2. Opening

```markdown
# 👀 {target_language} Reading Practice

Hallo {name}!

Today we're practicing **reading comprehension**. I'll show you a short {target_language} text, then ask you questions about it.

**Focus:** main ideas, details, vocabulary in context
**Level:** {CEFR}
**Duration:** 15-20 min

**Tips:**
- Read the whole text first
- Don't translate every word — get the gist
- Use context clues for unknown words
- Read the questions before rereading the text

**Ready? Let's read!** 📖
```

### 3. Pick text type + length

A2 types (100-200 words): personal email, short news, advertisement, instructions, simple story, blog post, social media post, info leaflet.

B1 (200-350 words): opinion pieces, longer narratives, structured guides.

B2+ (350-500): editorials, technical explanations, interviews.

Match the topic to `learner-profile.focus_areas` when possible.

### 4. Present the text

```markdown
## 📄 Reading Text {N}

**Topic:** {topic}
**Type:** {text_type}
**Length:** ~{word_count} words

---

{target-language text — clean formatting, no inline translation}

---

Take your time. When you're done, type **"ready"**.
```

### 5. Question sequence (one at a time)

Rotate across these types:

**Main idea:**
```markdown
## Vraag 1: Hoofdidee (main idea)

{question in target language}

a) {option 1}
b) {option 2}
c) {option 3}

**Type a, b, or c:**
```

**Details:**
```markdown
## Vraag 2: Details

{specific question about the text}

**Type your answer:**
```

**Vocabulary in context:**
```markdown
## Vraag 3: Vocabulaire

{"In the text it says '{word/phrase}'. What does it mean?", in the target language}

a) {meaning 1}
b) {meaning 2}
c) {meaning 3}
```

**Inference:**
```markdown
## Vraag 4: Begrijpen

{question requiring inference — not directly stated}

**Answer in {target language}:**
```

**True / false:**
```markdown
## Vraag 5: Waar of niet waar?

{statement}

**Type your answer:**
```

### 6. Feedback per question

Use the compact Session Rules format, in the target language:

```markdown
✅ {X}/10
```

or, when wrong:

```markdown
❌ {X}/10 → **{correct_answer}**
📄 "{short quote from the text that proves it}"
```

Then the next question, in the same message. Skip the long explanation; the quote does the work.

### 7. Vocabulary review

After the questions:

```markdown
## 📚 New Vocabulary from the Text

| {target_language} | {native_language} | Example from text |
|-------|---------|-------------------|
| {word 1} | {meaning} | "{sentence}" |
| {word 2} | {meaning} | "{sentence}" |

**Save these for future review?** (They'll enter spaced repetition.)

Type "yes" to add, "no" to skip.
```

If yes, stage each word for `new_vocabulary[]` in the end-of-session DB update.

### 8. Session summary

```markdown
## 📊 Reading Session Complete!

**Text:** {title/topic}
**Length:** {words} words
**Questions:** {N}
**Accuracy:** {percent}%

### Comprehension Breakdown
- Main idea: {✅ or ❌}
- Details: {score}
- Vocabulary: {score}
- Inference: {score}

### New Words Added: {count}
{list}

### For Next Time
- {suggestion based on which question type was weakest}

**{target-language well done}!** 📖✨
```

### 9. Update all databases

Save in the background, following Session Rules → "Session end". The fork's payload:

- `command_used: "/fluent-reading"`, `skills_practiced: ["reading"]`
- `skill_scores.reading: {exercises: N, correct: count_right, time_minutes}`
- `errors[]` — per question-type weakness (category `comprehension`, `vocabulary`, `inference`)
- `new_vocabulary[]` — words the learner chose to save
- `focus_next_session[]`

Results file: `/results/fluent-reading-session-{NNN}.md`. Include the full text plus the Q&A for later analysis.

## Examples

### Example 1 — personal email text (Dutch A2)

> ## 📄 Reading Text 1
>
> **Topic:** Making weekend plans
> **Type:** Personal email
> **Length:** ~75 words
>
> ---
>
> Beste Mohammad,
>
> Bedankt voor je email! Leuk dat je naar Amsterdam komt volgende maand. Ik heb tijd op zaterdag 15 maart. Zullen we om 14:00 uur afspreken bij het Centraal Station? We kunnen naar een café gaan en daarna door de stad wandelen.
>
> Het weer is meestal koud in maart, dus neem een warme jas mee! Ik verheug me erop om je te zien.
>
> Groetjes,
> Lisa
>
> ---
>
> Neem de tijd. Typ **"klaar"** als je klaar bent.

### Example 2 — main-idea question on the above

> ## Vraag 1: Hoofdidee
>
> Waar gaat deze email over?
>
> a) Lisa is op vakantie in Iran.
> b) Lisa en Mohammad maken plannen voor een ontmoeting.
> c) Lisa vraagt naar het weer.

Learner: "b"

> ✅ 10/10
>
> ## Vraag 2: Details
>
> Waar en hoe laat spreken ze af?

## Critical Rules

- **Wait for "ready"** (in the target language, e.g. "klaar" / "fertig") before asking the first question. Rushing the reading step defeats the purpose.
- **One question at a time.** Multiple at once invites skimming.
- **Ask questions in the target language** (at least from A2 up). Reading-comprehension checks should happen in the same language as the text.
- **Quote the text** in explanations so the learner can trace the answer back to the source.
- **Vocabulary opt-in.** Don't force-add every unknown word — ask the learner which they want to keep.
- **Never auto-invoke.** Gated; must fire only on explicit `/fluent-reading`.

## Sample Text Bank (Dutch A2)

### Advertisement
```
CURSUS NEDERLANDS VOOR BEGINNERS

Wil je Nederlands leren? Start deze maand nog!

- Kleine groepen (max 8 personen)
- Ervaren docenten
- 2x per week, 's avonds
- Locatie: Centrum Amsterdam
- Prijs: €150 per maand

Aanmelden kan via email: info@nederlandscursus.nl
Bel voor meer informatie: 020-123-4567

Eerste les gratis!
```

Keep a small bank per target language — don't reuse the same text in consecutive sessions.
