# RFC 0002: Agent

- **Status:** Draft
- **Author(s):** @qianxuege
- **Created:** 2026-10-03
- **Updated:** 2026-10-03
- **Affects:** openbark-agent

## Overview

This RFC defines the design of `openbark-agent`, the component that turns a member's message into a cited, verified, streamed answer, and that suspends mid-run when answering requires an approval. It covers the request lifecycle, the retrieval stage, tool selection, verification, memory, run state, the HTTP API, and model policy. It follows bark RFC 0002 closely and departs from it in four places: retrieval, suspendable runs, verification, and model routing.

## Motivation

The Agent is where OpenBark's answer quality, cost, and safety are decided, and it is the component a module author most wants to avoid touching. Two pressures shape it.

Answers have to be grounded. OpenBark's purpose is to answer from institutional material that members cannot easily search, and an assistant that paraphrases an RFC from memory is worse than no assistant, because it is confidently wrong about the thing the team relies on. Every factual claim about internal material needs a citation that resolves, and the answer needs to say when it does not know.

Answers can also cause things to happen. The Agent holds write tools, and the material it retrieves is written by people and by automated systems and can contain text addressed to the model. The combination of untrusted input and privileged tools is the dangerous one, and the Agent is where it is contained.

## Goals

- Define the request lifecycle and each module's responsibility
- Define the retrieval stage and how role-based filtering reaches it
- Define how a run suspends for approval and resumes, and where that state lives
- Define verification, in particular citation checking
- Define the HTTP API and stream events the Surface depends on
- Define model policy for private material

## Non-Goals

- Prompt wording, which changes often and is reviewed in pull requests
- The indexing and retrieval implementation behind the query interface (RFC 0005)
- The action contract and approval lifecycle, of which this RFC defines only the Agent's half (RFC 0006)
- The Surface's UI (RFC 0003) and tool authoring (RFC 0004)

## Detailed Design

### Stack

Python 3.12, FastAPI, LangGraph, and `langchain-mcp-adapters`, managed with `uv`. This matches `cmugpt-agent` so that the two share patterns and contributors. Run state and personal memory are stored in PostgreSQL with pgvector.

### Module layout

| Module | Responsibility |
|---|---|
| `api/` | FastAPI app, assertion verification, body-size checks, routes |
| `settings.py` | All environment variables, read through `get_settings()` |
| `schema.py` | Request, response, and assertion models |
| `identity.py` | Verifying the signed user assertion and exposing subject and roles |
| `planning.py` | Per-turn selection of retrieval, tool groups, and memory operations |
| `graph.py` | LangGraph control flow, including suspend and resume |
| `retrieval.py` | The knowledge query interface and result normalization |
| `mcp_tools.py` | Tool discovery across servers, group filtering, classification, description condensing |
| `actions.py` | Preview, approval checks, execution, idempotency, audit emission |
| `guards.py` | Verification that doesn't use a model, including citation checks |
| `memory/` | Personal memory store, recall, extraction, and management |
| `runs/` | Suspended run persistence |

### Request lifecycle

1. **Validation.** The service verifies the signed assertion, enforces request size limits (a query of at most 8,000 characters and the last 40 turns), and checks the member's daily token budget, before any model call.
2. **Planning.** `planning.py` decides whether to retrieve, which tool groups to bind, and whether memory operations are needed. A conversational message binds no data tools and skips retrieval. Anything that could be answered from institutional material retrieves.
3. **Retrieval.** `retrieval.py` queries the index with the member's role ids applied as a filter and attaches the results to the turn as untrusted, cited context.
4. **Execution.** `graph.py` recalls personal memory, invokes the model, executes tool calls, and repeats until the model produces a final answer or requests an action.
5. **Suspension, if an action is required.** `actions.py` calls the provider's `preview`, the Agent emits `action_preview`, and the run persists and stops. See Suspendable runs.
6. **Verification.** `guards.py` checks the finished answer without a model: every citation resolves, no claim about internal material is uncited, no action is described as done that was not executed, and no secret or system prompt text appears.
7. **Delivery.** The answer streams to the Surface as Server-Sent Events. After it completes, a background task extracts durable personal facts from the exchange.

