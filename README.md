# gegentic quickstart

Runs the public, educational `gegentic/*` Docker Hub images together with `docker compose` — `gegentic/api`, `gegentic/api-migration`, `gegentic/gateway`, `gegentic/guardian`, `gegentic/portal`, `gegentic/tracing` — plus the infrastructure they depend on (Postgres, SuperTokens, ClickHouse).

## Setup

```bash
cp .env.example .env
# edit .env: LICENSE_KEY, ENCRYPT_KEY_FOR_API_KEY, ENCRYPT_KEY_FOR_VIRTUAL_KEY, OPENAI_API_KEY
docker compose up
```

- **`LICENSE_KEY`** — `gegentic/api` runs in self-hosted mode and requires one. Register at [gegentic.com/gegentic/edu/register](https://gegentic.com/gegentic/edu/register) to get one by email. See [docs.gegentic.com](https://docs.gegentic.com) for more.
- **`ENCRYPT_KEY_FOR_API_KEY`** / **`ENCRYPT_KEY_FOR_VIRTUAL_KEY`** — generate each with `openssl rand -hex 32`. `ENCRYPT_KEY_FOR_VIRTUAL_KEY` encrypts the AI provider credentials stored in Gegentic (it keeps its historical name).
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

`api-migration` runs `prisma migrate deploy` against Postgres and exits; `api` waits for it to complete successfully (`depends_on: condition: service_completed_successfully`) before starting. `tracing` runs its ClickHouse schema setup idempotently on every container start (see its `entrypoint.sh`) — no separate migration job is needed for it; `api`/`guardian` only need to wait for it to start, not complete.

`tracing` uses ClickHouse only, with its own database (`gegentic_tracing`, created via `CLICKHOUSE_DB` on the `clickhouse` service and re-asserted idempotently by `tracing`'s entrypoint).

## Locking down the tracing service (optional)

The tracing service (port 5001) is published on the host. By default it accepts trace uploads and trace reads from anything that can reach that port, and it holds your prompts and responses. To restrict it to the stack's own services:

1. Generate a secret: `openssl rand -hex 32`
2. In `.env`, uncomment and fill in `INTERNAL_SERVICE_SHARED_SECRET`, and uncomment `TRACING_AUTH_MODE=enforce`.
3. `docker compose pull && docker compose up -d`. Pull first: images from before this feature don't send the secret, so enforcing would block their trace uploads.

The api, gateway and guardian send the secret; tracing answers `401` to calls without it. To find anything you missed before enforcing, set `TRACING_AUTH_MODE=report` instead, which logs calls that lack the secret but lets them through. A script that posts traces straight to port 5001 (for example with a trace SDK) will be refused once enforced; send them through the gateway's `/api/trace` with an access key instead.

## Updating

This stack always pulls `:latest` for each `gegentic/*` image. Run `docker compose pull && docker compose up -d` to update.
