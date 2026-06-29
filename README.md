# gegentic quickstart

Runs the public, educational `gegentic/*` Docker Hub images together with `docker compose` — `gegentic/api`, `gegentic/api-migration`, `gegentic/gateway`, `gegentic/guardian`, `gegentic/portal`, `gegentic/tracing` — plus the infrastructure they depend on (Postgres, SuperTokens, ClickHouse).

## Setup

```bash
cp .env.example .env
# edit .env: LICENSE_KEY, ENCRYPT_KEY_FOR_API_KEY, ENCRYPT_KEY_FOR_VIRTUAL_KEY, OPENAI_API_KEY
docker compose up
```

- **`LICENSE_KEY`** — `gegentic/api` runs in self-hosted mode and requires one. Register at [gegentic.com/gegentic/edu/register](https://gegentic.com/gegentic/edu/register) to get one by email. See [docs.gegentic.com](https://docs.gegentic.com) for more.
- **`ENCRYPT_KEY_FOR_API_KEY`** / **`ENCRYPT_KEY_FOR_VIRTUAL_KEY`** — generate each with `openssl rand -hex 32`.
- **`OPENAI_API_KEY`** — needed for guardian's AI-based guardrails (jailbreak detection, compliance/LLM-judge checks). Any OpenAI-compatible endpoint works via `OPENAI_BASE_URL`.

## What's running

| Service | Port | Image |
|---|---|---|
| api | 4001 (REST), 4002 (gRPC) | `gegentic/api:latest` |
| api-migration | — (runs once, exits) | `gegentic/api-migration:latest` |
| gateway | 8787 | `gegentic/gateway:latest` |
| guardian | 8700 | `gegentic/guardian:latest` |
| tracing | 5001 | `gegentic/tracing:latest` |
| clickhouse | 8123 (internal only) | `clickhouse/clickhouse-server:24.5.1.1763-alpine` |
| postgres | 5432 (internal only) | `postgres:16-alpine` |
| supertokens | 3567 (internal only) | `registry.supertokens.io/supertokens/supertokens-postgresql:9.3` |
| portal | 3002 | `gegentic/portal:latest` |

`api-migration` runs `prisma migrate deploy` against Postgres and exits; `api` waits for it to complete successfully (`depends_on: condition: service_completed_successfully`) before starting. `tracing` runs its own Prisma migrations and ClickHouse schema setup idempotently on every container start (see its `entrypoint.sh`) — no separate migration job is needed for it; `api`/`guardian` only need to wait for it to start, not complete.

`tracing` gets its own Postgres database (`gegentic_tracing`, created via `POSTGRES_MULTIPLE_DATABASES`) and its own ClickHouse database (`gegentic_tracing`, created via `CLICKHOUSE_DB` on the `clickhouse` service and re-asserted idempotently by `tracing`'s entrypoint).

## Updating

This stack always pulls `:latest` for each `gegentic/*` image. Run `docker compose pull && docker compose up -d` to update.
