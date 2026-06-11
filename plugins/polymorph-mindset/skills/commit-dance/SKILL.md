---
name: commit-dance
description: Use when a feature branch has the right end-state but messy commit history (original work + many follow-up fixes + maybe merge churn) and the user wants the PR to read as a single coherent pedagogical narrative reviewers can walk file-by-file
---

# Commit Dance

## Overview

Re-cluster the commits on a feature branch into a clean pedagogical narrative *after* the original work has accumulated additional fixes, bug repairs, or review tweaks. The "dance" is the choreographed sequence:

```
backup → soft reset → re-commit pedagogically → verify zero-diff → force-push with lease
```

**Core invariant:** the final tree must be byte-for-byte identical to the pre-dance tree. Only the commit history changes shape — never the working tree.

## When to Use

- Long PR review: many "address review" / "fix typo" / "wip" commits accumulated; reviewer wants to walk the changes as a story
- Combining the original feature work + bug fixes discovered during testing into thematic chapters
- A branch that has the right code but a history that hides the architecture
- BEFORE merging a PR you want to be readable later as a learning artifact (`pedagogical-pr-review`)

**Don't use when:**
- The branch has merged commits from main/development (rebase those out first, separately)
- Other people are actively pulling the branch (force-push will require them to reset)
- The history is already clean and the only goal is "tidy commit messages" (use `git commit --amend` per commit instead)
- You're not certain about the chapter boundaries (do `pedagogical-pr-review` or planning first)

## Quick Reference

| Step | Command | Critical |
|------|---------|----------|
| 1. Verify branch | `git branch --show-current && git status --porcelain` | Tree clean OR only intended unstaged fixes |
| 2. Backup tag | `git tag <branch>-original-$(date +%Y-%m-%d-%H%M)` | **Before** anything destructive |
| 3. Backup commit log | `git log --oneline <base>..HEAD > /tmp/<branch>-original.txt` | For human reference |
| 4. (optional) Backup branch | `git push origin <branch>:<branch>-backup` | Offer; never auto |
| 5. Plan chapters | Show original commits + file diff; propose mapping | **Get user approval** |
| 6. Soft reset | `git reset --mixed $(git merge-base HEAD <base>)` | `--mixed`, NEVER `--hard` |
| 7. Per chapter | `git add <files> && git commit -m "..."` | Stage subsets, not `-A` |
| 8. Verify zero-diff | `git diff HEAD <backup-tag>` shows ONLY intentional fixes | Stop if anything unexpected |
| 9. Run tests | Project-specific (lint + unit + integration) | Confirm new HEAD is functional |
| 10. Push | `git push --force-with-lease` | NEVER `--force` |

## Workflow

### Step 1 — Pre-flight checks

```bash
# Confirm branch
git branch --show-current  # must match user's intended branch

# Inspect working tree
git status --porcelain
# OK if: empty, OR only files the user has flagged as intended fixes
# NOT OK if: untracked files unrelated to the task, or modified files the user
# didn't mention. STOP and ask.

# Auto-detect base branch (try in order)
for base in development main master; do
  git rev-parse --verify "origin/$base" >/dev/null 2>&1 && BASE="$base" && break
done
echo "Base: $BASE"

# Sync check vs remote
git fetch origin "$(git branch --show-current)" 2>/dev/null
git rev-list --left-right --count "HEAD...origin/$(git branch --show-current)" 2>/dev/null
# Output is "<ahead> <behind>". Behind > 0 → STOP and pull/rebase first.

# Look for merge commits (these break the soft-reset assumption)
git log --merges --oneline "$BASE"..HEAD
# If non-empty → WARN. The dance still works, but the resulting tree may not
# match the merge's parents the way the user expects.
```

### Step 2 — Backup (BEFORE anything destructive)

