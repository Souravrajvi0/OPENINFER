# OpenInference

OpenInference is a self-hosted gateway for LLM applications. It gives teams one controlled API in front of hosted and local models, with routing, guardrails, retrieval, budgets, tracing, evaluations, and an admin console.

This repository also includes `oi`, a hardware-aware CLI for finding, installing, and running local open-source models. See [packages/cli/README.md](./packages/cli/README.md) for the CLI guide.

## What It Does

Applications call OpenInference instead of calling model providers directly. The gateway then handles policy, routing, model execution, observability, and persistence before returning a response.

```text
Client or SDK
  -> Gateway API (Fastify)
      -> auth, scopes, rate limits
      -> guardrails and PII handling
      -> budget checks
      -> model routing, A/B tests, fallback
      -> optional RAG retrieval
      -> hosted provider or self-hosted Ollama
      -> traces, audit logs, metrics
      -> async evaluation jobs
  -> Response
```

The production site is [openinference.tech](https://openinference.tech). Local API docs are served at `http://localhost:3000/api-docs`.
Judges: start with Quick Start, First API Call, and Project Layout for the fastest review path.
This branch is a documentation-focused snapshot; it does not change runtime behavior.

## Main Features

| Area | Capabilities |
| --- | --- |
| Unified model API | Native `/v1/chat` plus OpenAI-compatible `/v1/chat/completions` and `/v1/models` |
| Providers | OpenAI, Anthropic, Groq, Mistral, Cerebras, Gemini, and self-hosted Ollama |
| Routing | Provider/model pinning, default routing, fallback models, model catalog, plan tiers |
| Safety | Prompt-injection checks, PII redaction, configurable guardrail policies |
| Cost control | Tenant budgets, API-key budgets, platform budget, request accounting |
| Retrieval | Document upload, async chunking and embedding, hybrid vector plus keyword search |
| Agents | Tool-running agent endpoint, agent registry, tool policies, human approvals, MCP server governance |
| Observability | Request records, trace spans, audit logs, Prometheus metrics, Grafana dashboard |
| Evaluation | Async response scoring through BullMQ eval workers |
| Admin console | React dashboard for keys, providers, models, docs, traces, sessions, budgets, agents, guardrails, documents, and regression tests |

## Stack

| Layer | Technology |
| --- | --- |
| API | Node.js 20, TypeScript, Fastify |
| Web | React 19, Vite, TanStack Router |
| Data | PostgreSQL 16 with pgvector |
| Queues/cache | Redis 7, BullMQ |
| Local inference | Ollama |
| Observability | Prometheus, Grafana, structured request logs |
| Packaging | npm workspaces, Docker Compose |

## Quick Start

### Prerequisites

- Docker and Docker Compose
- Node.js 20+ if you want to run services outside Docker
- At least one LLM provider key for the default route. The sample env uses Groq by default.

### Run the full stack

```bash
git clone https://github.com/Souravrajvi0/OPENINFER.git
cd OPENINFER
cp .env.example .env
```

On Windows PowerShell:

```powershell
Copy-Item .env.example .env
```

Edit `.env` and set at minimum:

```bash
JWT_SECRET=replace-with-a-long-random-secret
GROQ_API_KEY=gsk_...
DEFAULT_PROVIDER=groq
DEFAULT_MODEL=llama-3.3-70b-versatile
```

Then start the stack:

```bash
docker compose up --build
```

Useful local URLs:

| Service | URL |
| --- | --- |
| Web app and gateway | `http://localhost:3000` |
| Swagger/OpenAPI docs | `http://localhost:3000/api-docs` |
| Health check | `http://localhost:3000/health` |
| Prometheus | `http://localhost:9090` |
| Grafana | `http://localhost:3001` |

Grafana uses password `sentinelai` in the local compose file.

The database seed creates a development tenant and API key:

```text
sentinel-dev-key
```

Use it only for local development.

## First API Call

Native gateway API:

```bash
curl -X POST http://localhost:3000/v1/chat \
  -H "X-Api-Key: sentinel-dev-key" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      { "role": "user", "content": "Explain RAG in one sentence." }
    ]
  }'
```

OpenAI-compatible API:

```bash
curl -X POST http://localhost:3000/v1/chat/completions \
  -H "Authorization: Bearer sentinel-dev-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-3.3-70b-versatile",
    "messages": [
      { "role": "user", "content": "Write a haiku about observability." }
    ]
  }'
```

List the models available to the current key:

```bash
curl http://localhost:3000/v1/models \
  -H "Authorization: Bearer sentinel-dev-key"
```

## Core Endpoints

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `POST` | `/v1/chat` | Native chat API with routing, sessions, RAG, guardrails, budgets, tracing, and eval jobs |
| `GET` | `/v1/models` | OpenAI-shaped model list filtered by provider config, tenant plan, and key allowlist |
| `POST` | `/v1/chat/completions` | OpenAI Chat Completions-compatible endpoint |
| `POST` | `/v1/retrieve` | Search indexed documents |
| `POST` | `/v1/documents` | Ingest text content for retrieval |
| `POST` | `/v1/documents/upload` | Upload `.txt`, `.md`, or `.pdf` documents up to 10 MB |
| `POST` | `/v1/agent` | Run an agent workflow with tool access |
| `GET` | `/v1/traces/:traceId` | Inspect trace spans for a request |
| `GET` | `/v1/sessions/:sessionId` | Read stored conversation memory |
| `GET` | `/metrics` | Prometheus metrics |

Admin routes live under `/v1/admin/*` and require an API key or user session with admin scope.

## Native Chat Request

```json
{
  "messages": [
    { "role": "user", "content": "Explain semantic caching." }
  ],
  "provider": "groq",
  "model": "llama-3.3-70b-versatile",
  "stream": false,
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "rag": {
    "enabled": true,
    "top_k": 5
  },
  "metadata": {
    "app": "support-bot"
  }
}
```

Typical response:

```json
{
  "id": "request-id",
  "trace_id": "trace-id",
  "content": "Semantic caching stores and reuses responses for meaningfully similar prompts.",
  "model": "llama-3.3-70b-versatile",
  "provider": "groq",
  "usage": {
    "prompt_tokens": 12,
    "completion_tokens": 18,
    "total_tokens": 30,
    "cost_usd": 0.00001
  },
  "latency_ms": 423
}
```

## Retrieval Workflow

Add a text document:

```bash
curl -X POST http://localhost:3000/v1/documents \
  -H "X-Api-Key: sentinel-dev-key" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Refund Policy",
    "content": "Refunds are available within 30 days of purchase."
  }'
```

Upload a file:

```bash
curl -X POST http://localhost:3000/v1/documents/upload \
  -H "X-Api-Key: sentinel-dev-key" \
  -F "file=@policy.pdf" \
  -F "title=Refund Policy"
```

Search indexed content:

```bash
curl -X POST http://localhost:3000/v1/retrieve \
  -H "X-Api-Key: sentinel-dev-key" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "How do refunds work?",
    "top_k": 5,
    "hybrid": true
  }'
```

Use retrieval during chat:

```bash
curl -X POST http://localhost:3000/v1/chat \
  -H "X-Api-Key: sentinel-dev-key" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      { "role": "user", "content": "What is our refund window?" }
    ],
    "rag": { "enabled": true, "top_k": 5 }
  }'
```

RAG embeddings use Mistral by default, so set `MISTRAL_API_KEY` when using document retrieval.

## Local Development

Install root workspace dependencies:

```bash
npm install
```

Run infrastructure only:

```bash
docker compose up -d postgres redis
```

Run the gateway:

```bash
npm run dev -w @sentinelai/gateway
```

Run workers in separate terminals:

```bash
npm run dev -w @sentinelai/ingestion-worker
npm run dev -w @sentinelai/eval-worker
```

Run the web app in dev mode:

```bash
cd web
npm install
npm run dev
```

The Vite dev server proxies `/v1`, `/health`, and `/api-docs` to `http://localhost:3000` by default. Set `API_TARGET` if your gateway runs elsewhere.

Common checks:

```bash
npm run build
npm run typecheck
npm run lint
npm test -w @sentinelai/gateway
```

## Environment

The main env file is `.env`, usually copied from [.env.example](./.env.example).

Required for local gateway startup:

```bash
JWT_SECRET=replace-with-a-long-random-secret
DATABASE_URL=postgresql://sentinel:sentinel@localhost:5432/openinference
REDIS_URL=redis://localhost:6379
```

Configure at least one model provider:

```bash
GROQ_API_KEY=gsk_...
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
MISTRAL_API_KEY=...
CEREBRAS_API_KEY=csk-...
GEMINI_API_KEY=...
OLLAMA_URL=http://ollama:11434
```

Routing defaults:

```bash
DEFAULT_PROVIDER=groq
DEFAULT_MODEL=llama-3.3-70b-versatile
FALLBACK_PROVIDER=groq
FALLBACK_MODEL=llama-3.1-8b-instant
```

## Project Layout

```text
.
|-- services/
|   |-- gateway/            Fastify API, auth, routing, guardrails, admin routes
|   |-- ingestion-worker/   BullMQ worker for document chunking and embeddings
|   `-- eval-worker/        BullMQ worker for response quality evaluation
|-- web/                    React admin console, playground, docs, and marketing pages
|-- shared/                 Shared TypeScript types and queue constants
|-- packages/
|   `-- cli/                `oi` local model CLI
|-- infra/
|   |-- postgres/           Schema and seed data
|   |-- redis/              Redis startup wrapper
|   |-- prometheus/         Metrics config and alerts
|   |-- grafana/            Dashboard provisioning
|   `-- nginx/              Reverse proxy config
|-- docker-compose.yml      Full local/production-style stack
`-- package.json            Root npm workspace for shared, services, and packages
```

## Deployment

The Docker Compose stack includes:

- `gateway` on port `3000`
- `postgres` with pgvector
- `redis`
- `ollama`
- `ingestion-worker`
- `eval-worker`
- `prometheus`
- `grafana`
- `nginx`

The gateway image builds the React app and serves the compiled SPA from the Fastify process. SQL migrations in `services/gateway/migrations` run before the gateway starts.

GitHub Actions deployment is configured for the `production` branch. Required repository secrets are:

```text
DROPLET_IP
DROPLET_USER
SSH_PRIVATE_KEY
```

Expected branch flow:

```text
dev -> main -> production
```

Only `production` deploys.

## Security Notes

- Do not use `sentinel-dev-key` outside local development.
- API keys are stored as SHA-256 hashes; raw keys are returned only at creation time.
- Use a long random `JWT_SECRET` in every deployed environment.
- Keep provider keys in `.env` or your deployment secret store, not in source control.
- Admin endpoints require `admin` scope or an authenticated admin session.

## CLI

The CLI scans local hardware, recommends models that fit, installs them through Ollama, and launches local chat. Full documentation is in [packages/cli/README.md](./packages/cli/README.md).
