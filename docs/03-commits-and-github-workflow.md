# 03 — Commits & GitHub Workflow

> Challenge criterion: **Estruturação de Commits** — organized, descriptive history.
> Reviewers read `git log` like a story. Each commit should be one understandable step.

---

## 1. Conventional Commits

Format:
```
<type>(<scope>): <short summary in imperative mood>

<body: WHY this change, not what — the diff already shows what>

<footer: refs, BREAKING CHANGE, co-authors>
```

### Types
| Type | Use for |
|---|---|
| `feat` | New behavior visible to users of the API/system |
| `fix` | Bug fix |
| `test` | Adding or fixing tests only |
| `refactor` | Code change with no behavior change |
| `docs` | README, docs/, `@moduledoc` |
| `chore` | Tooling, deps, config, scaffolding |
| `ci` | GitHub Actions |
| `perf` | Performance improvement |
| `build` | Docker, release config |

### Scopes for this project
`tokens`, `manager`, `api`, `db`, `openapi`, `telemetry`, `ci`, `docker`, `deps`.

### Rules
1. **Summary ≤ 72 chars**, imperative ("add", not "added"/"adds"), no trailing period.
   Test: *"If applied, this commit will ___"*.
2. **Atomic**: one logical change per commit. It should compile and pass tests on its own.
3. **Tests travel with the code** they test (same commit), unless it's a `test:`-only commit.
4. **Body explains why** when it isn't obvious — decisions, trade-offs, links to doc 01.
5. Never commit: secrets, `_build/`, `deps/`, `cover/`, `.env`, editor files.
6. Never mix formatting churn with logic changes.

### Good vs bad
```
❌ update stuff
❌ fix bug
❌ WIP
❌ feat: add manager, tests, controller, swagger and fix seeds

✅ feat(manager): evict oldest active token when pool is exhausted

   When all 100 tokens are active, a new activation releases the token
   with the oldest activated_at (reason: evicted) before granting it.
   Implements rule R5 from docs/01.

✅ fix(manager): ignore stale expiry timers after re-activation

   A timer armed for usage A could release the token after it was
   evicted and re-activated for usage B. Timers now carry usage_id and
   are dropped when it no longer matches the open usage.
```

## 2. Branching (GitHub Flow)

- `main` is always green and deployable.
- One short-lived branch per feature: `feat/manager-eviction`, `fix/stale-timers`, `ci/dialyzer-cache`.
- Open a **Pull Request** even when working alone — it documents the change and runs CI.
- **Merge strategy: rebase-and-merge** (or merge commit) — *not squash* — because the
  individual commits *are* the deliverable being graded. Clean them up locally with
  `git rebase -i` before opening the PR (run it in your own terminal).

### PR template (`.github/pull_request_template.md`)
```markdown
## What
## Why
## How to test
## Checklist
- [ ] `mix precommit` passes
- [ ] Docs/README updated
```

## 3. Useful habits

| Habit | Command |
|---|---|
| Stage only part of a file (keep commits atomic) | `git add -p` |
| Review exactly what you're committing | `git diff --staged` |
| Fix the last commit (before pushing) | `git commit --amend` |
| Make a fix that belongs to an earlier commit | `git commit --fixup <sha>` then `git rebase -i --autosquash main` |
| See the story | `git log --oneline --graph` |

Optional: a `pre-commit` git hook that runs `mix format --check-formatted && mix credo --strict`.

## 4. Planned commit history

A target "story" for the repo. Real history may differ, but it should read like this:

```
chore: scaffold Phoenix 1.8 JSON API project
chore(docker): add docker-compose with PostgreSQL 17
docs: add design, quality, commit, testing and CI docs
chore(deps): add credo, dialyxir, sobelow, mix_audit
ci: run format, credo, sobelow and tests on GitHub Actions
feat(db): create tokens and token_usages tables with constraints
feat(tokens): add Token and TokenUsage schemas and changesets
feat(db): seed the 100 pre-generated tokens idempotently
feat(tokens): add read queries for listing tokens and history
feat(manager): activate an available token for a user
feat(manager): evict oldest active token when pool is exhausted
feat(manager): release tokens automatically after TTL via timers
fix(manager): ignore stale expiry timers after re-activation
feat(manager): sweep expired tokens periodically as a safety net
feat(manager): recover timers and expired tokens on boot
feat(manager): release all active tokens on clear
test(manager): prove no double allocation under concurrent load
feat(api): add POST /api/tokens/activate
feat(api): add GET /api/tokens with optional status filter
feat(api): add GET /api/tokens/:id with active user
feat(api): add GET /api/tokens/:id/usages
feat(api): add DELETE /api/tokens/active
feat(openapi): document endpoints and serve Swagger UI
feat(telemetry): emit token lifecycle events and dashboard metrics
ci: add dialyzer with PLT cache and coverage threshold
docs: write README with setup, API usage and architecture
```
