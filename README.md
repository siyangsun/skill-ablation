# skill-ablation

A Claude Code plugin that audits your skills for instructions newer Claude models no longer need, and proposes slimmed variants. **Version 0.2: heuristic triage only.** It reads your skills and the model's release notes and judges what looks obsolete. It does not yet run the skill with and without the cut text to check; that is the planned `run` command.

Treat every proposed cut as a hypothesis. Read each diff before you activate anything.

## What it does

- `/skill-ablation:scan [path]`: ranks all your skills by likely savings, or just the skill or plugin at a path. Read-only.
- `/skill-ablation:triage <skill | plugin:skill | path> [--model opus|sonnet|haiku] [--refresh]`: classifies each block of one skill (CUT / SOFTEN / KEEP / UNSURE), writes a report, writes a slim variant, shows the diff, and asks before activating it.
- `/skill-ablation:prune [--dry-run]`: deletes skills you have marked `obsolete: true`, after explicit confirmation.

Each command is its own skill, namespaced by the plugin: `/skill-ablation:scan`, `/skill-ablation:triage <skill>`, `/skill-ablation:prune`.

## scan vs triage

`scan` is a quick ranking; `triage` is the full analysis of one skill.

| | `scan` | `triage` |
|---|---|---|
| Scope | All your skills, or one path | One skill |
| Depth | Measures what each skill is made of (code, tables, pointers, prose) and estimates % cuttable | Judges every block: CUT, SOFTEN, KEEP or UNSURE, with a reason for each |
| Model research | Uses cached release notes if present, never fetches | Looks up the current model, fetches or reuses its release notes, loads the skill-authoring rules |
| Output | One ranked table in the chat | A report file, a slim variant file, a diff, and structure flags |
| Writes files | No (creates `routing.yaml` only on a first full run) | Yes: report, variant, optional `routing.yaml` update |
| Changes your skill | Never | Only if you approve activation, and only for personal skills |
| Cost | Cheap: reads each skill once | Heavier: reads one skill closely, may do web lookups |
| Reliability | Judgment from content mix; good for ordering only | Per-block verdicts, with seed test prompts for the doubtful ones |

**Typical workflow:** run `scan` first to decide where to spend effort; the top rows are the best candidates. Run `triage <skill>` on those one at a time and read each diff before activating. Use `prune` only for skills that triage marks as fully obsolete. Neither command measures whether a slimmed skill still works; that is the planned `run` command.

## Install

```bash
claude plugin marketplace add siyangsun/skill-ablation   # or a local path to a clone
claude plugin install skill-ablation@skill-ablation
```

Restart Claude Code. After editing the plugin, bump `version` in `.claude-plugin/plugin.json` (the version pins the installed copy), then run `claude plugin marketplace update skill-ablation` and `claude plugin update skill-ablation@skill-ablation`.

## Files it creates

| Path | Purpose |
|---|---|
| `~/.claude/skill-ablation/routing.yaml` | Source of truth: target model per skill, which variant (full or slim) is active |
| `~/.claude/skill-ablation/models/<model-version>.md` | Release-notes summary, cached with source URLs and fetch date |
| `~/.claude/skill-ablation/models/skill-authoring.md` | Anthropic's skill-authoring rules (thresholds for structure flags), cached from the official docs; refreshed with `--refresh` |
| `~/.claude/skill-ablation/reports/<skill>-<model-version>.md` | Per-block verdicts, savings estimate, UNSURE list, seed test prompts |
| `<skill-dir>/variants/full.md` | Your original `SKILL.md`, moved here byte-for-byte on first activation |
| `<skill-dir>/variants/slim-<model>.md` | The proposed slim version |

## How a variant is activated

Settings `skillOverrides` can only show or hide a skill (`on`, `name-only`, `user-invocable-only`, `off`); it cannot choose a file per model. So activation is a symlink: the first time, `SKILL.md` is moved to `variants/full.md` and `SKILL.md` becomes a link to the active variant. Reverting is one command:

