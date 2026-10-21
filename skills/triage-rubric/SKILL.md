---
name: triage-rubric
description: Rubric for classifying each line or section of a skill as CUT, SOFTEN, KEEP or UNSURE. Loaded by /skill-ablation:scan and /skill-ablation:triage; not meant to be invoked directly.
user-invocable: false
---

# Triage rubric

Classify every section or line of the skill. Quote the first few words and give a one-line reason. Cite the release-notes bullet (by number) when relevant.

## Verdicts

- **CUT (generic)**: generic knowledge or best practice a strong model does by default ("think step by step", "write tests", "use clear names").
- **CUT (old workaround)**: a workaround for an older-model failure mode that the release notes suggest is fixed (truncation, placeholder code, incomplete output).
- **CUT (standard example)**: worked examples of standard, widely known formats or idioms (a README, a REST endpoint, a textbook pattern in the skill's own framework). Length is not required: a 10-line stock snippet counts. Keep examples that show the user's own conventions.
- **CUT (reference)**: content that restates documentation the model already knows and can fetch: API property/method/class tables, enum listings, keyboard shortcuts and menu walkthroughs, framework comparison or feature-support tables, install steps for platforms or plugins outside the user's setup, and "See references/..." paragraphs that only re-summarize the file they point to (shorten those to the link).
- **SOFTEN**: content is right, but heavy emphasis (ALL CAPS, MUST, NEVER, CRITICAL) likely causes over-application on a model that follows instructions more closely. Rewrite in plain tone; keep the instruction.
- **KEEP**: user-specific conventions, paths, or team process; internal or post-cutoff knowledge; deterministic scripts; guardrails against costly or irreversible mistakes (these keep their emphasis).
- **UNSURE**: can't judge from the text alone. Each UNSURE gets one realistic test prompt to run with the skill off.

## Tie-breaker
Would a strong senior engineer new to this codebase need to be told this? No → lean CUT. Yes because it's specific to the user → KEEP.

## Rules
- Never cut anything specific to the user's setup, even if it looks generic.
- Weight by target model: aggressive for opus, moderate for sonnet, conservative for haiku. For haiku default to KEEP unless the line is clearly useless.
- Knowledge sections apply the cutoff in **both** directions. Newer than (or within a month of) the cutoff → KEEP, including breaking changes where the model would silently write the old behavior. Older than the cutoff and widely documented → CUT (reference / standard example), **unless** it is a gotcha (behavior that defies expectation or fails silently, e.g. symptom→fix pairs), a security warning, or a house convention. Do not default to KEEP just because no CUT label obviously fits; the absence of a label is not evidence.
- Docs-available cost test (opus and sonnet): a skill load costs 1–3K tokens; fetching docs costs 5–20K plus a round trip. So keep compact gotchas and pointers; drop what the model can recall or look up as plain reference.
- Third-party skill packs rarely contain "my setup" content; the setup rule mostly protects paths, naming and process the user added. Check the user's actual setup before judging (languages in use, plugins installed, frameworks present): content for languages or platforms they don't use is CUT, not "generic, keep".
- **Classify per block, not per section type.** Batch runs must not rely on section-name rules alone (e.g. "C# blocks", "checklists"); apply the tie-breaker to every table, code block and paragraph. Report what was classified mechanically.
- Nothing is verified without an empirical run. Every aggressive CUT of recall-able content needs a seed case.
- A skill whose every section is CUT is **obsolete**: recommend setting `obsolete: true` in routing.yaml (see `/skill-ablation:prune`).

## Scan-mode heuristics (quick pass, no per-line work)
Measure token buckets first (code fences, table rows, "See references/" pointer lines, version-specific notes, other prose; a few lines of awk) and estimate % cuttable from them: tables and stock code are the main cuttable mass, version notes and gotcha tables are the main keepable mass. Also weigh: share of generic advice prose, count of emphasis words (MUST/NEVER/ALWAYS/CRITICAL/ALL CAPS), old-workaround phrases ("do not truncate", "never use placeholders", "output the complete file"), and how much is deterministic or user-specific (keep). Name the top 2 cut reasons from the verdict labels above.