```bash
BRANCH=$(git branch --show-current)
STAMP=$(date +%Y-%m-%d-%H%M)
TAG="${BRANCH}-original-${STAMP}"

git tag "$TAG"
git log --oneline "$BASE..HEAD" > "/tmp/${BRANCH}-original-commits.txt"
git log "$BASE..HEAD" --format='%H%n%B%n---SEPARATOR---' > "/tmp/${BRANCH}-original-commits-full.txt"

# Optional remote backup (offer, don't auto)
# git push origin "$BRANCH:${BRANCH}-backup"
```

The tag is local-only by default. The reflog is also a fallback (`git reflog` — entries last ~90 days).

### Step 3 — Plan the chapters

Before resetting, agree on the new chapter sequence. Show the user:

1. The original commit list (`/tmp/${BRANCH}-original-commits.txt`)
2. The file-level diff: `git diff --name-status "$BASE..HEAD"`
3. A proposed numbered chapter sequence with **file-to-chapter mapping**

Example chapter map:

```
1. feat: introduces FooDomain — Foo.kt, FooTest.kt, FooHelper.kt
2. feat: adds Foo readiness gate — FieldKeys.kt, Profiles.kt, *Test.kt
3. fix: corrects Foo race condition — FooComponent.tsx (parts), FooHook.ts
...
```

**Get explicit user approval** before proceeding. If you're in plan mode, use `ExitPlanMode`. Otherwise ask directly. The chapter mapping is the contract between user intent and what the dance produces.

**Splitting a single file across chapters** (e.g., a feature change + a bug fix in the same file) requires `git add -p` interactive hunk staging. If the chapters are clean per-file, the dance is straightforward; if not, decide upfront whether to split or bundle.

### Step 4 — Soft reset

```bash
git reset --mixed "$(git merge-base HEAD "$BASE")"
```

What `--mixed` does:
- Moves HEAD to the merge base
- Resets the index to match HEAD (so nothing is staged)
- **Leaves the working tree untouched** (all your changes are still there)

**Never use `--hard`** — it discards the working tree, losing all your fixes.

After reset, `git status` should show ALL the changes (committed + unstaged) as unstaged. Confirm to the user that nothing is lost.

### Step 5 — Re-commit pedagogically

For each chapter, stage only its files, then commit:

