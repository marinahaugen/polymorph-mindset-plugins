# Notes on commit-dance

## Pattern it demonstrates
A safety-railed destructive git workflow. The skill encodes an exact sequence (backup tag → soft reset → re-commit by theme → verify zero-diff → force-with-lease) and, crucially, a table of hard rules with the reason each exists (`--mixed` never `--hard`, `--force-with-lease` never `--force`). It's a good example of a skill whose main job is to stop the model taking a dangerous shortcut under pressure.

## What's specific to my context
- The chapter-mapping step pairs naturally with my `pedagogical-pr-review` skill: the "chapters" become the file-by-file story a reviewer walks.
- Base-branch auto-detection tries `development main master`; reorder for your team.
- The commit-message conventions and test commands are just examples. Your project's `CLAUDE.md` is the real source of truth.

## How you'd adapt it
Keep the safety rules verbatim. They're the whole point. Swap the example test commands and commit style for yours. If your team never force-pushes shared branches, add that as a hard "don't use when" at the top.
