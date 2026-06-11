---
name: autonomous-flow
description: >-
  Use to autonomously take freshly-implemented work all the way to a review-ready pull request.
  Runs a fixed finish-and-ship pipeline: self-verification, automated code review, browser /
  acceptance-criteria testing, commit-dance history cleanup, PR creation + push, then waiting for
  and responding to the Claude review bot. Trigger whenever the user says "/autonomous-flow", "run
  the autonomous flow", "finish and ship this", "take this to a PR", "run the full finish pipeline",
  or right after the tasks of a written implementation plan have been implemented and the next step
  is verification → review → PR → bot review. It is most often invoked mid-stream, after a plan was
  written and implementation has already started or completed. Always create a visible, step-by-step
  todo list with a progress bar so the user can monitor what is done and what is left. The pipeline
  is generic: prefer the project's own skills where they exist (e.g. a project-specific
  browser-testing skill, a commit-dance skill, a PR-review skill) and fall back to generic
  equivalents, and rebase onto / target the repository's documented base branch rather than
  assuming main.
---

# Autonomous Flow

Take just-implemented work through every gate between "code written" and "PR ready for human review,"
without stopping to ask permission between stages. The human's review window is the finished PR — your
job is to make that PR as correct, clean, and well-narrated as possible before they look.

## When to use

Invoke this after the implementation tasks of a plan are done (or nearly done) and you want to ship.
Signals: "run the autonomous flow", "finish this", "take it to PR", "ship it", or you've just
completed the last task of an implementation plan. If implementation is clearly *not* finished, say so
and finish it first — this skill is the *finish line*, not the whole race.

## Operating principles

- **Run continuously.** Do not pause between stages to ask "should I continue?" The user opted into an
  autonomous flow. The only reasons to stop are a hard blocker you cannot resolve, a genuine ambiguity
  that changes the outcome, or completion. Surface those; otherwise keep going.
- **Evidence over assertion.** Never claim a stage passed without showing the command output that
  proves it. A stage with failing checks is not done.
- **Fix as you go.** Each stage that can surface issues has a fix-loop: fix, re-run that stage's check,
  repeat until green — then advance.
- **Prefer project skills, degrade gracefully.** Where a project ships its own skill for a stage, use
  it. Otherwise use the generic fallback named in that stage. Announce which one you picked.
- **Respect repo conventions.** Branch strategy, commit message format, and signing rules come from the
  repo (check its `CLAUDE.md` / contributing docs). Never assume `main` is the base branch.

## Step 0 — Set up the progress tracker (do this first)

