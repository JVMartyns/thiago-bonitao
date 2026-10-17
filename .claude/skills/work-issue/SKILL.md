---
name: work-issue
description: Implement one GitHub issue end to end in this repo — claim it, branch, test first, implement, run mix precommit, commit conventionally and open a PR that closes it. Use when the user asks to work on, implement or pick up an issue.
argument-hint: <issue number, or "next" for the lowest unblocked agent:ready issue>
disable-model-invocation: true
---

# /work-issue — issue → tested PR

Target: **$ARGUMENTS**

Repo: `JVMartyns/thiago-bonitao`. Rules: `AGENTS.md`. Commit format: `docs/03-commits-and-github-workflow.md`.
Testing approach: `docs/04-testing-strategy.md`.

This is a learning project: while working, briefly explain the Elixir/OTP/Ecto decisions you make
(why this pattern, what the alternative was), so the user learns from each step.

## 1. Pick and check the issue
- If the target is `next`: `gh issue list --repo JVMartyns/thiago-bonitao --label agent:ready --state open`
  and choose the lowest number whose `Blocked by` issues are all closed.
- `gh issue view <N> --repo JVMartyns/thiago-bonitao --comments`
- Stop and tell the user if: a blocking issue is still open, the issue lacks `agent:ready`, or an
  acceptance criterion is ambiguous. For ambiguity, propose a comment for the issue and the
  `needs-human` label — ask before posting.

## 2. Claim and branch
```bash
gh issue edit <N> --repo JVMartyns/thiago-bonitao --add-label agent:in-progress --remove-label agent:ready
git switch main && git pull --ff-only
git switch -c <type>/<N>-<short-slug>
```
`<type>` comes from the issue's `type:` label.

## 3. Test first, then implement
For each acceptance criterion, in order:
1. Write a failing test (unit or integration, per docs/04). Run it and confirm it fails for the right reason.
2. Write the minimal code to make it pass.
3. Refactor while green.
4. Commit: one atomic Conventional Commit per logical step, scope from the issue's `area:` label.
   End every commit message with the attribution trailer configured for this session.

Stay inside the issue's **In scope**. Anything else you notice → note it for a follow-up issue, don't fix it.

## 4. Verify
```bash
mix precommit
mix dialyzer   # only if specs or public APIs changed
```
Run every command in the issue's **Verification** section. All must pass; if something fails,
fix it — never weaken a test or a check to get green. Tick every acceptance criterion against
real evidence (test names, command output).

## 5. Open the PR
```bash
git push -u origin HEAD
gh pr create --repo JVMartyns/thiago-bonitao --title "<issue title>" --body-file <tmp file>
```
Body follows `.github/pull_request_template.md`: What, Why, `Closes #<N>`, How to test, checklist.
Write the body file in the scratchpad directory, not the repo.

## 6. Hand off
Reply with the PR URL, a short summary of what was built and what you learned/decided, and any
follow-up issues worth creating (offer `/new-issue` for them). Never merge — a human reviews and merges.
