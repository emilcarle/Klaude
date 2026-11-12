# Changelog

All notable changes to Fluent will be documented in this file.

## [Unreleased]

### Changed

- **Compact feedback.** Per-answer feedback is now a score line, one line per
  error (wrong → right, severity, rule in 8 words or fewer) and the correct
  version, with the next exercise in the same message. Headings, praise lists
  and long explanations are gone; the learner can ask *warum?* for details.
  The six practice skills embed the template, `fluent-feedback-formatter`
  and `references/feedback-template.md` were rewritten, and the writing,
  speaking, reading, vocab and review variants were shortened to match.
- **Session language.** Practice skills (`/fluent-learn`, `-vocab`,
  `-writing`, `-speaking`, `-reading`, `-review`) run entirely in the target
  language, from the greeting to the closing summary. Outside a practice
  skill (general chat, `/fluent-setup`, `/fluent-progress`) Claude speaks
  English. Results files and DB payloads stay in English.
- **Faster turns.** No tool calls between an answer and its feedback, and no
  helper-skill loads mid-session: results are tallied in context. At session
  end the summary is shown first, then a background `fork` agent runs
  `update-db.py` and writes the `/results` file, so the learner never waits
  on bookkeeping. `CLAUDE.md` no longer requires reading `LEARNING_SYSTEM.md`
  at every session start, and the helper-skill descriptions now say not to
  load them mid-session.

### Fixed

- `update-db.py`: the first session on the same day as `/fluent-setup` left
  the streak at 0. A session today now always means a streak of at least 1.
- `fluent-db-updater` docs claimed resending a `session_id` replaces the
  earlier call. It actually appends and double-counts; the docs now say so.

## [0.3.0] — 2026-06-15

### Added

- Milestones support in the `update-db.py` session payload. The new
  `milestones[]` field accepts either a bare string or an object
  `{ "milestone": <required non-empty string>, "date": <optional YYYY-MM-DD,
  defaults to the session date> }`. Each milestone is recorded in both
  `session-log.milestones[]` and `learner-profile.achievements[]`. Validation
  rejects malformed entries (exit `1`, no files written); an unparseable
  `date` falls back to the session date.

## [0.2.1] — 2026-06-11

### Fixed

- Hooks no longer fail on Windows with `No such file or directory` (#5).
  Plugin hook commands in `hooks.json` used the bash default-value syntax
  `${CLAUDE_PLUGIN_ROOT:-${CLAUDE_PROJECT_DIR:-.}}`, which Claude Code's own
  variable substitution does not understand on Windows — it replaced the
  variable names but left the `:-` separators literal, producing a single
  garbage path. Hook commands now use plain `${CLAUDE_PLUGIN_ROOT}` (always
  set for plugin hooks) and invoke scripts via an explicit `python3`/`bash`
  interpreter so they don't depend on shebang handling under Git Bash.

## [0.2.0] — 2026-05-14

### Breaking changes

All 12 skills renamed with a `fluent-` prefix to prevent collisions with other
plugins and Claude Code built-ins. Update any muscle memory or external
references.

| Old | New |
|-----|-----|
| `/setup` | `/fluent-setup` |
| `/learn` | `/fluent-learn` |
| `/review` | `/fluent-review` |
| `/vocab` | `/fluent-vocab` |
| `/writing` | `/fluent-writing` |
| `/speaking` | `/fluent-speaking` |
| `/reading` | `/fluent-reading` |
| `/progress` | `/fluent-progress` |
| `sm2-calculator` | `fluent-sm2-calculator` |
| `db-updater` | `fluent-db-updater` |
| `feedback-formatter` | `fluent-feedback-formatter` |
| `session-analyzer` | `fluent-session-analyzer` |

New session result files use `/results/fluent-{skill}-session-{NNN}.md`.
Existing files using the older `{skill}-session-{NNN}.md` naming are still
read by `fluent-session-analyzer` — no migration required.

### Fixed

- Plugin install no longer fails on first DB read. Skills now invoke helper
  scripts via `${CLAUDE_PLUGIN_ROOT:-${CLAUDE_PROJECT_DIR:-.}}/.claude/hooks/...`
  so the path resolves regardless of CWD.
- Added missing `.claude/hooks/ensure_data_dir.py` referenced by
  `fluent-setup`.

### Migration

```bash
claude plugin update fluent@m98
```

Then use the new slash commands. Your data (`~/.claude/fluent-data/` or
`./data/`) is unchanged.

## [0.1.0] — 2026-03-15

Initial release.
