# gegentic quickstart

Runs the public, educational `gegentic/*` Docker Hub images together with `docker compose` — `gegentic/api`, `gegentic/api-migration`, `gegentic/gateway`, `gegentic/guardian`, `gegentic/portal`, `gegentic/tracing` — plus the infrastructure they depend on (Postgres, SuperTokens, ClickHouse).

## Setup

```bash
cp .env.example .env
# edit .env: LICENSE_KEY, ENCRYPT_KEY_FOR_API_KEY, ENCRYPT_KEY_FOR_VIRTUAL_KEY, OPENAI_API_KEY
docker compose up
```

- **`LICENSE_KEY`** — `gegentic/api` runs in self-hosted mode and requires one. Register at [gegentic.ai](https://gegentic.ai) to get one by email.
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

## Known gaps
- **Portal's runtime config is intentionally limited to 3 vars, and depends on a CI change not yet built/published.** Only `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_WWW_URL`, and `NEXT_PUBLIC_WEBSITE_URL` are runtime-configurable on `gegentic/portal`. Everything else is fixed for the gegentic brand at that image's build time and can't be overridden here: `NEXT_PUBLIC_APP_NAME`/`NEXT_PUBLIC_BRAND` = `Gegentic`, `NEXT_PUBLIC_COPYRIGHT` = `Gegentic Sdn Bhd`, plus a fixed PostHog key/host, license plan ID, experimental-features flag, and Stripe publishable key (see `obiguard-portal-service`'s `Dockerfile.gegentic`). The mechanism: a dedicated `Dockerfile.gegentic` (separate from the production `Dockerfile`) builds the 3 configurable vars as sentinel placeholder tokens, and `docker-entrypoint.gegentic.sh` substitutes them with real env vars at container start. **This only takes effect once that repo's CI has rebuilt and republished `gegentic/portal:latest` using the new `Dockerfile.gegentic`** (its `.gitlab-ci.yml` was updated to use `--file Dockerfile.gegentic`) — until then, `gegentic/portal:latest` is still the old bake-at-build-time image. The substitution logic itself was verified directly (ran the actual entrypoint script in a real Alpine container against fake `.next` build output, confirmed correct substitution for the 3 configurable vars and confirmed the fixed brand/PostHog values are left untouched) — what's *not* verified is the full real Next.js build + a real browser hitting the resulting `portal` container end-to-end.
- **Env var wiring verified against each service's own `.env.example`** (`API_SERVICE_HOST`, `GUARDRAILS_SERVICE_HOST`, `LLM_GATEWAY_SERVICE_HOST`, etc.) at the time this was written, but **not yet tested end-to-end as a running stack** — if a service fails to start, check its logs first; the most likely culprits are a missing required env var or a Postgres/SuperTokens connection race despite the `healthcheck`-gated `depends_on`.

## Updating

This stack always pulls `:latest` for each `gegentic/*` image. Run `docker compose pull && docker compose up -d` to update.
