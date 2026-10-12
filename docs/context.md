# context

survives /clear and compaction. keep it short and current.

## current task
first-draft architecture is complete; waiting on user review of docs.

## state
- docs/architecture.md, decisions.md updated with all decisions through 2026-10-12. docs/glossary.md is gitignored study notes.
- no code yet. git: old arquitecture.md deleted, docs/ and .gitignore changes untracked/uncommitted. user reviews and commits.

## next steps
- user reviews docs diff.
- strict tdd from the first line of code: failing test first, shown failing, then implementation (architecture.md section 5).
- scaffold flutter project with lib/core and lib/features/<feature>/{domain,data,presentation}.
- implement and test the progression engine first (pure dart, no flutter).
- add an import-boundary check to ci (tool not chosen; verify options first).

## open questions
- domain coverage threshold (proposed 90%), coverage tool, import-boundary check tool.
- licence (before first public release; check seed dataset licence).
- per-exercise increments for dumbbells and machines.
- overshoot rule, deload severity, catalog delivery (bundled vs downloaded).
- whether `progression` stays its own feature.
