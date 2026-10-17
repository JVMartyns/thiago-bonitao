# thiago-bonitao

Token management API built with **Elixir + Phoenix + PostgreSQL/Ecto** — a learning project based on the
Just Travel "Gerenciamento de Tokens" technical challenge.

- 100 pre-generated UUID tokens, at most 100 active at once
- Tokens auto-release after 2 minutes
- When all are active, the oldest active token is released to serve a new request
- Full usage history per token

> 🚧 Work in progress — developed issue by issue. See the [project docs](docs/README.md)
> and [AGENTS.md](AGENTS.md) for how work is organized.

Setup, API usage and architecture sections will be written as the implementation lands.
