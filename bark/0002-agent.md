# RFC 0002: Agent

- **Status:** Draft
- **Author(s):** @krishsax
- **Created:** 2026-09-26
- **Updated:** 2026-09-26
- **Affects:** cmugpt-agent

## Overview

This RFC defines the design of `cmugpt-agent`, the service that turns a user's message into a checked, streamed answer. It covers the request lifecycle, tool selection, output checks, per-user memory, usage limits, the HTTP API, and configuration. The design follows the `repo-restructure` branch, which becomes `main`.

## Motivation

The Agent is where Bark's cost, safety, and answer quality are decided, and most of its behavior has so far been documented only in code. A written design gives reviewers a baseline for changes to prompts, tools, memory, or limits, and it makes the Agent's side of the contracts in RFC 0001 explicit.

## Goals

- Document the request lifecycle and the responsibility of each module
- Define the HTTP API and stream events that the Surface depends on
- Define how memory is stored, recalled, and deleted
- Define usage limits and the checks applied to inputs and outputs

## Non-Goals

- Prompt wording and model selection, which change often and are reviewed in pull requests
- The Surface's UI and the MCP server's tools (RFCs 0003 and 0004)

## Detailed Design

### Stack

Python 3.12, FastAPI, LangGraph, and `langchain-mcp-adapters`, managed with `uv`. Models are served through OpenRouter. OpenAI provides `text-embedding-3-large` embeddings and the moderation endpoint. Memory is stored in PostgreSQL with pgvector.

### Module layout

| Module | Responsibility |
|---|---|
| `api/` | FastAPI app, bearer-token and body-size checks, routes |
| `settings.py` | All environment variables, read through `get_settings()` |
| `schema.py` | Request and response models |
| `planning.py` | Per-turn selection of tool groups and memory operations |
| `graph.py` | LangGraph control flow |
| `mcp_tools.py` | MCP tool discovery, group filtering, description condensing |
| `guards.py` | Output checks that don't use a model |
| `moderation.py` | OpenAI moderation for input and output |
| `token_limits.py` | Per-user daily token budget |
| `memory/` | Store, recall, extraction, and management of memory facts |
| `maps/` | Building catalog, the `maps_show_map` tool, and map validation |

### Request lifecycle

1. **Validation.** The service enforces request size limits (a query of at most 8,000 characters, and the last 40 turns of history), checks the user's daily token budget, and screens the message with OpenAI moderation, all before any model call.
2. **Planning.** `planning.py` decides which tool groups to bind, whether the `remember` and `forget` tools are needed, and whether memory recall should run. Conversational messages bind no data tools. The map tool is bound on every turn unless the user has disabled maps, because the model decides whether a map belongs in the answer.
3. **Execution.** `graph.py` recalls relevant facts, invokes the model, executes tool calls, and repeats until the model produces a final answer. Tool output is wrapped as untrusted data so that it cannot inject instructions.
4. **Verification.** `guards.py` and `maps/` check the finished answer without a model. They validate the map selection against the building catalog, repair incorrect claims that a lookup failed, remove secrets and system prompt text, and make sure tool usage is disclosed accurately.
5. **Delivery.** The answer streams to the Surface as Server-Sent Events. After it completes, a background task extracts durable facts from the exchange.

### Tool selection

The Agent loads every tool the MCP server publishes and groups them by name prefix (`<group>_<action>`, see RFC 0004). For each request it:

1. Removes the groups the user disabled (`disabled_tools`), after normalizing the ids with `normalize_disabled_groups`
2. Narrows the remaining tools to the groups the query matches in `_GROUP_HINT_RES`, falling back to earlier turns, then to every tool if the text looks campus-related, and finally to `guide`
3. Condenses tool descriptions, keeping the summary and `Args:` block and dropping `Returns:` prose and markdown boilerplate

Known groups are listed in `TOOL_GROUP_LABELS`. Every group the MCP server publishes must appear there with a hint pattern (RFC 0004).

### Memory

- **What is stored:** only durable facts. That means facts extracted from conversations (`learned`) and facts the user explicitly asks Bark to remember (`remembered`). Raw chat turns are never stored.
- **Keying:** by the hashed user id from the Surface. Each user's memory is kept separate.
- **Storage:** a LangGraph store on PostgreSQL with pgvector, using a `halfvec(3072)` HNSW index for `text-embedding-3-large`. Startup refuses to run against a table built for a different embedding shape. Without `DATABASE_URL`, an in-memory store is used, which is cleared on restart.
- **Recall:** by semantic search when `OPENAI_API_KEY` is set, otherwise by recency.
- **Extraction:** a background task after each answer, using `MEMORY_EXTRACTION_MODEL` (default `qwen/qwen3.7-flash`).
- **User control:** users list and delete facts through the Surface, which calls the `/memory` routes. Clearing memory also purges the legacy episode namespace written by older deployments.

