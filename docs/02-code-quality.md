# 02 — Code Quality

> Challenge criterion: **Qualidade do Código** — clean, readable structure following good practices.
> Rule of thumb: *a tool enforces it, or it doesn't exist*. Every rule below is checked by CI (doc 05).

---

## 1. Project structure

Phoenix separates the **core** (`lib/token_system`) from the **web boundary** (`lib/token_system_web`).
The web layer may call the core; the core never knows about HTTP.

```
lib/
├── token_system/
│   ├── application.ex            # supervision tree
│   ├── repo.ex
│   ├── tokens.ex                 # context = public API of the domain
│   └── tokens/
│       ├── manager.ex            # GenServer: writes, timers, sweep, recovery
│       ├── token.ex              # schema + changesets
│       ├── token_usage.ex        # schema + changesets
│       └── queries.ex            # composable Ecto queries
└── token_system_web/
    ├── router.ex
    ├── api_spec.ex               # OpenApiSpex root spec
    ├── schemas/                  # OpenApiSpex request/response schemas
    └── controllers/
        ├── token_controller.ex
        ├── token_json.ex         # rendering (Phoenix 1.8 style, no views)
        ├── fallback_controller.ex
        └── error_json.ex
priv/repo/
├── migrations/
└── seeds.exs                     # inserts the 100 tokens (idempotent)
test/
├── support/{data_case,conn_case,factory}.ex
├── token_system/tokens/...       # unit + DB integration
└── token_system_web/controllers/ # API integration
```

## 2. Elixir conventions we follow

| Practice | Example / rule |
|---|---|
| **Contexts own the domain** | Controllers call `Tokens.activate/1`, never `Repo` or `Manager` directly. |
| **Tagged tuples** | Public functions return `{:ok, value}` / `{:error, reason}`. Bang (`!`) versions only when crashing is the right behavior. |
| **`with` for happy paths** | Chain steps in controllers/contexts; let `action_fallback` handle errors. |
| **Pattern matching over conditionals** | Multiple function heads instead of nested `if`/`case`. |
| **Pipelines** | Start with a raw value, one transformation per line; no pipes of length 1. |
| **Typespecs** | `@spec` on every public function; `@type t` on schemas. Checked by Dialyzer. |
| **Docs** | `@moduledoc` on every module, `@doc` on every public function (Credo checks this). |
| **No magic numbers** | TTL, pool size and sweep interval come from config (doc 01 §9). |
| **Pure core, impure shell** | Decisions like "is this usage expired?" are pure functions taking `now` as an argument → trivial to unit test. |
| **Let it crash — but at the right level** | Validate input at the boundary (changesets, OpenApiSpex); let unexpected errors crash the process and rely on the supervisor. |
| **Naming** | `snake_case` functions, `PascalCase` modules, predicate functions end in `?` (`expired?/2`). |

### Small example of the target style
```elixir
@spec activate(Ecto.UUID.t()) :: {:ok, TokenUsage.t()} | {:error, Ecto.Changeset.t()}
def activate(user_id) do
  with {:ok, user_id} <- cast_user_id(user_id) do
    Manager.activate(user_id)
  end
end
```

## 3. Error handling

- One error JSON shape for the whole API (doc 01 §8).
- `FallbackController` converts `{:error, :not_found}` → 404 and `{:error, %Ecto.Changeset{}}` → 422.
- The Manager never returns raw DB errors to the client; it logs them with context and returns a domain error.

## 4. Tooling

| Tool | What it catches | Command |
|---|---|---|
| `mix format` | Style inconsistency | `mix format --check-formatted` |
| Compiler | Unused vars, dead code, type warnings (Elixir 1.19 has gradual type checks built in) | `mix compile --warnings-as-errors` |
| **Credo** | Readability, complexity, consistency, missing docs | `mix credo --strict` |
| **Dialyxir** | Typespec / success-typing errors | `mix dialyzer` |
| **Sobelow** | Phoenix security issues | `mix sobelow --config` |
| **mix_audit** | Dependencies with known CVEs | `mix deps.audit` |
| `mix deps.unlock --check-unused` | Stale entries in `mix.lock` | — |

### Dependencies to add in `mix.exs`
```elixir
{:credo, "~> 1.7", only: [:dev, :test], runtime: false},
{:dialyxir, "~> 1.4", only: [:dev, :test], runtime: false},
{:sobelow, "~> 0.13", only: [:dev, :test], runtime: false},
{:mix_audit, "~> 2.1", only: [:dev, :test], runtime: false},
{:open_api_spex, "~> 3.21"},
{:ex_machina, "~> 2.8", only: :test},
{:excoveralls, "~> 0.18", only: :test}
```

### One command before every commit
Phoenix 1.8 already generates a `precommit` alias; we extend it:
```elixir
defp aliases do
  [
    # ...
    precommit: [
      "compile --warnings-as-errors",
      "deps.unlock --check-unused",
      "format --check-formatted",
      "credo --strict",
      "sobelow --config",
      "test"
    ]
  ]
end
```
Dialyzer is slower, so it runs in CI and on demand (`mix dialyzer`), not on every commit.

## 5. Observability (quality you can see in the demo)

- `:telemetry` events emitted by the Manager:
  `[:token_system, :token, :activated | :expired | :evicted | :cleared]` with `token_id`, `user_id`, `duration_ms`.
- Metrics registered in `TokenSystemWeb.Telemetry` → visible live in **LiveDashboard** (`/dev/dashboard`, dev only).
- Structured `Logger` metadata (`token_id`, `user_id`, `usage_id`) on every state change.

## 6. Definition of Done (per feature)

- [ ] Behavior matches a rule in doc 01 (or a new decision was added there).
- [ ] Unit + integration tests written (doc 04), coverage not lowered.
- [ ] `@doc`, `@spec` present; OpenAPI schema updated for API changes.
- [ ] `mix precommit` green.
- [ ] README updated if usage changed.
- [ ] Committed as small, conventional commits (doc 03).
