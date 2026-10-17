# AGENTS.md — rules for AI agents (and humans) working in this repo

Project: **thiago-bonitao** — token management API (Elixir 1.19 / OTP 28, Phoenix 1.8, PostgreSQL + Ecto).
Learning project based on the Just Travel technical challenge.

## Read first
- `docs/01-problem-solving-and-design.md` — rules R1–R7, decisions A1–A8, architecture. **Source of truth.**
- `docs/02-code-quality.md` — structure and conventions.
- `docs/03-commits-and-github-workflow.md` — commit format.
- `docs/04-testing-strategy.md` — what and how to test.
- `docs/06-issue-driven-agentic-development.md` — how issues and PRs flow.

## Workflow
1. Work from exactly one GitHub issue. Do not start issues that are `Blocked by` an open issue.
2. Branch: `<type>/<issue#>-<short-slug>` (e.g. `feat/7-evict-oldest`). Never commit to `main`.
3. Write the failing test first, then the implementation.
4. Commits follow Conventional Commits with the scopes from doc 03. Small and atomic.
5. Run `mix precommit` (and `mix dialyzer` when specs changed) before opening the PR.
6. PR body: summary, how to test, and `Closes #N`.
7. If something is ambiguous or out of scope: comment on the issue, add the `needs-human` label, stop.

## Code rules
- Controllers call the `TokenSystem.Tokens` context only; all writes go through `Tokens.Manager`.
- Public functions: `@doc`, `@spec`, tagged tuples `{:ok, _} | {:error, _}`.
- Time-dependent logic takes `now` as an argument; TTL and intervals come from config.
- No new dependencies unless the issue allows it.
- Never use `Process.sleep/1` with fixed durations in tests.

## Commands
```bash
docker compose up -d db   # PostgreSQL
mix setup                 # deps, db, migrations, seeds
mix precommit             # format, compile, credo, sobelow, tests
mix test                  # tests
mix phx.server            # API on http://localhost:4000
```