Steps 1, 2, 4, 7 and the shape of the whole are Bark's. Steps 3, 5, and the content of 6 are new.

### Retrieval

Bark has no document retrieval; its only vector search is over personal memory facts. RFC 0005 defines the query contract and what a result carries. The Agent's obligations against it are:

- **Send role ids, never the subject.** The planner may add metadata filters it inferred, such as source, document type, or date range.
- **Do not post-filter.** The index applies the role filter inside the query, because a result the Agent has already seen has already entered the prompt.
- **Cite by the id the index returned.** Results carry provenance, and `guards.py` verifies that every cited id was one of them.
- **Surface staleness rather than hiding it.** When a result comes back flagged stale, the prompt says so, so the answer can.
- **Treat results as untrusted**, wrapped the way Bark wraps tool output. Instructions inside a retrieved document are data.

### Tool selection

The Agent loads tools from every configured MCP server and groups them by name prefix (`<group>_<action>`, see RFC 0004). Bark's narrowing logic carries over unchanged: drop the groups the member disabled, narrow to the groups the query matches by hint pattern with a fallback to the full set, and condense descriptions to the summary and `Args:` block.

Two additions:

- **Federation.** Bark builds a `MultiServerMCPClient` with one entry from `MCP_SERVER_URL`. The adapter library is already multi-server, so the Agent configures several: `openbark-knowledge`, `openbark-actions`, and Bark's public `mcp-server` for campus questions. RFC 0001 defines the trust half of federation, which servers are hard dependencies and where the assertion may be sent, and RFC 0004 the contract half; this is where both are implemented. In particular `get_tools()` returns a flat list with no namespacing by server, so the Agent validates at startup that no two configured servers publish the same group id or tool name, and refuses to start on a collision. A server marked foreign gets its classification from configuration rather than from itself, never receives the member's assertion, and is a soft dependency: when it is unreachable its groups are unavailable and `/api/health` reports degraded, but the Agent still answers.
- **Classification gates binding.** Every tool declares `read` or `write` and the permission it requires (RFC 0004). A write tool is bound only when the member's roles include its permission. A member without the permission gets an answer explaining what would be needed, not a failed tool call, and the model never sees a tool it cannot use.

A member's tool toggles are a preference and are applied before classification. They are not a security control; the permission check is.

### Suspendable runs

An action requires a human approval between preview and execution (RFC 0006), which can take minutes or days. The run cannot be held in memory.

- On `action_preview`, the Agent writes the run's state, the pending action, its arguments, and its idempotency key to `runs/`, then ends the stream with an `action_pending` event.
- The Surface stores the pending action and owns the approval UI and the record of who approved it.
- On approval, the Surface calls `POST /agent/runs/{run_id}/resume`. The Agent re-verifies that the approval is recorded, that the approver's permission is sufficient, and that the arguments match what was previewed, then executes with the stored idempotency key and continues the run.
- A pending action expires after a configurable window, default 72 hours. An expired run is not resumable and must be asked again, because the material it was based on may have changed.
- Re-executing a resumed run with the same idempotency key must not repeat the effect. That guarantee belongs to the provider (RFC 0006); the Agent's obligation is to reuse the key rather than mint a new one.

The arguments are checked against the preview on resume because the preview is what a human approved. If the retrieved context changed in between, the action does not silently change with it.

### Verification

Bark's guards validate map selection against a building catalog, repair false claims that a lookup failed, strip secrets, and check that tool usage is disclosed. The map checks have no OpenBark equivalent. The structure is kept and the checks are replaced:

- **Citations resolve.** Every citation id in the answer must be one the index returned this turn. A fabricated RFC number, file path, or commit fails the check.
- **Internal claims are cited.** A sentence that asserts something about institutional material without a citation is flagged. The answer is regenerated once, then degraded to a statement that the assistant could not ground the claim.
- **Action claims match reality.** An answer may not say an action was taken unless an execution was recorded this run. This is the equivalent of Bark's false-lookup-failure repair, pointed at the more dangerous direction.
- **The decision to act originates with a person.** Only a member's own message may supply the intent to take an action. Retrieved content may supply facts that fill in an action's arguments, such as a date, a recipient group, or a room, but it may never be the reason an action is proposed. An action of a class that no member utterance in the conversation requested is blocked and surfaced as a potential injection. This is the containment for untrusted retrieved material, and the check is on where the intent came from, not on whether the argument text appears in the conversation.
- **No leaks.** Secrets, system prompt text, and assertion contents are stripped, as in Bark.

All of these are deterministic. Bark's reasoning against a second model call for verification holds and is stronger here, because a verifier is itself vulnerable to the injection it is checking for.

### Memory

Personal memory keeps bark RFC 0002's design: only durable facts, never raw turns, keyed per member, stored in a LangGraph store on PostgreSQL with pgvector, recalled semantically, extracted by a background task, listed and deleted by the member through the Surface.

The one change is a boundary. Personal memory holds facts about the member. Institutional knowledge belongs in the index, where it is versioned, cited, and role-filtered. A fact the member states about the organization is not promoted into shared knowledge, because memory has no provenance and no access control, and shared knowledge needs both.

### Limits and model policy

- Each member has a daily token budget, tracked as in Bark. Requests over it receive HTTP 429.
- Prompts carry private material, so model providers must be allowlisted. `ALLOWED_MODEL_IDS` is the set the Agent will route to, every entry must be served by a provider configured for zero retention or self-hosted, and the Agent rejects an id outside the set rather than trusting the Surface's validation of its own curated subset. This reverses bark RFC 0002's argument that OpenRouter lets the model list change without Agent changes, a convenience that was priced against public campus data.
- Output passes the guards above. Input moderation is optional for an internal audience and off by default.
- With `AGENT_ENV=production`, startup fails when the following is true
    - No `DATABASE_URL`, 
    - Missing an assertion verification key, and 
    - One or both of OpenBark tool servers are not configured and reachable. 
- Assertion contents, read tool arguments, and message text are not logged. Write tool arguments go to the audit log through the Surface (RFC 0006).

### HTTP API

Every route except `/api/health` requires the service bearer token. Routes that act for a member additionally require a valid signed assertion.

| Route | Description |
|---|---|
| `POST /agent/respond` | The complete answer as JSON |
| `POST /agent/respond/stream` | The answer as Server-Sent Events |
| `POST /agent/title` | A short title from a chat's first message |
| `POST /agent/runs/{run_id}/resume` | Resumes a suspended run after an approval |
| `DELETE /agent/runs/{run_id}` | Abandons a suspended run after a rejection |
| `GET /memory/{member}` | Personal facts, with search and paging |
| `DELETE /memory/{member}/items/{kind}/{item_id}` | Deletes one fact |
| `DELETE /memory/{member}` | Deletes all of a member's facts |
| `GET /api/health` | Status of the memory store, the knowledge index, and each tool server, with soft dependencies reported separately so a foreign server's outage does not read as an OpenBark failure |

Request fields for the respond routes: `query` (required), `assertion` (required), `message_history`, `model`, and `disabled_tools`. Unknown fields are rejected.

