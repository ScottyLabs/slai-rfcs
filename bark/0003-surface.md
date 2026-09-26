# RFC 0003: Surface

- **Status:** Draft
- **Author(s):** @krishsax
- **Created:** 2026-09-26
- **Updated:** 2026-09-26
- **Affects:** cmugpt-surface

## Overview

This RFC defines the design of `cmugpt-surface`, the user-facing half of Bark: a React single-page app and an Express backend-for-frontend (BFF). The Surface signs users in with Andrew ID, stores their chats, relays answers from the Agent, and exposes memory and preference controls. The Agent does all of the model work.

## Motivation

The Surface holds Bark's only user-identifying data and its only authentication boundary, so its design decisions (where tokens live, what reaches the Agent, what is stored) need to be explicit and reviewable. It also defines the user-facing ends of several contracts from RFC 0001, such as the model list and tool toggles.

## Goals

- Document authentication and session handling
- Define what the Surface stores and what it sends to the Agent
- Document the API and how the frontend stays type-safe against it
- Define how the model list and tool toggles are maintained

## Non-Goals

- Visual design and UI copy
- Agent behavior and MCP tools (RFCs 0002 and 0004)

## Detailed Design

### Stack

A Deno workspace with two apps:

| App | Stack |
|---|---|
| `apps/web` | React 19, Vite, TanStack Router and TanStack Query, Tailwind CSS v4 |
| `apps/server` | Express 5, tsoa, Zod, Drizzle ORM on PostgreSQL |

### Type-safe API

`apps/server` is the source of truth for the API. tsoa generates an OpenAPI spec and Express routes from the controllers in `apps/server/src/controllers`, and `apps/web` imports the generated types and calls the API through `openapi-fetch` and `openapi-react-query`. `deno task generate` regenerates both, and it runs automatically before `dev` and `build`. Streaming and auth routes are written by hand in `apps/server/src/routes`, because they don't fit tsoa's request and response model.

### Authentication and sessions

- **Sign-in** uses OIDC against ScottyLabs' Keycloak (Andrew ID SSO), with PKCE and a nonce. The routes are `/api/auth/login`, `/api/auth/callback`, `/api/auth/logout`, and `/api/auth/me`.
- **Pending logins** are stored in `oidc_login_states`, keyed by the OAuth `state`, and expire after 10 minutes.
- **Sessions** are server-side. The browser holds only an opaque session id in an httpOnly cookie. Keycloak's access, refresh, and ID tokens live in `auth_sessions` and never reach the frontend. Sessions last 30 days, and expired rows are deleted.
- **Redirects through ricochet.** Keycloak redirects to the shared ricochet relay (`OAUTH_RELAY_URL`), which forwards the code to the deployment's own callback. That lets preview deployments sign in without registering their own redirect URIs. Locally, devenv runs ricochet on `127.0.0.1:8090`.
- **Admin access** is granted to members of the Keycloak group named in `ADMIN_GROUP` (`/me/oidc-admin`).

### Data

| Table | Contents |
|---|---|
| `auth_sessions`, `oidc_login_states` | Sessions and pending logins (above) |
| `chats` | One row per chat, owned by the user's OIDC subject, with a title and `starred` and `is_public` flags |
| `messages` | User and assistant messages, the campus map attached to an answer (`cmu_maps`), and the memory a turn saved |
| `user_preferences` | Preferred model |

Migrations live in `apps/server/drizzle` and run automatically when the server starts. Chats are deleted with their messages (`on delete cascade`). The Surface stores no memory facts. Those belong to the Agent.

### Chat flow

1. The web app posts to `POST /chats/{id}/messages/stream` with the message and the tool groups the user has switched off.
2. The server checks that the user owns the chat, stores the user message, and calls the Agent's stream route with the query, prior turns, `agentUserId(sub)`, the preferred model, and the disabled groups.
3. The server relays the Agent's `status`, `delta`, `map`, and `memory` events to the browser, and ignores event types it doesn't recognize. If the Agent stream sends nothing for 90 seconds, the server stops waiting and ends the stream.
4. On `done`, the server stores the assistant message and any map. The first message of a new chat gets a title from the Agent's `/agent/title`, or a shortened copy of the message if that call fails.