Before any stage runs, create a todo list (TodoWrite, or the session's task tool) with one item per
stage below, all `pending`. This is how the user monitors the run. At each stage boundary, mark the
finished item `completed`, the next one `in_progress`, and post a one-line progress bar so progress is
visible even without opening the todo panel:

```
Autonomous flow  ▓▓▓░░░░  3/7  ·  Browser & acceptance testing
```

Use 7 filled/empty blocks (`▓`/`░`) matching `N/7`. Keep the label to the current stage's name.

The seven stages:

1. Self-verification
2. Automated code review
3. Browser & acceptance-criteria testing
4. Commit-dance (history cleanup)
5. Create PR & push
6. Await Claude review bot
7. Respond to the bot (one refactor commit)

Also capture, once, up front:
- **Base branch** — read the repo's branch strategy (`CLAUDE.md`); e.g. some repos promote
  `development → staging → main`, so the PR base is `development`, not `main`. Fall back to the remote
  default branch if undocumented. Record it; stages 4 and 5 both need it.
- **Acceptance criteria (ACs)** — the contract you verify in stage 3 and reference in the PR body.
  Source them in this order: (1) the tracking issue (Linear/Jira/GitHub) — ACs are usually listed
  there; (2) the design **spec** produced by `/brainstorming` (and any ADR from
  `/design-first-collaboration`), which is where ACs discovered mid-design get written down — the
  typical lead-up to this skill is `/brainstorming → (optional) /design-first-collaboration →
  /writing-plans → subagent-driven dev`, so a spec usually exists and the ACs should be clear from it;
  (3) failing those, the implementation plan or the diff itself.

**Acceptance-criteria checkpoint.** Always state the ACs you'll verify (one line each) so the run is
auditable. If they're clear from the issue or the spec, just list them and proceed — no gate. If
they're **unclear, missing, or you had to infer them from the diff**, treat it as a soft checkpoint:
post the summarized ACs, invite corrections, and **proceed automatically if the user doesn't respond
within ~5 minutes** — schedule a wake-up for the timeout rather than blocking the flow indefinitely
(the user opted into autonomy). Fold in whatever they reply, then continue.

---

## Stage 1 — Self-verification

**Goal:** the working tree is internally consistent and all automated checks pass.

**How:** Prefer a `verification-before-completion` skill if one is available (e.g.
`superpowers:verification-before-completion`); it enforces "run the commands, show the output, don't
claim success without evidence." Otherwise, run the project's own checks directly — typically the test
suite, type-check, linter, and build — from the correct working directory (some monorepos require
running tooling from a sub-package, not the repo root; honor that).

**Fix-loop:** if anything fails, fix it, re-run, repeat until green. Capture the final passing output.

---

## Stage 2 — Automated code review

**Goal:** catch bugs, silent failures, weak types, and convention violations before a human or bot does.

**How:** Prefer a PR-review skill if the project/environment has one (e.g. a `review-pr` /
`pr-review-toolkit` skill, or a `/code-review` command). Otherwise do a structured self-review of the
diff: read every changed hunk, look for correctness bugs, error-handling that swallows failures,
missing tests, and deviations from sibling code.

**Fix-loop:** triage findings. Apply the ones you're confident in; for anything uncertain, verify
against the codebase before acting (don't perform-agree with a tool). Re-run stage 1's checks after
fixes so verification still holds.

---

## Stage 3 — Browser & acceptance-criteria testing

**Goal:** the feature actually works in the running app, and **every** acceptance criterion is met.

**How:** Prefer the **project's own** browser/app testing skill if one exists (these are usually named
for the app, e.g. `*-webapp-testing`). If none is accessible, use a generic browser-testing skill or
drive the app manually (start it, log in, navigate to the changed surfaces, screenshot). Walk the
acceptance criteria one by one and confirm each.

**Don't fabricate or force-produce hard-to-stage states.** Some ACs need data or a user role that
isn't available locally — a transient/error status with no seed data, a permission-restricted user, a
rare edge state. Don't contort the app or invent data to manufacture them. Verify what's genuinely
reachable, then **explicitly flag each un-exercised AC**: what it was, why it couldn't be run, and
where it's otherwise covered (unit tests, or the same code branch as an AC you *did* verify). Carry
these flags into the final summary and the PR body so the human can decide whether to seed data or run
a fuller E2E — a silently-skipped AC reads as "verified" when it wasn't.

**Fix-loop:** any bug found here goes back to code, gets fixed, and gets re-verified (stages 1–2 on the
touched code). Don't proceed with a known broken criterion.

---

## Stage 4 — Commit-dance (history cleanup)

**Goal:** the branch reads as a clean, pedagogical narrative a reviewer can walk file-by-file, with no
overlapping or duplicated changes across commits — freshly rebased on the base branch.

**How:** Prefer a `commit-dance` skill if available. The required shape:
- Rebase **freshly onto the up-to-date base branch** identified in step 0 (fetch first). Resolve
  conflicts.
- Re-commit the end state into a sequence of coherent commits where each commit is a self-contained,
  understandable step and **no two commits touch the same lines back-and-forth** (no "add then fix"
  churn). The final tree must be identical to the verified working tree.
- Follow the repo's commit message conventions (tense, scope prefix, trailers, signing policy).

**Fix-loop:** after the dance, re-run stage 1's checks on the final tree to prove the rewrite didn't
break anything.

---

## Stage 5 — Create PR & push

**Goal:** a pushed branch and an open PR targeting the correct base branch, with a clear body.

**How:** Push the (possibly force-updated after rebase) branch. Open the PR against the **base branch
from step 0** — not `main` unless that genuinely is the base. Write a PR body that summarizes intent,
the acceptance criteria and how they're met, and any follow-ups/known-gaps. Hold all pushes until this
stage — don't push incrementally during earlier stages.

---

## Stage 6 — Await the Claude review bot

**Goal:** get the automated review bot's comments.

**How:** Many repos run a Claude review bot (e.g. a GitHub Action) on new PRs. Poll the PR for its
review / inline comments rather than guessing how long it takes. If after a reasonable wait no bot
review appears (it may be disabled or unauthenticated in this repo), say so and move on — don't block
forever.

---

## Stage 7 — Respond to the bot (one refactor commit)

**Goal:** the PR reflects considered responses to the bot, with agreed fixes applied tidily.

**How:**
- Evaluate each bot comment with rigor (prefer a `receiving-code-review` skill if available): verify the
  claim against the code before agreeing. Agreement must be earned, not performed — a confidently-worded
  bot comment can still be wrong.
- Apply the fixes you agree with as **a single refactor commit** (not one commit per comment), so the
  follow-up reads cleanly.
- **Reply to the bot's comments**: for each, either "agree — fixed in <commit>" or "disagree — <reason>."
  Be concise; let descriptive code and commit messages carry the detail.
- Before pushing follow-ups, confirm the PR is still **open** (pushing to a merged PR's source branch
  silently does nothing useful). Then push.

---

## Done

When stage 7's commit is pushed and replies are posted, the PR is ready for the human. Post a final
summary: the PR link, what each stage found and fixed, acceptance-criteria status, and anything you
flagged as out-of-scope or unresolved. Then stop — the human reviews from here.

## Escalation

Stop and surface to the human (don't silently guess) when: verification can't be made to pass; the
acceptance criteria are ambiguous or unmet in a way you can't fix; a rebase conflict needs product
judgment; or the base-branch / push target is unclear. Bad shipped work is worse than a paused flow.
