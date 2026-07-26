---
name: contribution-readme-submissions
description: >-
  Update and prepare contribution_readme.md for CodePath/OSS phase check-ins
  (Phase I–IV). Use when the user asks to update the contribution README, prepare
  a phase submission, mark Phase I/II/III/IV complete, draft PR text into the
  README, or sync Implementation Notes / Testing Strategy / checkboxes for a
  course deliverable.
---

# Contribution README submissions

Maintain the local `contribution_readme.md` as the course progress log. It is the
check-in artifact for each phase — not a file to commit to the upstream PR.

## Hard rules

1. **Never commit** `contribution_readme.md`, `CLAUDE.md`, or other local process
   notes. In this repo root `/*.md` is gitignored; still verify with `git status`
   before any commit/stage.
2. **Never write secrets** into the README (API keys, tokens, Bearer values,
   `.env` contents). Record probe *status codes* and *shapes* only. If a key
   appeared in a terminal/transcript, tell the user to rotate it — do not copy it.
3. **Preserve earlier phases.** Update in place: bump status, check boxes, add a
   new week/phase section. Do not delete Phase I/II narrative when doing Phase III.
4. **First person, student voice** for narrative sections (Why I Chose, Learnings).
   Keep evidence tables factual and terse.
5. **Links over paste.** Prefer issue/PR/commit/branch URLs and short hashes.

## When to run this skill

| User intent | Action |
|-------------|--------|
| "Update contribution readme for Phase N" | Apply that phase checklist below |
| "Phase N Complete" / check-in ready | Finish checklist + set **Status** + one-line check-in blurb |
| "Draft PR in the readme" | Fill **Pull Request** draft; do **not** open the GitHub PR unless asked |
| "Sync readme with what we built" | Diff branch vs README claims; fix stale checkboxes/commits |

Default file path: `contribution_readme.md` at repo root. If missing, recreate from
the structure below using issue/branch context — do not invent merge/PR links.

## Document structure (keep these sections)

```markdown
# Contribution N: <Issue title>

**Contribution Number:** N
**Student:** …
**Issue:** <url>
**Fork:** <url>
**Status:** Phase X — … / Phase X Complete
**Working branch:** [name](fork-tree-url)
**Key commit:** (from Phase III+)
**Prior contribution:** (if any)

**Phase X check-in:** <2–4 sentences: what shipped, evidence, what's next>

## Why I Chose This Issue
## Understanding the Issue
## Reproduction Process          # Phase II+
## Solution Approach             # Plan; mark Implement/Review items done later
## Testing Strategy              # Checkboxes → tick as evidence lands
## Implementation Notes          # Week/Phase progress log (newest first)
## Pull Request                  # Draft in III; link in IV
## Learnings & Reflections
## Resources Used
```

## Phase checklists

### Phase I — claim + plan

- [ ] Header: issue, fork, status `Phase I — …`
- [ ] Why I Chose / Understanding filled from issue + maintainer comments
- [ ] Acceptance criteria listed as unchecked boxes
- [ ] High-level plan and file touch list (no implementation claims)
- [ ] Comment on the GitHub issue if branch confirmation is needed (note the link)
- [ ] Check-in blurb: plan ready; next is reproduction / sync

### Phase II — reproduction + plan lock

- [ ] Status → `Phase II — Complete` (or equivalent)
- [ ] Reproduction Process: environment table, steps run, observed vs expected
- [ ] Baseline failures recorded (count + root-cause class) so Phase III can diff
- [ ] Solution Approach + Implementation Plan locked (decisions + risks)
- [ ] Testing Strategy checkboxes present but mostly unchecked
- [ ] Implementation Notes: Week/Phase II progress; **no** "code landed" claims
- [ ] Check-in blurb: repro done, plan finalized, next is implement

### Phase III — implement + verify (PR usually deferred)

- [ ] Status → `Phase III Complete`
- [ ] Working branch + **key commit** hash/link (pushed to fork)
- [ ] Acceptance criteria boxes updated from evidence
- [ ] Testing Strategy: tick unit/integration items that actually ran; paste gate table
- [ ] Live probe / manual results: HTTP codes + envelope notes — **no keys**
- [ ] Implementation Notes: Phase III section (files, decisions, challenges)
- [ ] Pull Request: **draft** title/base/body only; `PR Link` still pending unless user opens it
- [ ] Learnings: add Phase III bullets
- [ ] Check-in blurb: what shipped, gate vs baseline, PR = Phase IV (if course says so)
- [ ] Confirm `git status` does not stage this README

### Phase IV — PR + review loop

For opening the PR, review requests, and feedback tone, follow
[`codepath-phase-iv-pr`](../codepath-phase-iv-pr/SKILL.md).

**README edits happen on `Reports` only** — never on the feature/PR head branch.

- [ ] `git checkout Reports` before editing this file
- [ ] Status → `Phase IV — Awaiting review` / `Iterating` / `Complete` as appropriate
- [ ] **PR Link** filled; Linked Issue / Closes #N matches the real PR
- [ ] Verification checklist mirrors PR template (tests, lint, typecheck, manual)
- [ ] Maintainer Feedback: summarize review threads + what you changed (or “Awaiting first review”)
- [ ] Implementation Notes: Phase IV week entry + review iterations
- [ ] Final check-in blurb when course asks for Phase IV Complete
- [ ] If committing the README on the fork, push to `origin/Reports` only

## How to edit safely

1. Confirm you are on **`Reports`** (not the PR feature branch) before editing.
2. Read the current `contribution_readme.md` fully before editing.
3. Gather facts from the feature branch / PR: `git log`, test output, issue/PR URLs.
4. Prefer surgical updates (status, checkboxes, new Week section, PR link).
5. When marking Complete, ensure every newly checked box has evidence in the same
   doc (command result, probe table, commit/PR link).
6. Remind the user: submit via the course LMS; do **not** put this file on the
   upstream PR head.

## Baseline & honesty rules

- Compare new test results to the **recorded Phase II baseline**. Say "zero new
  failures" only when true; list remaining pre-existing failures by name/class.
- Do not claim UI smoke, chat probe, or typecheck pass without a recorded result.
- Out-of-scope fixes done "in passing" must be called out (e.g. roster-order fix)
  so the PR description and README stay aligned.

## Secrets & probes

```text
Good:  GET /v1/models without auth → 401; with Bearer key → 200; envelope { object, data:[{id}] }
Bad:   Authorization: Bearer sk-...
Bad:   export DASHSCOPE_API_KEY='sk-...'
```

If the user pastes a key into chat or a terminal log is attached, warn to rotate;
strip any key if it accidentally entered the README.

## PR draft alignment

When drafting the README **Pull Request** section, match `.github/pull_request_template.md`
sections (Summary, Linked Issue, Changes, Verification, Checklist). Keep the same
facts as Implementation Notes — do not invent CI green or merge status.