`agentUserId` sends `oidc:` followed by the SHA-256 of `oidc:<sub>`. The raw subject, email, and name never reach the Agent.

### API

| Route | Purpose |
|---|---|
| `GET /chats`, `POST /chats` | List and create chats |
| `GET /chats/{id}`, `PATCH /chats/{id}`, `DELETE /chats/{id}` | Read, rename, and delete a chat |
| `GET /chats/{id}/messages`, `POST /chats/{id}/messages` | Read history and send a message without streaming |
| `POST /chats/{id}/messages/stream` | Send a message and stream the answer |
| `PUT /chats/{id}/messages/{messageId}/saved-memory` | Record which memory a message saved |
| `GET /me/models` | The model list |
| `GET /me/preferences`, `PATCH /me/preferences` | The preferred model |
| `GET /me/memories`, `DELETE /me/memories/{kind}/{id}`, `DELETE /me/memories` | List and delete memory, proxied to the Agent |
| `GET /me/oidc-admin` | Whether the user is an admin |

### Models and tool toggles

- **Models.** `AGENT_MODELS` in `apps/server/src/lib/models.ts` is the curated list of OpenRouter model ids a user can choose from, usually four to six. Every entry must support tool calling, and the first entry is the default. Preferences are checked against the list with `isValidModelId`.
- **Tool toggles.** `AGENT_TOOL_IDS` in `apps/server/src/lib/agentTools.ts` lists the tool groups a user can switch off, and it must match the group list in RFC 0004. The chat UI holds the toggles and sends them with each message. They are not stored on the server. `sanitizeDisabledToolIds` drops unknown ids, so a stale client can't break a request.

### Configuration

Secrets are declared in `secretspec.toml` and validated at startup by `apps/server/src/env.ts`. `OIDC_ISSUER_URL`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, `ALLOWED_ORIGINS_REGEX`, `DATABASE_URL`, and `AGENT_API_URL` are required. `AGENT_SHARED_SECRET` is required in production and must be at least 32 characters with no surrounding whitespace. `OAUTH_RELAY_URL`, `APP_URL`, and `ADMIN_GROUP` are optional. The server listens on `PORT`, then `SERVER_PORT`, and defaults to 80.

### Deployment

Kennel serves `apps/web` as a static SPA at `bark.scottylabs.org` and runs the API, compiled into a standalone binary with `deno compile`, at `api.cmugpt.com`. `flake.nix` defines both packages.

### Testing

- Unit tests for `apps/web` with Vitest and Testing Library: `deno task --cwd apps/web test`
- End-to-end tests with Playwright in `apps/web/e2e`: `deno task test:e2e`
- Type checks for both apps: `deno task check`

### Development environment

`devenv up` starts PostgreSQL, ricochet, the API on port 3001, and Vite on port 4173, with dev secrets from OpenBao (RFC 0001). By default the server talks to the production Agent. Without Nix, local sign-in isn't available, because ricochet is provided through devenv.

## Alternatives Considered

- **Tokens in the browser (an SPA with PKCE and no BFF).** It removes the session tables. But access and refresh tokens would be exposed to any script on the page, and the Agent secret would need somewhere else to live. The BFF keeps every credential on the server.
- **Registering a redirect URI per deployment.** It avoids the relay. Keycloak doesn't allow wildcard redirect URIs, so every preview deployment would need provisioning. ricochet solves this once for all ScottyLabs projects.
- **Hand-written API types.** They are simpler at first. But they drift from the server. Generating the client from tsoa's OpenAPI spec makes a server change a compile error on the frontend.
- **Storing tool toggles on the server.** It would carry them across devices. They are a per-session choice today, so keeping them client-side avoids a schema change. This can move into `user_preferences` later without changing the Agent contract.

## Implementation Phases

**Tool groups**

- Add `bus` to `AGENT_TOOL_IDS` and a toggle in the chat UI, after the Agent registers it (RFCs 0002 and 0004)

**Cleanup**

- Remove the `scripts/secrets` submodule, left over from the old Vault setup
- Change the `user_preferences.preferred_model` column default (`openai/gpt-4o`) to the current `DEFAULT_MODEL_ID`
