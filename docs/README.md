# Token System — Project Docs

Learning project based on the **Just Travel "Gerenciamento de Tokens"** technical challenge
(Elixir + Phoenix + PostgreSQL/Ecto).

These docs are written *before* the code. They describe what we will build, why, and
the rules we follow while building it. Each one maps to an evaluation criterion of the challenge.

| Doc | Challenge criterion | What it answers |
|---|---|---|
| [01 — Problem solving & design](01-problem-solving-and-design.md) | Resolução de Problemas, Design e Arquitetura | What are the rules? What's ambiguous? How does auto-release work? Why this architecture? |
| [02 — Code quality](02-code-quality.md) | Qualidade do Código | How is the code organized? Which conventions and tools keep it clean? |
| [03 — Commits & GitHub workflow](03-commits-and-github-workflow.md) | Estruturação de Commits | How do we write commits, branches and PRs? What is the planned commit history? |
| [04 — Testing strategy](04-testing-strategy.md) | Testes | What do we test at each layer (unit / integration / concurrency) and how? |
| [05 — CI pipeline](05-ci-pipeline.md) | All of the above, automated | What runs on every push, in which order, and why? |
| [06 — Issue-driven & agentic development](06-issue-driven-agentic-development.md) | Workflow | How issues are written so humans or AI agents can implement them; labels; GitHub MCP / `gh` usage |

## Stack at a glance

| Concern | Choice |
|---|---|
| Language / runtime | Elixir 1.19 on Erlang/OTP 28 |
| Web framework | Phoenix 1.8 (JSON API only) |
| Database | PostgreSQL 17 + Ecto (via Docker Compose) |
| Auto-release | Hybrid: GenServer with per-token timers + periodic sweep, DB as source of truth |
| API docs | `open_api_spex` (OpenAPI 3 + Swagger UI) |
| Quality | `mix format`, `credo`, `dialyxir`, `sobelow`, `mix_audit` |
| Tests | ExUnit, Ecto SQL Sandbox, `ex_machina`, `excoveralls` |
| Observability | `:telemetry` events + Phoenix LiveDashboard |
| CI | GitHub Actions |
