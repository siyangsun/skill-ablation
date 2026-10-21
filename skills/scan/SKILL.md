---
name: scan
description: Rank all your skills by how much of each is likely obsolete for a target Claude model. Read-only quick heuristic pass; recommends a triage order.
disable-model-invocation: true
---

# scan

Read `${CLAUDE_PLUGIN_ROOT}/shared/conventions.md` first (files, hard rules, skill discovery, token estimates). Then load the `skill-ablation:triage-rubric` skill and apply its scan-mode heuristics.

Read-only: no edits to any skill. It may create `routing.yaml` on the first run.

1. Discover skills. Size = `wc -c` of the original (`variants/full.md` if `SKILL.md` is a symlink, otherwise `SKILL.md`; tokens est. per the conventions). For an activated skill also show the active variant's size and mark it "slim active" plus a separate `references/` total. Report SKILL.md tokens as "size"; mention references only if large (they load on demand).
2. Measure token buckets per skill (code fences, table rows, "See references/" pointers, version-specific notes, other prose), read each SKILL.md, and estimate % cuttable and the top 2 cut reasons. Use the cached release notes for the target model if any; otherwise say "no release notes cached; heuristics only". Do not fetch release notes or do per-line verdicts.
3. Print one table: skill (with its source: personal / project / synced / plugin name; plugin and synced rows are marked report-only), target model, size (est. tokens), est. % cuttable, top 2 cut reasons, sorted by est. tokens saved descending. Number the rows: that order is the recommended triage order (`/skill-ablation:triage <skill>`). Add a line suggesting `/skill-doctor` to find never-used skills worth turning off outright.
