---
name: codepath-phase-iv-pr
description: >-
  Execute CodePath Phase IV: pre-submission checks, open a pull request, request
  review, respond to maintainer feedback, and update contribution_readme for
  check-in. Use when the user says Phase IV, open the PR, submit the PR, request
  review, or Phase IV Complete. Includes nous-core overrides so PRs target the
  assignment integration branch, not main.
---

# CodePath Phase IV — Pull Request

Course milestone: open the PR, get it in front of reviewers, keep the contribution
README updated, then check in as Phase IV Complete.

## Project adaptations (nous-core / provider leaves)

Course text often says rebase onto `main` and PR into upstream `main`. **For
provider-leaf issues on this fork, that is wrong.** Use the maintainer-assigned
integration branch (example for #315 / OpenRouter #410):

| Course generic | This assignment |
|---|---|
| `git rebase origin/main` | `git fetch upstream && git rebase upstream/<integration-branch>` |
| PR base = upstream `main` | PR base = `orthogonalhq:<integration-branch>` |
| Head | `PaulPio:<feat-branch>` |
| PR template | Prefer `.github/pull_request_template.md` over the course-only template |
| Reviewer | Tag the maintainer who guided the issue (e.g. `@atlamors`). Check recent PRs in the same area if unsure. No `CODEOWNERS` in this repo today. |
| `--force-with-lease` | Only after a rebase that rewrote commits |

For #315 specifically:

- Integration base: `feat/contributor-friendly-inference-provider-surface`
- Head: `feat/dashscope-provider-leaf`
- Title: `feat(providers): add DashScope (Qwen) provider leaf`
- `Closes #315`

## Branch split (hard rule on this fork)

| Work | Branch |
|---|---|
| Sync, verify, open PR, push review fixes | Feature branch only (e.g. `feat/dashscope-provider-leaf`) |
| `contribution_readme.md` + course skills under `.cursor/skills/` | **`Reports` only** |

Never put `contribution_readme.md` or course skills on the PR head. After the PR
URL exists: `git checkout Reports`, update the README there, commit/push to
`origin/Reports` if versioning it.

Cross-link: README checkbox/status edits follow
[`contribution-readme-submissions`](../contribution-readme-submissions/SKILL.md).

## Hard rules

1. No secrets in PR body, comments, or README (API keys, tokens).
2. Do not commit process docs into the upstream PR.
3. Prefer the repo PR template sections; put course “why / evidence” inside them.
4. Professional tone in public review threads (see Tone below).
5. Agent does **not** post to Slack/LMS — remind the user.

---

## Step 1: Final pre-submission checks (~30 min)

Before opening the PR:

1. Checkout the **feature** branch (not `Reports`).
2. Sync upstream integration branch (not `main` unless the issue says so):
   ```bash
   git fetch upstream
   git rebase upstream/<integration-branch>
   # push --force-with-lease only if rebase rewrote history
   git push origin HEAD
   ```
3. **Fix still works** — reproduce the original gap; it should be gone (e.g. leaf
   directory + catalog entry present; live probe if applicable).
4. **Tests** — run the relevant suite (or full). No **new** failures vs recorded
   baseline. Note known environment failures honestly.
5. **Diff hygiene** — review every changed line vs the integration base:
   ```bash
   git diff upstream/<integration-branch>...HEAD
   ```
   Remove debug prints, stray comments, unrelated formatting.
6. **Commits** — conventional commits preferred; messy history is OK for community
   PRs if messages are still readable.

## Step 2: Open the pull request (~30 min)

```bash
gh pr create --repo orthogonalhq/nous-core \
  --base <integration-branch> \
  --head PaulPio:<feat-branch> \
  --title "<conventional title>" \
  --body "$(cat <<'EOF'
## Summary
…

## Linked Issue
Closes #N

## Changes
- …

## Verification
- [ ] Tests pass (`pnpm test`) — note subset/baseline if needed
- [ ] Lint passes (`pnpm lint`)
- [ ] Typecheck passes (`pnpm typecheck`) — note pre-existing unrelated failures
- [ ] Build passes (`pnpm build`)
- [ ] Manually ran the change end-to-end (describe below)

### Manual behavior check
…

## Checklist
- [x] Branch follows assignment base (integration branch, not `main`)
- [x] Commits follow Conventional Commits
- [x] Docs updated if behavior changed (or N/A)
EOF
)"
```

### Writing a strong description

**Do:** explain why before what; `Closes #N`; specific approach rationale; before/after
evidence (probe HTTP codes, test counts — never keys).

**Don't:** "Fixed the bug"; empty body; ignore the repo template; paste secrets;
restate the entire diff.

Course-only sections (What / Why / Screenshots / acceptance criteria) map into
Summary + Manual behavior check + Verification checkboxes.

## Step 3: Request a review (~5 min)

1. Read CONTRIBUTING / PR template for reviewer instructions (if any).
2. Comment on the PR tagging the relevant maintainer:
   ```text
   Hi @atlamors — this PR adds the DashScope (Qwen) certified provider leaf for #315
   (OpenAI-compatible ChatCompletionsProvider leaf, same shape as OpenRouter #410).
   Would appreciate a review when you have time.
   ```
3. If unknown who to tag: recent merged PRs in the same area, or wait — many
   projects watch the PR queue. Follow up politely after 5–7 business days.

## Step 4: Respond to feedback (ongoing)

Reviewers may approve, request changes, ask questions, or suggest alternatives.

**When they request a code change:**

1. Read carefully; ask if unclear.
2. Make the change on the **feature** branch; push a new commit.
3. Reply on the thread: `Updated in <hash>. <what changed> as suggested.`
4. If you disagree, explain technical reasoning respectfully.
5. When all items are addressed, comment tagging the reviewer that it is ready
   for another look.

**Questions:** reply clearly and promptly; link code/issue as needed.

**Different approach:** prefer maintainer guidance when reasonable; otherwise
explain why with evidence.

**Silence:** after 5–7 business days — `Hello! Is there anything I can update to help move this forward?`

### Tone guidelines

You represent yourself and CodePath in public:

- Professional, clear, no slang
- Responsive (within 24h when possible)
- Grateful for review time, even when critical
- Specific: "Updated X in `path/file`" beats "Done"

## Step 5: Update README and check in (~15 min) — `Reports` only

1. `git checkout Reports`
2. Update `contribution_readme.md` (see contribution-readme-submissions):
   - **PR Link** — submitted PR URL
   - **PR Description** — 1–2 sentences
   - **Maintainer Feedback** — summary (or “Awaiting first review”)
   - **Status** — Awaiting review / Iterating / Approved / Merged; then Phase IV Complete for LMS
3. Commit README/skill updates on `Reports` only if the user versions them on the fork.
4. Remind user (do not post for them):
   - Submit contribution README to LMS as **Phase IV Complete**
   - Celebrate in Slack `#dts-su26-ai301-celebration`:
     `Phase IV Complete — PR submitted! <PR url>`
