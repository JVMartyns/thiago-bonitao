# 04 — Testing Strategy

> Challenge criterion: **Testes** — adequate unit **and** integration coverage.
> The interview includes *running the tests live*, so the suite must be fast, deterministic, and readable.

---

## 1. The testing pyramid for this project

```
            ▲  few   ┌────────────────────────────────────┐
            │        │ 4. Concurrency / stress            │  300 parallel activations
            │        ├────────────────────────────────────┤
            │        │ 3. API integration (ConnCase)      │  HTTP → controller → context → DB
            │        ├────────────────────────────────────┤
            │        │ 2. Core integration (DataCase)     │  context + Manager + real Postgres
            │        ├────────────────────────────────────┤
            ▼  many  │ 1. Unit (pure functions)           │  no DB, no processes, async
                     └────────────────────────────────────┘
```

| Layer | Tests | DB? | `async` | Speed |
|---|---|---|---|---|
| 1. Unit | changesets, expiry math, victim selection, JSON rendering | No | `true` | µs |
| 2. Core integration | `Tokens` context, `Manager` GenServer, queries, seeds | Sandbox | mostly `false` (Manager is shared) | ms |
| 3. API integration | each endpoint: status, body shape, errors, OpenAPI conformance | Sandbox | `false` | ms |
| 4. Concurrency | invariants under parallel load | Sandbox (shared) | `false` | < 1 s |

## 2. Libraries

| Library | Role |
|---|---|
| **ExUnit** | Test runner, `describe`/`setup`, tags, `start_supervised!/1` |
| **Ecto SQL Sandbox** | Each test runs in a transaction that is rolled back → isolated, no cleanup |
| **ex_machina** | Factories: `insert(:token)`, `insert(:active_token)`, `insert(:token_usage)` |
| **excoveralls** | Coverage report (HTML locally, threshold in CI) |
| **OpenApiSpex.TestAssertions** | `assert_schema/3` → responses match the documented contract |
| *(optional)* **StreamData** | Property tests: "for any sequence of operations, invariants hold" |

## 3. Making time testable (no `Process.sleep(120_000)`)

Three techniques, used together:

1. **Pure functions receive `now`** → `TokenUsage.expired?(usage, now, ttl_ms)` is tested with fixed `DateTime`s.
2. **Config-driven TTL** → in `config/test.exs`, `ttl_ms: 200`, `sweep_interval_ms: 50` (doc 01 §9).
3. **Drive the GenServer with messages** → instead of waiting for a timer, send it directly:
   `send(manager, {:expire, token_id, usage_id})` then make a synchronous call to flush the mailbox.
   For the few real-timer tests, use `assert_receive`/polling helpers with a short timeout, never fixed sleeps.

## 4. Handling the Manager + Sandbox

The Manager is a separate process that talks to the DB, so it must be allowed into the test's sandbox:

```elixir
# test/support/data_case.ex (excerpt)
setup tags do
  pid = Ecto.Adapters.SQL.Sandbox.start_owner!(TokenSystem.Repo, shared: not tags[:async])
  on_exit(fn -> Ecto.Adapters.SQL.Sandbox.stop_owner(pid) end)
  :ok
end
```

- Manager tests use `async: false` (shared sandbox mode) and start an **isolated Manager per test**:
  `start_supervised!({Manager, name: :"manager_#{System.unique_integer()}", ttl_ms: 100})`.
- The application-wide Manager is **not started in the test env** (config flag), so tests control it fully.
- API tests start one Manager in `setup` that the endpoint uses.

## 5. Test catalog (what we will write)

### 1. Unit
- [ ] `Token.changeset/2`: valid statuses only; active requires `current_user_id` + `activated_at`.
- [ ] `TokenUsage.changeset/2`: requires `token_id`, `user_id`; `release_reason` in allowed set.
- [ ] `expired?/3`: before, exactly at, and after TTL.
- [ ] `remaining_ms/3`: used by boot recovery; never negative.
- [ ] `TokenJSON`: renders `expires_at = activated_at + ttl`.

