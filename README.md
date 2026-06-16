# polymorph-mindset-plugins

Custom [Claude Code](https://code.claude.com) skills from my *Polymorph Mindset* / NDC AI talk.

> **Egne skills for egne behov.** Your own skills for your own needs. These are real skills I use, lightly cleaned for sharing. The point isn't to install them and be done. It's to see the shape: what makes a skill specific enough to be useful, and how it captures a workflow that would otherwise live in your head. Read them, fork what's useful, and make it yours. They're not maintained products.

## Two ways to use this

### Read for inspiration (recommended)
Browse [`plugins/polymorph-mindset/skills/`](./plugins/polymorph-mindset/skills/) on GitHub. Each skill has a `SKILL.md` (the skill itself) and a `NOTES.md` that says what to notice, what's specific to me, and how you'd adapt it.

### Install and try
In Claude Code:

```
/plugin marketplace add marinahaugen/polymorph-mindset-plugins
/plugin install polymorph-mindset@polymorph-mindset-plugins
```

Skills then run namespaced, e.g. `/polymorph-mindset:design-first-collaboration`.

## The skills

| Skill | What it does |
|---|---|
| `pedagogical-pr-review` | File-by-file PR review for *learning*, with per-file gates and a numbered improvement list |
| `design-first-collaboration` | A structured design talk before code (Capabilities → Components → Interactions → Contracts) that emits an ADR |
| `review-dep-update` | Dependency-PR review: breaking changes plus a security audit, pulling fresh docs via Context7 |
| `pentest` | A thin orchestration layer that runs a controlled pen-test via an MCP-backed Kali tool |
| `autonomous-flow` | Takes finished work to a review-ready PR: verify, review, browser-test, history cleanup, then the PR |
| `commit-dance` | Re-clusters a messy branch history into a clean, file-by-file narrative without changing the final tree |

## What these are NOT

They aren't auto-loaded, and they aren't generic templates. They're snapshots of skills I built because I needed them. Copy one, strip out what's specific to me, and make it yours.
