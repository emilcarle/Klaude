---
name: fluent-vocab
description: Run an interactive vocabulary drill session with flashcard-style prompts, spaced repetition, and per-answer feedback. Triggered only when the learner types /fluent-vocab. Reads spaced-repetition / mistakes / mastery DBs to pick words, presents one word at a time, scores each answer, and calls fluent-db-updater at the end.
allowed-tools: Read, Write, Bash, Agent
disable-model-invocation: true
---

# Vocabulary Drill Session

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

Flashcard-style vocabulary practice using spaced repetition. One word at a time, immediate feedback, DB update at the end. Interleaves three modes (recognition, production, cloze) to force active recall rather than passive re-reading.

## When to Use

Trigger this skill only when the learner types `/fluent-vocab`. The skill is gated with `disable-model-invocation: true` — a false-positive auto-trigger would launch a 15-min interactive session and mutate 6 JSON databases. Not worth the risk.

Skip this skill if no vocabulary items are due and no new words are queued — offer `/fluent-review` or `/fluent-learn` instead.

## Instructions

### 1. Load vocabulary data

```bash
python3 "${CLAUDE_PLUGIN_ROOT:-${CLAUDE_PROJECT_DIR:-.}}/.claude/hooks/read-db.py"
```

If the helper is unavailable, resolve `<data_dir>` via `fluent_paths.data_dir()` then read:

- `<data_dir>/spaced-repetition.json`
- `<data_dir>/mistakes-db.json`
- `<data_dir>/mastery-db.json`
- `<data_dir>/learner-profile.json` (for target_language, name, level)

If any are missing, direct the learner to `/fluent-setup` and stop.

### 2. Select words

Priority order:

1. Items in `spaced-repetition.review_queue.today` with `item_type == "vocabulary"`.
2. Words from `mistakes-db.json` where `category == "vocabulary"` and `mastery_level <= 2`.
3. New high-frequency words matching `learner-profile.focus_areas`.

Limit: `spaced-repetition.daily_limits.review_items_per_day` (default 20).

### 3. Present one word at a time

Rotate the three modes so the session is not monotonous.

**Recognition** (target_language → native):

```markdown
## Word {N}/{total}

**{target_language}:** {word}

**Context:** {example_sentence}

**What does it mean in {native_language}?**

**Type your answer:**
```

**Production** (native → target_language):

```markdown
## Word {N}/{total}

**{native_language}:** {word}

**Use it in a sentence (optional).**

**How do you say this in {target_language}?**

**Type your answer:**
```

**Cloze** (fill in the blank):

```markdown
## Word {N}/{total}

**Complete the sentence:**

{target_language sentence with _____ where the word goes}

**Type the missing word:**
```

### 4. Feedback after each answer

Use the compact Session Rules format. For a single word, that's usually one line:

- Correct: `✅ 10/10`
- Wrong: `❌ 3/10 → **das Fenster** 🟡 (die Tür = door)`, a hint of 8 words or fewer

Then the next word, in the same message.

Keep a running tally for the end-of-session save (no tool calls):

- Add to `review_results[]` with `quality = floor(score / 2)` (`update-db.py` does the SM-2 math, so there's nothing to compute now).
- If the learner met a new word, stage it for `new_vocabulary[]`.
- If the learner made an error, stage it for `errors[]`.

Do **not** call `update-db.py` after every word. Everything is saved once, in the background, at session end.

### 5. Session summary

```markdown
## 📚 Vocabulary Session Complete!

**Words Reviewed:** {N}
**Accuracy:** {X}%
**New Words Learned:** {Y}
**Words Mastered (→ level 5):** {Z}

**Strong:** {list words with mastery 4-5}
**Need more practice:** {list words with mastery 0-2}

**Next review:**
- Tomorrow: {count} words
- This week: {count} words

{target-language "well done"}! 🌟
```

### 6. Update all databases

Save in the background, following Session Rules → "Session end". The fork makes one `update-db.py` call with:

- `session_id`, `date`, `duration_minutes`
- `command_used: "/fluent-vocab"`
- `skills_practiced: ["vocabulary"]`
- `skill_scores.vocabulary`: `{exercises, correct, time_minutes}`
- `errors[]`, `new_vocabulary[]`, `review_results[]` collected during the session
- `focus_next_session[]` — top 2-3 weak words

Results file: `/results/fluent-vocab-session-{NNN}.md`.

## Examples

### Example 1 — recognition drill

> ## Word 1/15
>
> **Dutch:** de keuken
>
> **Context:** "De keuken is klein maar mooi."
>
> **Wat betekent dit in het Engels?**

Learner: "the kitchen"

> ✅ 10/10
>
> ## Woord 2/15
> …

### Example 2 — production with error

> ## Word 7/15
>
> **Engels:** to write
>
> **Hoe zeg je dit in het Nederlands?**

Learner: "schrieven"

> ❌ 6/10 → **schrijven** 🟢 *ij*, niet *ie*
>
> ## Woord 8/15
> …

### Example 3 — session end summary (rendered in the target language)

> ## 📚 Woordenschat klaar!
>
> **Woorden herhaald:** 15 · **Score:** 80% (12/15) · **Nieuw:** 3 · **Beheerst:** 1
>
> **Nog oefenen:** de sleutel, het raam, gisteren
>
> **Volgende herhaling:** morgen 4 woorden, deze week 8.
>
> Goed gedaan! 🌟

Then the background save, then `💾 Progress saved.` once the fork reports back.

## Critical Rules

- **One word at a time.** Wait for the learner's answer before showing the next.
- **Immediate, compact feedback** after each answer (Session Rules format), with no tool calls in between.
- **Mix modes.** Don't drill 20 recognition prompts in a row — interleave for discrimination.
- **Use the target language** for everything (see Session Rules). At A1-A2, add short native-language glosses.
- **Never** update the DBs mid-session. A background fork saves everything at the end.
- **Never auto-invoke.** This skill is gated; must fire only on explicit `/fluent-vocab`.

## Tips for the Learner (append if they seem tired or unsure)

- Review daily for best retention — spaced repetition depends on it.
- Focus time on weak words (mastery 0-2), not already-strong ones.
- Use words in sentences to build contextual memory.
- Say words out loud even though you're typing.
