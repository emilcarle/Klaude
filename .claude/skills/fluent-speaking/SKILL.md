---
name: fluent-speaking
description: Run an interactive typed conversation session simulating spoken practice — free-flowing dialogue, role-plays, and opinion questions prioritizing communication over perfect grammar. Triggered only when the learner types /fluent-speaking. Asks questions one at a time in the target language, evaluates clarity and naturalness first and grammar second, and updates all databases at the end.
allowed-tools: Read, Write, Bash, Agent
disable-model-invocation: true
---

# Speaking Practice (Typed)

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

Conversational practice through typed dialogue. Unlike `/fluent-writing`, prioritize **communication and naturalness** — grammar errors that don't block meaning are downplayed. Goal: build the learner's confidence to produce target-language output without over-analyzing.

## When to Use

Trigger this skill only when the learner types `/fluent-speaking`. The skill is gated with `disable-model-invocation: true` — 15-20 min interactive session with DB writes should never start from an ambiguous prompt.

Skip this skill below A1 mastery 2 — the learner needs a basic word bank and verb conjugations first (run `/fluent-vocab` a few times).

## Instructions

### 1. Load context

```bash
python3 "${CLAUDE_PLUGIN_ROOT:-${CLAUDE_PROJECT_DIR:-.}}/.claude/hooks/read-db.py"
```

Need: `learner-profile` (level, target language), `mastery-db.skills_mastery.speaking`.

### 2. Opening

```markdown
# 🗣️ {target_language} Speaking Practice

Hallo {name}!

Today we're practicing **speaking** through typed conversation. I'll ask you questions or give scenarios, you respond naturally in {target_language} — just like a real conversation.

**Focus:** natural expression, fluency, pronunciation (typed)
**Level:** {CEFR}
**Duration:** 15-20 min

**Tips:**
- Think in {target_language}, not {native_language}
- Don't chase perfect grammar — focus on getting your message across
- Use complete sentences
- Be natural and conversational

**Ready? Let's chat!** 💬
```

### 3. Pick topic based on mastery

A2 topics:
1. Personal introductions
2. Daily routine
3. Hobbies and interests
4. Shopping
5. Making appointments
6. Asking for directions
7. Ordering food
8. Talking about weather
9. Weekend plans
10. Work / study

B1+: opinions, comparisons, hypotheticals, complaints, narratives.

### 4. One question at a time

```markdown
## Question {N}: {Topic}

{Question in target language}

**Type your answer in {target_language}:**
```

Build the conversation naturally. After 3-4 questions on one topic, move on with a one-line transition in the target language (e.g. `Interessant! Lass uns über etwas anderes sprechen …`).

### 5. Evaluate

Check in this order:

1. **Communication** (most important, 0-5 points): was the message clear? Did it answer the question?
2. **Grammar** (0-3 points): verb conjugation, word order, articles. Note but don't belabor.
3. **Vocabulary** (0-2 points): appropriate word choice, no English mixing.

Use the compact Session Rules format, adjusted for conversation. Write it in the target language:

```markdown
✅ {X}/10 · Communication {Y}/5 · Grammar {Z}/3 · Vocabulary {W}/2
- ~~{wrong}~~ → **{correct}** 🟡 {reason, 8 words or fewer}
💬 {more natural phrasing, one line, optional}

{your next conversational turn / question}
```

- **At most 2 correction lines,** and only for errors that matter. Don't echo the learner's answer back.
- **Keep the conversation going** in the same message. The feedback is a short aside, not the main event.

### 6. Role-play (advanced)

For B1+ or when the learner is warmed up:

```markdown
## 🎭 Role-Play

**Scenario:** {description, in the target language}
**Your role:** {what the learner plays}
**I'll be:** {what Claude plays}

Ready? I'll start...

---

{first line in target language}

**Your turn:**
```

### 7. Session summary

```markdown
## 🎉 Speaking Session Complete!

**Duration:** {X} min
**Questions Answered:** {N}
**Topics Covered:** {list}

### Communication Scores
**Overall:** {percent}%
- Clear messages: {count}
- Natural expression: {rating}/5
- Confidence: Growing! 💪

### Vocabulary Used Well
- {words}

### For Next Time
- Try using: {new phrase}
- Practice: {weak area}

**{target-language well done}!** 🌟
```

### 8. Update all databases

Save in the background, following Session Rules → "Session end". The fork's payload:

- `command_used: "/fluent-speaking"`, `skills_practiced: ["speaking"]`
- `skill_scores.speaking: {exercises: N, correct: count_of_clear_answers, time_minutes}`
- `errors[]` — only communication-blocking ones (don't flood mistakes-db with minor speaking slips)
- `focus_next_session[]` — one topic + one pattern

Results file: `/results/fluent-speaking-session-{NNN}.md`.

## Examples

### Example 1 — personal intro (Dutch A2)

> ## Question 1: Introductions
>
> Vertel me over jezelf. Waar kom je vandaan?
> (Tell me about yourself. Where are you from?)

Learner: "Ik ben Mohammad. Ik kom van Iran maar ik woon nu in Nederland."

> ✅ 9/10 · Communicatie 5/5 · Grammatica 2/3 · Woordenschat 2/2
> - ~~van Iran~~ → **uit Iran** 🟢 herkomst = uit
> 💬 Ik kom uit Iran, maar ik woon al een paar jaar in Nederland.
>
> Leuk! Hoe lang woon je al in Nederland? (How long have you lived in the Netherlands?)

### Example 2 — role-play opener (café scenario)

> ## 🎭 Role-Play
>
> **Situatie:** Je zit in een café in Amsterdam. Je wilt koffie en iets vegetarisch voor de lunch.
> **Jouw rol:** klant
> **Ik ben:** de ober
>
> ---
>
> *"Goedemiddag! Wat wilt u drinken?"*
>
> **Your turn:**

## Critical Rules

- **Communication first.** A clear message with a missed article scores better than a grammatically perfect but confusing answer.
- **One question at a time.** Wait for reply before next.
- **Stay in the target language** for everything, explanations included (see Session Rules). Use English only if the learner asks for it.
- **Praise natural expression.** If the learner uses "Nou..." or "Eh..." correctly, call it out — those are fluency markers.
- **Don't over-correct.** A speaking session with 20 red marks kills confidence.
- **Never auto-invoke.** Gated; must fire only on explicit `/fluent-speaking`.

## Language Reference

### Dutch A2 conversational fillers

- "Nou..." (well / so)
- "Eh..." (uh / um)
- "Eigenlijk..." (actually)
- "Dus..." (so / therefore)
- "Ja, dat klopt" (yes, that's right)
- "Ik snap het niet" (I don't understand)

Add equivalents for other target languages as needed.
