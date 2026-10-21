---
name: triage
description: Full block-by-block triage of one skill against a target Claude model; writes a report, a slim variant and a diff, then asks before activating.
argument-hint: "<skill-name> [--model opus|sonnet|haiku] [--refresh]"
disable-model-invocation: true
---

# triage

Arguments: `$ARGUMENTS`

Read `${CLAUDE_PLUGIN_ROOT}/shared/conventions.md` first (files, hard rules, skill discovery, token estimates). Then load the `skill-ablation:triage-rubric` skill and apply it exactly. No skill name given: ask which one.

1. **Model**: `--model` flag, else `target` in routing.yaml. If neither exists, ask the user and save it to routing.yaml.
2. **Release notes**: resolve the alias to the current version by looking it up (WebSearch, e.g. "latest Claude Opus model"). If `models/<version>.md` exists and no `--refresh`, reuse it. Otherwise:
   - Search Anthropic sources first: anthropic.com/news, docs.claude.com (model pages, release notes), the model card / system card, and the model's prompting guide. Use third-party sources only to fill gaps and label each as `[third-party]`.
   - Write 5–10 numbered bullets: claimed capability improvements, instruction-following changes, training cutoff date, known behavior changes (e.g. follows instructions more literally). Include source URLs and fetch date. Save to `models/<version>.md` (create dirs as needed).
   - **Skill-authoring rules**: also load `~/.claude/skill-ablation/models/skill-authoring.md`. If it exists and no `--refresh`, reuse it. Otherwise fetch Anthropic's Skill authoring best practices (platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) and the Claude Code skills page (code.claude.com/docs/en/skills), and write the checkable rules as numbered bullets with their thresholds (SKILL.md line limit, reference-file length that needs a contents list, reference depth, description rules and length caps, time-sensitive text, path style), source URLs and fetch date. These are structure rules, not obsolescence rules; use only official sources.
3. **Classify**: read the whole skill. The original is `variants/full.md` when `SKILL.md` is a symlink (the skill is already activated), otherwise `SKILL.md`; never classify a slim variant as if it were the original. Read SKILL.md only ( skim references for size and duplication, but verdict SKILL.md lines). Apply the rubric per block with the model-weighting rule. Each verdict: quoted first words, label, one-line reason, release-note bullet number when relevant.
4. **Report** to `reports/<skill>-<version>.md` (if one exists, first rename it to `<skill>-<version>.prev-<date>.md` so history is kept): header (skill, model, version, date, notes file), per-block verdict table, token savings estimate (cut + softened), the UNSURE list, and a `## Seed cases` section: for each UNSURE item, one realistic prompt to run with the skill off (`skillOverrides: {"<skill>": "off"}`) to see if the model already does it. Keep that heading exact; the future `run` command reads it. Also add a `## Structure flags` section: flags only, never cuts. Check with plain shell against the thresholds in `skill-authoring.md`: `wc -l` of SKILL.md; line count of each `references/*.md` and whether a contents list is in its first ~15 lines; whether any reference file links to another reference file (depth); description length and whether it is third person; time-sensitive phrases (`grep -n -i -E '(before|after|until) (19|20)[0-9]{2}|as of '`); backslash paths. Write "none" when clean.
5. **Slim variant**: write the proposed variant: original with CUT lines removed, SOFTEN lines rewritten in a plain tone, KEEP and UNSURE untouched, frontmatter preserved. Write it to `<skill-dir>/variants/slim-<model>.md`, or to `slim-<model>.new.md` when `slim-<model>.md` already exists (it may be the active one; never overwrite it before approval). Create `variants/` if missing. Do not touch `SKILL.md`. **Verify before showing it**: frontmatter is intact; code fences are balanced; every kept line also exists in the original (`grep -v -x -F -f original variant`), apart from rewrites you list; relative links in the variant still resolve. Report any failure instead of activating.
6. **Diff**: show `diff -u <original> <slim>` in the session (and against the currently active variant if there is one).
7. **If every section is CUT**: say the skill looks obsolete and offer to set `obsolete: true` in routing.yaml (user approves).
8. **Activate?** Ask whether to activate the slim version for that model. On yes:
   - First activation only: `mkdir -p variants && mv SKILL.md variants/full.md && ln -s variants/full.md SKILL.md` (this relocates the original bytes, never alters them). Verify the checksum of `variants/full.md` matches the original.
   - Then point the link at the variant and record it: if the proposal is `slim-<model>.new.md`, first keep the old one as `slim-<model>.prev.md` and rename the new file to `slim-<model>.md`; then `ln -sfn variants/slim-<model>.md SKILL.md`; set `active.<model>: slim` in routing.yaml.
   - The link is global, not per session, so a skill has one active variant at a time. Say so in one line: with a single target model per skill this is what you want; per-model switching needs a SessionStart hook, which isn't included.
   - Revert: `ln -sfn variants/full.md SKILL.md` and set `active.<model>: full`.
   - A new session is needed to pick up the change.