```bash
cd ~/.claude/skills/<skill> && ln -sfn variants/full.md SKILL.md
```

then set `active.<model>: full` for that skill in `routing.yaml`.

Trade-offs: the link is global, not per session, so a skill has one active variant at a time. That fits a single target model per skill. A SessionStart hook could swap by model, but it only fires at session start and is not included.

## How it judges

The `triage-rubric` skill holds the rules. In short:

- **CUT**: generic knowledge, workarounds for old-model failures the release notes say are fixed, stock examples, and reference material the model already knows and can look up (API tables, install steps for platforms you don't use, pointer paragraphs that re-summarize a file).
- **SOFTEN**: right content, heavy emphasis.
- **KEEP**: your conventions, paths and process, knowledge newer than the model's cutoff, security and irreversible-action guardrails, compact gotchas.
- **UNSURE**: can't tell from the text. Each gets a test prompt to run with the skill turned off.

Weighting: aggressive for opus, moderate for sonnet, conservative for haiku.

Sources are official only: Anthropic's model pages and release notes, the model's prompting guide, and the skill-authoring best practices. Blog posts are not used. Each report also has a `Structure flags` section (SKILL.md over 500 lines, long reference files without a contents list, nested references, description rules, time-sensitive text). Flags never cut anything; they are checked against the cached authoring rules, so they follow the docs when you `--refresh`.

## Honest limits

- **Not verified empirically.** Savings are real byte counts but whether output quality holds is untested. Use the seed prompts in each report, with `skillOverrides: {"<skill>": "off"}`, before trusting a cut.
- **Token figures are estimates.** Size uses about 2.5 characters per token (Anthropic's figure for the current tokenizer), ±20%. Reports written before 0.1.2 used 4 characters per token and understate absolute tokens by about 1.6x; percentages are unaffected.
- **"% cuttable" in `scan` is judgment**, from content mix. Use it to order work, not to predict savings.
- **Tested:** `scan`, and `triage` with opus on personal skills (including activation and revert). **Not tested:** `prune`, `--refresh`, `--model sonnet|haiku`, project-level `.claude/skills/`.
- **Plugin-managed skills** (ponytail, good-css and so on, under `~/.claude/plugins/cache/`) are report-only: they are scanned and triaged, and variants are written to `~/.claude/skill-ablation/plugin-variants/`, but never activated or pruned. Updates replace the cache folder, and `skillOverrides` does not apply to plugin skills. Send the slimmed version to the plugin's repo instead.
- **Synced claude.ai skills** (`~/.claude/skills/synced/`) can be scanned and triaged but not activated or pruned: the docs say a later sync can overwrite or remove local edits. Copy the skill into `~/.claude/skills/<name>/` if you want a slim version.
- **Plugin updates can overwrite your symlinks.** If a plugin that installs into `~/.claude/skills/` updates, re-check which variant is active.
- **`references/` folders are not triaged.** Only `SKILL.md` is.
- **Repo skills** (a skill or plugin you are developing, passed as a path) are report-only too. Apply cuts by editing the repo on a branch, test with `claude --plugin-dir <folder>`, bump the version. A symlink there would be committed.
- Skills that were written for another harness may have parameters that don't exist in Claude Code (for example subagent types or model names). Triage notes it but does not rewrite it.

## Safety rules

- Original skill files are never edited or deleted, except by `prune` on a skill you confirm.
- Every write to a skill directory needs your approval in the session.
- `prune` always asks, even in auto mode.

## Roadmap

`/skill-ablation:run`: run each seed prompt with and without the skill (and with the slim variant) and compare. The report format already has a `## Seed cases` section for it, so nothing needs renaming.

## Layout

```
.claude-plugin/plugin.json, marketplace.json
skills/scan, skills/triage, skills/prune   the commands (one SKILL.md each)
shared/conventions.md            files, hard rules, discovery, token estimates (read by all three)
skills/triage-rubric/SKILL.md    the rubric (not user-invocable)
```