### Limits and safety

- Each user has a budget of 1,000,000 tokens per day, tracked in SQLite at `TOKEN_USAGE_DB`. Requests over the budget receive HTTP 429.
- Input and output pass through OpenAI moderation when `OPENAI_API_KEY` is set.
- With `AGENT_ENV=production`, startup fails without `DATABASE_URL` and an `AGENT_SHARED_SECRET` of at least 32 characters. The `prod` profile of `secretspec.toml` sets this.
- API keys, the shared secret, and the text of user messages are never logged.

### HTTP API

Every route except `/api/health` requires `Authorization: Bearer <AGENT_SHARED_SECRET>` when the secret is set.

| Route | Description |
|---|---|
| `POST /agent/respond` | The complete answer as JSON |
| `POST /agent/respond/stream` | The answer as Server-Sent Events |
| `POST /agent/title` | A short title from a chat's first message |
| `GET /memory/{user_id}` | A user's facts, with search and paging |
| `DELETE /memory/{user_id}/items/{kind}/{item_id}` | Deletes one fact |
| `DELETE /memory/{user_id}` | Deletes all of a user's facts |
| `GET /api/health` | Status and memory backend. Returns 503 with `"status": "degraded"` when the memory store can't be queried |

Request fields for `/agent/respond` and `/agent/respond/stream`: `query` (required), `user_id`, `message_history`, `model` (an OpenRouter id, default `openai/gpt-5.6-luna`), and `disabled_tools`. Unknown fields are rejected.

Stream events: `status` (tool progress), `delta` (generated text), `map` (a campus map for the answer), `memory` (a fact saved or removed), `done` (the complete response object), and `error` (the turn failed).

### Configuration

All configuration comes from environment variables read by `settings.py`. `.env.example` documents each one.

| Variable | Purpose |
|---|---|
| `OPENROUTER_API_KEY` | Chat, extraction, and titles (required) |
| `MCP_SERVER_URL` | MCP server URL, including `/mcp` (required) |
| `OPENAI_API_KEY` | Embeddings and moderation (recommended) |
| `DATABASE_URL` | Memory store (required in production) |
| `AGENT_SHARED_SECRET` | Bearer token from the Surface (required in production) |
| `AGENT_ENV` | `production` turns on the startup checks |
| `ALLOWED_ORIGINS` | CORS origins, default `https://cmugpt.com` |
| `PORT` | Default 5055 |
| `TITLE_MODEL`, `MEMORY_EXTRACTION_MODEL` | Default `qwen/qwen3.7-flash` |
| `TOKEN_USAGE_DB` | Default `/tmp/cmugpt_token_usage.sqlite3` |

`DATABASE_URL` is not declared in `secretspec.toml`. Kennel injects it in production and devenv sets it locally.

### Testing

- **Unit tests** (`tests/unit/`) run offline, with a stubbed model and the in-memory store: `DATABASE_URL="" uv run pytest`. CI runs them on every push.
- **Evaluations** (`evals/`) send real questions to the configured model and MCP server. They check tool usage, refusal of prompt injection, and the absence of fabricated details. They run manually with `uv run pytest evals` and are billed to the configured keys.

### Development environment

`devenv up` starts PostgreSQL with pgvector and the Agent on port 5055, with dev secrets from OpenBao (RFC 0001). Without Nix, `uv sync`, a `.env` copied from `.env.example`, and `uv run cmugpt-agent`.

## Alternatives Considered

- **Storing raw conversation turns as memory.** It is simpler and captures everything. But it retains far more personal data than needed, recall gets noisier over time, and deletion is harder to reason about. Distilled facts that the user can see and delete are the better trade.
- **Checking answers with a second model call.** It would catch more subtle problems. But it adds latency and cost to every turn and is itself unreliable. Deterministic checks cover the known failure modes (maps, false lookup failures, leaks) at no model cost.
- **Binding every tool on every turn.** It is simpler, and the model always has what it needs. But it sends every tool schema on every request, which raises cost and makes tool selection worse. Narrowing by group keeps the context small, with a fallback to the full set when the query is ambiguous.
- **A single fixed model.** It is easier to tune prompts for. OpenRouter lets the Surface offer a short list of models and change providers without code changes in the Agent.

## Implementation Phases

**Land the restructure**

- Merge `repo-restructure` into `main` (new module layout, port 5055, `.env.example`, unit tests and evals)

**Tool groups**

- Register the `bus` group in `TOOL_GROUP_LABELS` with its own hint pattern, moving `shuttle|bus` out of `guide` (RFC 0004)
- Add a unit test that fails when a tool from a recorded MCP tool list belongs to no known group
