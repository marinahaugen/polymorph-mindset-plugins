# Notes on autonomous-flow

## Pattern it demonstrates
A fixed, ordered pipeline encoded as a skill: verify → automated review → browser/acceptance test → history cleanup → PR → respond to the review bot. The value isn't any single step. It's that the sequence and its gates are written down once, so "finish and ship this" runs the same disciplined path every time instead of whatever the model improvises.

## What's specific to my context
- It leans on other skills I have (`commit-dance` for history, a browser-testing skill, a PR-review skill) and falls back to generic equivalents when a project doesn't have them.
- It assumes a PR-based workflow with a review bot, so adjust if your team merges differently.
- The visible todo-with-progress-bar is a deliberate choice so a human can watch the pipeline.

## How you'd adapt it
Strip the pipeline down to your definition of done. No review bot? Delete that stage. If "verify" is just `npm test`, say so explicitly. The skill earns its keep when every stage names a concrete command or sub-skill. Vague stages like "test it properly" get skipped under pressure.