```bash
git add path/to/file1 path/to/file2 ...

git commit -m "$(cat <<'EOF'
feat(SCOPE): chapter title in third-person singular present tense

Body explaining the WHY of this chapter. Aim for 2-4 sentences. Reference
related concerns or the design rationale. Don't restate WHAT — the diff
shows that.

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

Notes:
- Use `git add <files>` with explicit paths, **not `git add -A`** (avoids staging `.env`, `.wt-*`, scratch files)
- Use HEREDOC for the message to preserve formatting
- Match the project's commit-message style (check the project's `CLAUDE.md` for conventions)
- Include the `Co-Authored-By` trailer if the project uses it

### Step 6 — Verify zero-diff against backup

This is the **critical safety check**. The new HEAD's tree must match the backup tag's tree exactly.

```bash
git diff "$TAG" HEAD --name-only
```

Expected output: ONLY the files you intentionally fixed (the additions/changes that prompted the dance). If files appear that you didn't mean to touch → STOP. Something went wrong:

- Most common cause: a file was assigned to a chapter that no longer exists, or a file got staged in two chapters and the second commit reverted the first
- Diagnose: `git diff "$TAG" HEAD -- <unexpected-file>` shows what's different
- Fix by either staging the missed change in a follow-up commit, or starting over from the backup tag

If the diff is empty, the dance produced exactly the same tree (which would mean you bundled all the fixes into the chapters perfectly, including no fixes at all).

### Step 7 — Final sanity tests

Run the project's tests on the new HEAD. The recommit shouldn't have broken anything (the tree is identical), but the test suite is the truth.

Example (a Maven + npm monorepo):
```bash
cd backend && mvn verify
cd ../frontend && npm run lint && npm test
```

If anything fails, the dance is reversible: `git reset --hard "$TAG"` brings you back to the backup. (This is the one place `--hard` is safe — you're restoring from a known-good point.)

### Step 8 — Push with lease

```bash
git push --force-with-lease
```

**Never use plain `--force`** — `--force-with-lease` refuses if the remote has moved since you fetched, protecting against overwriting someone else's pushed work.

If the lease is rejected:
1. `git fetch origin` — see what changed
2. Inspect: `git log HEAD..origin/$(git branch --show-current)` — is it your CI bot? a human collaborator?
3. If safe: re-fetch, redo the dance from the backup tag, push
4. **Never** bypass with `--force` to "make it work"

## Critical Safeguards

| Rule | Why |
|------|-----|
| **Backup tag BEFORE soft reset** | The tag is your only recovery if something goes wrong (reflog is a fallback but not always reliable across `gc`) |
| **`git reset --mixed`, NEVER `--hard`** | `--hard` discards the working tree, losing every fix you made |
| **`git push --force-with-lease`, NEVER `--force`** | `--force` overwrites collaborators' work without warning |
| **Don't `git add -A`** | Stages untracked files (`.env`, scratch files, locally-generated artifacts) |
| **Always verify zero-diff** | The single source of truth that the dance preserved your work |
| **Stop if working tree has unrelated changes** | The dance assumes you know what's intentional vs accidental |

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Skipped the backup tag | Realized halfway something is wrong, no way back | `git reflog` to find the original HEAD; tag it now; restart |
| Used `git reset --hard` | Working tree is the base branch's; all fixes gone | If pushed: `git reset --hard <backup-branch-on-origin>`. If not pushed: hope reflog still has it |
| Used `--force` | Collaborator's commit silently overwritten | Restore from their machine; apologize; switch to `--force-with-lease` next time |
| `git add -A` staged scratch files | Random `.env`, `.tmp` files in commits | Reset the commit, restage with explicit paths |
| Bundled #12-fix-only into a feature commit | A pure bug fix is hidden inside a feature chapter | Either accept it (small bugs in feature commits is OK) or split via `git add -p` |
| Force-pushed to main/master | 🚨 disaster recovery scenario | Restore from backups; have a serious conversation about branch protection |

## Failure-Mode Awareness

**Branch has merge commits from main/development**: soft reset still produces the right end-tree, but you lose the merge information. Future `git log --graph` won't show the merges. Usually fine — but if your team relies on merge history for any reason, rebase out the merges first as a separate operation.

**Other people have pulled the branch**: force-push requires them to reset their local copy. Communicate clearly. If your team is small (1-3 people), just tell them. If your team is larger, consider creating a fresh branch with the new history and abandoning the old one (less elegant but no surprise).

**Partial rebase or stash in progress**: `git status` will show this. Resolve before starting the dance.

**Pre-commit / pre-push hooks fail mid-dance**: each chapter commit triggers them. If a hook fails:
- Don't `--no-verify` to bypass — fix the underlying issue
- The chapter's `git commit` failed cleanly (HEAD didn't move)
- Resolve, re-stage if needed, re-commit

## Reflog: The Safety Net Below the Safety Net

If everything goes wrong and your backup tag is also gone:

```bash
git reflog
# Find the SHA from before the dance started (look for "checkout" or
# "commit" entries from before your reset)
git reset --hard <SHA>
```

The reflog keeps entries for ~90 days by default. It's local-only — gone if you re-clone.

## Real-World Reference

This pattern was extracted from a real PR where a feature branch had accumulated a dozen "address review" commits. Re-clustering them into four thematic chapters (domain model → readiness gate → race-condition fix → tests) turned an unreadable history into a file-by-file narrative a reviewer could follow. Keep your own worked example handy — the chapter map is the reusable artifact.

## Status

This skill is **v1 — pre-baseline-test**. Per `superpowers:writing-skills` discipline, it should be tested with subagent baseline scenarios before being considered bulletproof. Specifically: drop a subagent on a synthetic feature branch with messy history, give them the same goal once without the skill and once with — capture their rationalizations and any safety-rule violations, then refine.

If you find yourself reaching for `--force` or `--hard` while applying this skill, that's a violation. Use the safer commands every time.
