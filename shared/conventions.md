# skill-ablation conventions (shared by scan, triage, prune)

## Files

| Path | Purpose |
|---|---|
| `~/.claude/skill-ablation/routing.yaml` | Source of truth: target model per skill, active variant per model |
| `~/.claude/skill-ablation/models/<model-version>.md` | Cached release notes |
| `~/.claude/skill-ablation/models/skill-authoring.md` | Cached skill-authoring rules (structure thresholds) |
| `~/.claude/skill-ablation/reports/<skill>-<model-version>.md` | Triage reports |
| `<skill-dir>/variants/full.md`, `slim-<model>.md` | Variants. `SKILL.md` becomes a symlink to the active one |

`routing.yaml` format (Claude reads and edits it directly):

```yaml
skills:
  how:
    path: /Users/me/.claude/skills/how
    target: opus            # opus | sonnet | haiku
    active: {opus: full, sonnet: full, haiku: full}   # full | slim
    obsolete: false
```

## Hard rules
- Never edit, overwrite, or delete an original skill file, except `prune` on a skill the user confirms. All changes go through `variants/`.
- Every write to a skill directory (variants, symlink, prune) needs explicit user approval in this session.
- Synced skills (path contains `/skills/synced/`) are scanned and triaged, but activation and `prune` are refused for them: per the Claude Code docs, edits under `~/.claude/skills/synced/` are never uploaded and a later sync "can overwrite or remove" them, and a deleted synced skill is re-downloaded. Tell the user to change them on claude.ai, or copy the skill into `~/.claude/skills/<name>/` as a personal skill (then hide the synced original with `skillOverrides: "off"`).

## Skill discovery
1. Find `SKILL.md` files: `~/.claude/skills/*/SKILL.md`, `.claude/skills/*/SKILL.md` in the current project, and `~/.claude/skills/synced/*/*/SKILL.md`. Follow symlinks (an activated skill's `SKILL.md` is a link).
2. Before the **first scan** (no `routing.yaml` yet), list what you found (name, location, count) and ask the user to confirm.
3. If `routing.yaml` is missing, create it with `target: opus` and all-`full` for every skill found. Add entries for newly found skills later; never remove entries except via prune.

## Token estimates
Tokens ≈ chars / 2.5 (Anthropic's docs put the current tokenizer at about 2.5 characters per token; treat as ±20% and say "est.").