### 2. Core integration
- [ ] Seeds create exactly 100 tokens; running seeds twice still yields 100 (**R1**).
- [ ] `activate/1` returns an available token and opens a usage (**R6**).
- [ ] LRU selection: the least recently released token is chosen (**A6**).
- [ ] With 100 active, `activate/1` evicts the **oldest** and closes it with `evicted` (**R5**).
- [ ] Timer expiry releases the token with `expired` (**R4**).
- [ ] **Stale timer is ignored** after evict + re-activate (doc 01 §6.2).
- [ ] Sweep releases expired rows the timers missed (**A7**).
- [ ] Boot recovery: pre-insert expired + active rows, start Manager → expired released, timers re-armed (**A8**).
- [ ] `clear_active/0` releases all, returns count, leaves history intact (**A2**).
- [ ] History lists all past users of a token, newest first (**R7**).
- [ ] DB constraint: inserting a second open usage for a token fails (partial unique index).
- [ ] Manager crash → supervisor restarts it → state still consistent.
- [ ] Token count never changes after any operation (**R2**).

### 3. API integration
| Endpoint | Cases |
|---|---|
| `POST /api/tokens/activate` | 201 + body; 422 missing `user_id`; 422 non-UUID `user_id`; eviction path returns 201 |
| `GET /api/tokens` | 200 lists 100; `?status=active` / `available` filters; invalid filter → 422 |
| `GET /api/tokens/:id` | 200 with `active_user`; 200 with `active_user: null`; 404 unknown; 404 malformed id |
| `GET /api/tokens/:id/usages` | 200 ordered newest first; empty list for never-used token; 404 |
| `DELETE /api/tokens/active` | 200 with `released_count`; idempotent (second call → 0) |
| All | Response matches OpenAPI schema (`assert_schema`) |

### 4. Concurrency
- [ ] 300 concurrent `activate/1` calls (`Task.async_stream`, `max_concurrency: 300`):
      all succeed, active count == 100, **no token has two open usages**, total tokens == 100.
- [ ] Concurrent `activate` + `clear_active` + sweep: invariants still hold.

## 6. Organization & commands

```
test/
├── support/
│   ├── data_case.ex      # DB tests
│   ├── conn_case.ex      # HTTP tests
│   └── factory.ex        # ExMachina
├── token_system/tokens/
│   ├── token_test.exs           # unit
│   ├── token_usage_test.exs     # unit
│   ├── tokens_test.exs          # context integration
│   ├── manager_test.exs         # GenServer integration
│   └── concurrency_test.exs     # @moduletag :concurrency
└── token_system_web/controllers/
    └── token_controller_test.exs
```

| Command | Purpose |
|---|---|
| `mix test` | Everything |
| `mix test --exclude concurrency` | Fast inner loop |
| `mix test --only concurrency` | Stress tests only |
| `mix test --failed` | Re-run last failures |
| `mix coveralls.html` | Coverage report → `cover/excoveralls.html` |
| `mix test --slowest 10` | Find slow tests |

### Coverage setup (Elixir 1.19 style)
```elixir
# mix.exs
def project do
  [..., test_coverage: [tool: ExCoveralls]]
end

def cli do
  [preferred_envs: ["coveralls": :test, "coveralls.html": :test, "coveralls.github": :test]]
end
```
```json
// coveralls.json
{
  "coverage_options": { "minimum_coverage": 90 },
  "skip_files": ["test/support", "lib/token_system_web/telemetry.ex", "lib/token_system/release.ex"]
}
```

**Target: ≥ 90% overall, 100% on `Tokens` and `Manager`.** Coverage is a floor, not the goal —
the catalog in §5 (every rule R1–R7, A1–A8 has a test) is the real measure.

## 7. Test-writing rules

- One behavior per test; the test name states the rule: `test "evicts the oldest active token when all 100 are active"`.
- Arrange / Act / Assert, separated by blank lines.
- Factories over fixtures; build only what the test needs.
- No `Process.sleep` with fixed durations; use `assert_receive` or a polling helper with a timeout.
- Write the failing test first when implementing a rule (TDD) — it doubles as the spec.
