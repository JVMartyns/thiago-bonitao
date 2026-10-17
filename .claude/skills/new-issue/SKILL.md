---
name: new-issue
description: Draft an agent-ready GitHub issue for this repo from a rule, idea or bug, show it for approval, then create it with labels. Use when the user asks to create, write or open an issue.
argument-hint: <rule id, idea or bug description>
disable-model-invocation: true
allowed-tools: Read, Grep, Glob, Bash(gh issue list:*), Bash(gh issue view:*), Bash(gh label list:*)
---

# /new-issue — draft an agent-ready issue

Request: **$ARGUMENTS**

Repo: `JVMartyns/thiago-bonitao`. The issue format and Definition of Ready are defined in
`docs/06-issue-driven-agentic-development.md` §2–3 and `.github/ISSUE_TEMPLATE/agent-task.yml`.
Follow them exactly.

## Steps

1. **Ground it.**
   - Read the relevant parts of `docs/01-problem-solving-and-design.md` (rules R1–R7, decisions A1–A8)
     and, if the code exists, the files the change would touch.
   - Run `gh issue list --repo JVMartyns/thiago-bonitao --state all --limit 50` to avoid duplicates
     and find what this issue is blocked by.

2. **Size it.** If it would be `size:L` (more than ~300 changed lines or more than one PR), propose
   a split into several issues instead and draft each one.

3. **Draft** the body with exactly these sections (same headings as the issue form):
   - `### Goal` — outcome in 1–2 sentences
   - `### Context` — links to rules/decisions in docs/01, related issues, files
   - `### Acceptance criteria` — `- [ ]` checkboxes, each objectively testable
   - `### Scope` — **In scope** / **Out of scope**
   - `### Files likely touched`
   - `### Constraints` — allowed libraries, patterns, things not to do
   - `### Verification` — exact commands (always include `mix precommit`)
   - `### Size` — `S (< 1h)` or `M (half a day)`

   Title: Conventional Commit style, `<type>(<scope>): <imperative summary>`, ≤ 72 chars.
   Blocked by: the issue numbers this depends on (shown in the draft, set with `--blocked-by`).
   Labels: `type:<type>`, `area:<scope>`, `size:<S|M>`, plus `agent:ready` only if every
   Definition of Ready item holds.

4. **Show the draft** (title, labels, body) to the user and ask for approval or edits.
   Do not create anything before they approve.

5. **Create** after approval:
   ```bash
   gh issue create --repo JVMartyns/thiago-bonitao \
     --title "<title>" --body-file <tmp file> --label "<comma,separated,labels>" \
     --blocked-by "<N,M>"   # omit if nothing blocks it
   ```
   Dependencies are native GitHub relationships, never a section in the body.
   Write the body to a temp file in the scratchpad directory, not the repo.

6. Reply with the issue URL and one line on what it unblocks.
