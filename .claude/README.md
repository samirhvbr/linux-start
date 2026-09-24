# Claude Code configuration — LINUX/START

Stack: **Shell**.

## Files
- `settings.json` — the active profile. It does not choose the model.
- `settings.local.json` — local override (gitignored), takes precedence over `settings.json`.

## Model
- **Nothing here chooses the model** (repodocs ADR-027): `settings.json` carries no
  `model`, `fallbackModel` or `availableModels`, and nothing in `env` that steers one
  (`ANTHROPIC_MODEL`, `ANTHROPIC_DEFAULT_*_MODEL`, `CLAUDE_CODE_SUBAGENT_MODEL`).
- The model is the user's choice, made with `/model`, per session. A subagent inherits
  the session's model.
- There are no stand-by profiles to copy over `settings.json` — `/model` does that job.
- Effort `max` via the `CLAUDE_CODE_EFFORT_LEVEL` env var (the `effortLevel` field only
  accepts low/medium/high/xhigh).

## Permissions
- `defaultMode: plan`; security denies (rm -rf, force push, reset --hard, clean -fd, curl|sh).
- **git push allowed** (in `allow`).
