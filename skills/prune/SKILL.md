---
name: prune
description: Delete skills you have marked obsolete in routing.yaml, after explicit confirmation. Use --dry-run to only list them.
argument-hint: "[--dry-run]"
disable-model-invocation: true
---

# prune

Arguments: `$ARGUMENTS`

Read `${CLAUDE_PLUGIN_ROOT}/shared/conventions.md` first (files, hard rules).

1. List skills with `obsolete: true` in routing.yaml: name, path, size. Drop synced ones with a note. None left: say so and stop.
2. `--dry-run`: print the list and stop.
3. Otherwise ask with AskUserQuestion (multiSelect) which to delete. Deletion is permanent (no variants kept), so state that once and show the path.
4. For each selected: `rm -r <skill-dir>` (use the resolved real directory, not through a symlinked SKILL.md), then remove its entry from routing.yaml. Report what was deleted.
5. Do not delete anything without the confirmation in step 3, even in auto mode.
