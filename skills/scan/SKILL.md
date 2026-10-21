---
name: scan
description: Rank your skills by how much of each is likely obsolete for a target Claude model. Read-only quick heuristic pass; recommends a triage order. Accepts a path to scan just one skill or plugin.
argument-hint: "[path to a SKILL.md, a skill folder, or a plugin folder]"
disable-model-invocation: true
---

# scan

Arguments: `$ARGUMENTS`

Read `${CLAUDE_PLUGIN_ROOT}/shared/conventions.md` first (files, hard rules, skill discovery, token estimates). Then load the `skill-ablation:triage-rubric` skill and apply its scan-mode heuristics.

Read-only: no edits to any skill. It may create `routing.yaml` on the first run (not when a path is given).

0. **Scope**: no argument means discover every skill (see the conventions). With a path, scan only the skill(s) there: a `SKILL.md`, a skill folder, or a plugin folder (every `skills/*/SKILL.md` in it). Label path targets as `repo` skills.
1. **Size**: `wc -c` of the original (`variants/full.md` if `SKILL.md` is a symlink, otherwise `SKILL.md`), tokens est. per the conventions. For an activated skill, also show the active variant's size and mark it "slim active". Add the `references/` total separately; mention it only if large (it loads on demand).
2. Measure token buckets per skill (code fences, table rows, "See references/" pointers, version-specific notes, other prose), read each SKILL.md, and estimate % cuttable and the top 2 cut reasons. Use the cached release notes for the target model if any; otherwise say "no release notes cached; heuristics only". Do not fetch release notes or do per-line verdicts.
3. Print one table: skill (with its source: personal / project / synced / plugin name / repo; plugin, synced and repo rows are report-only), target model, size (est. tokens), est. % cuttable, top 2 cut reasons, sorted by est. tokens saved descending. Number the rows: that order is the recommended triage order (`/skill-ablation:triage <skill>`). Add a line suggesting `/skill-doctor` to find never-used skills worth turning off outright.