Stream events: `status`, `delta`, `citation` (a source the answer relies on), `action_preview` (what an action would do), `action_pending` (the run has suspended, with its id), `action_result` (an executed action's outcome), `memory`, `done`, and `error`. Bark's `map` event has no equivalent; the first four of these are new.

### Configuration

| Variable | Purpose |
|---|---|
| `MODEL_API_KEY`, `ALLOWED_MODEL_IDS` | Model access and the allowlist (required) |
| `MCP_SERVERS` | Tool servers: url, transport, auth mode, whether the assertion is forwarded, classification default, and hard or soft (required) |
| `INDEX_URL` | Retrieval service (required) |
| `ASSERTION_PUBLIC_KEY` | Verifying the Surface's user assertions (required) |
| `SURFACE_API_URL`, `SURFACE_SHARED_SECRET` | Writing audit records and reading approvals (required) |
| `EMBEDDING_API_KEY` | Personal memory embeddings (recommended) |
| `DATABASE_URL` | Memory and run state (required in production) |
| `AGENT_ENV` | `production` turns on the startup checks |
| `ACTION_PENDING_TTL_HOURS` | Default 72 |
| `PORT` | Default 5056 |

### Testing

- **Unit tests** run offline with a stubbed model, a stubbed index, and the in-memory store. CI runs them on every push. A test asserts that every tool in a recorded tool list belongs to a known group and carries a classification.
- **Answer evaluations** measure citation accuracy and groundedness, as the answer-side half of the suite RFC 0005 defines and gates in CI.
- **Adversarial evaluations** are required before any write tool ships. A corpus seeded with documents containing instructions to the model must produce neither an action nor an action preview when no member utterance requested one. A paired set covers the other direction, so that a member's genuine request is not blocked when a retrieved document supplies part of its detail.

### Development environment

`devenv up` starts PostgreSQL with pgvector and the Agent on port 5056, with dev secrets from OpenBao (RFC 0001). By default it points at the local index and tool servers.

## Alternatives Considered

- **Retrieval as an MCP tool rather than a lifecycle stage.** It would need no new contract: the index becomes a tool group and the model decides when to search. Rejected for the main path because retrieval must be role-filtered and cited on every grounded answer, and leaving that to the model's discretion means some answers silently skip it. A `kb` tool group exists as well (RFC 0004) for follow-up searches the model chooses to run, but the planner's retrieval stage is what guarantees grounding.
- **Verifying answers with a second model call.** Bark rejects this for latency, cost, and reliability. The same reasoning applies, plus a new one: a model verifier reads the same untrusted retrieved content and can be subverted by it. Deterministic checks cannot.
- **Holding suspended runs in memory with a long-lived connection.** Simpler, no run persistence. Rejected because an approval can take days, a deploy would drop every pending action, and the Surface needs to show pending actions to people other than the member who triggered them.
- **Letting the model re-derive action arguments on resume.** It would handle the case where the context changed usefully. Rejected: a human approved a specific preview, and an action that executes with different arguments than the ones approved is the failure the approval exists to prevent.
- **Promoting personal memory into shared knowledge.** It would capture institutional facts members state in chat, which is real knowledge the index does not have. Rejected because memory has no provenance and no access control, so a fact learned from one member would be served to everyone, uncited. A member who wants something in the index should put it in a source the index ingests.
- **Keeping Bark's open model routing.** It keeps the model list a Surface-side concern. Rejected: a prompt containing private material cannot be sent to an arbitrary provider chosen at request time.

## Open Questions

- How should the planner decide to retrieve? Bark's hint patterns are hand-written per tool group and the same approach would work, but retrieval is nearly always useful and the cheaper default may be to retrieve unless the message is clearly conversational.
- Should the regeneration pass for uncited claims be one attempt or configurable? One is proposed to bound latency.
- Does the token budget need to be per-role rather than per-member, given that some members will use OpenBark far more heavily?
- Where exactly does intent end and detail begin? "Email everyone about the deadline change" is clearly intent from the member with the date supplied by a document, and a document that alone asks for a broadcast is clearly not. Between them sit cases like a member saying "do what the meeting notes say we agreed," which delegates intent to a document on purpose. Proposed: treat an explicit delegation as intent for the class the member named and no other, so it cannot widen into an unrelated action, and require the preview to show which document the detail came from. The rule needs to be written precisely enough to implement before the first write provider ships.

## Implementation Phases

**Answering**

- FastAPI app, assertion verification, planning, graph, streaming, personal memory, token budget
- Model allowlist enforced at startup

**Grounding**

- Retrieval stage against the index (RFC 0005), `citation` events, citation verification in `guards.py`
- Retrieval evaluations with a fixture corpus

**Actions**

- `actions.py`, run persistence, suspend and resume, audit emission (RFC 0006)
- Adversarial evaluations, which gate the first write tool
