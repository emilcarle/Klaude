# Feedback Template

The compact per-answer feedback format used by every Fluent practice skill. The `fluent-feedback-formatter` skill and the Session Rules block in each practice skill both point here. Keep all three in sync.

**Principle:** as short as possible while still getting the correction across. Write it in the target language.

## Template

```markdown
❌ {X}/10 {optional comment, 6 words or fewer}
- ~~{wrong}~~ → **{correct}** {🔴|🟡|🟢} {reason, 8 words or fewer}
➡️ {full correct version}

{next exercise}
```

- Fully correct → `✅ {X}/10` and nothing else.
- Single-word answers → `❌ 3/10 → **{correct}** 🟡 {hint}`.
- Long texts → at most 8 error lines, 🟢 slips merged into one line, then the corrected text in a code block. Breakdown goes on the score line.
- After a 🔴 error → `🔁 Bitte abtippen.` (in the target language).
- No headings, separators, praise lists or paragraphs. A longer explanation only when asked.

## Severity markers

| Symbol | Severity | Meaning |
|--------|----------|---------|
| 🔴 | Critical | Breaks communication or an exam-blocker |
| 🟡 | Moderate | Noticeable but understandable |
| 🟢 | Minor | Spelling, punctuation, capitalization |

## Category labels (tracked for the DB payload, not printed)

`grammar`, `formal_informal`, `vocabulary`, `spelling`, `prepositions`, `articles`, `missing`

## Examples (German B2)

### Fully correct

> ✅ 10/10 — Verb am Ende, perfekt.
>
> **Übung 4:** …

### Several errors

Learner: "Wir muss die Treffung zu nächsten Woche verschieben."

> ❌ 4/10
> - ~~Wir muss~~ → **Wir müssen** 🔴 *wir* → Pluralform
> - ~~Treffung~~ → **Besprechung** 🟡 kein deutsches Wort
> - ~~zu nächsten Woche~~ → **auf nächste Woche** 🟡 verschieben auf + Akk.
> ➡️ Wir müssen die Besprechung auf nächste Woche verschieben.
>
> 🔁 Bitte abtippen.

### Progress callout

> ✅ 9/10 — *sich freuen darauf, …* sitzt jetzt!
> - ~~Grüssen,~~ → **Grüßen** 🟢 ß in DE, kein Komma

### Previous (verbose) style: don't do this

> ❌ Good start: you picked the right verb and put it in the right place. Three small things to fix.
>
> **Corrections:**
> - ❌ "Wir **muss**" → **"Wir müssen"** 🔴 (grammar: with *wir*, modal verbs take the plural form: ich muss, du musst, er muss, **wir müssen**, ihr müsst, sie müssen)
> - …4 more lines of this length…
>
> **Correct version:** …
>
> **Score: 4/10** 🌱 The sentence structure is solid. The mistakes are in conjugation, word choice and preposition, which is very fixable.

## Score → SM-2 Quality

`quality = floor(score / 2)`. See `sm2-worked-examples.md`.
