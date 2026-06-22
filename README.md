# gegentic quickstart

Runs the public, educational `gegentic/*` Docker Hub images together with `docker compose` — `gegentic/api`, `gegentic/api-migration`, `gegentic/gateway`, `gegentic/guardian` — plus the infrastructure they depend on (Postgres, SuperTokens, Qdrant).

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
| postgres | 5432 (internal only) | `postgres:16-alpine` |
| supertokens | 3567 (internal only) | `registry.supertokens.io/supertokens/supertokens-postgresql:9.3` |
| qdrant | 6333 (internal only) | `qdrant/qdrant:latest` |

`api-migration` runs `prisma migrate deploy` against Postgres and exits; `api` waits for it to complete successfully (`depends_on: condition: service_completed_successfully`) before starting.

## Known gaps

- **No tracing backend.** There's no public `gegentic` image for `obiguard-tracing-service` yet. Both `api` and `guardian` handle this gracefully — failed OTEL span exports are logged, not fatal — but you won't get observability/traces out of the box.
- **No portal UI.** There's no public `gegentic` image for `obiguard-portal-service` either, so this stack is API/gateway/guardian only — no web UI for managing orgs/policies. Interact via the REST APIs directly (see each service's CLAUDE.md / API docs).
- **Env var wiring verified against each service's own `.env.example`** (`API_SERVICE_HOST`, `GUARDRAILS_SERVICE_HOST`, `LLM_GATEWAY_SERVICE_HOST`, `QDRANT_URL`, etc.) at the time this was written, but **not yet tested end-to-end as a running stack** — if a service fails to start, check its logs first; the most likely culprits are a missing required env var or a Postgres/SuperTokens connection race despite the `healthcheck`-gated `depends_on`.

## Updating

This stack always pulls `:latest` for each `gegentic/*` image. Run `docker compose pull && docker compose up -d` to update.
